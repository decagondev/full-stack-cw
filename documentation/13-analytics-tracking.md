# Module 13: Analytics and Tracking

## Overview
In this module, we'll implement comprehensive analytics and tracking functionality for URL redirects. We'll create systems to track clicks, gather visitor information, generate insights, and visualize data through interactive charts and reports.

## Prerequisites
- Completed Modules 0-12
- Understanding of data aggregation
- Knowledge of data visualization
- Familiarity with MongoDB aggregation pipeline

## Implementation Steps

### 1. Create Click Tracking Model

Create `src/models/click.model.ts`:
```typescript
import mongoose, { Document, Schema } from 'mongoose';

export interface IClick extends Document {
  urlId: mongoose.Types.ObjectId;
  userId: mongoose.Types.ObjectId;
  timestamp: Date;
  ipAddress: string;
  userAgent: string;
  referer: string;
  country: string;
  city: string;
  device: {
    type: string;
    browser: string;
    os: string;
  };
  metadata: Record<string, any>;
}

const clickSchema = new Schema<IClick>(
  {
    urlId: {
      type: Schema.Types.ObjectId,
      ref: 'Url',
      required: true,
      index: true,
    },
    userId: {
      type: Schema.Types.ObjectId,
      ref: 'User',
      required: true,
      index: true,
    },
    timestamp: {
      type: Date,
      default: Date.now,
      index: true,
    },
    ipAddress: String,
    userAgent: String,
    referer: String,
    country: String,
    city: String,
    device: {
      type: String,
      browser: String,
      os: String,
    },
    metadata: {
      type: Map,
      of: Schema.Types.Mixed,
    },
  },
  {
    timestamps: true,
  }
);

// Add compound indexes for common queries
clickSchema.index({ urlId: 1, timestamp: -1 });
clickSchema.index({ userId: 1, timestamp: -1 });
clickSchema.index({ country: 1, timestamp: -1 });

export const Click = mongoose.model<IClick>('Click', clickSchema);
```

### 2. Implement Click Tracking Middleware

Create `src/middleware/tracking.middleware.ts`:
```typescript
import { Request, Response, NextFunction } from 'express';
import { Click } from '../models/click.model';
import { parseUserAgent } from '../utils/device';
import { getGeoLocation } from '../utils/geo';
import logger from '../utils/logger';

export const trackClick = async (
  req: Request,
  res: Response,
  next: NextFunction
) => {
  try {
    const { urlId, userId } = req.params;
    const userAgent = req.headers['user-agent'];
    const referer = req.headers.referer;
    const ipAddress = req.ip;

    // Parse user agent
    const deviceInfo = parseUserAgent(userAgent);
    
    // Get geo location
    const geoInfo = await getGeoLocation(ipAddress);

    // Create click record
    const click = new Click({
      urlId,
      userId,
      ipAddress,
      userAgent,
      referer,
      country: geoInfo.country,
      city: geoInfo.city,
      device: deviceInfo,
      metadata: {
        headers: req.headers,
        query: req.query,
      },
    });

    // Save asynchronously - don't wait for completion
    click.save().catch((error) => {
      logger.error('Error saving click tracking:', error);
    });

    next();
  } catch (error) {
    logger.error('Error in click tracking:', error);
    next(); // Continue even if tracking fails
  }
};
```

### 3. Create Analytics Service

