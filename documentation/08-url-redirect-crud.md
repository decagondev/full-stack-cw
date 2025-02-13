# Module 8: URL Redirect CRUD

## Overview
In this module, we'll implement the core URL redirect functionality, including creating, reading, updating, and deleting URL redirects. We'll also implement URL validation, shortening logic, and user association.

## Prerequisites
- Completed Modules 0-7
- Understanding of CRUD operations
- Knowledge of URL validation
- Familiarity with MongoDB schemas

## Implementation Steps

### 1. Create URL Model

Create `src/models/url.model.ts`:
```typescript
import mongoose, { Document, Model } from 'mongoose';
import { nanoid } from 'nanoid';
import { isURL } from 'validator';
import { ApiError } from '../middleware/error.middleware';
import type { IUser } from './user.model';

export interface IUrl extends Document {
  shortId: string;
  originalUrl: string;
  title: string;
  description?: string;
  user: IUser['_id'];
  isActive: boolean;
  expiresAt?: Date;
  clicks: number;
  tags: string[];
  createdAt: Date;
  updatedAt: Date;
}

const urlSchema = new mongoose.Schema<IUrl>(
  {
    shortId: {
      type: String,
      required: true,
      unique: true,
      default: () => nanoid(8),
    },
    originalUrl: {
      type: String,
      required: true,
      validate: {
        validator: (value: string) => isURL(value, { require_protocol: true }),
        message: 'Invalid URL format. URL must include protocol (http:// or https://)',
      },
    },
    title: {
      type: String,
      required: true,
      trim: true,
      maxlength: 100,
    },
    description: {
      type: String,
      trim: true,
      maxlength: 500,
    },
    user: {
      type: mongoose.Schema.Types.ObjectId,
      ref: 'User',
      required: true,
    },
    isActive: {
      type: Boolean,
      default: true,
    },
    expiresAt: {
      type: Date,
      validate: {
        validator: (value: Date) => !value || value > new Date(),
        message: 'Expiration date must be in the future',
      },
    },
    clicks: {
      type: Number,
      default: 0,
    },
    tags: [{
      type: String,
      trim: true,
    }],
  },
  {
    timestamps: true,
  }
);

// Indexes
urlSchema.index({ shortId: 1 });
urlSchema.index({ user: 1 });
urlSchema.index({ tags: 1 });
urlSchema.index({ createdAt: 1 });
urlSchema.index({ clicks: -1 });

export const Url: Model<IUrl> = mongoose.model<IUrl>('Url', urlSchema);
```

### 2. Create URL Controller

Create `src/controllers/url.controller.ts`:
```typescript
import { Request, Response, NextFunction } from 'express';
import { Url } from '../models/url.model';
import { ApiError } from '../middleware/error.middleware';
import { validateUrlCreate, validateUrlUpdate } from '../validators/url.validator';

export class UrlController {
  static async create(req: Request, res: Response, next: NextFunction) {
    try {
      const validatedData = validateUrlCreate(req.body);
      const url = new Url({
        ...validatedData,
        user: req.user._id,
      });
      
      await url.save();
      
      res.status(201).json({
        status: 'success',
        data: url,
      });
    } catch (error) {
      next(error);
    }
  }

  static async list(req: Request, res: Response, next: NextFunction) {
    try {
      const page = parseInt(req.query.page as string) || 1;
      const limit = parseInt(req.query.limit as string) || 10;
      const search = req.query.search as string;
      
      const query: any = { user: req.user._id };
      if (search) {
        query.$or = [
          { title: new RegExp(search, 'i') },
          { tags: new RegExp(search, 'i') },
        ];
      }
      
      const [urls, total] = await Promise.all([
        Url.find(query)
          .sort({ createdAt: -1 })
          .skip((page - 1) * limit)
          .limit(limit),
        Url.countDocuments(query),
      ]);
      
      res.status(200).json({
        status: 'success',
        data: urls,
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

  static async get(req: Request, res: Response, next: NextFunction) {
    try {
      const url = await Url.findOne({
        shortId: req.params.shortId,
        user: req.user._id,
      });
      
      if (!url) {
        throw new ApiError(404, 'URL not found');
      }
      
      res.status(200).json({
        status: 'success',
        data: url,
      });
    } catch (error) {
      next(error);
    }
  }

  static async update(req: Request, res: Response, next: NextFunction) {
    try {
      const validatedData = validateUrlUpdate(req.body);
      const url = await Url.findOneAndUpdate(
        {
          shortId: req.params.shortId,
          user: req.user._id,
        },
        validatedData,
        { new: true }
      );
      
      if (!url) {
        throw new ApiError(404, 'URL not found');
      }
      
      res.status(200).json({
        status: 'success',
        data: url,
      });
    } catch (error) {
      next(error);
    }
  }

  static async delete(req: Request, res: Response, next: NextFunction) {
    try {
      const url = await Url.findOneAndDelete({
        shortId: req.params.shortId,
        user: req.user._id,
      });
      
      if (!url) {
        throw new ApiError(404, 'URL not found');
      }
      
      res.status(204).send();
    } catch (error) {
      next(error);
    }
  }

  static async redirect(req: Request, res: Response, next: NextFunction) {
    try {
      const url = await Url.findOne({
        shortId: req.params.shortId,
        isActive: true,
      });
      
      if (!url) {
        throw new ApiError(404, 'URL not found');
      }
      
      if (url.expiresAt && url.expiresAt < new Date()) {
        throw new ApiError(410, 'URL has expired');
      }
      
      // Increment clicks asynchronously
      url.clicks += 1;
      url.save().catch(console.error);
      
      res.redirect(url.originalUrl);
    } catch (error) {
      next(error);
    }
  }
}
```

