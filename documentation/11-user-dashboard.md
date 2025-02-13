# Module 11: User Dashboard

## Overview
In this module, we'll implement the user dashboard interface, allowing users to manage their personal URL redirects, view statistics, and monitor their account activity. We'll create a clean and intuitive interface focused on URL management and analytics.

## Prerequisites
- Completed Modules 0-10
- Understanding of React and TypeScript
- Knowledge of data visualization
- Familiarity with form handling

## Implementation Steps

### 1. Create User Dashboard Layout

Create `src/components/layout/UserLayout.tsx`:
```typescript
import React from 'react';
import { Outlet } from 'react-router-dom';
import { UserHeader } from './UserHeader';
import { UserSidebar } from './UserSidebar';

export const UserLayout: React.FC = () => {
  return (
    <div className="min-h-screen bg-gray-50">
      <UserHeader />
      <div className="flex">
        <UserSidebar />
        <main className="flex-1 p-6">
          <Outlet />
        </main>
      </div>
    </div>
  );
};
```

Create `src/components/layout/UserSidebar.tsx`:
```typescript
import React from 'react';
import { NavLink } from 'react-router-dom';
import { 
  Home,
  Link as LinkIcon,
  BarChart2,
  Settings,
  History
} from 'lucide-react';

const navItems = [
  {
    label: 'Dashboard',
    path: '/dashboard',
    icon: Home,
  },
  {
    label: 'My URLs',
    path: '/dashboard/urls',
    icon: LinkIcon,
  },
  {
    label: 'Analytics',
    path: '/dashboard/analytics',
    icon: BarChart2,
  },
  {
    label: 'History',
    path: '/dashboard/history',
    icon: History,
  },
  {
    label: 'Settings',
    path: '/dashboard/settings',
    icon: Settings,
  },
];

export const UserSidebar: React.FC = () => {
  return (
    <aside className="w-64 min-h-screen bg-white shadow-sm">
      <nav className="p-4 space-y-2">
        {navItems.map((item) => (
          <NavLink
            key={item.path}
            to={item.path}
            className={({ isActive }) =>
              `flex items-center space-x-3 px-4 py-2 rounded-lg transition-colors ${
                isActive
                  ? 'bg-primary/10 text-primary'
                  : 'text-gray-600 hover:bg-gray-50'
              }`
            }
          >
            <item.icon className="w-5 h-5" />
            <span>{item.label}</span>
          </NavLink>
        ))}
      </nav>
    </aside>
  );
};
```

### 2. Create URL Management Components

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

interface UrlFormData {
  originalUrl: string;
  title: string;
  description?: string;
  tags?: string[];
  expiresAt?: string;
}

export const UrlForm: React.FC<{
  onSubmit: (data: UrlFormData) => Promise<void>;
  initialData?: UrlFormData;
}> = ({ onSubmit, initialData }) => {
  const {
    register,
    handleSubmit,
    control,
    formState: { errors, isSubmitting },
  } = useForm<UrlFormData>({
    resolver: zodResolver(urlSchema),
    defaultValues: initialData,
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)} className="space-y-4">
      <Input
        label="Original URL"
        {...register('originalUrl')}
        error={errors.originalUrl?.message}
        placeholder="https://example.com/long-url"
      />
      <Input
        label="Title"
        {...register('title')}
        error={errors.title?.message}
        placeholder="My Awesome Link"
      />
      <Textarea
        label="Description"
        {...register('description')}
        error={errors.description?.message}
        placeholder="Optional description for your URL"
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
        isLoading={isSubmitting}
      >
        {initialData ? 'Update URL' : 'Create URL'}
      </Button>
    </form>
  );
};
```

Create `src/components/url/UrlList.tsx`:
```typescript
import React from 'react';
import { useQuery } from '@tanstack/react-query';
import { urlApi } from '@/services/api';
import { Button } from '../common/Button';
import { Card } from '../common/Card';
import { Badge } from '../common/Badge';
import { formatDistance } from 'date-fns';

