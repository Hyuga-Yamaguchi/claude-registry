---
name: frontend-patterns-react
description: Reagent-inspired React SPA patterns with bulletproof-react structure - subscription-driven UI, event/effect separation, side-effects at the edges, with TanStack Query as standard.
---

# React Frontend Patterns (Reagent-inspired + Bulletproof Structure)

Design guide applying Reagent/Re-frame philosophy (subscription model, event-driven, effect isolation) to React SPA (react-router-dom), using [bulletproof-react](https://github.com/alan2207/bulletproof-react) as the structural foundation.

For language-level TypeScript standards, see [coding-standards-typescript.md](coding-standards-typescript.md).

---

## Philosophy: Bulletproof + Re-frame Layer

**bulletproof-react** provides structural foundation (directory structure, boundaries), while **Reagent/Re-frame** enforces behavioral rules (pure views, event-driven, effect isolation).

**What bulletproof-react provides:**
- Features-based organization (domain boundaries)
- Unidirectional codebase (shared → domains → features → app)
- State classification (Component / Application / Server Cache / Form / URL)

**What we add (Re-frame layer):**
- Pure View: Subscribe & Render only
- Event-driven: Changes through events
- Effect isolation: Side effects at boundaries
- Subscriptions: Derived data via selectors

---

## Tech Stack

### Core
- **[React](https://react.dev)** (18+), **[TypeScript](https://www.typescriptlang.org/)** (required), **[react-router-dom](https://reactrouter.com/)**

### State
- **[TanStack Query](https://tanstack.com/query)** (v5): Server state (required)
- **[jotai](https://jotai.org/)**: Application state (minimal UI-only)

### Forms & UI
- **[react-hook-form](https://react-hook-form.com/)** + **[zod](https://github.com/colinhacks/zod)**: Forms/validation
- **[Tailwind CSS](https://tailwindcss.com/)**: Styling
- **[sonner](https://sonner.emilkowal.ski/)**: Toast notifications
- **[DOMPurify](https://github.com/cure53/DOMPurify)**: XSS sanitization (when rendering user HTML)

### Testing
- **[Vitest](https://vitest.dev)**, **[Testing Library](https://testing-library.com/)**, **[Playwright](https://playwright.dev)**, **[MSW](https://mswjs.io)**

---

## Directory Structure

Based on [bulletproof-react](https://github.com/alan2207/bulletproof-react) with Reagent/Re-frame layer:

```
src/
├── app/                        # Application layer
│   ├── App.tsx                 # Main application component (simple)
│   ├── Provider.tsx            # Global providers (QueryClient, Theme, etc)
│   ├── Router.tsx              # Router configuration (all route definitions here, lazy imports only)
│   ├── events/                 # Cross-feature orchestration (strict scope, see rules below)
│   │   ├── types.ts            # App-internal Event/Handler input/output types only (references Command from shared/types/commands.ts, doesn't define it)
│   │   └── handlers/
│   ├── effects/                # Cross-feature side effects (strict scope, see rules below)
│   │   ├── analytics.effects.ts
│   │   └── logging.effects.ts
│   └── store/                  # Global application state (jotai atoms)
│       └── ui.atoms.ts
│
├── domains/                    # Cross-cutting domains (importable by features, ESLint allowed)
│   └── auth/
│       ├── pages/              # Route entry points
│       │   └── LoginPage.tsx
│       ├── ui/                 # Reusable components
│       │   └── ProtectedRoute.tsx
│       ├── data/
│       │   ├── api.ts
│       │   ├── keys.ts
│       │   ├── queries.ts
│       │   └── mutations.ts
│       ├── model/
│       │   ├── types.ts
│       │   └── schemas.ts
│       └── index.ts            # Public exports (recommended for domains)
│
├── shared/                     # Shared across all features
│   ├── ui/                     # Shared UI components
│   │   ├── Button.tsx          # Primitives
│   │   ├── Card.tsx
│   │   ├── Spinner.tsx         # Feedback components
│   │   ├── ErrorView.tsx
│   │   └── index.ts            # Barrel exports for convenience
│   ├── lib/                    # Shared utilities & configurations
│   │   ├── query-client.ts     # TanStack Query setup
│   │   ├── http.ts             # httpJson, httpText
│   │   ├── errors.ts           # ApiError class
│   │   └── hooks/              # Shared hooks
│   │       ├── use-toggle.ts
│   │       └── use-debounce.ts
│   └── types/                  # Truly shared types only (see guidelines below)
│       ├── common.ts           # Pagination, ApiResponse<T>, etc
│       └── commands.ts         # Command pattern types (used by features/domains)
│
└── features/                   # Feature modules (strictly isolated, no cross-import)
    ├── settings/               # Isolated feature
    │   ├── pages/              # Route entry points
    │   │   ├── SettingsPage.tsx
    │   │   ├── AccountsPage.tsx
    │   │   └── DepartmentsPage.tsx
    │   ├── ui/                 # UI components (not pages)
    │   │   ├── AccountList.tsx
    │   │   ├── AccountCard.tsx
    │   │   └── DepartmentForm.tsx
    │   ├── data/               # Split by resource for proper granularity
    │   │   ├── accounts/
    │   │   │   ├── api.ts
    │   │   │   ├── keys.ts
    │   │   │   ├── queries.ts
    │   │   │   └── mutations.ts
    │   │   └── departments/
    │   │       ├── api.ts
    │   │       ├── keys.ts
    │   │       ├── queries.ts
    │   │       └── mutations.ts
    │   ├── model/
    │   │   └── types.ts
    │
    └── dashboard/
        └── ...
```

### App Layer (Simple & Clean)

```typescript
// app/App.tsx (EXAMPLE)
import { AppProvider } from '@/app/Provider'
import { AppRouter } from '@/app/Router'

function App() {
  return (
    <AppProvider>
      <AppRouter />
    </AppProvider>
  )
}

export default App
```

```typescript
// app/Provider.tsx (EXAMPLE)
import { QueryClientProvider } from '@tanstack/react-query'
import { queryClient } from '@/shared/lib/query-client'

export function AppProvider({ children }: { children: React.ReactNode }) {
  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}
```

```typescript
// app/Router.tsx (EXAMPLE)
import { BrowserRouter, Routes, Route } from 'react-router-dom'
import { lazy, Suspense } from 'react'

// Lazy load from features/*/pages/ and domains/*/pages/
const SettingsPage = lazy(() => import('@/features/settings/pages/SettingsPage'))
const DashboardPage = lazy(() => import('@/features/dashboard/pages/DashboardPage'))
const LoginPage = lazy(() => import('@/domains/auth/pages/LoginPage'))

export function AppRouter() {
  return (
    <BrowserRouter>
      <Suspense fallback={<div>Loading...</div>}>
        <Routes>
          <Route path="/" element={<DashboardPage />} />
          <Route path="/settings" element={<SettingsPage />} />
          <Route path="/login" element={<LoginPage />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  )
}
```

### Placement Rules

**domains/[domain]/** (Cross-cutting domains only):
- For domains that multiple features depend on (auth, pricing, billing)
- Importable by features (ESLint exception configured)
- Same structure as features: `pages/`, `ui/`, `data/`, `model/`, `index.ts`
- `index.ts`: **Recommended** - export public API for features to import (e.g., `export { useUser } from './data/queries'`)

**features/[domain]/** (Isolated features):
- `pages/`: Route entry points (page components that compose UI)
  - Examples: `SettingsPage.tsx`, `AccountsPage.tsx`, `DepartmentsPage.tsx`
  - These are lazy-loaded from `app/Router.tsx`
  - Contain routing logic, page layout, and UI composition
- `ui/`: Reusable UI components (NOT pages)
  - Examples: `AccountList.tsx`, `AccountCard.tsx`, `DepartmentForm.tsx`
  - Pure presenters or containers without routing logic
  - Organize by subdirectory when >5 components (e.g., `accounts/`, `departments/`)
- `data/`: Server cache layer (TanStack Query), split by resource for proper granularity
  - Structure: `data/<resource>/api.ts`, `data/<resource>/keys.ts`, `data/<resource>/queries.ts`, `data/<resource>/mutations.ts`
  - Examples: `data/accounts/`, `data/departments/` (API resources, not feature names)
  - Each resource subdirectory contains its own API client, query keys, queries, and mutations
  - **Rationale**: Resource-based organization provides consistent structure regardless of project size, improves file discoverability, and prevents individual files from becoming too large
- `model/`: Feature-local types, schemas, selectors
  - `types.ts`: Domain types specific to this feature
  - `schemas.ts`: Zod schemas for validation
- `index.ts`: ❌ **Prohibited** - features don't cross-import, so barrel exports serve no purpose and create boundary confusion. If sharing is needed: promote to `domains/` (for cross-cutting concerns) or move to `shared/` (for pure UI/types/utils)

**shared/**:
- `ui/`: Shared UI components (Button, Card, Spinner, ErrorView)
  - `index.ts`: Barrel exports for convenience (`export { Button } from './Button'`, etc.)
  - Import as: `import { Spinner, ErrorView } from '@/shared/ui'`
- `lib/`: Shared utilities (http, query-client, errors, hooks)
  - `http.ts`, `query-client.ts`, `errors.ts`
  - `hooks/`: Shared custom hooks (`use-toggle.ts`, `use-debounce.ts`)
- `types/`: **Truly shared types ONLY**
  - `common.ts`: `Pagination`, `ApiResponse<T>`, `SortOrder`, utility types
  - `commands.ts`: Command pattern types (`Command`, used by features/domains to avoid app layer dependency)
  - ✅ Allowed: Types used across multiple features/domains
  - ❌ Prohibited: Feature-specific types, API contract types (keep near data/api)
  - **Prevent "type graveyard"**: If a type is used by only one feature, keep it in that feature's `model/types.ts`

**Page vs UI Component Decision**:
- **pages/**: Route entry points with routing context (path params, query params, navigation)
- **ui/**: Reusable components without routing concerns
- **Rule**: If it needs `useParams()`, `useNavigate()`, or is directly referenced in `app/Router.tsx`, it goes in `pages/`

**Cross-feature Import Rules** (Enforced via ESLint):
- ✅ Allowed: `features → domains` (e.g., `import { useUser } from '@/domains/auth'`)
- ✅ Allowed: `features → shared`, `domains → shared` (e.g., `import type { Command } from '@/shared/types/commands'`)
- ❌ Prohibited: `features → features` (strict isolation)
- ❌ Prohibited: `features → app`, `domains → app` (unidirectional codebase)
- ❌ Prohibited: `shared → features`, `shared → domains` (unidirectional)

**Routing & Navigation (迷子防止)**:
- URL definitions live in **one place only**: `app/Router.tsx`
- Route entry points (screens) live in `features/*/pages/` or `domains/*/pages/`
- Reusable UI components live in `features/*/ui/` or `domains/*/ui/`
- When adding a new route: define it in `Router.tsx`, create the page component in the appropriate `pages/` directory

### ESLint Configuration (Import Enforcement)

Use `eslint-plugin-import` to enforce architectural boundaries:

```js
// .eslintrc.js (EXAMPLE)
module.exports = {
  rules: {
    'import/no-restricted-paths': [
      'error',
      {
        zones: [
          // Prevent features from importing other features (strict isolation)
          {
            target: './src/features/settings',
            from: './src/features',
            except: ['./settings'],
          },
          {
            target: './src/features/dashboard',
            from: './src/features',
            except: ['./dashboard'],
          },
          // Add zones when features increase to maintain isolation

          // Allow features → domains (cross-cutting domains are exempt)
          // No restriction needed here - features CAN import from domains

          // Enforce unidirectional codebase (prevent shared → features/app)
          {
            target: [
              './src/shared/ui',
              './src/shared/lib',
              './src/shared/types',
            ],
            from: ['./src/features', './src/domains', './src/app'],
          },

          // Prevent features from importing from app layer
          {
            target: './src/features',
            from: './src/app',
          },

          // Prevent domains from importing from app layer
          {
            target: './src/domains',
            from: './src/app',
          },
        ],
      },
    ],
  },
}
```

**Note**: This configuration ensures features remain isolated while allowing controlled dependencies on cross-cutting domains.

### Prohibiting features/index.ts (Barrel Exports)

To enforce the prohibition of `features/*/index.ts` files, use one of these approaches:

**Option A: Ban file existence** (using `eslint-plugin-disallow-filenames`):
```js
// .eslintrc.js
module.exports = {
  overrides: [
    {
      files: ['src/features/*/index.ts', 'src/features/*/index.tsx'],
      rules: {
        'disallow-filenames/disallow-filenames': ['error', { message: 'features/index.ts is prohibited. If sharing is needed, promote to domains/ or move to shared/' }]
      }
    }
  ]
}
```

**Option B: Ban barrel imports** (using `eslint-plugin-import`):
```js
// .eslintrc.js
module.exports = {
  rules: {
    'import/no-restricted-paths': [
      'error',
      {
        zones: [
          // Existing zones...

          // Prevent barrel imports from features (force explicit imports)
          {
            target: './src',
            from: './src/features/*/index.ts',
            message: 'Barrel imports from features are prohibited. Import from specific files instead (e.g., @/features/settings/pages/SettingsPage)'
          }
        ]
      }
    ]
  }
}
```

**Rationale**: Features don't cross-import by design, so barrel exports create false boundaries and enable accidental coupling. Explicit imports improve clarity and prevent boundary violations.

---

## Principles

### 1. Pure View (Subscribe & Render)

**Container components** subscribe to state (useQuery/useMutation) and wire event handlers. **Presenter components** receive props and render (no subscriptions, no side effects).

**Prohibited**: `useEffect` for "watch state → trigger side effect" (e.g., `useEffect(() => { if (isError) toast() }, [isError])`).

**Allowed useEffect**: DOM manipulation (charts, third-party widgets), focus control, Observers (Intersection/Resize), window listeners (scroll/resize).

```typescript
// ❌ BAD: State monitoring triggers side effect
useEffect(() => { if (isError) toast.error('Failed') }, [isError])

// ✅ GOOD: Subscribe and render, side effects in mutation/query hooks
function Page() {
  const { data, isLoading, isError } = useAccounts()
  if (isLoading) return <Spinner />
  if (isError) return <ErrorView />
  return <AccountList accounts={data} />
}
```

### 2. Event-driven (Change Through Events)

User actions = events. **"Events" = feature mutations/handlers** (entry points for user operations). Side effects in mutation callbacks (`onSuccess`/`onError`).

**Use `app/events`** only for cross-feature orchestration (multiple features, analytics/logging, complex error handling, router decoupling). Don't move all logic to `app/events`—default to feature hooks.

### 3. Effects at Edges (Side Effects at Boundaries)

Side effects aggregated at: API layer (ApiError normalization), QueryClient (opt-in error toast), mutation callbacks (success toast, navigation, invalidation).

### 4. Single Source of Truth (TanStack Query)

Server state via TanStack Query (required). No `useState + useEffect` for fetching. Use queryKey factories. Reactive updates via `invalidateQueries`.

### 5. Unidirectional Codebase (shared → domains → features → app)

`shared/` has no dependencies on other layers. `domains/` depend only on `shared/`. `features/` depend on `shared/` and `domains/` (but not other features). `app/` composes everything.

```typescript
// ✅ GOOD
import { Card } from '@/shared/ui'  // Barrel import from shared/ui
import { useUser } from '@/domains/auth'  // Cross-cutting domain OK (via index.ts)
import type { Command } from '@/shared/types/commands'  // Shared types OK

// ❌ BAD
import { useDashboard } from '@/features/dashboard'  // From another feature (PROHIBITED)
import type { Command } from '@/app/events/types'  // From app layer (PROHIBITED)
```

---

## Toast Responsibility (Unified Strategy)

**Approach**: Practical balance between Re-frame purity and real-world ergonomics.

**Error Toast (Mutations Only)**:
- Mutations: Opt-in via `meta: { toastError: true }` (MutationCache handles)
- Queries: NO toast (background refetch would spam). Use UI error display (`isError` → `<ErrorView />`).

**Success Toast**:
- Call `toast.success()` directly in mutation `onSuccess` (NOT in page components).

**Navigation**:
- Command pattern: mutations return commands, UI interprets (avoids router coupling in features).

**Rationale**: Mutations are explicit user actions (toast appropriate). Queries run in background (toast inappropriate). Success feedback is mutation-specific, so `onSuccess` is the right place.

```typescript
// shared/lib/query-client.ts (TypeScript meta augmentation)
import { QueryClient, MutationCache } from '@tanstack/react-query'
import { toast } from 'sonner'
import { ApiError } from '@/shared/lib/errors'

declare module '@tanstack/react-query' {
  interface Register {
    defaultError: ApiError
    queryMeta: {
      toastError?: boolean  // Queries: avoid (background refetch spam)
    }
    mutationMeta: {
      toastError?: boolean  // Mutations: opt-in for error toast
    }
  }
}

export const queryClient = new QueryClient({
  mutationCache: new MutationCache({
    onError: (error, _vars, _ctx, mutation) => {
      if (mutation.meta?.toastError) {
        if (error instanceof ApiError) {
          // Don't toast auth errors (handled by UI) or validation errors (handled by forms)
          if (error.status === 401 || error.status === 403 || error.status === 422) return
          toast.error(error.message)
        } else {
          // Non-ApiError: log and show generic message
          console.error('Unexpected mutation error:', error)
          toast.error('An unexpected error occurred')
        }
      }
    },
  }),
  defaultOptions: {
    queries: { staleTime: 60000, retry: 1 },
  },
})

// features/settings/data/departments/mutations.ts (EXAMPLE - useAddDepartment)
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { toast } from 'sonner'
import { departmentsApi } from '@/features/settings/data/departments/api'
import { departmentsKeys } from '@/features/settings/data/departments/keys'
import type { Department } from '@/features/settings/model/types'
import type { Command } from '@/shared/types/commands'

export function useAddDepartment(options?: { onCommand?: (cmd: Command) => void }) {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: departmentsApi.addDepartment,
    meta: { toastError: true },  // Error toast opt-in
    onSuccess: (data: Department) => {
      queryClient.invalidateQueries({ queryKey: departmentsKeys.list() })
      toast.success('Department added')  // Success toast here
      options?.onCommand?.({ type: 'NAVIGATE', to: `/settings/departments/${data.id}` })
    },
  })
}
```

---

## API Layer & HTTP Wrapper

### ApiError

```typescript
// shared/lib/errors.ts
export class ApiError extends Error {
  constructor(
    message: string,
    public status: number,
    public code?: string,
    public details?: unknown
  ) {
    super(message)
    this.name = 'ApiError'
  }
}
```

### HTTP Wrapper (Type-safe + FormData support)

**httpJson Contract**: Endpoints using `httpJson` MUST return `Content-Type: application/json` (except 204 No Content). Contract violations throw `ApiError` with `INVALID_CONTENT_TYPE` for early detection. For non-JSON responses, use `httpText()` or raw `fetch()`.

**Usage**: `httpJson` is for JSON request/response only. For FormData/Blob/URLSearchParams, use `fetch()` directly (FormData sets its own Content-Type with boundary).

```typescript
// shared/lib/http.ts
import { ApiError } from '@/shared/lib/errors'

export async function httpJson<T>(
  url: string,
  options?: RequestInit & { signal?: AbortSignal }
): Promise<T> {
  const headers = new Headers(options?.headers)

  // Only set Content-Type for JSON if body exists and not FormData
  if (options?.body && !(options.body instanceof FormData)) {
    if (!headers.has('Content-Type')) {
      headers.set('Content-Type', 'application/json')
    }
  }

  const res = await fetch(url, { ...options, headers })

  if (!res.ok) {
    let errorData: any = {}
    const contentType = res.headers.get('content-type')
    if (contentType?.includes('application/json')) {
      errorData = await res.json().catch(() => ({}))
    }
    throw new ApiError(
      errorData.message || `HTTP ${res.status}`,
      res.status,
      errorData.code,
      errorData.details
    )
  }

  if (res.status === 204) return undefined as T

  // Validate Content-Type for successful responses (contract enforcement)
  const contentType = res.headers.get('content-type')
  if (!contentType?.includes('application/json')) {
    throw new ApiError(
      'Expected JSON response',
      res.status,
      'INVALID_CONTENT_TYPE',
      { contentType }
    )
  }

  return res.json()
}

export async function httpText(url: string, options?: RequestInit): Promise<string> {
  const res = await fetch(url, options)
  if (!res.ok) {
    const bodyText = await res.text().catch(() => '')
    throw new ApiError(
      `HTTP ${res.status}`,
      res.status,
      undefined,
      { bodyText: bodyText.slice(0, 500) }  // Limit to 500 chars to avoid large logs
    )
  }
  return res.text()
}
```

### API Client Example

```typescript
// features/settings/data/accounts/api.ts (EXAMPLE)
import { httpJson } from '@/shared/lib/http'
import type { Account } from '@/features/settings/model/types'

export const accountsApi = {
  getAccounts: (signal?: AbortSignal) =>
    httpJson<Account[]>('/api/settings/accounts', { signal }),

  deleteAccount: (id: string) =>
    httpJson<void>(`/api/settings/accounts/${id}`, { method: 'DELETE' }),
}

// features/settings/data/departments/api.ts (EXAMPLE)
import { httpJson } from '@/shared/lib/http'
import type { Department, CreateDepartmentInput } from '@/features/settings/model/types'

export const departmentsApi = {
  addDepartment: (data: CreateDepartmentInput) =>
    httpJson<Department>('/api/settings/departments', {
      method: 'POST',
      body: JSON.stringify(data),
    }),
}
```

---

## Server State: TanStack Query

### QueryKey Factory

```typescript
// features/settings/data/accounts/keys.ts (EXAMPLE)
export const accountsKeys = {
  all: ['settings', 'accounts'] as const,
  list: () => [...accountsKeys.all, 'list'] as const,
  detail: (id: string) => [...accountsKeys.all, id] as const,
}

// features/settings/data/departments/keys.ts (EXAMPLE)
export const departmentsKeys = {
  all: ['settings', 'departments'] as const,
  list: () => [...departmentsKeys.all, 'list'] as const,
}
```

### Query Hook (No Toast)

```typescript
// features/settings/data/accounts/queries.ts (EXAMPLE - useAccounts)
import { useQuery } from '@tanstack/react-query'
import { accountsApi } from '@/features/settings/data/accounts/api'
import { accountsKeys } from '@/features/settings/data/accounts/keys'

export function useAccounts() {
  return useQuery({
    queryKey: accountsKeys.list(),
    queryFn: ({ signal }) => accountsApi.getAccounts(signal),
    // ⚠️ No meta.toastError for queries (background refetch would spam)
  })
}
```

### Derived Data (select)

```typescript
const { data: activeAccounts } = useQuery({
  queryKey: accountsKeys.list(),
  queryFn: ({ signal }) => accountsApi.getAccounts(signal),
  select: (accounts) => accounts.filter(a => a.status === 'active'),
})
```

### Cache Update Strategy

**Default**: Use `invalidateQueries` (refetch from server).

**Optimistic UX**: Use `setQueryData` for immediate feedback (e.g., list operations where stale data is acceptable).

```typescript
// features/settings/data/accounts/mutations.ts (EXAMPLE - useDeleteAccount)
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { accountsApi } from '@/features/settings/data/accounts/api'
import { accountsKeys } from '@/features/settings/data/accounts/keys'
import type { Account } from '@/features/settings/model/types'

export function useDeleteAccount() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: accountsApi.deleteAccount,
    onMutate: async (id: string) => {
      await queryClient.cancelQueries({ queryKey: accountsKeys.list() })
      const previous = queryClient.getQueryData<Account[]>(accountsKeys.list())
      queryClient.setQueryData<Account[]>(accountsKeys.list(), (old = []) =>
        old.filter(a => a.id !== id)
      )
      return { previous }
    },
    onError: (_err, _vars, context) => {
      if (context?.previous) {
        queryClient.setQueryData(accountsKeys.list(), context.previous)
      }
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: accountsKeys.list() })
    },
  })
}
```

---

## Event/Effect Layer (app/events)

### When to Use app/events

**Use feature hooks (queries/mutations)** when:
- Side effect is local to feature
- Simple API call + invalidation
- No cross-feature coordination

**Use app/events** when:
- Orchestration spans multiple features
- Complex flow (API + analytics + logging + multiple invalidations)
- Need centralized error handling
- Want testable, isolated effect implementations
- Need to decouple from router (testability)

### app/events and app/effects Scope Rules

**⚠️ PRINCIPLE: Single-feature logic MUST stay in feature hooks by default.**

**Allowed Responsibilities (Cross-feature Only)**:
- Workflows spanning multiple features (e.g., "create account → assign to department → send notification")
- Analytics/logging that aggregates data from multiple features
- Complex error handling requiring coordination across features
- Global side effects (e.g., WebSocket reconnection affecting multiple features)
- Navigation orchestration when multiple features are involved

**Prohibited Responsibilities**:
- Single-feature API calls (use feature `mutations.ts` instead)
- Simple CRUD operations within one feature (use feature hooks)
- Feature-specific state updates (use feature hooks or jotai atoms)
- UI-only logic that doesn't coordinate across features
- Server state invalidation for single features (use `queryClient.invalidateQueries` in mutation callbacks)

**Rationale**: `app/events` and `app/effects` are for cross-cutting concerns only. Keeping single-feature logic in feature hooks maintains feature isolation, improves testability, and prevents the app layer from becoming a dumping ground for arbitrary logic.

### Command Pattern (Navigation)

```typescript
// shared/types/commands.ts (EXAMPLE)
export type Command =
  | { type: 'NAVIGATE'; to: string }
  | { type: 'INVALIDATE_QUERY'; queryKey: unknown[] }

// features/settings/pages/CreateDepartmentPage.tsx (EXAMPLE)
import { useNavigate } from 'react-router-dom'
import { useAddDepartment } from '@/features/settings/data/departments/mutations'
import { CreateDepartmentForm } from '@/features/settings/ui/CreateDepartmentForm'
import type { Command } from '@/shared/types/commands'

export function CreateDepartmentPage() {
  const navigate = useNavigate()
  const { mutate } = useAddDepartment({
    onCommand: (cmd) => {
      if (cmd.type === 'NAVIGATE') navigate(cmd.to)
    },
  })
  return <CreateDepartmentForm onSubmit={mutate} />
}
```

### Effect Handlers

```typescript
// app/effects/analytics.effects.ts (EXAMPLE)
export const analyticsEffects = {
  trackEvent: (name: string, props?: Record<string, any>) => {
    console.log('Analytics:', name, props)
  },
}
```

### Cross-feature Subscriptions

```typescript
// app/store/cross-feature-hooks.ts (EXAMPLE - for derived data spanning features)
// Alternative naming: app/subscriptions/ for Re-frame alignment
export function useAccountsWithStats() {
  const { data: accounts } = useAccounts()
  const { data: stats } = useDashboardStats()
  return useMemo(() => {
    if (!accounts || !stats) return []
    return accounts.map(a => ({ ...a, activityCount: stats.activityByAccount[a.id] ?? 0 }))
  }, [accounts, stats])
}
```

---

## State Management

### Server State: TanStack Query (Required)

No `useState + useEffect` for fetching. Use queryKey factory. Reactive updates via `invalidateQueries`.

### Local UI State: useState / react-hook-form

Component-local state (form input, modal, tab selection).

### Application State: jotai (Minimal)

Small cross-page UI state only (sidebar open, theme). Don't store server data in atoms.

```typescript
// app/store/ui.atoms.ts (EXAMPLE)
import { atom } from 'jotai'

export const sidebarOpenAtom = atom(true)
export const themeAtom = atom<'light' | 'dark'>('light')
export const isDarkModeAtom = atom((get) => get(themeAtom) === 'dark')
```

---

## Component Patterns

### Container / Presenter

```typescript
// features/settings/ui/AccountList.tsx (EXAMPLE)
import { useAccounts } from '@/features/settings/data/accounts/queries'
import { useDeleteAccount } from '@/features/settings/data/accounts/mutations'
import { Spinner, ErrorView } from '@/shared/ui'
import type { Account } from '@/features/settings/model/types'

// Container: Subscribe + wire events
export function AccountListContainer() {
  const { data, isLoading, isError } = useAccounts()
  const { mutate: deleteAccount } = useDeleteAccount()

  if (isLoading) return <Spinner />
  if (isError) return <ErrorView />

  return <AccountListPresenter accounts={data ?? []} onDelete={deleteAccount} />
}

// Presenter: Pure render (props → JSX)
interface AccountListPresenterProps {
  accounts: Account[]
  onDelete: (id: string) => void
}

function AccountListPresenter({ accounts, onDelete }: AccountListPresenterProps) {
  return (
    <ul>
      {accounts.map(account => (
        <li key={account.id}>
          <span>{account.name}</span>
          <button onClick={() => onDelete(account.id)}>Delete</button>
        </li>
      ))}
    </ul>
  )
}
```

---

## Forms

Use `react-hook-form` + `zod` for forms.

```typescript
// EXAMPLE
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const schema = z.object({
  name: z.string().min(1).max(200),
  email: z.string().email(),
})

type FormData = z.infer<typeof schema>

export function CreateForm({ onSubmit }: { onSubmit: (data: FormData) => void }) {
  const { register, handleSubmit, formState: { errors } } = useForm<FormData>({
    resolver: zodResolver(schema),
  })

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('name')} />
      {errors.name && <span>{errors.name.message}</span>}
      <button type="submit">Create</button>
    </form>
  )
}
```

---

## Error Handling

### Request Errors

Normalize to `ApiError` in API layer. Opt-in mutation error toast via meta. Use `isError` for UI display branching (not toast).

**Status-based**:
- 401/403: Auth UI redirect (no toast)
- 422: Form errors (no toast)
- 5xx: Opt-in toast for critical mutations

### Render Errors

ErrorBoundary for render-time exceptions only.

---

## Performance

**Profile first, optimize second.** Write working code → profile (React DevTools Profiler, Chrome DevTools) → optimize where needed.

### JavaScript Performance

**Cache repeated function calls with module-level Map.**

```typescript
// Module-level cache (reusable across renders)
const slugifyCache = new Map<string, string>()

function cachedSlugify(text: string): string {
  if (slugifyCache.has(text)) return slugifyCache.get(text)!
  const result = slugify(text)
  slugifyCache.set(text, result)
  return result
}

function ProjectList({ projects }: { projects: Project[] }) {
  return (
    <div>
      {projects.map(project => (
        <ProjectCard key={project.id} slug={cachedSlugify(project.name)} />
      ))}
    </div>
  )
}
```

**⚠️ Cache Size Management**: Module-level Maps grow unbounded. For user-generated inputs, add size limits (e.g., `if (cache.size > 1000) cache.clear()`) or use LRU cache libraries to prevent memory leaks.

**Use Set/Map for O(1) lookups instead of Array.includes (O(n)).**

```typescript
// ❌ BAD: O(n) per check
const allowedIds = ['a', 'b', 'c', ...]
items.filter(item => allowedIds.includes(item.id))

// ✅ GOOD: O(1) per check
const allowedIds = new Set(['a', 'b', 'c', ...])
items.filter(item => allowedIds.has(item.id))
```

### React Performance

**Don't overuse `useMemo/useCallback` unnecessarily.**

```typescript
// ❌ BAD: Unnecessary useMemo
const doubled = useMemo(() => count * 2, [count])

// ✅ GOOD: If computation is cheap, just compute
const doubled = count * 2

// ✅ GOOD: useMemo for expensive computation
const sorted = useMemo(
  () => items.slice().sort((a, b) => complexComparison(a, b)),
  [items]
)
```

**Use lazy state initialization for expensive initial values.**

```typescript
// ❌ BAD: buildSearchIndex() runs on EVERY render
const [searchIndex, setSearchIndex] = useState(buildSearchIndex(items))

// ✅ GOOD: Runs ONLY on initial render
const [searchIndex, setSearchIndex] = useState(() => buildSearchIndex(items))

// ✅ GOOD: Lazy localStorage read
const [settings, setSettings] = useState(() => {
  const stored = localStorage.getItem('settings')
  return stored ? JSON.parse(stored) : {}
})
```

**Use `React.memo` for pure presenter components.**

```typescript
export const AccountCard = React.memo<{ account: Account }>(({ account }) => {
  return (
    <div className="account-card">
      <h3>{account.name}</h3>
      <p>{account.email}</p>
    </div>
  )
})

// For expensive work, extract to memoized component to enable early returns
const UserAvatar = memo(function UserAvatar({ user }: { user: User }) {
  const id = useMemo(() => computeAvatarId(user), [user])
  return <Avatar id={id} />
})

function Profile({ user, loading }: Props) {
  if (loading) return <Skeleton />  // Skip avatar computation when loading
  return <div><UserAvatar user={user} /></div>
}
```

### Rendering Optimization

**Use explicit conditional rendering to avoid rendering falsy values.**

```typescript
// ❌ BAD: Renders "0" when count is 0
{count && <span className="badge">{count}</span>}

// ✅ GOOD: Renders nothing when count is 0
{count > 0 ? <span className="badge">{count}</span> : null}
```

**Hoist static JSX outside components to avoid re-creation.**

```typescript
// ❌ BAD: Recreates element every render
function Container() {
  return <div>{loading && <div className="animate-pulse h-20 bg-gray-200" />}</div>
}

// ✅ GOOD: Reuses same element reference
const loadingSkeleton = <div className="animate-pulse h-20 bg-gray-200" />

function Container() {
  return <div>{loading && loadingSkeleton}</div>
}
```

### Bundle Optimization

**Code splitting for routes:**

```typescript
import { lazy, Suspense } from 'react'

const Dashboard = lazy(() => import('@/features/dashboard/pages/DashboardPage'))
const Settings = lazy(() => import('@/features/settings/pages/SettingsPage'))

function App() {
  return (
    <Suspense fallback={<PageLoader />}>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
      </Routes>
    </Suspense>
  )
}
```

**Dynamic imports for heavy components (charts, editors, large libraries):**

```typescript
import { lazy, Suspense } from 'react'

// ❌ BAD: Monaco bundles with main chunk (~300KB)
import { MonacoEditor } from '@/components/MonacoEditor'

// ✅ GOOD: Monaco loads on demand
const MonacoEditor = lazy(() => import('@/components/MonacoEditor').then(m => ({ default: m.MonacoEditor })))

function CodePanel({ code }: { code: string }) {
  return (
    <Suspense fallback={<EditorSkeleton />}>
      <MonacoEditor value={code} />
    </Suspense>
  )
}
```

**Note**: [React Compiler](https://react.dev/learn/react-compiler) auto-optimizes memo/useMemo/hoisting when enabled.

---

## Security

### Authentication (HttpOnly Cookies + CSRF)

```typescript
// shared/lib/auth.tsx (EXAMPLE)
const getUser = async () => {
  const res = await fetch('/api/auth/me')  // Token via HttpOnly cookie
  if (!res.ok) return null
  return res.json()
}

export const userQueryOptions = queryOptions({
  queryKey: ['auth', 'user'],
  queryFn: getUser,
  staleTime: Infinity,
})
```

**⚠️ CSRF Protection**: When using HttpOnly cookies, implement CSRF protection (SameSite=Strict/Lax + CSRF token for state-changing requests). Common patterns: double submit cookie (server sets token in cookie + custom header), or synchronizer token (server embeds token in page, client sends in header). Modern frameworks (Next.js API routes, etc.) often handle this automatically.

**⚠️ Cross-Origin Cookies**: For cross-origin requests with credentials, set `credentials: 'include'` in fetch options and configure CORS properly on the server (Access-Control-Allow-Credentials: true).

### Authorization (RBAC/PBAC)

**⚠️ Server-Side Validation Required**: UI authorization checks (like `<Authorization>` component) are for UX only. The server MUST validate all permissions for every request—never trust client-side checks.

```typescript
// shared/lib/authorization.tsx (EXAMPLE)
export function Authorization({ allowedRoles, policyCheck, children }: AuthorizationProps) {
  const { data: user } = useUser()
  if (!user) return null
  if (allowedRoles && !allowedRoles.includes(user.role)) return null
  if (policyCheck !== undefined && !policyCheck) return null
  return <>{children}</>
}

// Usage
<Authorization allowedRoles={['admin']}><AdminPanel /></Authorization>
<Authorization policyCheck={comment.authorId === user?.id}><DeleteButton /></Authorization>
```

### XSS Protection

```typescript
// EXAMPLE
import DOMPurify from 'dompurify'

export function MDPreview({ content }: { content: string }) {
  return <div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(content) }} />
}
```

---

## Testing

**Priority**: Integration tests (primary) → E2E (critical flows) → Unit tests (shared utils only).

**Tools**: Vitest, Testing Library, Playwright, MSW.

```typescript
// features/settings/ui/__tests__/AccountList.test.tsx (EXAMPLE)
import { render, screen, waitFor } from '@testing-library/react'
import userEvent from '@testing-library/user-event'
import { AccountListContainer } from '@/features/settings/ui/AccountList'

