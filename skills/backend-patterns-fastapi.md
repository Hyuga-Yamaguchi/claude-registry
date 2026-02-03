---
name: backend-patterns-fastapi
description: FastAPI with Clojure-inspired architecture using SQLAlchemy Core (no ORM) - data-centric, pure functions, explicit SQL-to-Domain conversion. Hexagonal architecture with functional core for production-ready backends.
---

# FastAPI Backend Architecture (SQLAlchemy Core)

**Clojure-inspired backend architecture for FastAPI**: data-centric, pure functions at the core, side effects isolated at boundaries. Based on Hexagonal Architecture (Ports & Adapters).

**Philosophy**: "Functional Core, Imperative Shell" - pure domain logic at the core, orchestration in use cases, side effects (DB, HTTP, external APIs) at the adapter boundaries.

**DB Strategy**: SQLAlchemy Core (async) - NO ORM. Explicit SQL → Domain conversion for transparency and Clojure-style data transformation.

---

## Required Libraries

**Python**: 3.11+ (for `|` union syntax, `Self`, better async performance)

| Layer | Libraries | Purpose |
|-------|-----------|---------|
| **API** | `fastapi>=0.104.0`, `pydantic>=2.5.0`, `pydantic-settings>=2.1.0` | HTTP routing, validation, config |
| **Domain** | `dataclasses` (stdlib), `decimal` (stdlib), `datetime` (stdlib) | Pure business logic |
| **Use Cases** | `typing.Protocol` (stdlib) | Port definitions |
| **Adapters (DB)** | `sqlalchemy[asyncio]>=2.0.23` (Core only, **NO ORM**), `asyncpg>=0.29.0`, `alembic>=1.12.0` | Async SQL queries, migrations |
| **Adapters (HTTP)** | `httpx>=0.25.0` | Async HTTP client |
| **Testing** | `pytest>=7.4.0`, `pytest-asyncio>=0.21.0` | Async tests |
| **Utils** | `structlog>=23.2.0`, `tenacity>=8.2.0` (optional), `orjson>=3.9.0` (optional) | Logging, retries, fast JSON |

**Critical**: Use **SQLAlchemy Core** with `Table` + `MetaData`. Do NOT use ORM (`DeclarativeBase`, `relationship()`, mapped classes).

---

## Design Philosophy (Clojure Heritage)

### 1. Data-Centric Domain

* **Domain = "Data + Pure Functions"**
  * Treat `dataclass` as immutable values (frozen)
  * Domain calculations have **NO I/O** (no DB, HTTP, `datetime.now()`, `random`, `uuid.uuid4()`)
  * Business logic is testable without mocks

```python
# domain/models.py
from decimal import Decimal
from dataclasses import dataclass

@dataclass(frozen=True)
class OrderItem:
    unit_price: Decimal
    quantity: int

@dataclass(frozen=True)
class Order:
    id: str
    items: tuple[OrderItem, ...]
    user_tier: str

    @property
    def total(self) -> Decimal:
        return sum((item.unit_price * item.quantity for item in self.items), start=Decimal("0"))

# domain/services/discount.py
def calculate_discount(order: Order) -> Decimal:
    """Pure function: same input → same output, no side effects."""
    rate = Decimal("0.1") if order.user_tier == "premium" else Decimal("0")
    return order.total * rate
```

### 2. Unidirectional Dependencies (Dependency Inversion)

Dependencies point **inward** (toward domain core). Outer layers depend on inner layer abstractions (Ports).

```
┌─────────────────────────────────────┐
│   API Layer                         │  ← FastAPI routes, HTTP
│   - Pydantic DTOs, validation       │  depends on ↓
└──────────────┬──────────────────────┘
┌──────────────▼──────────────────────┐
│   Use Cases                         │  ← Orchestration, calls ports
│   - Defines Port interfaces         │  depends on ↓
└──────────────┬──────────────────────┘
┌──────────────▼──────────────────────┐
│   Domain (Core)                     │  ← Pure business logic
│   - Immutable models, pure functions│  ✅ NO outer layer imports
└─────────────────────────────────────┘
               ↑ implements ports
┌──────────────┴──────────────────────┐
│   Adapters                          │  ← Side effects (DB, APIs)
│   - SQLAlchemy Core (NO ORM)        │  ← Explicit SQL → Domain
└─────────────────────────────────────┘
```

**Rules**: `domain` imports nothing from outer layers. `usecases` imports domain + defines ports. `adapters` implements ports. `api` wires dependencies.

---

## Directory Structure

```
src/app/
  main.py                  # FastAPI app, DI setup

  api/                     # HTTP boundary
    routes/, deps.py
    schemas/               # Pydantic DTOs
    adapters/              # DTO ↔ Domain conversion
    errors.py              # HTTP exception mapping

  domain/                  # Pure business logic
    models.py              # Immutable dataclasses
    errors.py              # Domain exceptions
    services/              # Pure functions

  usecases/                # Orchestration
    user/
      create_user.py       # Use case (transaction boundary)
      ports.py             # Port interfaces (Protocol)
    shared/ports.py        # UnitOfWork, Clock, IdGenerator

  adapters/                # Infrastructure
    db/
      session.py           # AsyncEngine, async_session_maker
      tables.py            # SQLAlchemy Core Table definitions
      user_repo.py         # Repository implementation (Core)
      uow.py               # Unit of Work
      migrations/          # Alembic
    external/, cache/

  shared/
    clock.py, idgen.py, logging.py, settings.py
```

