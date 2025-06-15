# 4. Foundation Configuration

This section covers the essential foundation setup for your React application following our architecture standards.

## 4.1 Configure TypeScript with Strict Mode

Configure TypeScript for maximum type safety and compatibility with our stack.

### TypeScript Configuration (`tsconfig.json`)

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "useDefineForClassFields": true,
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "skipLibCheck": true,
    
    /* Bundler mode */
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "isolatedModules": true,
    "moduleDetection": "force",
    "noEmit": true,
    "jsx": "react-jsx",
    
    /* Linting */
    "strict": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noFallthroughCasesInSwitch": true,
    
    /* Path mapping for FSD structure */
    "baseUrl": ".",
    "paths": {
      "@/*": ["./src/*"],
      "@/app/*": ["./src/app/*"],
      "@/pages/*": ["./src/pages/*"],
      "@/widgets/*": ["./src/widgets/*"],
      "@/features/*": ["./src/features/*"],
      "@/entities/*": ["./src/entities/*"],
      "@/shared/*": ["./src/shared/*"]
    }
  },
  "include": ["src"]
}
```

### App-specific TypeScript Configuration (`tsconfig.app.json`)

```json
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "composite": true,
    "tsBuildInfoFile": "./node_modules/.tmp/tsconfig.app.tsbuildinfo"
  },
  "include": ["src"]
}
```

**Key Features:**
- `isolatedModules: true` - Essential for esbuild compatibility
- `strict: true` - Enables all strict type checking
- Path mapping aligns with FSD architecture
- Modern ES2022 target for optimal performance

## 4.2 Set up Vite Configuration for Optimal Development

Configure Vite for the best development and build experience.

### Install Required Dependencies

```bash
npm install -D @types/node @tanstack/router-plugin
```

### Vite Configuration (`vite.config.ts`)

```typescript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react-swc'
import { TanStackRouterVite } from '@tanstack/router-plugin/vite'
import path from 'path'

export default defineConfig({
  plugins: [
    // TanStack Router plugin MUST come before React plugin
    TanStackRouterVite({ 
      target: 'react',
      autoCodeSplitting: true,
      experimental: {
        enableCodeSplitting: true
      }
    }),
    // Using SWC for better performance on large projects
    react(),
  ],
  
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
      '@/app': path.resolve(__dirname, './src/app'),
      '@/pages': path.resolve(__dirname, './src/pages'),
      '@/widgets': path.resolve(__dirname, './src/widgets'),
      '@/features': path.resolve(__dirname, './src/features'),
      '@/entities': path.resolve(__dirname, './src/entities'),
      '@/shared': path.resolve(__dirname, './src/shared'),
    },
  },
  
  server: {
    port: 3000,
    open: true,
    // Pre-warm frequently used files
    warmup: {
      clientFiles: ['./src/main.tsx', './src/app/**/*.tsx']
    }
  },
  
  build: {
    // Generate source maps for debugging
    sourcemap: true,
    // Optimize chunk size
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          router: ['@tanstack/react-router'],
          query: ['@tanstack/react-query'],
          ui: ['@radix-ui/react-slot', '@radix-ui/react-dialog']
        }
      }
    }
  },
  
  // Optimize dependency pre-bundling
  optimizeDeps: {
    include: [
      'react', 
      'react-dom', 
      '@tanstack/react-router',
      '@tanstack/react-query',
      'zustand'
    ]
  }
})
```

**Key Optimizations:**
- `react-swc` plugin for faster builds and HMR
- Manual chunk splitting for optimal loading
- Dependency pre-bundling for faster dev server startup
- Source maps enabled for production debugging

## 4.3 Create App Layer (Providers, Router Setup)

Set up the application layer with all required providers and routing.

### Install Core Dependencies

```bash
# Core routing and state management
npm install @tanstack/react-router @tanstack/react-router-devtools
npm install @tanstack/react-query @tanstack/react-query-devtools
npm install zustand

# Authentication and API
npm install axios
npm install react-hook-form @hookform/resolvers zod

# UI and styling
npm install tailwindcss @tailwindcss/vite
npx shadcn@latest init
```

### Main Entry Point (`src/main.tsx`)

```tsx
import { StrictMode } from 'react'
import { createRoot } from 'react-dom/client'
import { RouterProvider, createRouter } from '@tanstack/react-router'

// Import the generated route tree
import { routeTree } from './routeTree.gen'

// Create a new router instance
const router = createRouter({ 
  routeTree,
  context: {
    // Add any global context here
  }
})

// Register the router instance for type safety
declare module '@tanstack/react-router' {
  interface Register {
    router: typeof router
  }
}

const rootElement = document.getElementById('root')!

