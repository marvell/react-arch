# 8. State Management Foundation

This guide implements our architectural principle of **strict separation of concerns** between server state and client state. We use TanStack Query exclusively for server data and Zustand exclusively for client-side UI state.

## 8.1. TanStack Query Setup

### Installation

```bash
# Core library
npm install @tanstack/react-query

# DevTools for development
npm install -D @tanstack/react-query-devtools

# ESLint plugin to catch common mistakes
npm install -D @tanstack/eslint-plugin-query
```

### Query Client Configuration

Create the query client with production-ready defaults:

```typescript
// src/shared/api/query-client.ts
import { QueryClient } from '@tanstack/react-query'

export function createQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: {
        // Prevent unnecessary refetches
        staleTime: 60 * 1000, // 1 minute
        
        // Retry configuration
        retry: (failureCount, error) => {
          // Don't retry on 4xx errors
          if (error?.response?.status >= 400 && error?.response?.status < 500) {
            return false
          }
          return failureCount < 3
        },
        
        // Retry delay with exponential backoff
        retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
        
        // Refetch on window focus in development only
        refetchOnWindowFocus: process.env.NODE_ENV === 'development',
      },
      mutations: {
        // Retry mutations once on network errors
        retry: (failureCount, error) => {
          if (error?.code === 'NETWORK_ERROR' && failureCount < 1) {
            return true
          }
          return false
        },
      },
    },
  })
}
```

### Provider Setup

Set up the QueryClientProvider in your app root:

```typescript
// src/app/providers/query-provider.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'
import { createQueryClient } from '@/shared/api/query-client'

// Create a single instance for the app
const queryClient = createQueryClient()

interface QueryProviderProps {
  children: React.ReactNode
}

export function QueryProvider({ children }: QueryProviderProps) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools 
        initialIsOpen={false}
        position="bottom-right"
      />
    </QueryClientProvider>
  )
}
```

### ESLint Configuration

Add the TanStack Query ESLint plugin to catch common mistakes:

```json
// .eslintrc.json
{
  "extends": [
    // ... other configs
    "@tanstack/eslint-plugin-query/recommended"
  ]
}
```

## 8.2. Zustand Setup

### Installation

```bash
npm install zustand
```

### Basic Store Creation

Create stores in the appropriate FSD layer (usually `shared/store` for global UI state):

```typescript
// src/shared/store/ui-store.ts
import { create } from 'zustand'
import { devtools } from 'zustand/middleware'

interface UIState {
  isSidebarOpen: boolean
  theme: 'light' | 'dark'
  toggleSidebar: () => void
  setTheme: (theme: 'light' | 'dark') => void
}

export const useUIStore = create<UIState>()(
  devtools(
    (set) => ({
      isSidebarOpen: false,
      theme: 'light',
      
      toggleSidebar: () =>
        set((state) => ({ isSidebarOpen: !state.isSidebarOpen }), false, 'ui/toggleSidebar'),
      
      setTheme: (theme) =>
        set({ theme }, false, 'ui/setTheme'),
    }),
    { name: 'ui-store' }
  )
)
```

### Persistent Store Pattern

For state that needs persistence (auth tokens, user preferences):

```typescript
// src/shared/store/auth-store.ts
import { create } from 'zustand'
import { persist, devtools } from 'zustand/middleware'

interface AuthState {
  token: string | null
  user: User | null
  isAuthenticated: boolean
  setAuth: (token: string, user: User) => void
  clearAuth: () => void
}

export const useAuthStore = create<AuthState>()(
  devtools(
    persist(
      (set) => ({
        token: null,
        user: null,
        isAuthenticated: false,
        
        setAuth: (token, user) =>
          set({ token, user, isAuthenticated: true }, false, 'auth/setAuth'),
        
        clearAuth: () =>
          set({ token: null, user: null, isAuthenticated: false }, false, 'auth/clearAuth'),
      }),
      {
        name: 'auth-storage',
        // Only persist essential auth data
        partialize: (state) => ({
          token: state.token,
          user: state.user,
          isAuthenticated: state.isAuthenticated,
        }),
      }
    ),
    { name: 'auth-store' }
  )
)
```

### Store Organization Pattern

For complex stores, use slicing pattern:

```typescript
// src/shared/store/app-store.ts
import { create } from 'zustand'
import { devtools } from 'zustand/middleware'

// Slice interfaces
interface NavigationSlice {
  currentRoute: string
  setCurrentRoute: (route: string) => void
}

interface NotificationSlice {
  notifications: Notification[]
  addNotification: (notification: Notification) => void
  removeNotification: (id: string) => void
}

// Slice creators
const createNavigationSlice = (set: any): NavigationSlice => ({
  currentRoute: '/',
  setCurrentRoute: (route) =>
    set({ currentRoute: route }, false, 'nav/setCurrentRoute'),
})

const createNotificationSlice = (set: any): NotificationSlice => ({
  notifications: [],
  addNotification: (notification) =>
    set((state: any) => ({
      notifications: [...state.notifications, notification]
    }), false, 'notifications/add'),
  removeNotification: (id) =>
    set((state: any) => ({
      notifications: state.notifications.filter((n: any) => n.id !== id)
    }), false, 'notifications/remove'),
})

// Combined store
type AppState = NavigationSlice & NotificationSlice

export const useAppStore = create<AppState>()(
  devtools(
    (set) => ({
      ...createNavigationSlice(set),
      ...createNotificationSlice(set),
    }),
    { name: 'app-store' }
  )
)
```

