---
name: coding-standards-typescript
description: TypeScript language-level coding standards and best practices. Framework patterns should be in frontend-patterns.md. Formatting rules handled by Prettier/ESLint.
---

# TypeScript Coding Standards

Language-level standards for TypeScript. Framework-specific patterns in frontend-patterns.md. Formatting handled by Prettier/ESLint.

**Prerequisite**: Enable `strict` mode in `tsconfig.json`:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true
  }
}
```

## Naming Conventions

**Variables/Functions**: Descriptive names, verb-noun pattern for functions, `is`/`has`/`should`/`can` prefix for booleans.

```typescript
const userCount = 42, isEnabled = true
function calculateTotal(items: Item[]): number { }
function hasPermission(user: User): boolean { }
```

**Constants**: UPPER_SNAKE_CASE for configuration values (not all `const` declarations).

```typescript
const MAX_RETRIES = 3
const currentUser = getCurrentUser()  // Runtime value uses regular naming
```

**Note**: JavaScript has no compile-time constants. UPPER_SNAKE_CASE is a convention for configuration.

## Type Safety

### Avoid `any`, Use `unknown`

```typescript
function parseData(json: string): unknown {
  return JSON.parse(json)
}

const data = parseData(input)
if (typeof data === 'object' && data !== null && 'id' in data) {
  console.log(data.id)
}
```

### Union Types and Discriminated Unions

```typescript
type Status = 'pending' | 'active' | 'archived'

type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E }

function processResult<T>(result: Result<T>): void {
  if (result.success) {
    console.log(result.data)  // TypeScript narrows type
  } else {
    console.error(result.error)
  }
}
```

### Type Inference

Leverage inference for simple cases. Add explicit types when they catch errors or improve clarity.

```typescript
const count = 0  // Inferred: number
const config: Config = loadConfig()  // Validates return type
```

### `satisfies` Operator

Validates types without widening them.

```typescript
const config = {
  endpoint: '/api/data',
  timeout: 5000
} satisfies Config

config.endpoint  // Type: "/api/data" (literal, not string)
```

### `as const` Assertions

Creates deeply readonly literal types.

```typescript
const STATUSES = ['pending', 'active', 'archived'] as const
type Status = typeof STATUSES[number]  // 'pending' | 'active' | 'archived'

const ROUTES = { home: '/', profile: '/profile' } as const
```

### Optional (`?`) vs `| undefined`

**Function parameters**: `?` for optional (can omit), `| undefined` when undefined is meaningful.

```typescript
// Optional (can omit)
function greet(name?: string): void {
  console.log(name ?? 'Guest')
}
greet()  // OK
greet(undefined)  // OK

// Explicit undefined (must provide)
function setConfig(value: Config | undefined): void {
  // undefined means "clear config"
}
setConfig(undefined)  // OK
setConfig()  // ❌ Error: Expected 1 argument
```

**Object properties**: Prefer `?` for optional fields.

```typescript
interface User {
  id: string
  email?: string  // May or may not exist
}
```

**Key difference**: `?` allows omission. `| undefined` requires explicit value.

## Immutability

### Immutability Patterns

Prefer immutable updates, but understand tradeoffs.

```typescript
// Immutable update (shallow copy)
const updated = { ...user, name: 'Alice' }
const newItems = [...items, newItem]

// Readonly prevents mutation
function process(items: readonly Item[]): Item[] {
  // items.push(x)  // ❌ Compile error
  return [...items, newItem]
}

// ⚠️ Spread is shallow copy only
const nested = { ...config }
nested.api.timeout = 1000  // Mutates original!

// Deep update needs nested spread
const deepUpdate = {
  ...config,
  api: { ...config.api, timeout: 1000 }
}

// ⚠️ Performance tradeoff
const huge = new Array(100000).fill(0)
const updated = [...huge, 1]  // Copies 100k items

// ✅ Deliberate mutation when justified
function optimizedPush(items: Item[], item: Item): void {
  items.push(item)  // Document why
}
```

### Array Operations

```typescript
const filtered = items.filter(item => item.active)
const mapped = items.map(item => ({ ...item, processed: true }))
const sorted = [...items].sort((a, b) => a.value - b.value)  // Copy first
```

## Error Handling

### Typed Error Handling

Type guard `unknown` errors. Preserve context with `cause`.

```typescript
async function loadData(id: string): Promise<Data> {
  try {
    return await fetchFromSource(id)
  } catch (error) {
    const msg = error instanceof Error ? error.message : 'Unknown'
    throw new Error(`Failed to load ${id}: ${msg}`, { cause: error })
  }
}
```

### Result Pattern

Use for operations with expected failures.

```typescript
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E }

function parseInput(input: string): Result<ParsedData> {
  try {
    const data = parse(input)
    return isValid(data)
      ? { success: true, data }
      : { success: false, error: new Error('Invalid') }
  } catch (error) {
    return { success: false, error: error instanceof Error ? error : new Error('Parse failed') }
  }
}

const result = parseInput(input)
if (result.success) {
  process(result.data)
} else {
  handleError(result.error)
}
```

## Async/Await

### Parallel vs Sequential

```typescript
// Parallel
const [users, products, stats] = await Promise.all([
  fetchUsers(), fetchProducts(), fetchStats()
])

