---
name: coding-standards-python
description: Python language-level coding standards and best practices for Python 3.11+. Formatting handled by Black/Ruff.
---

# Python Coding Standards

Language-level standards for Python 3.11+. Project-specific backend patterns in backend-patterns-fastapi.md. Formatting handled by Black or Ruff formatter.

## Naming Conventions

### Variables and Functions

Use `snake_case` for variables and functions.

```python
user_count = 42
is_authenticated = True

def calculate_total(items: list[dict]) -> float:
    pass

def fetch_user_data(user_id: str) -> dict:
    pass
```

### Classes

Use `PascalCase` for class names.

```python
class UserRepository:
    pass

class OrderProcessor:
    pass
```

### Constants

Use `UPPER_SNAKE_CASE` for module-level constants.

```python
MAX_RETRIES = 3
DEFAULT_TIMEOUT = 30
API_BASE_URL = "https://api.example.com"
```

### Private Members

Use single underscore prefix for internal use.

```python
class User:
    def __init__(self, name: str):
        self._name = name

    def _validate(self) -> bool:
        return len(self._name) > 0
```

### Single Character Names

Acceptable for loop counters, lambda parameters, mathematical operations. Avoid for other cases.

```python
# Acceptable
for i, item in enumerate(items):
    pass

result = map(lambda x: x * 2, numbers)

# Avoid
n = fetch_count()  # Use: count = fetch_count()
d = get_data()     # Use: data = get_data()
```

## Type Hints

### Use Type Hints for Public APIs

All public functions must have type annotations.

```python
from typing import Any

def fetch_user(user_id: str) -> dict[str, Any]:
    pass

def process_items(items: list[str]) -> list[int]:
    pass
```

### Prefer Modern Syntax (Python 3.10+)

```python
# Use built-in generics
def process_data(data: dict[str, int]) -> list[str]:
    pass

def get_user(user_id: str) -> User | None:
    pass

# Avoid legacy typing module aliases
from typing import Dict, List, Optional

def process_data_old(data: Dict[str, int]) -> List[str]:
    pass
```

### Prefer Structured Types

Use TypedDict or dataclasses instead of generic `dict` or `list[dict]`.

**TypedDict**: For static type checking of dictionary structures. Not a runtime validator.

```python
from typing import TypedDict

class UserPayload(TypedDict):
    id: str
    name: str
    email: str

# Correct: dictionary literal with type annotation
def parse_response(data: dict) -> UserPayload:
    payload: UserPayload = {
        "id": data["id"],
        "name": data["name"],
        "email": data["email"]
    }
    return payload

# TypedDict is not a constructor
# payload = UserPayload(...)  # Wrong
```

**Dataclass**: For domain models with behavior.

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class User:
    id: str
    name: str
    email: str

def create_user(payload: UserPayload) -> User:
    return User(**payload)
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
        email=str(response["email"])
    )
```

### Use Literal for Fixed Values

```python
from typing import Literal

Status = Literal["active", "inactive", "archived"]

def set_status(status: Status) -> None:
    pass
```

## Error Handling

### Exception Inheritance

Domain exceptions must inherit from `Exception`, not `BaseException`.

**Rationale**: `BaseException` is reserved for system-level exceptions (KeyboardInterrupt, SystemExit). User exceptions should inherit from `Exception`.

```python
class ApplicationError(Exception):
    """Base for all application errors."""
    pass

class DomainError(ApplicationError):
    """Base for domain logic errors."""
    pass

class ValidationError(DomainError):
    """Validation failure."""
    pass

class ResourceNotFoundError(DomainError):
    """Resource not found."""
    pass
```

### Never Silence Exceptions

```python
import logging

logger = logging.getLogger(__name__)

# Never
try:
    risky_operation()
except Exception:
    pass  # Silent failure

# Always log or re-raise
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

### Use Context Managers

```python
with open("file.txt") as f:
    data = f.read()
```

### Result Pattern (Optional)

For expected domain errors, consider Result pattern. Use when explicit error handling improves clarity.

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

type Result[T, E] = Success[T] | Failure[E]

