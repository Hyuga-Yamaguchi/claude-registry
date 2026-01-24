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

**Note**: `noUncheckedIndexedAccess` treats array/dictionary access as `T | undefined`, improving null safety.

## Naming Conventions

**Variables/Functions**: Descriptive names, verb-noun pattern for functions, `is`/`has`/`should`/`can` prefix for booleans.

```typescript
const userCount = 42
const isEnabled = true
function calculateTotal(items: Item[]): number { }
function hasPermission(user: User): boolean { }
```

**Constants**: UPPER_SNAKE_CASE for configuration values (not all `const` declarations).

```typescript
const MAX_RETRIES = 3
const TIMEOUT_MS = 5000
const currentUser = getCurrentUser()  // Runtime value uses regular naming
```

**Note**: JavaScript has no compile-time constants. UPPER_SNAKE_CASE is a convention for configuration.

## Common Types

### Result Pattern

Use for operations with expected failures.

```typescript
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E }
```

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

function processResult<T>(result: Result<T>): void {
  if (result.success) {
    console.log(result.data)  // TypeScript narrows type
  } else {
    console.error(result.error)
  }
}
```

### Type Inference

Leverage inference for simple cases. Add explicit types when they validate behavior or improve clarity.

```typescript
const count = 0  // Inferred: number
const config: Config = loadConfig()  // Validates return type
```

### `satisfies` Operator

Validates type conformance while preserving inferred type precision.

```typescript
const config = {
  endpoint: 'data/source',
  timeout: 5000
} satisfies Config

config.endpoint  // Type typically remains "data/source" (not widened to string)
```

**When to use**: Catch mismatches at assignment while retaining precise types for autocomplete and refinement.

### `as const` Assertions

Creates deeply readonly literal types.

```typescript
const STATUSES = ['pending', 'active', 'archived'] as const
type Status = typeof STATUSES[number]  // 'pending' | 'active' | 'archived'
```

### Optional (`?`) vs `| undefined`

**Type-level similarity, call-site difference**: Both permit `undefined` values, but `?` allows omission while `| undefined` requires explicit argument.

```typescript
function greet(name?: string): void {
  console.log(name ?? 'Guest')
}
greet()  // OK
greet(undefined)  // OK

function setConfig(value: Config | undefined): void {
  // undefined signals "clear config"
}
setConfig(undefined)  // OK
setConfig()  // ❌ Error: Expected 1 argument
```

**Key insight**: `?` signals "may be absent". `| undefined` signals "undefined is a meaningful value".

## Imports and Modules

### Use `import type` for Type-Only Imports

Avoids emitting runtime imports for types. Reduces accidental runtime coupling and helps avoid circular import pitfalls.

```typescript
// ✅ Type-only import
import type { User, Product } from './models'
import { fetchUser } from './api'

// ❌ Mixed import when only types are used
import { User, Product } from './models'  // Runtime coupling
```

**Guideline**: Default to `import type` when importing only types.

## Immutability and `readonly`

### Design with Immutability by Default

Consider immutability during interface design. Favor `readonly` unless mutation is justified.

```typescript
// ✅ Immutable by design
interface Config {
  readonly endpoint: string
  readonly timeout: number
}

function updateConfig(config: Readonly<Config>): Config {
  return { ...config, timeout: 10000 }
}

// ✅ Function parameters
function process(items: readonly Item[]): Item[] {
  return [...items, newItem]
}
```

**Guideline**: Start with `readonly` and remove only when mutation is necessary.

### Immutability Patterns

Prefer immutable updates, but understand performance tradeoffs.

```typescript
// Immutable update (shallow copy)
const updated = { ...user, name: 'Alice' }
const newItems = [...items, newItem]

// ⚠️ Spread is shallow copy only
const nested = { ...config }
nested.database.timeout = 1000  // Mutates original!

// Deep update needs nested spread
const deepUpdate = {
  ...config,
  database: { ...config.database, timeout: 1000 }
}

// ⚠️ Performance consideration
const huge = new Array(100000).fill(0)
const copy = [...huge, 1]  // Copies 100k items

// ✅ Deliberate mutation when justified
function optimizedPush(items: Item[], item: Item): void {
  items.push(item)  // Performance-critical path
}
```

## Union Types vs Enums

### Prefer String Literal Unions

```typescript
type Status = 'pending' | 'active' | 'archived'

