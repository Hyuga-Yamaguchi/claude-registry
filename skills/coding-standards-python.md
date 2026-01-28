---
name: coding-standards-python
description: Production-ready Python standards with functional programming principles. Emphasizes comprehensions, immutability, pure functions, and type safety. For Python 3.11+ with pyright strict.
---

# Python Coding Standards

Production-ready standards for Python 3.11+ with functional programming principles. Backend patterns in backend-patterns-fastapi.md. Formatting handled by Ruff.

**Philosophy**: Write clear, testable, type-safe code. Favor pure functions and immutability where practical. Prefer comprehensions over loops. Let pyright strict guide correctness.

---

## Core Principles

### 1. Pure Functions When Possible

**Pure Function**: Same input → same output, no side effects.

```python
# ✅ Pure: deterministic and testable
def calculate_discount(price: float, rate: float) -> float:
    return price * (1 - rate)

# ❌ Impure: depends on and modifies global state
_total = 0.0

def add_to_total(amount: float) -> None:
    global _total
    _total += amount
```

**Benefits**: Easier to test, reason about, cache, and parallelize.

**When to use side effects**: I/O operations, database access, logging, external API calls are inherently impure—embrace this at the boundary layer.

### 2. Immutability for Domain Models

Use `frozen=True` for domain models to prevent accidental mutations.

```python
from dataclasses import dataclass
from decimal import Decimal

# ✅ Domain model: immutable
@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str

@dataclass(frozen=True)
class User:
    id: str
    email: str
    name: str

# Return new instances instead of mutating
def update_name(user: User, new_name: str) -> User:
    return dataclass.replace(user, name=new_name)
```

**When to allow mutability**:
- DTOs/configs where mutation is expected (e.g., Pydantic models)
- Performance-critical code (after profiling)
- Local variables never exposed outside function scope

### 3. Expressions Over Statements

```python
# ✅ Expression-oriented
status = "active" if user.verified else "pending"

# ❌ Statement-oriented
if user.verified:
    status = "active"
else:
    status = "pending"
```

Use ternary expressions, comprehensions, and pattern matching to minimize mutable state.

---

## Data Transformation

### Comprehensions First (Pythonic)

**Default choice**: Use comprehensions for transformations and filtering.

```python
# ✅ List comprehension (clear and Pythonic)
discounted = [price * 0.9 for price in prices]
adults = [user for user in users if user.age >= 18]
user_map = {u.id: u.name for u in users}

# ✅ Generator expression (memory efficient)
total = sum(item.price * item.quantity for item in cart.items)
```

### Map/Filter for Higher-Order Functions

Use `map()` and `filter()` when passing functions as arguments or for lazy evaluation.

```python
from operator import attrgetter

# ✅ map with function reference
prices = list(map(attrgetter("price"), items))

# ✅ filter with predicate function
def is_active(user: User) -> bool:
    return user.status == "active"

active_users = list(filter(is_active, users))

# ❌ Avoid map/filter with lambda (use comprehension)
prices = [item.price for item in items]  # Better than map(lambda i: i.price, items)
```

### Reduce for Aggregation

Prefer built-in aggregators (`sum`, `any`, `all`, `min`, `max`). Use `reduce` for custom aggregation.

```python
from functools import reduce
from operator import or_

# ✅ Built-in aggregators
total = sum(item.price for item in items)
has_errors = any(r.is_error for r in results)

# ✅ reduce for custom aggregation
permissions = reduce(or_, (user.permissions for user in users), set())
```

### For Loops: Practical Guidelines

**Principle**: Prefer comprehensions for pure transformations. Use `for` loops when:
- Performing side effects (I/O, logging, API calls)
- Readability is significantly improved
- Complex multi-step logic

