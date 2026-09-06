# React Frontend: Production Reference Templates

Copy-pasteable, production-ready blueprints adhering strictly to the `reactjs-frontend` architecture.

---

## 1. Fail-Fast Environment Validation (`src/config/env.config.ts`)

```typescript
import { z } from 'zod';

const envSchema = z.object({
  VITE_API_BASE_URL: z.string().url('VITE_API_BASE_URL must be a valid URL'),
  VITE_APP_ENV: z.enum(['development', 'staging', 'production']).default('development'),
  VITE_ENABLE_MOCKS: z
    .string()
    .optional()
    .transform((val) => val === 'true'),
});

const parsed = envSchema.safeParse(import.meta.env);

if (!parsed.success) {
  console.error('❌ Invalid environment variables:', parsed.error.format());
  throw new Error('Invalid environment variables. Application initialization halted.');
}

export const env = parsed.data;
```

---

## 2. API Envelope Contracts (`src/types/api.types.ts`)

```typescript
export interface ApiResponseSuccess<T> {
  ok: true;
  data: T;
  meta?: {
    requestId?: string;
    timestamp?: string;
  };
}

export interface ApiResponseError {
  ok: false;
  error: {
    code: string;
    message: string;
    details?: unknown;
  };
  meta?: {
    requestId?: string;
    timestamp?: string;
  };
}

export type ApiResponse<T> = ApiResponseSuccess<T> | ApiResponseError;

export interface PaginatedData<T> {
  items: T[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
}
```

---

## 3. Type-Safe API Client (`src/lib/api.ts`)

Unwraps `{ ok: true, data: T }` responses from `bun-backend` and throws typed `ApiClientError` on failure.

```typescript
import { env } from '@/config/env.config';
import { useAuthStore } from '@/stores/auth.store';
import type { ApiResponse } from '@/types/api.types';

export class ApiClientError extends Error {
  constructor(
    public readonly status: number,
    public readonly code: string,
    message: string,
    public readonly details?: unknown
  ) {
    super(message);
    this.name = 'ApiClientError';
  }
}

interface RequestOptions extends RequestInit {
  params?: Record<string, string | number | boolean | undefined | null>;
}

async function request<T>(endpoint: string, options: RequestOptions = {}): Promise<T> {
  const { params, headers: customHeaders, ...restOptions } = options;

  let url = `${env.VITE_API_BASE_URL.replace(/\/$/, '')}/${endpoint.replace(/^\//, '')}`;

  if (params) {
    const searchParams = new URLSearchParams();
    Object.entries(params).forEach(([key, value]) => {
      if (value !== undefined && value !== null) {
        searchParams.append(key, String(value));
      }
    });
    const queryString = searchParams.toString();
    if (queryString) {
      url += `?${queryString}`;
    }
  }

  const token = useAuthStore.getState().token;

  const headers = new Headers(customHeaders);
  headers.set('Accept', 'application/json');

  if (restOptions.body && !(restOptions.body instanceof FormData)) {
    headers.set('Content-Type', 'application/json');
  }

  if (token) {
    headers.set('Authorization', `Bearer ${token}`);
  }

  const response = await fetch(url, {
    ...restOptions,
    headers,
  });

  const rawJson = (await response.json().catch(() => null)) as ApiResponse<T> | null;

  if (!response.ok || !rawJson || !rawJson.ok) {
    const errorPayload = rawJson && !rawJson.ok ? rawJson.error : undefined;
    throw new ApiClientError(
      response.status,
      errorPayload?.code ?? 'UNKNOWN_ERROR',
      errorPayload?.message ?? response.statusText ?? 'An unexpected network error occurred',
      errorPayload?.details
    );
  }

  return rawJson.data;
}

export const api = {
  get: <T>(endpoint: string, options?: RequestOptions) =>
    request<T>(endpoint, { ...options, method: 'GET' }),

  post: <T>(endpoint: string, body?: unknown, options?: RequestOptions) =>
    request<T>(endpoint, {
      ...options,
      method: 'POST',
      body: body instanceof FormData ? body : JSON.stringify(body),
    }),

  put: <T>(endpoint: string, body?: unknown, options?: RequestOptions) =>
    request<T>(endpoint, {
      ...options,
      method: 'PUT',
      body: body instanceof FormData ? body : JSON.stringify(body),
    }),

  patch: <T>(endpoint: string, body?: unknown, options?: RequestOptions) =>
    request<T>(endpoint, {
      ...options,
      method: 'PATCH',
      body: body instanceof FormData ? body : JSON.stringify(body),
    }),

  delete: <T>(endpoint: string, options?: RequestOptions) =>
    request<T>(endpoint, { ...options, method: 'DELETE' }),
};
```

---

## 4. Query Client & Query Key Factory (`src/lib/queryClient.ts`)

```typescript
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60, // 1 minute fresh time
      gcTime: 1000 * 60 * 5, // 5 minutes garbage collection time
      retry: (failureCount, error) => {
        // Do not retry on client errors (4xx)
        if (typeof error === 'object' && error !== null && 'status' in error) {
          const status = (error as { status: number }).status;
          if (status >= 400 && status < 500) return false;
        }
        return failureCount < 2;
      },
      refetchOnWindowFocus: false,
    },
    mutations: {
      retry: false,
    },
  },
});
```

---

## 5. Global Client State Store (`src/stores/auth.store.ts`)

```typescript
import { create } from 'zustand';
import { persist, createJSONStorage } from 'zustand/middleware';

