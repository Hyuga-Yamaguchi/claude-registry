---
name: frontend-patterns-react
description: Reagent-inspired React SPA patterns with bulletproof-react structure - subscription-driven UI, event/effect separation, side-effects at the edges, with TanStack Query as standard.
---

# React Frontend Patterns (Reagent-inspired + Bulletproof Structure)

Design guide applying Reagent/Re-frame philosophy (subscription model, event-driven, side-effect separation) to React SPA (react-router-dom), using [bulletproof-react](https://github.com/alan2207/bulletproof-react) as the structural foundation.

For language-level TypeScript standards, see [coding-standards-typescript.md](coding-standards-typescript.md).

---

## Philosophy: Bulletproof + Re-frame Layer

**bulletproof-react** provides the "structural foundation" (directory structure, boundaries, tool selection), while **Reagent/Re-frame** principles enforce the "behavioral rules" (pure views, event-driven, effect isolation).

**What bulletproof-react provides:**
- Features-based organization (domain-driven boundaries)
- Unidirectional codebase (shared → features → app)
- State classification (Component / Application / Server Cache / Form / URL)
- Cross-feature import prohibition

**What we add (Re-frame layer):**
- Pure View: Subscribe & Render only (no side effects in UI)
- Event-driven: All changes through events (dispatch only)
- Effect handlers: Side effects isolated at boundaries
- Subscriptions: Derived data via selectors

**Result:** bulletproof-react's structure + re-frame's discipline = scalable, maintainable React architecture.

---

## Directory Structure

```
src/
├── app/                    # Application layer (composition + re-frame core)
│   ├── routes/             # Route definitions
│   ├── events/             # Event handlers (re-frame style)
│   ├── effects/            # Effect implementations (analytics, logging)
│   ├── subs/               # Cross-feature subscriptions (derived data)
│   ├── store/              # Application state (zustand/jotai)
│   └── provider.tsx        # Root providers (QueryClient, Router, etc)
│
├── features/               # Feature modules (domain-driven, isolated)
│   ├── auth/
│   │   ├── api/            # API calls
│   │   ├── components/     # Feature components
│   │   ├── hooks/          # queries/, mutations/, keys.ts
│   │   ├── types/          # Feature types
│   │   └── index.ts        # Public API (controlled exports)
│   ├── settings/
│   └── dashboard/
│
├── components/             # Shared UI components
│   ├── ui/                 # Primitives (Button, Card, etc)
│   └── layout/             # Layout (Header, Sidebar, etc)
│
├── lib/                    # Shared utilities
│   ├── query-client.ts     # TanStack Query setup
│   ├── http.ts             # HTTP wrapper
│   ├── errors.ts           # ApiError class
│   └── hooks/              # Shared hooks (use-toggle, use-debounce, etc)
│
└── types/                  # Shared types
    ├── api.ts
    └── common.ts
```

---

## Non-negotiable Principles

### 1. Pure View (Subscribe and Render Only)

**UI subscribes to state and renders. Nothing more.**

- `useEffect` for "watch state → trigger side effect" is **prohibited**
- Side effects (API, toast, navigation, logging) do not belong in UI

```typescript
// ❌ BAD: useEffect watching state to trigger side effects
useEffect(() => {
  if (isError) toast.error('Failed')
}, [isError])

// ✅ GOOD: Subscribe and render only
function Page() {
  const { data, isLoading, isError } = useAccounts()

  if (isLoading) return <Spinner />
  if (isError) return <ErrorView />
  return <AccountList accounts={data} />
}
```

**Allowed `useEffect` use cases:**
- Direct DOM manipulation (chart library, third-party widget)
- Focus control (Modal focus trap)
- IntersectionObserver, ResizeObserver
- Window event listeners (scroll, resize)

### 2. Event-driven (Change Through Events)

**User actions expressed as events (handlers / mutations).**

- Side effects in mutation's `onSuccess/onError` or app/events
- "Event happens → state changes" not "state changes → do something"

```typescript
// ✅ GOOD: Side effects in mutation
const { mutate: addDepartment } = useAddDepartment({
  onSuccess: () => {
    queryClient.invalidateQueries({ queryKey: settingsKeys.departments() })
    // For navigation, dispatch command (see Event/Effect Layer)
  }
})
```

### 3. Effects at the Edges (Side Effects at Boundaries)

**Side effects aggregated at boundaries (API layer, QueryClient, event handlers).**

- API errors normalized in API layer (`ApiError`)
- Toast displayed via opt-in meta (not global for all errors)
- ErrorBoundary is for render-time exceptions only

### 4. Single Source of Truth (TanStack Query as Standard)

**Server state managed with `@tanstack/react-query` (required).**

- `useState + useEffect` for fetching is prohibited
- Use queryKey factory to avoid hardcoded strings
- Reactive updates via `invalidateQueries`

### 5. Unidirectional Codebase (shared → features → app)

**Enforce dependency flow: shared code → features → app layer.**

- `components/`, `lib/`, `types/` are shared (no dependencies on features or app)
- `features/` can use shared code, but **cannot import from other features**
- `app/` can use both shared code and features (composition layer)

```typescript
// ✅ GOOD: Feature imports from shared
import { Card } from '@/components/ui/card'

// ❌ BAD: Feature imports from another feature
import { useDashboardData } from '@/features/dashboard/hooks'  // PROHIBITED

// ✅ GOOD: App layer composes features
import { AccountList } from '@/features/settings'
import { DashboardWidget } from '@/features/dashboard'
```

### 6. Feature Isolation (Controlled Exports via index.ts)

**Features expose only what's needed through index.ts.**

```typescript
// features/settings/index.ts
export { AccountList } from './components/AccountList'
export { useAccounts, useDepartments } from './hooks/queries'
export { useAddDepartment } from './hooks/mutations'
export type { Account, Department } from './types'
```

---

## Server State: TanStack Query (Standard)

### QueryKey Factory Pattern

```typescript
// features/settings/hooks/keys.ts
export const settingsKeys = {
  all: ['settings'] as const,
  accounts: () => [...settingsKeys.all, 'accounts'] as const,
  accountDetail: (id: string) => [...settingsKeys.accounts(), id] as const,
}
```

### API Layer: ApiError and http wrapper

```typescript
// lib/errors.ts
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

// lib/http.ts
import { ApiError } from './errors'

export async function http<T>(
  url: string,
  options?: RequestInit & { signal?: AbortSignal }
): Promise<T> {
  const res = await fetch(url, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options?.headers,
    },
  })

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

  // Handle 204 No Content
  if (res.status === 204) {
    return undefined as T
  }

  const contentType = res.headers.get('content-type')
  if (contentType?.includes('application/json')) {
    return res.json()
  }

  return res.text() as T
}

// features/settings/api/settings.api.ts
import { http } from '@/lib/http'
import type { Account, CreateDepartmentInput } from '../types'

export const settingsApi = {
  getAccounts: (signal?: AbortSignal) =>
    http<Account[]>('/api/settings/accounts', { signal }),
  addDepartment: (data: CreateDepartmentInput) =>
    http<Department>('/api/settings/departments', {
      method: 'POST',
      body: JSON.stringify(data),
    }),
}
```

### QueryClient: Opt-in Toast (Errors Only)

**Use `meta` to opt-in to error toast. Status-based branching.**

```typescript
// lib/query-client.ts
import { QueryClient, QueryCache, MutationCache } from '@tanstack/react-query'
import { toast } from 'sonner'
import { ApiError } from './errors'

export const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error, query) => {
      // Only show toast if explicitly opted-in via meta
      if (query.meta?.toastError) {
        if (error instanceof ApiError) {
          // 401/403: Don't toast, handle via auth UI
          if (error.status === 401 || error.status === 403) return

          // 422: Prefer form-level errors
          if (error.status === 422) return

          toast.error(error.message)
        } else {
          toast.error('An unexpected error occurred')
        }
      }
    },
  }),
  mutationCache: new MutationCache({
    onError: (error, _variables, _context, mutation) => {
      // Only show toast if explicitly opted-in via meta
      if (mutation.meta?.toastError) {
        if (error instanceof ApiError) {
          if (error.status === 401 || error.status === 403) return
          if (error.status === 422) return

          toast.error(error.message)
        } else {
          toast.error('An unexpected error occurred')
        }
      }
    },
  }),
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60,
      retry: 1,
    },
  },
})
```

**Don't call toast in pages. Only display branching.**

```typescript
// ✅ GOOD: Display branching only
function Page() {
  const { data, isLoading, isError } = useAccounts()

  if (isLoading) return <Spinner />
  if (isError) return <ErrorView message="Failed to load accounts" />

  return <AccountList accounts={data} />
}
```

### Feature Query Hooks

```typescript
// features/settings/hooks/queries/use-accounts.ts
import { useQuery } from '@tanstack/react-query'
import { settingsApi } from '../../api/settings.api'
import { settingsKeys } from '../keys'

export function useAccounts() {
  return useQuery({
    queryKey: settingsKeys.accounts(),
    queryFn: ({ signal }) => settingsApi.getAccounts(signal),
    meta: { toastError: true }, // Opt-in to error toast
  })
}

// features/settings/hooks/mutations/use-add-department.ts
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { settingsApi } from '../../api/settings.api'
import { settingsKeys } from '../keys'

export function useAddDepartment() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: settingsApi.addDepartment,
    meta: { toastError: true }, // Opt-in to error toast
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: settingsKeys.departments() })
      toast.success('Department added')
    },
  })
}
```

### Derived Data with `select`

```typescript
// ✅ GOOD: Compute derived data with select
function Page() {
  const { data: activeAccounts } = useQuery({
    queryKey: settingsKeys.accounts(),
    queryFn: ({ signal }) => settingsApi.getAccounts(signal),
    select: (accounts) => accounts.filter(a => a.status === 'active'),
  })

  return <div>{activeAccounts?.length ?? 0}</div>
}
```

---

## Event/Effect Layer (Re-frame Core)

### When to Use

**Use feature hooks (queries/mutations) when:**
- Side effect is local to the feature
- Simple API call + invalidation
- No cross-feature coordination needed

**Use app/events when:**
- Side effect spans multiple features
- Complex orchestration (API + analytics + logging)
- Need centralized error handling
- Want testable, isolated effect implementations

### Command Pattern for Navigation (Recommended)

**Events return commands, UI interprets them.**

```typescript
// app/events/types.ts
export type Command =
  | { type: 'NAVIGATE'; to: string }
  | { type: 'TOAST'; variant: 'success' | 'error'; message: string }
  | { type: 'INVALIDATE_QUERY'; queryKey: unknown[] }

// features/settings/hooks/mutations/use-add-department.ts
export function useAddDepartment(options?: {
  onCommand?: (cmd: Command) => void
}) {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: settingsApi.addDepartment,
    meta: { toastError: true },
    onSuccess: (data) => {
      queryClient.invalidateQueries({ queryKey: settingsKeys.departments() })

      // Dispatch commands for side effects
      options?.onCommand?.({ type: 'TOAST', variant: 'success', message: 'Department added' })
      options?.onCommand?.({ type: 'NAVIGATE', to: `/settings/departments/${data.id}` })
    },
  })
}

// app/routes/settings/create-department.tsx
export function CreateDepartmentPage() {
  const navigate = useNavigate()

  const { mutate: addDepartment } = useAddDepartment({
    onCommand: (cmd) => {
      if (cmd.type === 'NAVIGATE') navigate(cmd.to)
      if (cmd.type === 'TOAST') toast[cmd.variant](cmd.message)
    },
  })

  return <CreateDepartmentForm onSubmit={addDepartment} />
}
```

### Effect Handlers (app/effects/)

**Effects implement side effects (isolated and testable).**

```typescript
// app/effects/analytics.effects.ts
export const analyticsEffects = {
  trackEvent: (eventName: string, properties?: Record<string, any>) => {
    // Analytics implementation
    console.log('Analytics:', eventName, properties)
  },
}

// app/effects/logging.effects.ts
export const loggingEffects = {
  logError: (error: Error, context?: Record<string, any>) => {
    console.error('Error:', error, context)
  },
}
```

### Subscriptions (app/subs/)

**Cross-feature derived data only. Feature-local derived data stays in feature's subs/ or query select.**

```typescript
// app/subs/cross-feature.subs.ts
import { useAccounts } from '@/features/settings'
import { useDashboardStats } from '@/features/dashboard'

// ✅ GOOD: Combines data from multiple features
export function useAccountsWithStats() {
  const { data: accounts } = useAccounts()
  const { data: stats } = useDashboardStats()

  return useMemo(() => {
    if (!accounts || !stats) return []
    return accounts.map(a => ({
      ...a,
      activityCount: stats.activityByAccount[a.id] ?? 0
    }))
  }, [accounts, stats])
}

// ❌ BAD: Feature-local derived data in app/subs
// This should be in features/settings/subs/ or query select
export function useActiveAccounts() {
  const { data: accounts } = useAccounts()
  return accounts?.filter(a => a.status === 'active') ?? []
}
```

---

## State Management (State Hierarchy)

### Server State: TanStack Query (Required)

**Data fetched from API managed with TanStack Query.**

- Don't duplicate with `useState`
- Don't fetch with `useEffect`
- Use queryKey factory
- Reactive updates via `invalidateQueries`

### Local UI State: useState / react-hook-form

**UI-local state (form input, modal open/close, tab selection) managed with `useState`.**

```typescript
function SearchBar() {
  const [query, setQuery] = useState('')
  const debouncedQuery = useDebounce(query, 300)

  const { data } = useSearchResults(debouncedQuery)

  return (
    <div>
      <input value={query} onChange={e => setQuery(e.target.value)} />
      <SearchResults results={data ?? []} />
    </div>
  )
}
```

### Application State: app/store/ (Small and Minimal)

**Use zustand/jotai for small cross-page UI state only when necessary.**

```typescript
// app/store/ui.store.ts
import { create } from 'zustand'

interface UIState {
  sidebarOpen: boolean
  toggleSidebar: () => void
}

export const useUIStore = create<UIState>((set) => ({
  sidebarOpen: true,
  toggleSidebar: () => set((state) => ({ sidebarOpen: !state.sidebarOpen })),
}))

// ❌ BAD: Don't put server state in store
// accounts: Account[]  // Should be managed by TanStack Query
```

---

## Component Patterns

### Container / Presenter (Subscribe / Render)

**Separate subscription in Container, rendering in Presenter.**

```typescript
// features/settings/components/AccountList.tsx

// Container (subscribe, define events)
export function AccountListContainer() {
  const { data, isLoading, isError } = useAccounts()
  const { mutate: deleteAccount } = useDeleteAccount()

  if (isLoading) return <Spinner />
  if (isError) return <ErrorView />

  return (
    <AccountListPresenter
      accounts={data ?? []}
      onDelete={deleteAccount}
    />
  )
}

// Presenter (pure rendering)
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

### Composition

```typescript
// components/ui/card.tsx
export function Card({ children, variant = 'default' }: CardProps) {
  return <div className={`card card-${variant}`}>{children}</div>
}

export function CardHeader({ children }: { children: React.ReactNode }) {
  return <div className="card-header">{children}</div>
}

export function CardBody({ children }: { children: React.ReactNode }) {
  return <div className="card-body">{children}</div>
}
```

---

## Forms

**Use `react-hook-form` + `zod` for forms.**

```typescript
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const schema = z.object({
  name: z.string().min(1, 'Name is required').max(200),
  email: z.string().email('Invalid email'),
})

type FormData = z.infer<typeof schema>

export function CreateDepartmentForm({ onSubmit }: { onSubmit: (data: FormData) => void }) {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm<FormData>({
    resolver: zodResolver(schema),
  })

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <input {...register('name')} placeholder="Department name" />
        {errors.name && <span className="error">{errors.name.message}</span>}
      </div>

      <div>
        <input type="email" {...register('email')} placeholder="Email" />
        {errors.email && <span className="error">{errors.email.message}</span>}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Creating...' : 'Create'}
      </button>
    </form>
  )
}
```

**Keep form state and server state separate.**

```typescript
// ✅ GOOD: Set initial value with react-hook-form's reset
function EditAccountForm({ accountId }: { accountId: string }) {
  const { data: account } = useAccount(accountId)
  const { register, handleSubmit, reset } = useForm<FormData>()

  useEffect(() => {
    if (account) {
      reset({ name: account.name, email: account.email })
    }
  }, [account, reset])

  // ...
}
```

---

## Error Handling

### Request Errors

**Normalize to `ApiError` in API layer, opt-in toast via meta.**

- Don't call `toast()` in pages
- Use `isError` only for display branching

```typescript
// ✅ GOOD: Display branching only
function Page() {
  const { data, isLoading, isError } = useAccounts()

  if (isLoading) return <Spinner />
  if (isError) return <ErrorView message="Failed to load data" />

  return <AccountList accounts={data} />
}
```

**Status-based handling:**
- 401/403: Handle via auth UI/redirect (don't toast)
- 422: Display in form errors (don't toast)
- 5xx: Opt-in toast for critical operations

### Render Errors

**ErrorBoundary is for render-time exceptions only.**

```typescript
// components/layout/ErrorBoundary.tsx
export class ErrorBoundary extends React.Component<
  { children: React.ReactNode },
  { hasError: boolean; error: Error | null }
