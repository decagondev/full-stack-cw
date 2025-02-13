# Module 14: Feature Polish

## Overview
In this module, we'll polish the application by implementing rate limiting, input validation, error handling, loading states, and toast notifications. We'll also enhance the user experience with better feedback and smoother interactions.

## Prerequisites
- Completed Modules 0-13
- Understanding of UX best practices
- Knowledge of security measures
- Familiarity with error handling

## Implementation Steps

### 1. Implement Rate Limiting

Create `src/middleware/rate-limit.middleware.ts`:
```typescript
import rateLimit from 'express-rate-limit';
import RedisStore from 'rate-limit-redis';
import { redis } from '../lib/redis';
import { environment } from '../config/environment';

const createLimiter = (
  windowMs: number,
  max: number,
  message: string
) => {
  return rateLimit({
    store: new RedisStore({
      client: redis,
      prefix: 'rate-limit:',
    }),
    windowMs,
    max,
    message: {
      status: 'error',
      message,
    },
    standardHeaders: true,
    legacyHeaders: false,
  });
};

export const apiLimiter = createLimiter(
  15 * 60 * 1000, // 15 minutes
  environment.nodeEnv === 'production' ? 100 : 1000,
  'Too many requests from this IP, please try again later.'
);

export const authLimiter = createLimiter(
  60 * 60 * 1000, // 1 hour
  5,
  'Too many login attempts, please try again later.'
);

export const urlCreationLimiter = createLimiter(
  60 * 60 * 1000, // 1 hour
  environment.nodeEnv === 'production' ? 50 : 200,
  'URL creation limit reached, please try again later.'
);
```

### 2. Enhance Input Validation

Create `src/lib/validations/index.ts`:
```typescript
import { z } from 'zod';

export const urlSchema = z.object({
  originalUrl: z
    .string()
    .url('Invalid URL format')
    .max(2048, 'URL is too long'),
  title: z
    .string()
    .min(3, 'Title must be at least 3 characters')
    .max(100, 'Title must be less than 100 characters'),
  description: z
    .string()
    .max(500, 'Description must be less than 500 characters')
    .optional(),
  tags: z
    .array(z.string().max(30, 'Tag is too long'))
    .max(10, 'Maximum 10 tags allowed')
    .optional(),
  expiresAt: z
    .string()
    .datetime()
    .optional()
    .refine(
      (date) => !date || new Date(date) > new Date(),
      'Expiry date must be in the future'
    ),
});

export const userUpdateSchema = z.object({
  firstName: z
    .string()
    .min(2, 'First name must be at least 2 characters')
    .optional(),
  lastName: z
    .string()
    .min(2, 'Last name must be at least 2 characters')
    .optional(),
  email: z
    .string()
    .email('Invalid email format')
    .optional(),
  currentPassword: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .optional(),
  newPassword: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .optional(),
}).refine((data) => {
  if (data.newPassword && !data.currentPassword) {
    return false;
  }
  return true;
}, {
  message: 'Current password is required when setting new password',
  path: ['currentPassword'],
});
```

### 3. Improve Error Handling

Update `src/middleware/error.middleware.ts`:
```typescript
import { Request, Response, NextFunction } from 'express';
import { ZodError } from 'zod';
import { MongoError } from 'mongodb';
import logger from '../utils/logger';

export class ApiError extends Error {
  constructor(
    public statusCode: number,
    public message: string,
    public source?: string,
    public errors?: any[]
  ) {
    super(message);
    this.name = 'ApiError';
  }
}

export const errorHandler = (
  error: Error,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  logger.error('Error:', {
    error: error.message,
    stack: error.stack,
    path: req.path,
    method: req.method,
  });

  // Handle Zod validation errors
  if (error instanceof ZodError) {
    return res.status(400).json({
      status: 'error',
      message: 'Validation failed',
      errors: error.errors,
    });
  }

  // Handle MongoDB duplicate key errors
  if (error instanceof MongoError && error.code === 11000) {
    return res.status(409).json({
      status: 'error',
      message: 'Duplicate entry',
    });
  }

  // Handle custom API errors
  if (error instanceof ApiError) {
    return res.status(error.statusCode).json({
      status: 'error',
      message: error.message,
      errors: error.errors,
      source: error.source,
    });
  }

  // Handle all other errors
  return res.status(500).json({
    status: 'error',
    message: 'Internal server error',
  });
};
```

