# Module 12: Staff Dashboard

## Overview
In this module, we'll implement the staff dashboard interface, allowing staff members to manage all users' URLs, perform bulk operations, and monitor system activity. We'll create advanced search functionality and implement audit logging for staff actions.

## Prerequisites
- Completed Modules 0-11
- Understanding of role-based access control
- Knowledge of bulk operations
- Familiarity with audit logging

## Implementation Steps

### 1. Create Staff Dashboard Layout

Create `src/components/layout/StaffLayout.tsx`:
```typescript
import React from 'react';
import { Outlet } from 'react-router-dom';
import { StaffHeader } from './StaffHeader';
import { StaffSidebar } from './StaffSidebar';

export const StaffLayout: React.FC = () => {
  return (
    <div className="min-h-screen bg-gray-50">
      <StaffHeader />
      <div className="flex">
        <StaffSidebar />
        <main className="flex-1 p-6">
          <Outlet />
        </main>
      </div>
    </div>
  );
};
```

Create `src/components/layout/StaffSidebar.tsx`:
```typescript
import React from 'react';
import { NavLink } from 'react-router-dom';
import { 
  LayoutDashboard,
  Link as LinkIcon,
  Users,
  History,
  AlertCircle,
  Settings
} from 'lucide-react';

const navItems = [
  {
    label: 'Overview',
    path: '/staff',
    icon: LayoutDashboard,
  },
  {
    label: 'All URLs',
    path: '/staff/urls',
    icon: LinkIcon,
  },
  {
    label: 'Users',
    path: '/staff/users',
    icon: Users,
  },
  {
    label: 'Audit Log',
    path: '/staff/audit',
    icon: History,
  },
  {
    label: 'Reports',
    path: '/staff/reports',
    icon: AlertCircle,
  },
  {
    label: 'Settings',
    path: '/staff/settings',
    icon: Settings,
  },
];

export const StaffSidebar: React.FC = () => {
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

### 2. Create Advanced URL Management Components

Create `src/components/staff/AdvancedUrlSearch.tsx`:
```typescript
import React from 'react';
import { useForm } from 'react-hook-form';
import { Input } from '../common/Input';
import { Button } from '../common/Button';
import { Select } from '../common/Select';
import { DateRangePicker } from '../common/DateRangePicker';

interface SearchFilters {
  query: string;
  status: string;
  user: string;
  dateRange: {
    from: Date;
    to: Date;
  };
  clickRange: {
    min: number;
    max: number;
  };
}

export const AdvancedUrlSearch: React.FC<{
  onSearch: (filters: SearchFilters) => void;
}> = ({ onSearch }) => {
  const { register, handleSubmit, control } = useForm<SearchFilters>();

  return (
    <form onSubmit={handleSubmit(onSearch)} className="space-y-4">
      <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-4">
        <Input
          label="Search"
          placeholder="Search URLs, titles, or tags..."
          {...register('query')}
        />
        <Select
          label="Status"
          {...register('status')}
        >
          <option value="">All Status</option>
          <option value="active">Active</option>
          <option value="expired">Expired</option>
          <option value="disabled">Disabled</option>
        </Select>
        <Input
          label="User"
          placeholder="Search by user..."
          {...register('user')}
        />
        <DateRangePicker
          label="Date Range"
          control={control}
          name="dateRange"
        />
        <div className="flex space-x-2">
          <Input
            type="number"
            label="Min Clicks"
            {...register('clickRange.min')}
          />
          <Input
            type="number"
            label="Max Clicks"
            {...register('clickRange.max')}
          />
        </div>
      </div>
      <div className="flex justify-end space-x-2">
        <Button type="reset" variant="outline">
          Reset
        </Button>
        <Button type="submit">
          Search
        </Button>
      </div>
    </form>
  );
};
```

Create `src/components/staff/BulkOperations.tsx`:
```typescript
import React from 'react';
import { Button } from '../common/Button';
import { Select } from '../common/Select';
import { useToast } from '@/hooks/useToast';
import { staffApi } from '@/services/api';

interface BulkOperationsProps {
  selectedUrls: string[];
  onComplete: () => void;
}

