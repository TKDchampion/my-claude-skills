---
name: backend-architecture
description: Use when designing or scaffolding a new backend project. Guides architecture selection based on project scale, feature-based module structure, global error handling with decorator support, shared core modules, and standardized type/response conventions.
---

# Backend Architecture Skill

Apply these architecture decisions when designing or scaffolding a backend project.

---

## Step 1 — Choose Architecture by Project Scale

Before writing any code, assess scale and pick the appropriate pattern. Over-engineering kills small projects; under-engineering kills large ones.

| Scale | Signals | Pattern | Complexity |
|---|---|---|---|
| **Prototype / MVP** | Solo dev, <5 entities, throw-away | Flat MVC | Low |
| **Product / Startup** | Team of 2–5, growing features, real users | Feature-based MVC + Services | Medium |
| **Platform / Enterprise** | Multiple teams, complex domain rules, microservice-ready | DDD with Bounded Contexts | High |

### Flat MVC (Small)
```
src/
  models/          # ORM entities
  schemas/         # DTOs (request/response)
  routers/         # Endpoint definitions
  services/        # Business logic
  main.py
```
Use when the project is a quick internal tool, proof-of-concept, or single-developer effort. Avoid for anything that will grow beyond ~10 endpoints.

### Feature-based MVC + Services (Medium) ← Most Common
```
src/
  features/
    auth/
    users/
    billing/
  core/             # Shared infrastructure (see Step 3)
  shared/           # Cross-cutting helpers, constants, types
  main.py
```
Each feature owns all its layers. Only promote to `shared/` or `core/` when **genuinely reused by 2+ features**. Default to this pattern for product-stage backends.

### DDD with Bounded Contexts (Large)
```
src/
  domains/
    identity/       # Bounded context: auth, users, roles
      application/  # Use cases / command handlers
      domain/       # Entities, value objects, domain events
      infrastructure/ # Repos, external adapters
      interface/    # API layer
    billing/
    analytics/
  core/
  shared/
  main.py
```
Use when multiple teams own separate parts of the system, when domain rules are complex (invariants, aggregates), or when the service will eventually split into microservices.

---

## Step 2 — Feature Module Structure

Every feature follows the same internal layout. All files that serve one feature live together — never scatter a feature's logic across top-level directories.

```
features/
  {feature}/
    {feature}.entity.py     # ORM model (SQLAlchemy / Prisma / Mongoose…)
    {feature}.dto.py        # Pydantic / Zod / class-validator DTOs
    {feature}.repository.py # Raw DB CRUD, no business logic
    {feature}.service.py    # Business logic, orchestration
    {feature}.router.py     # HTTP endpoint definitions
    {feature}.errors.py     # Feature-specific exceptions (optional override)
    {feature}.types.py      # Local types/enums used only by this feature
```

**Rules:**
- Repository layer: only DB operations, no `if` business rules.
- Service layer: all `if`/`raise` logic lives here. Never import a router in a service.
- Router layer: thin — validate input (DTO), call service, return response. No business logic.
- If a helper is used by exactly one feature, keep it inside that feature folder.
- If a helper is used by two or more features, move it to `shared/`.

---

## Step 3 — Core Shared Modules

Always scaffold these 5 core modules. They are infrastructure, not business logic.

### `core/http/` — External HTTP Client Base

All services that call external APIs must extend `BaseHTTPService`. This centralizes timeout config, header injection, and error normalization.

```python
# core/http/base_http_service.py
import httpx
from app.core.errors.exceptions import DomainException

class BaseHTTPService:
    def __init__(self, base_url: str, timeout: int = 30):
        self._base_url = base_url
        self._timeout = timeout

    def _client(self) -> httpx.Client:
        return httpx.Client(base_url=self._base_url, timeout=self._timeout)

    def get(self, path: str, **kwargs) -> dict:
        try:
            with self._client() as c:
                res = c.get(path, **kwargs)
                res.raise_for_status()
                return res.json()
        except httpx.HTTPStatusError as e:
            raise DomainException(
                msg=f"Upstream error: {e.response.text}",
                type="upstream_error",
                code=e.response.status_code,
            )

    def post(self, path: str, json: dict = None, **kwargs) -> dict:
        try:
            with self._client() as c:
                res = c.post(path, json=json, **kwargs)
                res.raise_for_status()
                return res.json()
        except httpx.HTTPStatusError as e:
            raise DomainException(
                msg=f"Upstream error: {e.response.text}",
                type="upstream_error",
                code=e.response.status_code,
            )

    def post_stream(self, path: str, json: dict = None):
        """Yields raw bytes for SSE/streaming responses."""
        with httpx.Client(base_url=self._base_url, timeout=None) as c:
            with c.stream("POST", path, json=json) as res:
                res.raise_for_status()
                for chunk in res.iter_bytes():
                    yield chunk

# Usage: extend and inject base_url from env
# @external_api("wren_ai")
# class WrenAiService(BaseHTTPService):
#     def __init__(self):
#         super().__init__(base_url=os.getenv("WREN_API_URL", "http://localhost:9000"))
```

