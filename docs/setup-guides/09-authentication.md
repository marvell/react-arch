# 9. Authentication Infrastructure

This guide implements a complete OAuth 2.0 authentication system with JWT token management, following our architecture's strict separation of concerns.

## Overview

Our authentication flow follows these principles:
- **OAuth 2.0** for secure third-party authentication (Google, GitHub, etc.)
- **JWT tokens** stored in `localStorage` for persistence
- **Zustand store** for auth state management with automatic hydration
- **Axios interceptors** for automatic token attachment and refresh handling
- **TanStack Query** for auth-related API calls

## 1. Authentication Store Setup

Create the main authentication store using Zustand with persistence:

```typescript
// src/shared/lib/auth/auth-store.ts
import { create } from 'zustand'
import { persist, createJSONStorage } from 'zustand/middleware'

export interface User {
  id: string
  email: string
  name: string
  avatar?: string
}

interface AuthState {
  // State
  token: string | null
  user: User | null
  isAuthenticated: boolean
  isLoading: boolean
  
  // Actions
  login: (token: string, user: User) => void
  logout: () => void
  setLoading: (loading: boolean) => void
  updateUser: (user: Partial<User>) => void
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set, get) => ({
      // Initial state
      token: null,
      user: null,
      isAuthenticated: false,
      isLoading: false,

      // Actions
      login: (token: string, user: User) => {
        set({
          token,
          user,
          isAuthenticated: true,
          isLoading: false,
        })
      },

      logout: () => {
        set({
          token: null,
          user: null,
          isAuthenticated: false,
          isLoading: false,
        })
        // Clear persisted state
        useAuthStore.persist.clearStorage()
      },

      setLoading: (loading: boolean) => {
        set({ isLoading: loading })
      },

      updateUser: (userData: Partial<User>) => {
        const currentUser = get().user
        if (currentUser) {
          set({ user: { ...currentUser, ...userData } })
        }
      },
    }),
    {
      name: 'auth-store', // localStorage key
      storage: createJSONStorage(() => localStorage),
      // Only persist essential data
      partialize: (state) => ({
        token: state.token,
        user: state.user,
        isAuthenticated: state.isAuthenticated,
      }),
    }
  )
)

// Hydration hook for SSR compatibility
export const useAuthHydration = () => {
  const [hydrated, setHydrated] = useState(false)

  useEffect(() => {
    const unsubHydrate = useAuthStore.persist.onHydrate(() => setHydrated(false))
    const unsubFinishHydration = useAuthStore.persist.onFinishHydration(() => setHydrated(true))
    
    setHydrated(useAuthStore.persist.hasHydrated())

    return () => {
      unsubHydrate()
      unsubFinishHydration()
    }
  }, [])

  return hydrated
}
```

## 2. Axios Configuration with Authentication

Configure Axios with interceptors for automatic token management:

```typescript
// src/shared/api/axios-instance.ts
import axios, { AxiosRequestConfig, AxiosError } from 'axios'
import { useAuthStore } from '../lib/auth/auth-store'

// Create axios instance
export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || 'http://localhost:3000/api',
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
})

// Request interceptor - attach token
apiClient.interceptors.request.use(
  (config: AxiosRequestConfig) => {
    const { token } = useAuthStore.getState()
    
    if (token && config.headers) {
      config.headers.Authorization = `Bearer ${token}`
    }
    
    return config
  },
  (error) => Promise.reject(error)
)

// Response interceptor - handle 401 and token refresh
apiClient.interceptors.response.use(
  (response) => response,
  async (error: AxiosError) => {
    const originalRequest = error.config as AxiosRequestConfig & { _retry?: boolean }
    
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true
      
      // Clear auth state on 401
      const { logout } = useAuthStore.getState()
      logout()
      
      // Optional: Redirect to login
      if (typeof window !== 'undefined') {
        window.location.href = '/login'
      }
    }
    
    return Promise.reject(error)
  }
)
```

## 3. OAuth 2.0 Implementation

### Auth API Functions

```typescript
// src/shared/api/auth.ts
import { apiClient } from './axios-instance'
import { User } from '../lib/auth/auth-store'

export interface AuthResponse {
  token: string
  user: User
  expiresIn: number
}

export interface OAuthCallbackPayload {
  code: string
  state?: string
  provider: 'google' | 'github' | 'discord'
}

export const authApi = {
  // Exchange OAuth code for JWT
  exchangeOAuthCode: async (payload: OAuthCallbackPayload): Promise<AuthResponse> => {
    const { data } = await apiClient.post<AuthResponse>('/auth/oauth/callback', payload)
    return data
  },

  // Get current user profile
  getCurrentUser: async (): Promise<User> => {
    const { data } = await apiClient.get<User>('/auth/me')
    return data
  },

  // Refresh token (if implementing refresh token flow)
  refreshToken: async (refreshToken: string): Promise<AuthResponse> => {
    const { data } = await apiClient.post<AuthResponse>('/auth/refresh', {
      refreshToken,
    })
    return data
  },

  // Logout
  logout: async (): Promise<void> => {
    await apiClient.post('/auth/logout')
  },
}
```