> {
  state = { hasError: false, error: null }

  static getDerivedStateFromError(error: Error) {
    return { hasError: true, error }
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    loggingEffects.logError(error, { errorInfo })
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-fallback">
          <h2>An unexpected error occurred</h2>
          <button onClick={() => this.setState({ hasError: false, error: null })}>
            Retry
          </button>
        </div>
      )
    }

    return this.props.children
  }
}
```

---

## Performance

**Don't overuse `useMemo/useCallback` unnecessarily.**

- Write working code first
- Profile to find slow parts
- Optimize only where needed

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
```

**TanStack Query cache settings:**

```typescript
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60,  // Fresh for 1 minute
      gcTime: 1000 * 60 * 5,  // Keep cache for 5 minutes
      refetchOnWindowFocus: true,
      retry: 1,
    },
  },
})
```

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

---

## Accessibility

**Keyboard navigation and ARIA attributes:**

```typescript
// Focus management in modals
export function Modal({ isOpen, onClose, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null)

  useEffect(() => {
    if (isOpen) {
      const firstFocusable = modalRef.current?.querySelector<HTMLElement>(
        'button, [href], input, select, textarea'
      )
      firstFocusable?.focus()
    }
  }, [isOpen])

  if (!isOpen) return null

  return (
    <div
      ref={modalRef}
      role="dialog"
      aria-modal="true"
      onKeyDown={e => e.key === 'Escape' && onClose()}
    >
      {children}
    </div>
  )
}
```

