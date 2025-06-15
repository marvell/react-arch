# 13. First Feature Implementation

This guide walks you through implementing your first complete feature using our architectural conventions. We'll build a **Task Management** feature with full CRUD operations, authentication, and testing.

## Prerequisites

- Completed all previous setup guides (1-12)
- Development environment with TypeScript, Vite, and all dependencies installed
- Basic understanding of React, TypeScript, and our tech stack

## Overview: What We're Building

Our first feature will demonstrate:
- **Entity Layer**: Task entity with TypeScript types and API integration
- **Feature Layer**: CRUD operations for task management
- **Page Layer**: Complete task management page with routing
- **Authentication Flow**: OAuth 2.0 with JWT token management
- **Testing**: Unit and integration tests

## Step 1: Create the Task Entity

Following **Feature-Sliced Design**, we start with the `entities` layer.

### 1.1. Define Task Entity Structure

Create the entity directory structure:

```bash
mkdir -p src/entities/task/{model,api,ui}
```

**`src/entities/task/model/task.ts`**
```typescript
export interface Task {
  id: string;
  title: string;
  description: string;
  status: 'pending' | 'in_progress' | 'completed';
  priority: 'low' | 'medium' | 'high';
  createdAt: string;
  updatedAt: string;
  userId: string;
}

export interface CreateTaskInput {
  title: string;
  description: string;
  priority: Task['priority'];
}

export interface UpdateTaskInput extends Partial<CreateTaskInput> {
  status?: Task['status'];
}
```

### 1.2. Set up Task API Layer

**`src/entities/task/api/task.ts`**
```typescript
import { apiClient } from 'shared/api';
import type { Task, CreateTaskInput, UpdateTaskInput } from '../model/task';

export const taskApi = {
  // Fetch all tasks for current user
  getTasks: async (): Promise<Task[]> => {
    const response = await apiClient.get('/tasks');
    return response.data;
  },

  // Fetch single task by ID
  getTaskById: async (id: string): Promise<Task> => {
    const response = await apiClient.get(`/tasks/${id}`);
    return response.data;
  },

  // Create new task
  createTask: async (input: CreateTaskInput): Promise<Task> => {
    const response = await apiClient.post('/tasks', input);
    return response.data;
  },

  // Update existing task
  updateTask: async (id: string, input: UpdateTaskInput): Promise<Task> => {
    const response = await apiClient.patch(`/tasks/${id}`, input);
    return response.data;
  },

  // Delete task
  deleteTask: async (id: string): Promise<void> => {
    await apiClient.delete(`/tasks/${id}`);
  },
};
```

### 1.3. Create Task Query Hooks (TanStack Query v5)

**`src/entities/task/api/queries.ts`**
```typescript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';
import { taskApi } from './task';
import type { CreateTaskInput, UpdateTaskInput } from '../model/task';

// Query keys for consistent caching
export const taskKeys = {
  all: ['tasks'] as const,
  lists: () => [...taskKeys.all, 'list'] as const,
  list: (filters: string) => [...taskKeys.lists(), { filters }] as const,
  details: () => [...taskKeys.all, 'detail'] as const,
  detail: (id: string) => [...taskKeys.details(), id] as const,
};

// Fetch all tasks
export const useTasksQuery = () => {
  return useQuery({
    queryKey: taskKeys.lists(),
    queryFn: taskApi.getTasks,
    staleTime: 5 * 60 * 1000, // 5 minutes
  });
};

// Fetch single task
export const useTaskQuery = (id: string) => {
  return useQuery({
    queryKey: taskKeys.detail(id),
    queryFn: () => taskApi.getTaskById(id),
    enabled: !!id,
  });
};

// Create task mutation
export const useCreateTaskMutation = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: taskApi.createTask,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: taskKeys.lists() });
    },
  });
};

// Update task mutation
export const useUpdateTaskMutation = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: ({ id, input }: { id: string; input: UpdateTaskInput }) =>
      taskApi.updateTask(id, input),
    onSuccess: (data) => {
      // Update both list and detail caches
      queryClient.invalidateQueries({ queryKey: taskKeys.lists() });
      queryClient.setQueryData(taskKeys.detail(data.id), data);
    },
  });
};

// Delete task mutation
export const useDeleteTaskMutation = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: taskApi.deleteTask,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: taskKeys.lists() });
    },
  });
};
```

### 1.4. Create Task UI Components

