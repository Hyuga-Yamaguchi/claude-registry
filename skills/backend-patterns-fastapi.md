---
name: backend-patterns-fastapi
description: Backend architecture patterns and best practices for FastAPI. Type-safe async APIs, dependency injection, Pydantic models, and database integration.
---

# FastAPI Backend Patterns

Backend architecture patterns for FastAPI applications with async/await, Pydantic validation, and modern Python practices.

## Architecture Principles

### Separate Pure Logic from I/O

Keep business logic pure and free from I/O dependencies. Push side effects to the boundaries.

```python
from decimal import Decimal
from pydantic import BaseModel

# Pure domain logic
def calculate_discount(price: Decimal, user_tier: str) -> Decimal:
    if user_tier == "premium":
        return price * Decimal("0.9")
    return price

# I/O at boundaries
from dataclasses import replace

async def apply_discount_to_order(
    order_id: str,
    repository: OrderRepository
) -> Order:
    order = await repository.get(order_id)  # I/O
    discounted_price = calculate_discount(order.price, order.user.tier)  # Pure
    updated_order = order.copy(update={"price": discounted_price})  # Immutable update
    await repository.save(updated_order)  # I/O
    return updated_order
```

### Domain Layer Must Not Depend on Infrastructure

Domain models and business logic MUST NOT import database, HTTP, or framework code.

```python
from pydantic import BaseModel
from decimal import Decimal
from typing import List

# ✅ GOOD: Pure domain models
class OrderItem(BaseModel):
    product_id: str
    quantity: int
    price: Decimal

    class Config:
        frozen = True

class Order(BaseModel):
    id: str
    items: List[OrderItem]
    total: Decimal

    class Config:
        frozen = True

    def add_item(self, item: OrderItem) -> "Order":
        new_total = self.total + (item.price * item.quantity)
        return self.copy(update={
            "items": [*self.items, item],
            "total": new_total
        })

# ❌ BAD: Domain model depending on infrastructure
from sqlalchemy.orm import Session

class Order(BaseModel):
    # Domain logic should not know about database sessions
    def save(self, db: Session) -> None:
        db.add(self)
        db.commit()
```

### Avoid Passing Raw Dicts Across Boundaries

Use Pydantic models for data crossing layer boundaries.

```python
from pydantic import BaseModel

# ✅ GOOD: Typed models
class CreateUserRequest(BaseModel):
    name: str
    email: str
    role: str = "user"

class User(BaseModel):
    id: str
    name: str
    email: str
    role: str

async def create_user(request: CreateUserRequest) -> User:
    # Type-safe
    pass

# ❌ BAD: Using raw dicts
async def create_user_bad(data: dict) -> dict:
    # Unsafe - no type checking
    pass
```

### Side Effects Must Be Explicit

Functions with side effects SHOULD indicate this in naming or return type.

```python
# ✅ GOOD: Clear side effect in name
async def send_email_notification(user_id: str) -> None:
    pass

async def save_user_to_database(user: User) -> None:
    pass

# ✅ GOOD: Pure function - no side effects
def format_email_body(user: User) -> str:
    pass

def calculate_total_price(items: List[OrderItem]) -> Decimal:
    pass
```

## API Design with FastAPI

### RESTful Route Structure