Create `src/services/analytics.service.ts`:
```typescript
import { Click } from '../models/click.model';
import { startOfDay, endOfDay, subDays } from 'date-fns';

export class AnalyticsService {
  static async getUrlStats(urlId: string, days = 30) {
    const startDate = startOfDay(subDays(new Date(), days));
    const endDate = endOfDay(new Date());

    const [clickStats, deviceStats, locationStats] = await Promise.all([
      Click.aggregate([
        {
          $match: {
            urlId,
            timestamp: { $gte: startDate, $lte: endDate },
          },
        },
        {
          $group: {
            _id: {
              $dateToString: { format: '%Y-%m-%d', date: '$timestamp' },
            },
            count: { $sum: 1 },
          },
        },
        { $sort: { '_id': 1 } },
      ]),
      Click.aggregate([
        {
          $match: { urlId },
        },
        {
          $group: {
            _id: '$device.type',
            count: { $sum: 1 },
          },
        },
      ]),
      Click.aggregate([
        {
          $match: { urlId },
        },
        {
          $group: {
            _id: '$country',
            count: { $sum: 1 },
          },
        },
        { $sort: { count: -1 } },
        { $limit: 10 },
      ]),
    ]);

    return {
      clicks: clickStats,
      devices: deviceStats,
      locations: locationStats,
    };
  }

  static async getUserStats(userId: string, days = 30) {
    const startDate = startOfDay(subDays(new Date(), days));
    const endDate = endOfDay(new Date());

    const stats = await Click.aggregate([
      {
        $match: {
          userId,
          timestamp: { $gte: startDate, $lte: endDate },
        },
      },
      {
        $group: {
          _id: {
            urlId: '$urlId',
            date: { $dateToString: { format: '%Y-%m-%d', date: '$timestamp' } },
          },
          count: { $sum: 1 },
        },
      },
      {
        $group: {
          _id: '$_id.urlId',
          dailyClicks: {
            $push: {
              date: '$_id.date',
              count: '$count',
            },
          },
          totalClicks: { $sum: '$count' },
        },
      },
      {
        $lookup: {
          from: 'urls',
          localField: '_id',
          foreignField: '_id',
          as: 'url',
        },
      },
      { $unwind: '$url' },
      { $sort: { totalClicks: -1 } },
    ]);

    return stats;
  }

  static async getSystemStats() {
    const [totalClicks, activeUrls, topLocations, deviceDistribution] = await Promise.all([
      Click.countDocuments(),
      Click.distinct('urlId').exec(),
      Click.aggregate([
        {
          $group: {
            _id: {
              country: '$country',
              city: '$city',
            },
            count: { $sum: 1 },
          },
        },
        { $sort: { count: -1 } },
        { $limit: 10 },
      ]),
      Click.aggregate([
        {
          $group: {
            _id: '$device.type',
            count: { $sum: 1 },
          },
        },
      ]),
    ]);

    return {
      totalClicks,
      activeUrls: activeUrls.length,
      topLocations,
      deviceDistribution,
    };
  }
}
```

### 4. Create Analytics Controllers

Create `src/controllers/analytics.controller.ts`:
```typescript
import { Request, Response, NextFunction } from 'express';
import { AnalyticsService } from '../services/analytics.service';
import { ApiError } from '../middleware/error.middleware';

export class AnalyticsController {
  static async getUrlStats(req: Request, res: Response, next: NextFunction) {
    try {
      const { urlId } = req.params;
      const { days } = req.query;

      const stats = await AnalyticsService.getUrlStats(
        urlId,
        Number(days) || 30
      );

      res.status(200).json({
        status: 'success',
        data: stats,
      });
    } catch (error) {
      next(error);
    }
  }

  static async getUserStats(req: Request, res: Response, next: NextFunction) {
    try {
      const { userId } = req.params;
      const { days } = req.query;

      const stats = await AnalyticsService.getUserStats(
        userId,
        Number(days) || 30
      );

      res.status(200).json({
        status: 'success',
        data: stats,
      });
    } catch (error) {
      next(error);
    }
  }

  static async getSystemStats(req: Request, res: Response, next: NextFunction) {
    try {
      const stats = await AnalyticsService.getSystemStats();

      res.status(200).json({
        status: 'success',
        data: stats,
      });
    } catch (error) {
      next(error);
    }
  }
}
```

### 5. Create Analytics Components

Create `src/components/analytics/ClicksChart.tsx`:
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
import { format, parseISO } from 'date-fns';

interface ClicksChartProps {
  data: Array<{
    _id: string;
    count: number;
  }>;
}

export const ClicksChart: React.FC<ClicksChartProps> = ({ data }) => {
  const formattedData = data.map((item) => ({
    date: item._id,
    clicks: item.count,
  }));

  return (
    <div className="h-64">
      <ResponsiveContainer width="100%" height="100%">
        <LineChart data={formattedData}>
          <CartesianGrid strokeDasharray="3 3" />
          <XAxis
            dataKey="date"
            tickFormatter={(date) => format(parseISO(date), 'MMM d')}
          />
          <YAxis />
          <Tooltip
            labelFormatter={(date) =>
              format(parseISO(date as string), 'MMM d, yyyy')
            }
          />
          <Line
            type="monotone"
            dataKey="clicks"
            stroke="#4f46e5"
            strokeWidth={2}
          />
        </LineChart>
      </ResponsiveContainer>
    </div>
  );
};
```

Create `src/components/analytics/LocationMap.tsx`:
```typescript
import React from 'react';
import {
  ComposableMap,
  Geographies,
  Geography,
  Marker,
} from 'react-simple-maps';

