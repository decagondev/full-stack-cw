# Module 4: Express.js API with MongoDB Connection

## Overview
In this module, we'll implement the core Express.js API structure and establish a robust MongoDB connection. We'll set up essential middleware, error handling, and create basic health check endpoints to ensure everything is working correctly.

## Prerequisites
- Completed Modules 0-3
- MongoDB Atlas cluster configured
- Node.js and npm installed
- Basic understanding of Express.js and MongoDB

## Implementation Steps

### 1. Configure MongoDB Connection

Create `src/config/database.ts`:
```typescript
import mongoose from 'mongoose';
import { environment } from './environment';
import logger from '../utils/logger';

export async function connectDatabase(): Promise<void> {
  try {
    const options = {
      autoIndex: true,
      minPoolSize: 10,
      maxPoolSize: 50,
      connectTimeoutMS: 10000,
      socketTimeoutMS: 45000,
    };

    await mongoose.connect(environment.mongoUri, options);
    logger.info('Successfully connected to MongoDB');

    mongoose.connection.on('error', (error) => {
      logger.error('MongoDB connection error:', error);
    });

    mongoose.connection.on('disconnected', () => {
      logger.warn('MongoDB disconnected. Attempting to reconnect...');
    });

    mongoose.connection.on('reconnected', () => {
      logger.info('MongoDB reconnected');
    });
  } catch (error) {
    logger.error('Error connecting to MongoDB:', error);
    process.exit(1);
  }
}
```

### 2. Implement Error Handling Middleware

Create `src/middleware/error.middleware.ts`:
```typescript
import { Request, Response, NextFunction } from 'express';
import logger from '../utils/logger';

export class ApiError extends Error {
  constructor(
    public statusCode: number,
    public message: string,
    public source?: string,
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
  if (error instanceof ApiError) {
    logger.error(`API Error: ${error.message}`, {
      statusCode: error.statusCode,
      source: error.source,
      path: req.path,
    });

    return res.status(error.statusCode).json({
      status: 'error',
      message: error.message,
      source: error.source,
    });
  }

  logger.error('Unhandled Error:', error);
  
  return res.status(500).json({
    status: 'error',
    message: 'Internal server error',
  });
};
```

### 3. Implement Logging Middleware

Create `src/middleware/logging.middleware.ts`:
```typescript
import { Request, Response, NextFunction } from 'express';
import logger from '../utils/logger';

export const requestLogger = (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;
    logger.info('Request processed', {
      method: req.method,
      path: req.path,
      statusCode: res.statusCode,
      duration: `${duration}ms`,
    });
  });

  next();
};

export const errorLogger = (
  error: Error,
  req: Request,
  res: Response,
  next: NextFunction
) => {
  logger.error('Error occurred:', {
    error: error.message,
    stack: error.stack,
    path: req.path,
    method: req.method,
  });
  next(error);
};
```

### 4. Create Health Check Controller

Create `src/controllers/health.controller.ts`:
```typescript
import { Request, Response } from 'express';
import mongoose from 'mongoose';

export const getHealth = async (req: Request, res: Response) => {
  const healthcheck = {
    status: 'success',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    database: {
      status: mongoose.connection.readyState === 1 ? 'connected' : 'disconnected',
    },
    memory: {
      usage: process.memoryUsage(),
    },
  };

  res.status(200).json(healthcheck);
};

export const getDatabaseHealth = async (req: Request, res: Response) => {
  try {
    // Perform DB operation to verify connection
    await mongoose.connection.db.admin().ping();
    
    res.status(200).json({
      status: 'success',
      message: 'Database connection is healthy',
      details: {
        status: mongoose.connection.readyState === 1 ? 'connected' : 'disconnected',
        host: mongoose.connection.host,
        name: mongoose.connection.name,
      },
    });
  } catch (error) {
    res.status(503).json({
      status: 'error',
      message: 'Database connection is unhealthy',
      error: error instanceof Error ? error.message : 'Unknown error',
    });
  }
};
```

### 5. Set Up Routes