**Critical Rules**: `domain/` NO Pydantic/SQLAlchemy/HTTP. `usecases/` NO adapters imports. DB schemas defined with `Table` (not ORM models).

---

## Why SQLAlchemy Core (Not ORM)

### Problems with ORM
- **Stateful entities**: Lazy loading, session-bound state, implicit queries
- **Impure boundaries**: Domain models mixed with DB concerns (relationships, `Session` dependencies)
- **Opaque behavior**: Hard to reason about what SQL runs when
- **Clojure mismatch**: ORM promotes mutability and side effects in domain layer

### Benefits of Core
- **Explicit SQL**: You see exactly what queries run
- **Pure data flow**: SQL Row → Dict/Mapping → Domain (explicit transformation)
- **No hidden state**: Domain models are pure dataclasses with no DB dependencies
- **Parameterized safety**: Same protection against SQL injection as ORM
- **Async native**: `asyncpg` integration without ORM overhead
- **Dialect abstraction**: Still portable across DB backends

### Core API Requirements

**✅ REQUIRED (Use these)**:
- `Table` + `MetaData` for schema definitions
- `select()`, `insert()`, `update()`, `delete()` for queries
- `AsyncSession` with `session.execute(query)`
- `result.mappings()` for dict-like row access
- `.returning()` for INSERT/UPDATE (PostgreSQL/MySQL 8.0.1+/SQLite 3.35+)
- Explicit `_row_to_domain()` converters in Repository

**❌ PROHIBITED (Do NOT use)**:
- `DeclarativeBase` or any ORM base classes
- `relationship()` for foreign key navigation
- Mapped classes (`class User(Base)`)
- `Session.query()` or ORM query methods
- Lazy loading or session-bound entities
- Automatic eager loading or joinedload

**⚠️ OPTIONAL (Advanced)**:
- `AsyncConnection` instead of `AsyncSession` for lower-level control
- Raw SQL with `text()` for complex queries (still use parameter binding)
- CTE (Common Table Expressions) with `.cte()`
- Window functions for analytics

### Pattern: Row → Domain Conversion

```python
# adapters/db/user_repo.py
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from adapters.db.tables import users
from domain.models import User

class SQLAlchemyUserRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def find_by_id(self, user_id: str) -> User | None:
        query = select(users).where(users.c.id == user_id)
        result = await self.session.execute(query)
        row = result.mappings().one_or_none()
        return self._row_to_domain(row) if row else None

    def _row_to_domain(self, row) -> User:
        """Explicit Row → Domain conversion."""
        return User(
            id=row["id"],
            name=row["name"],
            email=row["email"],
            age=row["age"],
            tags=tuple(row["tags"]),  # DB array → tuple
            created_at=row["created_at"]
        )
```

**Key principle**: Repository adapters handle SQL ↔ Domain translation. Domain never imports SQLAlchemy. Repository holds `self.session` and all methods use it.

---

## Exception Design (Unified)

```python
# domain/errors.py
class DomainError(Exception):
    """Base for all domain errors."""

class ValidationError(DomainError):
    """Domain validation failure."""
    def __init__(self, errors: tuple[str, ...] | str):
        msg = "; ".join(errors) if isinstance(errors, tuple) else errors
        super().__init__(msg)
        self.errors = errors if isinstance(errors, tuple) else (errors,)

class NotFoundError(DomainError): pass
class ConflictError(DomainError): pass
class BusinessRuleViolation(DomainError): pass

# api/errors.py - Map to HTTP
from fastapi import Request
from fastapi.responses import JSONResponse
from domain.errors import NotFoundError, ConflictError, ValidationError

@app.exception_handler(NotFoundError)
async def not_found_handler(request: Request, exc: NotFoundError):
    return JSONResponse(status_code=404, content={"error": str(exc)})

@app.exception_handler(ConflictError)
async def conflict_handler(request: Request, exc: ConflictError):
    return JSONResponse(status_code=409, content={"error": str(exc)})

@app.exception_handler(ValidationError)
async def validation_handler(request: Request, exc: ValidationError):
    return JSONResponse(status_code=422, content={"errors": exc.errors})
```

---

## Transaction Management (Unit of Work)

**Use case controls transaction via UoW. Repository does NOT commit (only execute).**

```python
# usecases/shared/ports.py
from typing import Protocol

class UnitOfWork(Protocol):
    """Transaction boundary. Use case controls commit/rollback."""
    async def __aenter__(self) -> "UnitOfWork": ...
    async def __aexit__(self, exc_type, exc_val, exc_tb) -> None: ...

# adapters/db/uow.py
from sqlalchemy.ext.asyncio import AsyncSession

class SQLAlchemyUnitOfWork:
    """Commits on success, rolls back on exception."""
    def __init__(self, session: AsyncSession):
        self.session = session

    async def __aenter__(self):
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None:
            await self.session.commit()
        else:
            await self.session.rollback()
```

**NOTE**: Repository methods execute queries but do NOT commit. UoW handles commit/rollback at use case boundary.

---

## DateTime Standards

**Always timezone-aware UTC. Domain uses Clock port.**

