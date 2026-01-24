---
name: coding-standards-python
description: Python language-level coding standards and best practices for Python 3.11+. Formatting handled by Ruff.
---

# Python Coding Standards

Language-level standards for Python 3.11+. Formatting handled by Ruff formatter.

## Naming Conventions

```python
# Variables and functions: snake_case
user_count = 42
is_authenticated = True

def calculate_total(items: list[dict]) -> float:
    pass

# Classes: PascalCase
class UserRepository:
    pass

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
    pass

def process_items(items: list[str]) -> list[int]:
    pass
```

### Prefer Modern Syntax (Python 3.10+)

```python
def process_data(data: dict[str, int]) -> list[str]:
    pass

def get_user(user_id: str) -> User | None:
    pass
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

def parse_response(data: dict) -> UserPayload:
    payload: UserPayload = {
        "id": data["id"],
        "name": data["name"],
        "email": data["email"],
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

Use `Any` only at system boundaries. Document the reason.

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
    pass
```

## Error Handling

### Exception Inheritance

Domain exceptions must inherit from `Exception`, not `BaseException`.

**Rationale**: `BaseException` is reserved for system-level exceptions (KeyboardInterrupt, SystemExit).

```python
class ApplicationError(Exception):
    """Base for all application errors."""
    pass

class ValidationError(ApplicationError):
    """Validation failure."""
    pass
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

### Result Pattern

For expected domain errors, use Success/Failure pattern. Return type is always `Success[T] | Failure[E]`.

```python
import json
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

def parse_config(data: str) -> Success[dict[str, object]] | Failure[str]:
    try:
        parsed = json.loads(data)
        return Success(parsed)
    except json.JSONDecodeError as e:
        return Failure(f"Invalid JSON: {e}")

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
            pass
        case Status.CONFIRMED:
            pass
        case Status.SHIPPED:
            pass
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
def calculate_total(items: list[dict[str, float]]) -> float:
    return sum(item["price"] * item["quantity"] for item in items)

def validate_email(email: str) -> bool:
    """Basic email validation. Not RFC-compliant."""
    parts = email.split("@")
    return len(parts) == 2 and "." in parts[1]
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
    pass

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
    pass
```

### Comments Explain WHY

```python
# Exponential backoff prevents API overload during outages
delay = min(30, 2**retry_count)
```

## Code Quality

```python
# Avoid magic numbers
MAX_RETRIES = 3
TIMEOUT_SECONDS = 30

if retry_count > MAX_RETRIES:
    raise MaxRetriesExceeded()

# Avoid deep nesting - use early returns
def process_data(data: dict | None) -> Success[dict] | Failure[str]:
    if data is None:
        return Failure("No data")
    if not data.get("is_valid"):
        return Failure("Invalid")
    if not has_permission(data):
        return Failure("Forbidden")
    return Success(transform(data))
```

---

# Project / Backend Engineering Standards

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

# Never log PII or secrets
logger.info(f"User: {user.email}")  # Wrong
logger.debug(f"API key: {api_key}")  # Wrong

# Log events without sensitive data
logger.info("User logged in", extra={"user_id": user.id})
```

### Prefer Structured Logging and Lazy Formatting

Use `extra` for structured data. Use lazy formatting (%) for performance.

```python
# Structured logging (preferred)
logger.info(
    "Order created",
    extra={
        "order_id": order.id,
        "user_id": order.user_id,
        "total": str(order.total),
    },
)

# Lazy formatting (recommended for simple cases)
logger.info("Order %s created for user %s", order.id, order.user_id)

# f-strings work but are eagerly evaluated (not recommended)
logger.info(f"Order {order.id} created")
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

def get_user(user_id: str, conn: sqlite3.Connection) -> tuple | None:
    cursor = conn.execute("SELECT * FROM users WHERE id = ?", (user_id,))
    return cursor.fetchone()
```

**psycopg (PostgreSQL)**:

```python
import psycopg

def get_user(user_id: str, conn: psycopg.Connection) -> tuple | None:
    cursor = conn.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    return cursor.fetchone()
```

### Never Use shell=True

```python
import subprocess

subprocess.run(["ls", "-l", filename], capture_output=True, check=True)
```

### Do Not Deserialize Untrusted Pickle

```python
import json

data = json.loads(untrusted_string)
```

### Secrets from Environment

```python
import os

API_KEY = os.environ["API_KEY"]
DATABASE_URL = os.getenv("DATABASE_URL")
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

```python
from datetime import datetime

from freezegun import freeze_time

@freeze_time("2024-01-01 12:00:00")
def test_order_timestamp():
    order = create_order(items)
    assert order.created_at == datetime(2024, 1, 1, 12, 0, 0)
```

### Mock External I/O

```python
from unittest.mock import patch

@patch("myapp.services.send_email")
def test_registration_sends_email(mock_send):
    register_user(email="user@example.com")
    mock_send.assert_called_once()
```

### No Network in Unit Tests

```python
import pytest

@pytest.mark.integration
def test_api_call():
    response = api_client.fetch_user("123")
    assert response.status == 200
```

---

**Principles**: Explicit is better than implicit. Simple is better than complex. Readability counts.
