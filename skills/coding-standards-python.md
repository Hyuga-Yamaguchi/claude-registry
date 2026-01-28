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
# ✅ Pure function
def calculate_total(items: list[float]) -> float:
    """Pure: deterministic, no side effects."""
    return sum(items)

def apply_discount(price: float, discount: float) -> float:
    """Pure: no mutations, no I/O."""
    return price * (1 - discount)

# ❌ Impure function
total = 0  # Global state

def add_to_total(amount: float) -> None:
    """Impure: modifies global state."""
    global total
    total += amount
```

**Benefits**:
- Easy to test (no setup/teardown)
- Easy to reason about
- Parallelizable
- Cacheable (memoization)

### 2. Immutability by Default

**Principle**: Data should not be modified after creation.

```python
from dataclasses import dataclass

# ✅ Immutable data
@dataclass(frozen=True)
class Point:
    x: float
    y: float

def move(point: Point, dx: float, dy: float) -> Point:
    """Return new Point instead of modifying."""
    return Point(point.x + dx, point.y + dy)

# ❌ Mutable data
@dataclass
class MutablePoint:
    x: float
    y: float

def move_mutate(point: MutablePoint, dx: float, dy: float) -> None:
    """Mutates argument - harder to reason about."""
    point.x += dx
    point.y += dy
```

**When to allow mutability**:
- Performance-critical code (after profiling)
- Interfacing with imperative libraries
- Local variables never exposed (encapsulated mutation)

### 3. Prefer Expressions Over Statements

```python
# ✅ Expression-oriented
status = "active" if user.is_verified else "pending"

result = (
    Success(process(data))
    if validate(data)
    else Failure("Invalid data")
)

# ❌ Statement-oriented
if user.is_verified:
    status = "active"
else:
    status = "pending"
```

---

## List Processing (Map, Filter, Reduce)

### Map: Transform Each Element

**Use `map()` or comprehensions for transformations.**

```python
from typing import Callable

# ✅ Map - functional style
prices = [100, 200, 300]
discounted = list(map(lambda p: p * 0.9, prices))

# ✅ List comprehension - more Pythonic
discounted = [p * 0.9 for p in prices]

# ✅ Map with named function (best for complex logic)
def apply_tax(price: float) -> float:
    return price * 1.1

taxed_prices = list(map(apply_tax, prices))

# ❌ Explicit loop (avoid for pure transformations)
discounted = []
for price in prices:
    discounted.append(price * 0.9)
```

**When to use `map()` vs comprehensions**:
- Simple transformations: Use comprehensions (more readable)
- Higher-order functions: Use `map()` (e.g., `map(str.upper, names)`)
- Lazy evaluation needed: Use `map()` (returns iterator)

### Filter: Select Elements

**Use `filter()` or comprehensions for filtering.**

```python
# ✅ Filter - functional style
numbers = [1, 2, 3, 4, 5, 6]
evens = list(filter(lambda n: n % 2 == 0, numbers))

# ✅ List comprehension - more Pythonic
evens = [n for n in numbers if n % 2 == 0]

# ✅ Filter with predicate function
def is_adult(age: int) -> bool:
    return age >= 18

adults = list(filter(is_adult, ages))

# ❌ Explicit loop (avoid)
evens = []
for n in numbers:
    if n % 2 == 0:
        evens.append(n)
```

### Reduce: Aggregate Values

**Use `functools.reduce()` for aggregations.**

```python
from functools import reduce
from operator import add, mul

# ✅ Reduce - sum
numbers = [1, 2, 3, 4, 5]
total = reduce(add, numbers, 0)

# ✅ Reduce - product
product = reduce(mul, numbers, 1)

# ✅ Reduce - custom aggregation
def merge_dicts(acc: dict, item: dict) -> dict:
    return {**acc, **item}

dicts = [{"a": 1}, {"b": 2}, {"c": 3}]
merged = reduce(merge_dicts, dicts, {})

