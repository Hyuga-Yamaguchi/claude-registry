---
name: coding-standards-python
description: Python language-level coding standards and best practices. Follows PEP 8 and community conventions. Formatting rules handled by Black/Ruff.
---

# Python Coding Standards

Production-grade standards for Python 3.11+ backend and service development. Formatting handled by Black/Ruff. Based on PEP 8 and community best practices.

## Tooling and Static Analysis

### Required Tools

- Python 3.11 or later
- Ruff for linting
- Black for formatting
- pyright as the official type checker

### Enforcement Rules

CI MUST enforce:
- Ruff linting with zero errors
- Black formatting (check mode)
- pyright type checking with strict mode
- No implicit Any types in public APIs
- No untyped function definitions in public modules

Pre-commit hooks SHOULD include:
- Ruff auto-fix
- Black formatting
- Type checking

Strict type checking MUST include:
- `reportGeneralTypeIssues: true`
- `reportUnknownMemberType: true`
- `disallowUntyped: true` for public API modules

## Imports and Module Organization

Import grouping MUST follow this order (Ruff enforces):
1. Standard library imports
2. Third-party imports
3. Local application imports

```python
import json
import os
from typing import Any

import httpx
from pydantic import BaseModel

from myapp.domain.models import User
from myapp.infrastructure.repository import UserRepository
```

Rules:
- MUST NOT use wildcard imports (`from module import *`)
- MUST NOT create circular dependencies
- SHOULD use absolute imports; relative imports only within the same package
- SHOULD group related imports with a blank line separator

## Public API Definition

A module's public API consists of:
- Top-level functions and classes without underscore prefix
- Names explicitly listed in `__all__` if present
- Anything imported in `__init__.py` of a package

Public API requirements:
- MUST have complete type annotations
- MUST have docstrings
- MUST NOT use `Any` unless documented
- MUST validate all external inputs

Internal/private code (prefixed with `_`):
- MAY omit docstrings if purpose is obvious
- SHOULD still have type hints

## Naming Conventions

### Variables and Functions

Use `snake_case` for variables and functions.

```python
market_search_query = "election"
is_user_authenticated = True
total_revenue = 1000

def fetch_market_data(market_id: str) -> dict:
    pass

def calculate_similarity(a: list[float], b: list[float]) -> float:
    pass
```

### Classes

Use `PascalCase` for class names.

```python
class MarketAnalyzer:
    pass

class UserProfile:
    pass
```

### Constants

Use `UPPER_SNAKE_CASE` for module-level constants.

```python
MAX_RETRIES = 3
API_BASE_URL = "https://api.example.com"
DEFAULT_TIMEOUT_SECONDS = 5
```

### Private Members

Use single underscore prefix for internal use.

```python
class User:
    def __init__(self, name: str):
        self._name = name

    def _validate_email(self, email: str) -> bool:
        return "@" in email
```

### Avoid Single Character Names

Exception: loop counters, lambda parameters, mathematical operations.

```python
# Acceptable
for i, item in enumerate(items):
    pass

result = map(lambda x: x * 2, numbers)

# Avoid
n = len(users)
d = fetch_data()
```

## Type Hints

### Always Use Type Hints for Public APIs

All public functions MUST have complete type annotations.

```python
from typing import Any

# Correct - use typing.Any for truly dynamic data
def fetch_user(user_id: str) -> dict[str, Any]:
    pass

# Incorrect - 'any' is the built-in function, not a valid type annotation
def fetch_user_bad(user_id: str) -> dict[str, any]:
    pass
```

### Prefer Structured Types Over Generic Containers

Use TypedDict or dataclasses instead of `dict` or `list[dict]`.

```python
from typing import TypedDict
from dataclasses import dataclass

# TypedDict for data transfer objects
class UserPayload(TypedDict):
    id: str
    name: str
    email: str
    age: int

def parse_user_response(data: dict) -> UserPayload:
    return UserPayload(
        id=data["id"],
        name=data["name"],
        email=data["email"],
        age=data["age"]
    )

# Dataclass for domain models
@dataclass(frozen=True)
class User:
    id: str
    name: str
    email: str
    age: int = 0

def create_user(payload: UserPayload) -> User:
    return User(**payload)

# Avoid weak types
def create_user_bad(data: dict) -> dict:
    return data

def get_users_bad() -> list[dict]:
    pass
```

