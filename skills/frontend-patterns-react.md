---
name: frontend-patterns-react
description: Reagent-inspired React/Next.js patterns - subscription-driven UI, event/effect separation, side-effects at the edges, with TanStack Query as standard.
---

# React Frontend Patterns (Reagent-inspired)

Design guide applying Reagent/Re-frame philosophy (subscription model, event-driven, side-effect separation) to React/Next.js.

For language-level TypeScript standards, see [coding-standards-typescript.md](coding-standards-typescript.md).

---

## Non-negotiable Principles

### 1. Pure View (Subscribe and Render Only)

**UI subscribes to state and renders. Nothing more.**

- Page/Component follows: "read state → render"
- No side effects (API calls, toasts, navigation, logging) scattered in UI
- **`useEffect` for "watch state → trigger side effect" is prohibited**

```typescript
// ❌ BAD: useEffect watching state to trigger side effects
function Page() {
  const { data, isError } = useQuery(...)

  useEffect(() => {
    if (isError) {
      toast.error('Failed to load data')  // Side effect in UI layer
    }
  }, [isError])

  return <div>{data?.name}</div>
}

// ✅ GOOD: Subscribe and render only
function Page() {
  const { data, isLoading, isError } = useAccounts()

  if (isLoading) return <Spinner />
  if (isError) return <ErrorView />  // Display branching only, no side effects

  return <div>{data.map(account => <AccountCard key={account.id} account={account} />)}</div>
}
```

### 2. Event-driven (Change Through Events)

**User actions and updates are expressed as events (handlers / mutations).**

- Side effects confined to event side (mutation / handler) or boundaries (API layer / QueryClient / interceptor)
- "Event happens → state changes" not "state changes → do something"

```typescript
// ❌ BAD: useEffect watching state to do something
useEffect(() => {
  if (isSuccess) {
    toast.success('Created!')
    navigate('/list')
  }
}, [isSuccess])

// ✅ GOOD: Side effects in mutation's onSuccess
const { mutate: addDepartment } = useAddDepartment({
  onSuccess: () => {
    // Define side effects in mutation
    queryClient.invalidateQueries({ queryKey: settingsKeys.departments() })
    toast.success('Department added')
    navigate('/settings/departments')
  }
})
```

### 3. Effects at the Edges (Side Effects at Boundaries)

**Side effects are aggregated at boundaries (API layer, QueryClient, interceptor).**

