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

### Testing
- **[Vitest](https://vitest.dev)**, **[Testing Library](https://testing-library.com/)**, **[Playwright](https://playwright.dev)**, **[MSW](https://mswjs.io)**

---

## Directory Structure

```
src/
├── app/                    # App layer (composition + re-frame core)
│   ├── routes/             # Route definitions
│   ├── events/             # Event handlers (cross-feature orchestration)
│   ├── effects/            # Effect implementations (analytics, logging)
│   ├── subs/               # Cross-feature subscriptions
│   ├── store/              # Application state (jotai atoms)
│   └── provider.tsx        # Root providers
│
├── features/               # Feature modules (domain-driven, isolated)
│   ├── auth/               # ⚠️ Cross-feature domain (auth used everywhere)
│   ├── settings/
│   └── dashboard/
│
├── components/             # Shared UI components
├── lib/                    # Shared utilities (query-client, http, errors, hooks)
└── types/                  # Shared types
```

**Cross-feature exceptions**: Auth, pricing, plan, or other cross-cutting concerns can live in `features/` but are imported by other features as shared dependencies. Alternative: Create `src/domains/` for truly cross-cutting logic (features can import, but domains cannot import features/app).

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

User actions = events (handlers/mutations). Side effects in mutation callbacks or app/events.

### 3. Effects at Edges (Side Effects at Boundaries)

Side effects aggregated at: API layer (ApiError normalization), QueryClient (opt-in error toast), mutation callbacks (success toast, navigation, invalidation).

### 4. Single Source of Truth (TanStack Query)

Server state via TanStack Query (required). No `useState + useEffect` for fetching. Use queryKey factories. Reactive updates via `invalidateQueries`.

### 5. Unidirectional Codebase (shared → features → app)

`lib/`, `components/`, `types/` are shared (no feature/app dependencies). `features/` use shared code, cannot import other features (except cross-cutting domains like auth). `app/` composes features.

```typescript
// ✅ GOOD
import { Card } from '@/components/ui/card'
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
// lib/query-client.ts (TypeScript meta augmentation)
import '@tanstack/react-query'

declare module '@tanstack/react-query' {
  interface Register {
    defaultError: ApiError
  }
  interface QueryMeta {
    toastError?: boolean  // Opt-in error toast (queries: avoid; mutations: OK)
  }
  interface MutationMeta {
    toastError?: boolean
  }
}

export const queryClient = new QueryClient({
  mutationCache: new MutationCache({
    onError: (error, _vars, _ctx, mutation) => {
      if (mutation.meta?.toastError && error instanceof ApiError) {
        if (error.status === 401 || error.status === 403 || error.status === 422) return
        toast.error(error.message)
      }
    },
  }),
  defaultOptions: {
    queries: { staleTime: 60000, retry: 1 },
  },
})

// features/settings/hooks/mutations/use-add-department.ts
import type { Department } from '../types'  // ✅ FIX: Import Department

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
```

### HTTP Wrapper (Fixed Content-Type Issue)

```typescript
// lib/http.ts
import { ApiError } from './errors'

export async function http<T>(
  url: string,
  options?: RequestInit & { signal?: AbortSignal }
): Promise<T> {
  const hasBody = options?.body !== undefined
  const res = await fetch(url, {
    ...options,
    headers: {
      // ✅ FIX: Only add Content-Type if body exists and no custom Content-Type
      ...(hasBody && !options?.headers?.['Content-Type']
        ? { 'Content-Type': 'application/json' }
        : {}),
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

  if (res.status === 204) return undefined as T

  const contentType = res.headers.get('content-type')
  if (contentType?.includes('application/json')) return res.json()

  // ✅ Type-safe: If T is string, this is safe; otherwise caller must handle
  return res.text() as T
}
```

---

## Server State: TanStack Query

### QueryKey Factory

```typescript
// features/settings/hooks/keys.ts
export const settingsKeys = {
  all: ['settings'] as const,
  accounts: () => [...settingsKeys.all, 'accounts'] as const,
  accountDetail: (id: string) => [...settingsKeys.accounts(), id] as const,
}
```

### Query Hook (No Toast)

```typescript
// features/settings/hooks/queries/use-accounts.ts
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
// Optimistic delete example
export function useDeleteAccount() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: settingsApi.deleteAccount,
    onMutate: async (id) => {
      await queryClient.cancelQueries({ queryKey: settingsKeys.accounts() })
      const previous = queryClient.getQueryData(settingsKeys.accounts())
      queryClient.setQueryData(settingsKeys.accounts(), (old: Account[] = []) =>
        old.filter(a => a.id !== id)
      )
      return { previous }
    },
    onError: (_err, _vars, context) => {
      queryClient.setQueryData(settingsKeys.accounts(), context?.previous)
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
// app/events/types.ts
export type Command =
  | { type: 'NAVIGATE'; to: string }
  | { type: 'INVALIDATE_QUERY'; queryKey: unknown[] }

// app/routes/settings/create-department.tsx
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
// app/effects/analytics.effects.ts
export const analyticsEffects = {
  trackEvent: (name: string, props?: Record<string, any>) => {
    console.log('Analytics:', name, props)
  },
}
```

### Cross-feature Subscriptions

```typescript
// app/subs/cross-feature.subs.ts
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
// app/store/ui.atoms.ts
import { atom } from 'jotai'

export const sidebarOpenAtom = atom(true)
export const themeAtom = atom<'light' | 'dark'>('light')
export const isDarkModeAtom = atom((get) => get(themeAtom) === 'dark')
```

---

## Component Patterns

### Container / Presenter

```typescript
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

**Combine multiple array iterations into one loop.**

```typescript
// ❌ BAD: 3 iterations over users array
const admins = users.filter(u => u.isAdmin)
const testers = users.filter(u => u.isTester)
const inactive = users.filter(u => !u.isActive)

// ✅ GOOD: 1 iteration
const admins: User[] = []
const testers: User[] = []
const inactive: User[] = []

for (const user of users) {
  if (user.isAdmin) admins.push(user)
  if (user.isTester) testers.push(user)
  if (!user.isActive) inactive.push(user)
}
```

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
// lib/auth.tsx
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

**⚠️ CSRF Protection**: When using HttpOnly cookies, implement CSRF protection (SameSite=Strict/Lax + CSRF token for state-changing requests). Modern frameworks (Next.js API routes, etc.) often handle this automatically.

### Authorization (RBAC/PBAC)

```typescript
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
// features/settings/components/__tests__/account-list.test.tsx
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
- **Absolute Imports**: `@/*` in tsconfig (`import { Card } from '@/components/ui/card'`)
- **File Naming**: kebab-case (`account-list.tsx`), enforce with ESLint `check-file` plugin

---

**Remember**: Write working code first. Optimize after measuring. Add complexity only when needed. Use bulletproof structure for organization, re-frame principles for behavior.