```python
# ✅ Side effects: for loop is appropriate
for user in users:
    logger.info("Processing user %s", user.id)
    send_email(user.email, template)

# ✅ Comprehension for pure transformation
validated = [validate_user(u) for u in users]

# ⚠️ Complex logic: readability trumps purity
results = []
for item in items:
    if item.price > 100:
        discounted = item.price * 0.9
        tax = discounted * 0.1
        results.append({"price": discounted, "tax": tax})
    else:
        results.append({"price": item.price, "tax": 0})

# If this becomes unreadable, extract a function:
def calculate_pricing(item: Item) -> dict:
    if item.price > 100:
        discounted = item.price * 0.9
        return {"price": discounted, "tax": discounted * 0.1}
    return {"price": item.price, "tax": 0}

results = [calculate_pricing(item) for item in items]
```

**Avoid**: `for` loops that build lists via `.append()` when a comprehension is clearer.

---

## Control Flow & Patterns

### Early Returns

Reduce nesting with early returns for validation and error cases.

```python
def process_order(order: Order | None) -> dict:
    if order is None:
        raise ValueError("Order not found")
    if not order.items:
        raise ValueError("Order has no items")
    if order.total <= 0:
        raise ValueError("Invalid order total")

    # Happy path at top level
    return {"id": order.id, "total": order.total}
```

### Pattern Matching (Python 3.10+)

Use pattern matching for type-based dispatch and data destructuring.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Success:
    value: str

@dataclass(frozen=True)
class Error:
    message: str

Result = Success | Error

def handle_result(result: Result) -> str:
    match result:
        case Success(value):
            return f"Success: {value}"
        case Error(message):
            return f"Error: {message}"
```

### Lazy Evaluation with Generators

Use generators for large datasets and infinite sequences.

```python
from typing import Iterator

def read_large_file(path: str) -> Iterator[dict]:
    """Process file lazily without loading into memory."""
    with open(path) as f:
        for line in f:
            yield parse_line(line)

# Consume lazily
for record in read_large_file("data.jsonl"):
    process(record)

# Or use itertools
from itertools import islice
first_10 = list(islice(read_large_file("data.jsonl"), 10))
```

---

## Architecture Layers

Organize code into clear layers to separate pure logic from side effects.

### Domain Layer (Pure Business Logic)

Pure functions and immutable domain models. No I/O, no framework dependencies.

```python
# domain/models.py
from dataclasses import dataclass
from decimal import Decimal

@dataclass(frozen=True)
class Order:
    id: str
    items: list["OrderItem"]

    @property
    def total(self) -> Decimal:
        return sum(item.subtotal for item in self.items)

@dataclass(frozen=True)
class OrderItem:
    product_id: str
    quantity: int
    unit_price: Decimal

    @property
    def subtotal(self) -> Decimal:
        return self.unit_price * self.quantity

# domain/services.py
def apply_discount(order: Order, rate: Decimal) -> Order:
    """Pure function: returns new order with discounted items."""
    discounted_items = [
        dataclass.replace(item, unit_price=item.unit_price * (1 - rate))
        for item in order.items
    ]
    return dataclass.replace(order, items=discounted_items)
```

### Infrastructure Layer (Side Effects)

Database access, external APIs, file I/O. Implements interfaces defined by domain.

```python
# infrastructure/repositories.py
from typing import Protocol

class OrderRepository(Protocol):
    def get(self, order_id: str) -> Order | None: ...
    def save(self, order: Order) -> None: ...

class PostgresOrderRepository:
    def __init__(self, conn: Connection):
        self._conn = conn

    def get(self, order_id: str) -> Order | None:
        # Database access (side effect)
        row = self._conn.execute("SELECT * FROM orders WHERE id = ?", [order_id]).fetchone()
        return parse_order(row) if row else None

    def save(self, order: Order) -> None:
        # Database mutation (side effect)
        self._conn.execute("INSERT INTO orders ...", order_to_dict(order))
```

### Boundary Layer (Validation & Adaptation)

Validate external input, adapt between external formats and domain models.

```python
# api/schemas.py (Pydantic for validation)
from pydantic import BaseModel, Field

class CreateOrderRequest(BaseModel):
    items: list[OrderItemInput]

    class Config:
        frozen = True  # Immutable after validation

class OrderItemInput(BaseModel):
    product_id: str = Field(min_length=1)
    quantity: int = Field(gt=0)
    unit_price: float = Field(gt=0)

