# 7. Routing Setup

This guide covers the complete setup and configuration of TanStack Router for 100% type-safe routing in your React application.

## Installation & Dependencies

Install the required packages:

```bash
pnpm add @tanstack/react-router @tanstack/react-router-devtools
pnpm add -D @tanstack/router-plugin
```

## Vite Configuration

Configure the TanStack Router plugin in your `vite.config.ts`:

```typescript
// vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import { TanStackRouterVite } from '@tanstack/router-plugin/vite'

export default defineConfig({
  plugins: [
    // IMPORTANT: TanStack Router plugin must come before React plugin
    TanStackRouterVite({ 
      target: 'react',
      autoCodeSplitting: true 
    }),
    react(),
  ],
})
```

## File-Based Routing Structure

TanStack Router uses **file-based routing** as the recommended approach. Create routes in `src/routes/` directory following these conventions:

### Basic File Structure

```
src/routes/
├── __root.tsx          # Root layout (required)
├── index.tsx           # Home page (/)
├── about.tsx           # About page (/about)
├── login.tsx           # Login page (/login)
├── dashboard/
│   ├── index.tsx       # Dashboard home (/dashboard)
│   ├── profile.tsx     # Profile page (/dashboard/profile)
│   └── settings.tsx    # Settings page (/dashboard/settings)
└── posts/
    ├── index.tsx       # Posts list (/posts)
    ├── $postId.tsx     # Single post (/posts/123)
    └── $postId.edit.tsx # Edit post (/posts/123/edit)
```

### File Naming Conventions

| Pattern | Description | Example |
|---------|-------------|---------|
| `__root.tsx` | Root layout (required) | Application shell |
| `index.tsx` | Exact route match | `/posts` for `posts/index.tsx` |
| `$param.tsx` | Dynamic parameter | `$postId.tsx` → `/posts/123` |
| `_layout.tsx` | Pathless layout route | Wraps children without URL segment |
| `about.tsx` | Static route | `/about` |
| `posts.$postId.edit.tsx` | Flat file structure | `/posts/123/edit` |

## Root Route Setup

Create the root route that provides the application shell:

```tsx
// src/routes/__root.tsx
import { createRootRoute, Link, Outlet } from '@tanstack/react-router'
import { TanStackRouterDevtools } from '@tanstack/react-router-devtools'

export const Route = createRootRoute({
  component: () => (
    <>
      <div className="p-2 flex gap-2">
        <Link to="/" className="[&.active]:font-bold">
          Home
        </Link>
        <Link to="/about" className="[&.active]:font-bold">
          About
        </Link>
        <Link to="/dashboard" className="[&.active]:font-bold">
          Dashboard
        </Link>
      </div>
      <hr />
      <Outlet />
      {process.env.NODE_ENV === 'development' && <TanStackRouterDevtools />}
    </>
  ),
})
```

## Basic Route Examples

### Static Route

```tsx
// src/routes/about.tsx
import { createLazyFileRoute } from '@tanstack/react-router'

export const Route = createLazyFileRoute('/about')({
  component: About,
})

function About() {
  return (
    <div className="p-4">
      <h1 className="text-2xl font-bold">About Us</h1>
      <p>Welcome to our application!</p>
    </div>
  )
}
```

### Dynamic Route with Parameters

```tsx
// src/routes/posts/$postId.tsx
import { createLazyFileRoute } from '@tanstack/react-router'
import { z } from 'zod'

const postSearchSchema = z.object({
  tab: z.enum(['overview', 'comments']).optional(),
})

export const Route = createLazyFileRoute('/posts/$postId')({
  validateSearch: postSearchSchema,
  component: PostDetail,
})

function PostDetail() {
  const { postId } = Route.useParams()
  const { tab } = Route.useSearch()
  
  return (
    <div className="p-4">
      <h1>Post {postId}</h1>
      <div>Current tab: {tab || 'overview'}</div>
    </div>
  )
}
```

## Router Instance Setup

Create and configure the router in your main application file:

```tsx
// src/main.tsx
import { StrictMode } from 'react'
import ReactDOM from 'react-dom/client'
import { RouterProvider, createRouter } from '@tanstack/react-router'

// Import the generated route tree
import { routeTree } from './routeTree.gen'

// Create the router instance
const router = createRouter({ routeTree })

// Register the router for type safety
declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router
  }
}

// Render the application
const rootElement = document.getElementById('root')!
if (!rootElement.innerHTML) {
  const root = ReactDOM.createRoot(rootElement)
  root.render(
    <StrictMode>
      <RouterProvider router={router} />
    </StrictMode>,
  )
}
```

## Type-Safe Navigation Utilities

### Link Component

Use the `Link` component for type-safe navigation:

```tsx
import { Link } from '@tanstack/react-router'

// Basic navigation
<Link to="/about">About</Link>

// Navigation with parameters
<Link to="/posts/$postId" params={{ postId: '123' }}>
  View Post
</Link>

// Navigation with search parameters
<Link 
  to="/posts" 
  search={{ 
    filter: 'published',
    page: 1 
  }}
>
  Published Posts
</Link>

// Navigation with state
<Link 
  to="/posts/$postId" 
  params={{ postId: '123' }}
  state={{ from: 'homepage' }}
>
  View Post
</Link>
```

### Navigation Hooks

#### useNavigate Hook