### `core/errors/` — Exception Hierarchy + Global Handler

Define a single `DomainException` that every layer raises. The global handler catches it and converts to a structured HTTP response. Individual routes can catch specific subtypes to override behavior.

```python
# core/errors/exceptions.py
from typing import Optional

class DomainException(Exception):
    def __init__(self, msg: str, type: str, code: int = 400):
        self.msg = msg
        self.type = type
        self.code = code
        super().__init__(msg)

# Typed subtypes — raise these for precise overrides
class NotFoundException(DomainException):
    def __init__(self, msg: str, type: str = "not_found"):
        super().__init__(msg, type, code=404)

class UnauthorizedException(DomainException):
    def __init__(self, msg: str, type: str = "unauthorized"):
        super().__init__(msg, type, code=401)

class ForbiddenException(DomainException):
    def __init__(self, msg: str, type: str = "forbidden"):
        super().__init__(msg, type, code=403)

class ConflictException(DomainException):
    def __init__(self, msg: str, type: str = "conflict"):
        super().__init__(msg, type, code=409)
```

```python
# core/errors/handler.py  — register on FastAPI app
import logging
from fastapi import Request
from fastapi.responses import JSONResponse
from app.core.errors.exceptions import DomainException

logger = logging.getLogger(__name__)

async def domain_exception_handler(request: Request, exc: DomainException):
    return JSONResponse(
        status_code=exc.code,
        content={"type": exc.type, "msg": exc.msg},
    )

async def unhandled_exception_handler(request: Request, exc: Exception):
    logger.exception("Unhandled exception: %s", exc)
    return JSONResponse(
        status_code=500,
        content={"type": "internal_error", "msg": "An unexpected error occurred"},
    )

# In main.py:
# app.add_exception_handler(DomainException, domain_exception_handler)
# app.add_exception_handler(Exception, unhandled_exception_handler)
```

### `core/decorators/` — Route & Transaction Decorators

Decorators reduce boilerplate. Every route must be wrapped. Individual routes that need different error behavior simply catch and re-raise before the decorator sees it.

```python
# core/decorators/router_try.py
import functools
import logging
from fastapi import HTTPException
from app.core.errors.exceptions import DomainException

logger = logging.getLogger(__name__)

def router_try():
    """
    Wraps a FastAPI route handler.
    - DomainException → HTTPException with structured body
    - Any other Exception → 500
    Individual handlers can catch specific subtypes BEFORE this decorator runs.
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            try:
                return func(*args, **kwargs)
            except DomainException as e:
                raise HTTPException(
                    status_code=e.code,
                    detail={"type": e.type, "msg": e.msg},
                )
            except Exception as e:
                logger.exception("Unhandled route error in %s: %s", func.__name__, e)
                raise HTTPException(
                    status_code=500,
                    detail={"type": "internal_error", "msg": "An unexpected error occurred"},
                )
        return wrapper
    return decorator
```

```python
# core/decorators/db_transaction.py
import functools
from sqlalchemy.orm import Session

def db_tx(func):
    """Auto-commits on success, rolls back on any exception. Requires `db: Session` kwarg."""
    @functools.wraps(func)
    def wrapper(*args, **kwargs):
        db: Session = kwargs.get("db") or next(
            (a for a in args if isinstance(a, Session)), None
        )
        try:
            result = func(*args, **kwargs)
            if db:
                db.commit()
            return result
        except Exception:
            if db:
                db.rollback()
            raise
    return wrapper
```

### `core/response/` — Standardized Response Types

Define all response shapes once. Every endpoint returns one of these shapes.

```python
# core/response/types.py
from typing import TypeVar, Generic, Optional, List
from pydantic import BaseModel

T = TypeVar("T")

class SuccessResponse(BaseModel, Generic[T]):
    data: T
    msg: Optional[str] = None

class PaginatedResponse(BaseModel, Generic[T]):
    data: List[T]
    total: int
    page: int
    page_size: int

class ErrorResponse(BaseModel):
    type: str
    msg: str

# Usage in routes:
# @router.get("/users", response_model=PaginatedResponse[UserReadDTO])
# @router.post("/users", response_model=SuccessResponse[UserReadDTO])
```