if (!rootElement.innerHTML) {
  const root = createRoot(rootElement)
  root.render(
    <StrictMode>
      <RouterProvider router={router} />
    </StrictMode>,
  )
}
```

### Root Route Configuration (`src/routes/__root.tsx`)

```tsx
import { createRootRoute, Outlet } from '@tanstack/react-router'
import { TanStackRouterDevtools } from '@tanstack/react-router-devtools'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'
import { useState } from 'react'

import { Toaster } from '@/shared/ui/toaster'
import { ThemeProvider } from '@/shared/providers/theme-provider'
import '@/app/globals.css'

export const Route = createRootRoute({
  component: RootComponent,
})

function RootComponent() {
  // Create stable QueryClient instance
  const [queryClient] = useState(
    () => new QueryClient({
      defaultOptions: {
        queries: {
          // Prevent immediate refetching on client hydration
          staleTime: 60 * 1000, // 1 minute
          // Retry failed requests 3 times with exponential backoff
          retry: 3,
          // Enable structural sharing for better performance
          refetchOnWindowFocus: false,
        },
        mutations: {
          // Show error toasts for failed mutations by default
          onError: (error) => {
            console.error('Mutation failed:', error)
          }
        }
      },
    }),
  )

  return (
    <QueryClientProvider client={queryClient}>
      <ThemeProvider defaultTheme="system" storageKey="app-theme">
        <div className="min-h-screen bg-background font-sans antialiased">
          <Outlet />
        </div>
        <Toaster />
        
        {/* Development tools */}
        {process.env.NODE_ENV === 'development' && (
          <>
            <TanStackRouterDevtools />
            <ReactQueryDevtools initialIsOpen={false} />
          </>
        )}
      </ThemeProvider>
    </QueryClientProvider>
  )
}
```

### App-Level Providers (`src/app/providers.tsx`)

```tsx
import { ReactNode } from 'react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'

import { ThemeProvider } from '@/shared/providers/theme-provider'
import { AuthProvider } from '@/shared/providers/auth-provider'
import { Toaster } from '@/shared/ui/toaster'

interface ProvidersProps {
  children: ReactNode
  queryClient: QueryClient
}

export function Providers({ children, queryClient }: ProvidersProps) {
  return (
    <QueryClientProvider client={queryClient}>
      <ThemeProvider defaultTheme="system" storageKey="app-theme">
        <AuthProvider>
          {children}
          <Toaster />
          
          {process.env.NODE_ENV === 'development' && (
            <ReactQueryDevtools initialIsOpen={false} />
          )}
        </AuthProvider>
      </ThemeProvider>
    </QueryClientProvider>
  )
}
```

## 4.4 Initialize Shared Layer (UI Components, Utilities, API Config)

Set up the shared layer with reusable components, utilities, and API configuration.

### Global Styles (`src/app/globals.css`)

```css
@import "tailwindcss";

@layer base {
  :root {
    --background: 0 0% 100%;
    --foreground: 222.2 84% 4.9%;
    --card: 0 0% 100%;
    --card-foreground: 222.2 84% 4.9%;
    --popover: 0 0% 100%;
    --popover-foreground: 222.2 84% 4.9%;
    --primary: 222.2 47.4% 11.2%;
    --primary-foreground: 210 40% 98%;
    --secondary: 210 40% 96%;
    --secondary-foreground: 222.2 47.4% 11.2%;
    --muted: 210 40% 96%;
    --muted-foreground: 215.4 16.3% 46.9%;
    --accent: 210 40% 96%;
    --accent-foreground: 222.2 47.4% 11.2%;
    --destructive: 0 84.2% 60.2%;
    --destructive-foreground: 210 40% 98%;
    --border: 214.3 31.8% 91.4%;
    --input: 214.3 31.8% 91.4%;
    --ring: 222.2 47.4% 11.2%;
    --radius: 0.5rem;
  }

  .dark {
    --background: 222.2 84% 4.9%;
    --foreground: 210 40% 98%;
    --card: 222.2 84% 4.9%;
    --card-foreground: 210 40% 98%;
    --popover: 222.2 84% 4.9%;
    --popover-foreground: 210 40% 98%;
    --primary: 210 40% 98%;
    --primary-foreground: 222.2 47.4% 11.2%;
    --secondary: 217.2 32.6% 17.5%;
    --secondary-foreground: 210 40% 98%;
    --muted: 217.2 32.6% 17.5%;
    --muted-foreground: 215 20.2% 65.1%;
    --accent: 217.2 32.6% 17.5%;
    --accent-foreground: 210 40% 98%;
    --destructive: 0 62.8% 30.6%;
    --destructive-foreground: 210 40% 98%;
    --border: 217.2 32.6% 17.5%;
    --input: 217.2 32.6% 17.5%;
    --ring: 212.7 26.8% 83.9%;
  }
}

@layer base {
  * {
    @apply border-border;
  }
  body {
    @apply bg-background text-foreground;
  }
}
```

### API Configuration (`src/shared/api/axios-instance.ts`)

```typescript
import axios from 'axios'

