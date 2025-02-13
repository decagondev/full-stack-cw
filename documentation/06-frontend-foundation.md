# Module 6: Frontend Foundation

## Overview
In this module, we'll set up the core frontend structure using React, TypeScript, and Tailwind CSS. We'll implement the basic layout, routing system, state management, and API service structure.

## Prerequisites
- Completed Modules 0-5
- Node.js and npm installed
- Basic understanding of React and TypeScript
- Familiarity with Tailwind CSS

## Implementation Steps

### 1. Set Up Project Structure

Create the following directory structure:
```
src/
├── components/
│   ├── common/
│   │   ├── Button.tsx
│   │   ├── Input.tsx
│   │   └── Card.tsx
│   ├── layout/
│   │   ├── MainLayout.tsx
│   │   ├── Navbar.tsx
│   │   └── Footer.tsx
│   └── ui/
│       ├── Alert.tsx
│       ├── Modal.tsx
│       └── Toast.tsx
├── hooks/
│   ├── useAuth.ts
│   └── useToast.ts
├── lib/
│   └── axios.ts
├── pages/
│   ├── auth/
│   │   ├── Login.tsx
│   │   └── Register.tsx
│   ├── dashboard/
│   │   └── Dashboard.tsx
│   ├── Home.tsx
│   └── NotFound.tsx
├── services/
│   └── api.ts
├── store/
│   └── auth.ts
├── types/
│   └── index.ts
└── utils/
    ├── constants.ts
    └── helpers.ts
```

### 2. Configure Base Components

Create `src/components/common/Button.tsx`:
```typescript
import React from 'react';
import { cva, type VariantProps } from 'class-variance-authority';
import { cn } from '@/utils/helpers';

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50',
  {
    variants: {
      variant: {
        default: 'bg-primary text-primary-foreground hover:bg-primary/90',
        destructive: 'bg-destructive text-destructive-foreground hover:bg-destructive/90',
        outline: 'border border-input bg-background hover:bg-accent hover:text-accent-foreground',
        secondary: 'bg-secondary text-secondary-foreground hover:bg-secondary/80',
        ghost: 'hover:bg-accent hover:text-accent-foreground',
        link: 'text-primary underline-offset-4 hover:underline',
      },
      size: {
        default: 'h-10 px-4 py-2',
        sm: 'h-9 rounded-md px-3',
        lg: 'h-11 rounded-md px-8',
        icon: 'h-10 w-10',
      },
    },
    defaultVariants: {
      variant: 'default',
      size: 'default',
    },
  }
);

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  isLoading?: boolean;
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, isLoading, children, ...props }, ref) => {
    return (
      <button
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        disabled={isLoading}
        {...props}
      >
        {isLoading ? (
          <span className="mr-2 h-4 w-4 animate-spin">⚪</span>
        ) : null}
        {children}
      </button>
    );
  }
);

Button.displayName = 'Button';

export { Button, buttonVariants };
```

### 3. Implement Layout Components

Create `src/components/layout/MainLayout.tsx`:
```typescript
import React from 'react';
import { Outlet } from 'react-router-dom';
import { Navbar } from './Navbar';
import { Footer } from './Footer';
import { Toast } from '../ui/Toast';

export const MainLayout: React.FC = () => {
  return (
    <div className="min-h-screen bg-gray-50">
      <Navbar />
      <main className="container mx-auto px-4 py-8">
        <Outlet />
      </main>
      <Footer />
      <Toast />
    </div>
  );
};
```

Create `src/components/layout/Navbar.tsx`:
```typescript
import React from 'react';
import { Link } from 'react-router-dom';
import { useAuth } from '@/hooks/useAuth';
import { Button } from '../common/Button';

export const Navbar: React.FC = () => {
  const { user, logout } = useAuth();

  return (
    <nav className="bg-white shadow">
      <div className="container mx-auto px-4">
        <div className="flex h-16 justify-between items-center">
          <Link to="/" className="text-xl font-bold">
            URL Redirector
          </Link>

          <div className="flex items-center gap-4">
            {user ? (
              <>
                <Link to="/dashboard">
                  <Button variant="ghost">Dashboard</Button>
                </Link>
                <Button variant="outline" onClick={logout}>
                  Logout
                </Button>
              </>
            ) : (
              <>
                <Link to="/auth/login">
                  <Button variant="ghost">Login</Button>
                </Link>
                <Link to="/auth/register">
                  <Button>Register</Button>
                </Link>
              </>
            )}
          </div>
        </div>
      </div>
    </nav>
  );
};
```

### 4. Set Up Routing

Create `src/App.tsx`:
```typescript
import { BrowserRouter, Routes, Route } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { MainLayout } from './components/layout/MainLayout';
import { Home } from './pages/Home';
import { Login } from './pages/auth/Login';
import { Register } from './pages/auth/Register';
import { Dashboard } from './pages/dashboard/Dashboard';
import { NotFound } from './pages/NotFound';
import { ProtectedRoute } from './components/common/ProtectedRoute';

const queryClient = new QueryClient();

function App() {
  return (
    <QueryClientProvider client={queryClient}>
      <BrowserRouter>
        <Routes>
          <Route element={<MainLayout />}>
            <Route index element={<Home />} />
            <Route path="auth">
              <Route path="login" element={<Login />} />
              <Route path="register" element={<Register />} />
            </Route>
            <Route
              path="dashboard"
              element={
                <ProtectedRoute>
                  <Dashboard />
                </ProtectedRoute>
              }
            />
            <Route path="*" element={<NotFound />} />
          </Route>
        </Routes>
      </BrowserRouter>
    </QueryClientProvider>
  );
}

export default App;
```