function setStatus(status: Status): void { }
setStatus('pending')
```

### Use `enum` Sparingly

Enums introduce runtime artifacts. Prefer unions unless you need:

1. **Reverse mapping** (numeric enums only)
2. **Namespace grouping** for exported constants

```typescript
enum HttpStatusCode {
  OK = 200,
  NotFound = 404,
  ServerError = 500
}

function handleResponse(code: HttpStatusCode): void {
  console.log(HttpStatusCode[code])  // Reverse mapping: "OK"
}
```

**Guideline**: Default to string literal unions. Avoid `const enum` in libraries due to `isolatedModules` and transpiler compatibility issues.

## Error Handling

### `throw` vs `Result` Pattern

**`throw` for programming errors** (invariant violations, contract breaches):

```typescript
function divide(a: number, b: number): number {
  if (b === 0) throw new Error('Division by zero')  // Contract violation
  return a / b
}
```

**`Result` for expected failures** (validation, parsing, external dependencies):

```typescript
function parseInput(input: string): Result<ParsedData> {
  try {
    const data = parse(input)
    return isValid(data)
      ? { success: true, data }
      : { success: false, error: new Error('Validation failed') }
  } catch (error) {
    return {
      success: false,
      error: error instanceof Error ? error : new Error('Parse failed')
    }
  }
}
```

**Consistency rule**: Avoid mixing `throw` and `Result` within the same layer. Internal logic may use `throw`; external boundaries should convert to `Result`.

### Typed Error Handling

Type guard `unknown` errors. Preserve context with `cause`.

```typescript
async function loadData(id: string): Promise<Data> {
  try {
    return await fetchFromSource(id)
  } catch (error) {
    const message = error instanceof Error ? error.message : 'Unknown error'
    throw new Error(`Failed to load ${id}: ${message}`, { cause: error })
  }
}
```

## Async/Await

### Parallel vs Sequential Execution

```typescript
// Parallel execution
const [users, products] = await Promise.all([fetchUsers(), fetchProducts()])

// Handle partial failures
const results = await Promise.allSettled([fetchUsers(), fetchProducts()])
results.forEach((result, index) => {
  if (result.status === 'fulfilled') {
    process(result.value)
  } else {
    logError(`Operation ${index} failed`, result.reason)
  }
})
```

**Type note**: `Promise.all<[T1, T2]>` returns tuple types. `Promise.allSettled` returns `PromiseSettledResult<T>[]`.

### Async Initialization

```typescript
// ✅ Static factory method
class DataLoader {
  private constructor(private data: Data) {}

  static async create(source: string): Promise<DataLoader> {
    const data = await loadData(source)
    return new DataLoader(data)
  }
}

// ❌ Async work in constructor
class DataLoader {
  constructor(source: string) {
    this.init(source)  // Fire-and-forget: no type safety for completion
  }
}
```

## Functions

### Small and Focused

Keep functions under 20 lines with single responsibility.

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
interface CreateUserOptions {
  name: string
  email: string
  role?: 'admin' | 'user'
}

function createUser(options: CreateUserOptions): User { }
```

### Early Returns

```typescript
function processItem(item: Item | null): Result<ProcessedItem> {
  if (!item) return { success: false, error: new Error('No item') }
  if (!item.isValid) return { success: false, error: new Error('Invalid item') }
  if (!hasPermission(item)) return { success: false, error: new Error('Forbidden') }
  return { success: true, data: transform(item) }
}
```

## Exhaustiveness Checking

Use `never` to ensure all union cases are handled.

```typescript
type Shape =
  | { kind: 'circle'; radius: number }
  | { kind: 'square'; size: number }

function area(shape: Shape): number {
  switch (shape.kind) {
    case 'circle':
      return Math.PI * shape.radius ** 2
    case 'square':
      return shape.size ** 2
    default:
      const _exhaustive: never = shape
      throw new Error(`Unhandled shape: ${_exhaustive}`)
  }
}
```

## Type Utilities

### Built-in Utilities