export interface UserProfile {
  id: string;
  name: string;
  email: string;
  role: string;
}

interface AuthState {
  user: UserProfile | null;
  token: string | null;
  isAuthenticated: boolean;
  setAuth: (user: UserProfile, token: string) => void;
  clearAuth: () => void;
}

export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({
      user: null,
      token: null,
      isAuthenticated: false,
      setAuth: (user, token) =>
        set({
          user,
          token,
          isAuthenticated: true,
        }),
      clearAuth: () =>
        set({
          user: null,
          token: null,
          isAuthenticated: false,
        }),
    }),
    {
      name: 'app_auth_session',
      storage: createJSONStorage(() => localStorage),
    }
  )
);
```

---

## 6. Complete Vertical Feature Slice (`src/features/users/`)

### A. Domain Types (`src/features/users/types/user.types.ts`)
```typescript
export interface User {
  id: string;
  name: string;
  email: string;
  role: 'admin' | 'member' | 'guest';
  createdAt: string;
}

export interface UserFilterParams {
  page?: number;
  limit?: number;
  search?: string;
}
```

### B. Validation Schemas (`src/features/users/schemas/user.schema.ts`)
```typescript
import { z } from 'zod';

export const createUserSchema = z.object({
  name: z.string().trim().min(2, 'Name must be at least 2 characters'),
  email: z.string().trim().toLowerCase().email('Invalid email address'),
  role: z.enum(['admin', 'member', 'guest']).default('member'),
});

export type CreateUserInput = z.infer<typeof createUserSchema>;
```

### C. Endpoints (`src/features/users/api/endpoints.ts`)
```typescript
import { api } from '@/lib/api';
import type { PaginatedData } from '@/types/api.types';
import type { CreateUserInput } from '../schemas/user.schema';
import type { User, UserFilterParams } from '../types/user.types';

export const userEndpoints = {
  getUsers: (params?: UserFilterParams) =>
    api.get<PaginatedData<User>>('/users', { params }),

  getUserById: (id: string) =>
    api.get<User>(`/users/${id}`),

  createUser: (data: CreateUserInput) =>
    api.post<User>('/users', data),
};
```

### D. Queries & Key Factory (`src/features/users/api/queries.ts`)
```typescript
import { queryOptions, useQuery } from '@tanstack/react-query';
import { userEndpoints } from './endpoints';
import type { UserFilterParams } from '../types/user.types';

export const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  list: (params?: UserFilterParams) => [...userKeys.lists(), params] as const,
  details: () => [...userKeys.all, 'detail'] as const,
  detail: (id: string) => [...userKeys.details(), id] as const,
};

export const userQueryOptions = {
  list: (params?: UserFilterParams) =>
    queryOptions({
      queryKey: userKeys.list(params),
      queryFn: () => userEndpoints.getUsers(params),
    }),
  detail: (id: string) =>
    queryOptions({
      queryKey: userKeys.detail(id),
      queryFn: () => userEndpoints.getUserById(id),
      enabled: Boolean(id),
    }),
};

export function useUsersQuery(params?: UserFilterParams) {
  return useQuery(userQueryOptions.list(params));
}

export function useUserDetailQuery(id: string) {
  return useQuery(userQueryOptions.detail(id));
}
```

### E. Mutations (`src/features/users/api/mutations.ts`)
```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { toast } from 'sonner';
import { userEndpoints } from './endpoints';
import { userKeys } from './queries';
import type { CreateUserInput } from '../schemas/user.schema';

export function useCreateUserMutation() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (data: CreateUserInput) => userEndpoints.createUser(data),
    onSuccess: (newUser) => {
      queryClient.invalidateQueries({ queryKey: userKeys.lists() });
      toast.success(`User "${newUser.name}" created successfully`);
    },
    onError: (error: Error) => {
      toast.error(error.message || 'Failed to create user');
    },
  });
}
```

### F. Headless Custom Hook (`src/features/users/hooks/useUserState.ts`)
```typescript
import { useState } from 'react';
import { useCreateUserMutation } from '../api/mutations';
import { createUserSchema, type CreateUserInput } from '../schemas/user.schema';

export function useUserState() {
  const [formData, setFormData] = useState<CreateUserInput>({
    name: '',
    email: '',
    role: 'member',
  });
  const [errors, setErrors] = useState<Record<string, string>>({});

  const createMutation = useCreateUserMutation();

  const handleFieldChange = (field: keyof CreateUserInput, value: string) => {
    setFormData((prev) => ({ ...prev, [field]: value }));
    if (errors[field]) {
      setErrors((prev) => {
        const next = { ...prev };
        delete next[field];
        return next;
      });
    }
  };

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    const result = createUserSchema.safeParse(formData);

    if (!result.success) {
      const fieldErrors: Record<string, string> = {};
      result.error.issues.forEach((issue) => {
        if (issue.path[0]) {
          fieldErrors[issue.path[0].toString()] = issue.message;
        }
      });
      setErrors(fieldErrors);
      return;
    }

    setErrors({});
    await createMutation.mutateAsync(result.data);
    setFormData({ name: '', email: '', role: 'member' });
  };

  return {
    formData,
    errors,
    handleFieldChange,
    handleSubmit,
    isSubmitting: createMutation.isPending,
  };
}
```

### G. Presentational Component (`src/features/users/components/userCard.tsx`)
```tsx
import { Mail, Shield, User as UserIcon } from 'lucide-react';
import type { User } from '../types/user.types';

