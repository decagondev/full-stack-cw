# Module 0: Project Setup and Repository Creation

## Overview
In this module, we'll set up the foundation for our URL Redirector project by creating and configuring two GitHub repositories: one for the backend API and one for the frontend application.

## Prerequisites
- GitHub account
- Git installed locally
- Code editor (VSCode recommended)

## Steps

### 1. Create GitHub Organization (Optional but Recommended)
1. Go to GitHub.com and click the '+' icon in the top right
2. Select "New organization"
3. Choose the free plan
4. Name it something like `url-redirector-project`
5. Add a description: "A modern URL redirector application with user management and analytics"

### 2. Backend Repository Setup

1. Create the repository
   ```bash
   # Repository name: url-redirector-api
   # Description: Backend API for URL Redirector built with Express.js and MongoDB
   ```

2. Initialize with basic files:
   - README.md
   - .gitignore (Node)
   - MIT License

3. Clone and set up basic structure:
   ```bash
   git clone https://github.com/[organization]/url-redirector-api.git
   cd url-redirector-api
   ```

4. Create initial README.md content:
   ```markdown
   # URL Redirector API

   Backend API for the URL Redirector project built with Express.js and MongoDB.

   ## Tech Stack
   - Node.js
   - Express.js
   - MongoDB
   - TypeScript
   - JWT Authentication

   ## Getting Started
   Instructions for setup and development will be added as the project progresses.

   ## Features (Planned)
   - User authentication (JWT)
   - URL management
   - Click tracking
   - Analytics
   - Role-based access control
   ```

### 3. Frontend Repository Setup

1. Create the repository
   ```bash
   # Repository name: url-redirector-client
   # Description: Frontend application for URL Redirector built with React, TypeScript, and Vite
   ```

2. Initialize with basic files:
   - README.md
   - .gitignore (Node)
   - MIT License

3. Clone and set up basic structure:
   ```bash
   git clone https://github.com/[organization]/url-redirector-client.git
   cd url-redirector-client
   ```

4. Create initial README.md content:
   ```markdown
   # URL Redirector Client

   Frontend application for the URL Redirector project built with React, TypeScript, and Vite.

   ## Tech Stack
   - React 18
   - TypeScript
   - Vite
   - Tailwind CSS 4.0
   - React Router

   ## Getting Started
   Instructions for setup and development will be added as the project progresses.

   ## Features (Planned)
   - User authentication
   - URL management dashboard
   - Analytics dashboard
   - Admin panel
   - Role-based access control
   ```

### 4. Project Board Setup (Optional)

1. Create a project board in your GitHub organization
2. Set up columns:
   - Backlog
   - To Do
   - In Progress
   - Review
   - Done

3. Add initial issues for upcoming modules:
   - Backend setup and folder structure
   - Frontend setup and folder structure
   - MongoDB Atlas setup
   - Basic API implementation
   - etc.

### 5. Repository Settings

For both repositories:

1. Configure branch protection rules:
   - Require pull request reviews before merging
   - Require status checks to pass before merging
   - Include administrators in these restrictions

2. Set up labels for issues:
   - bug
   - enhancement
   - documentation
   - good first issue
   - help wanted

3. Configure repository settings:
   - Enable issues
   - Enable projects
   - Enable wiki if needed

## Next Steps

With the repositories set up, we're ready to:
1. Clone the backend repository and set up the Express.js project structure (Module 1)
2. Clone the frontend repository and set up the Vite/React project structure (Module 2)

## Common Issues and Solutions

1. Git configuration issues:
   ```bash
   # Set your Git username
   git config --global user.name "Your Name"
   
   # Set your Git email
   git config --global user.email "your.email@example.com"
   ```

2. Permission issues:
   - Ensure you're added as a collaborator if working in an organization
   - Check that your SSH keys are properly set up if using SSH

3. .gitignore mistakes:
   - Double-check that node_modules and .env files are properly ignored
   - Review .gitignore before initial commit

## Best Practices

1. Use clear, descriptive commit messages
2. Keep repository descriptions up to date
3. Document setup steps in README files
4. Use consistent naming conventions
5. Set up proper .gitignore files from the start

## Validation Checklist

- [ ] Both repositories created successfully
- [ ] README files initialized with basic information
- [ ] .gitignore files properly configured
- [ ] Repositories cloned locally
- [ ] Branch protection rules configured
- [ ] Labels and project board set up (if using)
- [ ] Collaborators added (if working in a team)