**`src/entities/task/ui/TaskCard.tsx`**
```typescript
import React from 'react';
import { Card, CardContent, CardHeader, CardTitle } from 'shared/ui/card';
import { Badge } from 'shared/ui/badge';
import { cn } from 'shared/lib/utils';
import type { Task } from '../model/task';

interface TaskCardProps {
  task: Task;
  className?: string;
}

export const TaskCard: React.FC<TaskCardProps> = ({ task, className }) => {
  const statusColors = {
    pending: 'bg-yellow-100 text-yellow-800',
    in_progress: 'bg-blue-100 text-blue-800',
    completed: 'bg-green-100 text-green-800',
  };

  const priorityColors = {
    low: 'bg-gray-100 text-gray-800',
    medium: 'bg-orange-100 text-orange-800',
    high: 'bg-red-100 text-red-800',
  };

  return (
    <Card className={cn('cursor-pointer hover:shadow-md transition-shadow', className)}>
      <CardHeader className="pb-2">
        <div className="flex items-center justify-between">
          <CardTitle className="text-lg">{task.title}</CardTitle>
          <div className="flex gap-2">
            <Badge className={statusColors[task.status]}>
              {task.status.replace('_', ' ')}
            </Badge>
            <Badge className={priorityColors[task.priority]}>
              {task.priority}
            </Badge>
          </div>
        </div>
      </CardHeader>
      <CardContent>
        <p className="text-sm text-muted-foreground">{task.description}</p>
        <p className="text-xs text-muted-foreground mt-2">
          Created: {new Date(task.createdAt).toLocaleDateString()}
        </p>
      </CardContent>
    </Card>
  );
};
```

### 1.5. Task Entity Public API

**`src/entities/task/index.ts`**
```typescript
export type { Task, CreateTaskInput, UpdateTaskInput } from './model/task';
export { TaskCard } from './ui/TaskCard';
export {
  useTasksQuery,
  useTaskQuery,
  useCreateTaskMutation,
  useUpdateTaskMutation,
  useDeleteTaskMutation,
  taskKeys,
} from './api/queries';
```

## Step 2: Create Task Management Features

### 2.1. Create Task Feature

**`src/features/create-task/ui/CreateTaskForm.tsx`**
```typescript
import React from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import * as z from 'zod';
import { Button } from 'shared/ui/button';
import { Input } from 'shared/ui/input';
import { Textarea } from 'shared/ui/textarea';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from 'shared/ui/select';
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from 'shared/ui/form';
import { useCreateTaskMutation } from 'entities/task';
import type { CreateTaskInput } from 'entities/task';

const createTaskSchema = z.object({
  title: z.string().min(1, 'Title is required').max(100, 'Title too long'),
  description: z.string().min(1, 'Description is required').max(500, 'Description too long'),
  priority: z.enum(['low', 'medium', 'high']),
});

interface CreateTaskFormProps {
  onSuccess?: () => void;
}

export const CreateTaskForm: React.FC<CreateTaskFormProps> = ({ onSuccess }) => {
  const createTaskMutation = useCreateTaskMutation();

  const form = useForm<CreateTaskInput>({
    resolver: zodResolver(createTaskSchema),
    defaultValues: {
      title: '',
      description: '',
      priority: 'medium',
    },
  });

  const onSubmit = async (data: CreateTaskInput) => {
    try {
      await createTaskMutation.mutateAsync(data);
      form.reset();
      onSuccess?.();
    } catch (error) {
      // Error handling is managed by TanStack Query
      console.error('Failed to create task:', error);
    }
  };

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        <FormField
          control={form.control}
          name="title"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Title</FormLabel>
              <FormControl>
                <Input placeholder="Enter task title" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        
        <FormField
          control={form.control}
          name="description"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Description</FormLabel>
              <FormControl>
                <Textarea placeholder="Enter task description" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        
        <FormField
          control={form.control}
          name="priority"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Priority</FormLabel>
              <Select onValueChange={field.onChange} defaultValue={field.value}>
                <FormControl>
                  <SelectTrigger>
                    <SelectValue placeholder="Select priority" />
                  </SelectTrigger>
                </FormControl>
                <SelectContent>
                  <SelectItem value="low">Low</SelectItem>
                  <SelectItem value="medium">Medium</SelectItem>
                  <SelectItem value="high">High</SelectItem>
                </SelectContent>
              </Select>
              <FormMessage />
            </FormItem>
          )}
        />
        
        <Button 
          type="submit" 
          disabled={createTaskMutation.isPending}
          className="w-full"
        >
          {createTaskMutation.isPending ? 'Creating...' : 'Create Task'}
        </Button>
        
        {createTaskMutation.isError && (
          <p className="text-sm text-destructive">
            Failed to create task. Please try again.
          </p>
        )}
      </form>
    </Form>
  );
};
```

