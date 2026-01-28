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
from decimal import Decimal

# ✅ Pure: deterministic and testable
def calculate_discount(price: Decimal, rate: Decimal) -> Decimal:
    return price * (Decimal("1") - rate)

# ❌ Impure: depends on and modifies global state
_total = Decimal("0")

def add_to_total(amount: Decimal) -> None:
    global _total
    _total += amount
```

**Benefits**: Easier to test, reason about, cache, and parallelize.

**When to use side effects**: I/O operations, database access, logging, external API calls are inherently impure—embrace this at the boundary and infrastructure layers.

### 2. Immutability for Domain Models

Use `frozen=True` and immutable collections for domain models to prevent accidental mutations.

```python
from dataclasses import dataclass, replace
from decimal import Decimal

# ✅ Domain model: immutable with immutable collections
@dataclass(frozen=True)
class Money:
    amount: Decimal
    currency: str

@dataclass(frozen=True)
class User:
    id: str
    email: str
    name: str
    tags: tuple[str, ...]  # Immutable collection

@dataclass(frozen=True)
class Order:
    id: str
    items: tuple["OrderItem", ...]  # Immutable collection
    user_id: str

@dataclass(frozen=True)
class OrderItem:
    product_id: str
    quantity: int
    unit_price: Decimal

# Return new instances using replace()
def update_name(user: User, new_name: str) -> User:
    return replace(user, name=new_name)

def add_tag(user: User, tag: str) -> User:
    return replace(user, tags=(*user.tags, tag))
```

**Immutable collections policy**:
- **Domain models**: Use `tuple` for sequences, `frozenset` for sets
- **DTOs/configs**: Allow `list`/`dict` where mutation is expected (e.g., Pydantic models)
- **Local scope**: Mutable collections OK if never exposed outside function (see note below)
- **Type hints**: Use `Sequence[T]` for read-only parameters, `tuple[T, ...]` for owned data

**Local mutability exception**: Pure functions may use mutable local variables (e.g., `results: list[str] = []`) as long as they're never exposed outside the function. This is a practical trade-off for readability—the function remains pure from the caller's perspective.

```python
from typing import Sequence

# ✅ Pure function with mutable local (not exposed)
def filter_adults(users: Sequence[User]) -> tuple[User, ...]:
    results: list[User] = []  # Local mutable: OK
    for user in users:
        if user.age >= 18:
            results.append(user)
    return tuple(results)  # Converted to immutable before returning

# ✅ Alternative: comprehension (preferred when simple)
def filter_adults(users: Sequence[User]) -> tuple[User, ...]:
    return tuple(user for user in users if user.age >= 18)
```

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
from decimal import Decimal

# ✅ List comprehension (clear and Pythonic)
discounted = [price * Decimal("0.9") for price in prices]
adults = [user for user in users if user.age >= 18]
user_map = {u.id: u.name for u in users}

# ✅ Generator expression (memory efficient)
total = sum(item.price * item.quantity for item in cart.items)
```

### Map/Filter for Higher-Order Functions

Use `map()` and `filter()` when passing functions as arguments or for lazy evaluation.

```python
from operator import attrgetter
from typing import Sequence

# ✅ map with function reference
prices = tuple(map(attrgetter("price"), items))

# ✅ filter with predicate function
def is_active(user: User) -> bool:
    return user.status == "active"

active_users = tuple(filter(is_active, users))

# ❌ Avoid map/filter with lambda (use comprehension instead)
prices = tuple(item.price for item in items)  # Better than map(lambda i: i.price, items)
```

### Reduce for Aggregation (Last Resort)

**Prefer built-in aggregators** (`sum`, `any`, `all`, `min`, `max`). Use `reduce` only when no built-in alternative exists.

```python
from functools import reduce
from operator import or_

# ✅ Built-in aggregators (preferred)
total = sum(item.price for item in items)
has_errors = any(r.is_error for r in results)

# ✅ Prefer built-in methods over reduce
all_permissions: frozenset[str] = frozenset().union(
    *(user.permissions for user in users)
)

# ⚠️ reduce only when necessary (no built-in alternative)
from operator import mul
factorial = reduce(mul, range(1, n + 1), 1)
```

**Why avoid reduce**: Less readable than comprehensions or built-in methods. Use only when you genuinely need custom aggregation logic.

### For Loops: Practical Guidelines