### When to Use `Any`

Use `Any` only at system boundaries (external APIs, legacy code, truly dynamic data). MUST document why.

```python
from typing import Any

# Acceptable - external API response is untyped
def parse_external_api_response(response: dict[str, Any]) -> User:
    """Parse external API response into typed User model.

    Uses Any because external API schema is not under our control.
    """
    return User(
        id=str(response["id"]),
        name=str(response["name"]),
        email=str(response["email"])
    )

# Avoid - internal application code should be typed
def process_user_bad(user: dict[str, Any]) -> dict[str, Any]:
    pass
```

### Use Modern Type Syntax (Python 3.10+)

```python
# Prefer Python 3.10+ syntax
def process_data(data: dict[str, int]) -> list[str]:
    pass

def get_user(user_id: str) -> User | None:
    pass

# Avoid legacy typing module aliases
from typing import Dict, List, Optional

def process_data_old(data: Dict[str, int]) -> List[str]:
    pass

def get_user_old(user_id: str) -> Optional[User]:
    pass
```

### Use Literal for Fixed Values

```python
from typing import Literal

Status = Literal["active", "inactive", "archived"]

def set_status(status: Status) -> None:
    pass
```

## Error Handling

### Exception Philosophy

**Use exceptions for unexpected failures.** Programming errors, network failures, missing files, invalid state.

**Use Result pattern for expected domain errors.** Validation failures, business rule violations, not-found cases.

### Domain Exceptions Must Inherit from Base Exception

```python
class ApplicationError(Exception):
    """Base exception for all application errors."""
    pass

class DomainError(ApplicationError):
    """Base for domain/business logic errors."""
    pass

class ValidationError(DomainError):
    """Raised when data validation fails."""
    pass

class ResourceNotFoundError(DomainError):
    """Raised when requested resource doesn't exist."""
    pass

class InfrastructureError(ApplicationError):
    """Base for infrastructure errors."""
    pass

class DatabaseError(InfrastructureError):
    """Raised when database operations fail."""
    pass
```

### Never Silently Swallow Exceptions

```python
import logging

logger = logging.getLogger(__name__)

# Never do this
try:
    risky_operation()
except Exception:
    pass  # Silent failure

# Always log or re-raise
try:
    risky_operation()
except ValueError:
    logger.exception("Operation failed with validation error")
    raise
except Exception:
    logger.exception("Unexpected error in risky_operation")
    raise
```

### Use logger.exception() for Unexpected Errors

Use `logger.exception()` to automatically include stack trace. See Logging section for details.

```python
import logging

logger = logging.getLogger(__name__)

def process_data(data: str) -> dict:
    try:
        return parse_and_validate(data)
    except ValidationError:
        logger.warning("Validation failed for data", extra={"data_length": len(data)})
        raise
    except Exception:
        logger.exception("Unexpected error processing data")
        raise
```

### Re-raise with Context

Always use `raise ... from e` to preserve the exception chain.

```python
import json
from typing import Any

# Correct - preserve exception chain
def parse_config(config_str: str) -> dict[str, Any]:
    try:
        return json.loads(config_str)
    except json.JSONDecodeError as e:
        raise ValueError("Invalid configuration format") from e

# Avoid - loses original exception
def parse_config_bad(config_str: str) -> dict[str, Any]:
    try:
        return json.loads(config_str)
    except json.JSONDecodeError:
        raise ValueError("Invalid configuration format")
```

### Use Context Managers

```python
# Automatic resource cleanup
with open("file.txt", "r") as f:
    data = f.read()
```

### Result Pattern for Expected Errors