- API errors normalized in API layer (`ApiError` etc), not interpreted in UI layer
- Toast displayed at **global layer** (QueryClient's `QueryCache/MutationCache` or HTTP interceptor)
- ErrorBoundary is for "render-time exceptions". Don't rely on it for request error toasts

### 4. Single Source of Truth (TanStack Query as Standard)

**Server state managed with `@tanstack/react-query` (required).**

- `useState + useEffect` for fetching is prohibited
- Custom `useQuery` / `DataLoader` should be removed
- Use queryKey factory to avoid hardcoded strings
- Reactive updates via `invalidateQueries` (Reagent-style)

### 5. Allowed `useEffect` Use Cases

**`useEffect` allowed only for UI-local concerns (DOM manipulation, measurement, focus).**

Allowed:
- Direct DOM manipulation (chart library, third-party widget)
- Focus control (Modal focus trap)
- IntersectionObserver, ResizeObserver
- Window event listeners (scroll, resize)

Prohibited:
- Data fetching (use TanStack Query)
- Watch state to toast/navigate/log (use mutation's onSuccess/onError)

---

## Server State: TanStack Query (Standard)

### QueryKey Factory Pattern

**Manage queryKey with factory functions to avoid hardcoded strings.**

```typescript
// ✅ GOOD: queryKey factory
export const settingsKeys = {
  all: ['settings'] as const,
  accounts: () => [...settingsKeys.all, 'accounts'] as const,
  accountDetail: (id: string) => [...settingsKeys.accounts(), id] as const,
  departments: () => [...settingsKeys.all, 'departments'] as const,
  departmentDetail: (id: string) => [...settingsKeys.departments(), id] as const,
}

// ❌ BAD: Hardcoded strings
useQuery({ queryKey: ['settings', 'accounts'], ... })
useQuery({ queryKey: ['accounts'], ... })  // Prone to inconsistency
```

### API Layer: ApiError and http wrapper

**Define unified error type in API layer and wrap fetch.**

```typescript
// api/errors.ts
export class ApiError extends Error {
  constructor(
    message: string,
    public status: number,
    public code?: string
  ) {
    super(message)
    this.name = 'ApiError'
  }
}

// api/http.ts
export async function http<T>(url: string, options?: RequestInit): Promise<T> {
  const res = await fetch(url, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      ...options?.headers,
    },
  })

  if (!res.ok) {
    const errorData = await res.json().catch(() => ({}))
    throw new ApiError(
      errorData.message || `HTTP ${res.status}`,
      res.status,
      errorData.code
    )
  }

  return res.json()
}

// api/settings.ts
export const settingsApi = {
  getAccounts: () => http<Account[]>('/api/settings/accounts'),
  getDepartments: () => http<Department[]>('/api/settings/departments'),
  addDepartment: (data: CreateDepartmentInput) =>
    http<Department>('/api/settings/departments', {
      method: 'POST',
      body: JSON.stringify(data),
    }),
}
```

### QueryClient: Global Toast (Errors Only)

**Define onError in QueryCache/MutationCache. Don't call toast in pages.**

```typescript
// lib/query-client.ts
import { QueryClient, QueryCache, MutationCache } from '@tanstack/react-query'
import { toast } from 'sonner'
import { ApiError } from '@/api/errors'

export const queryClient = new QueryClient({
  queryCache: new QueryCache({
    onError: (error) => {
      if (error instanceof ApiError) {
        toast.error(`Error: ${error.message}`)
      } else {
        toast.error('An unexpected error occurred')
      }
    },
  }),
  mutationCache: new MutationCache({
    onError: (error) => {
      if (error instanceof ApiError) {
        toast.error(`Operation failed: ${error.message}`)
      } else {
        toast.error('An unexpected error occurred')
      }
    },
  }),
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60,  // 1 minute
      retry: 1,
    },
  },
})
```

**Don't call toast in pages. Only display branching (error views).**

```typescript
// ❌ BAD: Calling toast in page
function Page() {
  const { data, isError, error } = useAccounts()

  useEffect(() => {
    if (isError) {
      toast.error(error.message)  // Already displayed at global layer
    }
  }, [isError, error])

  return <div>...</div>
}

// ✅ GOOD: Display branching only
function Page() {
  const { data, isLoading, isError } = useAccounts()

  if (isLoading) return <Spinner />
  if (isError) return <ErrorView message="Failed to load accounts" />

  return <AccountList accounts={data} />
}
```

### Feature Query Hooks

**Place `useXxxQuery` / `useXxxMutation` in feature layer.**

```typescript
// features/settings/queries/use-accounts.ts
import { useQuery } from '@tanstack/react-query'
import { settingsApi } from '@/api/settings'
import { settingsKeys } from './keys'

export function useAccounts() {
  return useQuery({
    queryKey: settingsKeys.accounts(),
    queryFn: settingsApi.getAccounts,
  })
}

// features/settings/queries/use-departments.ts
export function useDepartments() {
  return useQuery({
    queryKey: settingsKeys.departments(),
    queryFn: settingsApi.getDepartments,
  })
}

// features/settings/mutations/use-add-department.ts
import { useMutation, useQueryClient } from '@tanstack/react-query'
import { settingsApi } from '@/api/settings'
import { settingsKeys } from '../queries/keys'

export function useAddDepartment() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: settingsApi.addDepartment,
    onSuccess: () => {
      // Reagent-style reactive update via invalidateQueries
      queryClient.invalidateQueries({ queryKey: settingsKeys.departments() })
    },
  })
}
```

### Derived Data with `select` (No Local State Copy)

**Compute derived data with `select` or `useMemo`. Don't duplicate with `useState`.**

```typescript
// ❌ BAD: Copying server state to local state
function Page() {
  const { data } = useAccounts()
  const [activeAccounts, setActiveAccounts] = useState<Account[]>([])

  useEffect(() => {
    if (data) {
      setActiveAccounts(data.filter(a => a.status === 'active'))
    }
  }, [data])

  return <div>{activeAccounts.length}</div>
}

// ✅ GOOD: Compute derived data with select
function Page() {
  const { data: activeAccounts } = useQuery({
    queryKey: settingsKeys.accounts(),
    queryFn: settingsApi.getAccounts,
    select: (accounts) => accounts.filter(a => a.status === 'active'),
  })

  return <div>{activeAccounts?.length ?? 0}</div>
}

// ✅ GOOD: Compute derived data with useMemo
function Page() {
  const { data } = useAccounts()
  const activeAccounts = useMemo(
    () => data?.filter(a => a.status === 'active') ?? [],
    [data]
  )

  return <div>{activeAccounts.length}</div>
}
```

---

## Page Example: Subscribe → Render → Event

**Pure subscription-driven UI with no useEffect, no toast calls.**

```typescript
// pages/settings/accounts/index.tsx
import { useAccounts } from '@/features/settings/queries/use-accounts'
import { useDepartments } from '@/features/settings/queries/use-departments'
import { useAddDepartment } from '@/features/settings/mutations/use-add-department'
import { RequireRole } from '@/components/auth/require-role'

export default function AccountManagementPage() {
  return (
    <RequireRole role="admin">
      <AccountManagementContent />
    </RequireRole>
  )
}

function AccountManagementContent() {
  // Subscribe only (no side effects)
  const { data: accounts, isLoading: loadingAccounts } = useAccounts()
  const { data: departments, isLoading: loadingDepartments } = useDepartments()
  const { mutate: addDepartment, isPending } = useAddDepartment()

  // Event handler (just dispatch to mutation)
  const handleAddDepartment = (name: string) => {
    addDepartment({ name })
    // Side effects handled by mutation (invalidateQueries) or QueryClient (toast)
  }

  // Display branching (rendering only, no side effects)
  if (loadingAccounts || loadingDepartments) {
    return <Spinner />
  }

  return (
    <div>
      <h1>Account Management</h1>

      <section>
        <h2>Accounts</h2>
        <AccountList accounts={accounts ?? []} />
      </section>

      <section>
        <h2>Departments</h2>
        <DepartmentList
          departments={departments ?? []}
          onAdd={handleAddDepartment}
          isAdding={isPending}
        />
      </section>
    </div>
  )
}
```

**Key points:**
- No useEffect
- No toast calls (QueryClient handles it)
- Display branching only with data/isLoading
- Events dispatched to mutations (invalidateQueries defined in mutation)

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
// ✅ GOOD: UI local state
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

function Modal({ isOpen, onClose }: ModalProps) {
  // Local state within modal
  const [step, setStep] = useState(1)

  return (
    <Dialog open={isOpen} onClose={onClose}>
      {step === 1 && <Step1 onNext={() => setStep(2)} />}
      {step === 2 && <Step2 onBack={() => setStep(1)} />}
    </Dialog>
  )
}
```

### Cross-Page UI State: Only When Needed, Keep Small

**Introduce small cross-page state (jotai/zustand etc) only when necessary.**

- Don't create huge Context/Reducer upfront
- Don't put server state in Context (subscribe via TanStack Query)
- Don't put derived data in Context (compute with useMemo / select)

```typescript
// ✅ GOOD: Small UI state only (theme, sidebar toggle etc)
import { atom, useAtom } from 'jotai'

const sidebarOpenAtom = atom(true)

export function useSidebar() {
  return useAtom(sidebarOpenAtom)
}

// ❌ BAD: Duplicate server state in Context/Atom
const accountsAtom = atom<Account[]>([])  // Should be managed by TanStack Query
```

### Context + Reducer is "Last Resort"

**Use Context + Reducer only when complex UI state is needed.**

- Don't use for server state (use TanStack Query)
- Don't use for form state (use react-hook-form)
- Consider lightweight libraries (jotai/zustand) for cross-page state

```typescript
// ✅ GOOD: Complex UI state (multi-step wizard etc)
type WizardState = {
  step: number
  formData: Partial<CreateAccountInput>
  validationErrors: Record<string, string>
}

type WizardAction =
  | { type: 'NEXT_STEP' }
  | { type: 'PREV_STEP' }
  | { type: 'UPDATE_FORM'; payload: Partial<CreateAccountInput> }
  | { type: 'SET_ERRORS'; payload: Record<string, string> }

function wizardReducer(state: WizardState, action: WizardAction): WizardState {
  switch (action.type) {
    case 'NEXT_STEP':
      return { ...state, step: state.step + 1 }
    case 'PREV_STEP':
      return { ...state, step: Math.max(1, state.step - 1) }
    case 'UPDATE_FORM':
      return { ...state, formData: { ...state.formData, ...action.payload } }
    case 'SET_ERRORS':
      return { ...state, validationErrors: action.payload }
    default:
      return state
  }
}

// Use only within Wizard (not global)
function CreateAccountWizard() {
  const [state, dispatch] = useReducer(wizardReducer, {
    step: 1,
    formData: {},
    validationErrors: {},
  })

  // ...
}
```

---

## Component Patterns

### Composition

**Combine small components.**

```typescript
// ✅ GOOD: Split small and compose
export function Card({ children, variant = 'default' }: CardProps) {
  return <div className={`card card-${variant}`}>{children}</div>
}

export function CardHeader({ children }: { children: React.ReactNode }) {
  return <div className="card-header">{children}</div>
}

export function CardBody({ children }: { children: React.ReactNode }) {
  return <div className="card-body">{children}</div>
}

// Usage
<Card variant="elevated">
  <CardHeader>
    <h2>Title</h2>
  </CardHeader>
  <CardBody>
    <p>Content</p>
  </CardBody>
</Card>
```

### Compound Components

**Share state between parent and children using Context.**

```typescript
const TabsContext = createContext<{
  activeTab: string
  setActiveTab: (tab: string) => void
} | undefined>(undefined)

function useTabs() {
  const context = useContext(TabsContext)
  if (!context) throw new Error('Tabs components must be used within <Tabs>')
  return context
}

export function Tabs({ children, defaultTab }: TabsProps) {
  const [activeTab, setActiveTab] = useState(defaultTab)

  return (
    <TabsContext.Provider value={{ activeTab, setActiveTab }}>
      <div className="tabs">{children}</div>
    </TabsContext.Provider>
  )
}

export function Tab({ id, children }: { id: string; children: React.ReactNode }) {
  const { activeTab, setActiveTab } = useTabs()

  return (
    <button
      className={activeTab === id ? 'tab active' : 'tab'}
      onClick={() => setActiveTab(id)}
    >
      {children}
    </button>
  )
}

export function TabPanel({ id, children }: { id: string; children: React.ReactNode }) {
  const { activeTab } = useTabs()
  if (activeTab !== id) return null
  return <div className="tab-panel">{children}</div>
}

// Usage
<Tabs defaultTab="overview">
  <Tab id="overview">Overview</Tab>
  <Tab id="analytics">Analytics</Tab>

  <TabPanel id="overview"><Overview /></TabPanel>
  <TabPanel id="analytics"><Analytics /></TabPanel>
</Tabs>
```

### Container / Presenter (Subscribe / Render)

**Separate subscription in Container, rendering in Presenter.**

```typescript
// ✅ GOOD: Container (subscribe, define events)
function AccountListContainer() {
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

### Render Props for Rendering Flexibility Only

**Use Render Props for "rendering flexibility". Don't use for data fetching (use TanStack Query).**

```typescript
// ❌ BAD: Render Props for data fetching (prohibited)
function DataLoader({ url, children }: { url: string; children: (data: any) => ReactNode }) {
  const [data, setData] = useState(null)
  useEffect(() => { fetch(url).then(r => r.json()).then(setData) }, [url])
  return <>{children(data)}</>
}

// ✅ GOOD: Render Props for rendering flexibility
function List<T>({
  items,
  renderItem
}: {
  items: T[]
  renderItem: (item: T) => ReactNode
}) {
  return (
    <ul>
      {items.map((item, index) => (
        <li key={index}>{renderItem(item)}</li>
      ))}
    </ul>
  )
}

// Usage
<List
  items={accounts}
  renderItem={account => <AccountCard account={account} />}
/>
```

---

## Custom Hooks

### Toggle / Counter (UI state)

```typescript
// ✅ GOOD: UI local state management
export function useToggle(initialValue = false) {
  const [value, setValue] = useState(initialValue)

  const toggle = useCallback(() => setValue(v => !v), [])
  const setTrue = useCallback(() => setValue(true), [])
  const setFalse = useCallback(() => setValue(false), [])

  return { value, toggle, setTrue, setFalse }
}

// Usage
function Page() {
  const modal = useToggle()

  return (
    <>
      <button onClick={modal.setTrue}>Open</button>
      <Modal isOpen={modal.value} onClose={modal.setFalse} />
    </>
  )
}
```

### Debounce

```typescript
export function useDebounce<T>(value: T, delay: number): T {
  const [debouncedValue, setDebouncedValue] = useState<T>(value)

  useEffect(() => {
    const handler = setTimeout(() => setDebouncedValue(value), delay)
    return () => clearTimeout(handler)
  }, [value, delay])

  return debouncedValue
}

// Usage
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

### Local Storage

```typescript
export function useLocalStorage<T>(key: string, initialValue: T) {
  const [value, setValue] = useState<T>(() => {
    try {
      const item = window.localStorage.getItem(key)
      return item ? JSON.parse(item) : initialValue
    } catch {
      return initialValue
    }
  })

  const setStoredValue = useCallback(
    (newValue: T | ((prev: T) => T)) => {
      setValue(prev => {
        const valueToStore = newValue instanceof Function ? newValue(prev) : newValue
        window.localStorage.setItem(key, JSON.stringify(valueToStore))
        return valueToStore
      })
    },
    [key]
  )

  return [value, setStoredValue] as const
}

// Usage
const [theme, setTheme] = useLocalStorage('theme', 'dark')
```

---

## Performance

### Avoid Premature Optimization

**Don't overuse `useMemo/useCallback` unnecessarily.**

- Write working code first
- Profile to find slow parts
- Optimize only where needed

```typescript
// ❌ BAD: Unnecessary useMemo
function Component({ count }: { count: number }) {
  const doubled = useMemo(() => count * 2, [count])  // Unnecessary
  return <div>{doubled}</div>
}

// ✅ GOOD: If computation is cheap, just compute
function Component({ count }: { count: number }) {
  const doubled = count * 2
  return <div>{doubled}</div>
}

// ✅ GOOD: useMemo for expensive computation
function Component({ items }: { items: Item[] }) {
  const sorted = useMemo(
    () => items.slice().sort((a, b) => complexComparison(a, b)),
    [items]
  )
  return <List items={sorted} />
}
```

### React.memo for Pure Components

```typescript
// ✅ GOOD: Pure Presenter component
export const AccountCard = React.memo<{ account: Account }>(({ account }) => {
  return (
    <div className="account-card">
      <h3>{account.name}</h3>
      <p>{account.email}</p>
    </div>
  )
})
```

### TanStack Query Cache Settings

```typescript
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60,  // Fresh for 1 minute (no refetch)
      cacheTime: 1000 * 60 * 5,  // Keep cache for 5 minutes
      refetchOnWindowFocus: true,  // Refetch on window focus
      retry: 1,
    },
  },
})
```

### Virtualization (Large Lists)

```typescript
import { useVirtualizer } from '@tanstack/react-virtual'