```python
from fastapi import FastAPI, HTTPException, Query, Path
from pydantic import BaseModel
from typing import List, Optional

app = FastAPI()

# ✅ GOOD: Resource-based routing
@app.get("/api/markets")
async def get_markets(
    status: Optional[str] = Query(None),
    limit: int = Query(10, ge=1, le=100),
    offset: int = Query(0, ge=0)
):
    """List all markets with filtering and pagination."""
    markets = await market_service.find_all(status=status, limit=limit, offset=offset)
    return {"success": True, "data": markets}

@app.get("/api/markets/{market_id}")
async def get_market(market_id: str = Path(..., min_length=1)):
    """Get a single market by ID."""
    market = await market_service.find_by_id(market_id)
    if not market:
        raise HTTPException(status_code=404, detail="Market not found")
    return {"success": True, "data": market}

@app.post("/api/markets")
async def create_market(market: CreateMarketRequest):
    """Create a new market."""
    created = await market_service.create(market)
    return {"success": True, "data": created}

@app.put("/api/markets/{market_id}")
async def update_market(market_id: str, market: UpdateMarketRequest):
    """Update an existing market."""
    updated = await market_service.update(market_id, market)
    return {"success": True, "data": updated}

@app.delete("/api/markets/{market_id}")
async def delete_market(market_id: str):
    """Delete a market."""
    await market_service.delete(market_id)
    return {"success": True}
```

### Pydantic Models for Validation

```python
from pydantic import BaseModel, EmailStr, Field, validator
from typing import Optional, List
from datetime import datetime
from enum import Enum

# ✅ GOOD: Strict type definitions
class MarketStatus(str, Enum):
    ACTIVE = "active"
    RESOLVED = "resolved"
    CLOSED = "closed"

class CreateMarketRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=200)
    description: str = Field(..., min_length=1, max_length=2000)
    end_date: datetime
    categories: List[str] = Field(..., min_items=1)

    @validator("end_date")
    def end_date_must_be_future(cls, v):
        if v <= datetime.now():
            raise ValueError("end_date must be in the future")
        return v

class Market(BaseModel):
    id: str
    name: str
    description: str
    status: MarketStatus
    volume: float = 0.0
    created_at: datetime

    class Config:
        orm_mode = True  # Allow creation from ORM models

# ❌ BAD: Using dicts without validation
@app.post("/api/markets/bad")
async def create_market_bad(market: dict):
    # No validation, type safety, or documentation
    return await market_service.create(market)
```

## Repository Pattern with Async

```python
from abc import ABC, abstractmethod
from typing import List, Optional
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select, update, delete

# ✅ GOOD: Abstract repository pattern
class MarketRepository(ABC):
    @abstractmethod
    async def find_all(self, filters: Optional[dict] = None) -> List[Market]:
        pass

    @abstractmethod
    async def find_by_id(self, market_id: str) -> Optional[Market]:
        pass

    @abstractmethod
    async def create(self, market_data: CreateMarketRequest) -> Market:
        pass

    @abstractmethod
    async def update(self, market_id: str, market_data: UpdateMarketRequest) -> Market:
        pass

    @abstractmethod
    async def delete(self, market_id: str) -> None:
        pass

# Implementation with SQLAlchemy
class SQLAlchemyMarketRepository(MarketRepository):
    def __init__(self, session: AsyncSession):
        self.session = session

    async def find_all(self, filters: Optional[dict] = None) -> List[Market]:
        query = select(MarketModel)

        if filters:
            if "status" in filters:
                query = query.where(MarketModel.status == filters["status"])
            if "limit" in filters:
                query = query.limit(filters["limit"])
            if "offset" in filters:
                query = query.offset(filters["offset"])

        result = await self.session.execute(query)
        return result.scalars().all()

    async def find_by_id(self, market_id: str) -> Optional[Market]:
        query = select(MarketModel).where(MarketModel.id == market_id)
        result = await self.session.execute(query)
        return result.scalar_one_or_none()

    async def create(self, market_data: CreateMarketRequest) -> Market:
        market = MarketModel(**market_data.dict())
        self.session.add(market)
        await self.session.commit()
        await self.session.refresh(market)
        return market

    async def update(self, market_id: str, market_data: UpdateMarketRequest) -> Market:
        query = (
            update(MarketModel)
            .where(MarketModel.id == market_id)
            .values(**market_data.dict(exclude_unset=True))
            .returning(MarketModel)
        )
        result = await self.session.execute(query)
        await self.session.commit()
        return result.scalar_one()

    async def delete(self, market_id: str) -> None:
        query = delete(MarketModel).where(MarketModel.id == market_id)
        await self.session.execute(query)
        await self.session.commit()
```

