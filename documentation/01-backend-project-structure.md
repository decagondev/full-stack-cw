# Module 1: Backend Project Structure Setup

## Overview
In this module, we'll set up a structured Express.js backend with TypeScript, establishing a clean architecture that supports scalability and maintainability.

## Prerequisites
- Node.js (v18 or higher)
- Git
- Code editor (VSCode recommended)
- Completed Module 0

## Project Structure
```
url-redirector-api/
├── src/
│   ├── config/
│   │   ├── database.ts
│   │   └── environment.ts
│   ├── controllers/
│   │   └── health.controller.ts
│   ├── middleware/
│   │   ├── error.middleware.ts
│   │   └── logging.middleware.ts
│   ├── models/
│   │   └── index.ts
│   ├── routes/
│   │   ├── index.ts
│   │   └── health.routes.ts
│   ├── services/
│   │   └── index.ts
│   ├── types/
│   │   └── index.ts
│   ├── utils/
│   │   ├── logger.ts
│   │   └── response.ts
│   └── app.ts
├── tests/
│   └── health.test.ts
├── .env.example
├── .eslintrc.js
├── .gitignore
├── .prettierrc
├── jest.config.js
├── nodemon.json
├── package.json
├── README.md
├── tsconfig.json
└── ecosystem.config.js
```

## Step-by-Step Setup

### 1. Initialize Project
```bash
# Navigate to your cloned repository
cd url-redirector-api

# Initialize npm project
npm init -y

# Install dependencies
npm install express dotenv cors helmet morgan mongoose express-validator joi
npm install jsonwebtoken bcryptjs

# Install development dependencies
npm install -D typescript @types/express @types/node @types/cors @types/morgan
npm install -D @types/jsonwebtoken @types/bcryptjs nodemon ts-node
npm install -D jest @types/jest supertest @types/supertest
npm install -D eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin
npm install -D prettier eslint-config-prettier eslint-plugin-prettier
```

### 2. Configure TypeScript
Create `tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "moduleResolution": "node",
    "resolveJsonModule": true,
    "baseUrl": ".",
    "paths": {
      "@/*": ["src/*"]
    }
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "tests"]
}
```

### 3. Configure ESLint
Create `.eslintrc.js`:
```javascript
module.exports = {
  parser: '@typescript-eslint/parser',
  extends: [
    'plugin:@typescript-eslint/recommended',
    'prettier',
    'plugin:prettier/recommended',
  ],
  parserOptions: {
    ecmaVersion: 2020,
    sourceType: 'module',
  },
  rules: {
    '@typescript-eslint/explicit-function-return-type': 'off',
    '@typescript-eslint/no-explicit-any': 'off',
    '@typescript-eslint/no-unused-vars': ['error', { argsIgnorePattern: '^_' }],
  },
};
```

### 4. Configure Prettier
Create `.prettierrc`:
```json
{
  "semi": true,
  "trailingComma": "all",
  "singleQuote": true,
  "printWidth": 100,
  "tabWidth": 2
}
```

### 5. Create Base Files

#### src/app.ts
```typescript
import express, { Application } from 'express';
import cors from 'cors';
import helmet from 'helmet';
import morgan from 'morgan';
import routes from './routes';
import { errorHandler } from './middleware/error.middleware';
import { setupLogging } from './middleware/logging.middleware';

const app: Application = express();

// Middleware
app.use(helmet());
app.use(cors());
app.use(express.json());
app.use(morgan('dev'));
setupLogging(app);

// Routes
app.use('/api', routes);

// Error handling
app.use(errorHandler);

export default app;
```

#### src/config/environment.ts
```typescript
import dotenv from 'dotenv';

dotenv.config();

export const environment = {
  nodeEnv: process.env.NODE_ENV || 'development',
  port: process.env.PORT || 3000,
  mongoUri: process.env.MONGODB_URI || 'mongodb://localhost:27017/url-redirector',
  jwtSecret: process.env.JWT_SECRET || 'your-secret-key',
  jwtExpiresIn: process.env.JWT_EXPIRES_IN || '1d',
};
```

#### src/utils/logger.ts
```typescript
import winston from 'winston';

const logger = winston.createLogger({
  level: 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' })
  ]
});

if (process.env.NODE_ENV !== 'production') {
  logger.add(new winston.transports.Console({
    format: winston.format.simple()
  }));
}

export default logger;
```

### 6. Update package.json Scripts
```json
{
  "scripts": {
    "start": "node dist/app.js",
    "dev": "nodemon",
    "build": "tsc",
    "lint": "eslint . --ext .ts",
    "lint:fix": "eslint . --ext .ts --fix",
    "test": "jest",
    "test:watch": "jest --watch"
  }
}
```

### 7. Configure Nodemon
Create `nodemon.json`:
```json
{
  "watch": ["src"],
  "ext": ".ts,.js",
  "ignore": [],
  "exec": "ts-node ./src/app.ts"
}
```

### 8. Create Environment Variables
Create `.env.example`:
```plaintext
NODE_ENV=development
PORT=3000
MONGODB_URI=mongodb://localhost:27017/url-redirector
JWT_SECRET=your-secret-key
JWT_EXPIRES_IN=1d
```

### 9. Update .gitignore
```plaintext
# Dependencies
node_modules/

# Build output
dist/

# Environment variables
.env

# Logs
logs/
*.log

# IDE
.vscode/
.idea/

# Testing
coverage/

# OS
.DS_Store
```

## Testing the Setup

1. Create a basic health check route:

#### src/routes/health.routes.ts
```typescript
import { Router } from 'express';
import { getHealth } from '../controllers/health.controller';

const router = Router();

router.get('/', getHealth);

export default router;
```

#### src/controllers/health.controller.ts
```typescript
import { Request, Response } from 'express';

export const getHealth = (_req: Request, res: Response) => {
  res.status(200).json({
    status: 'success',
    message: 'Server is healthy',
    timestamp: new Date().toISOString(),
  });
};
```

2. Test the setup:
```bash
# Install dependencies
npm install

# Start development server
npm run dev
```

3. Visit `http://localhost:3000/api/health` to verify the setup

## Best Practices

1. Keep the folder structure organized and modular
2. Use TypeScript for better type safety and developer experience
3. Implement proper error handling from the start
4. Set up logging early in development
5. Follow consistent coding standards
6. Use environment variables for configuration
7. Implement proper security measures from the beginning

## Common Issues and Solutions

1. TypeScript compilation errors:
   - Check `tsconfig.json` settings
   - Ensure all required types are installed
   - Verify import paths are correct

2. Nodemon not restarting:
   - Check `nodemon.json` configuration
   - Verify file extensions in watch list
   - Check for syntax errors in TypeScript files

3. ESLint/Prettier conflicts:
   - Verify `.eslintrc.js` and `.prettierrc` configurations
   - Install necessary editor extensions
   - Run `npm run lint:fix` to automatically fix issues

## Next Steps

After completing this module, you should have:
- [ ] A fully configured TypeScript Express.js project
- [ ] Proper project structure with separated concerns
- [ ] Development environment setup with hot reloading
- [ ] Basic health check endpoint
- [ ] Logging and error handling middleware
- [ ] Code linting and formatting tools

Ready to proceed to Module 2: Frontend Project Structure.