test('deletes account on click', async () => {
  render(<AccountListContainer />)
  await waitFor(() => expect(screen.getByText('John Doe')).toBeInTheDocument())
  await userEvent.click(screen.getByRole('button', { name: /delete/i }))
  await waitFor(() => expect(screen.queryByText('John Doe')).not.toBeInTheDocument())
})
```

---

## Project Standards

- **ESLint + Prettier + TypeScript + Husky**: Code quality, formatting, type safety, pre-commit hooks
- **Absolute Imports**: `@/*` in tsconfig (`import { Card } from '@/shared/ui'`)
- **File Naming**:
  - **Component files (.tsx)**: PascalCase (`AccountList.tsx`, `Button.tsx`, `LoginPage.tsx`)
  - **Non-component files (.ts)**: kebab-case (`api.ts`, `keys.ts`, `queries.ts`, `mutations.ts`, `use-toggle.ts`)
  - Enforce with ESLint `check-file` plugin
- **File Size**: Target 100-200 lines, max 400 lines per `.ts`/`.tsx` file for readability
  - **If exceeding 400 lines**:
    - Components: Split into Container/Presenter or extract sub-components
    - Data layer: Already organized by resource in `data/<resource>/` (api.ts, keys.ts, queries.ts, mutations.ts)
    - Utils: Split by responsibility (e.g., `date.utils.ts`, `string.utils.ts`)

---

**Remember**: Write working code first. Optimize after measuring. Add complexity only when needed. Use bulletproof structure for organization, re-frame principles for behavior.
