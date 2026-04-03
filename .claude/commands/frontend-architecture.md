# Frontend Architecture Design

You are a senior frontend architect. Generate a complete, production-ready frontend architecture plan for the user's project.

Analyze the user's request and generate a detailed architecture document covering the sections below. Tailor technology choices to the project's scale and complexity.

## Core Implementation Rules (Must Follow)

1. **Always add descriptions**: Every page, component, and section must include descriptive text (title + subtitle/description) to explain its purpose to the user.

2. **Use Tailwind CSS exclusively**: All styling must use Tailwind utility classes. No inline styles, no CSS modules, no separate CSS files.

3. **Unified shared components — no exceptions**:
   - Any UI element used more than once (Button, Input, Badge, Card, Modal, Table, PageHeader, EmptyState, etc.) **must** be extracted to `shared/components/`.
   - All instances across the entire app must import from `shared/components` — never redefine locally.
   - Shared components own their Tailwind styles, variants, sizes, and interactive behaviors (loading, disabled, hover, focus states). Feature components must not override or re-implement these.
   - When generating any feature code, always import shared components rather than writing raw HTML elements like `<button>` or `<input>` directly.

---

## Architecture Output Format

Generate a comprehensive architecture plan in Traditional Chinese (繁體中文) with code examples in English. The plan must include all of the following sections:

---

### 1. 技術棧選擇 (Technology Stack)

Based on project complexity, recommend:

- **Framework**: React 19 + TypeScript (strict mode)
- **Build Tool**: Vite
- **Server State**: TanStack Query v5 (React Query) — for all async data fetching, caching, and synchronization
- **Client State**:
  - Simple/Medium projects: Zustand
  - Complex projects with complex shared state: Redux Toolkit + RTK Query
- **HTTP Client**: Axios with a custom instance (interceptors for auth, error normalization)
- **Routing**: React Router v6 (data router pattern)
- **Styling**: Tailwind CSS + shadcn/ui or CSS Modules
- **Form**: React Hook Form + Zod
- **Testing**: Vitest + React Testing Library + MSW (Mock Service Worker)

---

### 2. 目錄結構 (Directory Structure)

Use **Feature-based modular architecture**. Co-locate everything a feature needs. Only extract to shared when used by 2+ features.

```
src/
├── app/                          # App-level setup (providers, router, global styles)
│   ├── providers.tsx             # All context providers composed here
│   ├── router.tsx                # Route definitions
│   └── App.tsx
│
├── core/                         # Shared infrastructure (no business logic)
│   ├── http/
│   │   ├── axios-instance.ts     # Configured axios instance
│   │   ├── http-client.ts        # Base HTTP service class
│   │   └── query-client.ts       # TanStack Query client config
│   ├── error/
│   │   ├── types.ts              # ApiError, AppError types
│   │   ├── error-handler.ts      # Global error handler
│   │   └── error-boundary.tsx    # React error boundary
│   ├── auth/
│   │   ├── auth-store.ts         # Zustand auth store
│   │   ├── token.ts              # Token storage/refresh helpers
│   │   └── use-auth.ts           # Auth hook
│   ├── router/
│   │   ├── protected-route.tsx   # Auth guard wrapper
│   │   └── route-constants.ts    # Typed route paths
│   └── ui/
│       ├── loading.tsx           # Global loading spinner
│       ├── toast.ts              # Toast notification helper
│       └── confirm-dialog.tsx    # Reusable confirm modal
│
├── features/                     # Feature modules (self-contained)
│   └── [feature-name]/
│       ├── api/
│       │   ├── [feature].api.ts  # API call functions (use http-client)
│       │   └── [feature].keys.ts # TanStack Query key factory
│       ├── hooks/
│       │   ├── use-[feature].ts  # Query/mutation hooks
│       │   └── use-[feature]-store.ts  # Feature-local Zustand slice
│       ├── components/
│       │   ├── [Feature]Page.tsx
│       │   └── [Feature]Form.tsx
│       ├── types/
│       │   └── index.ts          # Feature-specific DTOs and types
│       └── index.ts              # Public API of the feature (barrel export)
│
├── shared/                       # Reusable UI components and utilities
│   ├── components/               # Generic UI components (Button, Modal, Table…)
│   ├── hooks/                    # Generic hooks (useDebounce, usePagination…)
│   └── utils/                    # Pure utility functions
│
└── types/                        # Global type declarations
    ├── api.ts                    # ApiResponse<T>, PaginatedResponse<T>, ApiError
    └── env.d.ts                  # Vite env variable types
```

