# Module 7: Frontend Authentication

## Overview
In this module, we'll implement the frontend authentication system, including login and registration forms, protected routes, and authentication state management. We'll create reusable form components and implement proper validation and error handling.

## Prerequisites
- Completed Modules 0-6
- Understanding of React forms and validation
- Familiarity with React Hook Form and Zod
- Knowledge of JWT authentication

## Implementation Steps

### 1. Install Dependencies

```bash
npm install @hookform/resolvers react-hook-form zod
```

### 2. Create Form Validation Schemas

Create `src/lib/validations/auth.ts`:
```typescript
import * as z from 'zod';

export const loginSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
});

export const registerSchema = z.object({
  email: z.string().email('Invalid email address'),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .regex(
      /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
      'Password must contain at least one uppercase letter, one lowercase letter, and one number'
    ),
  confirmPassword: z.string(),
  firstName: z.string().min(2, 'First name must be at least 2 characters'),
  lastName: z.string().min(2, 'Last name must be at least 2 characters'),
}).refine((data) => data.password === data.confirmPassword, {
  message: "Passwords don't match",
  path: ['confirmPassword'],
});

export type LoginFormData = z.infer<typeof loginSchema>;
export type RegisterFormData = z.infer<typeof registerSchema>;
```

### 3. Create Authentication Forms

Create `src/components/auth/LoginForm.tsx`:
```typescript
import React from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { LoginFormData, loginSchema } from '@/lib/validations/auth';
import { useAuth } from '@/hooks/useAuth';
import { Button } from '../common/Button';
import { Input } from '../common/Input';
import { Card } from '../common/Card';

export const LoginForm: React.FC = () => {
  const { login } = useAuth();
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<LoginFormData>({
    resolver: zodResolver(loginSchema),
  });

  const onSubmit = async (data: LoginFormData) => {
    try {
      await login(data);
    } catch (error) {
      console.error('Login failed:', error);
    }
  };

  return (
    <Card className="w-full max-w-md mx-auto">
      <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
        <div>
          <Input
            label="Email"
            type="email"
            {...register('email')}
            error={errors.email?.message}
          />
        </div>
        <div>
          <Input
            label="Password"
            type="password"
            {...register('password')}
            error={errors.password?.message}
          />
        </div>
        <Button
          type="submit"
          className="w-full"
          isLoading={isSubmitting}
        >
          Log In
        </Button>
      </form>
    </Card>
  );
};
```

Create `src/components/auth/RegisterForm.tsx`:
```typescript
import React from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { RegisterFormData, registerSchema } from '@/lib/validations/auth';
import { useAuth } from '@/hooks/useAuth';
import { Button } from '../common/Button';
import { Input } from '../common/Input';
import { Card } from '../common/Card';

export const RegisterForm: React.FC = () => {
  const { register: registerUser } = useAuth();
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
  } = useForm<RegisterFormData>({
    resolver: zodResolver(registerSchema),
  });

  const onSubmit = async (data: RegisterFormData) => {
    try {
      await registerUser(data);
    } catch (error) {
      console.error('Registration failed:', error);
    }
  };

  return (
    <Card className="w-full max-w-md mx-auto">
      <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
        <div className="grid grid-cols-2 gap-4">
          <Input
            label="First Name"
            {...register('firstName')}
            error={errors.firstName?.message}
          />
          <Input
            label="Last Name"
            {...register('lastName')}
            error={errors.lastName?.message}
          />
        </div>
        <Input
          label="Email"
          type="email"
          {...register('email')}
          error={errors.email?.message}
        />
        <Input
          label="Password"
          type="password"
          {...register('password')}
          error={errors.password?.message}
        />
        <Input
          label="Confirm Password"
          type="password"
          {...register('confirmPassword')}
          error={errors.confirmPassword?.message}
        />
        <Button
          type="submit"
          className="w-full"
          isLoading={isSubmitting}
        >
          Register
        </Button>
      </form>
    </Card>
  );
};
```

### 4. Create Protected Route Component