**Principle**: Prefer comprehensions for pure transformations. Use `for` loops when:
- Performing side effects (I/O, logging, API calls)
- Readability is significantly improved
- Complex multi-step logic

```python
from decimal import Decimal
import logging

logger = logging.getLogger(__name__)

# ✅ Side effects: for loop is appropriate
for user in users:
    logger.info("Processing user %s", user.id)
    send_notification(user.email)

# ✅ Comprehension for pure transformation
validated = tuple(validate_user(u) for u in users)

# ⚠️ Complex logic: readability trumps purity
results = []
for item in items:
    if item.price > Decimal("100"):
        discounted = item.price * Decimal("0.9")
        tax = discounted * Decimal("0.1")
        results.append({"price": discounted, "tax": tax})
    else:
        results.append({"price": item.price, "tax": Decimal("0")})

# If this becomes unreadable, extract a function:
def calculate_pricing(item: OrderItem) -> dict[str, Decimal]:
    if item.price > Decimal("100"):
        discounted = item.price * Decimal("0.9")
        return {"price": discounted, "tax": discounted * Decimal("0.1")}
    return {"price": item.price, "tax": Decimal("0")}

results = [calculate_pricing(item) for item in items]
```

**Avoid**: `for` loops that build lists via `.append()` when a comprehension is clearer.

---

## Control Flow & Patterns

### Early Returns

Reduce nesting with early returns for validation and error cases.

```python
from decimal import Decimal

def process_order(order: Order | None) -> dict[str, str]:
    if order is None:
        raise ValueError("Order not found")
    if not order.items:
        raise ValueError("Order has no items")
    if order.total <= Decimal("0"):
        raise ValueError("Invalid order total")

    # Happy path at top level
    return {"id": order.id, "status": "processed"}
```

### Pattern Matching (Python 3.10+)

Use pattern matching for type-based dispatch and data destructuring.

```python
from dataclasses import dataclass
from typing import TypeAlias

@dataclass(frozen=True)
class Success:
    value: str

@dataclass(frozen=True)
class Error:
    message: str

# Specific type alias for this use case
StatusResult: TypeAlias = Success | Error

def handle_result(result: StatusResult) -> str:
    match result:
        case Success(value):
            return f"Success: {value}"
        case Error(message):
            return f"Error: {message}"
```

### Lazy Evaluation with Generators

Use generators for large datasets and infinite sequences.

```python
from collections.abc import Iterator

def read_large_file(path: str) -> Iterator[dict[str, str]]:
    """Process file lazily without loading into memory."""
    with open(path, encoding="utf-8") as f:
        for line in f:
            # In real code, parse_line would be defined
            yield {"line": line.strip()}

# Consume lazily
for record in read_large_file("data.jsonl"):
    process_record(record)

# Or use itertools
from itertools import islice
first_10 = list(islice(read_large_file("data.jsonl"), 10))
```

**Note**: Use `collections.abc.Iterator` (not `typing.Iterator`) for Python 3.11+.

---

## Architecture Layers

Organize code into clear layers to separate pure logic from side effects.

### Layer Dependency Rules

**Critical principle**: Dependencies flow inward (toward domain core).

```
┌─────────────────────────────────────┐
│   API Layer (Boundary)              │  ← HTTP, CLI, external input
│   - Pydantic schemas, validation    │
│   - Adapters (DTO ↔ Domain)         │
│   - Routes, controllers             │
└──────────────┬──────────────────────┘
               │ depends on ↓
┌──────────────▼──────────────────────┐
│   Infrastructure Layer              │  ← Side effects
│   - Repositories (DB access)        │
│   - External API clients            │
│   - File I/O, messaging             │
└──────────────┬──────────────────────┘
               │ depends on ↓
┌──────────────▼──────────────────────┐
│   Domain Layer (Core)               │  ← Pure business logic
│   - Domain models (frozen)          │  ✅ NO imports from outer layers
│   - Domain services (pure functions)│  ✅ NO I/O, NO logging, NO side effects
│   - Business rules                  │  ✅ Framework-agnostic
└─────────────────────────────────────┘
```

**Prohibited**: Domain importing from Infrastructure or API layers.

### Domain Layer (Pure Business Logic)

Pure functions and immutable domain models. **No I/O, no logging, no framework dependencies.**