```tsx
import { useNavigate } from '@tanstack/react-router'

function MyComponent() {
  const navigate = useNavigate()

  const handleSubmit = async (data: FormData) => {
    await savePost(data)
    
    // Type-safe programmatic navigation
    navigate({ 
      to: '/posts/$postId', 
      params: { postId: data.id },
      search: { tab: 'overview' }
    })
  }

  return <form onSubmit={handleSubmit}>...</form>
}
```

#### useParams and useSearch Hooks

```tsx
import { useParams, useSearch } from '@tanstack/react-router'

function PostComponent() {
  // Type-safe parameter access
  const { postId } = useParams({ from: '/posts/$postId' })
  
  // Type-safe search parameter access
  const { filter, page } = useSearch({ from: '/posts' })
  
  return <div>Post {postId}, Filter: {filter}, Page: {page}</div>
}
```

## Route Guards & Protected Routes

### Authentication Context Setup

First, set up authentication context for your router:

```tsx
// src/routes/__root.tsx
import { createRootRouteWithContext } from '@tanstack/react-router'
import { Outlet } from '@tanstack/react-router'

interface AuthContext {
  isAuthenticated: boolean
  user: User | null
}

interface RouterContext {
  auth: AuthContext
}

export const Route = createRootRouteWithContext<RouterContext>()({
  component: () => <Outlet />,
})
```

### Router Configuration with Context

```tsx
// src/router.tsx
import { createRouter } from '@tanstack/react-router'
import { routeTree } from './routeTree.gen'

export const router = createRouter({
  routeTree,
  context: {
    auth: undefined!, // Will be provided by AuthProvider
  },
})
```

### App Setup with Authentication

```tsx
// src/App.tsx
import { RouterProvider } from '@tanstack/react-router'
import { AuthProvider, useAuth } from './shared/auth'
import { router } from './router'

function InnerApp() {
  const auth = useAuth()
  return <RouterProvider router={router} context={{ auth }} />
}

export function App() {
  return (
    <AuthProvider>
      <InnerApp />
    </AuthProvider>
  )
}
```

### Protected Route Implementation

#### Method 1: Redirect-Based Protection

```tsx
// src/routes/_authenticated.tsx
import { createFileRoute, redirect } from '@tanstack/react-router'

export const Route = createFileRoute('/_authenticated')({
  beforeLoad: async ({ context, location }) => {
    if (!context.auth.isAuthenticated) {
      throw redirect({
        to: '/login',
        search: {
          redirect: location.href,
        },
      })
    }
  },
})
```

#### Method 2: Inline Authentication Check

```tsx
// src/routes/_authenticated.tsx
import { createFileRoute, Outlet } from '@tanstack/react-router'
import { LoginForm } from '../shared/ui/login-form'

export const Route = createFileRoute('/_authenticated')({
  component: () => {
    const { auth } = Route.useRouteContext()
    
    if (!auth.isAuthenticated) {
      return <LoginForm />
    }

    return <Outlet />
  },
})
```

### Nested Protected Routes

```tsx
// src/routes/_authenticated/dashboard.tsx
import { createLazyFileRoute } from '@tanstack/react-router'

export const Route = createLazyFileRoute('/_authenticated/dashboard')({
  component: Dashboard,
})

function Dashboard() {
  const { auth } = Route.useRouteContext()
  
  return (
    <div className="p-4">
      <h1>Welcome, {auth.user?.name}!</h1>
      <p>This is your dashboard.</p>
    </div>
  )
}
```

## Advanced Patterns

### Search Parameter Validation

Use Zod schemas for type-safe search parameters:

```tsx
// src/routes/posts/index.tsx
import { createLazyFileRoute } from '@tanstack/react-router'
import { z } from 'zod'

const postsSearchSchema = z.object({
  page: z.number().min(1).default(1),
  filter: z.enum(['all', 'published', 'draft']).default('all'),
  sort: z.enum(['date', 'title', 'author']).default('date'),
  order: z.enum(['asc', 'desc']).default('desc'),
})

export const Route = createLazyFileRoute('/posts/')({
  validateSearch: postsSearchSchema,
  component: PostsList,
})

function PostsList() {
  const { page, filter, sort, order } = Route.useSearch()
  
  // All search params are now type-safe and validated
  return <div>Posts page {page}</div>
}
```

### Route-Specific Error Boundaries

```tsx
// src/routes/posts/$postId.tsx
import { createLazyFileRoute, ErrorComponent } from '@tanstack/react-router'

export const Route = createLazyFileRoute('/posts/$postId')({
  component: PostDetail,
  errorComponent: ({ error, reset }) => (
    <div className="p-4 text-center">
      <h2 className="text-xl font-bold text-red-600">
        Error loading post
      </h2>
      <p className="text-gray-600">{error.message}</p>
      <button 
        onClick={() => reset()}
        className="mt-4 px-4 py-2 bg-blue-500 text-white rounded"
      >
        Try Again
      </button>
    </div>
  ),
})
```

## Git Configuration

Add the generated route tree to your `.gitignore`:

```gitignore
# TanStack Router
src/routeTree.gen.ts
```

## ESLint Configuration

Configure ESLint to work with TanStack Router:

```json
{
  "extends": ["@tanstack/eslint-plugin-router/recommended"]
}
```

---

**Next Steps**: Continue to [State Management](./08-state-management.md) for data management setup.