### 5. Configure API Service

Create `src/lib/axios.ts`:
```typescript
import axios from 'axios';
import { environment } from '@/utils/constants';

const api = axios.create({
  baseURL: environment.apiUrl,
  headers: {
    'Content-Type': 'application/json',
  },
});

api.interceptors.request.use((config) => {
  const token = localStorage.getItem('token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

api.interceptors.response.use(
  (response) => response,
  async (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('token');
      window.location.href = '/auth/login';
    }
    return Promise.reject(error);
  }
);

export default api;
```

Create `src/services/api.ts`:
```typescript
import api from '@/lib/axios';

export interface LoginCredentials {
  email: string;
  password: string;
}

export interface RegisterData extends LoginCredentials {
  firstName: string;
  lastName: string;
}

export const authApi = {
  login: (credentials: LoginCredentials) =>
    api.post('/auth/login', credentials),
  
  register: (data: RegisterData) =>
    api.post('/auth/register', data),
  
  logout: () =>
    api.post('/auth/logout'),
};
```

### 6. Implement State Management

Create `src/store/auth.ts`:
```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface User {
  id: string;
  email: string;
  firstName: string;
  lastName: string;
  role: string;
}

interface AuthState {
  user: User | null;
  token: string | null;
  setAuth: (user: User, token: string) => void;
  clearAuth: () => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      user: null,
      token: null,
      setAuth: (user, token) => set({ user, token }),
      clearAuth: () => set({ user: null, token: null }),
    }),
    {
      name: 'auth-storage',
    }
  )
);
```

### 7. Create Custom Hooks

Create `src/hooks/useAuth.ts`:
```typescript
import { useNavigate } from 'react-router-dom';
import { useAuthStore } from '@/store/auth';
import { authApi, LoginCredentials, RegisterData } from '@/services/api';
import { useToast } from './useToast';

export const useAuth = () => {
  const navigate = useNavigate();
  const { showToast } = useToast();
  const { user, token, setAuth, clearAuth } = useAuthStore();

  const login = async (credentials: LoginCredentials) => {
    try {
      const response = await authApi.login(credentials);
      const { user, token } = response.data.data;
      setAuth(user, token);
      showToast('Successfully logged in', 'success');
      navigate('/dashboard');
    } catch (error) {
      showToast('Invalid credentials', 'error');
      throw error;
    }
  };

  const register = async (data: RegisterData) => {
    try {
      const response = await authApi.register(data);
      const { user, token } = response.data.data;
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
    token,
    login,
    register,
    logout,
    isAuthenticated: !!token,
  };
};
```

## Testing the Implementation

1. Test the routing system:
```typescript
import { render, screen } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import App from './App';

describe('App', () => {
  it('renders home page by default', () => {
    render(
      <BrowserRouter>
        <App />
      </BrowserRouter>
    );
    expect(screen.getByText(/URL Redirector/i)).toBeInTheDocument();
  });
});
```

2. Test protected routes:
```typescript
import { render, screen } from '@testing-library/react';
import { MemoryRouter } from 'react-router-dom';
import { useAuth } from '@/hooks/useAuth';
import { Dashboard } from './pages/dashboard/Dashboard';

jest.mock('@/hooks/useAuth');

describe('Protected Routes', () => {
  it('redirects to login when accessing protected route without auth', () => {
    (useAuth as jest.Mock).mockReturnValue({ isAuthenticated: false });
    
    render(
      <MemoryRouter initialEntries={['/dashboard']}>
        <Dashboard />
      </MemoryRouter>
    );

    expect(screen.getByText(/please login/i)).toBeInTheDocument();
  });
});
```

## Best Practices

1. Component Organization:
   - Use atomic design principles
   - Keep components small and focused
   - Implement proper prop typing
   - Use composition over inheritance

2. State Management:
   - Centralize authentication state
   - Use local state when appropriate
   - Implement proper error handling
   - Cache API responses

3. Routing:
   - Implement lazy loading
   - Use proper route protection
   - Handle 404 cases
   - Maintain clean URLs

4. API Integration:
   - Centralize API calls
   - Handle errors consistently
   - Implement proper loading states
   - Use proper type definitions

## Common Issues and Solutions

1. Routing Issues:
   - Check route definitions
   - Verify navigation guards
   - Test nested routes
   - Handle loading states

2. State Management Problems:
   - Review state updates
   - Check persistence logic
   - Verify state synchronization
   - Debug state mutations

3. Component Problems:
   - Check prop types
   - Verify event handlers
   - Test component rendering
   - Debug style issues

## Next Steps

After completing this module, you should have:
- [ ] Set up project structure
- [ ] Implemented base components
- [ ] Created layout system
- [ ] Configured routing
- [ ] Set up API service
- [ ] Implemented state management
- [ ] Created custom hooks

Ready to proceed to Module 7: Frontend Authentication.

## Additional Resources

- [React Router Documentation](https://reactrouter.com/)
- [TanStack Query Documentation](https://tanstack.com/query/latest)
- [Zustand Documentation](https://github.com/pmndrs/zustand)
- [Tailwind CSS Documentation](https://tailwindcss.com/)
- [React TypeScript Cheatsheet](https://react-typescript-cheatsheet.netlify.app/) 