### 4. Add Loading States

Create `src/components/common/LoadingSpinner.tsx`:
```typescript
import React from 'react';
import { cn } from '@/utils/helpers';

interface LoadingSpinnerProps {
  size?: 'sm' | 'md' | 'lg';
  className?: string;
}

export const LoadingSpinner: React.FC<LoadingSpinnerProps> = ({
  size = 'md',
  className,
}) => {
  const sizeClasses = {
    sm: 'w-4 h-4',
    md: 'w-8 h-8',
    lg: 'w-12 h-12',
  };

  return (
    <div
      className={cn(
        'animate-spin rounded-full border-2 border-primary border-t-transparent',
        sizeClasses[size],
        className
      )}
    />
  );
};
```

Create `src/components/common/LoadingOverlay.tsx`:
```typescript
import React from 'react';
import { LoadingSpinner } from './LoadingSpinner';

interface LoadingOverlayProps {
  message?: string;
}

export const LoadingOverlay: React.FC<LoadingOverlayProps> = ({
  message = 'Loading...',
}) => {
  return (
    <div className="fixed inset-0 bg-black/50 flex items-center justify-center z-50">
      <div className="bg-white rounded-lg p-8 flex flex-col items-center">
        <LoadingSpinner size="lg" />
        <p className="mt-4 text-gray-600">{message}</p>
      </div>
    </div>
  );
};
```

### 5. Implement Toast Notifications

Create `src/components/common/Toast.tsx`:
```typescript
import React from 'react';
import { Transition } from '@headlessui/react';
import { XMarkIcon } from '@heroicons/react/24/outline';
import { cn } from '@/utils/helpers';

interface ToastProps {
  message: string;
  type: 'success' | 'error' | 'info';
  onClose: () => void;
}

export const Toast: React.FC<ToastProps> = ({
  message,
  type,
  onClose,
}) => {
  const [show, setShow] = React.useState(true);

  React.useEffect(() => {
    const timer = setTimeout(() => {
      setShow(false);
      setTimeout(onClose, 300);
    }, 5000);

    return () => clearTimeout(timer);
  }, [onClose]);

  return (
    <Transition
      show={show}
      enter="transition ease-out duration-300"
      enterFrom="transform opacity-0 scale-95"
      enterTo="transform opacity-100 scale-100"
      leave="transition ease-in duration-300"
      leaveFrom="transform opacity-100 scale-100"
      leaveTo="transform opacity-0 scale-95"
    >
      <div
        className={cn(
          'rounded-lg p-4 flex items-center shadow-lg',
          {
            'bg-green-50 text-green-800': type === 'success',
            'bg-red-50 text-red-800': type === 'error',
            'bg-blue-50 text-blue-800': type === 'info',
          }
        )}
      >
        <p className="flex-1">{message}</p>
        <button
          onClick={() => setShow(false)}
          className="ml-4 text-gray-400 hover:text-gray-500"
        >
          <XMarkIcon className="w-5 h-5" />
        </button>
      </div>
    </Transition>
  );
};
```

Create `src/hooks/useToast.ts`:
```typescript
import { create } from 'zustand';

interface Toast {
  id: string;
  message: string;
  type: 'success' | 'error' | 'info';
}

interface ToastStore {
  toasts: Toast[];
  addToast: (message: string, type: Toast['type']) => void;
  removeToast: (id: string) => void;
}

export const useToastStore = create<ToastStore>((set) => ({
  toasts: [],
  addToast: (message, type) =>
    set((state) => ({
      toasts: [
        ...state.toasts,
        { id: Date.now().toString(), message, type },
      ],
    })),
  removeToast: (id) =>
    set((state) => ({
      toasts: state.toasts.filter((toast) => toast.id !== id),
    })),
}));

export const useToast = () => {
  const { addToast } = useToastStore();

  return {
    showToast: (message: string, type: Toast['type'] = 'info') =>
      addToast(message, type),
  };
};
```

