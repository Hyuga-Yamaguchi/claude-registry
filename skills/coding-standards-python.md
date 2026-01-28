---
name: coding-standards-python
description: Python coding standards with functional programming principles inspired by Lisp. Emphasizes map/filter/reduce, immutability, pure functions, and function composition. For Python 3.11+.
---

# Python Coding Standards (Functional Style)

Language-level standards for Python 3.11+ with functional programming principles inspired by Lisp. Backend patterns in backend-patterns-fastapi.md. Formatting handled by Ruff.

**Philosophy**: Favor pure functions, immutability, and composable transformations. Minimize side effects and explicit loops.

---

## Functional Programming Principles

### 1. Pure Functions First

**Pure Function**: Same input → same output, no side effects.

```python
# ✅ Pure
def calculate_total(items: list[float]) -> float:
    return sum(items)

# ❌ Impure (modifies global state)
total = 0
def add_to_total(amount: float) -> None:
    global total
    total += amount
```

**Benefits**: Easy to test, reason about, parallelize, and cache.

### 2. Immutability by Default

```python
from dataclasses import dataclass

# ✅ Immutable
@dataclass(frozen=True)
class Point:
    x: float
    y: float

def move(point: Point, dx: float, dy: float) -> Point:
    return Point(point.x + dx, point.y + dy)

# ❌ Mutable (harder to reason about)
@dataclass
class MutablePoint:
    x: float
    y: float

def move_mutate(point: MutablePoint, dx: float, dy: float) -> None:
    point.x += dx
    point.y += dy
```

**Allow mutability**: Performance-critical code (after profiling), interfacing with imperative libraries, local variables never exposed.

### 3. Prefer Expressions Over Statements

```python
# ✅ Expression-oriented
status = "active" if user.is_verified else "pending"

# ❌ Statement-oriented
if user.is_verified:
    status = "active"
else:
    status = "pending"
```

---

## List Processing (Map, Filter, Reduce)

### Map: Transform Elements

```python
# ✅ Comprehension (Pythonic)
discounted = [p * 0.9 for p in prices]

# ✅ map() with named function (complex logic)
def apply_tax(price: float) -> float:
    return price * 1.1

taxed = list(map(apply_tax, prices))

# ❌ Explicit loop (avoid for pure transformations)
result = []
for price in prices:
    result.append(price * 0.9)
```

**Use comprehensions** for simple transformations, **map()** for higher-order functions or lazy evaluation.

### Filter: Select Elements

```python
# ✅ Comprehension
evens = [n for n in numbers if n % 2 == 0]

# ✅ filter() with predicate
def is_adult(age: int) -> bool:
    return age >= 18

adults = list(filter(is_adult, ages))
```

### Reduce: Aggregate Values

```python
from functools import reduce
from operator import add, mul

total = reduce(add, numbers, 0)
product = reduce(mul, numbers, 1)

# Prefer built-ins when available
total = sum(numbers)
all_true = all(conditions)
```

### Combining Transformations

```python
# ✅ Generator expression (memory efficient)
total = sum(
    item["price"] * item["quantity"]
    for item in items
    if item["price"] > 100
)

# ✅ Explicit pipeline (clearer intent)
total = sum(map(get_value, filter(is_expensive, items)))
```

---

## For Loops: Only for Side Effects

**Principle**: Use `for` only for side effects (I/O, mutations). For pure transformations, use map/filter/comprehensions.

### ✅ Allowed: Side Effects

```python
# I/O operations
for user in users:
    logger.info("Processing: %s", user.id)
    send_email(user.email)

# External API calls
for url in urls:
    response = requests.get(url)
    process_response(response)

# Database operations
for record in records:
    db.insert(record)
```

### ❌ Prohibited: Pure Transformations

```python
# ❌ BAD: for loop for transformation
result = []
for item in items:
    result.append(transform(item))

# ✅ GOOD: comprehension
result = [transform(item) for item in items]

# ❌ BAD: for loop for aggregation
total = 0
for item in items:
    total += item.price

# ✅ GOOD: sum
total = sum(item.price for item in items)
```

**Exception**: Performance-critical code (after profiling). Document the reason.

---

## Function Composition and Pipelines

### Composition