## Dependency Injection

```python
from fastapi import Depends
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker

# Database setup
engine = create_async_engine("postgresql+asyncpg://user:pass@localhost/db")
async_session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

# ✅ GOOD: Dependency injection
async def get_db() -> AsyncSession:
    async with async_session() as session:
        yield session

async def get_market_repository(
    db: AsyncSession = Depends(get_db)
) -> MarketRepository:
    return SQLAlchemyMarketRepository(db)

async def get_market_service(
    repo: MarketRepository = Depends(get_market_repository)
) -> MarketService:
    return MarketService(repo)

# Usage in routes
@app.get("/api/markets")
async def get_markets(
    service: MarketService = Depends(get_market_service),
    status: Optional[str] = None
):
    markets = await service.find_all(status=status)
    return {"success": True, "data": markets}

# ✅ GOOD: Authentication dependency
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db)
) -> User:
    credentials_exception = HTTPException(
        status_code=401,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        user_id: str = payload.get("sub")
        if user_id is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception

    user = await user_repository.find_by_id(db, user_id)
    if user is None:
        raise credentials_exception
    return user

@app.get("/api/profile")
async def get_profile(current_user: User = Depends(get_current_user)):
    return {"success": True, "data": current_user}
```

## Service Layer Pattern

```python
from typing import List, Optional

# ✅ GOOD: Service layer with business logic
class MarketService:
    def __init__(self, repository: MarketRepository):
        self.repository = repository

    async def find_all(
        self,
        status: Optional[str] = None,
        limit: int = 10,
        offset: int = 0
    ) -> List[Market]:
        filters = {"limit": limit, "offset": offset}
        if status:
            filters["status"] = status
        return await self.repository.find_all(filters)

    async def find_by_id(self, market_id: str) -> Optional[Market]:
        return await self.repository.find_by_id(market_id)

    async def create(self, market_data: CreateMarketRequest) -> Market:
        # Business logic: validate market doesn't already exist
        existing = await self.repository.find_by_name(market_data.name)
        if existing:
            raise ValueError("Market with this name already exists")

        # Create market with defaults
        return await self.repository.create(market_data)

    async def search_with_embeddings(
        self,
        query: str,
        limit: int = 10
    ) -> List[Market]:
        # Complex business logic
        embedding = await self.embedding_service.generate(query)
        results = await self.vector_search(embedding, limit)
        market_ids = [r["id"] for r in results]
        markets = await self.repository.find_by_ids(market_ids)

        # Sort by similarity score
        score_map = {r["id"]: r["score"] for r in results}
        return sorted(markets, key=lambda m: score_map.get(m.id, 0), reverse=True)
```

## Error Handling Patterns

```python
from fastapi import HTTPException, Request
from fastapi.responses import JSONResponse
from fastapi.exceptions import RequestValidationError

# ✅ GOOD: Custom exception classes
class MarketNotFoundError(Exception):
    def __init__(self, market_id: str):
        self.market_id = market_id
        super().__init__(f"Market {market_id} not found")

class InsufficientPermissionsError(Exception):
    def __init__(self, action: str):
        self.action = action
        super().__init__(f"Insufficient permissions for {action}")

# Global exception handlers
@app.exception_handler(MarketNotFoundError)
async def market_not_found_handler(request: Request, exc: MarketNotFoundError):
    return JSONResponse(
        status_code=404,
        content={"success": False, "error": str(exc)}
    )

@app.exception_handler(InsufficientPermissionsError)
async def permission_error_handler(request: Request, exc: InsufficientPermissionsError):
    return JSONResponse(
        status_code=403,
        content={"success": False, "error": str(exc)}
    )

@app.exception_handler(RequestValidationError)
async def validation_exception_handler(request: Request, exc: RequestValidationError):
    return JSONResponse(
        status_code=400,
        content={
            "success": False,
            "error": "Validation error",
            "details": exc.errors()
        }
    )

@app.exception_handler(Exception)
async def general_exception_handler(request: Request, exc: Exception):
    logger.exception("Unexpected error", exc_info=exc)
    return JSONResponse(
        status_code=500,
        content={"success": False, "error": "Internal server error"}
    )
```