```python
# core/response/helpers.py
from typing import List, TypeVar
from app.core.response.types import SuccessResponse, PaginatedResponse

T = TypeVar("T")

def ok(data: T, msg: str = None) -> SuccessResponse[T]:
    return SuccessResponse(data=data, msg=msg)

def paginate(data: List[T], total: int, page: int, page_size: int) -> PaginatedResponse[T]:
    return PaginatedResponse(data=data, total=total, page=page, page_size=page_size)
```

### `core/pagination/` — Reusable Pagination Query Params

```python
# core/pagination/params.py
from fastapi import Query
from dataclasses import dataclass

@dataclass
class PaginationParams:
    page: int
    page_size: int

    @property
    def offset(self) -> int:
        return (self.page - 1) * self.page_size

def pagination_params(
    page: int = Query(default=1, ge=1),
    page_size: int = Query(default=20, ge=1, le=100),
) -> PaginationParams:
    return PaginationParams(page=page, page_size=page_size)

# Usage in route:
# @router.get("/users")
# def list_users(pagination: PaginationParams = Depends(pagination_params), ...):
#     items = repo.list(offset=pagination.offset, limit=pagination.page_size)
#     return paginate(items, total, pagination.page, pagination.page_size)
```

---

## Step 4 — Type Conventions

### Request DTOs (Input validation)
```python
from pydantic import BaseModel, Field, EmailStr
from typing import Optional

class UserCreateDTO(BaseModel):
    email: EmailStr
    name: str = Field(..., min_length=1, max_length=100)
    role_id: Optional[int] = None
```

### Response DTOs (Output shape)
```python
from datetime import datetime

class UserReadDTO(BaseModel):
    id: int
    email: str
    name: str
    created_at: datetime

    class Config:
        from_attributes = True  # ORM mode
```

### Enums for controlled values
```python
from enum import Enum

class UserRole(str, Enum):
    ADMIN = "admin"
    VIEWER = "viewer"
    EDITOR = "editor"
```

---

## Step 5 — Error Handling Flow

The error handling has three levels. Lower levels can **override** higher ones by catching before the decorator.

```
Service raises DomainException
        ↓
  @router_try() catches it
        ↓
  Returns { type, msg } + HTTP status

  ─── Override: route-level catch ───
  @router.post("/foo")
  @router_try()
  def foo(...):
      try:
          service.do_thing()
      except ConflictException:
          # Custom handling for this specific case
          return {"type": "already_exists", "msg": "Custom override message"}

  ─── Global fallback for unregistered exceptions ───
  app.add_exception_handler(Exception, unhandled_exception_handler)
```

**Rules:**
- Never let a raw `Exception` surface to the client — always normalize.
- Never expose stack traces or internal system details in error messages.
- Use typed exception subtypes (`NotFoundException`, `ConflictException`) so callers can `except` precisely.
- Log at `ERROR` level with context before re-raising or converting.

---

## Step 6 — Registration Checklist (main.py)

When adding a new feature, check off each item:

```python
# main.py
from fastapi import FastAPI
from app.core.errors.handler import domain_exception_handler, unhandled_exception_handler
from app.core.errors.exceptions import DomainException
from app.features.auth.auth_router import auth_router
from app.features.users.users_router import users_router
# ... import new feature routers

app = FastAPI()

# 1. Register global exception handlers
app.add_exception_handler(DomainException, domain_exception_handler)
app.add_exception_handler(Exception, unhandled_exception_handler)

# 2. Include routers with prefix
app.include_router(auth_router, prefix="/auth", tags=["Auth"])
app.include_router(users_router, prefix="/users", tags=["Users"])
# app.include_router(new_feature_router, prefix="/...", tags=["..."])
```

For **Alembic** / DB migrations — after adding a new entity:
1. Import entity in `app/entities/__init__.py`
2. `alembic revision --autogenerate -m "add {entity_name}"`
3. Review the generated migration
4. `alembic upgrade head`

---

## Quick Reference

| Layer | Location | Allowed imports | Forbidden imports |
|---|---|---|---|
| Entity | `features/*/entity.py` | SQLAlchemy | Services, Routers |
| DTO | `features/*/dto.py` | Pydantic, Enums | SQLAlchemy, Services |
| Repository | `features/*/repository.py` | Entity, Session | Services, Routers |
| Service | `features/*/service.py` | Repos, Exceptions, core | Routers, FastAPI |
| Router | `features/*/router.py` | Services, DTOs, Decorators | Repos, Entities directly |
| Core | `core/**` | stdlib, third-party | Features |
| Shared | `shared/**` | stdlib, Pydantic | Features, Core |

---

## When to Use This Skill

Invoke when:
- Starting a new backend project from scratch
- Adding a new service/module to an existing backend
- Refactoring a monolithic router file into feature modules
- Deciding between DDD, MVC, or simpler patterns
- Designing error handling strategy
- Standardizing response shapes across endpoints
