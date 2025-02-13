# Module 5: Backend Authentication

## Overview
In this module, we'll implement a secure authentication system using JWT (JSON Web Tokens) with role-based access control. We'll create user management endpoints, implement password hashing, and set up authentication middleware.

## Prerequisites
- Completed Modules 0-4
- Understanding of JWT and authentication concepts
- Basic knowledge of password hashing and security

## Implementation Steps

### 1. Create User Model

Create `src/models/user.model.ts`:
```typescript
import mongoose, { Document, Model } from 'mongoose';
import bcrypt from 'bcryptjs';
import { ApiError } from '../middleware/error.middleware';

export enum UserRole {
  USER = 'user',
  STAFF = 'staff',
  ADMIN = 'admin',
}

export interface IUser extends Document {
  email: string;
  password: string;
  firstName: string;
  lastName: string;
  role: UserRole;
  isActive: boolean;
  lastLogin?: Date;
  comparePassword(candidatePassword: string): Promise<boolean>;
}

const userSchema = new mongoose.Schema<IUser>(
  {
    email: {
      type: String,
      required: true,
      unique: true,
      lowercase: true,
      trim: true,
    },
    password: {
      type: String,
      required: true,
      minlength: 8,
    },
    firstName: {
      type: String,
      required: true,
      trim: true,
    },
    lastName: {
      type: String,
      required: true,
      trim: true,
    },
    role: {
      type: String,
      enum: Object.values(UserRole),
      default: UserRole.USER,
    },
    isActive: {
      type: Boolean,
      default: true,
    },
    lastLogin: {
      type: Date,
    },
  },
  {
    timestamps: true,
  }
);

// Hash password before saving
userSchema.pre('save', async function (next) {
  if (!this.isModified('password')) return next();

  try {
    const salt = await bcrypt.genSalt(12);
    this.password = await bcrypt.hash(this.password, salt);
    next();
  } catch (error) {
    next(error as Error);
  }
});

// Compare password method
userSchema.methods.comparePassword = async function (
  candidatePassword: string
): Promise<boolean> {
  return bcrypt.compare(candidatePassword, this.password);
};

export const User: Model<IUser> = mongoose.model<IUser>('User', userSchema);
```

### 2. Implement JWT Service

Create `src/services/jwt.service.ts`:
```typescript
import jwt from 'jsonwebtoken';
import { environment } from '../config/environment';
import { UserRole } from '../models/user.model';

interface TokenPayload {
  userId: string;
  email: string;
  role: UserRole;
}

export class JwtService {
  static generateToken(payload: TokenPayload): string {
    return jwt.sign(payload, environment.jwtSecret, {
      expiresIn: environment.jwtExpiresIn,
    });
  }

  static verifyToken(token: string): TokenPayload {
    try {
      return jwt.verify(token, environment.jwtSecret) as TokenPayload;
    } catch (error) {
      throw new Error('Invalid token');
    }
  }

  static generateRefreshToken(userId: string): string {
    return jwt.sign({ userId }, environment.jwtRefreshSecret, {
      expiresIn: '7d',
    });
  }
}
```

### 3. Create Authentication Controller

Create `src/controllers/auth.controller.ts`:
```typescript
import { Request, Response, NextFunction } from 'express';
import { User } from '../models/user.model';
import { JwtService } from '../services/jwt.service';
import { ApiError } from '../middleware/error.middleware';
import { validateRegistration, validateLogin } from '../validators/auth.validator';

export class AuthController {
  static async register(req: Request, res: Response, next: NextFunction) {
    try {
      const validatedData = validateRegistration(req.body);
      
      const existingUser = await User.findOne({ email: validatedData.email });
      if (existingUser) {
        throw new ApiError(400, 'Email already registered');
      }

      const user = new User(validatedData);
      await user.save();

      const token = JwtService.generateToken({
        userId: user._id,
        email: user.email,
        role: user.role,
      });

      res.status(201).json({
        status: 'success',
        data: {
          token,
          user: {
            id: user._id,
            email: user.email,
            firstName: user.firstName,
            lastName: user.lastName,
            role: user.role,
          },
        },
      });
    } catch (error) {
      next(error);
    }
  }

  static async login(req: Request, res: Response, next: NextFunction) {
    try {
      const validatedData = validateLogin(req.body);

      const user = await User.findOne({ email: validatedData.email });
      if (!user || !user.isActive) {
        throw new ApiError(401, 'Invalid credentials');
      }

      const isPasswordValid = await user.comparePassword(validatedData.password);
      if (!isPasswordValid) {
        throw new ApiError(401, 'Invalid credentials');
      }

      user.lastLogin = new Date();
      await user.save();

      const token = JwtService.generateToken({
        userId: user._id,
        email: user.email,
        role: user.role,
      });

      const refreshToken = JwtService.generateRefreshToken(user._id);

      res.status(200).json({
        status: 'success',
        data: {
          token,
          refreshToken,
          user: {
            id: user._id,
            email: user.email,
            firstName: user.firstName,
            lastName: user.lastName,
            role: user.role,
          },
        },
      });
    } catch (error) {
      next(error);
    }
  }
}
```

### 4. Implement Authentication Middleware