## 8.3. DevTools Integration

### TanStack Query DevTools

The DevTools are already included in our QueryProvider setup above. Key features:

- **Query Inspector**: View all queries, their status, and cached data
- **Mutation Tracker**: Monitor ongoing mutations
- **Cache Explorer**: Inspect query cache structure
- **Network Timeline**: Visualize query timing and dependencies

Access via the floating toggle button (bottom-right) or press `Ctrl+Shift+Q`.

### Zustand DevTools

Zustand integrates with Redux DevTools for powerful debugging:

1. **Install Redux DevTools Extension** in your browser
2. **Wrap stores with `devtools` middleware** (shown in examples above)
3. **Use meaningful action names** in `set()` calls:

```typescript
// Good: descriptive action names
set({ count: count + 1 }, false, 'counter/increment')
set({ user: null }, false, 'auth/logout')

// Bad: generic names
set({ count: count + 1 }) // defaults to 'anonymous'
```

DevTools features:
- **Time Travel**: Step through state changes
- **Action Inspector**: See what triggered each state change
- **State Diff**: Compare before/after state
- **Persistence**: Export/import state snapshots

## 8.4. Usage Patterns & Best Practices

### Server State with TanStack Query

```typescript
// src/entities/user/api/user-queries.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query'
import { userApi } from './user-api'

// Query hooks
export const useUser = (id: string) =>
  useQuery({
    queryKey: ['user', id],
    queryFn: () => userApi.getUser(id),
    enabled: !!id, // Only run if id exists
  })

export const useUsers = (filters?: UserFilters) =>
  useQuery({
    queryKey: ['users', filters],
    queryFn: () => userApi.getUsers(filters),
    staleTime: 5 * 60 * 1000, // 5 minutes for list data
  })

// Mutation hooks
export const useUpdateUser = () => {
  const queryClient = useQueryClient()
  
  return useMutation({
    mutationFn: userApi.updateUser,
    onSuccess: (updatedUser) => {
      // Update user in cache
      queryClient.setQueryData(['user', updatedUser.id], updatedUser)
      
      // Invalidate user list to refetch
      queryClient.invalidateQueries({ queryKey: ['users'] })
    },
  })
}
```

### Client State with Zustand

```typescript
// In a component
function Header() {
  const { isSidebarOpen, toggleSidebar } = useUIStore()
  
  return (
    <header>
      <button onClick={toggleSidebar}>
        {isSidebarOpen ? 'Close' : 'Open'} Sidebar
      </button>
    </header>
  )
}

// Selective subscriptions to prevent unnecessary re-renders
function ThemeToggle() {
  const theme = useUIStore(state => state.theme)
  const setTheme = useUIStore(state => state.setTheme)
  
  return (
    <button onClick={() => setTheme(theme === 'light' ? 'dark' : 'light')}>
      {theme} mode
    </button>
  )
}
```

### Integration with Axios Interceptors

Set up automatic token attachment:

```typescript
// src/shared/api/axios-instance.ts
import axios from 'axios'
import { useAuthStore } from '@/shared/store/auth-store'

export const apiClient = axios.create({
  baseURL: process.env.VITE_API_URL,
})

// Request interceptor to add auth token
apiClient.interceptors.request.use((config) => {
  const token = useAuthStore.getState().token
  if (token) {
    config.headers.Authorization = `Bearer ${token}`
  }
  return config
})

// Response interceptor for auth errors
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      useAuthStore.getState().clearAuth()
      window.location.href = '/login'
    }
    return Promise.reject(error)
  }
)
```

## 8.5. Testing State Management

### Testing TanStack Query

```typescript
// src/entities/user/api/__tests__/user-queries.test.tsx
import { renderHook, waitFor } from '@testing-library/react'
import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { useUser } from '../user-queries'

const createTestQueryClient = () =>
  new QueryClient({
    defaultOptions: { queries: { retry: false } },
  })

test('useUser fetches user data', async () => {
  const queryClient = createTestQueryClient()
  
  const { result } = renderHook(() => useUser('1'), {
    wrapper: ({ children }) => (
      <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
    ),
  })

  await waitFor(() => expect(result.current.isSuccess).toBe(true))
  expect(result.current.data).toEqual({ id: '1', name: 'John' })
})
```

### Testing Zustand Stores

```typescript
// src/shared/store/__tests__/ui-store.test.ts
import { useUIStore } from '../ui-store'

test('toggleSidebar changes sidebar state', () => {
  const { toggleSidebar, isSidebarOpen } = useUIStore.getState()
  
  expect(isSidebarOpen).toBe(false)
  
  toggleSidebar()
  expect(useUIStore.getState().isSidebarOpen).toBe(true)
  
  toggleSidebar()
  expect(useUIStore.getState().isSidebarOpen).toBe(false)
})
```

## ✅ Checklist

- [ ] Install TanStack Query and DevTools
- [ ] Create and configure QueryClient with retry logic
- [ ] Set up QueryProvider in app root
- [ ] Install Zustand
- [ ] Create UI store for client state
- [ ] Set up persistent auth store
- [ ] Configure DevTools for both libraries
- [ ] Add ESLint plugin for TanStack Query
- [ ] Integrate with Axios interceptors
- [ ] Write tests for critical state logic

---

**Next Steps**: Continue to [Authentication](./09-authentication.md) for security setup.