```python
# domain/models.py
from dataclasses import dataclass, replace
from decimal import Decimal

@dataclass(frozen=True)
class OrderItem:
    product_id: str
    quantity: int
    unit_price: Decimal  # Always Decimal for money

    @property
    def subtotal(self) -> Decimal:
        return self.unit_price * self.quantity

@dataclass(frozen=True)
class Order:
    id: str
    items: tuple[OrderItem, ...]  # Immutable collection

    @property
    def total(self) -> Decimal:
        return sum((item.subtotal for item in self.items), start=Decimal("0"))

# domain/services.py
def apply_discount(order: Order, rate: Decimal) -> Order:
    """Pure function: returns new order with discounted items.

    No side effects, no logging, no I/O.
    """
    discounted_items = tuple(
        replace(item, unit_price=item.unit_price * (Decimal("1") - rate))
        for item in order.items
    )
    return replace(order, items=discounted_items)

def validate_order(order: Order) -> tuple[Order, tuple[str, ...]]:
    """Pure validation: returns order and error messages.

    Note: Local mutable (errors) is OK—never exposed outside function.
    """
    errors: list[str] = []  # Local mutable: allowed

    if not order.items:
        errors.append("Order must have at least one item")

    if order.total <= Decimal("0"):
        errors.append("Order total must be positive")

    return order, tuple(errors)  # Converted to immutable
```

**Domain layer rules**:
- ✅ Pure functions only
- ✅ Immutable models (`frozen=True`, `tuple`, `frozenset`)
- ✅ Use `Decimal` for all monetary values
- ✅ Local mutable variables OK (if never exposed)
- ❌ No logging (not even logger.debug)
- ❌ No I/O (files, DB, network)
- ❌ No framework dependencies (FastAPI, Django, etc.)
- ❌ No imports from Infrastructure or API layers

### Infrastructure Layer (Side Effects)

Database access, external APIs, file I/O. Implements interfaces defined by domain using Protocol.

```python
# infrastructure/repositories.py
from typing import Protocol
from domain.models import Order

# Protocol for dependency inversion
class OrderRepository(Protocol):
    """Protocol defines interface (structural subtyping).

    Benefits:
    - Domain doesn't depend on implementation details
    - Easy to mock for testing
    - Swap implementations without changing domain
    """
    def get(self, order_id: str) -> Order | None: ...
    def save(self, order: Order) -> None: ...

# Concrete implementation (example sketch)
class PostgresOrderRepository:
    """Infrastructure implementation with side effects."""

    def __init__(self, connection_string: str):
        # In production, use connection pool
        self._conn_string = connection_string

    def get(self, order_id: str) -> Order | None:
        # Pseudo-code: Database access (side effect)
        # row = execute_query("SELECT * FROM orders WHERE id = %s", [order_id])
        # if row is None:
        #     return None
        # return self._parse_order(row)
        ...

    def save(self, order: Order) -> None:
        # Pseudo-code: Database mutation (side effect)
        # execute_query(
        #     "INSERT INTO orders (id, items, total) VALUES (%s, %s, %s)",
        #     [order.id, serialize(order.items), str(order.total)]
        # )
        ...
```

**Note**: Infrastructure code examples are shown as pseudo-code or minimal sketches to focus on architecture. In production, add proper error handling, transactions, and connection management.

### Boundary Layer (Validation & Adaptation)

Validate external input, adapt between external formats and domain models.

```python
# api/schemas.py (Pydantic for validation)
from pydantic import BaseModel, Field

class OrderItemInput(BaseModel):
    """DTO: mutable collections OK here."""
    product_id: str = Field(min_length=1)
    quantity: int = Field(gt=0)
    unit_price: float = Field(gt=0)  # float from external API

class CreateOrderRequest(BaseModel):
    """Boundary validates external input."""
    items: list[OrderItemInput]

    model_config = {"frozen": True}

# api/adapters.py
from decimal import Decimal
from domain.models import Order, OrderItem

def to_domain_order(req: CreateOrderRequest, order_id: str) -> Order:
    """Adapt validated DTO to domain model.

    Key responsibility: Convert float → Decimal at boundary.
    """
    return Order(
        id=order_id,
        items=tuple(
            OrderItem(
                product_id=item.product_id,
                quantity=item.quantity,
                unit_price=Decimal(str(item.unit_price)),  # float → Decimal
            )
            for item in req.items
        ),
    )

def from_domain_order(order: Order) -> dict[str, object]:
    """Adapt domain model to API response.

    Float vs String: Use string for precision-critical APIs, float for compatibility.
    """
    return {
        "id": order.id,
        "items": [
            {
                "product_id": item.product_id,
                "quantity": item.quantity,
                # Option A: string (recommended for precision)
                "unit_price": str(item.unit_price),
                # Option B: float (for external API compatibility)
                # "unit_price": float(item.unit_price),
            }
            for item in order.items
        ],
        "total": str(order.total),  # Prefer string for money
    }
```