export function VirtualizedList({ items }: { items: Item[] }) {
  const parentRef = useRef<HTMLDivElement>(null)

  const virtualizer = useVirtualizer({
    count: items.length,
    getScrollElement: () => parentRef.current,
    estimateSize: () => 80,
    overscan: 5,
  })

  return (
    <div ref={parentRef} style={{ height: '600px', overflow: 'auto' }}>
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: 'relative' }}>
        {virtualizer.getVirtualItems().map(virtualRow => (
          <div
            key={virtualRow.index}
            style={{
              position: 'absolute',
              top: 0,
              left: 0,
              width: '100%',
              height: `${virtualRow.size}px`,
              transform: `translateY(${virtualRow.start}px)`,
            }}
          >
            <ItemCard item={items[virtualRow.index]} />
          </div>
        ))}
      </div>
    </div>
  )
}
```

### Code Splitting

```typescript
import { lazy, Suspense } from 'react'

// ✅ GOOD: Route-based splitting
const Dashboard = lazy(() => import('./pages/Dashboard'))
const Settings = lazy(() => import('./pages/Settings'))

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

// ✅ GOOD: Heavy component splitting
const HeavyChart = lazy(() => import('./components/HeavyChart'))

function DashboardPage() {
  return (
    <div>
      <Suspense fallback={<ChartSkeleton />}>
        <HeavyChart data={data} />
      </Suspense>
    </div>
  )
}
```

---

## Forms

### react-hook-form Recommended

**Use `react-hook-form` + `zod` for forms.**

```typescript
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod'