# Use for expected failures
def parse_input(data: str) -> Result[dict, str]:
    try:
        parsed = json.loads(data)
        return Success(parsed)
    except json.JSONDecodeError:
        return Failure("Invalid JSON")

# Pattern matching
result = parse_input(input_string)
match result:
    case Success(value):
        process(value)
    case Failure(error):
        handle_error(error)
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
class Order:
    id: str
    user_id: str
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
            pass
        case Status.CONFIRMED:
            pass
        case Status.SHIPPED:
            pass
```

### Prefer Immutable Data

```python
from dataclasses import dataclass

# Immutable by default
@dataclass(frozen=True)
class Point:
    x: float
    y: float

# Mutable only when necessary
@dataclass
class Counter:
    count: int = 0

    def increment(self) -> None:
        self.count += 1
```

## Pythonic Idioms

### Comprehensions for Simple Cases

```python
squares = [x ** 2 for x in range(10)]
even_squares = [x ** 2 for x in range(10) if x % 2 == 0]
user_map = {user.id: user.name for user in users}
```

### Don't Nest Comprehensions Deeply

```python
# Readable
matrix = [[1, 2, 3], [4, 5, 6]]
flat = [item for row in matrix for item in row]

# Too complex - use loops
result = []
for row in matrix:
    if sum(row) > 5:
        result.append([x * y for x in row if x > 0])
```

### Generators for Large Data

```python
from typing import Iterator

def read_large_file(path: str) -> Iterator[str]:
    with open(path) as f:
        for line in f:
            yield line.strip()

for line in read_large_file("data.txt"):
    process(line)
```

### Tuple Unpacking

```python
x, y = (1, 2)
first, *rest = [1, 2, 3, 4]
```

### Use `enumerate` and `zip`

```python
for index, item in enumerate(items):
    print(f"{index}: {item}")

for name, age in zip(names, ages):
    print(f"{name} is {age}")
```

### Use `dict.get()` with Default

```python
name = user.get("name", "Anonymous")
count = data.get("count", 0)
```

### Use `any()` and `all()`

```python
has_admin = any(user.role == "admin" for user in users)
all_active = all(user.active for user in users)
```

## Documentation

### Docstrings for Public APIs

All public functions must have docstrings.

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
    pass
```

### Comments Explain WHY, Not WHAT

```python
# Explains reasoning
# Exponential backoff prevents API overload during outages
delay = min(30, 2 ** retry_count)

# Avoid stating obvious
# Increment counter
counter += 1
```

## Code Smells

### Avoid Magic Numbers

```python
MAX_RETRIES = 3
TIMEOUT_SECONDS = 30

if retry_count > MAX_RETRIES:
    raise MaxRetriesExceeded()
```

### Avoid Deep Nesting

```python
def process_data(data: dict | None) -> Result[dict, str]:
    if data is None:
        return Failure("No data")
    if not data.get("is_valid"):
        return Failure("Invalid")
    if not has_permission(data):
        return Failure("Forbidden")
    return Success(transform(data))
```

### Avoid Code Duplication

```python
def format_price(amount: float, currency: str = "USD") -> str:
    return f"{currency} {amount:.2f}"

price_usd = format_price(100.0)
price_eur = format_price(100.0, "EUR")
```

## Functions

### Single Responsibility

```python
def calculate_total(items: list[dict]) -> float:
    return sum(item["price"] * item["quantity"] for item in items)

def validate_email(email: str) -> bool:
    return "@" in email and "." in email.split("@")[1]
```

### Keyword-Only Arguments