### TanStack Query Hooks

```typescript
// src/shared/lib/auth/auth-queries.ts
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import { authApi, type OAuthCallbackPayload, type AuthResponse } from '../../api/auth'
import { useAuthStore } from './auth-store'

export const useOAuthCallback = () => {
  const { login, setLoading } = useAuthStore()
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: (payload: OAuthCallbackPayload) => authApi.exchangeOAuthCode(payload),
    onMutate: () => setLoading(true),
    onSuccess: (data: AuthResponse) => {
      login(data.token, data.user)
      // Invalidate and refetch any user-related queries
      queryClient.invalidateQueries({ queryKey: ['user'] })
    },
    onError: (error) => {
      console.error('OAuth callback failed:', error)
      setLoading(false)
    },
  })
}

export const useCurrentUser = () => {
  const { isAuthenticated, token } = useAuthStore()
  
  return useQuery({
    queryKey: ['user', 'current'],
    queryFn: authApi.getCurrentUser,
    enabled: isAuthenticated && !!token,
    staleTime: 5 * 60 * 1000, // 5 minutes
    retry: false, // Don't retry on 401
  })
}

export const useLogout = () => {
  const { logout } = useAuthStore()
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: authApi.logout,
    onSuccess: () => {
      logout()
      queryClient.clear() // Clear all cached data
    },
    onError: () => {
      // Even if API call fails, clear local state
      logout()
      queryClient.clear()
    },
  })
}
```

## 4. OAuth Provider URLs

Create helper functions for OAuth redirects:

```typescript
// src/shared/lib/auth/oauth-providers.ts
export interface OAuthProvider {
  name: string
  url: string
  icon?: string
}

const generateState = () => Math.random().toString(36).substring(2, 15)

export const getOAuthUrl = (provider: 'google' | 'github' | 'discord'): string => {
  const baseUrl = import.meta.env.VITE_APP_URL || 'http://localhost:5173'
  const redirectUri = `${baseUrl}/auth/callback`
  const state = generateState()
  
  // Store state in sessionStorage for validation
  sessionStorage.setItem('oauth_state', state)
  
  const params = new URLSearchParams({
    redirect_uri: redirectUri,
    state,
    scope: provider === 'google' 
      ? 'openid email profile' 
      : provider === 'github'
      ? 'user:email'
      : 'identify email',
  })

  const urls = {
    google: `https://accounts.google.com/o/oauth2/v2/auth?${params}&client_id=${import.meta.env.VITE_GOOGLE_CLIENT_ID}&response_type=code`,
    github: `https://github.com/login/oauth/authorize?${params}&client_id=${import.meta.env.VITE_GITHUB_CLIENT_ID}`,
    discord: `https://discord.com/api/oauth2/authorize?${params}&client_id=${import.meta.env.VITE_DISCORD_CLIENT_ID}&response_type=code`,
  }

  return urls[provider]
}

export const oauthProviders: OAuthProvider[] = [
  { name: 'Google', url: getOAuthUrl('google') },
  { name: 'GitHub', url: getOAuthUrl('github') },
  { name: 'Discord', url: getOAuthUrl('discord') },
]
```

## 5. Authentication Components

### Protected Route Component

```typescript
// src/shared/ui/protected-route.tsx
import { ReactNode } from 'react'
import { Navigate } from '@tanstack/react-router'
import { useAuthStore, useAuthHydration } from '../lib/auth/auth-store'

interface ProtectedRouteProps {
  children: ReactNode
  fallback?: ReactNode
  redirectTo?: string
}

export const ProtectedRoute = ({ 
  children, 
  fallback = <div>Loading...</div>,
  redirectTo = '/login' 
}: ProtectedRouteProps) => {
  const hydrated = useAuthHydration()
  const { isAuthenticated, isLoading } = useAuthStore()

  // Wait for hydration to complete
  if (!hydrated || isLoading) {
    return <>{fallback}</>
  }

  if (!isAuthenticated) {
    return <Navigate to={redirectTo} />
  }

  return <>{children}</>
}
```

### OAuth Callback Handler

```typescript
// src/pages/auth/callback.tsx
import { useEffect } from 'react'
import { useNavigate, useSearch } from '@tanstack/react-router'
import { useOAuthCallback } from '../../shared/lib/auth/auth-queries'