```python
# shared/clock.py
from datetime import datetime, timezone

class SystemClock:
    def now(self) -> datetime:
        return datetime.now(timezone.utc)

class FixedClock:
    def __init__(self, fixed_time: datetime):
        if fixed_time.tzinfo is None:
            raise ValueError("Must be timezone-aware")
        self._time = fixed_time
    def now(self) -> datetime:
        return self._time

# API validation (Pydantic v2)
from pydantic import field_validator

@field_validator("end_date")
@classmethod
def end_date_must_be_future(cls, v: datetime) -> datetime:
    now = datetime.now(timezone.utc)
    if v.tzinfo is None:
        v = v.replace(tzinfo=timezone.utc)
    if v <= now:
        raise ValueError("end_date must be in the future")
    return v
```

---

## DTO vs Domain

**API DTOs**: Pydantic `BaseModel`, `list`/`dict` for JSON. Validation-focused.
**Domain Models**: `@dataclass(frozen=True)`, `tuple`/`frozenset`. Business logic-focused.

**DTO Policy**: Use `model_config = {"frozen": True}` for Pydantic v2 request DTOs to prevent accidental mutation after validation. **Limitation**: Does NOT provide true immutability—only prevents attribute assignment. Internal `list`/`dict` fields remain mutable.

```python
# api/schemas/user.py (DTO)
from pydantic import BaseModel, Field, EmailStr

class CreateUserRequest(BaseModel):
    name: str = Field(min_length=1, max_length=100)
    email: EmailStr
    age: int = Field(ge=0, le=150)
    tags: list[str] = Field(default_factory=list)
    model_config = {"frozen": True}

class UserResponse(BaseModel):
    id: str
    name: str
    email: str
    age: int
    tags: list[str]
    created_at: str  # ISO 8601 format

# domain/models.py (Domain)
from dataclasses import dataclass
from datetime import datetime

@dataclass(frozen=True)
class NewUser:
    """User before persistence - id and created_at not yet set."""
    name: str
    email: str
    age: int
    tags: tuple[str, ...]

@dataclass(frozen=True)
class User:
    """User after persistence - fully populated with id and created_at."""
    id: str
    name: str
    email: str
    age: int
    tags: tuple[str, ...]
    created_at: datetime  # Always set (NOT NULL in DB)

# api/adapters/user_adapter.py (Conversion)
from api.schemas.user import CreateUserRequest, UserResponse
from domain.models import NewUser, User

def to_domain_new_user(req: CreateUserRequest) -> NewUser:
    """DTO → NewUser (before persistence). No id/created_at yet."""
    return NewUser(
        name=req.name,
        email=req.email,
        age=req.age,
        tags=tuple(req.tags),
    )

def from_domain_user(user: User) -> UserResponse:
    """User (after persistence) → DTO. All fields guaranteed non-null."""
    return UserResponse(
        id=user.id,
        name=user.name,
        email=user.email,
        age=user.age,
        tags=list(user.tags),
        created_at=user.created_at.isoformat()  # Always present
    )
```

**NOTE**: Separate types for pre/post-persistence state. `NewUser` has no `id`/`created_at`. Use case converts `NewUser` → `User` (with id/created_at from IdGenerator/Clock) before calling Repository. This ensures type safety and eliminates `None` handling in persistence layer.

---

## Complete Flow: Create User (SQLAlchemy Core)

### 1. API Layer

```python
# api/routes/users.py
from fastapi import APIRouter, Depends, status
from api.schemas.user import CreateUserRequest, UserResponse
from api.adapters.user_adapter import to_domain_new_user, from_domain_user
from api.deps import get_create_user_use_case
from usecases.user.create_user import CreateUserUseCase

router = APIRouter()

@router.post("/users", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(
    request: CreateUserRequest,
    use_case: CreateUserUseCase = Depends(get_create_user_use_case)
):
    new_user = to_domain_new_user(request)
    created_user = await use_case.execute(new_user)
    return from_domain_user(created_user)
```

### 2. Use Case (Orchestration + Transaction)

```python
# usecases/user/create_user.py
from domain.models import NewUser, User
from domain.errors import ValidationError, ConflictError
from domain.services.user_rules import validate_new_user
from usecases.user.ports import UserRepository
from usecases.shared.ports import UnitOfWork, Clock, IdGenerator

class CreateUserUseCase:
    def __init__(self, repo: UserRepository, uow: UnitOfWork,
                 clock: Clock, id_gen: IdGenerator):
        self.repo, self.uow, self.clock, self.id_gen = repo, uow, clock, id_gen

    async def execute(self, new_user: NewUser) -> User:
        """Accept NewUser (before persistence), return User (after persistence)."""
        async with self.uow:
            # Pure: Validate
            errors = validate_new_user(new_user)
            if errors:
                raise ValidationError(errors)

            # I/O: Check uniqueness
            if await self.repo.find_by_email(new_user.email):
                raise ConflictError(f"Email {new_user.email} exists")

            # Pure: Convert NewUser → User with id/created_at
            user = User(
                id=self.id_gen.generate(),
                name=new_user.name,
                email=new_user.email,
                age=new_user.age,
                tags=new_user.tags,
                created_at=self.clock.now()
            )

            # I/O: Persist (repository receives fully populated User)
            return await self.repo.create(user)
```

### 3. Ports (Protocol)