const schema = z.object({
  name: z.string().min(1, 'Name is required').max(200),
  email: z.string().email('Invalid email'),
  description: z.string().min(1, 'Description is required'),
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

      <div>
        <textarea {...register('description')} placeholder="Description" />
        {errors.description && <span className="error">{errors.description.message}</span>}
      </div>

      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Creating...' : 'Create'}
      </button>
    </form>
  )
}
```

### Submit to Mutation

**Sync screen updates on submit success with `invalidateQueries` (Reagent-style).**

```typescript
function CreateDepartmentPage() {
  const { mutate: addDepartment } = useAddDepartment({
    onSuccess: () => {
      // Reactive update via invalidateQueries
      queryClient.invalidateQueries({ queryKey: settingsKeys.departments() })
      toast.success('Department added')
      navigate('/settings/departments')
    },
  })

  return (
    <CreateDepartmentForm onSubmit={addDepartment} />
  )
}
```

### Don't Mix Form State with Server State

**Keep form state and server state separate.**

```typescript
// ❌ BAD: Copy server state to form initial value via useState
function EditAccountForm({ accountId }: { accountId: string }) {
  const { data: account } = useAccount(accountId)
  const [formData, setFormData] = useState({ name: '', email: '' })

  useEffect(() => {
    if (account) {
      setFormData({ name: account.name, email: account.email })
    }
  }, [account])

  // ...
}

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