## Middleware Patterns

```python
from fastapi import Request
from time import time
import logging

logger = logging.getLogger(__name__)

# ✅ GOOD: Logging middleware
@app.middleware("http")
async def log_requests(request: Request, call_next):
    start_time = time()
    response = await call_next(request)
    duration = time() - start_time

    logger.info(
        "Request completed",
        extra={
            "method": request.method,
            "path": request.url.path,
            "status_code": response.status_code,
            "duration_ms": duration * 1000,
        }
    )
    return response

# ✅ GOOD: CORS middleware
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://example.com"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# ✅ GOOD: Rate limiting middleware
from slowapi import Limiter, _rate_limit_exceeded_handler
from slowapi.util import get_remote_address
from slowapi.errors import RateLimitExceeded

limiter = Limiter(key_func=get_remote_address)
app.state.limiter = limiter
app.add_exception_handler(RateLimitExceeded, _rate_limit_exceeded_handler)

@app.get("/api/markets")
@limiter.limit("100/minute")
async def get_markets(request: Request):
    return {"success": True, "data": []}
```

## Logging Patterns

### Never Log Secrets or PII

Must not log:
- Passwords, API keys, tokens
- Email addresses, names, phone numbers
- Full request/response bodies

```python
import logging

logger = logging.getLogger(__name__)

# ❌ BAD: Logging PII or secrets
logger.info(f"User: {user.email}")
logger.debug(f"API key: {api_key}")

# ✅ GOOD: Log events without sensitive data
logger.info("User logged in", extra={"user_id": user.id})
```

### Prefer Structured Logging and Lazy Formatting

Use `extra` for structured data. Use lazy formatting (%) for performance.

```python
# ✅ GOOD: Structured logging (preferred)
logger.info(
    "Order created",
    extra={
        "order_id": order.id,
        "user_id": order.user_id,
        "total": str(order.total),
    },
)

# ✅ GOOD: Lazy formatting (recommended for simple cases)
logger.info("Order %s created for user %s", order.id, order.user_id)

# ⚠️ Acceptable but not preferred: f-strings (eagerly evaluated)
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

## Security Best Practices

### Never Build SQL with String Concatenation

Use parameterized queries. Parameter style varies by driver.

**SQLAlchemy (recommended for FastAPI)**:

```python
from sqlalchemy import text
from sqlalchemy.ext.asyncio import AsyncSession

async def get_user(user_id: str, db: AsyncSession) -> User | None:
    query = text("SELECT * FROM users WHERE id = :user_id")
    result = await db.execute(query, {"user_id": user_id})
    return result.fetchone()
```

**asyncpg (PostgreSQL)**:

```python
import asyncpg

async def get_user(user_id: str, conn: asyncpg.Connection) -> dict | None:
    return await conn.fetchrow("SELECT * FROM users WHERE id = $1", user_id)
```

**Never**:

```python
# ❌ BAD: SQL injection vulnerability
query = f"SELECT * FROM users WHERE id = '{user_id}'"
```

### Never Use shell=True

```python
import subprocess

# ✅ GOOD: Safe subprocess execution
subprocess.run(["ls", "-l", filename], capture_output=True, check=True)

# ❌ BAD: Command injection vulnerability
subprocess.run(f"ls -l {filename}", shell=True)
```

### Do Not Deserialize Untrusted Pickle

```python
import json
import pickle

# ❌ BAD: Code execution vulnerability
data = pickle.loads(untrusted_bytes)

# ✅ GOOD: Use JSON for untrusted data
data = json.loads(untrusted_string)
```

### Secrets from Environment

```python
import os