---

### 3. 全局類型定義 (Global Type System)

Define in `src/types/api.ts`:

```typescript
// Standard API response wrapper (matches backend contract)
export interface ApiResponse<T = unknown> {
  data: T;
  message?: string;
  code?: number;
}

export interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  pageSize: number;
}

// Normalized error shape (after interceptor processing)
export interface ApiError {
  type: string;          // e.g. "token_expired", "validation_error"
  msg: string;           // Human-readable message
  code: number;          // HTTP status code
  originalError?: unknown;
}

// Error types enum for type-safe error handling
export enum ApiErrorType {
  TOKEN_EXPIRED    = "token_expired",
  TOKEN_INVALID    = "token_invalid",
  TOKEN_MISSING    = "token_missing",
  NOT_FOUND        = "not_found",
  FORBIDDEN        = "forbidden",
  VALIDATION       = "validation_error",
  SERVER_ERROR     = "server_error",
  NETWORK_ERROR    = "network_error",
  UNKNOWN          = "unknown",
}
```

---

### 4. Core HTTP 模組 (Core HTTP Module)

**`src/core/http/axios-instance.ts`** — Single axios instance, all requests go through this:

```typescript
import axios from "axios";
import { normalizeApiError } from "@/core/error/error-handler";
import { getToken, refreshToken } from "@/core/auth/token";

export const axiosInstance = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 15_000,
  headers: { "Content-Type": "application/json" },
});

// Request interceptor: attach auth token
axiosInstance.interceptors.request.use((config) => {
  const token = getToken();
  if (token) config.headers.Authorization = `Bearer ${token}`;
  return config;
});

// Response interceptor: normalize errors globally
axiosInstance.interceptors.response.use(
  (res) => res,
  async (error) => {
    const originalRequest = error.config;

    // Auto-refresh on 401
    if (error.response?.status === 401 && !originalRequest._retry) {
      originalRequest._retry = true;
      const newToken = await refreshToken();
      if (newToken) {
        originalRequest.headers.Authorization = `Bearer ${newToken}`;
        return axiosInstance(originalRequest);
      }
    }

    // Normalize to ApiError and rethrow
    return Promise.reject(normalizeApiError(error));
  }
);
```

**`src/core/http/http-client.ts`** — Base class for feature API modules:

```typescript
import { axiosInstance } from "./axios-instance";
import type { ApiResponse } from "@/types/api";
import type { ApiError } from "@/types/api";
import { handleError } from "@/core/error/error-handler";
import type { AxiosRequestConfig } from "axios";

type ErrorHandlerFn = (error: ApiError) => void;

export class HttpClient {
  protected readonly basePath: string;

  constructor(basePath: string) {
    this.basePath = basePath;
  }

  protected async get<T>(
    path: string,
    config?: AxiosRequestConfig,
    onError?: ErrorHandlerFn
  ): Promise<T> {
    try {
      const res = await axiosInstance.get<ApiResponse<T>>(
        `${this.basePath}${path}`,
        config
      );
      return res.data.data;
    } catch (err) {
      // Use local override or fall back to global handler
      (onError ?? handleError)(err as ApiError);
      throw err;
    }
  }

  protected async post<T>(
    path: string,
    body?: unknown,
    config?: AxiosRequestConfig,
    onError?: ErrorHandlerFn
  ): Promise<T> {
    try {
      const res = await axiosInstance.post<ApiResponse<T>>(
        `${this.basePath}${path}`,
        body,
        config
      );
      return res.data.data;
    } catch (err) {
      (onError ?? handleError)(err as ApiError);
      throw err;
    }
  }

  protected async put<T>(
    path: string,
    body?: unknown,
    config?: AxiosRequestConfig,
    onError?: ErrorHandlerFn
  ): Promise<T> {
    try {
      const res = await axiosInstance.put<ApiResponse<T>>(
        `${this.basePath}${path}`,
        body,
        config
      );
      return res.data.data;
    } catch (err) {
      (onError ?? handleError)(err as ApiError);
      throw err;
    }
  }

  protected async delete<T>(
    path: string,
    config?: AxiosRequestConfig,
    onError?: ErrorHandlerFn
  ): Promise<T> {
    try {
      const res = await axiosInstance.delete<ApiResponse<T>>(
        `${this.basePath}${path}`,
        config
      );
      return res.data.data;
    } catch (err) {
      (onError ?? handleError)(err as ApiError);
      throw err;
    }
  }
}
```

---

### 5. 全局錯誤處理 (Global Error Handling)

**`src/core/error/error-handler.ts`**:

```typescript
import { ApiErrorType, type ApiError } from "@/types/api";
import { toast } from "@/core/ui/toast";
import { authStore } from "@/core/auth/auth-store";
import { ROUTES } from "@/core/router/route-constants";

// Normalize raw axios error → ApiError
export function normalizeApiError(error: unknown): ApiError {
  if (axios.isAxiosError(error)) {
    const status = error.response?.status ?? 0;
    const body   = error.response?.data;

    return {
      type: body?.type ?? resolveErrorType(status),
      msg:  body?.msg  ?? error.message,
      code: status,
      originalError: error,
    };
  }
  return { type: ApiErrorType.UNKNOWN, msg: String(error), code: 0 };
}

function resolveErrorType(status: number): ApiErrorType {
  const map: Record<number, ApiErrorType> = {
    401: ApiErrorType.TOKEN_EXPIRED,
    403: ApiErrorType.FORBIDDEN,
    404: ApiErrorType.NOT_FOUND,
    422: ApiErrorType.VALIDATION,
    500: ApiErrorType.SERVER_ERROR,
  };
  return map[status] ?? ApiErrorType.UNKNOWN;
}

// Global default error handler — shows toast and handles auth errors
export function handleError(error: ApiError): void {
  switch (error.type) {
    case ApiErrorType.TOKEN_EXPIRED:
    case ApiErrorType.TOKEN_INVALID:
    case ApiErrorType.TOKEN_MISSING:
      authStore.getState().logout();
      window.location.href = ROUTES.LOGIN;
      break;

    case ApiErrorType.FORBIDDEN:
      toast.error("您沒有權限執行此操作");
      break;

    case ApiErrorType.NOT_FOUND:
      toast.error("找不到請求的資源");
      break;

    case ApiErrorType.VALIDATION:
      toast.error(error.msg || "輸入資料有誤，請重新確認");
      break;

    case ApiErrorType.SERVER_ERROR:
      toast.error("伺服器發生錯誤，請稍後再試");
      break;

    default:
      toast.error(error.msg || "發生未知錯誤");
  }
}
```

**`src/core/error/error-boundary.tsx`**:

```typescript
import { Component, type ReactNode } from "react";

interface Props { children: ReactNode; fallback?: ReactNode; }
interface State { hasError: boolean; error?: Error; }

export class GlobalErrorBoundary extends Component<Props, State> {
  state: State = { hasError: false };

  static getDerivedStateFromError(error: Error): State {
    return { hasError: true, error };
  }

  componentDidCatch(error: Error) {
    console.error("[GlobalErrorBoundary]", error);
  }

  render() {
    if (this.state.hasError) {
      return this.props.fallback ?? (
        <div className="error-screen">
          <h2>頁面發生錯誤</h2>
          <button onClick={() => this.setState({ hasError: false })}>
            重新整理
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}
```

