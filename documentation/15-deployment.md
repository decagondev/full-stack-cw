# Module 15: Deployment

## Overview
In this module, we'll deploy our URL Redirector application to Render, setting up both the backend API and frontend client. We'll configure environment variables, set up continuous deployment, and implement monitoring.

## Prerequisites
- Completed Modules 0-14
- Render account
- Domain name (optional)
- Git repository pushed to GitHub

## Implementation Steps

### 1. Prepare Backend for Deployment

Update `src/config/environment.ts`:
```typescript
import dotenv from 'dotenv';

dotenv.config();

export const environment = {
  nodeEnv: process.env.NODE_ENV || 'development',
  port: process.env.PORT || 3000,
  mongoUri: process.env.MONGODB_URI,
  jwtSecret: process.env.JWT_SECRET,
  jwtExpiresIn: process.env.JWT_EXPIRES_IN || '1d',
  corsOrigin: process.env.CORS_ORIGIN || 'http://localhost:5173',
  redisUrl: process.env.REDIS_URL,
};

// Validate required environment variables
const requiredEnvVars = ['MONGODB_URI', 'JWT_SECRET', 'REDIS_URL'];
requiredEnvVars.forEach((envVar) => {
  if (!process.env[envVar]) {
    throw new Error(`Missing required environment variable: ${envVar}`);
  }
});
```

Create `Dockerfile` in the backend root:
```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build

EXPOSE 3000

CMD ["npm", "start"]
```

Create `.dockerignore`:
```plaintext
node_modules
npm-debug.log
dist
.env
.git
.gitignore
README.md
```

### 2. Prepare Frontend for Deployment

Update `vite.config.ts`:
```typescript
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  build: {
    outDir: 'dist',
    sourcemap: true,
  },
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: process.env.VITE_API_URL || 'http://localhost:3000',
        changeOrigin: true,
      },
    },
  },
});
```

Create `public/_redirects` for client-side routing:
```plaintext
/* /index.html 200
```

### 3. Set Up Render Services

#### Backend Web Service:
1. Create new Web Service
2. Connect GitHub repository
3. Configure build settings:
   ```bash
   # Build Command
   npm ci && npm run build

   # Start Command
   npm start
   ```
4. Add environment variables:
   ```plaintext
   NODE_ENV=production
   PORT=3000
   MONGODB_URI=your_mongodb_uri
   JWT_SECRET=your_jwt_secret
   CORS_ORIGIN=https://your-frontend-url.render.com
   REDIS_URL=your_redis_url
   ```

#### Frontend Static Site:
1. Create new Static Site
2. Connect GitHub repository
3. Configure build settings:
   ```bash
   # Build Command
   npm ci && npm run build

   # Publish Directory
   dist
   ```
4. Add environment variables:
   ```plaintext
   VITE_API_URL=https://your-backend-url.render.com
   ```

### 4. Configure Custom Domain (Optional)

1. Add custom domain in Render dashboard
2. Update DNS records:
   ```plaintext
   # A Record
   @  A  76.76.21.21

   # CNAME Record
   www  CNAME  your-site.render.com
   ```

### 5. Set Up Monitoring

Create `src/middleware/monitoring.middleware.ts`:
```typescript
import { Request, Response, NextFunction } from 'express';
import { Gauge, Counter, register } from 'prom-client';

// Initialize metrics
const httpRequestDurationMicroseconds = new Gauge({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
});

const totalRequests = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
});

export const monitoringMiddleware = (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  const start = Date.now();

  res.on('finish', () => {
    const duration = Date.now() - start;
    
    httpRequestDurationMicroseconds
      .labels(req.method, req.path, res.statusCode.toString())
      .set(duration / 1000);
    
    totalRequests
      .labels(req.method, req.path, res.statusCode.toString())
      .inc();
  });

  next();
};

// Metrics endpoint
export const metricsEndpoint = async (_req: Request, res: Response) => {
  try {
    res.set('Content-Type', register.contentType);
    res.end(await register.metrics());
  } catch (error) {
    res.status(500).end(error);
  }
};
```

Update `src/app.ts`:
```typescript
import { monitoringMiddleware, metricsEndpoint } from './middleware/monitoring.middleware';

// Add monitoring middleware
app.use(monitoringMiddleware);

// Add metrics endpoint
app.get('/metrics', metricsEndpoint);
```

### 6. Set Up CI/CD

Create `.github/workflows/deploy.yml`:
```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'
      - run: npm ci
      - run: npm test

  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Render
        uses: johnbeynon/render-deploy-action@v0.0.8
        with:
          service-id: ${{ secrets.RENDER_SERVICE_ID }}
          api-key: ${{ secrets.RENDER_API_KEY }}
```

### 7. Health Checks

Update `src/controllers/health.controller.ts`:
```typescript
import { Request, Response } from 'express';
import mongoose from 'mongoose';
import { redis } from '../lib/redis';

export const getHealth = async (req: Request, res: Response) => {
  try {
    // Check MongoDB connection
    await mongoose.connection.db.admin().ping();
    
    // Check Redis connection
    await redis.ping();

    const healthcheck = {
      status: 'healthy',
      timestamp: new Date().toISOString(),
      services: {
        database: 'connected',
        redis: 'connected',
      },
      uptime: process.uptime(),
    };

    res.status(200).json(healthcheck);
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      timestamp: new Date().toISOString(),
      error: error instanceof Error ? error.message : 'Unknown error',
    });
  }
};
```

## Testing Deployment

1. Test environment variables:
```typescript
describe('Environment Configuration', () => {
  it('validates required environment variables', () => {
    const requiredVars = ['MONGODB_URI', 'JWT_SECRET', 'REDIS_URL'];
    
    requiredVars.forEach((envVar) => {
      expect(process.env[envVar]).toBeDefined();
    });
  });
});
```

2. Test health endpoint:
```typescript
describe('Health Check', () => {
  it('returns healthy status when services are up', async () => {
    const response = await request(app).get('/api/health');
    
    expect(response.status).toBe(200);
    expect(response.body.status).toBe('healthy');
    expect(response.body.services.database).toBe('connected');
    expect(response.body.services.redis).toBe('connected');
  });
});
```

## Best Practices

1. Environment Variables:
   - Use separate environments
   - Never commit secrets
   - Validate required variables
   - Use appropriate defaults

2. Deployment:
   - Use Docker containers
   - Implement health checks
   - Set up monitoring
   - Configure auto-scaling

3. Security:
   - Enable HTTPS
   - Configure CORS properly
   - Set security headers
   - Implement rate limiting

4. Monitoring:
   - Track key metrics
   - Set up alerts
   - Monitor errors
   - Track performance

## Common Issues and Solutions

1. Deployment Failures:
   - Check build logs
   - Verify environment variables
   - Test locally first
   - Check dependencies

2. Performance Issues:
   - Enable caching
   - Optimize database queries
   - Use CDN
   - Implement compression

3. Security Concerns:
   - Review security headers
   - Update dependencies
   - Monitor logs
   - Regular security audits

## Next Steps

After completing this module, you should have:
- [ ] Prepared backend for deployment
- [ ] Prepared frontend for deployment
- [ ] Set up Render services
- [ ] Configured custom domain
- [ ] Implemented monitoring
- [ ] Set up CI/CD
- [ ] Added health checks

Ready to proceed to Module 16: Project Retrospective.

## Additional Resources

- [Render Documentation](https://render.com/docs)
- [Docker Documentation](https://docs.docker.com/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [Prometheus Node.js Client](https://github.com/siimon/prom-client) 