### Separate into Two Layers

#### 1. Request Errors

**Normalize to `ApiError` in API layer, global toast in QueryClient.**

- Don't call `toast()` in pages
- Use `isError` only for display branching, not side effects

```typescript
// ✅ GOOD: Display branching only
function Page() {
  const { data, isLoading, isError } = useAccounts()

  if (isLoading) return <Spinner />
  if (isError) return <ErrorView message="Failed to load data" />

  return <AccountList accounts={data} />
}

// ❌ BAD: useEffect for toast (already displayed at global layer)
function Page() {
  const { data, isError } = useAccounts()

  useEffect(() => {
    if (isError) {
      toast.error('An error occurred')  // Duplicate
    }
  }, [isError])

  return <div>...</div>
}
```

#### 2. Render Errors

**ErrorBoundary is for render-time exceptions. Cannot catch async request errors.**

```typescript
interface ErrorBoundaryProps {
  children: React.ReactNode
  fallback?: (error: Error, reset: () => void) => React.ReactNode
}

interface ErrorBoundaryState {
  hasError: boolean
  error: Error | null
}

export class ErrorBoundary extends React.Component<
  ErrorBoundaryProps,
  ErrorBoundaryState
> {
  state: ErrorBoundaryState = {
    hasError: false,
    error: null,
  }

  static getDerivedStateFromError(error: Error): ErrorBoundaryState {
    return { hasError: true, error }
  }

  componentDidCatch(error: Error, errorInfo: React.ErrorInfo) {
    console.error('ErrorBoundary caught:', error, errorInfo)
    // Send to error reporting service
  }

  reset = () => {
    this.setState({ hasError: false, error: null })
  }

  render() {
    if (this.state.hasError && this.state.error) {
      if (this.props.fallback) {
        return this.props.fallback(this.state.error, this.reset)
      }

      return (
        <div className="error-fallback">
          <h2>An unexpected error occurred</h2>
          <p>{this.state.error.message}</p>
          <button onClick={this.reset}>Retry</button>
        </div>
      )
    }

    return this.props.children
  }
}

// Usage
<ErrorBoundary>
  <App />
</ErrorBoundary>
```