export const BulkOperations: React.FC<BulkOperationsProps> = ({
  selectedUrls,
  onComplete,
}) => {
  const [operation, setOperation] = React.useState('');
  const [isLoading, setIsLoading] = React.useState(false);
  const { showToast } = useToast();

  const handleOperation = async () => {
    if (!operation || selectedUrls.length === 0) return;

    setIsLoading(true);
    try {
      await staffApi.bulkOperation(selectedUrls, operation);
      showToast('Bulk operation completed successfully', 'success');
      onComplete();
    } catch (error) {
      showToast('Failed to perform bulk operation', 'error');
    } finally {
      setIsLoading(false);
    }
  };

  return (
    <div className="flex items-center space-x-2">
      <Select
        value={operation}
        onChange={(e) => setOperation(e.target.value)}
        disabled={selectedUrls.length === 0}
      >
        <option value="">Select Operation</option>
        <option value="enable">Enable</option>
        <option value="disable">Disable</option>
        <option value="delete">Delete</option>
        <option value="extend">Extend Expiry</option>
      </Select>
      <Button
        onClick={handleOperation}
        disabled={!operation || selectedUrls.length === 0}
        isLoading={isLoading}
      >
        Apply to {selectedUrls.length} URLs
      </Button>
    </div>
  );
};
```

### 3. Create Audit Logging Components

Create `src/components/staff/AuditLog.tsx`:
```typescript
import React from 'react';
import { useQuery } from '@tanstack/react-query';
import { staffApi } from '@/services/api';
import { Card } from '../common/Card';
import { Badge } from '../common/Badge';
import { formatDistance } from 'date-fns';

export const AuditLog: React.FC = () => {
  const { data, isLoading } = useQuery(['auditLog'], () =>
    staffApi.getAuditLog()
  );

  if (isLoading) return <div>Loading...</div>;

  return (
    <div className="space-y-4">
      {data?.logs.map((log) => (
        <Card key={log.id}>
          <div className="p-4">
            <div className="flex items-center justify-between">
              <div>
                <span className="font-medium text-gray-900">
                  {log.user.name}
                </span>
                <span className="text-gray-500 mx-2">
                  {log.action}
                </span>
                <span className="font-medium text-gray-900">
                  {log.target}
                </span>
              </div>
              <Badge variant={getActionBadgeVariant(log.action)}>
                {log.action}
              </Badge>
            </div>
            <div className="mt-2 text-sm text-gray-500">
              {log.details}
            </div>
            <div className="mt-2 text-xs text-gray-400">
              {formatDistance(new Date(log.timestamp), new Date(), {
                addSuffix: true,
              })}
            </div>
          </div>
        </Card>
      ))}
    </div>
  );
};
```

### 4. Create Staff Pages

Create `src/pages/staff/Overview.tsx`:
```typescript
import React from 'react';
import { useQuery } from '@tanstack/react-query';
import { staffApi } from '@/services/api';
import { Card } from '@/components/common/Card';
import { SystemStats } from '@/components/staff/SystemStats';
import { RecentActivity } from '@/components/staff/RecentActivity';

export const Overview: React.FC = () => {
  const { data: stats } = useQuery(['systemStats'], () =>
    staffApi.getSystemStats()
  );

  return (
    <div className="space-y-6">
      <h1 className="text-2xl font-bold text-gray-900">Staff Dashboard</h1>
      <SystemStats stats={stats} />
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        <Card>
          <div className="p-6">
            <h2 className="text-lg font-semibold text-gray-900">
              Recent Activity
            </h2>
            <RecentActivity />
          </div>
        </Card>
        <Card>
          <div className="p-6">
            <h2 className="text-lg font-semibold text-gray-900">
              System Alerts
            </h2>
            {/* Add system alerts component */}
          </div>
        </Card>
      </div>
    </div>
  );
};
```

Create `src/pages/staff/UrlManagement.tsx`:
```typescript
import React from 'react';
import { useQuery } from '@tanstack/react-query';
import { staffApi } from '@/services/api';
import { AdvancedUrlSearch } from '@/components/staff/AdvancedUrlSearch';
import { BulkOperations } from '@/components/staff/BulkOperations';
import { UrlTable } from '@/components/staff/UrlTable';

