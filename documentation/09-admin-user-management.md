# Module 9: Admin User Management API

## Overview
In this module, we'll implement the admin user management functionality, allowing administrators to manage users, roles, and permissions. We'll create endpoints for user CRUD operations, role management, and implement search and filtering capabilities.

## Prerequisites
- Completed Modules 0-8
- Understanding of role-based access control (RBAC)
- Knowledge of pagination and filtering
- Familiarity with MongoDB aggregation

## Implementation Steps

### 1. Update User Model

Update `src/models/user.model.ts`:
```typescript
import mongoose, { Document, Model } from 'mongoose';
import bcrypt from 'bcryptjs';
import { ApiError } from '../middleware/error.middleware';

export enum UserRole {
  USER = 'user',
  STAFF = 'staff',
  ADMIN = 'admin',
}

export enum UserStatus {
  ACTIVE = 'active',
  SUSPENDED = 'suspended',
  DEACTIVATED = 'deactivated',
}

export interface IUser extends Document {
  email: string;
  password: string;
  firstName: string;
  lastName: string;
  role: UserRole;
  status: UserStatus;
  lastLogin?: Date;
  metadata: {
    loginAttempts: number;
    lastFailedLogin?: Date;
    passwordChangedAt?: Date;
  };
  createdAt: Date;
  updatedAt: Date;
}

const userSchema = new mongoose.Schema<IUser>(
  {
    // ... existing fields ...
    status: {
      type: String,
      enum: Object.values(UserStatus),
      default: UserStatus.ACTIVE,
    },
    metadata: {
      loginAttempts: {
        type: Number,
        default: 0,
      },
      lastFailedLogin: Date,
      passwordChangedAt: Date,
    },
  },
  { timestamps: true }
);

// Add indexes for admin queries
userSchema.index({ email: 1 });
userSchema.index({ role: 1 });
userSchema.index({ status: 1 });
userSchema.index({ createdAt: 1 });
```

### 2. Create Admin Controller

Create `src/controllers/admin.controller.ts`:
```typescript
import { Request, Response, NextFunction } from 'express';
import { User, UserRole, UserStatus } from '../models/user.model';
import { ApiError } from '../middleware/error.middleware';
import {
  validateUserCreate,
  validateUserUpdate,
} from '../validators/admin.validator';

export class AdminController {
  static async listUsers(req: Request, res: Response, next: NextFunction) {
    try {
      const page = parseInt(req.query.page as string) || 1;
      const limit = parseInt(req.query.limit as string) || 10;
      const search = req.query.search as string;
      const role = req.query.role as UserRole;
      const status = req.query.status as UserStatus;
      const sortBy = req.query.sortBy as string || 'createdAt';
      const sortOrder = req.query.sortOrder as string || 'desc';

      const query: any = {};
      if (search) {
        query.$or = [
          { email: new RegExp(search, 'i') },
          { firstName: new RegExp(search, 'i') },
          { lastName: new RegExp(search, 'i') },
        ];
      }
      if (role) query.role = role;
      if (status) query.status = status;

      const [users, total] = await Promise.all([
        User.find(query)
          .select('-password')
          .sort({ [sortBy]: sortOrder === 'desc' ? -1 : 1 })
          .skip((page - 1) * limit)
          .limit(limit),
        User.countDocuments(query),
      ]);

      res.status(200).json({
        status: 'success',
        data: users,
        pagination: {
          page,
          limit,
          total,
          pages: Math.ceil(total / limit),
        },
      });
    } catch (error) {
      next(error);
    }
  }

  static async createUser(req: Request, res: Response, next: NextFunction) {
    try {
      const validatedData = validateUserCreate(req.body);
      
      const existingUser = await User.findOne({ email: validatedData.email });
      if (existingUser) {
        throw new ApiError(400, 'Email already registered');
      }

      const user = new User(validatedData);
      await user.save();

      res.status(201).json({
        status: 'success',
        data: user,
      });
    } catch (error) {
      next(error);
    }
  }

  static async updateUser(req: Request, res: Response, next: NextFunction) {
    try {
      const validatedData = validateUserUpdate(req.body);
      
      const user = await User.findByIdAndUpdate(
        req.params.userId,
        validatedData,
        { new: true }
      ).select('-password');

      if (!user) {
        throw new ApiError(404, 'User not found');
      }

      res.status(200).json({
        status: 'success',
        data: user,
      });
    } catch (error) {
      next(error);
    }
  }

  static async deleteUser(req: Request, res: Response, next: NextFunction) {
    try {
      const user = await User.findById(req.params.userId);
      
      if (!user) {
        throw new ApiError(404, 'User not found');
      }

      if (user.role === UserRole.ADMIN) {
        throw new ApiError(403, 'Cannot delete admin user');
      }

      await user.remove();

      res.status(204).send();
    } catch (error) {
      next(error);
    }
  }

  static async getUserStats(req: Request, res: Response, next: NextFunction) {
    try {
      const stats = await User.aggregate([
        {
          $group: {
            _id: null,
            total: { $sum: 1 },
            active: {
              $sum: {
                $cond: [{ $eq: ['$status', UserStatus.ACTIVE] }, 1, 0],
              },
            },
            suspended: {
              $sum: {
                $cond: [{ $eq: ['$status', UserStatus.SUSPENDED] }, 1, 0],
              },
            },
            deactivated: {
              $sum: {
                $cond: [{ $eq: ['$status', UserStatus.DEACTIVATED] }, 1, 0],
              },
            },
          },
        },
      ]);

      const roleStats = await User.aggregate([
        {
          $group: {
            _id: '$role',
            count: { $sum: 1 },
          },
        },
      ]);

      res.status(200).json({
        status: 'success',
        data: {
          userStats: stats[0],
          roleStats: roleStats.reduce((acc, curr) => ({
            ...acc,
            [curr._id]: curr.count,
          }), {}),
        },
      });
    } catch (error) {
      next(error);
    }
  }
}
```