```python
from dataclasses import dataclass
from typing import Generic, TypeVar
import json

T = TypeVar("T")
E = TypeVar("E")

@dataclass(frozen=True)
class Success(Generic[T]):
    value: T

@dataclass(frozen=True)
class Failure(Generic[E]):
    error: E

Result = Success[T] | Failure[E]

# Use for expected domain errors
def parse_user_input(data: str) -> Result[UserPayload, str]:
    try:
        parsed = json.loads(data)
        if "email" not in parsed:
            return Failure("Missing required field: email")
        return Success(UserPayload(
            id=parsed["id"],
            name=parsed["name"],
            email=parsed["email"],
            age=parsed.get("age", 0)
        ))
    except json.JSONDecodeError:
        return Failure("Invalid JSON format")

# Caller handles both cases explicitly
result = parse_user_input(input_string)
match result:
    case Success(user):
        process_user(user)
    case Failure(error):
        handle_error(error)
```

## Logging

### Never Log Secrets or PII

MUST NOT log:
- Passwords, API keys, tokens, secrets
- Email addresses, names, phone numbers
- Full request/response bodies (may contain sensitive data)

```python
import logging

logger = logging.getLogger(__name__)

# Never log sensitive data
logger.info(f"User logged in: {user.email}")  # PII
logger.debug(f"API key: {api_key}")  # Secret
logger.info(f"Password: {password}")  # Secret
logger.debug(f"Request body: {request_body}")  # May contain secrets

# Log events without sensitive data
logger.info("User logged in", extra={"user_id": user.id})
logger.debug("API request initiated", extra={"endpoint": "/users"})
```

### Prefer Structured Logging

Use `extra` parameter for structured data. Avoid f-strings in log messages because they eagerly evaluate and can't be deferred or optimized by the logging system.

```python
import logging

logger = logging.getLogger(__name__)

# Prefer structured logging with extra
logger.info(
    "Order created",
    extra={
        "order_id": order.id,
        "user_id": order.user_id,
        "total": str(order.total),
        "item_count": len(order.items)
    }
)

# Avoid f-string formatting (eager evaluation, loses structure)
logger.info(f"Order {order.id} created for user {order.user_id}")
```

### Use logger.exception() for Unexpected Errors

`logger.exception()` automatically includes stack trace. Use for unexpected errors only.

```python
import logging

logger = logging.getLogger(__name__)

try:
    process_payment(order)
except PaymentError:
    logger.exception("Payment processing failed", extra={"order_id": order.id})
    raise
```

### Logging Should Describe System Events

Log what the system is doing, not internal variable values.

```python
import logging

logger = logging.getLogger(__name__)

# Good - describes system event
logger.info("Database connection pool exhausted, waiting for available connection")
logger.warning("API rate limit exceeded, implementing backoff")

# Avoid - internal variable dump
logger.debug(f"x = {x}, y = {y}, z = {z}")
```

## Testing

### Use pytest

MUST use pytest as the test framework.

```python
import pytest
from decimal import Decimal
from datetime import datetime
from unittest.mock import Mock, patch

# One concept per test
def test_calculate_discount_for_premium_user():
    price = Decimal("100.00")
    result = calculate_discount(price, "premium")
    assert result == Decimal("90.00")

def test_calculate_discount_for_regular_user():
    price = Decimal("100.00")
    result = calculate_discount(price, "regular")
    assert result == Decimal("100.00")
```

### Tests Must Be Deterministic

MUST NOT depend on current time, random values, or external state.

```python
from freezegun import freeze_time
from datetime import datetime

# Freeze time for deterministic tests
@freeze_time("2024-01-01 12:00:00")
def test_order_creation_timestamp():
    order = create_order(items=[item1, item2])
    assert order.created_at == datetime(2024, 1, 1, 12, 0, 0)

# Avoid non-deterministic tests
def test_order_creation_timestamp_bad():
    order = create_order(items=[item1, item2])
    # Flaky - depends on current time
    assert order.created_at <= datetime.now()
```

### Mock External I/O

```python
from unittest.mock import Mock, patch

# Mock external services
@patch("myapp.services.email_service.send_email")
def test_user_registration_sends_email(mock_send_email: Mock):
    register_user(email="user@example.com", name="Alice")

    mock_send_email.assert_called_once_with(
        to="user@example.com",
        subject="Welcome to our platform",
        body="Hello Alice..."
    )
```