export const UrlList: React.FC = () => {
  const [page, setPage] = React.useState(1);
  const { data, isLoading } = useQuery(
    ['urls', page],
    () => urlApi.list({ page })
  );

  if (isLoading) return <div>Loading...</div>;

  return (
    <div className="space-y-4">
      {data?.data.map((url) => (
        <Card key={url.id}>
          <div className="p-4">
            <div className="flex justify-between items-start">
              <div>
                <h3 className="text-lg font-semibold text-gray-900">
                  {url.title}
                </h3>
                <p className="text-sm text-gray-500 mt-1">
                  {url.description}
                </p>
              </div>
              <div className="flex space-x-2">
                <Button
                  variant="outline"
                  size="sm"
                  onClick={() => navigator.clipboard.writeText(url.shortUrl)}
                >
                  Copy
                </Button>
                <Button
                  variant="ghost"
                  size="sm"
                  onClick={() => onEdit(url)}
                >
                  Edit
                </Button>
              </div>
            </div>
            <div className="mt-4">
              <div className="text-sm">
                <span className="text-gray-500">Original URL: </span>
                <a
                  href={url.originalUrl}
                  target="_blank"
                  rel="noopener noreferrer"
                  className="text-primary hover:underline"
                >
                  {url.originalUrl}
                </a>
              </div>
              <div className="text-sm mt-1">
                <span className="text-gray-500">Short URL: </span>
                <a
                  href={url.shortUrl}
                  target="_blank"
                  rel="noopener noreferrer"
                  className="text-primary hover:underline"
                >
                  {url.shortUrl}
                </a>
              </div>
            </div>
            <div className="mt-4 flex items-center justify-between">
              <div className="flex space-x-2">
                {url.tags.map((tag) => (
                  <Badge key={tag} variant="secondary">
                    {tag}
                  </Badge>
                ))}
              </div>
              <div className="text-sm text-gray-500">
                Created {formatDistance(new Date(url.createdAt), new Date(), { addSuffix: true })}
              </div>
            </div>
          </div>
        </Card>
      ))}
      <div className="flex justify-between items-center mt-6">
        <div className="text-sm text-gray-500">
          Showing {data?.pagination.page} of {data?.pagination.pages} pages
        </div>
        <div className="flex space-x-2">
          <Button
            variant="outline"
            disabled={page === 1}
            onClick={() => setPage(page - 1)}
          >
            Previous
          </Button>
          <Button
            variant="outline"
            disabled={page === data?.pagination.pages}
            onClick={() => setPage(page + 1)}
          >
            Next
          </Button>
        </div>
      </div>
    </div>
  );
};
```

### 3. Create Analytics Components

Create `src/components/analytics/UrlStats.tsx`:
```typescript
import React from 'react';
import { useQuery } from '@tanstack/react-query';
import { urlApi } from '@/services/api';
import { Card } from '../common/Card';
import {
  BarChart,
  Bar,
  XAxis,
  YAxis,
  Tooltip,
  ResponsiveContainer,
} from 'recharts';

export const UrlStats: React.FC = () => {
  const { data, isLoading } = useQuery(['urlStats'], () =>
    urlApi.getStats()
  );

  if (isLoading) return <div>Loading...</div>;

  return (
    <div className="space-y-6">
      <div className="grid grid-cols-1 md:grid-cols-3 gap-6">
        <Card>
          <div className="p-6">
            <h3 className="text-lg font-semibold text-gray-700">
              Total URLs
            </h3>
            <p className="text-3xl font-bold mt-2">
              {data.totalUrls}
            </p>
          </div>
        </Card>
        <Card>
          <div className="p-6">
            <h3 className="text-lg font-semibold text-gray-700">
              Total Clicks
            </h3>
            <p className="text-3xl font-bold mt-2">
              {data.totalClicks}
            </p>
          </div>
        </Card>
        <Card>
          <div className="p-6">
            <h3 className="text-lg font-semibold text-gray-700">
              Active URLs
            </h3>
            <p className="text-3xl font-bold mt-2">
              {data.activeUrls}
            </p>
          </div>
        </Card>
      </div>
      <Card>
        <div className="p-6">
          <h3 className="text-lg font-semibold text-gray-700">
            Clicks Over Time
          </h3>
          <div className="h-64 mt-4">
            <ResponsiveContainer width="100%" height="100%">
              <BarChart data={data.clicksOverTime}>
                <XAxis dataKey="date" />
                <YAxis />
                <Tooltip />
                <Bar dataKey="clicks" fill="#4f46e5" />
              </BarChart>
            </ResponsiveContainer>
          </div>
        </div>
      </Card>
    </div>
  );
};
```

### 4. Create Dashboard Pages

Create `src/pages/dashboard/Overview.tsx`:
```typescript
import React from 'react';
import { UrlStats } from '@/components/analytics/UrlStats';
import { RecentUrls } from '@/components/url/RecentUrls';