# ✅ Built-in alternatives (prefer when available)
total = sum(numbers)  # Better than reduce(add, ...)
all_true = all(conditions)  # Better than reduce(and_, ...)
any_true = any(conditions)  # Better than reduce(or_, ...)
```

### Combining Map/Filter/Reduce

```python
from functools import reduce

# ✅ Pipeline of transformations
items = [
    {"price": 100, "quantity": 2},
    {"price": 200, "quantity": 1},
    {"price": 50, "quantity": 5},
]

# Calculate total value of items over $100
total = sum(
    item["price"] * item["quantity"]
    for item in items
    if item["price"] > 100
)

# Or using map/filter (more explicit pipeline)
def get_value(item: dict) -> float:
    return item["price"] * item["quantity"]

def is_expensive(item: dict) -> bool:
    return item["price"] > 100

total = sum(map(get_value, filter(is_expensive, items)))
```

---

## For Loops: Only for Side Effects

**Principle**: Use `for` loops only when performing side effects (I/O, mutations, state changes). For pure transformations, use map/filter/comprehensions.

### ✅ Allowed: Side Effects

```python
import logging

logger = logging.getLogger(__name__)

# ✅ I/O operations
for user in users:
    logger.info("Processing user: %s", user.id)
    send_email(user.email)

# ✅ Mutations (when necessary)
for item in cart.items:
    item.mark_as_processed()  # External state change

# ✅ External API calls
for url in urls:
    response = requests.get(url)  # Network I/O
    process_response(response)

# ✅ Database operations
for record in records:
    db.insert(record)  # Database side effect
```

### ❌ Prohibited: Pure Transformations

```python
# ❌ BAD: Using for loop for transformation
result = []
for item in items:
    result.append(transform(item))

# ✅ GOOD: Use comprehension
result = [transform(item) for item in items]

# ❌ BAD: Using for loop for filtering
filtered = []
for item in items:
    if predicate(item):
        filtered.append(item)

# ✅ GOOD: Use filter or comprehension
filtered = [item for item in items if predicate(item)]

# ❌ BAD: Using for loop for aggregation
total = 0
for item in items:
    total += item.price

# ✅ GOOD: Use sum or reduce
total = sum(item.price for item in items)
```

### Exception: Performance-Critical Code

When profiling shows comprehensions are a bottleneck, explicit loops with pre-allocation may be used. Document the reason.

```python
# Performance-critical: pre-allocate array
# Profiling showed 2x speedup for 1M+ items
result = [0] * len(items)
for i, item in enumerate(items):
    result[i] = expensive_transform(item)
```

---

## Function Composition and Pipelines

### Function Composition

**Compose small functions into larger ones.**

```python
from typing import Callable, TypeVar

T = TypeVar("T")
U = TypeVar("U")
V = TypeVar("V")

def compose(f: Callable[[U], V], g: Callable[[T], U]) -> Callable[[T], V]:
    """Compose two functions: compose(f, g)(x) = f(g(x))."""
    return lambda x: f(g(x))

# Example
def add_tax(price: float) -> float:
    return price * 1.1

def apply_discount(price: float) -> float:
    return price * 0.9

# Compose: first discount, then tax
final_price = compose(add_tax, apply_discount)
result = final_price(100)  # (100 * 0.9) * 1.1 = 99
```

### Pipeline Pattern

**Process data through a series of transformations.**

```python
from typing import Callable, TypeVar

T = TypeVar("T")

def pipe(value: T, *functions: Callable) -> T:
    """Apply functions left-to-right."""
    result = value
    for func in functions:
        result = func(result)
    return result

# Example
def normalize(text: str) -> str:
    return text.lower().strip()

def remove_spaces(text: str) -> str:
    return text.replace(" ", "")

def add_prefix(text: str) -> str:
    return f"user_{text}"

# Pipeline
username = pipe(
    "  John Doe  ",
    normalize,       # "john doe"
    remove_spaces,   # "johndoe"
    add_prefix,      # "user_johndoe"
)
```

### Using `toolz` Library (Optional)

```python
from toolz import pipe, compose