### 3. Create URL Validation

Create `src/validators/url.validator.ts`:
```typescript
import * as z from 'zod';
import { ApiError } from '../middleware/error.middleware';

const urlSchema = z.object({
  originalUrl: z.string().url('Invalid URL format'),
  title: z.string().min(1, 'Title is required').max(100),
  description: z.string().max(500).optional(),
  tags: z.array(z.string()).optional(),
  expiresAt: z.string().datetime().optional(),
  isActive: z.boolean().optional(),
});

export const validateUrlCreate = (data: unknown) => {
  try {
    return urlSchema.parse(data);
  } catch (error) {
    if (error instanceof z.ZodError) {
      throw new ApiError(400, error.errors[0].message);
    }
    throw error;
  }
};

export const validateUrlUpdate = (data: unknown) => {
  try {
    return urlSchema.partial().parse(data);
  } catch (error) {
    if (error instanceof z.ZodError) {
      throw new ApiError(400, error.errors[0].message);
    }
    throw error;
  }
};
```

### 4. Set Up URL Routes

Create `src/routes/url.routes.ts`:
```typescript
import { Router } from 'express';
import { UrlController } from '../controllers/url.controller';
import { authenticate } from '../middleware/auth.middleware';

const router = Router();

// Protected routes
router.use(authenticate);
router.post('/', UrlController.create);
router.get('/', UrlController.list);
router.get('/:shortId', UrlController.get);
router.patch('/:shortId', UrlController.update);
router.delete('/:shortId', UrlController.delete);

// Public route
router.get('/r/:shortId', UrlController.redirect);

export default router;
```

Update `src/routes/index.ts`:
```typescript
import { Router } from 'express';
import authRoutes from './auth.routes';
import urlRoutes from './url.routes';

const router = Router();

router.use('/auth', authRoutes);
router.use('/urls', urlRoutes);

export default router;
```

### 5. Create URL Service

Create `src/services/url.service.ts`:
```typescript
import { Url, IUrl } from '../models/url.model';
import { ApiError } from '../middleware/error.middleware';

export class UrlService {
  static async generateUniqueShortId(length = 8): Promise<string> {
    const chars = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789';
    let shortId: string;
    let isUnique = false;
    
    while (!isUnique) {
      shortId = Array.from(
        { length },
        () => chars[Math.floor(Math.random() * chars.length)]
      ).join('');
      
      const existing = await Url.findOne({ shortId });
      if (!existing) {
        isUnique = true;
      }
    }
    
    return shortId!;
  }

  static async validateUrl(url: string): Promise<boolean> {
    try {
      new URL(url);
      return true;
    } catch {
      return false;
    }
  }

  static async getUrlStats(userId: string) {
    const [total, active, expired] = await Promise.all([
      Url.countDocuments({ user: userId }),
      Url.countDocuments({ user: userId, isActive: true }),
      Url.countDocuments({
        user: userId,
        expiresAt: { $lt: new Date() },
      }),
    ]);

    const topUrls = await Url.find({ user: userId })
      .sort({ clicks: -1 })
      .limit(5);

    return {
      total,
      active,
      expired,
      topUrls,
    };
  }
}
```

### 6. Implement Frontend URL Components

