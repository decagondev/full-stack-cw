# Module 10: Admin Dashboard

## Overview
In this module, we'll implement the admin dashboard frontend, providing a comprehensive interface for user management, statistics, and administrative tasks. We'll create reusable components, implement data visualization, and ensure a smooth user experience.

## Prerequisites
- Completed Modules 0-9
- Understanding of React and TypeScript
- Knowledge of data visualization libraries
- Familiarity with Tailwind CSS

## Implementation Steps

### 1. Create Admin Layout

Create `src/components/layout/AdminLayout.tsx`:
```typescript
import React from 'react';
import { Outlet } from 'react-router-dom';
import { AdminSidebar } from './AdminSidebar';
import { AdminHeader } from './AdminHeader';

export const AdminLayout: React.FC = () => {
  return (
    <div className="min-h-screen bg-gray-100">
      <AdminHeader />
      <div className="flex">
        <AdminSidebar />
        <main className="flex-1 p-8">
          <Outlet />
        </main>
      </div>
    </div>
  );
};
```

Create `src/components/layout/AdminSidebar.tsx`:
```typescript
import React from 'react';
import { NavLink } from 'react-router-dom';
import { 
  Users, 
  BarChart2, 
  Link as LinkIcon, 
  Settings,
  Shield 
} from 'lucide-react';

const navItems = [
  {
    label: 'Dashboard',
    path: '/admin',
    icon: BarChart2,
  },
  {
    label: 'Users',
    path: '/admin/users',
    icon: Users,
  },
  {
    label: 'URLs',
    path: '/admin/urls',
    icon: LinkIcon,
  },
  {
    label: 'Roles',
    path: '/admin/roles',
    icon: Shield,
  },
  {
    label: 'Settings',
    path: '/admin/settings',
    icon: Settings,
  },
];

export const AdminSidebar: React.FC = () => {
  return (
    <aside className="w-64 min-h-screen bg-white shadow-lg">
      <nav className="p-4 space-y-2">
        {navItems.map((item) => (
          <NavLink
            key={item.path}
            to={item.path}
            className={({ isActive }) =>
              `flex items-center space-x-3 px-4 py-3 rounded-lg transition-colors ${
                isActive
                  ? 'bg-primary text-white'
                  : 'text-gray-600 hover:bg-gray-100'
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

### 2. Create Dashboard Components

Create `src/components/admin/DashboardStats.tsx`:
```typescript
import React from 'react';
import { Card } from '../common/Card';
import { useQuery } from '@tanstack/react-query';
import { adminApi } from '@/services/api';
import { BarChart, Bar, XAxis, YAxis, Tooltip, ResponsiveContainer } from 'recharts';

export const DashboardStats: React.FC = () => {
  const { data: stats, isLoading } = useQuery(['adminStats'], () =>
    adminApi.getUserStats()
  );

  if (isLoading) {
    return <div>Loading...</div>;
  }

  const chartData = [
    { name: 'Active', value: stats.data.userStats.active },
    { name: 'Suspended', value: stats.data.userStats.suspended },
    { name: 'Deactivated', value: stats.data.userStats.deactivated },
  ];

  return (
    <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-6">
      <Card>
        <div className="p-6">
          <h3 className="text-lg font-semibold text-gray-700">Total Users</h3>
          <p className="text-3xl font-bold mt-2">{stats.data.userStats.total}</p>
        </div>
      </Card>
      <Card>
        <div className="p-6">
          <h3 className="text-lg font-semibold text-gray-700">Active Users</h3>
          <p className="text-3xl font-bold mt-2 text-green-600">
            {stats.data.userStats.active}
          </p>
        </div>
      </Card>
      <Card>
        <div className="p-6">
          <h3 className="text-lg font-semibold text-gray-700">User Status</h3>
          <div className="h-48 mt-4">
            <ResponsiveContainer width="100%" height="100%">
              <BarChart data={chartData}>
                <XAxis dataKey="name" />
                <YAxis />
                <Tooltip />
                <Bar dataKey="value" fill="#4f46e5" />
              </BarChart>
            </ResponsiveContainer>
          </div>
        </div>
      </Card>
    </div>
  );
};
```

### 3. Create User Management Components

Create `src/components/admin/UserTable.tsx`:
```typescript
import React from 'react';
import { useQuery } from '@tanstack/react-query';
import { adminApi } from '@/services/api';
import { Button } from '../common/Button';
import { Badge } from '../common/Badge';
import { UserRole, UserStatus } from '@/types';

export const UserTable: React.FC = () => {
  const [page, setPage] = React.useState(1);
  const [search, setSearch] = React.useState('');
  const [role, setRole] = React.useState<UserRole | ''>('');
  const [status, setStatus] = React.useState<UserStatus | ''>('');

  const { data, isLoading } = useQuery(
    ['users', page, search, role, status],
    () =>
      adminApi.listUsers({
        page,
        search,
        role,
        status,
      })
  );

  const handleDelete = async (userId: string) => {
    if (window.confirm('Are you sure you want to delete this user?')) {
      await adminApi.deleteUser(userId);
      // Refetch users
      queryClient.invalidateQueries(['users']);
    }
  };

  return (
    <div className="bg-white rounded-lg shadow">
      <div className="p-6 border-b border-gray-200">
        <div className="flex justify-between items-center">
          <h2 className="text-xl font-semibold text-gray-800">Users</h2>
          <Button onClick={() => setShowCreateModal(true)}>Add User</Button>
        </div>
        <div className="mt-4 flex gap-4">
          <input
            type="text"
            placeholder="Search users..."
            className="form-input"
            value={search}
            onChange={(e) => setSearch(e.target.value)}
          />
          <select
            className="form-select"
            value={role}
            onChange={(e) => setRole(e.target.value as UserRole)}
          >
            <option value="">All Roles</option>
            {Object.values(UserRole).map((r) => (
              <option key={r} value={r}>
                {r}
              </option>
            ))}
          </select>
          <select
            className="form-select"
            value={status}
            onChange={(e) => setStatus(e.target.value as UserStatus)}
          >
            <option value="">All Status</option>
            {Object.values(UserStatus).map((s) => (
              <option key={s} value={s}>
                {s}
              </option>
            ))}
          </select>
        </div>
      </div>
      <div className="overflow-x-auto">
        <table className="w-full">
          <thead className="bg-gray-50">
            <tr>
              <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                User
              </th>
              <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                Role
              </th>
              <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                Status
              </th>
              <th className="px-6 py-3 text-left text-xs font-medium text-gray-500 uppercase tracking-wider">
                Created
              </th>
              <th className="px-6 py-3 text-right text-xs font-medium text-gray-500 uppercase tracking-wider">
                Actions
              </th>
            </tr>
          </thead>
          <tbody className="bg-white divide-y divide-gray-200">
            {data?.data.map((user) => (
              <tr key={user.id}>
                <td className="px-6 py-4 whitespace-nowrap">
                  <div className="flex items-center">
                    <div>
                      <div className="text-sm font-medium text-gray-900">
                        {user.firstName} {user.lastName}
                      </div>
                      <div className="text-sm text-gray-500">{user.email}</div>
                    </div>
                  </div>
                </td>
                <td className="px-6 py-4 whitespace-nowrap">
                  <Badge variant={getRoleBadgeVariant(user.role)}>
                    {user.role}
                  </Badge>
                </td>
                <td className="px-6 py-4 whitespace-nowrap">
                  <Badge variant={getStatusBadgeVariant(user.status)}>
                    {user.status}
                  </Badge>
                </td>
                <td className="px-6 py-4 whitespace-nowrap text-sm text-gray-500">
                  {new Date(user.createdAt).toLocaleDateString()}
                </td>
                <td className="px-6 py-4 whitespace-nowrap text-right text-sm font-medium">
                  <Button
                    variant="ghost"
                    onClick={() => setEditingUser(user)}
                  >
                    Edit
                  </Button>
                  <Button
                    variant="destructive"
                    onClick={() => handleDelete(user.id)}
                  >
                    Delete
                  </Button>
                </td>
              </tr>
            ))}
          </tbody>
        </table>
      </div>
      <div className="px-6 py-4 border-t border-gray-200">
        <div className="flex justify-between items-center">
          <div className="text-sm text-gray-500">
            Showing {data?.pagination.page} of {data?.pagination.pages} pages
          </div>
          <div className="flex gap-2">
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
    </div>
  );
};
```

### 4. Create Admin Pages

Create `src/pages/admin/Dashboard.tsx`:
```typescript
import React from 'react';
import { DashboardStats } from '@/components/admin/DashboardStats';
import { RecentUsers } from '@/components/admin/RecentUsers';
import { RecentUrls } from '@/components/admin/RecentUrls';

export const Dashboard: React.FC = () => {
  return (
    <div className="space-y-6">
      <h1 className="text-2xl font-bold text-gray-900">Dashboard</h1>
      <DashboardStats />
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <RecentUsers />
        <RecentUrls />
      </div>
    </div>
  );
};
```

Create `src/pages/admin/Users.tsx`:
```typescript
import React from 'react';
import { UserTable } from '@/components/admin/UserTable';
import { CreateUserModal } from '@/components/admin/CreateUserModal';
import { EditUserModal } from '@/components/admin/EditUserModal';

export const Users: React.FC = () => {
  const [showCreateModal, setShowCreateModal] = React.useState(false);
  const [editingUser, setEditingUser] = React.useState<any>(null);

  return (
    <div className="space-y-6">
      <h1 className="text-2xl font-bold text-gray-900">User Management</h1>
      <UserTable
        onCreateUser={() => setShowCreateModal(true)}
        onEditUser={setEditingUser}
      />
      {showCreateModal && (
        <CreateUserModal onClose={() => setShowCreateModal(false)} />
      )}
      {editingUser && (
        <EditUserModal
          user={editingUser}
          onClose={() => setEditingUser(null)}
        />
      )}
    </div>
  );
};
```

### 5. Update Routes

Update `src/App.tsx`:
```typescript
import { AdminLayout } from './components/layout/AdminLayout';
import { Dashboard } from './pages/admin/Dashboard';
import { Users } from './pages/admin/Users';

// ... existing imports and code ...

function App() {
  return (
    <Routes>
      {/* ... existing routes ... */}
      <Route
        path="/admin"
        element={
          <ProtectedRoute requiredRole={UserRole.ADMIN}>
            <AdminLayout />
          </ProtectedRoute>
        }
      >
        <Route index element={<Dashboard />} />
        <Route path="users" element={<Users />} />
        {/* Add more admin routes as needed */}
      </Route>
    </Routes>
  );
}
```

### 6. Add Data Visualization

Install required packages:
```bash
npm install recharts @tanstack/react-table date-fns
```

Create `src/components/admin/charts/UserChart.tsx`:
```typescript
import React from 'react';
import {
  LineChart,
  Line,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  ResponsiveContainer,
} from 'recharts';
import { format } from 'date-fns';
import { useQuery } from '@tanstack/react-query';
import { adminApi } from '@/services/api';

export const UserChart: React.FC = () => {
  const { data, isLoading } = useQuery(['userGrowth'], () =>
    adminApi.getUserGrowth()
  );

  if (isLoading) return <div>Loading...</div>;

  return (
    <div className="h-96">
      <ResponsiveContainer width="100%" height="100%">
        <LineChart data={data}>
          <CartesianGrid strokeDasharray="3 3" />
          <XAxis
            dataKey="date"
            tickFormatter={(date) => format(new Date(date), 'MMM d')}
          />
          <YAxis />
          <Tooltip
            labelFormatter={(date) =>
              format(new Date(date), 'MMM d, yyyy')
            }
          />
          <Line
            type="monotone"
            dataKey="count"
            stroke="#4f46e5"
            strokeWidth={2}
          />
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
};
```

## Testing Admin Dashboard

Create `src/__tests__/admin/Dashboard.test.tsx`:
```typescript
import { render, screen } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { Dashboard } from '@/pages/admin/Dashboard';

const queryClient = new QueryClient();

describe('Admin Dashboard', () => {
  it('renders dashboard components', () => {
    render(
      <QueryClientProvider client={queryClient}>
        <Dashboard />
      </QueryClientProvider>
    );

    expect(screen.getByText(/dashboard/i)).toBeInTheDocument();
    expect(screen.getByText(/total users/i)).toBeInTheDocument();
    expect(screen.getByText(/recent activity/i)).toBeInTheDocument();
  });
});
```

## Best Practices

1. Component Organization:
   - Use atomic design principles
   - Implement proper layouts
   - Create reusable components
   - Maintain consistent styling

2. State Management:
   - Use React Query for server state
   - Implement proper caching
   - Handle loading states
   - Manage error boundaries

3. User Experience:
   - Add loading indicators
   - Implement error messages
   - Use proper validation
   - Add confirmation dialogs

4. Performance:
   - Implement pagination
   - Use proper caching
   - Optimize renders
   - Lazy load components

## Common Issues and Solutions

1. Layout Issues:
   - Use proper grid systems
   - Handle responsive design
   - Manage overflow content
   - Fix alignment problems

2. State Management:
   - Handle stale data
   - Manage cache invalidation
   - Sync multiple queries
   - Handle optimistic updates

3. Performance:
   - Optimize table rendering
   - Implement virtual scrolling
   - Reduce bundle size
   - Cache API responses

## Next Steps

After completing this module, you should have:
- [ ] Created admin layout
- [ ] Implemented dashboard components
- [ ] Added user management interface
- [ ] Created data visualization
- [ ] Implemented filtering and sorting
- [ ] Added pagination support
- [ ] Created admin tests

Ready to proceed to Module 11: User Dashboard.

## Additional Resources

- [Recharts Documentation](https://recharts.org/)
- [TanStack Table Documentation](https://tanstack.com/table/v8)
- [React Query Documentation](https://tanstack.com/query/latest)
- [Tailwind CSS Components](https://tailwindui.com/) 