# pipe: left-to-right execution
result = pipe(
    data,
    normalize,
    validate,
    transform,
)

# compose: right-to-left (mathematical composition)
process = compose(transform, validate, normalize)
result = process(data)
```

---

## Higher-Order Functions

**Functions that take or return functions.**

```python
from typing import Callable

# HOF: takes function as argument
def apply_twice(func: Callable[[int], int], value: int) -> int:
    """Apply function twice."""
    return func(func(value))

def double(n: int) -> int:
    return n * 2

result = apply_twice(double, 5)  # double(double(5)) = 20

# HOF: returns function
def make_multiplier(factor: int) -> Callable[[int], int]:
    """Return function that multiplies by factor."""
    def multiplier(n: int) -> int:
        return n * factor
    return multiplier

times_three = make_multiplier(3)
result = times_three(10)  # 30

# Partial application
from functools import partial

def greet(greeting: str, name: str) -> str:
    return f"{greeting}, {name}!"

say_hello = partial(greet, "Hello")
say_hello("Alice")  # "Hello, Alice!"
```

---

## Lazy Evaluation

**Use generators and iterators for lazy evaluation.**

```python
from typing import Iterator
from itertools import islice, takewhile

# ✅ Generator - lazy evaluation
def fibonacci() -> Iterator[int]:
    """Infinite Fibonacci sequence."""
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

# Take first 10 Fibonacci numbers
first_10 = list(islice(fibonacci(), 10))

# ✅ Generator expression - memory efficient
squares = (x**2 for x in range(1000000))  # No memory allocated yet
first_5 = list(islice(squares, 5))  # Only compute first 5

# ✅ Lazy pipeline
def process_large_file(path: str) -> Iterator[dict]:
    """Process file lazily - never loads entire file."""
    with open(path) as f:
        for line in f:
            yield parse_line(line)

# Only loads and processes lines as needed
for record in islice(process_large_file("data.txt"), 100):
    print(record)
```

---

## Recursion (Use Carefully)

**Recursion is natural for tree structures and divide-and-conquer, but Python has limited stack depth.**

```python
# ✅ Tail recursion (but Python doesn't optimize)
def factorial(n: int, acc: int = 1) -> int:
    """Factorial with accumulator."""
    if n == 0:
        return acc
    return factorial(n - 1, acc * n)

# ✅ Better: Use reduce for factorial
from functools import reduce
from operator import mul

def factorial_reduce(n: int) -> int:
    return reduce(mul, range(1, n + 1), 1)

# ✅ Tree traversal (natural recursion)
from dataclasses import dataclass

@dataclass(frozen=True)
class Node:
    value: int
    left: "Node | None" = None
    right: "Node | None" = None

def sum_tree(node: Node | None) -> int:
    """Sum all values in binary tree."""
    if node is None:
        return 0
    return node.value + sum_tree(node.left) + sum_tree(node.right)

# ❌ Avoid deep recursion (stack overflow)
# Python default recursion limit: 1000
# Use iteration or increase limit with sys.setrecursionlimit (not recommended)
```

---

## Pattern Matching (Python 3.10+)

**Use pattern matching for data destructuring.**

```python
from dataclasses import dataclass
from typing import Literal

@dataclass(frozen=True)
class Point:
    x: float
    y: float

@dataclass(frozen=True)
class Circle:
    center: Point
    radius: float

@dataclass(frozen=True)
class Rectangle:
    top_left: Point
    bottom_right: Point

Shape = Circle | Rectangle

def area(shape: Shape) -> float:
    """Calculate area using pattern matching."""
    match shape:
        case Circle(center=_, radius=r):
            return 3.14159 * r * r
        case Rectangle(
            top_left=Point(x=x1, y=y1),
            bottom_right=Point(x=x2, y=y2)
        ):
            return abs(x2 - x1) * abs(y2 - y1)