TanStack Query 全局錯誤配置 (`src/core/http/query-client.ts`):

```typescript
import { QueryClient } from "@tanstack/react-query";
import { handleError } from "@/core/error/error-handler";
import type { ApiError } from "@/types/api";

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5,   // 5 min
      retry: (failureCount, error) => {
        const apiErr = error as ApiError;
        // Don't retry on auth/validation/not-found errors
        if ([401, 403, 404, 422].includes(apiErr.code)) return false;
        return failureCount < 2;
      },
    },
    mutations: {
      onError: (error) => handleError(error as ApiError), // Global mutation error
    },
  },
});
```

---

### 6. Feature Module 範例 (Feature Module Example)

以 `features/users` 為例，示範如何繼承 `HttpClient` 並 override error handler：

**`src/features/users/types/index.ts`**:
```typescript
export interface User {
  id: number;
  name: string;
  email: string;
  role: string;
}

export interface CreateUserDTO {
  name: string;
  email: string;
  roleId: number;
}
```

**`src/features/users/api/users.api.ts`**:
```typescript
import { HttpClient } from "@/core/http/http-client";
import type { User, CreateUserDTO } from "../types";

class UsersApi extends HttpClient {
  constructor() {
    super("/users");
  }

  getAll() {
    return this.get<User[]>("/");
  }

  getById(id: number) {
    return this.get<User>(`/${id}`);
  }

  create(dto: CreateUserDTO) {
    // Override error handler for this specific call
    return this.post<User>("/", dto, undefined, (err) => {
      if (err.type === "validation_error") {
        toast.error(`建立失敗：${err.msg}`);
      } else {
        handleError(err); // Fall back to global for other errors
      }
    });
  }

  update(id: number, dto: Partial<CreateUserDTO>) {
    return this.put<User>(`/${id}`, dto);
  }

  remove(id: number) {
    return this.delete<void>(`/${id}`);
  }
}

export const usersApi = new UsersApi();
```

**`src/features/users/api/users.keys.ts`**:
```typescript
export const usersKeys = {
  all:    ()         => ["users"]               as const,
  lists:  ()         => ["users", "list"]       as const,
  detail: (id: number) => ["users", "detail", id] as const,
};
```

**`src/features/users/hooks/use-users.ts`**:
```typescript
import { useQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { usersApi } from "../api/users.api";
import { usersKeys } from "../api/users.keys";
import type { CreateUserDTO } from "../types";

export function useUsers() {
  return useQuery({
    queryKey: usersKeys.lists(),
    queryFn: () => usersApi.getAll(),
  });
}

export function useUser(id: number) {
  return useQuery({
    queryKey: usersKeys.detail(id),
    queryFn: () => usersApi.getById(id),
    enabled: !!id,
  });
}

export function useCreateUser() {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: (dto: CreateUserDTO) => usersApi.create(dto),
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: usersKeys.lists() });
    },
    // onError is optional here — api layer already handles it
  });
}
```

---

### 7. 其他共用 Core 模組 (Additional Core Modules)

#### 7a. Auth Store (`src/core/auth/auth-store.ts`)
```typescript
import { create } from "zustand";
import { persist } from "zustand/middleware";
import type { User } from "@/features/users/types";

interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  login: (user: User, token: string) => void;
  logout: () => void;
}

export const authStore = create<AuthState>()(
  persist(
    (set) => ({
      user: null,
      token: null,
      isAuthenticated: false,
      login: (user, token) => set({ user, token, isAuthenticated: true }),
      logout: () => set({ user: null, token: null, isAuthenticated: false }),
    }),
    { name: "auth-store" }
  )
);

export const useAuth = () => authStore();
```

