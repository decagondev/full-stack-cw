# Appendix: Design Guidelines and Additional Resources

## UI/UX Guidelines

### Color Scheme
```typescript
interface ColorPalette {
  primary: {
    50: '#f0f9ff';
    100: '#e0f2fe';
    200: '#bae6fd';
    300: '#7dd3fc';
    400: '#38bdf8';
    500: '#0ea5e9';  // Primary brand color
    600: '#0284c7';
    700: '#0369a1';
    800: '#075985';
    900: '#0c4a6e';
  };
  neutral: {
    50: '#f8fafc';
    100: '#f1f5f9';
    200: '#e2e8f0';
    300: '#cbd5e1';
    400: '#94a3b8';
    500: '#64748b';
    600: '#475569';
    700: '#334155';
    800: '#1e293b';
    900: '#0f172a';
  };
  success: {
    main: '#10b981';
    light: '#34d399';
    dark: '#059669';
  };
  error: {
    main: '#ef4444';
    light: '#f87171';
    dark: '#dc2626';
  };
  warning: {
    main: '#f59e0b';
    light: '#fbbf24';
    dark: '#d97706';
  };
  info: {
    main: '#3b82f6';
    light: '#60a5fa';
    dark: '#2563eb';
  };
}
```

### Typography System
```typescript
interface Typography {
  fontFamily: {
    sans: 'Inter, system-ui, sans-serif';
    mono: 'JetBrains Mono, monospace';
  };
  fontSize: {
    xs: '0.75rem';    // 12px
    sm: '0.875rem';   // 14px
    base: '1rem';     // 16px
    lg: '1.125rem';   // 18px
    xl: '1.25rem';    // 20px
    '2xl': '1.5rem';  // 24px
    '3xl': '1.875rem';// 30px
    '4xl': '2.25rem'; // 36px
  };
  fontWeight: {
    normal: '400';
    medium: '500';
    semibold: '600';
    bold: '700';
  };
  lineHeight: {
    tight: '1.25';
    normal: '1.5';
    relaxed: '1.75';
  };
}
```

### Spacing System
```typescript
interface Spacing {
  space: {
    px: '1px';
    0: '0';
    0.5: '0.125rem';  // 2px
    1: '0.25rem';     // 4px
    2: '0.5rem';      // 8px
    3: '0.75rem';     // 12px
    4: '1rem';        // 16px
    5: '1.25rem';     // 20px
    6: '1.5rem';      // 24px
    8: '2rem';        // 32px
    10: '2.5rem';     // 40px
    12: '3rem';       // 48px
    16: '4rem';       // 64px
    20: '5rem';       // 80px
  };
}
```

### Component Design Guidelines

#### Buttons
```css
/* Primary Button */
.btn-primary {
  @apply bg-primary-500 text-white px-4 py-2 rounded-lg
         hover:bg-primary-600 focus:ring-2 focus:ring-primary-500/50
         disabled:opacity-50 disabled:cursor-not-allowed
         transition-colors duration-200;
}

/* Secondary Button */
.btn-secondary {
  @apply bg-gray-100 text-gray-900 px-4 py-2 rounded-lg
         hover:bg-gray-200 focus:ring-2 focus:ring-gray-500/50
         disabled:opacity-50 disabled:cursor-not-allowed
         transition-colors duration-200;
}

/* Danger Button */
.btn-danger {
  @apply bg-error-main text-white px-4 py-2 rounded-lg
         hover:bg-error-dark focus:ring-2 focus:ring-error-main/50
         disabled:opacity-50 disabled:cursor-not-allowed
         transition-colors duration-200;
}
```

#### Forms
```css
/* Input Field */
.input-field {
  @apply w-full px-3 py-2 rounded-lg border border-gray-300
         focus:outline-none focus:ring-2 focus:ring-primary-500/50
         focus:border-primary-500 transition-colors duration-200;
}

/* Form Label */
.form-label {
  @apply block text-sm font-medium text-gray-700 mb-1;
}

/* Form Error */
.form-error {
  @apply text-sm text-error-main mt-1;
}
```

## UX Best Practices

### Loading States
1. Skeleton Loading
```typescript
interface SkeletonProps {
  lines?: number;
  height?: string;
  width?: string;
  animate?: boolean;
}

// Usage Example
<Skeleton
  lines={3}
  height="20px"
  width="100%"
  animate={true}
/>
```

2. Progress Indicators
```typescript
interface ProgressIndicator {
  type: 'spinner' | 'bar' | 'dots';
  size?: 'sm' | 'md' | 'lg';
  color?: string;
  thickness?: number;
}
```

