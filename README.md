# URL Redirector: Modern Full-Stack Development Course

## Overview
A comprehensive full-stack development course building a production-ready URL shortening and management platform. This project demonstrates modern web development practices using TypeScript, React, Express.js, and MongoDB.

## Project Documentation Structure

### Getting Started
- [Prelogue: Project Overview](documentation/00-prelogue.md) - Product overview and user stories
- [Module 0: Project Setup](documentation/00-project-setup.md) - Initial repository and environment setup
- [Appendix: Design Guidelines](documentation/appendix.md) - UI/UX guidelines and design system

### Backend Development
1. [Backend Project Structure](documentation/01-backend-project-structure.md)
   - Express.js setup with TypeScript
   - Project organization
   - Development tooling

2. [Frontend Project Structure](documentation/02-frontend-project-structure.md)
   - React with Vite setup
   - TypeScript configuration
   - Project organization

3. [MongoDB Atlas Setup](documentation/03-mongodb-atlas-setup.md)
   - Database configuration
   - Connection setup
   - Security settings

4. [Express.js API with MongoDB](documentation/04-express-api-mongodb.md)
   - API implementation
   - Database integration
   - Error handling

5. [Backend Authentication](documentation/05-backend-authentication.md)
   - JWT implementation
   - User authentication
   - Role-based access control

### Frontend Development
6. [Frontend Foundation](documentation/06-frontend-foundation.md)
   - Component structure
   - Routing setup
   - State management

7. [Frontend Authentication](documentation/07-frontend-authentication.md)
   - Authentication flow
   - Protected routes
   - User session management

8. [URL Redirect CRUD](documentation/08-url-redirect-crud.md)
   - URL management
   - Redirection logic
   - URL validation

### User Management & Dashboards
9. [Admin User Management API](documentation/09-admin-user-management.md)
   - User CRUD operations
   - Role management
   - User permissions

10. [Admin Dashboard](documentation/10-admin-dashboard.md)
    - Admin interface
    - User management UI
    - System monitoring

11. [User Dashboard](documentation/11-user-dashboard.md)
    - Personal URL management
    - User statistics
    - Profile settings

12. [Staff Dashboard](documentation/12-staff-dashboard.md)
    - Moderation tools
    - URL management
    - User support features

### Advanced Features
13. [Analytics and Tracking](documentation/13-analytics-tracking.md)
    - Click tracking
    - Analytics dashboard
    - Reporting features

14. [Feature Polish](documentation/14-feature-polish.md)
    - Performance optimization
    - Error handling
    - UX improvements

### Deployment & Review
15. [Deployment](documentation/15-deployment.md)
    - Render deployment
    - Environment setup
    - Monitoring configuration

16. [Project Retrospective](documentation/16-project-retrospective.md)
    - Code review
    - Best practices
    - Future improvements

## Tech Stack

### Backend
- Node.js & Express.js
- TypeScript
- MongoDB with Mongoose
- Redis for caching
- JWT authentication

### Frontend
- React 18
- TypeScript
- Vite
- Tailwind CSS 4.0
- React Query
- React Router

### Development Tools
- ESLint & Prettier
- Jest for testing
- GitHub Actions
- Docker

### Infrastructure
- Render for hosting
- MongoDB Atlas
- Redis Cloud
- GitHub for version control

## Getting Started

### Prerequisites
```bash
# Required software
- Node.js (v18 or higher)
- MongoDB (v5 or higher)
- Redis (v6 or higher)
- Git
```

### Installation
```bash
# Clone repositories
git clone https://github.com/[organization]/url-redirector-api.git
git clone https://github.com/[organization]/url-redirector-client.git

# Install API dependencies
cd url-redirector-api
npm install

# Install client dependencies
cd ../url-redirector-client
npm install
```

### Development
```bash
# Start API server
cd url-redirector-api
npm run dev

# Start client development server
cd url-redirector-client
npm run dev
```

## Project Structure

### Backend Structure
```
url-redirector-api/
├── src/
│   ├── config/          # Configuration files
│   ├── controllers/     # Request handlers
│   ├── middleware/      # Custom middleware
│   ├── models/          # Database models
│   ├── routes/          # API routes
│   ├── services/        # Business logic
│   ├── utils/          # Helper functions
│   └── validators/     # Input validation
├── tests/              # Test files
└── infrastructure/     # Deployment configs
```

### Frontend Structure
```
url-redirector-client/
├── src/
│   ├── components/     # Reusable components
│   ├── hooks/         # Custom React hooks
│   ├── lib/           # Third-party integrations
│   ├── pages/         # Route components
│   ├── services/      # API services
│   ├── store/         # State management
│   └── utils/         # Helper functions
├── public/            # Static assets
└── tests/             # Test files
```

## Contributing
Please read our [Contributing Guide](CONTRIBUTING.md) for details on our code of conduct and the process for submitting pull requests.

## License
This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Acknowledgments
- [Express.js Documentation](https://expressjs.com/)
- [React Documentation](https://react.dev/)
- [MongoDB Documentation](https://docs.mongodb.com/)
- [Tailwind CSS Documentation](https://tailwindcss.com/docs)