```python
# usecases/user/ports.py
from typing import Protocol
from domain.models import User

class UserRepository(Protocol):
    async def find_by_id(self, user_id: str) -> User | None: ...
    async def find_by_email(self, email: str) -> User | None: ...
    async def create(self, user: User) -> User: ...
    async def update(self, user: User) -> User: ...  # Uses user.id (no duplicate id parameter)

# usecases/shared/ports.py (shared ports)
from typing import Protocol
from datetime import datetime

class Clock(Protocol):
    def now(self) -> datetime: ...

class IdGenerator(Protocol):
    def generate(self) -> str: ...

class UnitOfWork(Protocol):
    """Transaction boundary. Defined once in shared/ports.py."""
    async def __aenter__(self) -> "UnitOfWork": ...
    async def __aexit__(self, exc_type, exc_val, exc_tb) -> None: ...
```

### 4. Domain (Pure Logic)

```python
# domain/models.py
from dataclasses import dataclass
from datetime import datetime

@dataclass(frozen=True)
class NewUser:
    """User before persistence - id and created_at not yet set."""
    name: str
    email: str
    age: int
    tags: tuple[str, ...]

@dataclass(frozen=True)
class User:
    """User after persistence - fully populated."""
    id: str
    name: str
    email: str
    age: int
    tags: tuple[str, ...]
    created_at: datetime  # Always set (NOT NULL in DB)

# domain/services/user_rules.py
from domain.models import NewUser

def validate_new_user(new_user: NewUser) -> tuple[str, ...]:
    """Pure function: no I/O, same input → same output."""
    errors = []
    if not new_user.name.strip():
        errors.append("Name cannot be empty")
    if new_user.age < 0 or new_user.age > 150:
        errors.append("Age must be 0-150")
    if "@" not in new_user.email:
        errors.append("Invalid email")
    return tuple(errors)
```

### 5. Adapter (Infrastructure - SQLAlchemy Core)

```python
# adapters/db/tables.py (Schema definition with Core)
from sqlalchemy import MetaData, Table, Column, String, Integer, TIMESTAMP
from sqlalchemy.dialects.postgresql import ARRAY

metadata = MetaData()

users = Table(
    "users",
    metadata,
    Column("id", String, primary_key=True),
    Column("name", String, nullable=False),
    Column("email", String, unique=True, nullable=False),
    Column("age", Integer, nullable=False),
    Column("tags", ARRAY(String), nullable=False),
    Column("created_at", TIMESTAMP(timezone=True), nullable=False),
)

# adapters/db/user_repo.py (Repository using Core)
from sqlalchemy import select, insert, update
from sqlalchemy.ext.asyncio import AsyncSession
from domain.models import User
from domain.errors import NotFoundError
from adapters.db.tables import users

class SQLAlchemyUserRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def find_by_id(self, user_id: str) -> User | None:
        query = select(users).where(users.c.id == user_id)
        result = await self.session.execute(query)
        row = result.mappings().one_or_none()
        return self._row_to_domain(row) if row else None

    async def find_by_email(self, email: str) -> User | None:
        query = select(users).where(users.c.email == email)
        result = await self.session.execute(query)
        row = result.mappings().one_or_none()
        return self._row_to_domain(row) if row else None

    async def create(self, user: User) -> User:
        """Insert and return created user using RETURNING.

        NOTE: RETURNING is supported in PostgreSQL (all versions), MySQL 8.0.1+,
        SQLite 3.35+. For older databases or drivers without RETURNING support,
        use INSERT followed by SELECT (see create_without_returning example).
        """
        query = (
            insert(users)
            .values(
                id=user.id,
                name=user.name,
                email=user.email,
                age=user.age,
                tags=list(user.tags),  # tuple → list for DB
                created_at=user.created_at
            )
            .returning(users)
        )
        result = await self.session.execute(query)
        row = result.mappings().one()
        return self._row_to_domain(row)

    async def update(self, user: User) -> User:
        """Update and return updated user using RETURNING. Uses user.id."""
        query = (
            update(users)
            .where(users.c.id == user.id)
            .values(
                name=user.name,
                email=user.email,
                age=user.age,
                tags=list(user.tags)
            )
            .returning(users)
        )
        result = await self.session.execute(query)
        row = result.mappings().one_or_none()
        if not row:
            raise NotFoundError(f"User {user.id} not found")
        return self._row_to_domain(row)

    def _row_to_domain(self, row) -> User:
        """Convert SQLAlchemy Row/Mapping to Domain model."""
        return User(
            id=row["id"],
            name=row["name"],
            email=row["email"],
            age=row["age"],
            tags=tuple(row["tags"]),  # list → tuple
            created_at=row["created_at"]
        )

    async def create_without_returning(self, user: User) -> User:
        """Alternative for databases/drivers without RETURNING support.

        Uses INSERT then SELECT pattern. Slightly less efficient but
        compatible with older database versions.
        """
        query = insert(users).values(
            id=user.id, name=user.name, email=user.email,
            age=user.age, tags=list(user.tags), created_at=user.created_at
        )
        await self.session.execute(query)
        # Fetch inserted row
        return await self.find_by_id(user.id)
```

### 6. DI Wiring