**`src/features/create-task/index.ts`**
```typescript
export { CreateTaskForm } from './ui/CreateTaskForm';
```

### 2.2. Task Actions Feature

**`src/features/task-actions/ui/TaskActions.tsx`**
```typescript
import React from 'react';
import { Button } from 'shared/ui/button';
import { DropdownMenu, DropdownMenuContent, DropdownMenuItem, DropdownMenuTrigger } from 'shared/ui/dropdown-menu';
import { MoreHorizontal, Edit, Trash2, CheckCircle } from 'lucide-react';
import { useUpdateTaskMutation, useDeleteTaskMutation } from 'entities/task';
import type { Task } from 'entities/task';

interface TaskActionsProps {
  task: Task;
  onEdit?: (task: Task) => void;
}

export const TaskActions: React.FC<TaskActionsProps> = ({ task, onEdit }) => {
  const updateTaskMutation = useUpdateTaskMutation();
  const deleteTaskMutation = useDeleteTaskMutation();

  const handleStatusToggle = async () => {
    const newStatus = task.status === 'completed' ? 'pending' : 'completed';
    await updateTaskMutation.mutateAsync({
      id: task.id,
      input: { status: newStatus },
    });
  };

  const handleDelete = async () => {
    if (window.confirm('Are you sure you want to delete this task?')) {
      await deleteTaskMutation.mutateAsync(task.id);
    }
  };

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="ghost" className="h-8 w-8 p-0">
          <MoreHorizontal className="h-4 w-4" />
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end">
        <DropdownMenuItem onClick={handleStatusToggle}>
          <CheckCircle className="mr-2 h-4 w-4" />
          {task.status === 'completed' ? 'Mark Pending' : 'Mark Complete'}
        </DropdownMenuItem>
        <DropdownMenuItem onClick={() => onEdit?.(task)}>
          <Edit className="mr-2 h-4 w-4" />
          Edit
        </DropdownMenuItem>
        <DropdownMenuItem onClick={handleDelete} className="text-destructive">
          <Trash2 className="mr-2 h-4 w-4" />
          Delete
        </DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  );
};
```

**`src/features/task-actions/index.ts`**
```typescript
export { TaskActions } from './ui/TaskActions';
```

## Step 3: Authentication Setup

### 3.1. Auth Store (Zustand)

**`src/shared/stores/auth.ts`**
```typescript
import { create } from 'zustand';
import { persist } from 'zustand/middleware';

interface User {
  id: string;
  email: string;
  name: string;
  avatar?: string;
}

interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
}

interface AuthActions {
  setAuth: (user: User, token: string) => void;
  clearAuth: () => void;
  updateUser: (user: Partial<User>) => void;
}

type AuthStore = AuthState & AuthActions;

export const useAuthStore = create<AuthStore>()(
  persist(
    (set, get) => ({
      user: null,
      token: null,
      isAuthenticated: false,
      
      setAuth: (user, token) => {
        localStorage.setItem('auth_token', token);
        set({ user, token, isAuthenticated: true });
      },
      
      clearAuth: () => {
        localStorage.removeItem('auth_token');
        set({ user: null, token: null, isAuthenticated: false });
      },
      
      updateUser: (userData) => {
        const currentUser = get().user;
        if (currentUser) {
          set({ user: { ...currentUser, ...userData } });
        }
      },
    }),
    {
      name: 'auth-storage',
      partialize: (state) => ({ 
        user: state.user, 
        token: state.token, 
        isAuthenticated: state.isAuthenticated 
      }),
    }
  )
);
```

### 3.2. API Client with Auth Interceptor

