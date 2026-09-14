---
name: fastapi-patterns
description: >-
  FastAPI application layout, routers, Pydantic v2 schemas, dependency
  injection, settings, and exception handling for my-python-app. Use when
  adding HTTP endpoints, request/response models, app startup, or FastAPI
  services in this Python project.
---

# FastAPI Patterns (my-python-app)

Stack-specific HTTP conventions for this project. Use `api-design` for REST
contracts and `error-handling` for typed exceptions; this skill covers FastAPI
wiring.

## When to Activate

- Adding or changing FastAPI routers, dependencies, or lifespan hooks
- Defining Pydantic request/response schemas
- Wiring settings from environment variables
- Mapping domain errors to HTTP responses

## Project Layout

```
app/
├── main.py                 # FastAPI() + include_router + exception handlers
├── api/
│   └── v1/
│       ├── router.py       # aggregates feature routers
│       └── users.py        # one module per resource
├── core/
│   ├── config.py           # pydantic-settings
│   └── deps.py             # shared Depends()
├── schemas/                # Pydantic DTOs (never leak ORM models)
├── services/               # business logic
└── repositories/           # data access
```

Keep handlers thin: parse → call service → return schema.

## App Factory

```python
from contextlib import asynccontextmanager
from fastapi import FastAPI

from app.api.v1.router import api_router
from app.core.config import settings
from app.core.errors import register_exception_handlers


@asynccontextmanager
async def lifespan(app: FastAPI):
    # startup: engine / connection pool
    yield
    # shutdown: dispose engine


def create_app() -> FastAPI:
    app = FastAPI(title=settings.app_name, lifespan=lifespan)
    app.include_router(api_router, prefix="/api/v1")
    register_exception_handlers(app)
    return app
```

## Settings

```python
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", extra="ignore")

    app_name: str = "my-python-app"
    database_url: str
    debug: bool = False


settings = Settings()
```

Never hardcode secrets. Fail fast if required fields are missing.

## Router + Service

```python
from fastapi import APIRouter, Depends, status
from app.schemas.user import CreateUserRequest, UserResponse
from app.services.user_service import UserService
from app.core.deps import get_user_service

router = APIRouter(prefix="/users", tags=["users"])


@router.post("", response_model=UserResponse, status_code=status.HTTP_201_CREATED)
async def create_user(
    body: CreateUserRequest,
    service: UserService = Depends(get_user_service),
) -> UserResponse:
    user = await service.create(body)
    return UserResponse.model_validate(user)
```

```python
# PASS: constructor injection in deps.py
def get_user_service(session: AsyncSession = Depends(get_session)) -> UserService:
    return UserService(UserRepository(session))
```

## Pydantic v2 Schemas

```python
from pydantic import BaseModel, ConfigDict, EmailStr, Field


class CreateUserRequest(BaseModel):
    email: EmailStr
    name: str = Field(min_length=1, max_length=100)


class UserResponse(BaseModel):
    model_config = ConfigDict(from_attributes=True)

    id: str
    email: EmailStr
    name: str
```

- Request models reject extra fields by default; do not use `model_config` to
  allow extras on write DTOs.
- Response models use `from_attributes=True` so services can return ORM objects.
- Never return SQLAlchemy models from a path operation.

## Exception Mapping

```python
from fastapi import FastAPI, Request
from fastapi.responses import JSONResponse
from app.core.errors import AppError, NotFoundError, ValidationError


def register_exception_handlers(app: FastAPI) -> None:
    @app.exception_handler(AppError)
    async def app_error_handler(_request: Request, exc: AppError) -> JSONResponse:
        payload: dict = {"error": {"code": exc.code, "message": str(exc)}}
        if isinstance(exc, ValidationError) and exc.details:
            payload["error"]["details"] = exc.details
        return JSONResponse(status_code=exc.status_code, content=payload)
```

Map `NotFoundError` → 404, validation → 422, conflict → 409. Unexpected
exceptions stay 500 with a generic message; log the traceback server-side.

## Checklist

- [ ] Resource URL is plural, kebab-case, under `/api/v1/`
- [ ] Correct status code (201 + Location for creates, 204 for deletes)
- [ ] Input validated with Pydantic; ORM models are not request bodies
- [ ] Handler has no SQL or business rules
- [ ] Settings come from env / `.env`, not literals
- [ ] Error envelope is `{ "error": { "code", "message" } }`
