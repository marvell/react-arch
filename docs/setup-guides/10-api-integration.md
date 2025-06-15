# 10. API Integration

This guide covers setting up a robust API integration layer following our architecture's strict separation of concerns. All server state management **must** use TanStack Query, with Axios handling HTTP communication.

## 1. Configure Axios Instance with Base Settings

### Create the Base Axios Instance

Create `src/shared/api/axios-instance.ts`:

```typescript
import axios, { AxiosInstance, AxiosError, AxiosResponse } from 'axios';

const API_BASE_URL = import.meta.env.VITE_API_BASE_URL || 'https://api.example.com';

export const axiosInstance: AxiosInstance = axios.create({
  baseURL: API_BASE_URL,
  timeout: 30000, // 30 seconds
  headers: {
    'Content-Type': 'application/json',
  },
});
```

### Add Authentication Interceptor

Add request interceptor for JWT tokens:

```typescript
// Request interceptor for authentication
axiosInstance.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('authToken');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => {
    return Promise.reject(error);
  }
);
```

### Add Response Error Interceptor

Handle common HTTP errors centrally:

```typescript
// Response interceptor for error handling
axiosInstance.interceptors.response.use(
  (response: AxiosResponse) => response,
  (error: AxiosError) => {
    if (error.response) {
      switch (error.response.status) {
        case 401:
          // Clear auth token and redirect to login
          localStorage.removeItem('authToken');
          window.location.href = '/login';
          break;
        case 403:
          console.error('Forbidden: Insufficient permissions');
          break;
        case 422:
          // Validation errors - let the component handle these
          break;
        case 500:
          console.error('Server Error: Please try again later');
          break;
        default:
          console.error(`API Error: ${error.response.status}`);
      }
    } else if (error.request) {
      console.error('Network Error: No response received');
    } else {
      console.error('Request Setup Error:', error.message);
    }
    return Promise.reject(error);
  }
);

export default axiosInstance;
```

## 2. Set Up API Layer Structure in shared/api

### Directory Structure

```
src/shared/api/
├── axios-instance.ts          # Base Axios configuration
├── query-client.ts           # TanStack Query client setup
├── types/                    # API response types
│   ├── common.ts            # Common API types (pagination, etc.)
│   └── index.ts             # Type exports
└── index.ts                 # Public API exports
```

### Configure TanStack Query Client

Create `src/shared/api/query-client.ts`:

```typescript
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
      gcTime: 10 * 60 * 1000,   // 10 minutes (formerly cacheTime)
      retry: (failureCount, error: any) => {
        // Don't retry on 4xx errors (client errors)
        if (error?.response?.status >= 400 && error?.response?.status < 500) {
          return false;
        }
        return failureCount < 3;
      },
      retryDelay: (attemptIndex) => Math.min(1000 * 2 ** attemptIndex, 30000),
    },
    mutations: {
      retry: false, // Don't retry mutations by default
    },
  },
});
```

### Common API Types

Create `src/shared/api/types/common.ts`:

```typescript
export interface ApiResponse<T> {
  data: T;
  message?: string;
  success: boolean;
}

export interface PaginatedResponse<T> {
  data: T[];
  pagination: {
    page: number;
    limit: number;
    total: number;
    totalPages: number;
  };
}

export interface ApiError {
  message: string;
  code?: string;
  details?: Record<string, string[]>; // For validation errors
}
```

## 3. Configure Orval for OpenAPI Code Generation

### Install Dependencies

```bash
pnpm add -D orval
```

### Create Orval Configuration

Create `orval.config.ts` in project root:

```typescript
import { defineConfig } from 'orval';

export default defineConfig({
  api: {
    input: {
      target: './openapi.yaml', // or URL to your OpenAPI spec
    },
    output: {
      mode: 'tags-split', // Split by OpenAPI tags
      target: 'src/shared/api/generated',
      client: 'react-query',
      httpClient: 'axios',
      baseUrl: '/api',
      override: {
        mutator: {
          path: 'src/shared/api/axios-instance.ts',
          name: 'axiosInstance',
        },
        query: {
          useQuery: true,
          useSuspenseQuery: true,
          useMutation: true,
          signal: true, // Enable AbortController support
        },
      },
    },
  },
});
```

### Add Generation Scripts

Add to `package.json`:

```json
{
  "scripts": {
    "generate:api": "orval",
    "dev": "pnpm generate:api && vite",
    "build": "pnpm generate:api && vite build"
  }
}
```

### Alternative: Manual Hook Pattern

If not using Orval, create query hooks manually. Example for user entity:

Create `src/entities/user/api/user-queries.ts`:

```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { queryOptions } from '@tanstack/react-query';
import axiosInstance from '@/shared/api/axios-instance';
import type { User } from '../model/types';

// Query options for type safety and reuse
export const userQueries = {
  all: () => ['users'] as const,
  lists: () => [...userQueries.all(), 'list'] as const,
  list: (filters: UserFilters) => [...userQueries.lists(), filters] as const,
  details: () => [...userQueries.all(), 'detail'] as const,
  detail: (id: number) => [...userQueries.details(), id] as const,
};

// Fetch functions
const fetchUser = async (id: number): Promise<User> => {
  const { data } = await axiosInstance.get<ApiResponse<User>>(`/users/${id}`);
  return data.data;
};

const fetchUsers = async (filters: UserFilters): Promise<User[]> => {
  const { data } = await axiosInstance.get<ApiResponse<User[]>>('/users', {
    params: filters,
  });
  return data.data;
};

// Query hooks using queryOptions for type safety
export const useUser = (id: number) => {
  return useQuery({
    ...queryOptions({
      queryKey: userQueries.detail(id),
      queryFn: () => fetchUser(id),
      enabled: !!id,
    }),
  });
};

export const useUsers = (filters: UserFilters = {}) => {
  return useQuery({
    ...queryOptions({
      queryKey: userQueries.list(filters),
      queryFn: () => fetchUsers(filters),
    }),
  });
};

// Mutation hooks
export const useCreateUser = () => {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: async (userData: CreateUserRequest): Promise<User> => {
      const { data } = await axiosInstance.post<ApiResponse<User>>('/users', userData);
      return data.data;
    },
    onSuccess: () => {
      // Invalidate and refetch user queries
      queryClient.invalidateQueries({ queryKey: userQueries.all() });
    },
  });
};
```

## 4. Create API Error Handling and Retry Logic

### Error Boundary Integration

Create `src/shared/ui/query-error-boundary.tsx`:

```typescript
import { QueryErrorResetBoundary } from '@tanstack/react-query';
import { ErrorBoundary } from 'react-error-boundary';
import { Button } from './button';

function ErrorFallback({ error, resetErrorBoundary }: any) {
  return (
    <div className="flex flex-col items-center justify-center min-h-[200px] p-6">
      <h2 className="text-lg font-semibold text-destructive mb-2">
        Something went wrong
      </h2>
      <p className="text-sm text-muted-foreground mb-4">
        {error.message || 'An unexpected error occurred'}
      </p>
      <Button onClick={resetErrorBoundary} variant="outline">
        Try again
      </Button>
    </div>
  );
}

export function QueryErrorBoundary({ children }: { children: React.ReactNode }) {
  return (
    <QueryErrorResetBoundary>
      {({ reset }) => (
        <ErrorBoundary FallbackComponent={ErrorFallback} onReset={reset}>
          {children}
        </ErrorBoundary>
      )}
    </QueryErrorResetBoundary>
  );
}
```

### Form Integration with Server Errors

Create `src/shared/lib/api-form-helpers.ts`:

```typescript
import { AxiosError } from 'axios';
import { UseFormSetError } from 'react-hook-form';
import type { ApiError } from '@/shared/api/types/common';

export const handleApiFormErrors = <T extends Record<string, any>>(
  error: AxiosError<ApiError>,
  setError: UseFormSetError<T>
) => {
  if (error.response?.status === 422 && error.response.data.details) {
    // Handle validation errors
    Object.entries(error.response.data.details).forEach(([field, messages]) => {
      setError(field as keyof T, {
        type: 'server',
        message: messages[0],
      });
    });
  } else {
    // Handle general server errors
    setError('root.server', {
      type: 'server',
      message: error.response?.data.message || 'An error occurred',
    });
  }
};
```

### Usage in Components

Example form component with API integration:

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';
import { useCreateUser } from '@/entities/user/api/user-queries';
import { handleApiFormErrors } from '@/shared/lib/api-form-helpers';

const createUserSchema = z.object({
  name: z.string().min(2, 'Name must be at least 2 characters'),
  email: z.string().email('Invalid email address'),
});

type CreateUserForm = z.infer<typeof createUserSchema>;

export function CreateUserForm() {
  const form = useForm<CreateUserForm>({
    resolver: zodResolver(createUserSchema),
  });

  const createUserMutation = useCreateUser();

  const onSubmit = async (data: CreateUserForm) => {
    try {
      await createUserMutation.mutateAsync(data);
      form.reset();
      // Handle success (toast, redirect, etc.)
    } catch (error) {
      handleApiFormErrors(error as AxiosError, form.setError);
    }
  };

  return (
    <form onSubmit={form.handleSubmit(onSubmit)}>
      {/* Form fields */}
      {form.formState.errors.root?.server && (
        <p className="text-sm text-destructive">
          {form.formState.errors.root.server.message}
        </p>
      )}
      <Button 
        type="submit" 
        disabled={createUserMutation.isPending}
      >
        {createUserMutation.isPending ? 'Creating...' : 'Create User'}
      </Button>
    </form>
  );
}
```

## 5. App Integration

### Add Providers to App

Update `src/app/providers/index.tsx`:

```typescript
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ReactQueryDevtools } from '@tanstack/react-query-devtools';
import { queryClient } from '@/shared/api/query-client';
import { QueryErrorBoundary } from '@/shared/ui/query-error-boundary';

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      <QueryErrorBoundary>
        {children}
      </QueryErrorBoundary>
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  );
}
```

## Key Principles

1. **Single Source of Truth**: TanStack Query is the **only** way to manage server state
2. **Type Safety**: Use TypeScript throughout the API layer with proper error types
3. **Error Boundaries**: Wrap components in QueryErrorBoundary for graceful error handling
4. **Consistent Structure**: Place API logic in `shared/api/` or entity-specific `model/` directories
5. **Authentication**: JWT tokens in localStorage with automatic header injection
6. **Validation**: Server validation errors should be handled in forms using React Hook Form

---

**Next Steps**: Continue to [Testing Setup](./11-testing-setup.md) for quality assurance.