Create `src/components/common/ProtectedRoute.tsx`:
```typescript
import React from 'react';
import { Navigate, useLocation } from 'react-router-dom';
import { useAuth } from '@/hooks/useAuth';

interface ProtectedRouteProps {
  children: React.ReactNode;
  requiredRole?: string;
}

export const ProtectedRoute: React.FC<ProtectedRouteProps> = ({
  children,
  requiredRole,
}) => {
  const { isAuthenticated, user } = useAuth();
  const location = useLocation();

  if (!isAuthenticated) {
    return <Navigate to="/auth/login" state={{ from: location }} replace />;
  }

  if (requiredRole && user?.role !== requiredRole) {
    return <Navigate to="/" replace />;
  }

  return <>{children}</>;
};
```

### 5. Create Authentication Pages

Create `src/pages/auth/Login.tsx`:
```typescript
import React from 'react';
import { Link } from 'react-router-dom';
import { LoginForm } from '@/components/auth/LoginForm';

export const Login: React.FC = () => {
  return (
    <div className="min-h-[80vh] flex flex-col items-center justify-center">
      <div className="w-full max-w-md">
        <h1 className="text-3xl font-bold text-center mb-8">
          Welcome Back
        </h1>
        <LoginForm />
        <p className="text-center mt-4">
          Don't have an account?{' '}
          <Link
            to="/auth/register"
            className="text-primary hover:underline"
          >
            Register here
          </Link>
        </p>
      </div>
    </div>
  );
};
```

Create `src/pages/auth/Register.tsx`:
```typescript
import React from 'react';
import { Link } from 'react-router-dom';
import { RegisterForm } from '@/components/auth/RegisterForm';

export const Register: React.FC = () => {
  return (
    <div className="min-h-[80vh] flex flex-col items-center justify-center">
      <div className="w-full max-w-md">
        <h1 className="text-3xl font-bold text-center mb-8">
          Create Account
        </h1>
        <RegisterForm />
        <p className="text-center mt-4">
          Already have an account?{' '}
          <Link
            to="/auth/login"
            className="text-primary hover:underline"
          >
            Login here
          </Link>
        </p>
      </div>
    </div>
  );
};
```

### 6. Implement Token Management

Create `src/utils/token.ts`:
```typescript
const TOKEN_KEY = 'auth_token';
const REFRESH_TOKEN_KEY = 'refresh_token';

export const tokenService = {
  getToken: () => localStorage.getItem(TOKEN_KEY),
  setToken: (token: string) => localStorage.setItem(TOKEN_KEY, token),
  removeToken: () => localStorage.removeItem(TOKEN_KEY),
  
  getRefreshToken: () => localStorage.getItem(REFRESH_TOKEN_KEY),
  setRefreshToken: (token: string) => localStorage.setItem(REFRESH_TOKEN_KEY, token),
  removeRefreshToken: () => localStorage.removeItem(REFRESH_TOKEN_KEY),
  
  clearTokens: () => {
    localStorage.removeItem(TOKEN_KEY);
    localStorage.removeItem(REFRESH_TOKEN_KEY);
  },
};
```

### 7. Update Auth Hook with Token Management