```python
def create_user(
    name: str,
    email: str,
    *,
    role: str = "user",
    send_welcome: bool = False
) -> User:
    pass

create_user(
    "Alice",
    "alice@example.com",
    role="admin",
    send_welcome=True
)
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

# Project / Backend Engineering Standards

## Tooling and Static Analysis

### Required Tools

- Python 3.11+
- Ruff for linting
- Black or Ruff formatter for formatting
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
- Formatting check (Black or Ruff)
- pyright strict mode
- No implicit `Any` in public APIs

## Public API Definition

A module's public API consists of:
- Top-level functions/classes without `_` prefix
- Names in `__all__` if present
- Imports in `__init__.py`

Public API requirements:
- Must have type annotations
- Must have docstrings
- Must validate external inputs
- Must not use `Any` without documentation

Private code (`_` prefix):
- May omit docstrings if obvious
- Should still have type hints

## Logging

### Never Log Secrets or PII

Must not log:
- Passwords, API keys, tokens
- Email addresses, names, phone numbers
- Full request/response bodies

```python
import logging

logger = logging.getLogger(__name__)

# Never
logger.info(f"User: {user.email}")  # PII
logger.debug(f"API key: {api_key}")  # Secret

# Log events without sensitive data
logger.info("User logged in", extra={"user_id": user.id})
```

### Prefer Structured Logging

Use `extra` for structured data. Avoid eager evaluation.

```python
# Prefer structured logging
logger.info(
    "Order created",
    extra={
        "order_id": order.id,
        "user_id": order.user_id,
        "total": str(order.total)
    }
)

# Acceptable with lazy formatting
logger.info("Order %s created for user %s", order.id, order.user_id)

# Avoid f-strings (eager evaluation)
logger.info(f"Order {order.id} created")  # Acceptable but not preferred
```

### Use logger.exception() for Unexpected Errors

```python
try:
    process_payment(order)
except PaymentError:
    logger.exception("Payment failed", extra={"order_id": order.id})
    raise
```

## Security

### Never Build SQL with String Concatenation

Use parameterized queries. Parameter style varies by driver.

**sqlite3**:

```python
import sqlite3

def get_user(user_id: str, conn: sqlite3.Connection) -> dict | None:
    cursor = conn.execute(
        "SELECT * FROM users WHERE id = ?",
        (user_id,)
    )
    return cursor.fetchone()
```

**psycopg (PostgreSQL)**:

```python
import psycopg

def get_user(user_id: str, conn: psycopg.Connection) -> dict | None:
    cursor = conn.execute(
        "SELECT * FROM users WHERE id = %s",
        (user_id,)
    )
    return cursor.fetchone()
```

**Never**:

```python
# SQL injection vulnerability
query = f"SELECT * FROM users WHERE id = '{user_id}'"
```

### Never Use shell=True

```python
import subprocess

# Correct
subprocess.run(
    ["ls", "-l", filename],
    capture_output=True,
    check=True
)

# Never - command injection
subprocess.run(
    f"ls -l {filename}",
    shell=True  # Dangerous
)
```

### Do Not Deserialize Untrusted Pickle

```python
import pickle
import json

# Never - code execution vulnerability
data = pickle.loads(untrusted_bytes)

# Use JSON for untrusted data
data = json.loads(untrusted_string)
```

### Secrets from Environment

```python
import os

API_KEY = os.environ["API_KEY"]
DATABASE_URL = os.getenv("DATABASE_URL")

# Never hardcode
API_KEY = "sk-1234..."  # Exposed in git
```

## Testing

### Use pytest

```python
import pytest

def test_calculate_discount():
    price = 100.0
    result = calculate_discount(price, "premium")
    assert result == 90.0
```

### Tests Must Be Deterministic

No dependency on current time, random values, or external state.

```python
from freezegun import freeze_time
from datetime import datetime

@freeze_time("2024-01-01 12:00:00")
def test_order_timestamp():
    order = create_order(items)
    assert order.created_at == datetime(2024, 1, 1, 12, 0, 0)
```

### Mock External I/O

```python
from unittest.mock import Mock, patch

@patch("myapp.services.send_email")
def test_registration_sends_email(mock_send: Mock):
    register_user(email="user@example.com")
    mock_send.assert_called_once()
```

### No Network in Unit Tests

Unit tests must not make network calls. Use integration test markers.

```python
@pytest.mark.integration
def test_api_call():
    response = api_client.fetch_user("123")
    assert response.status == 200
```

---

**Principles**: Explicit is better than implicit. Simple is better than complex. Readability counts.