**`src/shared/api/client.ts`**
```typescript
import axios from 'axios';
import { useAuthStore } from 'shared/stores/auth';

export const apiClient = axios.create({
  baseURL: process.env.VITE_API_URL || 'http://localhost:3001/api',
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json',
  },
});

// Request interceptor to add auth token
apiClient.interceptors.request.use(
  (config) => {
    const token = localStorage.getItem('auth_token');
    if (token) {
      config.headers.Authorization = `Bearer ${token}`;
    }
    return config;
  },
  (error) => Promise.reject(error)
);

// Response interceptor for auth errors
apiClient.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      // Clear auth state on 401
      useAuthStore.getState().clearAuth();
      // Redirect to login or refresh token logic here
    }
    return Promise.reject(error);
  }
);
```

### 3.3. OAuth Authentication Feature

**`src/features/auth/ui/LoginButton.tsx`**
```typescript
import React from 'react';
import { Button } from 'shared/ui/button';

export const LoginButton: React.FC = () => {
  const handleLogin = () => {
    // Redirect to OAuth provider (Google, GitHub, etc.)
    const authUrl = `${process.env.VITE_API_URL}/auth/google`;
    window.location.href = authUrl;
  };

  return (
    <Button onClick={handleLogin} size="lg">
      Sign in with Google
    </Button>
  );
};
```

**`src/features/auth/ui/AuthCallback.tsx`**
```typescript
import React, { useEffect } from 'react';
import { useNavigate, useSearch } from '@tanstack/react-router';
import { useAuthStore } from 'shared/stores/auth';
import { apiClient } from 'shared/api';

export const AuthCallback: React.FC = () => {
  const navigate = useNavigate();
  const search = useSearch({ from: '/auth/callback' });
  const setAuth = useAuthStore((state) => state.setAuth);

  useEffect(() => {
    const handleCallback = async () => {
      try {
        const { code } = search as { code?: string };
        if (!code) {
          throw new Error('No authorization code received');
        }

        const response = await apiClient.post('/auth/callback', { code });
        const { user, token } = response.data;

        setAuth(user, token);
        navigate({ to: '/tasks' });
      } catch (error) {
        console.error('Auth callback error:', error);
        navigate({ to: '/login' });
      }
    };

    handleCallback();
  }, [search, setAuth, navigate]);

  return (
    <div className="flex items-center justify-center min-h-screen">
      <div className="text-center">
        <p>Completing authentication...</p>
      </div>
    </div>
  );
};
```

## Step 4: Create Pages with TanStack Router

### 4.1. Tasks Page

**`src/pages/tasks/ui/TasksPage.tsx`**
```typescript
import React, { useState } from 'react';
import { Button } from 'shared/ui/button';
import { Dialog, DialogContent, DialogHeader, DialogTitle, DialogTrigger } from 'shared/ui/dialog';
import { Plus } from 'lucide-react';
import { TaskCard, useTasksQuery } from 'entities/task';
import { CreateTaskForm } from 'features/create-task';
import { TaskActions } from 'features/task-actions';

export const TasksPage: React.FC = () => {
  const [createDialogOpen, setCreateDialogOpen] = useState(false);
  const { data: tasks, isPending, error } = useTasksQuery();

  if (isPending) {
    return (
      <div className="container mx-auto py-8">
        <div className="text-center">Loading tasks...</div>
      </div>
    );
  }

  if (error) {
    return (
      <div className="container mx-auto py-8">
        <div className="text-center text-destructive">
          Error loading tasks: {error.message}
        </div>
      </div>
    );
  }

  return (
    <div className="container mx-auto py-8">
      <div className="flex items-center justify-between mb-8">
        <h1 className="text-3xl font-bold">My Tasks</h1>
        
        <Dialog open={createDialogOpen} onOpenChange={setCreateDialogOpen}>
          <DialogTrigger asChild>
            <Button>
              <Plus className="mr-2 h-4 w-4" />
              Add Task
            </Button>
          </DialogTrigger>
          <DialogContent>
            <DialogHeader>
              <DialogTitle>Create New Task</DialogTitle>
            </DialogHeader>
            <CreateTaskForm onSuccess={() => setCreateDialogOpen(false)} />
          </DialogContent>
        </Dialog>
      </div>

      <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
        {tasks?.map((task) => (
          <div key={task.id} className="relative">
            <TaskCard task={task} />
            <div className="absolute top-2 right-2">
              <TaskActions task={task} />
            </div>
          </div>
        ))}
      </div>

      {tasks?.length === 0 && (
        <div className="text-center py-12">
          <p className="text-muted-foreground">No tasks yet. Create your first task!</p>
        </div>
      )}
    </div>
  );
};
```