**Decimal vs float policy**:
- **Domain**: Always `Decimal` for monetary values (precision, no rounding errors)
- **Boundary (input)**: Convert `float` → `Decimal` using `Decimal(str(value))`
- **Boundary (output)**:
  - **Prefer string** (`str(decimal_value)`) for precision-critical APIs
  - **Use float** (`float(decimal_value)`) only for external API compatibility
- **Infrastructure**: Store as `NUMERIC`/`DECIMAL` in DB, convert to `Decimal` when loading
- **Never**: Mix `Decimal` and `float` in domain calculations

---

## Type System

### Use Strict Type Checking

Configure `pyright` in strict mode to catch errors at development time.

```json
// pyrightconfig.json
{
  "typeCheckingMode": "strict",
  "pythonVersion": "3.11",
  "reportMissingTypeStubs": false,
  "exclude": ["**/node_modules", "**/__pycache__", ".venv"]
}
```

### Type All Public APIs

```python
from typing import Sequence

def fetch_user(user_id: str) -> User | None:
    """Fetch user by ID. Returns None if not found."""
    ...

def process_items(items: Sequence[str]) -> tuple[int, ...]:
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
    """Uses Any because external API schema is not under our control.

    Validate and convert to typed domain model immediately.
    """
    return User(
        id=str(response["id"]),
        email=str(response["email"]),
        name=str(response["name"]),
    )
```

### Use TypeAlias for Complex Types

```python
from typing import TypeAlias, Literal

# TypeAlias for clarity (define per use case)
Status: TypeAlias = Literal["pending", "active", "archived"]
UserId: TypeAlias = str
Email: TypeAlias = str

def send_notification(user_id: UserId, email: Email, status: Status) -> None:
    ...
```

### Use Enums for Rich Semantics

```python
from enum import Enum

class OrderStatus(Enum):
    """Enum provides methods and type safety."""
    PENDING = "pending"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"
    CANCELLED = "cancelled"

    @property
    def is_final(self) -> bool:
        return self in {OrderStatus.SHIPPED, OrderStatus.CANCELLED}

    @property
    def can_modify(self) -> bool:
        return self in {OrderStatus.PENDING, OrderStatus.CONFIRMED}
```

### Protocol for Structural Subtyping

**Why Protocol**: Enables structural subtyping (duck typing with type safety).

```python
from typing import Protocol

class EmailSender(Protocol):
    """Protocol: any class with send() method satisfies this interface.

    Benefits:
    - No inheritance required (structural subtyping)
    - Domain doesn't depend on infrastructure
    - Easy to test (simple mocks)
    - Swap implementations freely
    """
    def send(self, to: str, subject: str, body: str) -> None: ...

# Any class matching this structure works:
class SMTPEmailSender:
    def send(self, to: str, subject: str, body: str) -> None:
        # Implementation with side effects
        ...

class MockEmailSender:
    def send(self, to: str, subject: str, body: str) -> None:
        # Test implementation
        print(f"Mock: sending to {to}")

# Both satisfy EmailSender protocol without inheritance
def send_welcome_email(sender: EmailSender, user: User) -> None:
    sender.send(user.email, "Welcome", "Welcome to our service!")
```

---

## Error Handling

### None vs Exceptions

**Use `None` return** (Optional):
- Expected absence (user not found is normal)
- Query that may return no results
- Optional configuration values

**Use exceptions**:
- Exceptional conditions (network error, invalid state)
- Validation failures
- Programming errors

```python
from decimal import Decimal

# ✅ None: expected absence
def find_user_by_email(email: str) -> User | None:
    """Returns None if user not found (normal case)."""
    ...

# ✅ Exception: invalid state
def charge_order(order: Order) -> dict[str, str]:
    """Raises ValueError if order total is zero."""
    if order.total <= Decimal("0"):
        raise ValueError("Cannot charge order with zero total")
    return {"status": "charged"}

# ✅ Exception: validation failure
def create_user(email: str, name: str) -> User:
    """Raises ValidationError if inputs are invalid."""
    if not email or "@" not in email:
        raise ValidationError("Invalid email format")
    return User(id="u123", email=email, name=name)

class ValidationError(Exception):
    """Input validation failure."""
```

