---
name: pytest-workflow
description: >-
  pytest TDD workflow for my-python-app: fixtures, parametrize, FastAPI
  TestClient/httpx, coverage, and test layout. Use when writing or fixing
  Python tests, adding FastAPI endpoint tests, or running pytest/coverage
  in this project.
---

# pytest Workflow (my-python-app)

Python TDD for this repo. The shared `tdd-workflow` skill is Node-oriented;
use this skill for pytest commands, layout, and FastAPI tests.

## When to Activate

- Adding a feature, bug fix, or refactor in Python
- Writing unit, API, or repository tests
- Setting up fixtures, factories, or coverage gates

## Commands

Resolve commands from `pyproject.toml` before assuming names:

```bash
pytest                              # RED / GREEN gate
pytest -k test_create_user -q       # focused target
pytest --cov=app --cov-report=term-missing
ruff check . && ruff format --check .
```

Do not edit production code until the new test has been collected and failed
for the intended reason (missing behavior, not import/setup noise).

## Layout

```
tests/
├── conftest.py                     # shared fixtures
├── unit/                           # no I/O; mock repositories
├── api/                            # FastAPI TestClient / httpx
└── repositories/                   # DB tests (test database or Testcontainers)
```

Name files `test_<module>.py`. One behavior per test; name the assertion.

## RED → GREEN

1. Write the failing test for the user-visible behavior.
2. Run the focused pytest target; confirm RED.
3. Implement the minimum code to pass.
4. Re-run the same target; confirm GREEN.
5. Refactor only while GREEN. Keep coverage ≥ 80% on changed modules.

## Fixtures

```python
# tests/conftest.py
import pytest
from httpx import ASGITransport, AsyncClient

from app.main import create_app
from app.core.deps import get_session


@pytest.fixture
def app():
    return create_app()


@pytest.fixture
async def client(app, session):
    app.dependency_overrides[get_session] = lambda: session
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as ac:
        yield ac
    app.dependency_overrides.clear()
```

Override dependencies per test. Do not hit real external APIs in unit/API tests.

## API Test

```python
import pytest


@pytest.mark.asyncio
async def test_create_user_returns_201(client):
    response = await client.post(
        "/api/v1/users",
        json={"email": "ada@example.com", "name": "Ada"},
    )
    assert response.status_code == 201
    assert response.json()["email"] == "ada@example.com"


@pytest.mark.asyncio
async def test_create_user_rejects_invalid_email(client):
    response = await client.post(
        "/api/v1/users",
        json={"email": "not-an-email", "name": "Ada"},
    )
    assert response.status_code == 422
    assert response.json()["error"]["code"] == "VALIDATION_ERROR"
```

Match the handler's actual JSON. `fastapi-patterns` returns the Pydantic model
directly (no `data` wrapper) unless the project already uses an envelope.

## Unit Test

```python
import pytest
from app.core.errors import ConflictError
from app.services.user_service import UserService


@pytest.mark.asyncio
async def test_create_user_rejects_duplicate_email(mock_users):
    mock_users.find_by_email.return_value = existing_user
    service = UserService(mock_users)

    with pytest.raises(ConflictError):
        await service.create(CreateUserRequest(email="ada@example.com", name="Ada"))
```

## Parametrize Edge Cases

```python
@pytest.mark.parametrize(
    "payload",
    [
        {"email": "ada@example.com", "name": ""},
        {"email": "ada@example.com", "name": "x" * 101},
        {"email": "ada@example.com"},
    ],
)
@pytest.mark.asyncio
async def test_create_user_validates_name(client, payload):
    response = await client.post("/api/v1/users", json=payload)
    assert response.status_code == 422
```

## Anti-Patterns

| Avoid | Do instead |
|-------|------------|
| Asserting private helpers | Assert HTTP status, JSON, or public return values |
| Tests that depend on run order | Each test creates its own data |
| Hitting production DB | Dedicated test DB or transactional rollback |
| `time.sleep` for async | `pytest-asyncio` + awaited calls |

## Coverage

```toml
# pyproject.toml
[tool.pytest.ini_options]
asyncio_mode = "auto"
testpaths = ["tests"]

[tool.coverage.run]
source = ["app"]
omit = ["app/main.py"]
```

Fail the change if coverage on touched files drops below 80% without an
explicit, documented gap.