### 4.2. Route Configuration

**`src/app/router.tsx`**
```typescript
import React from 'react';
import { createRouter, createRoute, createRootRoute } from '@tanstack/react-router';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { Outlet } from '@tanstack/react-router';
import { TasksPage } from 'pages/tasks';
import { LoginButton } from 'features/auth';
import { AuthCallback } from 'features/auth';
import { useAuthStore } from 'shared/stores/auth';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000,
      retry: 1,
    },
  },
});

// Root route with layout
const rootRoute = createRootRoute({
  component: () => (
    <QueryClientProvider client={queryClient}>
      <div className="min-h-screen bg-background">
        <Outlet />
      </div>
    </QueryClientProvider>
  ),
});

// Index route - redirect to tasks or show login
const indexRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/',
  component: () => {
    const isAuthenticated = useAuthStore((state) => state.isAuthenticated);
    
    if (isAuthenticated) {
      return <TasksPage />;
    }
    
    return (
      <div className="flex items-center justify-center min-h-screen">
        <div className="text-center">
          <h1 className="text-4xl font-bold mb-8">Task Manager</h1>
          <LoginButton />
        </div>
      </div>
    );
  },
});

// Tasks route
const tasksRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/tasks',
  component: TasksPage,
  beforeLoad: () => {
    const isAuthenticated = useAuthStore.getState().isAuthenticated;
    if (!isAuthenticated) {
      throw new Error('Authentication required');
    }
  },
});

// Auth callback route
const authCallbackRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/auth/callback',
  component: AuthCallback,
});

const routeTree = rootRoute.addChildren([indexRoute, tasksRoute, authCallbackRoute]);

export const router = createRouter({ routeTree });
```

## Step 5: Testing

### 5.1. Task Entity Tests

**`src/entities/task/api/__tests__/queries.test.ts`**
```typescript
import { renderHook, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { useTasksQuery, useCreateTaskMutation } from '../queries';
import { taskApi } from '../task';

// Mock the API
jest.mock('../task');
const mockedTaskApi = taskApi as jest.Mocked<typeof taskApi>;

const createWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false },
    },
  });
  
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
};

describe('Task Queries', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  describe('useTasksQuery', () => {
    it('should fetch tasks successfully', async () => {
      const mockTasks = [
        { id: '1', title: 'Test Task', description: 'Test', status: 'pending', priority: 'medium', createdAt: '2024-01-01', updatedAt: '2024-01-01', userId: 'user1' },
      ];
      
      mockedTaskApi.getTasks.mockResolvedValue(mockTasks);

      const { result } = renderHook(() => useTasksQuery(), {
        wrapper: createWrapper(),
      });

      await waitFor(() => {
        expect(result.current.data).toEqual(mockTasks);
      });
      
      expect(mockedTaskApi.getTasks).toHaveBeenCalledTimes(1);
    });
    
    it('should handle fetch error', async () => {
      mockedTaskApi.getTasks.mockRejectedValue(new Error('API Error'));

      const { result } = renderHook(() => useTasksQuery(), {
        wrapper: createWrapper(),
      });

      await waitFor(() => {
        expect(result.current.error).toBeTruthy();
      });
    });
  });

  describe('useCreateTaskMutation', () => {
    it('should create task successfully', async () => {
      const newTask = { title: 'New Task', description: 'Description', priority: 'medium' as const };
      const createdTask = { ...newTask, id: '1', status: 'pending' as const, createdAt: '2024-01-01', updatedAt: '2024-01-01', userId: 'user1' };
      
      mockedTaskApi.createTask.mockResolvedValue(createdTask);

      const { result } = renderHook(() => useCreateTaskMutation(), {
        wrapper: createWrapper(),
      });

      result.current.mutate(newTask);

      await waitFor(() => {
        expect(result.current.isSuccess).toBe(true);
      });
      
      expect(mockedTaskApi.createTask).toHaveBeenCalledWith(newTask);
    });
  });
});
```

### 5.2. Component Tests