Create `src/middleware/auth.middleware.ts`:
```typescript
import { Request, Response, NextFunction } from 'express';
import { JwtService } from '../services/jwt.service';
import { User, UserRole } from '../models/user.model';
import { ApiError } from './error.middleware';

declare global {
  namespace Express {
    interface Request {
      user?: any;
    }
  }
}

export const authenticate = async (
  req: Request,
  _res: Response,
  next: NextFunction
) => {
  try {
    const authHeader = req.headers.authorization;
    if (!authHeader?.startsWith('Bearer ')) {
      throw new ApiError(401, 'No token provided');
    }

    const token = authHeader.split(' ')[1];
    const decoded = JwtService.verifyToken(token);

    const user = await User.findById(decoded.userId).select('-password');
    if (!user || !user.isActive) {
      throw new ApiError(401, 'User not found or inactive');
    }

    req.user = user;
    next();
  } catch (error) {
    next(new ApiError(401, 'Invalid token'));
  }
};

export const authorize = (...roles: UserRole[]) => {
  return (req: Request, _res: Response, next: NextFunction) => {
    if (!req.user) {
      return next(new ApiError(401, 'Not authenticated'));
    }

    if (!roles.includes(req.user.role)) {
      return next(new ApiError(403, 'Not authorized'));
    }

    next();
  };
};
```

### 5. Create Input Validators

Create `src/validators/auth.validator.ts`:
```typescript
import Joi from 'joi';
import { ApiError } from '../middleware/error.middleware';

const registrationSchema = Joi.object({
  email: Joi.string().email().required(),
  password: Joi.string().min(8).required(),
  firstName: Joi.string().required(),
  lastName: Joi.string().required(),
});

const loginSchema = Joi.object({
  email: Joi.string().email().required(),
  password: Joi.string().required(),
});

export const validateRegistration = (data: any) => {
  const { error, value } = registrationSchema.validate(data);
  if (error) {
    throw new ApiError(400, error.details[0].message);
  }
  return value;
};

export const validateLogin = (data: any) => {
  const { error, value } = loginSchema.validate(data);
  if (error) {
    throw new ApiError(400, error.details[0].message);
  }
  return value;
};
```

### 6. Set Up Authentication Routes

Create `src/routes/auth.routes.ts`:
```typescript
import { Router } from 'express';
import { AuthController } from '../controllers/auth.controller';

const router = Router();

router.post('/register', AuthController.register);
router.post('/login', AuthController.login);

export default router;
```

Update `src/routes/index.ts`:
```typescript
import { Router } from 'express';
import authRoutes from './auth.routes';
import healthRoutes from './health.routes';

const router = Router();

router.use('/auth', authRoutes);
router.use('/health', healthRoutes);

export default router;
```

### 7. Update Environment Configuration

Update `src/config/environment.ts`:
```typescript
export const environment = {
  // ... existing config
  jwtSecret: process.env.JWT_SECRET || 'your-jwt-secret',
  jwtExpiresIn: process.env.JWT_EXPIRES_IN || '1d',
  jwtRefreshSecret: process.env.JWT_REFRESH_SECRET || 'your-refresh-secret',
};
```

## Testing Authentication

Create `src/__tests__/auth.test.ts`:
```typescript
import request from 'supertest';
import app from '../app';
import { User } from '../models/user.model';

describe('Authentication Endpoints', () => {
  beforeEach(async () => {
    await User.deleteMany({});
  });

  describe('POST /api/auth/register', () => {
    it('should register a new user', async () => {
      const response = await request(app)
        .post('/api/auth/register')
        .send({
          email: 'test@example.com',
          password: 'password123',
          firstName: 'Test',
          lastName: 'User',
        });

      expect(response.status).toBe(201);
      expect(response.body.data).toHaveProperty('token');
      expect(response.body.data.user).toHaveProperty('email', 'test@example.com');
    });
  });

  describe('POST /api/auth/login', () => {
    it('should login an existing user', async () => {
      // First register a user
      await request(app)
        .post('/api/auth/register')
        .send({
          email: 'test@example.com',
          password: 'password123',
          firstName: 'Test',
          lastName: 'User',
        });

      // Then try to login
      const response = await request(app)
        .post('/api/auth/login')
        .send({
          email: 'test@example.com',
          password: 'password123',
        });

      expect(response.status).toBe(200);
      expect(response.body.data).toHaveProperty('token');
      expect(response.body.data).toHaveProperty('refreshToken');
    });
  });
});
```

## Best Practices

1. Password Security:
   - Use strong password hashing (bcrypt)
   - Implement password complexity requirements
   - Never store plain-text passwords
   - Use secure password reset flows

2. JWT Security:
   - Use strong secrets
   - Set appropriate expiration times
   - Implement refresh token rotation
   - Secure token storage

3. Input Validation:
   - Validate all input data
   - Sanitize user inputs
   - Use strong validation schemas
   - Handle validation errors gracefully

4. Error Handling:
   - Use consistent error responses
   - Don't leak sensitive information
   - Log authentication failures
   - Implement rate limiting

## Common Issues and Solutions

1. Token Issues:
   - Verify token expiration times
   - Check token signing algorithm
   - Validate token payload
   - Implement proper error handling

2. Password Problems:
   - Review password hashing configuration
   - Check password comparison logic
   - Verify password reset flow
   - Implement account lockout

3. Security Concerns:
   - Enable HTTPS
   - Implement CORS properly
   - Use secure headers
   - Monitor for suspicious activity

## Next Steps

After completing this module, you should have:
- [ ] Implemented user model with roles
- [ ] Created JWT service
- [ ] Set up authentication controller
- [ ] Implemented authentication middleware
- [ ] Added input validation
- [ ] Created authentication routes
- [ ] Written authentication tests

Ready to proceed to Module 6: Frontend Foundation.

## Additional Resources

- [JWT.io](https://jwt.io/)
- [bcrypt Documentation](https://github.com/kelektiv/node.bcrypt.js)
- [Express JWT Documentation](https://github.com/auth0/express-jwt)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Node.js Security Best Practices](https://github.com/goldbergyoni/nodebestpractices#6-security-best-practices) 