### 6. Add Form Feedback

Create `src/components/common/FormError.tsx`:
```typescript
import React from 'react';
import { ExclamationCircleIcon } from '@heroicons/react/24/solid';

interface FormErrorProps {
  message: string;
}

export const FormError: React.FC<FormErrorProps> = ({ message }) => {
  return (
    <div className="flex items-center mt-1 text-sm text-red-600">
      <ExclamationCircleIcon className="w-4 h-4 mr-1" />
      <span>{message}</span>
    </div>
  );
};
```

### 7. Implement Optimistic Updates

Update `src/hooks/useUrls.ts`:
```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { urlApi } from '@/services/api';
import { useToast } from './useToast';

export const useUrls = () => {
  const queryClient = useQueryClient();
  const { showToast } = useToast();

  const createUrl = useMutation({
    mutationFn: urlApi.create,
    onMutate: async (newUrl) => {
      await queryClient.cancelQueries(['urls']);

      const previousUrls = queryClient.getQueryData(['urls']);

      queryClient.setQueryData(['urls'], (old: any) => ({
        ...old,
        data: [
          { id: 'temp', ...newUrl, createdAt: new Date().toISOString() },
          ...old.data,
        ],
      }));

      return { previousUrls };
    },
    onError: (err, newUrl, context) => {
      queryClient.setQueryData(['urls'], context?.previousUrls);
      showToast('Failed to create URL', 'error');
    },
    onSuccess: () => {
      showToast('URL created successfully', 'success');
    },
    onSettled: () => {
      queryClient.invalidateQueries(['urls']);
    },
  });

  return { createUrl };
};
```

## Testing Polish Features

Create `src/__tests__/polish/validation.test.ts`:
```typescript
import { urlSchema, userUpdateSchema } from '@/lib/validations';

describe('URL Validation', () => {
  it('validates correct URL data', () => {
    const data = {
      originalUrl: 'https://example.com',
      title: 'Example Website',
      description: 'A great website',
      tags: ['example', 'website'],
    };

    const result = urlSchema.safeParse(data);
    expect(result.success).toBe(true);
  });

  it('rejects invalid URL data', () => {
    const data = {
      originalUrl: 'not-a-url',
      title: 'Ex',
      tags: Array(11).fill('tag'),
    };

    const result = urlSchema.safeParse(data);
    expect(result.success).toBe(false);
  });
});
```

## Best Practices

1. Rate Limiting:
   - Use appropriate limits
   - Implement per-route limits
   - Handle limit errors gracefully
   - Use distributed rate limiting

2. Input Validation:
   - Validate all inputs
   - Provide clear error messages
   - Implement client-side validation
   - Use strong validation schemas

3. Error Handling:
   - Use consistent error format
   - Log errors appropriately
   - Handle all error types
   - Provide helpful messages

4. User Experience:
   - Show loading states
   - Provide feedback
   - Implement smooth transitions
   - Handle edge cases

## Common Issues and Solutions

1. Rate Limiting:
   - Handle distributed systems
   - Implement proper storage
   - Account for proxies
   - Handle burst traffic

2. Form Validation:
   - Handle async validation
   - Validate file uploads
   - Implement CSRF protection
   - Handle special characters

3. Error Handling:
   - Handle network errors
   - Implement retry logic
   - Log sensitive data safely
   - Handle concurrent errors

## Next Steps

After completing this module, you should have:
- [ ] Implemented rate limiting
- [ ] Enhanced input validation
- [ ] Improved error handling
- [ ] Added loading states
- [ ] Implemented toast notifications
- [ ] Added form feedback
- [ ] Created optimistic updates

Ready to proceed to Module 15: Deployment.

## Additional Resources

- [Express Rate Limit](https://github.com/nfriedly/express-rate-limit)
- [Zod Documentation](https://github.com/colinhacks/zod)
- [React Query Optimistic Updates](https://tanstack.com/query/latest/docs/react/guides/optimistic-updates)
- [UX Best Practices](https://www.smashingmagazine.com/2020/02/design-better-forms/) 