**Guidelines**:
- **NotFound**: Return `None` (caller decides how to handle)
- **Invalid input**: Raise `ValidationError`
- **Invalid state**: Raise `ValueError` or custom exception
- **External failure**: Raise specific exception (e.g., `DatabaseError`, `APIError`)

### Exception Hierarchy

```python
class ApplicationError(Exception):
    """Base for all application errors."""

class ValidationError(ApplicationError):
    """Input validation failure."""

class NotFoundError(ApplicationError):
    """Resource not found (use when exception is preferred over None)."""

class AuthorizationError(ApplicationError):
    """User not authorized for action."""

class ExternalServiceError(ApplicationError):
    """External service (DB, API) failure."""
```

### Preserve Exception Context

```python
import json

def load_config(config_str: str) -> dict[str, object]:
    try:
        return json.loads(config_str)
    except json.JSONDecodeError as e:
        # Preserve original exception for debugging
        raise ValueError(f"Invalid config format: {config_str[:50]}...") from e
```

### Result Pattern (Optional)

Use when you want to make failure explicit in return types (library code, validation pipelines).

**Important**: Define Result types per use case (not as a generic TypeAlias) for type safety.

```python
from dataclasses import dataclass
from typing import TypeAlias

@dataclass(frozen=True)
class Success[T]:
    """Generic success wrapper."""
    value: T

@dataclass(frozen=True)
class Failure[E]:
    """Generic failure wrapper."""
    error: E

# ✅ Define Result per use case (type-safe)
ParseAgeResult: TypeAlias = Success[int] | Failure[str]
ValidateEmailResult: TypeAlias = Success[str] | Failure[str]

# Library function: returns Result instead of raising
def parse_age(value: str) -> ParseAgeResult:
    """Parse age from string. Returns Result instead of raising.

    Use Result pattern when:
    - Building libraries that shouldn't raise
    - Railway-oriented validation pipelines
    - Making errors explicit in type signatures
    """
    try:
        age = int(value)
        if age < 0 or age > 150:
            return Failure("Age out of valid range (0-150)")
        return Success(age)
    except ValueError:
        return Failure(f"Invalid integer: {value}")

# Consumer uses pattern matching
def process_age_input(input_str: str) -> None:
    match parse_age(input_str):
        case Success(age):
            print(f"Valid age: {age}")
        case Failure(error):
            print(f"Error: {error}")
```

**Why per-use-case TypeAlias**: Prevents type inference issues with generic `Result[T, E]` when copying code across modules. Each function gets its own concrete Result type.

---

## Code Organization

### Naming Conventions

```python
from typing import Sequence, TypeAlias
from decimal import Decimal

# Variables and functions: snake_case
user_count = 42
def calculate_total(items: Sequence[OrderItem]) -> Decimal: ...

# Classes: PascalCase
class UserRepository: ...

# Constants: UPPER_SNAKE_CASE
MAX_RETRIES = 3
DEFAULT_TIMEOUT = 30

# Type aliases: PascalCase
UserId: TypeAlias = str

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
├── domain/              # Pure business logic (no dependencies on outer layers)
│   ├── models.py       # Domain models (frozen dataclasses, tuple)
│   ├── services.py     # Pure business functions
│   └── errors.py       # Domain exceptions
├── infrastructure/      # Side effects (DB, APIs, file I/O)
│   ├── repositories.py # Repository implementations
│   ├── external_api.py # External API clients
│   └── database.py     # DB connection, migrations
├── api/                 # Boundary layer (validation, adaptation)
│   ├── schemas.py      # Pydantic models (DTOs)
│   ├── adapters.py     # Domain ↔ API conversion
│   └── routes.py       # FastAPI routes
└── shared/              # Shared utilities
    ├── types.py        # Common type aliases
    └── config.py       # Configuration
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
    "S",   # flake8-bandit (security)
]
```

### Documentation

Use Google-style docstrings for public APIs:

```python
def search_users(
    query: str,
    limit: int = 10,
    offset: int = 0,
) -> tuple[User, ...]:
    """Search users by query string.

    Args:
        query: Search query (name or email)
        limit: Maximum results (default: 10)
        offset: Pagination offset (default: 0)

    Returns:
        Tuple of users matching query, sorted by relevance

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
- **Pure functions** for domain logic (no side effects, no logging)
- **Immutable domain models** (`frozen=True`, `tuple`, `frozenset`)
- **`Decimal`** for monetary values in domain
- **String output** for Decimal in APIs (precision)
- **Type annotations** on all public APIs
- **Early returns** to reduce nesting
- **Generator expressions** for large datasets
- **Pattern matching** for type dispatch
- **Protocol** for dependency inversion
- **None** for expected absence, **exceptions** for errors
- **`collections.abc.Iterator`** for Python 3.11+

### Prefer ✅

- **Comprehensions** over map/filter with lambda
- **Built-in methods** (`frozenset().union()`) over `reduce`
- **Built-in aggregators** (`sum`, `any`, `all`) over custom loops
- **Specific types** over `Any`
- **Dataclasses** over plain dicts
- **Enums** over string constants
- **`tuple`** over `list` for domain collections
- **`Sequence[T]`** for read-only parameters
- **Per-use-case Result types** over generic `Result[T, E]`

### Avoid ❌

- **Global mutable state**
- **`for` loops that build lists** via `.append()` (use comprehensions)
- **Mutating function arguments**
- **`Any` in internal code** (OK at system boundaries with docs)
- **Deep recursion** (Python stack limit ~1000)
- **Magic numbers** (use named constants)
- **Domain importing Infrastructure/API**
- **Side effects in domain layer** (logging, I/O)
- **Mixing `Decimal` and `float`** in calculations
- **Mutable collections in domain models** (`list`, `dict`)
- **`reduce` when built-in alternatives exist**
- **Float output for monetary values** (prefer string)

### Context-Dependent ⚠️

- **`for` loops**: OK for side effects, complex multi-step logic, or when clearer
- **Local mutable collections**: OK in pure functions (if never exposed)
- **Result pattern**: Use in library code or validation pipelines
- **`NotFoundError`**: Use when exception is semantically better than `None`
- **Float in API responses**: Only for external API compatibility (document why)

---

## Decimal Guidelines

**Critical for financial applications**: Use `Decimal` consistently to avoid rounding errors.

```python
from dataclasses import dataclass
from decimal import Decimal
from pydantic import BaseModel

# ✅ Domain: Always Decimal
@dataclass(frozen=True)
class OrderItem:
    unit_price: Decimal  # Not float

    def apply_tax(self, rate: Decimal) -> Decimal:
        return self.unit_price * rate

# ✅ Boundary (input): Convert at edges
class OrderItemInput(BaseModel):
    unit_price: float  # External API uses float

def to_domain_item(dto: OrderItemInput) -> OrderItem:
    return OrderItem(
        unit_price=Decimal(str(dto.unit_price))  # float → Decimal
    )

# ✅ Boundary (output): Prefer string for precision
def to_api_response(item: OrderItem) -> dict[str, object]:
    return {
        "unit_price": str(item.unit_price),  # Decimal → string (recommended)
        # Alternative: float(item.unit_price) only for external compatibility
    }

# ✅ Literals: Use string for exact precision
discount_rate = Decimal("0.9")  # Exact
tax_rate = Decimal("0.1")       # Exact

# ❌ Don't use float literals
discount_rate = Decimal(0.9)  # Imprecise (binary float representation)
```

**Decimal rules**:
1. **Domain**: Always `Decimal` for money
2. **Input conversion**: `Decimal(str(float_value))`
3. **Literal creation**: `Decimal("0.1")` (string)
4. **Output format**:
   - **Prefer string**: `str(decimal_value)` for precision
   - **Use float**: `float(decimal_value)` only for external API compatibility
5. **Infrastructure**: Store as `NUMERIC`/`DECIMAL`, convert to `Decimal` on load

---

**Core Philosophy**:

Write code that is **clear**, **correct**, and **maintainable**. Use functional principles (pure functions, immutability, composition) to achieve this, but prioritize **Python idioms** and **readability**. Let **pyright strict** guide you toward type safety. Keep side effects at the boundaries. Test domain logic thoroughly (it's pure, so it's easy).

**From The Zen of Python**:
- Explicit is better than implicit
- Simple is better than complex
- Readability counts

**From Functional Programming**:
- Pure functions are easier to test and reason about
- Immutability prevents bugs
- Composition creates reusable building blocks

**From Domain-Driven Design**:
- Keep the domain pure and framework-agnostic
- Dependencies point inward
- Side effects belong to infrastructure