**`src/entities/task/ui/__tests__/TaskCard.test.tsx`**
```typescript
import React from 'react';
import { render, screen } from '@testing-library/react';
import { TaskCard } from '../TaskCard';
import type { Task } from '../../model/task';

const mockTask: Task = {
  id: '1',
  title: 'Test Task',
  description: 'This is a test task',
  status: 'pending',
  priority: 'high',
  createdAt: '2024-01-01T00:00:00Z',
  updatedAt: '2024-01-01T00:00:00Z',
  userId: 'user1',
};

describe('TaskCard', () => {
  it('should render task information correctly', () => {
    render(<TaskCard task={mockTask} />);
    
    expect(screen.getByText('Test Task')).toBeInTheDocument();
    expect(screen.getByText('This is a test task')).toBeInTheDocument();
    expect(screen.getByText('pending')).toBeInTheDocument();
    expect(screen.getByText('high')).toBeInTheDocument();
  });

  it('should apply correct status styling', () => {
    render(<TaskCard task={mockTask} />);
    
    const statusBadge = screen.getByText('pending');
    expect(statusBadge).toHaveClass('bg-yellow-100', 'text-yellow-800');
  });

  it('should apply correct priority styling', () => {
    render(<TaskCard task={mockTask} />);
    
    const priorityBadge = screen.getByText('high');
    expect(priorityBadge).toHaveClass('bg-red-100', 'text-red-800');
  });
});
```

### 5.3. Integration Test

**`src/__tests__/task-flow.test.tsx`**
```typescript
import React from 'react';
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { TasksPage } from 'pages/tasks';
import { taskApi } from 'entities/task/api/task';

jest.mock('entities/task/api/task');
const mockedTaskApi = taskApi as jest.Mocked<typeof taskApi>;

const createTestWrapper = () => {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false },
    },
  });
  
  return ({ children }: { children: React.ReactNode }) => (
    <QueryClientProvider client={queryClient}>{children}</QueryClientProvider>
  );
};

describe('Task Flow Integration', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  it('should complete full task creation flow', async () => {
    const user = userEvent.setup();
    
    // Mock empty initial state
    mockedTaskApi.getTasks.mockResolvedValue([]);
    
    // Mock successful task creation
    const newTask = {
      id: '1',
      title: 'New Task',
      description: 'Task description',
      status: 'pending' as const,
      priority: 'medium' as const,
      createdAt: '2024-01-01',
      updatedAt: '2024-01-01',
      userId: 'user1',
    };
    mockedTaskApi.createTask.mockResolvedValue(newTask);
    
    render(<TasksPage />, { wrapper: createTestWrapper() });

    // Wait for initial load
    await waitFor(() => {
      expect(screen.getByText('No tasks yet. Create your first task!')).toBeInTheDocument();
    });

    // Click Add Task button
    const addButton = screen.getByText('Add Task');
    await user.click(addButton);

    // Fill out form
    const titleInput = screen.getByPlaceholderText('Enter task title');
    const descriptionInput = screen.getByPlaceholderText('Enter task description');
    
    await user.type(titleInput, 'New Task');
    await user.type(descriptionInput, 'Task description');

    // Submit form
    const createButton = screen.getByText('Create Task');
    await user.click(createButton);

    // Verify API call
    await waitFor(() => {
      expect(mockedTaskApi.createTask).toHaveBeenCalledWith({
        title: 'New Task',
        description: 'Task description',
        priority: 'medium',
      });
    });
  });
});
```

## Step 6: Running and Testing

### 6.1. Development Commands

```bash
# Start development server
npm run dev

# Run tests
npm run test

# Run tests in watch mode
npm run test:watch

# Run linting
npm run lint

# Run type checking
npm run type-check

# Build for production
npm run build
```

### 6.2. Testing Checklist

- [ ] Entity layer unit tests pass
- [ ] Feature component tests pass
- [ ] API integration tests pass
- [ ] Authentication flow works
- [ ] CRUD operations work end-to-end
- [ ] Error handling works correctly
- [ ] TypeScript compilation passes
- [ ] ESLint rules pass

## Summary

You've now implemented a complete first feature following our architectural conventions:

✅ **Entity Layer**: Task entity with TypeScript types, API client, and TanStack Query hooks  
✅ **Feature Layer**: CRUD operations with form validation using React Hook Form + Zod  
✅ **Page Layer**: Complete task management page with TanStack Router  
✅ **Authentication**: OAuth 2.0 flow with JWT token management using Zustand  
✅ **Testing**: Comprehensive test coverage with Vitest and React Testing Library  

This foundation demonstrates all the key patterns you'll use for future features in your application.

---

**Next Steps**: Continue to [Quality Assurance](./14-quality-assurance.md) for final verification and deployment preparation.