```python
from typing import Callable, TypeVar

T = TypeVar("T")
U = TypeVar("U")
V = TypeVar("V")

def compose(f: Callable[[U], V], g: Callable[[T], U]) -> Callable[[T], V]:
    """compose(f, g)(x) = f(g(x))"""
    return lambda x: f(g(x))

# Example
final_price = compose(add_tax, apply_discount)
result = final_price(100)  # (100 * 0.9) * 1.1
```

### Pipeline

```python
def pipe(value: T, *functions: Callable) -> T:
    """Apply functions left-to-right."""
    result = value
    for func in functions:
        result = func(result)
    return result

# Example
username = pipe(
    "  John Doe  ",
    str.lower,
    str.strip,
    lambda s: s.replace(" ", ""),
    lambda s: f"user_{s}",
)
```

**Optional**: Use `toolz` library for `pipe()` and `compose()`.

---

## Higher-Order Functions

```python
# Takes function as argument
def apply_twice(func: Callable[[int], int], value: int) -> int:
    return func(func(value))

# Returns function
def make_multiplier(factor: int) -> Callable[[int], int]:
    return lambda n: n * factor

# Partial application
from functools import partial

say_hello = partial(greet, "Hello")
```

---

## Lazy Evaluation

```python
from typing import Iterator
from itertools import islice

# Generator - infinite sequence
def fibonacci() -> Iterator[int]:
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

first_10 = list(islice(fibonacci(), 10))

# Generator expression - memory efficient
squares = (x**2 for x in range(1000000))
first_5 = list(islice(squares, 5))

# Lazy file processing
def process_large_file(path: str) -> Iterator[dict]:
    with open(path) as f:
        for line in f:
            yield parse_line(line)
```

---

## Recursion (Use Carefully)

**Python has limited stack depth (default: 1000). Prefer iteration or reduce.**

```python
# ✅ Tree traversal (natural recursion)
@dataclass(frozen=True)
class Node:
    value: int
    left: "Node | None" = None
    right: "Node | None" = None

def sum_tree(node: Node | None) -> int:
    if node is None:
        return 0
    return node.value + sum_tree(node.left) + sum_tree(node.right)

# ✅ Better: reduce for factorial
from functools import reduce
from operator import mul

def factorial(n: int) -> int:
    return reduce(mul, range(1, n + 1), 1)
```

---

## Pattern Matching (Python 3.10+)

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Circle:
    radius: float

@dataclass(frozen=True)
class Rectangle:
    width: float
    height: float

Shape = Circle | Rectangle

def area(shape: Shape) -> float:
    match shape:
        case Circle(radius=r):
            return 3.14159 * r * r
        case Rectangle(width=w, height=h):
            return w * h
```

---

## Naming Conventions

```python
# Variables and functions: snake_case
user_count = 42
def calculate_total(items: list) -> float: ...

# Classes: PascalCase
class UserRepository: ...

# Constants: UPPER_SNAKE_CASE
MAX_RETRIES = 3

# Private: _prefix
class User:
    def __init__(self, name: str):
        self._name = name
```

---

## Type Hints

### Use Type Hints for Public APIs

```python
def fetch_user(user_id: str) -> dict[str, Any]: ...
def process_items(items: list[str]) -> list[int]: ...

# Modern syntax (Python 3.10+)
def get_user(user_id: str) -> User | None: ...
```

### Prefer Structured Types

```python
from typing import TypedDict
from dataclasses import dataclass

# TypedDict: static type checking
class UserPayload(TypedDict):
    id: str
    name: str
    email: str

# Dataclass: domain models
@dataclass(frozen=True)
class User:
    id: str
    name: str
    email: str
```

### When to Use `Any`

Use `Any` only at system boundaries. Document the reason.

```python
def parse_external_api(response: dict[str, Any]) -> User:
    """Uses Any because external schema is not under our control."""
    return User(
        id=str(response["id"]),
        name=str(response["name"]),
        email=str(response["email"]),
    )
```

### Use Literal for Fixed Values

```python
from typing import Literal

Status = Literal["active", "inactive", "archived"]
```

---

## Error Handling

### Exception Inheritance

```python
class ApplicationError(Exception):
    """Base for all application errors."""

class ValidationError(ApplicationError):
    """Validation failure."""
```

### Preserve Exception Chain

```python
try:
    return json.loads(config_str)
except json.JSONDecodeError as e:
    raise ValueError("Invalid config") from e