Update `src/hooks/useAuth.ts`:
```typescript
import { useNavigate } from 'react-router-dom';
import { useAuthStore } from '@/store/auth';
import { authApi } from '@/services/api';
import { useToast } from './useToast';
import { tokenService } from '@/utils/token';
import type { LoginFormData, RegisterFormData } from '@/lib/validations/auth';

export const useAuth = () => {
  const navigate = useNavigate();
  const { showToast } = useToast();
  const { user, setAuth, clearAuth } = useAuthStore();

  const login = async (credentials: LoginFormData) => {
    try {
      const response = await authApi.login(credentials);
      const { user, token, refreshToken } = response.data.data;
      
      tokenService.setToken(token);
      tokenService.setRefreshToken(refreshToken);
      setAuth(user, token);
      
      showToast('Successfully logged in', 'success');
      navigate('/dashboard');
    } catch (error) {
      showToast('Invalid credentials', 'error');
      throw error;
    }
  };

  const register = async (data: RegisterFormData) => {
    try {
      const response = await authApi.register(data);
      const { user, token, refreshToken } = response.data.data;
      
      tokenService.setToken(token);
      tokenService.setRefreshToken(refreshToken);
      setAuth(user, token);
      
      showToast('Successfully registered', 'success');
      navigate('/dashboard');
    } catch (error) {
      showToast('Registration failed', 'error');
      throw error;
    }
  };

  const logout = async () => {
    try {
      await authApi.logout();
      tokenService.clearTokens();
      clearAuth();
      showToast('Successfully logged out', 'success');
      navigate('/');
    } catch (error) {
      showToast('Logout failed', 'error');
      throw error;
    }
  };

  return {
    user,
    login,
    register,
    logout,
    isAuthenticated: !!tokenService.getToken(),
  };
};
```

## Testing Authentication

Create `src/__tests__/auth/LoginForm.test.tsx`:
```typescript
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { LoginForm } from '@/components/auth/LoginForm';
import { useAuth } from '@/hooks/useAuth';

jest.mock('@/hooks/useAuth');

describe('LoginForm', () => {
  it('validates required fields', async () => {
    render(<LoginForm />);
    
    fireEvent.click(screen.getByText(/log in/i));
    
    await waitFor(() => {
      expect(screen.getByText(/invalid email/i)).toBeInTheDocument();
      expect(screen.getByText(/password must be/i)).toBeInTheDocument();
    });
  });

  it('submits form with valid data', async () => {
    const mockLogin = jest.fn();
    (useAuth as jest.Mock).mockReturnValue({ login: mockLogin });
    
    render(<LoginForm />);
    
    fireEvent.change(screen.getByLabelText(/email/i), {
      target: { value: 'test@example.com' },
    });
    fireEvent.change(screen.getByLabelText(/password/i), {
      target: { value: 'Password123' },
    });
    
    fireEvent.click(screen.getByText(/log in/i));
    
    await waitFor(() => {
      expect(mockLogin).toHaveBeenCalledWith({
        email: 'test@example.com',
        password: 'Password123',
      });
    });
  });
});
```

## Best Practices

1. Form Handling:
   - Use form validation libraries
   - Implement proper error messages
   - Show loading states
   - Handle submission errors

2. Authentication Flow:
   - Implement token refresh
   - Secure token storage
   - Handle expired sessions
   - Implement proper redirects

3. User Experience:
   - Clear error messages
   - Loading indicators
   - Smooth transitions
   - Helpful validation feedback

4. Security:
   - Secure password handling
   - Protect sensitive routes
   - Implement CSRF protection
   - Handle token expiration

## Common Issues and Solutions

1. Form Validation:
   - Validate all inputs
   - Show clear error messages
   - Handle async validation
   - Prevent multiple submissions

2. Authentication State:
   - Handle token expiration
   - Manage refresh tokens
   - Sync auth state
   - Handle network errors

3. Route Protection:
   - Check auth status
   - Handle role-based access
   - Redirect unauthorized users
   - Preserve intended destination

## Next Steps

After completing this module, you should have:
- [ ] Implemented login form
- [ ] Created registration form
- [ ] Set up form validation
- [ ] Added protected routes
- [ ] Implemented token management
- [ ] Created auth state management
- [ ] Added error handling

Ready to proceed to Module 8: URL Redirect CRUD.

## Additional Resources

- [React Hook Form Documentation](https://react-hook-form.com/)
- [Zod Documentation](https://github.com/colinhacks/zod)
- [JWT Authentication Best Practices](https://auth0.com/blog/jwt-authentication-best-practices/)
- [React Router Authentication](https://reactrouter.com/docs/en/v6/examples/auth)
- [Web Storage Security](https://cheatsheetseries.owasp.org/cheatsheets/HTML5_Security_Cheat_Sheet.html#local-storage)