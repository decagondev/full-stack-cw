# Module 3: MongoDB Atlas Setup

## Overview
In this module, we'll set up a MongoDB Atlas cluster to serve as our database for the URL Redirector application. MongoDB Atlas provides a cloud-hosted MongoDB service with a generous free tier perfect for development and small applications.

## Prerequisites
- MongoDB Atlas account (free tier)
- Completed Modules 0-2
- Basic understanding of MongoDB concepts

## Steps

### 1. Create MongoDB Atlas Account

1. Visit [MongoDB Atlas](https://www.mongodb.com/cloud/atlas)
2. Click "Try Free" or "Get Started Free"
3. Fill in registration details:
   - Name
   - Email
   - Password
4. Accept terms of service
5. Click "Create account"

### 2. Create New Project

1. After logging in, click "New Project"
2. Enter project details:
   ```
   Project Name: url-redirector
   Project Description: URL Redirector Application Database
   ```
3. Click "Next"
4. Add project members (optional)
5. Click "Create Project"

### 3. Create Database Cluster

1. Click "Build a Database"
2. Choose "FREE" tier:
   - Select M0 Sandbox (Shared Free Cluster)
   - Choose closest region to your deployment location
3. Select Cloud Provider & Region:
   - Choose AWS, Google Cloud, or Azure
   - Select a region close to your target users
4. Configure cluster:
   - Leave default cluster name or customize
   - Click "Create"

### 4. Configure Security Settings

1. Create Database User:
   ```
   Username: url_redirector_user
   Password: [Generate a Strong Password]
   Built-in Role: Read and write to any database
   ```
   - Store these credentials securely; you'll need them later

2. Configure Network Access:
   - Click "Network Access" in sidebar
   - Click "Add IP Address"
   - For development: Add `0.0.0.0/0` (Allow access from anywhere)
   - For production: Add specific IP addresses

### 5. Get Connection String

1. Click "Connect" on your cluster
2. Choose "Connect your application"
3. Select options:
   - Driver: Node.js
   - Version: Latest
4. Copy the connection string:
   ```
   mongodb+srv://<username>:<password>@cluster0.xxxxx.mongodb.net/?retryWrites=true&w=majority
   ```

### 6. Create Database and Collections

1. Click "Browse Collections"
2. Click "Create Database"
3. Enter details:
   ```
   Database Name: url_redirector
   Collection Name: users
   ```
4. Create additional collections:
   ```
   Collection: urls
   Collection: clicks
   ```

### 7. Update Environment Variables

1. In your backend project, update `.env`:
   ```plaintext
   MONGODB_URI=mongodb+srv://<username>:<password>@cluster0.xxxxx.mongodb.net/url_redirector?retryWrites=true&w=majority
   ```
   - Replace `<username>` and `<password>` with your credentials
   - Add `/url_redirector` to specify database name

2. Update `.env.example` (without sensitive data):
   ```plaintext
   MONGODB_URI=mongodb+srv://username:password@cluster0.xxxxx.mongodb.net/url_redirector?retryWrites=true&w=majority
   ```

## Testing the Connection

1. Create a test script in your backend project:

```typescript
// src/scripts/testConnection.ts
import mongoose from 'mongoose';
import { environment } from '../config/environment';

async function testConnection() {
  try {
    await mongoose.connect(environment.mongoUri);
    console.log('Successfully connected to MongoDB Atlas!');
    
    // List all collections
    const collections = await mongoose.connection.db.collections();
    console.log('Available collections:');
    collections.forEach(collection => {
      console.log(`- ${collection.collectionName}`);
    });
    
  } catch (error) {
    console.error('Error connecting to MongoDB Atlas:', error);
  } finally {
    await mongoose.disconnect();
  }
}

testConnection();
```

2. Run the test:
```bash
npx ts-node src/scripts/testConnection.ts
```

## Best Practices

1. Security:
   - Use strong, unique passwords
   - Restrict network access in production
   - Never commit credentials to version control
   - Use environment variables for sensitive data

2. Database Design:
   - Plan collections structure before creating
   - Use meaningful names for databases and collections
   - Consider indexing requirements early

3. Monitoring:
   - Enable MongoDB Atlas monitoring
   - Set up alerts for important metrics
   - Monitor connection pooling in production

## Common Issues and Solutions

1. Connection Errors:
   - Verify IP whitelist includes your IP
   - Check username and password
   - Ensure connection string is correct
   - Verify network connectivity

2. Authentication Issues:
   - Confirm user has correct permissions
   - Check if username/password are URL encoded
   - Verify database name in connection string

3. Performance Issues:
   - Monitor Atlas metrics
   - Check query patterns
   - Review index usage
   - Consider upgrading tier if needed

## Next Steps

After completing this module, you should have:
- [ ] Active MongoDB Atlas account
- [ ] Configured database cluster
- [ ] Created database user
- [ ] Set up network access
- [ ] Obtained connection string
- [ ] Created necessary collections
- [ ] Tested database connection

Ready to proceed to Module 4: Express.js API with MongoDB Connection.

## Additional Resources

- [MongoDB Atlas Documentation](https://docs.atlas.mongodb.com/)
- [MongoDB Node.js Driver Documentation](https://mongodb.github.io/node-mongodb-native/)
- [Mongoose Documentation](https://mongoosejs.com/docs/)
- [MongoDB Atlas Security Checklist](https://docs.atlas.mongodb.com/security-checklist/) 