```typescript
type PartialUser = Partial<User>
type ReadonlyUser = Readonly<User>
type UserWithoutEmail = Omit<User, 'email'>
type UserIdAndName = Pick<User, 'id' | 'name'>
type UserKeys = keyof User
type NonNullableValue = NonNullable<string | null>
```

### Type Aliases vs Branded Types

**Type aliases** provide documentation but no runtime safety:

```typescript
type UserId = string
type ProductId = string

getUser(productId)  // ✅ TypeScript allows (same runtime type)
```

**Branded types** enforce type-level distinction:

```typescript
type UserId = string & { readonly __brand: 'UserId' }

// ✅ Factory function (validation/normalization boundary)
function createUserId(id: string): UserId {
  if (!id.startsWith('user-')) throw new Error('Invalid user ID format')
  return id as UserId
}

function getUser(id: UserId): User { }

const userId = createUserId('user-123')
getUser(userId)  // OK

// ❌ Avoid direct casting at usage sites
const productId = 'prod-456' as UserId  // Defeats type safety
```

**Guideline**: Type aliases for documentation. Branded types when preventing mixing is critical. Factory functions serve as validation boundaries.

### Avoid Over-Engineering Types

```typescript
// ⚠️ Complex mapped type (hard to debug)
type DeepReadonly<T> = {
  readonly [K in keyof T]: T[K] extends object ? DeepReadonly<T[K]> : T[K]
}

// ✅ Prefer simpler alternatives
type UserReadonly = Readonly<User>
```

**Guideline**: Favor clarity over cleverness. Complex types require clear documentation.

## Null Safety

### Optional Chaining and Nullish Coalescing

```typescript
const city = user?.address?.city
const firstItem = items?.[0]
const result = callback?.(arg)

// Nullish coalescing - Only null/undefined trigger default
const timeout = config.timeout ?? 5000

// ⚠️ Avoid || for numbers/booleans
const timeout = config.timeout || 5000  // 0 becomes 5000!
```

### Avoid Non-Null Assertion

```typescript
// ✅ Type guard
function processUser(user: User | null): void {
  if (!user) return
  console.log(user.name)
}

// ❌ Non-null assertion bypasses type safety
function processUser(user: User | null): void {
  console.log(user!.name)  // Runtime error if null
}

// ✅ Acceptable when invariant is guaranteed
function initialize(config: Config | null): void {
  if (!config) throw new Error('Config required')
  useConfig(config!)  // Safe: already checked
}
```

**Guideline**: Avoid `!` unless you have explicit runtime guarantees immediately before usage.

## Code Quality

### Avoid Magic Numbers

```typescript
const MAX_RETRIES = 3
const TIMEOUT_MS = 500

if (retryCount > MAX_RETRIES) { }
```

### Avoid Deep Nesting

```typescript
// ✅ Early returns
function validateOrder(order: Order): Result<Order> {
  if (!order) return { success: false, error: new Error('No order') }
  if (order.items.length === 0) return { success: false, error: new Error('Empty') }
  if (!order.customer) return { success: false, error: new Error('No customer') }
  return { success: true, data: order }
}

// ❌ Nested conditions
function validateOrder(order: Order): Result<Order> {
  if (order) {
    if (order.items.length > 0) {
      if (order.customer) {
        return { success: true, data: order }
      }
    }
  }
  return { success: false, error: new Error('Invalid') }
}
```

## Comments and Documentation

### JSDoc for Public APIs

```typescript
/**
 * Processes items using provided transformer.
 *
 * @param items - Items to process
 * @param transform - Transformation function
 * @returns Transformed items
 * @throws {ValidationError} If items contain invalid data
 */
export function processItems<T, R>(
  items: T[],
  transform: (item: T) => R
): R[] {
  return items.map(transform)
}
```

### Comments Explain WHY, Not WHAT

```typescript
// ✅ Explains rationale
// Binary search: array is pre-sorted with 10k+ items
const index = binarySearch(items, target)

// ❌ States the obvious
// Increment counter
count++
```

### Actionable TODOs

```typescript
// ✅ Context and ownership
// TODO(alice): Add caching before v2.0 release (issue #123)

// ❌ Vague
// TODO: fix this
```

---

**Principles**: Favor clarity over cleverness. Use TypeScript's type system to catch errors early, not to demonstrate advanced techniques. Write code that is easy to understand, modify, and delete.