export const UrlManagement: React.FC = () => {
  const [selectedUrls, setSelectedUrls] = React.useState<string[]>([]);
  const [searchFilters, setSearchFilters] = React.useState({});

  const { data, isLoading } = useQuery(
    ['staffUrls', searchFilters],
    () => staffApi.searchUrls(searchFilters)
  );

  return (
    <div className="space-y-6">
      <div className="flex justify-between items-center">
        <h1 className="text-2xl font-bold text-gray-900">URL Management</h1>
        <BulkOperations
          selectedUrls={selectedUrls}
          onComplete={() => setSelectedUrls([])}
        />
      </div>
      <Card>
        <div className="p-6">
          <AdvancedUrlSearch onSearch={setSearchFilters} />
        </div>
      </Card>
      <UrlTable
        urls={data?.urls}
        isLoading={isLoading}
        selectedUrls={selectedUrls}
        onSelectUrls={setSelectedUrls}
      />
    </div>
  );
};
```

### 5. Update Routes

Update `src/App.tsx`:
```typescript
import { StaffLayout } from './components/layout/StaffLayout';
import { StaffOverview } from './pages/staff/Overview';
import { UrlManagement } from './pages/staff/UrlManagement';
import { AuditLog } from './pages/staff/AuditLog';

function App() {
  return (
    <Routes>
      {/* ... existing routes ... */}
      <Route
        path="/staff"
        element={
          <ProtectedRoute requiredRole={UserRole.STAFF}>
            <StaffLayout />
          </ProtectedRoute>
        }
      >
        <Route index element={<StaffOverview />} />
        <Route path="urls" element={<UrlManagement />} />
        <Route path="audit" element={<AuditLog />} />
        {/* Add more staff routes as needed */}
      </Route>
    </Routes>
  );
}
```

## Testing Staff Dashboard

Create `src/__tests__/staff/UrlManagement.test.tsx`:
```typescript
import { render, screen, fireEvent } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { UrlManagement } from '@/pages/staff/UrlManagement';

const queryClient = new QueryClient();

describe('URL Management', () => {
  it('renders search filters', () => {
    render(
      <QueryClientProvider client={queryClient}>
        <UrlManagement />
      </QueryClientProvider>
    );

    expect(screen.getByPlaceholderText(/search urls/i)).toBeInTheDocument();
    expect(screen.getByText(/status/i)).toBeInTheDocument();
    expect(screen.getByText(/date range/i)).toBeInTheDocument();
  });

  it('handles bulk operations', () => {
    render(
      <QueryClientProvider client={queryClient}>
        <UrlManagement />
      </QueryClientProvider>
    );

    const selectOperation = screen.getByText(/select operation/i);
    fireEvent.change(selectOperation, { target: { value: 'disable' } });

    expect(screen.getByText(/apply to 0 urls/i)).toBeDisabled();
  });
});
```

## Best Practices

1. Staff Access Control:
   - Validate staff permissions
   - Implement action logging
   - Restrict sensitive operations
   - Monitor staff activity

2. Bulk Operations:
   - Handle large datasets
   - Implement batch processing
   - Provide progress feedback
   - Handle failures gracefully

3. Audit Logging:
   - Log all staff actions
   - Include relevant context
   - Implement searchable logs
   - Maintain audit history

4. Performance:
   - Optimize bulk operations
   - Implement caching
   - Use pagination
   - Handle timeouts

## Common Issues and Solutions

1. Bulk Operations:
   - Handle timeouts
   - Implement retries
   - Process in batches
   - Provide rollback

2. Search Performance:
   - Optimize queries
   - Cache results
   - Use proper indexes
   - Implement filters

3. Audit Logging:
   - Handle high volume
   - Implement rotation
   - Optimize storage
   - Maintain compliance

## Next Steps

After completing this module, you should have:
- [ ] Created staff dashboard layout
- [ ] Implemented advanced URL search
- [ ] Added bulk operations
- [ ] Created audit logging
- [ ] Implemented staff permissions
- [ ] Added system monitoring
- [ ] Created staff tests

Ready to proceed to Module 13: Analytics and Tracking.

## Additional Resources

- [Role-Based Access Control](https://en.wikipedia.org/wiki/Role-based_access_control)
- [Audit Logging Best Practices](https://www.owasp.org/index.php/Logging_Cheat_Sheet)
- [Bulk Operations Patterns](https://docs.microsoft.com/en-us/azure/architecture/patterns/bulk-head)
- [MongoDB Aggregation](https://docs.mongodb.com/manual/aggregation/) 