export const Overview: React.FC = () => {
  return (
    <div className="space-y-6">
      <h1 className="text-2xl font-bold text-gray-900">Dashboard</h1>
      <UrlStats />
      <div className="mt-8">
        <h2 className="text-xl font-semibold text-gray-900 mb-4">
          Recent URLs
        </h2>
        <RecentUrls />
      </div>
    </div>
  );
};
```

Create `src/pages/dashboard/Urls.tsx`:
```typescript
import React from 'react';
import { UrlList } from '@/components/url/UrlList';
import { Button } from '@/components/common/Button';
import { CreateUrlModal } from '@/components/url/CreateUrlModal';

export const Urls: React.FC = () => {
  const [showCreateModal, setShowCreateModal] = React.useState(false);

  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <h1 className="text-2xl font-bold text-gray-900">My URLs</h1>
        <Button onClick={() => setShowCreateModal(true)}>
          Create New URL
        </Button>
      </div>
      <UrlList />
      {showCreateModal && (
        <CreateUrlModal onClose={() => setShowCreateModal(false)} />
      )}
    </div>
  );
};
```

### 5. Update Routes

Update `src/App.tsx`:
```typescript
import { UserLayout } from './components/layout/UserLayout';
import { Overview } from './pages/dashboard/Overview';
import { Urls } from './pages/dashboard/Urls';
import { Analytics } from './pages/dashboard/Analytics';
import { Settings } from './pages/dashboard/Settings';

function App() {
  return (
    <Routes>
      {/* ... existing routes ... */}
      <Route
        path="/dashboard"
        element={
          <ProtectedRoute>
            <UserLayout />
          </ProtectedRoute>
        }
      >
        <Route index element={<Overview />} />
        <Route path="urls" element={<Urls />} />
        <Route path="analytics" element={<Analytics />} />
        <Route path="settings" element={<Settings />} />
      </Route>
    </Routes>
  );
}
```

## Testing User Dashboard

Create `src/__tests__/dashboard/Overview.test.tsx`:
```typescript
import { render, screen } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { Overview } from '@/pages/dashboard/Overview';

const queryClient = new QueryClient();

describe('Dashboard Overview', () => {
  it('renders dashboard components', () => {
    render(
      <QueryClientProvider client={queryClient}>
        <Overview />
      </QueryClientProvider>
    );

    expect(screen.getByText(/dashboard/i)).toBeInTheDocument();
    expect(screen.getByText(/total urls/i)).toBeInTheDocument();
    expect(screen.getByText(/recent urls/i)).toBeInTheDocument();
  });
});
```

## Best Practices

1. User Experience:
   - Implement clear navigation
   - Add loading states
   - Show success/error messages
   - Use intuitive layouts

2. Form Handling:
   - Validate inputs
   - Show validation errors
   - Handle submission states
   - Provide clear feedback

3. Data Management:
   - Cache API responses
   - Implement pagination
   - Handle loading states
   - Update data optimistically

4. Performance:
   - Lazy load components
   - Optimize API calls
   - Implement proper caching
   - Use proper indexing

## Common Issues and Solutions

1. Form Handling:
   - Validate URLs properly
   - Handle form submission errors
   - Manage loading states
   - Clear form after submission

2. Data Synchronization:
   - Update lists after creation
   - Handle optimistic updates
   - Manage cache invalidation
   - Handle concurrent updates

3. Analytics:
   - Handle large datasets
   - Implement data aggregation
   - Optimize chart rendering
   - Cache analytics data

## Next Steps

After completing this module, you should have:
- [ ] Created user dashboard layout
- [ ] Implemented URL management
- [ ] Added URL analytics
- [ ] Created URL forms
- [ ] Implemented pagination
- [ ] Added data visualization
- [ ] Created dashboard tests

Ready to proceed to Module 12: Staff Dashboard.

## Additional Resources

- [React Hook Form Documentation](https://react-hook-form.com/)
- [Recharts Documentation](https://recharts.org/)
- [TanStack Query Documentation](https://tanstack.com/query/latest)
- [Date-fns Documentation](https://date-fns.org/) 