### 3. Create Admin Validation

Create `src/validators/admin.validator.ts`:
```typescript
import * as z from 'zod';
import { UserRole, UserStatus } from '../models/user.model';
import { ApiError } from '../middleware/error.middleware';

const userCreateSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(
      /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
      'Password must contain at least one uppercase letter, one lowercase letter, and one number'
    ),
  firstName: z.string().min(2, 'First name must be at least 2 characters'),
  lastName: z.string().min(2, 'Last name must be at least 2 characters'),
  role: z.enum(Object.values(UserRole) as [string, ...string[]]),
  status: z.enum(Object.values(UserStatus) as [string, ...string[]]),
});

const userUpdateSchema = userCreateSchema
  .partial()
  .omit({ password: true });

export const validateUserCreate = (data: unknown) => {
  try {
    return userCreateSchema.parse(data);
  } catch (error) {
    if (error instanceof z.ZodError) {
      throw new ApiError(400, error.errors[0].message);
    }
    throw error;
  }
};

export const validateUserUpdate = (data: unknown) => {
  try {
    return userUpdateSchema.parse(data);
  } catch (error) {
    if (error instanceof z.ZodError) {
      throw new ApiError(400, error.errors[0].message);
    }
    throw error;
  }
};
```

### 4. Set Up Admin Routes

Create `src/routes/admin.routes.ts`:
```typescript
import { Router } from 'express';
import { AdminController } from '../controllers/admin.controller';
import { authenticate, authorize } from '../middleware/auth.middleware';
import { UserRole } from '../models/user.model';

const router = Router();

// All routes require authentication and admin role
router.use(authenticate, authorize(UserRole.ADMIN));

router.get('/users', AdminController.listUsers);
router.post('/users', AdminController.createUser);
router.patch('/users/:userId', AdminController.updateUser);
router.delete('/users/:userId', AdminController.deleteUser);
router.get('/stats', AdminController.getUserStats);

export default router;
```

Update `src/routes/index.ts`:
```typescript
import { Router } from 'express';
import authRoutes from './auth.routes';
import urlRoutes from './url.routes';
import adminRoutes from './admin.routes';

const router = Router();

router.use('/auth', authRoutes);
router.use('/urls', urlRoutes);
router.use('/admin', adminRoutes);

export default router;
```

### 5. Create Admin Service

Create `src/services/admin.service.ts`:
```typescript
import { User, UserRole, UserStatus } from '../models/user.model';
import { ApiError } from '../middleware/error.middleware';

export class AdminService {
  static async validateUserOperation(userId: string, adminId: string) {
    const [targetUser, adminUser] = await Promise.all([
      User.findById(userId),
      User.findById(adminId),
    ]);

    if (!targetUser) {
      throw new ApiError(404, 'User not found');
    }

    if (!adminUser || adminUser.role !== UserRole.ADMIN) {
      throw new ApiError(403, 'Not authorized');
    }

    // Prevent admin from modifying other admins
    if (targetUser.role === UserRole.ADMIN && targetUser.id !== adminUser.id) {
      throw new ApiError(403, 'Cannot modify other admin users');
    }

    return { targetUser, adminUser };
  }

  static async getUserAuditLog(userId: string) {
    const user = await User.findById(userId);
    if (!user) {
      throw new ApiError(404, 'User not found');
    }

    return {
      id: user.id,
      email: user.email,
      status: user.status,
      role: user.role,
      loginAttempts: user.metadata.loginAttempts,
      lastFailedLogin: user.metadata.lastFailedLogin,
      passwordChangedAt: user.metadata.passwordChangedAt,
      createdAt: user.createdAt,
      updatedAt: user.updatedAt,
    };
  }
}
```