# ✅ GOOD: Load secrets from environment
API_KEY = os.environ["API_KEY"]
DATABASE_URL = os.getenv("DATABASE_URL")

# ❌ BAD: Hardcoded secrets (exposed in git)
API_KEY = "sk-1234..."
```

## Async Database Operations

```python
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from typing import List

# ✅ GOOD: Async queries
async def get_markets_with_positions(
    db: AsyncSession,
    user_id: str
) -> List[Market]:
    # Fetch markets and positions in parallel
    markets_query = select(MarketModel).where(MarketModel.status == "active")
    positions_query = select(PositionModel).where(PositionModel.user_id == user_id)

    markets_result, positions_result = await asyncio.gather(
        db.execute(markets_query),
        db.execute(positions_query)
    )

    markets = markets_result.scalars().all()
    positions = positions_result.scalars().all()

    # Combine results
    position_map = {p.market_id: p for p in positions}
    for market in markets:
        market.user_position = position_map.get(market.id)

    return markets

# ❌ BAD: N+1 query problem
async def get_markets_with_positions_bad(db: AsyncSession, user_id: str):
    markets = await db.execute(select(MarketModel))
    for market in markets.scalars():
        # Separate query for each market - slow!
        position = await db.execute(
            select(PositionModel).where(
                PositionModel.market_id == market.id,
                PositionModel.user_id == user_id
            )
        )
        market.user_position = position.scalar_one_or_none()
    return markets
```

## Caching with Redis

```python
from redis.asyncio import Redis
import json
from typing import Optional

# ✅ GOOD: Redis caching layer
class CacheService:
    def __init__(self, redis: Redis):
        self.redis = redis

    async def get(self, key: str) -> Optional[dict]:
        data = await self.redis.get(key)
        if data:
            return json.loads(data)
        return None

    async def set(self, key: str, value: dict, ttl: int = 300):
        await self.redis.setex(key, ttl, json.dumps(value))

    async def delete(self, key: str):
        await self.redis.delete(key)

# Cached repository
class CachedMarketRepository(MarketRepository):
    def __init__(self, base_repo: MarketRepository, cache: CacheService):
        self.base_repo = base_repo
        self.cache = cache

    async def find_by_id(self, market_id: str) -> Optional[Market]:
        cache_key = f"market:{market_id}"

        # Check cache
        cached = await self.cache.get(cache_key)
        if cached:
            return Market(**cached)

        # Cache miss - fetch from database
        market = await self.base_repo.find_by_id(market_id)
        if market:
            await self.cache.set(cache_key, market.dict(), ttl=300)

        return market

    async def update(self, market_id: str, market_data: UpdateMarketRequest) -> Market:
        # Update database
        market = await self.base_repo.update(market_id, market_data)

        # Invalidate cache
        await self.cache.delete(f"market:{market_id}")

        return market
```

## Background Tasks

```python
from fastapi import BackgroundTasks

# ✅ GOOD: Background task for async operations
async def send_email_notification(user_email: str, message: str):
    """Send email notification (runs in background)."""
    await email_service.send(user_email, message)

@app.post("/api/markets")
async def create_market(
    market: CreateMarketRequest,
    background_tasks: BackgroundTasks,
    current_user: User = Depends(get_current_user)
):
    created = await market_service.create(market)

    # Queue background task
    background_tasks.add_task(
        send_email_notification,
        current_user.email,
        f"Market '{created.name}' created successfully"
    )

    return {"success": True, "data": created}

# ✅ GOOD: Worker queue for heavy tasks
from celery import Celery

celery_app = Celery("worker", broker="redis://localhost:6379")

@celery_app.task
def index_market_embeddings(market_id: str):
    """Generate and index market embeddings (CPU intensive)."""
    market = market_repository.find_by_id(market_id)
    embeddings = embedding_service.generate(market.description)
    vector_store.index(market_id, embeddings)