# Pattern matching with guards
def classify_point(point: Point) -> Literal["origin", "x_axis", "y_axis", "quadrant"]:
    match point:
        case Point(x=0, y=0):
            return "origin"
        case Point(x=0, y=_):
            return "y_axis"
        case Point(x=_, y=0):
            return "x_axis"
        case _:
            return "quadrant"
```

---

## Naming Conventions

```python
# Variables and functions: snake_case
user_count = 42
is_authenticated = True

def calculate_total(items: list["LineItem"]) -> float:
    ...

# Classes: PascalCase
class UserRepository:
    ...

# Constants: UPPER_SNAKE_CASE
MAX_RETRIES = 3
API_BASE_URL = "https://api.example.com"

# Private members: _prefix
class User:
    def __init__(self, name: str):
        self._name = name

    def _validate(self) -> bool:
        return len(self._name) > 0
```

---

## Type Hints

### Use Type Hints for Public APIs

All public functions must have type annotations.

```python
from typing import Any

def fetch_user(user_id: str) -> dict[str, Any]:
    ...

def process_items(items: list[str]) -> list[int]:
    ...
```

### Prefer Modern Syntax (Python 3.10+)

```python
def process_data(data: dict[str, int]) -> list[str]:
    ...

def get_user(user_id: str) -> User | None:
    ...
```

### Prefer Structured Types

Use TypedDict or dataclasses instead of generic `dict`.

**TypedDict**: For static type checking. Not a runtime validator.

```python
from typing import TypedDict

class UserPayload(TypedDict):
    id: str
    name: str
    email: str

def parse_response(data: dict[str, object]) -> UserPayload:
    payload: UserPayload = {
        "id": str(data["id"]),
        "name": str(data["name"]),
        "email": str(data["email"]),
    }
    return payload
```

**Dataclass**: For domain models with behavior.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class User:
    id: str
    name: str
    email: str
```

### When to Use `Any`

Use `Any` only at system boundaries (external APIs, legacy code). Document the reason.

```python
from typing import Any

def parse_external_api(response: dict[str, Any]) -> User:
    """Parse external API response.

    Uses Any because external schema is not under our control.
    """
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

def set_status(status: Status) -> None:
    ...
```

---

## Error Handling

### Exception Inheritance

Domain exceptions must inherit from `Exception`, not `BaseException`.

```python
class ApplicationError(Exception):
    """Base for all application errors."""
    ...

class ValidationError(ApplicationError):
    """Validation failure."""
    ...
```

### Never Silence Exceptions

```python
import logging

logger = logging.getLogger(__name__)

try:
    risky_operation()
except ValueError:
    logger.exception("Validation error")
    raise
except Exception:
    logger.exception("Unexpected error")
    raise
```

### Preserve Exception Chain

Always use `raise ... from e`.

```python
import json

def parse_config(config_str: str) -> dict:
    try:
        return json.loads(config_str)
    except json.JSONDecodeError as e:
        raise ValueError("Invalid config format") from e
```

### Result Pattern (Functional Error Handling)

For expected domain errors, use Success/Failure pattern. For unexpected errors, use exceptions.

```python
import json
import logging
from dataclasses import dataclass
from typing import Generic, TypeVar

logger = logging.getLogger(__name__)

T = TypeVar("T")
E = TypeVar("E")

@dataclass(frozen=True)
class Success(Generic[T]):
    value: T

@dataclass(frozen=True)
class Failure(Generic[E]):
    error: E

Result = Success[T] | Failure[E]

def parse_config(data: str) -> Result[dict[str, object], str]:
    try:
        parsed = json.loads(data)
        return Success(parsed)
    except json.JSONDecodeError as e:
        return Failure(f"Invalid JSON: {e}")

# Using Result with pattern matching
result = parse_config('{"key": "value"}')
match result:
    case Success(value):
        print(f"Config: {value}")
    case Failure(error):
        logger.error("Parse failed: %s", error)
```

---