**Note: Don't use ErrorBoundary for request error toasts.**

---

## Accessibility

### Keyboard Navigation

```typescript
export function Dropdown({ options, onSelect }: DropdownProps) {
  const [isOpen, setIsOpen] = useState(false)
  const [activeIndex, setActiveIndex] = useState(0)

  const handleKeyDown = (e: React.KeyboardEvent) => {
    switch (e.key) {
      case 'ArrowDown':
        e.preventDefault()
        if (isOpen) {
          setActiveIndex(i => Math.min(i + 1, options.length - 1))
        } else {
          setIsOpen(true)
        }
        break

      case 'ArrowUp':
        e.preventDefault()
        if (isOpen) {
          setActiveIndex(i => Math.max(i - 1, 0))
        }
        break

      case 'Enter':
      case ' ':
        e.preventDefault()
        if (isOpen) {
          onSelect(options[activeIndex])
          setIsOpen(false)
        } else {
          setIsOpen(true)
        }
        break

      case 'Escape':
        setIsOpen(false)
        break
    }
  }

  return (
    <div onKeyDown={handleKeyDown}>
      <button
        role="combobox"
        aria-expanded={isOpen}
        aria-haspopup="listbox"
        onClick={() => setIsOpen(!isOpen)}
      >
        Select option
      </button>

      {isOpen && (
        <ul role="listbox">
          {options.map((option, index) => (
            <li
              key={option.id}
              role="option"
              aria-selected={index === activeIndex}
              onClick={() => onSelect(option)}
            >
              {option.label}
            </li>
          ))}
        </ul>
      )}
    </div>
  )
}
```