# api/adapters.py
def to_domain_order(req: CreateOrderRequest, order_id: str) -> Order:
    """Adapt validated input to domain model."""
    return Order(
        id=order_id,
        items=[
            OrderItem(
                product_id=item.product_id,
                quantity=item.quantity,
                unit_price=Decimal(str(item.unit_price)),
            )
            for item in req.items
        ],
    )
```

---

## Type System

### Use Strict Type Checking

Configure `pyright` in strict mode to catch errors at development time.

```json
// pyrightconfig.json
{
  "typeCheckingMode": "strict",
  "reportMissingTypeStubs": false
}
```

### Type All Public APIs

```python
def fetch_user(user_id: str) -> User | None:
    """Fetch user by ID. Returns None if not found."""
    ...

def process_items(items: list[str]) -> list[int]:
    """Convert string items to integers."""
    ...
```

### Prefer Specific Types Over `Any`

```python
from typing import Any

# ❌ Avoid Any in internal code
def process(data: Any) -> Any:
    ...

# ✅ Use specific types
def process(data: dict[str, int]) -> list[int]:
    ...

# ✅ OK: System boundaries (document why)
def parse_external_api(response: dict[str, Any]) -> User:
    """Uses Any because external API schema is not under our control."""
    return User(
        id=str(response["id"]),
        email=str(response["email"]),
        name=str(response["name"]),
    )
```

### Use Literal and Enums

```python
from typing import Literal
from enum import Enum

# Literal for simple fixed values
Status = Literal["pending", "active", "archived"]

