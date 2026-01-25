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
- Unidirectional codebase (shared → features → app)
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
│   ├── Router.tsx              # Router configuration
│   ├── routes/                 # Route definitions (lazy imports + layouts)
│   ├── events/                 # Cross-feature orchestration only
│   │   ├── types.ts
│   │   └── handlers/
│   ├── effects/                # Cross-feature effect implementations
│   │   ├── analytics.effects.ts
│   │   └── logging.effects.ts
│   └── store/                  # Global application state (jotai atoms)
│       └── ui.atoms.ts
│
├── shared/                     # Shared across all features
│   ├── ui/                     # Shared UI components
│   │   ├── button.tsx          # Primitives
│   │   ├── card.tsx
│   │   ├── spinner.tsx         # Feedback components
│   │   └── error-view.tsx
│   ├── lib/                    # Shared utilities & configurations
│   │   ├── query-client.ts     # TanStack Query setup
│   │   ├── http.ts             # httpJson, httpText
│   │   ├── errors.ts           # ApiError class
│   │   └── hooks/              # Shared hooks
│   │       ├── use-toggle.ts
│   │       └── use-debounce.ts
│   └── types/                  # Shared types
│       ├── api.ts
│       └── common.ts
│
└── features/                   # Feature modules (domain-driven, isolated)
    ├── auth/                   # ⚠️ Cross-feature domain (can be imported by other features)
    │   ├── ui/                 # Feature-private UI
    │   │   ├── login-page/
    │   │   │   ├── LoginPage.tsx
    │   │   │   └── LoginForm.tsx
    │   │   └── ProtectedRoute.tsx
    │   ├── data/               # Server cache layer (TanStack Query)
    │   │   ├── api.ts          # Fetchers (no React)
    │   │   ├── keys.ts         # QueryKey factory
    │   │   ├── queries.ts      # queryOptions / useQuery hooks
    │   │   └── mutations.ts    # useMutation hooks
    │   ├── model/              # Feature-local domain logic
    │   │   ├── types.ts        # Types
    │   │   └── schemas.ts      # Zod schemas (if needed)
    │   └── index.ts            # Public exports only
    │
    ├── settings/               # Domain feature (isolated)
    │   ├── ui/
    │   │   ├── accounts-page/
    │   │   │   ├── AccountsPage.tsx
    │   │   │   ├── AccountList.tsx
    │   │   │   └── AccountCard.tsx
    │   │   └── departments-page/
    │   │       ├── DepartmentsPage.tsx
    │   │       └── DepartmentForm.tsx
    │   ├── data/
    │   │   ├── api.ts
    │   │   ├── keys.ts
    │   │   ├── queries.ts
    │   │   └── mutations.ts
    │   ├── model/
    │   │   └── types.ts
    │   └── index.ts
    │
    └── dashboard/
        └── ...
```

### App Layer (Simple & Clean)

```typescript
// app/App.tsx (EXAMPLE)
import { AppProvider } from './Provider'
import { AppRouter } from './Router'

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

const SettingsPage = lazy(() => import('./routes/settings'))
const DashboardPage = lazy(() => import('./routes/dashboard'))

export function AppRouter() {
  return (
    <BrowserRouter>
      <Suspense fallback={<div>Loading...</div>}>
        <Routes>
          <Route path="/" element={<DashboardPage />} />
          <Route path="/settings" element={<SettingsPage />} />
        </Routes>
      </Suspense>
    </BrowserRouter>
  )
}
```

### Placement Rules

**features/[domain]/**:
- `ui/`: Feature-private components (Container/Presenter). Organize by page (e.g., `accounts-page/`, `departments-page/`) when >5 components.
- `data/`: Server cache layer (TanStack Query)
  - `api.ts`: Fetchers (no React hooks, just `httpJson`/`httpText` calls)
  - `keys.ts`: QueryKey factory
  - `queries.ts`: `queryOptions` or `useQuery` hooks
  - `mutations.ts`: `useMutation` hooks
- `model/`: Feature-local types, schemas, selectors
- `index.ts`: Public API (minimal exports for cross-feature use)

**shared/**:
- `ui/`: Shared components (Button, Card, Spinner, etc)
- `lib/`: Shared utilities (http, query-client, hooks)
- `types/`: Shared types

**Cross-feature exceptions**: `features/auth/` can be imported by other features. For additional cross-cutting domains (pricing, plan), mark with ⚠️ comment or create `src/domains/` directory.

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

### 5. Unidirectional Codebase (shared → features → app)

`lib/`, `components/`, `types/` are shared (no feature/app dependencies). `features/` use shared code, cannot import other features (except cross-cutting domains like auth). `app/` composes features.

```typescript
// ✅ GOOD
import { Card } from '@/shared/ui/card'
import { useUser } from '@/features/auth'  // Cross-cutting domain OK

// ❌ BAD
import { useDashboard } from '@/features/dashboard'  // From another feature (PROHIBITED)
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
import { ApiError } from './errors'

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

// features/settings/data/mutations.ts (EXAMPLE - useAddDepartment)
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { toast } from 'sonner'
import { settingsApi } from './api'
import { settingsKeys } from './keys'
import type { Department } from '../model/types'
import type { Command } from '@/app/events/types'