interface UserCardProps {
  user: User;
  onSelect?: (id: string) => void;
}

export function UserCard({ user, onSelect }: UserCardProps) {
  return (
    <div
      onClick={() => onSelect?.(user.id)}
      className="group relative flex flex-col justify-between rounded-lg border border-neutral-200 bg-white p-5 shadow-xs transition hover:border-neutral-300 hover:shadow-sm"
    >
      <div className="flex items-start justify-between gap-3">
        <div className="flex items-center gap-3">
          <div className="flex size-9 items-center justify-center rounded-full bg-neutral-100 text-neutral-600">
            <UserIcon className="size-4.5" strokeWidth={1.75} />
          </div>
          <div>
            <h3 className="text-sm font-semibold text-neutral-900">{user.name}</h3>
            <div className="flex items-center gap-1.5 text-xs text-neutral-500">
              <Mail className="size-3.5" strokeWidth={1.75} />
              <span>{user.email}</span>
            </div>
          </div>
        </div>
        <span className="inline-flex items-center gap-1 rounded-md bg-neutral-50 px-2 py-1 text-xs font-medium text-neutral-600 ring-1 ring-neutral-200/50">
          <Shield className="size-3 text-neutral-400" strokeWidth={1.75} />
          {user.role}
        </span>
      </div>
    </div>
  );
}
```

### H. Co-located Vitest Component Test (`src/features/users/components/userCard.test.tsx`)
```tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import { UserCard } from './userCard';
import type { User } from '../types/user.types';

const mockUser: User = {
  id: 'usr-123',
  name: 'Jane Doe',
  email: 'jane@example.com',
  role: 'admin',
  createdAt: '2026-01-01T00:00:00Z',
};

describe('UserCard', () => {
  it('renders user details cleanly', () => {
    render(<UserCard user={mockUser} />);

    expect(screen.getByText('Jane Doe')).toBeInTheDocument();
    expect(screen.getByText('jane@example.com')).toBeInTheDocument();
    expect(screen.getByText('admin')).toBeInTheDocument();
  });

  it('triggers onSelect with correct ID when clicked', () => {
    const handleSelect = vi.fn();
    render(<UserCard user={mockUser} onSelect={handleSelect} />);

    fireEvent.click(screen.getByText('Jane Doe'));
    expect(handleSelect).toHaveBeenCalledWith('usr-123');
  });
});
```

### I. Feature Public Barrel Export (`src/features/users/index.ts`)
```typescript
export { UserCard } from './components/userCard';
export { useUserState } from './hooks/useUserState';
export { useUsersQuery, useUserDetailQuery, userKeys } from './api/queries';
export { useCreateUserMutation } from './api/mutations';
export type { User, UserFilterParams } from './types/user.types';
export type { CreateUserInput } from './schemas/user.schema';
```

---

## 7. TanStack Router Core Setup (`src/app/router.tsx`)

```tsx
import {
  createRouter,
  createRootRoute,
  createRoute,
  Outlet,
} from '@tanstack/react-router';
import { QueryClient } from '@tanstack/react-query';
import { Toaster } from 'sonner';

export interface RouterContext {
  queryClient: QueryClient;
}

const rootRoute = createRootRoute({
  component: () => (
    <div className="min-h-screen bg-neutral-50 text-neutral-900 antialiased">
      <Outlet />
      <Toaster richColors position="top-right" />
    </div>
  ),
});

const indexRoute = createRoute({
  getParentRoute: () => rootRoute,
  path: '/',
  component: () => (
    <main className="mx-auto max-w-5xl p-8">
      <h1 className="text-2xl font-bold tracking-tight text-neutral-900">Dashboard</h1>
    </main>
  ),
});

const routeTree = rootRoute.addChildren([indexRoute]);

export function createAppRouter(queryClient: QueryClient) {
  return createRouter({
    routeTree,
    context: { queryClient },
    defaultPreload: 'intent',
  });
}

declare module '@tanstack/react-router' {
  interface Register {
    router: ReturnType<typeof createAppRouter>;
  }
}
```

---

## 8. Co-located E2E Test (`tests/e2e/auth.spec.ts`)

```typescript
import { test, expect } from '@playwright/test';

test.describe('Authentication Journey', () => {
  test('redirects unauthenticated user to login view', async ({ page }) => {
    await page.goto('/dashboard');
    await expect(page).toHaveURL(/.*login/);
    await expect(page.getByRole('heading', { name: /Sign in/i })).toBeVisible();
  });
});
```