Create `src/routes/health.routes.ts`:
```typescript
import { Router } from 'express';
import { getHealth, getDatabaseHealth } from '../controllers/health.controller';

const router = Router();

router.get('/', getHealth);
router.get('/database', getDatabaseHealth);

export default router;
```

Update `src/routes/index.ts`:
```typescript
import { Router } from 'express';
import healthRoutes from './health.routes';

const router = Router();

router.use('/health', healthRoutes);

export default router;
```

### 6. Update Main Application File

Update `src/app.ts`:
```typescript
import express from 'express';
import cors from 'cors';
import helmet from 'helmet';
import compression from 'compression';
import { connectDatabase } from './config/database';
import { errorHandler } from './middleware/error.middleware';
import { requestLogger, errorLogger } from './middleware/logging.middleware';
import routes from './routes';
import { environment } from './config/environment';

const app = express();

// Middleware
app.use(helmet());
app.use(cors());
app.use(compression());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(requestLogger);

// Routes
app.use('/api', routes);

// Error handling
app.use(errorLogger);
app.use(errorHandler);

// Start server
const startServer = async () => {
  try {
    await connectDatabase();
    
    app.listen(environment.port, () => {
      console.log(`Server running on port ${environment.port}`);
    });
  } catch (error) {
    console.error('Failed to start server:', error);
    process.exit(1);
  }
};

startServer();

export default app;
```

### 7. Add Rate Limiting

Install required package:
```bash
npm install express-rate-limit
```

Create `src/middleware/rate-limit.middleware.ts`:
```typescript
import rateLimit from 'express-rate-limit';
import { environment } from '../config/environment';

export const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: environment.nodeEnv === 'production' ? 100 : 1000, // limit each IP
  message: {
    status: 'error',
    message: 'Too many requests from this IP, please try again later.',
  },
  standardHeaders: true,
  legacyHeaders: false,
});
```

## Testing the Implementation

1. Create a test file `src/__tests__/health.test.ts`:
```typescript
import request from 'supertest';
import app from '../app';

describe('Health Check Endpoints', () => {
  it('should return 200 for main health check', async () => {
    const response = await request(app).get('/api/health');
    expect(response.status).toBe(200);
    expect(response.body.status).toBe('success');
  });

  it('should return database health status', async () => {
    const response = await request(app).get('/api/health/database');
    expect(response.status).toBe(200);
    expect(response.body.status).toBe('success');
  });
});
```

2. Run the tests:
```bash
npm test
```

## Best Practices

1. Error Handling:
   - Use custom error classes
   - Implement centralized error handling
   - Log errors with appropriate context
   - Return consistent error responses

2. Security:
   - Use helmet for security headers
   - Implement rate limiting
   - Validate input data
   - Use CORS appropriately

3. Performance:
   - Enable compression
   - Configure appropriate connection pools
   - Implement request logging
   - Monitor response times

4. Monitoring:
   - Implement comprehensive health checks
   - Log important events
   - Track database connection status
   - Monitor memory usage

## Common Issues and Solutions

1. Connection Issues:
   - Verify MongoDB URI
   - Check network connectivity
   - Ensure proper error handling
   - Monitor connection events

2. Performance Problems:
   - Adjust connection pool size
   - Monitor memory usage
   - Check for memory leaks
   - Optimize database queries

3. Security Concerns:
   - Review security headers
   - Adjust rate limits
   - Update CORS settings
   - Validate all inputs

## Next Steps

After completing this module, you should have:
- [ ] Configured MongoDB connection
- [ ] Implemented error handling
- [ ] Set up logging middleware
- [ ] Created health check endpoints
- [ ] Added security measures
- [ ] Implemented rate limiting
- [ ] Written basic tests

Ready to proceed to Module 5: Backend Authentication.

## Additional Resources

- [Express.js Documentation](https://expressjs.com/)
- [Mongoose Documentation](https://mongoosejs.com/)
- [Express Rate Limit](https://github.com/nfriedly/express-rate-limit)
- [Helmet.js Documentation](https://helmetjs.github.io/)
- [Express.js Security Best Practices](https://expressjs.com/en/advanced/best-practice-security.html) 