// Create the main Axios instance
export const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || '/api',
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
})

// Request interceptor for authentication
api.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('auth-token')
    if (token) {
      config.headers.Authorization = `Bearer ${token}`
    }
    return config
  },
  (error) => Promise.reject(error)
)

// Response interceptor for error handling
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Clear invalid token and redirect to login
      localStorage.removeItem('auth-token')
      window.location.href = '/login'
    }
    return Promise.reject(error)
  }
)

export default api
```

### Utilities (`src/shared/lib/utils.ts`)

```typescript
import { type ClassValue, clsx } from 'clsx'
import { twMerge } from 'tailwind-merge'

/**
 * Merge classes with tailwind-merge for optimal CSS
 */
export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}

/**
 * Format date to locale string
 */
export function formatDate(date: Date | string | number): string {
  return new Intl.DateTimeFormat('en-US', {
    year: 'numeric',
    month: 'long',
    day: 'numeric',
  }).format(new Date(date))
}

/**
 * Sleep utility for development/testing
 */
export function sleep(ms: number): Promise<void> {
  return new Promise(resolve => setTimeout(resolve, ms))
}

/**
 * Debounce function calls
 */
export function debounce<T extends (...args: any[]) => any>(
  func: T,
  wait: number
): (...args: Parameters<T>) => void {
  let timeout: NodeJS.Timeout
  return (...args: Parameters<T>) => {
    clearTimeout(timeout)
    timeout = setTimeout(() => func(...args), wait)
  }
}
```

### Client State Store (`src/shared/stores/ui-store.ts`)

```typescript
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface UIState {
  // Sidebar state
  isSidebarOpen: boolean
  toggleSidebar: () => void
  setSidebarOpen: (open: boolean) => void
  
  // Modal state
  activeModal: string | null
  openModal: (modalId: string) => void
  closeModal: () => void
  
  // Loading states
  isLoading: boolean
  setLoading: (loading: boolean) => void
}

export const useUIStore = create<UIState>()(
  persist(
    (set) => ({
      // Sidebar
      isSidebarOpen: true,
      toggleSidebar: () => set((state) => ({ 
        isSidebarOpen: !state.isSidebarOpen 
      })),
      setSidebarOpen: (open) => set({ isSidebarOpen: open }),
      
      // Modal
      activeModal: null,
      openModal: (modalId) => set({ activeModal: modalId }),
      closeModal: () => set({ activeModal: null }),
      
      // Loading
      isLoading: false,
      setLoading: (loading) => set({ isLoading: loading }),
    }),
    {
      name: 'ui-store',
      // Only persist sidebar state
      partialize: (state) => ({ isSidebarOpen: state.isSidebarOpen }),
    }
  )
)
```

### Authentication Store (`src/shared/stores/auth-store.ts`)

```typescript
import { create } from 'zustand'
import { persist } from 'zustand/middleware'

interface User {
  id: string
  email: string
  name: string
  avatar?: string
}

interface AuthState {
  user: User | null
  token: string | null
  isAuthenticated: boolean
  isLoading: boolean
  
  // Actions
  setAuth: (user: User, token: string) => void
  clearAuth: () => void
  setLoading: (loading: boolean) => void
  updateUser: (updates: Partial<User>) => void
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set, get) => ({
      user: null,
      token: null,
      isAuthenticated: false,
      isLoading: false,
      
      setAuth: (user, token) => {
        localStorage.setItem('auth-token', token)
        set({
          user,
          token,
          isAuthenticated: true,
          isLoading: false
        })
      },
      
      clearAuth: () => {
        localStorage.removeItem('auth-token')
        set({
          user: null,
          token: null,
          isAuthenticated: false,
          isLoading: false
        })
      },
      
      setLoading: (loading) => set({ isLoading: loading }),
      
      updateUser: (updates) => {
        const currentUser = get().user
        if (currentUser) {
          set({ user: { ...currentUser, ...updates } })
        }
      },
    }),
    {
      name: 'auth-store',
      partialize: (state) => ({
        user: state.user,
        token: state.token,
        isAuthenticated: state.isAuthenticated
      })
    }
  )
)
```

### Environment Variables (`.env.example`)

```bash
# API Configuration
VITE_API_BASE_URL=http://localhost:8000/api

# Authentication
VITE_AUTH_GOOGLE_CLIENT_ID=your-google-client-id

# Feature Flags
VITE_ENABLE_DEVTOOLS=true
VITE_ENABLE_ANALYTICS=false
```

**Next Steps:** Your foundation is now configured! Continue to [Development Tooling](./05-development-tooling.md) to set up code quality tools and testing frameworks.

---

**Next Steps**: Continue to [Development Tooling](./05-development-tooling.md) for code quality setup.