```python
# api/deps.py
from typing import AsyncGenerator
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession
from adapters.db.session import async_session_maker
from usecases.user.ports import UserRepository
from usecases.shared.ports import UnitOfWork, Clock, IdGenerator
from adapters.db.user_repo import SQLAlchemyUserRepository
from adapters.db.uow import SQLAlchemyUnitOfWork
from shared.clock import SystemClock
from shared.idgen import UUIDGenerator
from usecases.user.create_user import CreateUserUseCase

# NOTE: FastAPI dependency injection caches results per request by default.
# When multiple dependencies call Depends(get_db), FastAPI returns the SAME
# AsyncSession instance for that request, ensuring:
# 1 HTTP request = 1 session = 1 transaction boundary.
# To disable caching, use Depends(get_db, use_cache=False).

async def get_db() -> AsyncGenerator[AsyncSession, None]:
    """Single source of AsyncSession per request."""
    async with async_session_maker() as session:
        yield session

async def get_user_repository(db: AsyncSession = Depends(get_db)) -> UserRepository:
    """Repository shares session from get_db."""
    return SQLAlchemyUserRepository(db)

async def get_uow(db: AsyncSession = Depends(get_db)) -> UnitOfWork:
    """UoW shares SAME session from get_db (cached by FastAPI)."""
    return SQLAlchemyUnitOfWork(db)

def get_clock() -> Clock:
    return SystemClock()

def get_id_generator() -> IdGenerator:
    return UUIDGenerator()

async def get_create_user_use_case(
    repo: UserRepository = Depends(get_user_repository),
    uow: UnitOfWork = Depends(get_uow),
    clock: Clock = Depends(get_clock),
    id_gen: IdGenerator = Depends(get_id_generator)
) -> CreateUserUseCase:
    return CreateUserUseCase(repo, uow, clock, id_gen)
```

---

## Repository Pattern (SQLAlchemy Core)

```python
# domain/models.py (add Market)
from dataclasses import dataclass
from datetime import datetime

@dataclass(frozen=True)
class NewMarket:
    """Market before persistence - id and created_at not yet set."""
    name: str
    description: str | None
    status: str

@dataclass(frozen=True)
class Market:
    """Market after persistence - fully populated."""
    id: str
    name: str
    description: str | None
    status: str
    created_at: datetime  # Always set (NOT NULL in DB)

# usecases/market/ports.py (Port - Protocol)
from typing import Protocol
from domain.models import Market

class MarketRepository(Protocol):
    async def find_all(self, filters: dict | None = None) -> tuple[Market, ...]: ...
    async def find_by_id(self, market_id: str) -> Market | None: ...
    async def create(self, market: Market) -> Market: ...
    async def update(self, market: Market) -> Market: ...

# domain/services/market_rules.py (Pure validation)
from domain.models import NewMarket

def validate_new_market(new_market: NewMarket) -> tuple[str, ...]:
    """Pure function: no I/O, same input → same output."""
    errors = []
    if not new_market.name.strip():
        errors.append("Market name cannot be empty")
    if new_market.status not in ("active", "inactive", "archived"):
        errors.append("Status must be one of: active, inactive, archived")
    return tuple(errors)

# usecases/market/create_market.py (Use Case example)
from domain.models import NewMarket, Market
from domain.errors import ValidationError
from domain.services.market_rules import validate_new_market
from usecases.market.ports import MarketRepository
from usecases.shared.ports import UnitOfWork, Clock, IdGenerator

class CreateMarketUseCase:
    def __init__(self, repo: MarketRepository, uow: UnitOfWork,
                 clock: Clock, id_gen: IdGenerator):
        self.repo, self.uow, self.clock, self.id_gen = repo, uow, clock, id_gen

    async def execute(self, new_market: NewMarket) -> Market:
        """Accept NewMarket (before persistence), return Market (after persistence)."""
        async with self.uow:
            # Pure: Validate
            errors = validate_new_market(new_market)
            if errors:
                raise ValidationError(errors)

            # Pure: Convert NewMarket → Market with id/created_at
            market = Market(
                id=self.id_gen.generate(),
                name=new_market.name,
                description=new_market.description,
                status=new_market.status,
                created_at=self.clock.now()
            )

            # I/O: Persist
            return await self.repo.create(market)

# adapters/db/tables.py (add Market table)
from sqlalchemy import MetaData, Table, Column, String, TIMESTAMP

# metadata is shared across all tables (defined once in tables.py)
# from adapters.db.tables import metadata

markets = Table(
    "markets",
    metadata,  # Uses the same metadata instance from tables.py
    Column("id", String, primary_key=True),
    Column("name", String, nullable=False),
    Column("description", String, nullable=True),
    Column("status", String, nullable=False),
    Column("created_at", TIMESTAMP(timezone=True), nullable=False),
)

# adapters/db/market_repo.py (Adapter - Core implementation)
from sqlalchemy import select, insert, update
from sqlalchemy.ext.asyncio import AsyncSession
from domain.models import Market
from domain.errors import NotFoundError
from adapters.db.tables import markets

class SQLAlchemyMarketRepository:
    def __init__(self, session: AsyncSession):
        self.session = session

    async def find_all(self, filters: dict | None = None) -> tuple[Market, ...]:
        query = select(markets)
        if filters:
            if "status" in filters:
                query = query.where(markets.c.status == filters["status"])
        result = await self.session.execute(query)
        rows = result.mappings().all()
        return tuple(self._row_to_domain(r) for r in rows)

    async def find_by_id(self, market_id: str) -> Market | None:
        query = select(markets).where(markets.c.id == market_id)
        result = await self.session.execute(query)
        row = result.mappings().one_or_none()
        return self._row_to_domain(row) if row else None

    async def create(self, market: Market) -> Market:
        query = (
            insert(markets)
            .values(
                id=market.id,
                name=market.name,
                description=market.description,
                status=market.status,
                created_at=market.created_at
            )
            .returning(markets)
        )
        result = await self.session.execute(query)
        row = result.mappings().one()
        return self._row_to_domain(row)

    async def update(self, market: Market) -> Market:
        """Update and return updated market using RETURNING. Uses market.id."""
        query = (
            update(markets)
            .where(markets.c.id == market.id)
            .values(name=market.name, description=market.description, status=market.status)
            .returning(markets)
        )
        result = await self.session.execute(query)
        row = result.mappings().one_or_none()
        if not row:
            raise NotFoundError(f"Market {market.id} not found")
        return self._row_to_domain(row)

    def _row_to_domain(self, row) -> Market:
        return Market(
            id=row["id"],
            name=row["name"],
            description=row["description"],
            status=row["status"],
            created_at=row["created_at"]
        )
```