#### 7b. Protected Route (`src/core/router/protected-route.tsx`)
```typescript
import { Navigate, Outlet, useLocation } from "react-router-dom";
import { useAuth } from "@/core/auth/auth-store";
import { ROUTES } from "./route-constants";

export function ProtectedRoute() {
  const { isAuthenticated } = useAuth();
  const location = useLocation();

  if (!isAuthenticated) {
    return <Navigate to={ROUTES.LOGIN} state={{ from: location }} replace />;
  }
  return <Outlet />;
}
```

#### 7c. Pagination Hook (`src/shared/hooks/use-pagination.ts`)
```typescript
import { useState } from "react";

export function usePagination(initialPageSize = 20) {
  const [page, setPage]         = useState(1);
  const [pageSize, setPageSize] = useState(initialPageSize);

  const reset = () => setPage(1);

  return {
    page, pageSize,
    setPage, setPageSize, reset,
    queryParams: { page, pageSize },
  };
}
```

#### 7d. Form with Zod (`src/features/[feature]/components/[Feature]Form.tsx`)
```typescript
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const schema = z.object({
  name:  z.string().min(1, "名稱不可為空"),
  email: z.string().email("請輸入有效的 Email"),
});

type FormValues = z.infer<typeof schema>;

export function UserForm({ onSubmit }: { onSubmit: (v: FormValues) => void }) {
  const { register, handleSubmit, formState: { errors } } = useForm<FormValues>({
    resolver: zodResolver(schema),
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register("name")} />
      {errors.name && <span>{errors.name.message}</span>}
      <input {...register("email")} />
      {errors.email && <span>{errors.email.message}</span>}
      <button type="submit">送出</button>
    </form>
  );
}
```

#### 7e. App Providers (`src/app/providers.tsx`)
```typescript
import { QueryClientProvider } from "@tanstack/react-query";
import { ReactQueryDevtools } from "@tanstack/react-query-devtools";
import { Toaster } from "sonner";
import { queryClient } from "@/core/http/query-client";
import { GlobalErrorBoundary } from "@/core/error/error-boundary";

export function Providers({ children }: { children: React.ReactNode }) {
  return (
    <GlobalErrorBoundary>
      <QueryClientProvider client={queryClient}>
        {children}
        <Toaster richColors position="top-right" />
        {import.meta.env.DEV && <ReactQueryDevtools />}
      </QueryClientProvider>
    </GlobalErrorBoundary>
  );
}
```

---

### 8. 錯誤處理覆蓋規則 (Error Handling Override Rules)

| Layer | Handles | Override? |
|-------|---------|-----------|
| Axios interceptor | Token refresh, normalize to `ApiError` | No — always runs |
| `handleError()` | Global toast + auth redirect | Yes — pass `onError` to HttpClient methods |
| `queryClient.defaultOptions.mutations.onError` | Mutation fallback | Yes — define `onError` in `useMutation()` |
| `GlobalErrorBoundary` | Render crashes | Yes — per-route `<ErrorBoundary>` |

**Override priority**: Feature-level `onError` → Query/Mutation `onError` → Global `handleError` → ErrorBoundary

---

### 9. 環境變數規範 (Environment Variables)

```bash
# .env.example
VITE_API_BASE_URL=http://localhost:8000
VITE_APP_ENV=development
```

```typescript
// src/types/env.d.ts
interface ImportMetaEnv {
  readonly VITE_API_BASE_URL: string;
  readonly VITE_APP_ENV: "development" | "staging" | "production";
}
```

---

### Output Instructions

After presenting the architecture plan, ask the user:
1. 確認技術棧選擇是否符合需求（是否需要調整 state management 方案？）
2. 需要產生哪個 feature 的完整程式碼骨架？
3. 需要補充哪些 core 模組的實作細節？