interface LocationData {
  _id: {
    country: string;
    city: string;
  };
  count: number;
  coordinates: [number, number];
}

interface LocationMapProps {
  data: LocationData[];
}

export const LocationMap: React.FC<LocationMapProps> = ({ data }) => {
  return (
    <div className="h-96">
      <ComposableMap>
        <Geographies geography="/world-110m.json">
          {({ geographies }) =>
            geographies.map((geo) => (
              <Geography
                key={geo.rsmKey}
                geography={geo}
                fill="#D6D6DA"
                stroke="#FFFFFF"
              />
            ))
          }
        </Geographies>
        {data.map((location) => (
          <Marker
            key={`${location._id.country}-${location._id.city}`}
            coordinates={location.coordinates}
          >
            <circle
              r={Math.sqrt(location.count) * 2}
              fill="#4f46e5"
              opacity={0.7}
            />
          </Marker>
        ))}
      </ComposableMap>
    </div>
  );
};
```

### 6. Update Routes

Create `src/routes/analytics.routes.ts`:
```typescript
import { Router } from 'express';
import { AnalyticsController } from '../controllers/analytics.controller';
import { authenticate, authorize } from '../middleware/auth.middleware';
import { UserRole } from '../models/user.model';

const router = Router();

router.get(
  '/urls/:urlId',
  authenticate,
  AnalyticsController.getUrlStats
);

router.get(
  '/users/:userId',
  authenticate,
  AnalyticsController.getUserStats
);

router.get(
  '/system',
  authenticate,
  authorize(UserRole.STAFF),
  AnalyticsController.getSystemStats
);

export default router;
```

## Testing Analytics

Create `src/__tests__/analytics/tracking.test.ts`:
```typescript
import request from 'supertest';
import { app } from '../../app';
import { Click } from '../../models/click.model';
import { createTestUrl, createTestUser, getAuthToken } from '../helpers';

describe('Click Tracking', () => {
  let url: any;
  let token: string;

  beforeAll(async () => {
    const user = await createTestUser();
    token = await getAuthToken(user);
    url = await createTestUrl(user.id);
  });

  beforeEach(async () => {
    await Click.deleteMany({});
  });

  it('tracks click when accessing URL', async () => {
    await request(app)
      .get(`/api/r/${url.shortId}`)
      .expect(302);

    const clicks = await Click.find({ urlId: url.id });
    expect(clicks).toHaveLength(1);
    expect(clicks[0]).toHaveProperty('ipAddress');
    expect(clicks[0]).toHaveProperty('userAgent');
  });

  it('retrieves URL statistics', async () => {
    const response = await request(app)
      .get(`/api/analytics/urls/${url.id}`)
      .set('Authorization', `Bearer ${token}`)
      .expect(200);

    expect(response.body.data).toHaveProperty('clicks');
    expect(response.body.data).toHaveProperty('devices');
    expect(response.body.data).toHaveProperty('locations');
  });
});
```

## Best Practices

1. Data Collection:
   - Track essential metrics only
   - Respect user privacy
   - Handle tracking failures gracefully
   - Implement data retention policies

2. Performance:
   - Use efficient aggregation
   - Implement caching
   - Handle large datasets
   - Optimize queries

3. Visualization:
   - Use appropriate chart types
   - Implement responsive design
   - Handle loading states
   - Provide interactive features

4. Privacy:
   - Anonymize sensitive data
   - Follow GDPR guidelines
   - Implement data protection
   - Allow opt-out options

## Common Issues and Solutions

1. Performance:
   - Use appropriate indexes
   - Implement data aggregation
   - Cache frequent queries
   - Handle timeouts

2. Data Accuracy:
   - Validate tracking data
   - Handle duplicate clicks
   - Account for bots
   - Implement retry logic

3. Scalability:
   - Use efficient storage
   - Implement data archiving
   - Handle high traffic
   - Optimize queries

## Next Steps

After completing this module, you should have:
- [ ] Created click tracking model
- [ ] Implemented tracking middleware
- [ ] Added analytics service
- [ ] Created visualization components
- [ ] Implemented privacy measures
- [ ] Added analytics routes
- [ ] Created analytics tests

Ready to proceed to Module 14: Feature Polish.

## Additional Resources

- [MongoDB Aggregation Pipeline](https://docs.mongodb.com/manual/core/aggregation-pipeline/)
- [Recharts Documentation](https://recharts.org/)
- [React Simple Maps](https://www.react-simple-maps.io/)
- [Web Analytics Privacy](https://gdpr.eu/web-analytics/) 