---

## ⚠️ Critical: AsyncSession Concurrency

**NEVER use `asyncio.gather()` with same `AsyncSession`** - causes race conditions.

### ✅ GOOD: Single query with explicit JOIN

```python
# domain/models.py (add Position)
from dataclasses import dataclass

@dataclass(frozen=True)
class Position:
    id: str
    market_id: str
    user_id: str
    shares: int

# adapters/db/tables.py (add Position table)
from sqlalchemy import MetaData, Table, Column, String, Integer

positions = Table(
    "positions",
    metadata,
    Column("id", String, primary_key=True),
    Column("market_id", String, nullable=False),
    Column("user_id", String, nullable=False),
    Column("shares", Integer, nullable=False),
)

# adapters/db/market_repo.py
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from adapters.db.tables import markets, positions
from domain.models import Market

async def get_markets_with_positions(
    session: AsyncSession, user_id: str
) -> tuple[Market, ...]:
    """Get markets with user's positions using explicit JOIN."""
    query = (
        select(markets)
        .join(positions, markets.c.id == positions.c.market_id)
        .where(markets.c.status == "active")
        .where(positions.c.user_id == user_id)
        .distinct()
    )
    result = await session.execute(query)
    rows = result.mappings().all()
    return tuple(_row_to_market(r) for r in rows)

def _row_to_market(row) -> Market:
    """Convert Row/Mapping to Market domain model."""
    return Market(
        id=row["id"],
        name=row["name"],
        description=row["description"],
        status=row["status"],
        created_at=row["created_at"]
    )
```

### ✅ GOOD: Sequential queries with IN clause

```python
from dataclasses import dataclass
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from adapters.db.tables import markets, positions
from domain.models import Market, Position

@dataclass(frozen=True)
class MarketWithPositions:
    """Aggregate result combining Market with user's Positions."""
    market: Market
    positions: tuple[Position, ...]

async def get_user_markets_with_positions(
    session: AsyncSession, user_id: str
) -> tuple[MarketWithPositions, ...]:
    """Fetch markets and positions sequentially, then merge in pure code.

    Returns type-safe aggregate with Market + Positions instead of dict.

    NOTE: IN clause considerations:
    - Deduplication: Use dict.fromkeys(market_ids) to preserve order
    - Large lists: For very large lists (>1000 items), test performance and consider
      alternatives like JOIN, CTE, or temp tables depending on your DB and use case
    - Performance varies by database engine, query planner, and data distribution
    """
    # 1. Get market IDs where user has positions
    pos_query = select(positions.c.market_id).where(positions.c.user_id == user_id)
    pos_result = await session.execute(pos_query)
    market_ids = [row["market_id"] for row in pos_result.mappings().all()]

    if not market_ids:
        return ()

    # Deduplicate while preserving order
    unique_market_ids = list(dict.fromkeys(market_ids))

    # 2. Get markets by IDs (single query with IN)
    market_query = select(markets).where(markets.c.id.in_(unique_market_ids))
    market_result = await session.execute(market_query)
    market_rows = market_result.mappings().all()

    # 3. Get all positions for those markets
    all_pos_query = (
        select(positions)
        .where(positions.c.market_id.in_(unique_market_ids))
        .where(positions.c.user_id == user_id)
    )
    all_pos_result = await session.execute(all_pos_query)
    pos_rows = all_pos_result.mappings().all()

    # 4. Merge in pure Python (functional composition) with type-safe dataclass
    markets_dict = {r["id"]: r for r in market_rows}
    positions_by_market = {}
    for p in pos_rows:
        positions_by_market.setdefault(p["market_id"], []).append(p)

    return tuple(
        MarketWithPositions(
            market=_row_to_market(markets_dict[mid]),
            positions=tuple(_row_to_position(p) for p in positions_by_market.get(mid, []))
        )
        for mid in unique_market_ids if mid in markets_dict
    )

def _row_to_position(row) -> Position:
    """Convert Row/Mapping to Position domain model."""
    return Position(
        id=row["id"],
        market_id=row["market_id"],
        user_id=row["user_id"],
        shares=row["shares"]
    )
```

### ❌ BAD: Parallel with same session

```python
# ❌ NEVER DO THIS - Race condition!
import asyncio

markets_result, positions_result = await asyncio.gather(
    session.execute(markets_query),  # ❌ Same session
    session.execute(positions_query)  # ❌ Same session - CORRUPTS STATE
)
```