### No Network Calls in Unit Tests

Unit tests MUST NOT make network calls.

```python
# Unit tests - no network
def test_format_api_url():
    url = format_api_url("users", user_id="123")
    assert url == "https://api.example.com/users/123"

# Integration tests - network allowed
# Mark with @pytest.mark.integration and run in separate test suite
@pytest.mark.integration
def test_fetch_user_from_api():
    user = api_client.fetch_user("123")
    assert user.id == "123"
```

## Security

### Never Build SQL with String Concatenation

Use parameterized queries with your database driver's parameter style. Never concatenate user input into SQL strings.

```python
# Correct - parameterized query (use your driver's parameter style)
# Examples: ? (SQLite), :name (Oracle), %s (PostgreSQL/MySQL), etc.
def get_user(user_id: str, db_connection) -> User | None:
    # Named parameters (SQLite, PostgreSQL with psycopg2)
    query = "SELECT * FROM users WHERE id = :user_id"
    result = db_connection.execute(query, {"user_id": user_id}).fetchone()
    return User(**result) if result else None

# NEVER DO THIS - SQL injection vulnerability
def get_user_bad(user_id: str, db_connection) -> User | None:
    query = f"SELECT * FROM users WHERE id = '{user_id}'"
    return db_connection.execute(query).fetchone()
```

### Never Use shell=True

```python
import subprocess

# Correct - list of arguments
def run_command(filename: str) -> str:
    result = subprocess.run(
        ["ls", "-l", filename],
        capture_output=True,
        text=True,
        check=True
    )
    return result.stdout

# NEVER DO THIS - command injection vulnerability
def run_command_bad(filename: str) -> str:
    result = subprocess.run(
        f"ls -l {filename}",
        shell=True,  # Dangerous
        capture_output=True
    )
    return result.stdout
```

### Do Not Deserialize Untrusted Pickle

```python
import pickle
import json
from typing import Any

# NEVER deserialize untrusted pickle - code execution vulnerability
def load_data_bad(data: bytes) -> Any:
    return pickle.loads(data)  # Dangerous

# Use JSON for untrusted data
def load_data(data: str) -> dict:
    return json.loads(data)
```

### Secrets Must Come from Environment or Secret Manager

```python
import os

# Correct - from environment
API_KEY = os.environ["API_KEY"]
DATABASE_URL = os.getenv("DATABASE_URL")

# NEVER hardcode secrets
API_KEY_BAD = "sk-1234567890abcdef"  # Exposed in version control
```

## Functions

### Single Responsibility

Each function SHOULD do one thing well.

```python
from decimal import Decimal
from dataclasses import dataclass

# Each function has clear purpose
def calculate_total(items: list[dict]) -> Decimal:
    return sum(Decimal(str(item["price"])) * item["quantity"] for item in items)

def validate_email(email: str) -> bool:
    return "@" in email and "." in email.split("@")[1]

def format_currency(amount: Decimal, currency: str = "USD") -> str:
    return f"{currency} {amount:.2f}"

# Avoid doing too much
def process_order_bad(order_data: dict) -> None:
    # Validation, calculation, formatting, saving, emailing - too many responsibilities
    pass
```

### Use Keyword Arguments for Clarity

Force keyword-only arguments for functions with multiple parameters.

```python
def create_user(
    name: str,
    email: str,
    *,
    role: str = "user",
    send_welcome: bool = False,
    notify_admin: bool = False
) -> User:
    pass

# Clear usage
create_user(
    name="Alice",
    email="alice@example.com",
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
        return Failure("Order has no items")

    if not has_inventory(order.items):
        return Failure("Insufficient inventory")

    return Success(complete_order(order))
```

## Data Structures

### Use Dataclasses for Domain Models

```python
from dataclasses import dataclass
from decimal import Decimal
from datetime import datetime

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
    created_at: datetime

# Avoid raw dicts for domain models
order_bad = {
    "id": "123",
    "user_id": "456",
    "items": [],
    "total": {"amount": 100.0, "currency": "USD"}
}
```