---

## Testing

**Focus on integration tests. Unit tests for shared components/utils only.**

### Test Priority

1. **Integration Tests** (Primary): Test features as users interact
2. **E2E Tests**: Test critical user flows
3. **Unit Tests**: Shared components/utils only

### Tools

- **[Vitest](https://vitest.dev)**: Test runner
- **[Testing Library](https://testing-library.com/)**: Test user behavior
- **[Playwright](https://playwright.dev)**: E2E automation
- **[MSW](https://mswjs.io)**: Mock API server

### Example

```typescript
// features/settings/components/__tests__/account-list.test.tsx
import { render, screen, waitFor } from '@testing-library/react'
import { userEvent } from '@testing-library/user-event'

test('deletes account on click', async () => {
  render(<AccountListContainer />)

  await waitFor(() => expect(screen.getByText('John Doe')).toBeInTheDocument())

  await userEvent.click(screen.getByRole('button', { name: /delete/i }))

  await waitFor(() => expect(screen.queryByText('John Doe')).not.toBeInTheDocument())
})
```

---

## Security

### Authentication

**Use JWT. Prefer HttpOnly cookies over localStorage (XSS protection).**

```typescript
// lib/auth.tsx
const getUser = async () => {
  const res = await fetch('/api/auth/me') // Token via HttpOnly cookie
  if (!res.ok) return null
  return res.json()
}

export const userQueryOptions = queryOptions({
  queryKey: ['auth', 'user'],
  queryFn: getUser,
  staleTime: Infinity,
})

export function useUser() {
  return useQuery(userQueryOptions)
}
```

### Authorization

**RBAC (Role-Based) and PBAC (Permission-Based)**

```typescript
// lib/authorization.tsx
export function Authorization({ allowedRoles, policyCheck, children }: AuthorizationProps) {
  const { data: user } = useUser()
  if (!user) return null

  // RBAC: Role check
  if (allowedRoles && !allowedRoles.includes(user.role)) return null

  // PBAC: Policy check (granular, e.g. owner-only delete)
  if (policyCheck !== undefined && !policyCheck) return null

  return <>{children}</>
}

// Usage
<Authorization allowedRoles={['admin']}>
  <AdminPanel />
</Authorization>

<Authorization policyCheck={comment.authorId === user?.id}>
  <DeleteButton />
</Authorization>
```

### XSS Protection

**Sanitize user input with DOMPurify.**

```typescript
import DOMPurify from 'dompurify'

export function MDPreview({ content }: { content: string }) {
  return <div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(content) }} />
}
```

---

## Project Standards

### Tooling

- **ESLint**: Code quality, **Prettier**: Auto-format, **TypeScript**: Type safety (required), **Husky**: Pre-commit hooks

### Absolute Imports

**Configure `@/*` path alias.**

```json
// tsconfig.json
{ "compilerOptions": { "baseUrl": ".", "paths": { "@/*": ["./src/*"] } } }
```

```typescript
// ✅ GOOD
import { Card } from '@/components/ui/card'

// ❌ BAD
import { Card } from '../../../components/ui/card'
```

### File Naming

**Use kebab-case. Enforce with ESLint `check-file` plugin.**

```
user-settings/components/account-list.tsx
```

---

**Remember**: Write working code first. Optimize after measuring. Add complexity only when needed. Use bulletproof structure for organization, re-frame principles for behavior.