Create `src/components/url/UrlForm.tsx`:
```typescript
import React from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { urlSchema } from '@/lib/validations/url';
import { Button } from '../common/Button';
import { Input } from '../common/Input';
import { Textarea } from '../common/Textarea';
import { TagInput } from '../common/TagInput';

interface UrlFormProps {
  initialData?: any;
  onSubmit: (data: any) => Promise<void>;
  isLoading?: boolean;
}

export const UrlForm: React.FC<UrlFormProps> = ({
  initialData,
  onSubmit,
  isLoading,
}) => {
  const {
    register,
    handleSubmit,
    control,
    formState: { errors },
  } = useForm({
    resolver: zodResolver(urlSchema),
    defaultValues: initialData,
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <Input
        label="Original URL"
        {...register('originalUrl')}
        error={errors.originalUrl?.message}
      />
      <Input
        label="Title"
        {...register('title')}
        error={errors.title?.message}
      />
      <Textarea
        label="Description"
        {...register('description')}
        error={errors.description?.message}
      />
      <TagInput
        control={control}
        name="tags"
        label="Tags"
        error={errors.tags?.message}
      />
      <Input
        type="datetime-local"
        label="Expires At"
        {...register('expiresAt')}
        error={errors.expiresAt?.message}
      />
      <Button
        type="submit"
        className="w-full"
        isLoading={isLoading}
      >
        {initialData ? 'Update URL' : 'Create URL'}
      </Button>
    </form>
  );
};
```

### 7. Add URL API Service

Update `src/services/api.ts`:
```typescript
import api from '@/lib/axios';

// ... existing auth API endpoints

export const urlApi = {
  create: (data: any) =>
    api.post('/urls', data),
    
  list: (params?: any) =>
    api.get('/urls', { params }),
    
  get: (shortId: string) =>
    api.get(`/urls/${shortId}`),
    
  update: (shortId: string, data: any) =>
    api.patch(`/urls/${shortId}`, data),
    
  delete: (shortId: string) =>
    api.delete(`/urls/${shortId}`),
    
  getStats: () =>
    api.get('/urls/stats'),
};
```

## Testing URL Operations

Create `src/__tests__/url/url.test.ts`:
```typescript
import request from 'supertest';
import { app } from '../../app';
import { Url } from '../../models/url.model';
import { createTestUser, getAuthToken } from '../helpers';

describe('URL CRUD Operations', () => {
  let authToken: string;
  
  beforeAll(async () => {
    const user = await createTestUser();
    authToken = await getAuthToken(user);
  });
  
  beforeEach(async () => {
    await Url.deleteMany({});
  });
  
  it('creates a new URL', async () => {
    const response = await request(app)
      .post('/api/urls')
      .set('Authorization', `Bearer ${authToken}`)
      .send({
        originalUrl: 'https://example.com',
        title: 'Example',
      });
      
    expect(response.status).toBe(201);
    expect(response.body.data).toHaveProperty('shortId');
  });
  
  it('lists user URLs', async () => {
    const response = await request(app)
      .get('/api/urls')
      .set('Authorization', `Bearer ${authToken}`);
      
    expect(response.status).toBe(200);
    expect(response.body).toHaveProperty('pagination');
  });
});
```

## Best Practices

1. URL Handling:
   - Validate URLs thoroughly
   - Handle URL encoding
   - Check for malicious URLs
   - Implement rate limiting

2. Database Operations:
   - Use proper indexes
   - Implement pagination
   - Handle concurrent updates
   - Monitor performance

3. Security:
   - Validate user ownership
   - Sanitize inputs
   - Prevent URL manipulation
   - Monitor for abuse

4. Performance:
   - Cache frequent redirects
   - Use efficient queries
   - Implement batch operations
   - Monitor response times

## Common Issues and Solutions

1. URL Validation:
   - Handle different URL formats
   - Validate protocols
   - Check URL accessibility
   - Handle international URLs

2. Concurrency:
   - Handle race conditions
   - Implement locking
   - Use atomic operations
   - Handle duplicates

3. Performance:
   - Optimize database queries
   - Implement caching
   - Use proper indexes
   - Monitor bottlenecks

## Next Steps

After completing this module, you should have:
- [ ] Implemented URL model
- [ ] Created CRUD endpoints
- [ ] Added URL validation
- [ ] Implemented shortening logic
- [ ] Created frontend components
- [ ] Added URL statistics
- [ ] Implemented testing

Ready to proceed to Module 9: Admin User Management API.

## Additional Resources

- [MongoDB Indexing Strategies](https://docs.mongodb.com/manual/applications/indexes/)
- [URL Validation Best Practices](https://www.rfc-editor.org/rfc/rfc3986)
- [Rate Limiting Strategies](https://cloud.google.com/architecture/rate-limiting-strategies-patterns)
- [REST API Best Practices](https://docs.microsoft.com/en-us/azure/architecture/best-practices/api-design) 