### Use Enums for Fixed Sets

```python
from enum import Enum, auto

class OrderStatus(Enum):
    PENDING = auto()
    CONFIRMED = auto()
    SHIPPED = auto()
    DELIVERED = auto()
    CANCELLED = auto()

def update_status(order: Order, status: OrderStatus) -> Order:
    match status:
        case OrderStatus.PENDING:
            pass
        case OrderStatus.CONFIRMED:
            pass
        case _:
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
class MutableCounter:
    count: int = 0

    def increment(self) -> None:
        self.count += 1
```

## Comprehensions and Generators

### Use Comprehensions for Simple Cases

```python
# Clear and concise
squares = [x ** 2 for x in range(10)]
even_squares = [x ** 2 for x in range(10) if x % 2 == 0]
user_map = {user.id: user.name for user in users}
```

### Don't Nest Comprehensions Too Deeply

```python
# Readable
matrix = [[1, 2, 3], [4, 5, 6]]
flat = [item for row in matrix for item in row]

# Too complex - use loops
result_bad = [[x * y for x in row if x > 0] for row in matrix if sum(row) > 5]

# Better
result = []
for row in matrix:
    if sum(row) > 5:
        result.append([x * y for x in row if x > 0])
```

### Use Generators for Large Data

```python
from typing import Iterator

def read_large_file(path: str) -> Iterator[str]:
    with open(path) as f:
        for line in f:
            yield line.strip()

# Memory efficient
for line in read_large_file("huge_file.txt"):
    process(line)
```

## Code Smells

### Avoid Magic Numbers

```python
# Named constants
MAX_RETRIES = 3
TIMEOUT_SECONDS = 30
ITEMS_PER_PAGE = 20

class MaxRetriesExceededError(Exception):
    pass

if retry_count > MAX_RETRIES:
    raise MaxRetriesExceededError()
```

### Avoid Deep Nesting

```python
# Guard clauses
def process_data(data: dict | None) -> Result[dict, str]:
    if data is None:
        return Failure("No data")

    if not data.get("is_valid"):
        return Failure("Invalid data")

    if not has_permission(data):
        return Failure("Permission denied")

    return Success(transform(data))
```

### Avoid Code Duplication

```python
from decimal import Decimal

def format_price(amount: Decimal, currency: str = "USD") -> str:
    return f"{currency} {amount:.2f}"

price_usd = format_price(Decimal("100.00"))
price_eur = format_price(Decimal("100.00"), "EUR")
```

## Documentation

### Docstrings for Public APIs

All public functions MUST have docstrings.

```python
from typing import TypedDict

class Market(TypedDict):
    id: str
    name: str
    score: float

def search_markets(query: str, limit: int = 10) -> list[Market]:
    """Search markets using semantic similarity.

    Args:
        query: Natural language search query
        limit: Maximum number of results (default: 10)

    Returns:
        List of markets sorted by similarity score

    Raises:
        ValueError: If query is empty
        ConnectionError: If API is unavailable

    Example:
        >>> results = search_markets("election", 5)
        >>> print(results[0]["name"])
        "2024 Election"
    """
    pass
```

### Comments Explain WHY, Not WHAT

```python
# Explains reasoning
# Use exponential backoff to prevent overwhelming the API during outages
delay = min(30, 2 ** retry_count)

# Avoid stating the obvious
# Increment counter
counter += 1
```

## Pythonic Idioms

### Use `with` for Resources

```python
with open("file.txt") as f:
    data = f.read()
```

### Use Tuple Unpacking

```python
x, y = (1, 2)
first, *rest = [1, 2, 3, 4]
name, age, *_ = get_user_data()
```

### Use `enumerate` for Index + Value

```python
for index, item in enumerate(items):
    print(f"{index}: {item}")
```

### Use `zip` for Parallel Iteration

```python
for name, age in zip(names, ages):
    print(f"{name} is {age} years old")
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

---

**Remember**: Write code for humans. Explicit is better than implicit. Simple is better than complex.