```

### Result Pattern (Functional Error Handling)

```python
from dataclasses import dataclass
from typing import Generic, TypeVar

T = TypeVar("T")
E = TypeVar("E")

@dataclass(frozen=True)
class Success(Generic[T]):
    value: T

@dataclass(frozen=True)
class Failure(Generic[E]):
    error: E

Result = Success[T] | Failure[E]

# Usage with pattern matching
result = parse_config(data)
match result:
    case Success(value):
        process(value)
    case Failure(error):
        logger.error("Failed: %s", error)
```

---

## Data Structures

### Dataclasses for Domain Models

```python
from dataclasses import dataclass
from decimal import Decimal

@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str

@dataclass(frozen=True)
class OrderItem:
    product_id: str
    quantity: int
    price: Decimal
```

### Enums for Fixed Sets

```python
from enum import Enum, auto

class Status(Enum):
    PENDING = auto()
    CONFIRMED = auto()
    SHIPPED = auto()

def update_status(status: Status) -> None:
    match status:
        case Status.PENDING: ...
        case Status.CONFIRMED: ...
        case Status.SHIPPED: ...
```

---

## Itertools and Functools

```python
from itertools import chain, groupby, islice, takewhile
from functools import reduce, partial, lru_cache

# chain: flatten nested lists
flat = list(chain.from_iterable([[1, 2], [3, 4]]))

# partial: specialized functions
double = partial(mul, 2)

# lru_cache: memoization for pure functions
@lru_cache(maxsize=128)
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)
```

---

## Documentation

### Docstrings for Public APIs

```python
def search_items(query: str, limit: int = 10) -> list[dict]:
    """Search items using semantic similarity.

    Args:
        query: Search query string
        limit: Maximum results (default: 10)

    Returns:
        List of items sorted by relevance

    Raises:
        ValueError: If query is empty
    """
```

### Comments Explain WHY

```python
# Exponential backoff prevents API overload during outages
delay = min(30, 2**retry_count)
```

---

## Code Quality

### Avoid Magic Numbers

```python
MAX_RETRIES = 3
TIMEOUT_SECONDS = 30

if retry_count > MAX_RETRIES:
    raise MaxRetriesExceeded()
```

### Early Returns

```python
def process_order(order: Order | None) -> Result[Order, str]:
    if order is None:
        return Failure("Order not found")
    if not order.items:
        return Failure("No items")
    return Success(complete_order(order))
```

---

## Tooling and Static Analysis

### Required Tools

- Python 3.11+
- Ruff for linting and formatting
- pyright for type checking (strict mode)

### Type Checking Configuration

```json
{
  "typeCheckingMode": "strict"
}
```

### CI Enforcement

- Ruff linting (zero errors)
- Ruff formatting check
- pyright strict mode
- No implicit `Any` in public APIs (except documented boundaries)

---

## Public API Definition

A module's public API consists of:
- Top-level functions/classes without `_` prefix
- Names in `__all__` if present
- Imports in `__init__.py`

**Public API requirements**:
- Must have type annotations
- Must have docstrings
- Must validate external inputs
- Should avoid `Any` (boundary exceptions documented)

**Private code** (`_` prefix):
- May omit docstrings if obvious
- Should still have type hints

---

## Functional Style Summary

### Prefer ✅

- Pure functions (no side effects)
- Immutable data (`frozen=True`)
- map/filter/reduce over explicit loops
- List comprehensions for simple transformations
- Generator expressions for lazy evaluation
- Function composition and pipelines
- Pattern matching for data destructuring
- Result type for expected errors
- Expressions over statements

### Avoid ❌

- Global state
- Mutable data structures (unless necessary)
- `for` loops for pure transformations
- Deep recursion (Python stack limit)
- Mutating function arguments
- Side effects in pure functions
- Magic numbers

### Allow (with Justification) ⚠️

- `for` loops for side effects (I/O, mutations)
- Mutable data for performance (after profiling)
- Recursion for tree structures (shallow depth)
- `Any` at system boundaries (documented)

---

**Principles** (from The Zen of Python):
- Explicit is better than implicit
- Simple is better than complex
- Readability counts

**Functional Principles** (from Lisp/FP):
- Data and functions are separate
- Functions are first-class
- Immutability prevents bugs
- Composition over inheritance