export const AuthCallbackPage = () => {
  const navigate = useNavigate()
  const search = useSearch({ from: '/auth/callback' }) as { code?: string; state?: string; provider?: string }
  const oauthCallback = useOAuthCallback()

  useEffect(() => {
    const { code, state, provider } = search
    
    if (!code || !provider) {
      navigate({ to: '/login', search: { error: 'invalid_callback' } })
      return
    }

    // Validate state parameter
    const storedState = sessionStorage.getItem('oauth_state')
    if (state !== storedState) {
      navigate({ to: '/login', search: { error: 'invalid_state' } })
      return
    }

    // Clean up stored state
    sessionStorage.removeItem('oauth_state')

    // Exchange code for token
    oauthCallback.mutate(
      { code, state, provider: provider as any },
      {
        onSuccess: () => {
          navigate({ to: '/dashboard' })
        },
        onError: () => {
          navigate({ to: '/login', search: { error: 'auth_failed' } })
        },
      }
    )
  }, [search, navigate, oauthCallback])

  return (
    <div className="flex items-center justify-center min-h-screen">
      <div className="text-center">
        <h2 className="text-xl font-semibold mb-4">Completing authentication...</h2>
        <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-primary mx-auto"></div>
      </div>
    </div>
  )
}
```

## 6. Environment Variables

Add these environment variables to your `.env` file:

```bash
# API Configuration
VITE_API_BASE_URL=http://localhost:3000/api
VITE_APP_URL=http://localhost:5173

# OAuth Providers
VITE_GOOGLE_CLIENT_ID=your-google-client-id
VITE_GITHUB_CLIENT_ID=your-github-client-id
VITE_DISCORD_CLIENT_ID=your-discord-client-id
```

## 7. Security Considerations

### Token Storage Security
- **localStorage vs sessionStorage**: While we use `localStorage` for persistence, consider `sessionStorage` for enhanced security in sensitive applications
- **XSS Protection**: Ensure your app is protected against XSS attacks since tokens in localStorage are vulnerable
- **HTTPS Only**: Always use HTTPS in production to prevent token interception

### Custom Secure Storage (Optional)
For enhanced security, implement encrypted token storage:

```typescript
// src/shared/lib/auth/secure-storage.ts
import { StateStorage } from 'zustand/middleware'

const encrypt = (text: string, key: string): string => {
  // Implement your encryption logic
  return btoa(text) // Simple base64 for example
}

const decrypt = (encryptedText: string, key: string): string => {
  // Implement your decryption logic
  return atob(encryptedText) // Simple base64 for example
}

export const createSecureStorage = (encryptionKey: string): StateStorage => ({
  getItem: (name: string): string | null => {
    const item = localStorage.getItem(name)
    if (!item) return null
    try {
      return decrypt(item, encryptionKey)
    } catch {
      return null
    }
  },
  setItem: (name: string, value: string): void => {
    localStorage.setItem(name, encrypt(value, encryptionKey))
  },
  removeItem: (name: string): void => {
    localStorage.removeItem(name)
  },
})
```

### CSRF Protection
Implement state parameter validation in OAuth flows to prevent CSRF attacks (already included in the examples above).

## 8. Testing

### Auth Store Tests
```typescript
// src/shared/lib/auth/__tests__/auth-store.test.ts
import { act, renderHook } from '@testing-library/react'
import { useAuthStore } from '../auth-store'

describe('useAuthStore', () => {
  beforeEach(() => {
    useAuthStore.getState().logout()
  })

  it('should login user correctly', () => {
    const { result } = renderHook(() => useAuthStore())
    const mockUser = { id: '1', email: 'test@example.com', name: 'Test User' }
    const mockToken = 'jwt-token'

    act(() => {
      result.current.login(mockToken, mockUser)
    })

    expect(result.current.isAuthenticated).toBe(true)
    expect(result.current.user).toEqual(mockUser)
    expect(result.current.token).toBe(mockToken)
  })

  it('should logout user correctly', () => {
    const { result } = renderHook(() => useAuthStore())

    act(() => {
      result.current.login('token', { id: '1', email: 'test@example.com', name: 'Test' })
    })

    act(() => {
      result.current.logout()
    })

    expect(result.current.isAuthenticated).toBe(false)
    expect(result.current.user).toBeNull()
    expect(result.current.token).toBeNull()
  })
})
```

---

**Next Steps**: Continue to [API Integration](./10-api-integration.md) for backend communication.