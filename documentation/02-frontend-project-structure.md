# Module 2: Frontend Project Structure Setup

## Overview
In this module, we'll set up a React frontend with TypeScript, Vite, and Tailwind CSS v4, following the official installation guide.

## Prerequisites
- Node.js (v18 or higher)
- Git
- Code editor (VSCode recommended)
- Completed Module 0

## Quick Setup Steps

### 1. Create Vite Project
```bash
# Navigate to your cloned repository
cd url-redirector-client

# Create new Vite project
npm create vite@latest . -- --template react-ts
```

### 2. Install Tailwind CSS v4
```bash
# Install Tailwind CSS and Vite plugin
npm install tailwindcss @tailwindcss/vite
```

### 3. Configure Vite
Update `vite.config.ts`:
```typescript
import { defineConfig } from 'vite'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
  plugins: [
    tailwindcss(),
  ],
})
```

### 4. Import Tailwind CSS
Create `src/styles.css`:
```css
@import "tailwindcss";
```

### 5. Update HTML Template
Update `index.html`:
```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <link href="/src/styles.css" rel="stylesheet">
</head>
<body>
  <div id="root"></div>
  <script type="module" src="/src/main.tsx"></script>
</body>
</html>
```

### 6. Update Entry Point
Update `src/main.tsx`:
```typescript
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App.tsx'
import './styles.css'

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
```

### 7. Add Additional Dependencies
```bash
# Install routing and API dependencies
npm install react-router-dom @tanstack/react-query axios

# Install form handling dependencies
npm install react-hook-form zod @hookform/resolvers

# Install UI utilities
npm install clsx lucide-react

# Install development dependencies
npm install -D @types/node prettier eslint-config-prettier eslint-plugin-prettier
npm install -D @typescript-eslint/eslint-plugin @typescript-eslint/parser
```

### 8. Test the Setup
Update `src/App.tsx`:
```typescript
function App() {
  return (
    <div className="min-h-screen bg-gray-100">
      <div className="max-w-7xl mx-auto py-6 sm:px-6 lg:px-8">
        <h1 className="text-3xl font-bold underline">
          Hello world!
        </h1>
      </div>
    </div>
  )
}

export default App
```

### 9. Run the Development Server
```bash
npm run dev
```

## Project Structure
```
url-redirector-client/
├── src/
│   ├── components/
│   │   ├── common/
│   │   ├── layout/
│   │   └── ui/
│   ├── hooks/
│   ├── lib/
│   ├── pages/
│   ├── services/
│   ├── types/
│   ├── utils/
│   ├── App.tsx
│   ├── main.tsx
│   └── styles.css
├── public/
├── index.html
├── package.json
├── vite.config.ts
└── tsconfig.json
```

## Key Differences from Previous Versions
- No need for PostCSS configuration
- Direct import of Tailwind CSS using `@import "tailwindcss"`
- Use of `@tailwindcss/vite` plugin instead of PostCSS plugin
- Simplified configuration without `tailwind.config.js`