export function useAddDepartment(options?: { onCommand?: (cmd: Command) => void }) {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: settingsApi.addDepartment,
    meta: { toastError: true },  // Error toast opt-in
    onSuccess: (data: Department) => {
      queryClient.invalidateQueries({ queryKey: settingsKeys.departments() })
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
import { ApiError } from './errors'

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
// features/settings/data/api.ts (EXAMPLE)
import { httpJson } from '@/shared/lib/http'
import type { Account, Department, CreateDepartmentInput } from '../model/types'

export const settingsApi = {
  getAccounts: (signal?: AbortSignal) =>
    httpJson<Account[]>('/api/settings/accounts', { signal }),

  addDepartment: (data: CreateDepartmentInput) =>
    httpJson<Department>('/api/settings/departments', {
      method: 'POST',
      body: JSON.stringify(data),
    }),

  deleteAccount: (id: string) =>
    httpJson<void>(`/api/settings/accounts/${id}`, { method: 'DELETE' }),
}
```

---

## Server State: TanStack Query

### QueryKey Factory

```typescript
// features/settings/data/keys.ts (EXAMPLE)
export const settingsKeys = {
  all: ['settings'] as const,
  accounts: () => [...settingsKeys.all, 'accounts'] as const,
  accountDetail: (id: string) => [...settingsKeys.accounts(), id] as const,
  departments: () => [...settingsKeys.all, 'departments'] as const,
}
```

### Query Hook (No Toast)

```typescript
// features/settings/data/queries.ts (EXAMPLE - useAccounts)
import { useQuery } from '@tanstack/react-query'
import { settingsApi } from './api'
import { settingsKeys } from './keys'

export function useAccounts() {
  return useQuery({
    queryKey: settingsKeys.accounts(),
    queryFn: ({ signal }) => settingsApi.getAccounts(signal),
    // ⚠️ No meta.toastError for queries (background refetch would spam)
  })
}
```

### Derived Data (select)

```typescript
const { data: activeAccounts } = useQuery({
  queryKey: settingsKeys.accounts(),
  queryFn: ({ signal }) => settingsApi.getAccounts(signal),
  select: (accounts) => accounts.filter(a => a.status === 'active'),
})
```

### Cache Update Strategy

**Default**: Use `invalidateQueries` (refetch from server).

**Optimistic UX**: Use `setQueryData` for immediate feedback (e.g., list operations where stale data is acceptable).

```typescript
// features/settings/data/mutations.ts (EXAMPLE - useDeleteAccount)
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { settingsApi } from './api'
import { settingsKeys } from './keys'
import type { Account } from '../model/types'

export function useDeleteAccount() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: settingsApi.deleteAccount,
    onMutate: async (id: string) => {
      await queryClient.cancelQueries({ queryKey: settingsKeys.accounts() })
      const previous = queryClient.getQueryData<Account[]>(settingsKeys.accounts())
      queryClient.setQueryData<Account[]>(settingsKeys.accounts(), (old = []) =>
        old.filter(a => a.id !== id)
      )
      return { previous }
    },
    onError: (_err, _vars, context) => {
      if (context?.previous) {
        queryClient.setQueryData(settingsKeys.accounts(), context.previous)
      }
    },
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: settingsKeys.accounts() })
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

### Command Pattern (Navigation)

```typescript
// app/events/types.ts (EXAMPLE)
export type Command =
  | { type: 'NAVIGATE'; to: string }
  | { type: 'INVALIDATE_QUERY'; queryKey: unknown[] }

// app/routes/settings/create-department.tsx (EXAMPLE)
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
// features/settings/ui/accounts-page/AccountList.tsx (EXAMPLE)
import { useAccounts, useDeleteAccount } from '../../data/queries'
import { Spinner, ErrorView } from '@/shared/ui'

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

const Dashboard = lazy(() => import('./routes/Dashboard'))
const Settings = lazy(() => import('./routes/Settings'))

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
import { MonacoEditor } from './monaco-editor'

// ✅ GOOD: Monaco loads on demand
const MonacoEditor = lazy(() => import('./monaco-editor').then(m => ({ default: m.MonacoEditor })))

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
// features/settings/ui/accounts-page/__tests__/AccountList.test.tsx (EXAMPLE)
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
- **Absolute Imports**: `@/*` in tsconfig (`import { Card } from '@/shared/ui/card'`)
- **File Naming**: kebab-case (`account-list.tsx`), enforce with ESLint `check-file` plugin
- **File Size**: Target 100-200 lines, max 400 lines per `.ts`/`.tsx` file for readability
  - **If exceeding 400 lines**:
    - Components: Split into Container/Presenter or extract sub-components
    - Hooks: Split queries/mutations into separate files
    - API clients: Split by resource (e.g., `accounts.api.ts`, `departments.api.ts`)
    - Utils: Split by responsibility (e.g., `date.utils.ts`, `string.utils.ts`)

---

**Remember**: Write working code first. Optimize after measuring. Add complexity only when needed. Use bulletproof structure for organization, re-frame principles for behavior.