## Data Structures

### Use Dataclasses for Domain Models

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

@dataclass(frozen=True)
class Order:
    id: str
    user_id: str
    items: list[OrderItem]
    total: Money
```

### Use Enums for Fixed Sets

```python
from enum import Enum, auto

class Status(Enum):
    PENDING = auto()
    CONFIRMED = auto()
    SHIPPED = auto()

def update_status(order: Order, status: Status) -> None:
    match status:
        case Status.PENDING:
            ...
        case Status.CONFIRMED:
            ...
        case Status.SHIPPED:
            ...
```

---

## Itertools and Functools

**Leverage standard library for functional patterns.**

```python
from itertools import (
    chain,        # Flatten iterables
    groupby,      # Group consecutive items
    islice,       # Slice iterators
    takewhile,    # Take while predicate true
    dropwhile,    # Drop while predicate true
    accumulate,   # Running totals
)
from functools import (
    reduce,       # Fold/aggregate
    partial,      # Partial application
    lru_cache,    # Memoization
)

# Chain: flatten nested lists
nested = [[1, 2], [3, 4], [5, 6]]
flat = list(chain.from_iterable(nested))  # [1, 2, 3, 4, 5, 6]

# GroupBy: group consecutive items
data = [("a", 1), ("a", 2), ("b", 3), ("b", 4)]
for key, group in groupby(data, key=lambda x: x[0]):
    print(key, list(group))

# Partial: create specialized functions
from operator import mul
double = partial(mul, 2)
print(double(5))  # 10

# LRU Cache: memoization for pure functions
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
def search_items(query: str, limit: int = 10) -> list[dict[str, object]]:
    """Search items using semantic similarity.

    Args:
        query: Search query string
        limit: Maximum results (default: 10)

    Returns:
        List of items sorted by relevance

    Raises:
        ValueError: If query is empty
    """
    ...
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
# Constants
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
- pyright for type checking

### Type Checking Configuration

Use pyright strict mode:

```json
{
  "typeCheckingMode": "strict"
}
```

### CI Enforcement

CI must enforce:
- Ruff linting (zero errors)
- Ruff formatting check
- pyright strict mode
- No implicit `Any` in public APIs (except documented boundary cases)

---

## Public API Definition

A module's public API consists of:
- Top-level functions/classes without `_` prefix
- Names in `__all__` if present
- Imports in `__init__.py`

Public API requirements:
- Must have type annotations
- Must have docstrings
- Must validate external inputs
- Should avoid `Any` (boundary exceptions documented)

Private code (`_` prefix):
- May omit docstrings if obvious
- Should still have type hints

---

## Functional Style Summary

### Prefer

✅ Pure functions (no side effects)
✅ Immutable data (`frozen=True`)
✅ map/filter/reduce over explicit loops
✅ List comprehensions for simple transformations
✅ Generator expressions for lazy evaluation
✅ Function composition and pipelines
✅ Pattern matching for data destructuring
✅ Result type for expected errors
✅ Expressions over statements

### Avoid

❌ Global state
❌ Mutable data structures (unless necessary)
❌ `for` loops for pure transformations
❌ Deep recursion (Python stack limit)
❌ Mutating function arguments
❌ Side effects in pure functions
❌ Magic numbers

### Allow (with Justification)

⚠️ `for` loops for side effects (I/O, mutations)
⚠️ Mutable data for performance (after profiling)
⚠️ Recursion for tree structures (shallow depth)
⚠️ `Any` at system boundaries (documented)

---

**Principles** (from The Zen of Python):
- Explicit is better than implicit
- Simple is better than complex
- Readability counts
- **Special cases aren't special enough to break the rules** (use functional style consistently)

**Functional Principles** (from Lisp/FP):
- **Data and functions are separate** (avoid mixing state and behavior)
- **Functions are first-class** (pass them around, return them)
- **Immutability prevents bugs** (no hidden state changes)
- **Composition over inheritance** (build complex behavior from simple functions)
