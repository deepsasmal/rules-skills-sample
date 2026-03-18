# Testing Rules

## Framework
- **pytest** + `pytest-asyncio` for all async tests
- Use `httpx.AsyncClient` for async FastAPI endpoint testing (not `TestClient`)

## Test Types
- **Unit** — services, utilities, pure functions (mandatory)
- **Integration** — API + DB + external services
- **End-to-end** — critical user flows

## Coverage
- Minimum **80%** coverage; enforced in CI

## Async Tests
```python
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_create_insight(async_client: AsyncClient):
    response = await async_client.post("/v1/insights", json={...})
    assert response.status_code == 200
```

## Fixtures
```python
# conftest.py
@pytest.fixture
async def async_client(app):
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client
```

Provide fixtures for: DB connections, Redis, test data, auth tokens.

## Mocking
- Mock: LLM calls, external APIs, data warehouse queries
- Use `unittest.mock.AsyncMock` for async callables
- Use `pytest-mock` (`mocker` fixture) for convenience

## ContextVar Testing
- When testing code that reads `request_id_var`, set it explicitly in the test:
  ```python
  from middleware.logging import request_id_var
  token = request_id_var.set("test-req-123")
  # ... run code ...
  request_id_var.reset(token)
  ```

## Data / DB Testing
- Use a dedicated test database or transaction rollback per test
- Validate dbt models with `dbt test`

## CI/CD
- Tests run on every PR; merge blocked on failure
- Run `pytest --cov --cov-fail-under=80`