### Error Handling
1. Form Validation
```typescript
interface ValidationRules {
  required?: boolean;
  minLength?: number;
  maxLength?: number;
  pattern?: RegExp;
  custom?: (value: any) => boolean;
}

interface ValidationError {
  field: string;
  message: string;
  type: 'error' | 'warning';
}
```

2. Error Messages
```typescript
const errorMessages = {
  required: 'This field is required',
  minLength: 'Must be at least {min} characters',
  maxLength: 'Must be less than {max} characters',
  email: 'Please enter a valid email address',
  url: 'Please enter a valid URL',
  password: 'Password must contain at least 8 characters, one uppercase letter, and one number',
};
```

## Accessibility Guidelines

### ARIA Labels
```typescript
interface AriaProps {
  'aria-label'?: string;
  'aria-describedby'?: string;
  'aria-hidden'?: boolean;
  role?: string;
  tabIndex?: number;
}
```

### Keyboard Navigation
```typescript
const keyboardShortcuts = {
  navigation: {
    'Tab': 'Move focus to next element',
    'Shift + Tab': 'Move focus to previous element',
    'Enter/Space': 'Activate focused element',
    'Esc': 'Close modal/popup',
  },
  application: {
    'Ctrl + K': 'Open search',
    'Ctrl + /': 'Show keyboard shortcuts',
    'Ctrl + S': 'Save changes',
  },
};
```

## Animation Guidelines

### Transition Presets
```typescript
interface Transitions {
  default: 'all 0.2s ease-in-out';
  fast: 'all 0.1s ease-in-out';
  slow: 'all 0.3s ease-in-out';
  bounce: 'all 0.2s cubic-bezier(0.68, -0.55, 0.265, 1.55)';
}

interface AnimationPresets {
  fadeIn: {
    from: { opacity: 0 };
    to: { opacity: 1 };
  };
  slideIn: {
    from: { transform: 'translateY(20px)', opacity: 0 };
    to: { transform: 'translateY(0)', opacity: 1 };
  };
  scaleIn: {
    from: { transform: 'scale(0.95)', opacity: 0 };
    to: { transform: 'scale(1)', opacity: 1 };
  };
}
```

## Responsive Design

### Breakpoints
```typescript
interface Breakpoints {
  sm: '640px';   // Small devices
  md: '768px';   // Medium devices
  lg: '1024px';  // Large devices
  xl: '1280px';  // Extra large devices
  '2xl': '1536px'; // 2X Extra large devices
}
```

### Container Sizes
```typescript
interface ContainerSizes {
  sm: 'max-width: 640px';
  md: 'max-width: 768px';
  lg: 'max-width: 1024px';
  xl: 'max-width: 1280px';
  '2xl': 'max-width: 1536px';
}
```

## Icon System

### Icon Guidelines
```typescript
interface IconProps {
  size?: 'sm' | 'md' | 'lg' | number;
  color?: string;
  strokeWidth?: number;
  className?: string;
}

// Using Lucide Icons
import { 
  Link2, 
  Settings, 
  User, 
  ChartBar,
  Shield 
} from 'lucide-react';
```

## Additional Resources

### Design Tools
- Figma for UI design
- Storybook for component documentation
- Chromatic for visual testing
- Contrast checker for accessibility

### UI Libraries
- Headless UI for accessible components
- Radix UI for advanced components
- React Hook Form for form handling
- TanStack Query for data fetching

### Performance Tools
- Lighthouse for performance auditing
- WebPageTest for detailed analysis
- Bundle analyzer for package size optimization
- React DevTools for component debugging

### Monitoring Tools
- LogRocket for session replay
- Sentry for error tracking
- Google Analytics for user behavior
- HotJar for heatmaps

## Version Control Guidelines

### Branch Naming
```typescript
const branchTypes = {
  feature: 'feature/',  // New features
  bugfix: 'bugfix/',    // Bug fixes
  hotfix: 'hotfix/',    // Urgent fixes
  release: 'release/',  // Release branches
  docs: 'docs/',        // Documentation updates
};
```

### Commit Message Format
```typescript
interface CommitMessage {
  type: 'feat' | 'fix' | 'docs' | 'style' | 'refactor' | 'test' | 'chore';
  scope?: string;
  subject: string;
  body?: string;
  footer?: string;
}

// Example:
// feat(auth): add password reset functionality
// 
// - Implement password reset email
// - Add reset token validation
// - Create new password form
// 
// Closes #123
```

## Next Steps
This appendix serves as a reference for maintaining consistency in design and development. For implementation details, refer to the specific modules in the documentation. 