# Enum for richer semantics
class OrderStatus(Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"

    @property
    def is_final(self) -> bool:
        return self in {OrderStatus.SHIPPED}
```

### Leverage TypedDict for Structured Dicts

```python
from typing import TypedDict

class UserDict(TypedDict):
    id: str
    email: str
    name: str

def format_user(user: UserDict) -> str:
    return f"{user['name']} <{user['email']}>"
```

---

## Error Handling

### Exceptions vs Result Pattern

**Use exceptions** (default):
- Exceptional conditions (file not found, network error, invalid state)
- Framework integration (FastAPI, Django)
- Application-level errors

**Use Result pattern** (optional):
- Library code that shouldn't raise exceptions
- Railway-oriented programming for validation pipelines
- When you want to make errors explicit in type signatures

### Exception Hierarchy

```python
class ApplicationError(Exception):
    """Base for all application errors."""

class ValidationError(ApplicationError):
    """Input validation failure."""

class NotFoundError(ApplicationError):
    """Resource not found."""

class AuthorizationError(ApplicationError):
    """User not authorized."""
```

### Preserve Exception Context

```python
import json

try:
    config = json.loads(config_str)
except json.JSONDecodeError as e:
    # Preserve original exception for debugging
    raise ValueError(f"Invalid config format: {config_str}") from e
```

### Result Pattern (Optional)

Use when you want to make failure explicit in return types.

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

# Library function: returns Result instead of raising
def parse_age(value: str) -> Result[int, str]:
    try:
        age = int(value)
        if age < 0 or age > 150:
            return Failure("Age out of valid range")
        return Success(age)
    except ValueError:
        return Failure(f"Invalid integer: {value}")

# Consumer uses pattern matching
match parse_age(input_str):
    case Success(age):
        print(f"Age: {age}")
    case Failure(error):
        print(f"Error: {error}")
```

---

## Code Organization

### Naming Conventions

```python
# Variables and functions: snake_case
user_count = 42
def calculate_total(items: list[Item]) -> Decimal: ...

# Classes: PascalCase
class UserRepository: ...

# Constants: UPPER_SNAKE_CASE
MAX_RETRIES = 3
DEFAULT_TIMEOUT = 30

# Private: leading underscore
class User:
    def __init__(self, name: str):
        self._name = name

    def _internal_helper(self) -> str:
        return self._name.upper()
```

### Module Organization

```
project/
├── domain/              # Pure business logic
│   ├── models.py       # Domain models (frozen dataclasses)
│   └── services.py     # Pure business functions
├── infrastructure/      # Side effects (DB, APIs)
│   ├── repositories.py
│   └── external_api.py
├── api/                 # Boundary layer
│   ├── schemas.py      # Pydantic models
│   ├── adapters.py     # Domain ↔ API conversion
│   └── routes.py       # FastAPI routes
└── shared/              # Shared utilities
    ├── types.py        # Common types
    └── errors.py       # Exception hierarchy
```

### Public API Definition

A module's public API consists of:
- Top-level functions/classes without `_` prefix
- Names in `__all__` if present
- Exports in `__init__.py`

```python
# my_module.py
def public_function() -> str:
    """Public: exported."""
    return _internal_helper()

def _internal_helper() -> str:
    """Private: not exported."""
    return "internal"

__all__ = ["public_function"]
```

**Public API requirements**:
- Complete type annotations
- Docstrings (Google or NumPy style)
- Input validation at boundaries
- Avoid `Any` (document exceptions)

---

## Tooling & Quality

### Required Tools

- **Python 3.11+**: Modern syntax (match, union types, dataclass improvements)
- **Ruff**: Linting and formatting (replaces Black, isort, flake8)
- **pyright**: Type checking in strict mode

### Ruff Configuration

```toml
# pyproject.toml
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = [
    "E",   # pycodestyle errors
    "F",   # pyflakes
    "I",   # isort
    "N",   # pep8-naming
    "UP",  # pyupgrade
    "B",   # flake8-bugbear
]
```

### Pyright Configuration

```json
// pyrightconfig.json
{
  "typeCheckingMode": "strict",
  "pythonVersion": "3.11",
  "exclude": ["**/node_modules", "**/__pycache__", ".venv"]
}
```

### Documentation

Use Google-style docstrings for public APIs:

```python
def search_users(
    query: str,
    limit: int = 10,
    offset: int = 0,
) -> list[User]:
    """Search users by query string.

    Args:
        query: Search query (name or email)
        limit: Maximum results (default: 10)
        offset: Pagination offset (default: 0)

    Returns:
        List of users matching query, sorted by relevance

    Raises:
        ValueError: If query is empty or limit is negative
    """
```

### CI Enforcement

```yaml
# .github/workflows/ci.yml
- run: ruff check .
- run: ruff format --check .
- run: pyright
```

---

## Best Practices Summary

### Always ✅

- **Comprehensions** for data transformations
- **Pure functions** for business logic (domain layer)
- **Immutable domain models** (`frozen=True`)
- **Type annotations** on all public APIs
- **Early returns** to reduce nesting
- **Generator expressions** for large datasets
- **Pattern matching** for type dispatch
- **Exceptions** for error handling (default)

### Prefer ✅

- **Comprehensions** over map/filter with lambda
- **Built-in aggregators** (`sum`, `any`, `all`) over `reduce`
- **Specific types** over `Any`
- **Dataclasses** over plain dicts for structured data
- **Enums/Literal** over string constants

### Avoid ❌

- **Global mutable state**
- **`for` loops that build lists** via `.append()` (use comprehensions)
- **Mutating function arguments**
- **`Any` in internal code** (OK at system boundaries with docs)
- **Deep recursion** (Python stack limit ~1000)
- **Magic numbers** (use named constants)

### Context-Dependent ⚠️

- **`for` loops**: OK for side effects, complex multi-step logic, or when clearer than comprehensions
- **Mutable data**: OK for performance (after profiling), DTOs, local scope
- **Result pattern**: Use in library code or validation pipelines
- **map/filter**: Use with function references or lazy evaluation

---

**Core Philosophy**:

Write code that is **clear**, **correct**, and **maintainable**. Use functional principles (pure functions, immutability, composition) to achieve this, but prioritize **Python idioms** and **readability**. Let **pyright strict** guide you toward type safety. Test business logic thoroughly. Keep side effects at the boundaries.

**From The Zen of Python**:
- Explicit is better than implicit
- Simple is better than complex
- Readability counts

**From Functional Programming**:
- Pure functions are easier to test and reason about
- Immutability prevents bugs
- Composition creates reusable building blocks