### Focus Management (Modal)

```typescript
export function Modal({ isOpen, onClose, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null)
  const previousFocusRef = useRef<HTMLElement | null>(null)

  useEffect(() => {
    if (isOpen) {
      // Save currently focused element
      previousFocusRef.current = document.activeElement as HTMLElement

      // Focus first focusable element in modal
      const firstFocusable = modalRef.current?.querySelector<HTMLElement>(
        'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
      )
      firstFocusable?.focus()

      // Trap focus within modal with Tab key
      const handleTab = (e: KeyboardEvent) => {
        if (e.key !== 'Tab') return

        const focusableElements = modalRef.current?.querySelectorAll<HTMLElement>(
          'button, [href], input, select, textarea, [tabindex]:not([tabindex="-1"])'
        )
        if (!focusableElements) return

        const first = focusableElements[0]
        const last = focusableElements[focusableElements.length - 1]

        if (e.shiftKey && document.activeElement === first) {
          e.preventDefault()
          last.focus()
        } else if (!e.shiftKey && document.activeElement === last) {
          e.preventDefault()
          first.focus()
        }
      }

      document.addEventListener('keydown', handleTab)
      return () => document.removeEventListener('keydown', handleTab)
    } else {
      // Restore focus when closing modal
      previousFocusRef.current?.focus()
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

## Advanced: Global Success Toast (Optional)

**Use mutationKey and branch in MutationCache.onSuccess.**

```typescript
// lib/query-client.ts
export const queryClient = new QueryClient({
  mutationCache: new MutationCache({
    onSuccess: (_data, _variables, _context, mutation) => {
      // Branch by mutationKey
      const key = mutation.options.mutationKey?.[0]

      if (key === 'addDepartment') {
        toast.success('Department added')
      } else if (key === 'updateAccount') {
        toast.success('Account updated')
      }
      // Default: do nothing
    },
    onError: (error) => {
      if (error instanceof ApiError) {
        toast.error(`Operation failed: ${error.message}`)
      } else {
        toast.error('An unexpected error occurred')
      }
    },
  }),
})

// mutations/use-add-department.ts
export function useAddDepartment() {
  const queryClient = useQueryClient()

  return useMutation({
    mutationKey: ['addDepartment'],  // Define key
    mutationFn: settingsApi.addDepartment,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: settingsKeys.departments() })
    },
  })
}
```

---

## Supplement: Integration with Next.js App Router

**Even when using Next.js App Router Server Components / Route Handlers, TanStack Query remains the standard.**

- Server Components: Initial data fetch (SSR)
- Client Components: Dynamic updates with TanStack Query
- Route Handlers: API endpoints

```typescript
// app/settings/accounts/page.tsx (Server Component)
import { getAccounts } from '@/api/settings-server'
import { AccountListClient } from './client'

export default async function AccountsPage() {
  const initialAccounts = await getAccounts()

  return <AccountListClient initialAccounts={initialAccounts} />
}

// app/settings/accounts/client.tsx (Client Component)
'use client'

import { useAccounts } from '@/features/settings/queries/use-accounts'

export function AccountListClient({ initialAccounts }: { initialAccounts: Account[] }) {
  const { data } = useAccounts({
    initialData: initialAccounts,  // Use SSR data as initial
  })

  return <AccountList accounts={data} />
}
```

---

## Reagent-inspired Checklist (Self-Check)

Verify the following:

- [ ] `useEffect` is limited to UI-local concerns (DOM/focus/measurement)
- [ ] Data fetching uses TanStack Query only (no custom fetch hooks)
- [ ] Error toasts displayed at global layer (QueryCache/MutationCache or interceptor)
- [ ] No toast calls in pages/components (success toasts externalized when possible)
- [ ] No duplicate server state management with local state
- [ ] Using queryKey factory (no hardcoded strings)
- [ ] Mutation success syncs via `invalidateQueries`
- [ ] UI follows "subscribe → render → event" flow
- [ ] Side effects (API calls, toasts, navigation) aggregated at boundaries

---

**Remember**: Write working code first. Optimize after measuring. Add complexity only when needed.