---

## Security Best Practices

```python
# ✅ GOOD Option A: Parameterized queries with Table (recommended)
from sqlalchemy import select
from sqlalchemy.ext.asyncio import AsyncSession
from adapters.db.tables import users

async def get_user_via_table(user_id: str, session: AsyncSession) -> dict | None:
    """Recommended: Use Table with parameterized where clause."""
    query = select(users).where(users.c.id == user_id)
    result = await session.execute(query)
    row = result.mappings().one_or_none()
    return dict(row) if row else None

# ✅ GOOD Option B: Raw SQL with text() and parameters
from sqlalchemy import text

async def get_user_via_text(user_id: str, session: AsyncSession) -> dict | None:
    """Alternative: Raw SQL with bound parameters (still safe)."""
    result = await session.execute(
        text("SELECT * FROM users WHERE id = :user_id"),
        {"user_id": user_id}
    )
    row = result.mappings().one_or_none()
    return dict(row) if row else None

# ❌ BAD: SQL injection
async def get_user_unsafe(user_id: str, session: AsyncSession):
    query = f"SELECT * FROM users WHERE id = '{user_id}'"  # NEVER DO THIS
    # Attacker can inject: user_id = "'; DROP TABLE users; --"

# ✅ GOOD: Safe subprocess
import subprocess

def list_files_safe(filename: str):
    subprocess.run(["ls", "-l", filename], capture_output=True, check=True)

# ❌ BAD: Command injection
def list_files_unsafe(filename: str):
    subprocess.run(f"ls -l {filename}", shell=True)  # NEVER DO THIS
    # Attacker can inject: filename = "file.txt; rm -rf /"

# ✅ GOOD: Secrets from environment
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    database_url: str
    api_key: str
    model_config = {"env_file": ".env"}
```

---

## Async HTTP Patterns

```python
# ✅ GOOD: All async with timeout
import httpx
import asyncio
from pydantic import BaseModel

class UserPayload(BaseModel):
    id: str
    name: str

async def fetch_users(base_url: str) -> tuple[UserPayload, ...]:
    timeout = httpx.Timeout(10.0, connect=5.0)
    async with httpx.AsyncClient(base_url=base_url, timeout=timeout) as client:
        response = await client.get("/users")
        response.raise_for_status()
        return tuple(UserPayload(**u) for u in response.json())

# ✅ GOOD: Parallel requests (OK - different HTTP clients, NOT same AsyncSession)
async def fetch_multiple(user_id: str, base_url: str) -> dict:
    async with httpx.AsyncClient(base_url=base_url, timeout=httpx.Timeout(10.0)) as client:
        user_res, orders_res = await asyncio.gather(
            client.get(f"/api/users/{user_id}"),
            client.get(f"/api/orders?user_id={user_id}")
        )
        return {"user": user_res.json(), "orders": orders_res.json()}
```

---

## Testing Patterns

### Domain Tests (Pure - No Mocks)

```python
# tests/domain/test_user_rules.py
from domain.models import NewUser
from domain.services.user_rules import validate_new_user

def test_validate_new_user_success():
    new_user = NewUser(
        name="Alice",
        email="alice@example.com",
        age=25,
        tags=("premium",)
    )
    assert validate_new_user(new_user) == ()

def test_validate_new_user_invalid():
    new_user = NewUser(
        name="",
        email="invalid",
        age=200,
        tags=()
    )
    errors = validate_new_user(new_user)
    assert "Name cannot be empty" in errors
    assert "Invalid email" in errors
```

### Use Case Tests (With Test Doubles)

```python
# tests/usecases/test_create_user.py
import pytest
from datetime import datetime, timezone
from domain.models import NewUser, User
from usecases.user.create_user import CreateUserUseCase
from shared.clock import FixedClock
from shared.idgen import SequentialIdGen

class MockUserRepository:
    def __init__(self):
        self.users = {}
    async def find_by_email(self, email: str) -> User | None:
        return next((u for u in self.users.values() if u.email == email), None)
    async def create(self, user: User) -> User:
        self.users[user.id] = user
        return user

class MockUnitOfWork:
    def __init__(self):
        self.committed = False
    async def __aenter__(self):
        return self
    async def __aexit__(self, exc_type, exc_val, exc_tb):
        if exc_type is None:
            self.committed = True

@pytest.mark.asyncio
async def test_create_user_success():
    repo = MockUserRepository()
    uow = MockUnitOfWork()
    clock = FixedClock(datetime(2024, 1, 1, tzinfo=timezone.utc))
    id_gen = SequentialIdGen()
    use_case = CreateUserUseCase(repo, uow, clock, id_gen)

    # NewUser: before persistence (no id/created_at yet)
    new_user = NewUser(
        name="Alice",
        email="alice@example.com",
        age=25,
        tags=("premium",)
    )
    created = await use_case.execute(new_user)

    assert created.id == "user-1"
    assert created.email == "alice@example.com"
    assert created.created_at == datetime(2024, 1, 1, tzinfo=timezone.utc)
    assert uow.committed is True
```

### Integration Tests

```python
# tests/api/test_users_integration.py
import pytest
from httpx import AsyncClient, ASGITransport
from main import app

@pytest.fixture
async def async_client():
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        yield client

@pytest.mark.asyncio
async def test_create_user_integration(async_client: AsyncClient):
    response = await async_client.post("/api/users", json={
        "name": "Alice",
        "email": "alice@example.com",
        "age": 25,
        "tags": ["premium"]
    })
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "Alice"
    assert data["age"] == 25
```