### 6. Add Admin API Service

Update `src/services/api.ts`:
```typescript
import api from '@/lib/axios';

// ... existing API endpoints

export const adminApi = {
  listUsers: (params?: any) =>
    api.get('/admin/users', { params }),
    
  createUser: (data: any) =>
    api.post('/admin/users', data),
    
  updateUser: (userId: string, data: any) =>
    api.patch(`/admin/users/${userId}`, data),
    
  deleteUser: (userId: string) =>
    api.delete(`/admin/users/${userId}`),
    
  getUserStats: () =>
    api.get('/admin/stats'),
};
```

## Testing Admin Operations

Create `src/__tests__/admin/admin.test.ts`:
```typescript
import request from 'supertest';
import { app } from '../../app';
import { User, UserRole, UserStatus } from '../../models/user.model';
import { createTestUser, getAuthToken } from '../helpers';

describe('Admin User Management', () => {
  let adminToken: string;
  
  beforeAll(async () => {
    const admin = await createTestUser({
      role: UserRole.ADMIN,
    });
    adminToken = await getAuthToken(admin);
  });
  
  beforeEach(async () => {
    await User.deleteMany({ role: { $ne: UserRole.ADMIN } });
  });
  
  it('lists users with pagination and filters', async () => {
    const response = await request(app)
      .get('/api/admin/users')
      .set('Authorization', `Bearer ${adminToken}`)
      .query({
        page: 1,
        limit: 10,
        role: UserRole.USER,
        status: UserStatus.ACTIVE,
      });
      
    expect(response.status).toBe(200);
    expect(response.body).toHaveProperty('pagination');
    expect(response.body.data).toBeInstanceOf(Array);
  });
  
  it('creates a new user', async () => {
    const response = await request(app)
      .post('/api/admin/users')
      .set('Authorization', `Bearer ${adminToken}`)
      .send({
        email: 'test@example.com',
        password: 'Password123',
        firstName: 'Test',
        lastName: 'User',
        role: UserRole.USER,
        status: UserStatus.ACTIVE,
      });
      
    expect(response.status).toBe(201);
    expect(response.body.data).toHaveProperty('email', 'test@example.com');
  });
});
```

## Best Practices

1. Access Control:
   - Implement strict role checks
   - Validate admin permissions
   - Log admin actions
   - Prevent privilege escalation

2. User Management:
   - Handle user status changes
   - Implement soft deletes
   - Track user activity
   - Monitor suspicious behavior

3. Data Security:
   - Sanitize sensitive data
   - Implement audit logs
   - Secure admin routes
   - Rate limit admin actions

4. Performance:
   - Optimize queries
   - Implement caching
   - Use proper indexes
   - Handle large datasets

## Common Issues and Solutions

1. Permission Issues:
   - Validate role hierarchy
   - Check operation permissions
   - Handle edge cases
   - Implement role inheritance

2. Data Management:
   - Handle bulk operations
   - Validate data integrity
   - Manage relationships
   - Handle cascading updates

3. Security Concerns:
   - Prevent privilege escalation
   - Monitor admin actions
   - Implement audit trails
   - Handle sensitive data

## Next Steps

After completing this module, you should have:
- [ ] Updated user model
- [ ] Implemented admin controller
- [ ] Added validation rules
- [ ] Created admin routes
- [ ] Implemented admin service
- [ ] Added API endpoints
- [ ] Created admin tests

Ready to proceed to Module 10: Admin Dashboard.

## Additional Resources

- [Role-Based Access Control (RBAC)](https://en.wikipedia.org/wiki/Role-based_access_control)
- [MongoDB Aggregation](https://docs.mongodb.com/manual/aggregation/)
- [API Security Best Practices](https://owasp.org/www-project-api-security/)
- [User Management Patterns](https://www.oauth.com/oauth2-servers/access-control/) 