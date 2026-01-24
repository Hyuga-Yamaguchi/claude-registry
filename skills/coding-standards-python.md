---
name: coding-standards-python
description: Python language-level coding standards and best practices for Python 3.11+. Formatting handled by Ruff.
---

# Python Coding Standards

Language-level standards for Python 3.11+. Backend patterns in backend-patterns-fastapi.md. Formatting handled by Ruff formatter.

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

**Boundary exception**: Public APIs at system boundaries may use `Any` with documentation. Internal public APIs should avoid `Any`.

### Use Literal for Fixed Values

```python
from typing import Literal

Status = Literal["active", "inactive", "archived"]

def set_status(status: Status) -> None:
    ...
```

## Error Handling

### Exception Inheritance

Domain exceptions must inherit from `Exception`, not `BaseException`.

**Rationale**: `BaseException` is reserved for system-level exceptions (KeyboardInterrupt, SystemExit).

```python
class ApplicationError(Exception):
    """Base for all application errors."""
    ...

class ValidationError(ApplicationError):
    """Validation failure."""
    ...

class MaxRetriesExceeded(ApplicationError):
    """Maximum retry attempts exceeded."""
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

### Result Pattern (Optional)

For expected domain errors, consider Success/Failure pattern when explicit error modeling improves readability. Use exceptions for unexpected errors.

**When to use Result**:
- Validation with multiple failure modes
- Parsing with expected failures
- Domain operations with business rule violations

**When to use exceptions**:
- Unexpected errors (network failures, programming errors)
- Framework/library integration (most Python code uses exceptions)

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

def parse_config(data: str) -> Success[dict[str, object]] | Failure[str]:
    try:
        parsed = json.loads(data)
        return Success(parsed)
    except json.JSONDecodeError as e:
        return Failure(f"Invalid JSON: {e}")

def use_config(config: dict[str, object]) -> None:
    """Process validated configuration."""
    ...

config_string = '{"key": "value"}'
result = parse_config(config_string)
match result:
    case Success(value):
        use_config(value)
    case Failure(error):
        logger.error("Config parse failed: %s", error)
```

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

### Prefer Immutable Data

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Point:
    x: float
    y: float

@dataclass
class Counter:
    count: int = 0

    def increment(self) -> None:
        self.count += 1
```

## Pythonic Idioms

```python
from typing import Iterator

# Comprehensions
squares = [x**2 for x in range(10)]
user_map = {user.id: user.name for user in users}

# Generators for large data
def read_large_file(path: str) -> Iterator[str]:
    with open(path) as f:
        for line in f:
            yield line.strip()

# Built-in functions
for index, item in enumerate(items):
    print(f"{index}: {item}")

name = user.get("name", "Anonymous")
has_admin = any(user.role == "admin" for user in users)
all_active = all(user.active for user in users)
```

## Functions

### Single Responsibility

```python
from typing import TypedDict

class LineItem(TypedDict):
    price: float
    quantity: int

def calculate_total(items: list[LineItem]) -> float:
    return sum(item["price"] * item["quantity"] for item in items)

def validate_email(email: str) -> bool:
    """Basic email validation. Not RFC-compliant."""
    local, sep, domain = email.partition("@")
    return bool(local) and sep == "@" and bool(domain) and "." in domain
```

### Keyword-Only Arguments

```python
def create_user(
    name: str,
    email: str,
    *,
    role: str = "user",
    send_welcome: bool = False,
) -> User:
    ...

create_user("Alice", "alice@example.com", role="admin", send_welcome=True)
```

### Early Returns

```python
def process_order(order: Order | None) -> Success[Order] | Failure[str]:
    if order is None:
        return Failure("Order not found")
    if not order.items:
        return Failure("No items")
    return Success(complete_order(order))
```

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

## Code Quality

```python
from typing import TypedDict

# Avoid magic numbers
MAX_RETRIES = 3
TIMEOUT_SECONDS = 30

if retry_count > MAX_RETRIES:
    raise MaxRetriesExceeded()

# Avoid deep nesting - use early returns
class ProcessData(TypedDict):
    is_valid: bool

def has_permission(data: ProcessData) -> bool:
    """Check if data has required permissions."""
    return True

def transform(data: ProcessData) -> dict[str, object]:
    """Transform validated data."""
    return {"status": "ok"}

def process_data(
    data: ProcessData | None,
) -> Success[dict[str, object]] | Failure[str]:
    if data is None:
        return Failure("No data")
    if data.get("is_valid") is not True:
        return Failure("Invalid")
    if not has_permission(data):
        return Failure("Forbidden")
    return Success(transform(data))
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

Avoid common mistakes:
- Use `typeCheckingMode`, not `disallowUntyped` (mypy concept)
- Ensure `reportGeneralTypeIssues` is enabled in strict mode

### CI Enforcement

CI must enforce:
- Ruff linting (zero errors)
- Ruff formatting check
- pyright strict mode
- No implicit `Any` in public APIs (except documented boundary cases)

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

**Principles**: Explicit is better than implicit. Simple is better than complex. Readability counts.