---

## Database Session Setup (SQLAlchemy Core)

```python
# adapters/db/session.py
from sqlalchemy.ext.asyncio import create_async_engine, async_sessionmaker, AsyncSession
from shared.settings import settings

# Create async engine
engine = create_async_engine(
    settings.database_url,
    echo=settings.debug,
    pool_pre_ping=True,
    pool_size=10,
    max_overflow=20
)

# Create session factory
async_session_maker = async_sessionmaker(
    engine,
    class_=AsyncSession,
    expire_on_commit=False
)
```

---

## Alembic Migrations (Core)

```python
# adapters/db/migrations/env.py
from alembic import context
from sqlalchemy import pool
from sqlalchemy.ext.asyncio import async_engine_from_config
from adapters.db.tables import metadata  # Import metadata, not Base

# Set target metadata for autogenerate
target_metadata = metadata

# ... rest of Alembic config
```

---

## Recommended Libraries

**API**: FastAPI, pydantic v2, pydantic-settings
**DB**: SQLAlchemy 2.0 Core (NO ORM), alembic, asyncpg
**DI/Ports**: typing.Protocol, (optional) dependency-injector
**Utils**: returns, structlog, httpx, tenacity, orjson
**Jobs**: Celery+Redis, ARQ, Dramatiq, AWS SQS
**Test**: pytest, pytest-asyncio, httpx (ASGITransport), faker

---

## Shared Utilities Implementation

```python
# shared/clock.py
from datetime import datetime, timezone
from usecases.shared.ports import Clock

class SystemClock:
    """Production clock - returns current UTC time."""
    def now(self) -> datetime:
        return datetime.now(timezone.utc)

class FixedClock:
    """Test clock - returns fixed time for deterministic tests."""
    def __init__(self, fixed_time: datetime):
        if fixed_time.tzinfo is None:
            raise ValueError("Must be timezone-aware")
        self._time = fixed_time

    def now(self) -> datetime:
        return self._time

# shared/idgen.py
import uuid
from usecases.shared.ports import IdGenerator

class UUIDGenerator:
    """Production ID generator - generates UUID v4."""
    def generate(self) -> str:
        return str(uuid.uuid4())

class SequentialIdGen:
    """Test ID generator - generates sequential IDs for deterministic tests."""
    def __init__(self, prefix: str = "user"):
        self.prefix = prefix
        self.counter = 0

    def generate(self) -> str:
        self.counter += 1
        return f"{self.prefix}-{self.counter}"
```

---

## Key Principles (13 Critical Rules)

1. **Domain purity**: `@dataclass(frozen=True)` + `tuple`
2. **3-layer separation**: Domain ≠ DB Tables ≠ API DTOs
3. **Immutable returns**: Return `tuple` from repos/use cases
4. **Decimal for money**: Always `Decimal` in domain
5. **Protocol for ports**: Dependency inversion
6. **Async only**: No sync I/O in async functions
7. **Type safety**: Python 3.11+ `|` syntax, strict pyright
8. **Unidirectional deps**: Outer→inner (via ports)
9. **Test w/o mocks**: Domain testable without mocking
10. **UoW for TX**: Use case controls transaction boundaries
11. **Timezone-aware**: Always UTC, `datetime.now(timezone.utc)`
12. **NO AsyncSession∥**: Never same session for parallel queries
13. **Explicit SQL→Domain**: Use Core, no ORM stateful entities

---

## ⚠️ Critical Warnings

**SQLAlchemy Core**: Use `Table` + `MetaData`, NOT ORM (`DeclarativeBase`, `relationship()`). Explicit Row → Domain conversion.

**AsyncSession**: NEVER `asyncio.gather()` with same session. Use JOIN or sequential queries with IN clause.

**Timezone**: ALWAYS timezone-aware datetime. Use `datetime.now(timezone.utc)`.

**UoW**: Transaction boundary at use case. Repos execute queries, UoW commits.

**Repository Pattern**: Repository holds `self.session: AsyncSession`. All methods use `self.session.execute()`.

**RETURNING Clause**: Supported in PostgreSQL (all versions), MySQL 8.0.1+, SQLite 3.35+. For older databases or drivers, use INSERT then SELECT.

**DI Session Sharing**: FastAPI caches dependency results per request by default. All dependencies using `Depends(get_db)` receive the SAME AsyncSession instance within a request. Use `use_cache=False` to disable caching if needed.

**IN Clause**: Use `list(dict.fromkeys(ids))` for order-preserving deduplication. For large lists, test performance and consider JOIN, CTE, or temp tables as alternatives.

**DTO frozen**: `model_config = {"frozen": True}` prevents attribute reassignment but NOT mutation of list/dict fields.

**Domain Type Safety**: Use separate types for pre/post-persistence (e.g., `NewUser` without id/created_at, `User` with non-null id/created_at). UseCase converts NewUser → User with Clock/IdGenerator before calling Repository.

---

**Remember**: This mirrors Clojure's philosophy—data-centric, pure core, orchestration in use cases, side effects at adapter boundaries. SQLAlchemy Core provides explicit SQL control while maintaining safety and portability. Follow [coding-standards-python.md](coding-standards-python.md) for functional programming principles.