// Partial failures
const results = await Promise.allSettled([fetchUsers(), fetchProducts()])
results.forEach((r, i) => {
  if (r.status === 'fulfilled') {
    process(r.value)
  } else {
    logError(`Op ${i} failed`, r.reason)
  }
})
```

### Async Initialization

```typescript
// ✅ Static factory
class DataLoader {
  private constructor(private data: Data) {}

  static async create(source: string): Promise<DataLoader> {
    const data = await loadData(source)
    return new DataLoader(data)
  }
}

// ❌ Fire and forget
class DataLoader {
  constructor(source: string) {
    this.init(source)  // Race conditions
  }
}
```

## Functions

### Small and Focused

Keep <20 lines, single responsibility.

```typescript
function validateEmail(email: string): boolean {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email)
}

function validateUser(user: User): boolean {
  return user.name.length > 0 && validateEmail(user.email)
}
```

### Parameter Limits

Max 3-4 parameters. Use objects for complex signatures.

```typescript
interface CreateOptions {
  name: string
  email: string
  role?: 'admin' | 'user'
}

function createUser(options: CreateOptions): User { }

createUser({ name: 'Alice', email: 'alice@example.com' })
```

### Early Returns

```typescript
function processItem(item: Item | null): Result {
  if (!item) return { error: 'No item' }
  if (!item.isValid) return { error: 'Invalid' }
  if (!hasPermission(item)) return { error: 'Forbidden' }
  return { success: transform(item) }
}
```

## Exhaustiveness Checking

Use `never` to ensure all cases are handled.

```typescript
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'square'; size: number }
  | { kind: 'rectangle'; width: number; height: number }

function area(shape: Shape): number {
  switch (shape.kind) {
    case 'circle': return Math.PI * shape.radius ** 2
    case 'square': return shape.size ** 2
    case 'rectangle': return shape.width * shape.height
    default:
      const _exhaustive: never = shape  // Compile error if case missed
      throw new Error(`Unhandled: ${_exhaustive}`)
  }
}
```

If you add a new shape type, TypeScript errors at the `never` assignment.

## Type Utilities

### Built-in Utilities

```typescript
type PartialUser = Partial<User>
type ReadonlyUser = Readonly<User>
type UserWithoutEmail = Omit<User, 'email'>
type UserIdAndName = Pick<User, 'id' | 'name'>
type UserKeys = keyof User
```

### Type Aliases vs Branded Types

**Type aliases** (readability, no runtime safety):

```typescript
type UserId = string
type ProductId = string

getUser(productId)  // ✅ TypeScript allows (same runtime type)
```

**Branded types** (type-level safety):

```typescript
type UserId = string & { readonly __brand: 'UserId' }
type ProductId = string & { readonly __brand: 'ProductId' }

function createUserId(id: string): UserId { return id as UserId }
function getUser(id: UserId): User { }

const userId = createUserId('user-123')
const productId = 'prod-456' as ProductId
getUser(productId)  // ❌ Type error
```

**Guideline**: Type aliases for documentation. Branded types when preventing mixing is critical.

## Null Safety

### Optional Chaining and Nullish Coalescing

```typescript
// Optional chaining
const city = user?.address?.city
const firstItem = items?.[0]
const result = callback?.(arg)

// Nullish coalescing - Only null/undefined use default
const timeout = config.timeout ?? 5000
const count = userInput ?? 0

// ❌ Avoid || for numbers - 0, '', false trigger default
const timeout = config.timeout || 5000  // 0 becomes 5000!
```

### Avoid Non-Null Assertion

```typescript
// ✅ Type guard
function processUser(user: User | null): void {
  if (!user) return
  console.log(user.name)
}

// ❌ Non-null assertion
function processUser(user: User | null): void {
  console.log(user!.name)  // Runtime error if null
}

// ✅ ACCEPTABLE when you know it's safe
const element = document.getElementById('root')!
```

## Code Quality

### Avoid Magic Numbers

```typescript
const MAX_RETRIES = 3
const DEBOUNCE_MS = 500

if (retryCount > MAX_RETRIES) { }
```

### Avoid Deep Nesting

```typescript
function validateOrder(order: Order): ValidationResult {
  if (!order) return { valid: false, reason: 'No order' }
  if (order.items.length === 0) return { valid: false, reason: 'Empty' }
  if (!order.customer) return { valid: false, reason: 'No customer' }
  return { valid: true }
}
```

### Extract Common Logic

```typescript
function formatNumber(value: number, decimals: number): string {
  return value.toFixed(decimals)
}

const price = formatNumber(19.99, 2)
const percentage = formatNumber(0.156, 1)
```

## Comments and Documentation

### JSDoc for Public APIs

```typescript
/**
 * Processes items using transformer.
 *
 * @param items - Items to process
 * @param transform - Transformation function
 * @returns Transformed items
 * @throws {ValidationError} If invalid data
 */
export function processItems<T, R>(items: T[], transform: (item: T) => R): R[] {
  return items.map(transform)
}
```

### Comments Explain WHY

```typescript
// ✅ GOOD
// Binary search: array is pre-sorted with 10k+ items
const index = binarySearch(items, target)

// Deliberate mutation for 60fps requirement
positions.push(newPosition)

// ❌ BAD
// Increment counter
count++
```

### Actionable TODOs

```typescript
// ✅ GOOD
// TODO(alice): Add caching before v2.0 (issue #123)

// ❌ BAD
// TODO: fix this
```

---

**Remember**: Write code for humans first, machines second. TypeScript's type system is a tool for clarity and correctness, not an end in itself.