@app.post("/api/markets/{market_id}/index")
async def index_market(market_id: str):
    # Offload to Celery worker
    index_market_embeddings.delay(market_id)
    return {"success": True, "message": "Indexing queued"}
```

## Async HTTP Client Patterns

### Do Not Mix Sync and Async I/O

```python
import httpx
import asyncio

# ✅ GOOD: All async
async def fetch_users(base_url: str) -> list[UserPayload]:
    async with httpx.AsyncClient(base_url=base_url) as client:
        response = await client.get("/users")
        return [UserPayload(**u) for u in response.json()]

# ❌ BAD: Mixing sync and async - blocks event loop
import requests

async def fetch_users_bad(base_url: str) -> list[UserPayload]:
    response = requests.get(f"{base_url}/users")  # Blocks event loop!
    return [UserPayload(**u) for u in response.json()]
```

### Avoid Blocking Calls Inside Async Functions

```python
# ✅ GOOD: Use async file I/O library
import aiofiles

async def process_file(path: str) -> str:
    async with aiofiles.open(path) as f:
        return await f.read()

# ❌ BAD: Synchronous I/O blocks event loop
async def process_file_bad(path: str) -> str:
    with open(path) as f:  # Synchronous I/O blocks
        return f.read()
```

### Always Use Explicit Timeouts

MUST set timeouts on all outbound HTTP requests to prevent indefinite hangs.

```python
import httpx

# ✅ GOOD: Explicit timeouts
async def fetch_external_data(url: str) -> dict:
    timeout = httpx.Timeout(10.0, connect=5.0)
    async with httpx.AsyncClient(timeout=timeout) as client:
        response = await client.get(url)
        response.raise_for_status()
        return response.json()

# ❌ BAD: No timeout (can hang forever)
async def fetch_external_data_bad(url: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(url)
        return response.json()
```

### Parallel HTTP Requests

```python
import asyncio
import httpx

# ✅ GOOD: Parallel requests with asyncio.gather
async def fetch_multiple_resources(user_id: str) -> dict:
    async with httpx.AsyncClient(timeout=httpx.Timeout(10.0)) as client:
        user_task = client.get(f"/api/users/{user_id}")
        orders_task = client.get(f"/api/orders?user_id={user_id}")
        preferences_task = client.get(f"/api/preferences/{user_id}")

        user_res, orders_res, prefs_res = await asyncio.gather(
            user_task, orders_task, preferences_task
        )

        return {
            "user": user_res.json(),
            "orders": orders_res.json(),
            "preferences": prefs_res.json()
        }

# ❌ BAD: Sequential requests
async def fetch_multiple_resources_bad(user_id: str) -> dict:
    async with httpx.AsyncClient() as client:
        user = await client.get(f"/api/users/{user_id}")
        orders = await client.get(f"/api/orders?user_id={user_id}")
        prefs = await client.get(f"/api/preferences/{user_id}")

        return {
            "user": user.json(),
            "orders": orders.json(),
            "preferences": prefs.json()
        }
```

## Input Validation & Security

### Validate All External Input with Pydantic

MUST validate:
- All user input from request bodies, query params, path params
- User-provided file paths (prevent directory traversal)
- User-provided filenames (prevent command injection)
- All data from external APIs

```python
from pydantic import BaseModel, EmailStr, field_validator, validator
from typing import List

# ✅ GOOD: Strict validation with Pydantic
class CreateUserRequest(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    email: EmailStr
    age: int = Field(..., ge=0, le=150)
    tags: List[str] = Field(default_factory=list, max_items=10)

    @field_validator("name")
    @classmethod
    def validate_name(cls, v: str) -> str:
        if not v.strip():
            raise ValueError("Name cannot be empty or whitespace")
        return v.strip()

    @field_validator("age")
    @classmethod
    def validate_age(cls, v: int) -> int:
        if v < 0 or v > 150:
            raise ValueError("Age must be between 0 and 150")
        return v

@app.post("/api/users")
async def create_user(request: CreateUserRequest):
    # Input already validated by Pydantic
    user = await user_service.create(request)
    return {"success": True, "data": user}
```

### Validate and Sanitize File Paths

MUST validate user-provided file paths to prevent directory traversal attacks.

```python
from pathlib import Path
from fastapi import HTTPException

# ✅ GOOD: Path validation
def read_user_file(filename: str, allowed_dir: str) -> str:
    # Resolve to absolute path and check it's within allowed directory
    file_path = Path(allowed_dir) / filename
    file_path = file_path.resolve()
    allowed_path = Path(allowed_dir).resolve()

    if not file_path.is_relative_to(allowed_path):
        raise HTTPException(
            status_code=400,
            detail="Path traversal detected"
        )

    if not file_path.exists():
        raise HTTPException(status_code=404, detail="File not found")

    with open(file_path) as f:
        return f.read()

@app.get("/api/files/{filename}")
async def get_file(filename: str):
    allowed_directory = "/var/app/uploads"
    content = read_user_file(filename, allowed_directory)
    return {"success": True, "content": content}

# ❌ BAD: No validation - directory traversal vulnerability
@app.get("/api/files/{filename}")
async def get_file_bad(filename: str):
    # User could pass "../../../etc/passwd"
    with open(f"/var/app/uploads/{filename}") as f:
        return {"content": f.read()}
```

### Sanitize Filenames

```python
import re
from fastapi import HTTPException

# ✅ GOOD: Filename sanitization
def sanitize_filename(filename: str) -> str:
    # Remove any path separators
    filename = filename.replace("/", "").replace("\\", "")

    # Allow only alphanumeric, dots, dashes, underscores
    if not re.match(r'^[\w\-. ]+$', filename):
        raise HTTPException(
            status_code=400,
            detail="Invalid filename characters"
        )

    # Prevent hidden files
    if filename.startswith("."):
        raise HTTPException(
            status_code=400,
            detail="Hidden files not allowed"
        )

    return filename

@app.post("/api/upload")
async def upload_file(filename: str, content: bytes):
    safe_filename = sanitize_filename(filename)
    # Proceed with safe filename
    pass
```

## Testing Patterns

```python
import pytest
from httpx import AsyncClient
from sqlalchemy.ext.asyncio import AsyncSession, create_async_engine
from sqlalchemy.orm import sessionmaker

# ✅ GOOD: Async test fixtures
@pytest.fixture
async def async_client():
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client

@pytest.fixture
async def test_db():
    engine = create_async_engine("postgresql+asyncpg://test:test@localhost/test_db")
    async_session = sessionmaker(engine, class_=AsyncSession, expire_on_commit=False)

    async with async_session() as session:
        yield session

    await engine.dispose()

# ✅ GOOD: Integration tests
@pytest.mark.asyncio
async def test_create_market(async_client: AsyncClient, test_db: AsyncSession):
    market_data = {
        "name": "Test Market",
        "description": "Test description",
        "end_date": "2024-12-31T23:59:59Z",
        "categories": ["test"]
    }

    response = await async_client.post("/api/markets", json=market_data)

    assert response.status_code == 200
    data = response.json()
    assert data["success"] is True
    assert data["data"]["name"] == "Test Market"

# ✅ GOOD: Mock dependencies
@pytest.mark.asyncio
async def test_get_market_not_found(async_client: AsyncClient):
    async def mock_get_market_service():
        mock_repo = Mock(spec=MarketRepository)
        mock_repo.find_by_id.return_value = None
        return MarketService(mock_repo)

    app.dependency_overrides[get_market_service] = mock_get_market_service

    response = await async_client.get("/api/markets/nonexistent")

    assert response.status_code == 404
    assert response.json()["success"] is False
```

**Remember**: FastAPI combines Python's type system with async/await for high-performance APIs. Use Pydantic for validation, dependency injection for clean architecture, and async patterns for scalability.
