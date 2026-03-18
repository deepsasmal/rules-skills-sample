---
name: gen-tests
description: Generate complete, production-ready pytest test suites — unit tests, integration tests, fixtures, and conftest.py — following this project's testing standards. Use this skill whenever the user asks to write tests, generate test coverage, add pytest fixtures, test an endpoint, test a service, or improve test coverage. Triggers on: "write tests for", "generate tests", "add coverage", "test this service", "create fixtures", "I need a conftest", or any request involving pytest. Always use this skill for test generation — it ensures correct async patterns (pytest-asyncio, httpx.AsyncClient), proper ContextVar setup, mocking conventions, and the 80% coverage target are applied consistently.
---

# Test Generator

Generate a complete, immediately runnable pytest test suite for the provided code. Covers unit tests, integration tests, fixtures, and conftest scaffolding — all following this project's async-first testing standards.

The user provides source code, a file path, or a description of what needs testing.

---

## Step 1 — Understand what to test

Before generating, identify:

- **What layer is this?** — service (unit), router (integration), utility (unit)
- **What are the happy paths?** — normal inputs that should succeed
- **What are the failure cases?** — validation errors, not-found, unauthorized, DB errors
- **Are there async operations?** — all service/router code in this project is async
- **Does this code read `request_id_var`?** — if yes, the ContextVar must be seeded in the test

If the user provides a code snippet, read it carefully and derive all cases automatically. Do not ask unless something is genuinely ambiguous.

---

## Step 2 — conftest.py (generate or extend)

Always produce or show what to add to `conftest.py`. Standard fixtures every project needs:

```python
# tests/conftest.py
import pytest
import pytest_asyncio
from httpx import AsyncClient
from unittest.mock import AsyncMock

from core.main import app                          # your FastAPI app
from middleware.logging import request_id_var

@pytest_asyncio.fixture
async def async_client():
    async with AsyncClient(app=app, base_url="http://test") as client:
        yield client

@pytest.fixture
def request_id():
    """Seed the ContextVar so any service logging doesn't blow up."""
    token = request_id_var.set("test-req-123")
    yield "test-req-123"
    request_id_var.reset(token)

@pytest.fixture
def mock_db(mocker):
    return mocker.patch("db.session.get_session", new_callable=AsyncMock)

@pytest.fixture
def auth_headers():
    return {"Authorization": "Bearer test-token"}
```

Rules:
- Use `pytest_asyncio.fixture` (not `pytest.fixture`) for async fixtures
- Always include the `request_id` fixture — any test that exercises code touching `request_id_var` must use it
- Never use `TestClient` — always `httpx.AsyncClient`

---

## Step 3 — Unit tests (service layer)

Template for a service under `tests/unit/test_{resource}_service.py`:

```python
import pytest
from unittest.mock import AsyncMock, patch, MagicMock
from services.{resource}_service import {Resource}Service
from schemas.{resource} import {Resource}Request

class TestCreate{Resource}:
    @pytest.mark.asyncio
    async def test_create_success(self, request_id, mock_db):
        """Happy path — valid input produces expected output."""
        service = {Resource}Service(db=mock_db)
        payload = {Resource}Request(name="test-name")

        result = await service.create(payload)

        assert result.id is not None
        assert result.name == "test-name"

    @pytest.mark.asyncio
    async def test_create_raises_on_duplicate(self, request_id, mock_db):
        """DB unique constraint → service raises ValueError."""
        mock_db.execute.side_effect = Exception("unique violation")
        service = {Resource}Service(db=mock_db)
        payload = {Resource}Request(name="duplicate")

        with pytest.raises(ValueError, match="already exists"):
            await service.create(payload)
```

Rules:
- Every async test must have `@pytest.mark.asyncio`
- Always include `request_id` fixture in async tests that call services
- Use `AsyncMock` for any awaitable dependency
- Test **both** the happy path **and** at least two failure cases per method
- Do not import `httpx` in unit tests — unit tests never hit the HTTP layer

---

## Step 4 — Integration tests (API layer)

Template for `tests/integration/test_{resource}_api.py`:

```python
import pytest
from httpx import AsyncClient

class TestCreate{Resource}API:
    @pytest.mark.asyncio
    async def test_post_success(self, async_client: AsyncClient, auth_headers):
        response = await async_client.post(
            "/v1/{resource}s/",
            json={"name": "test-name"},
            headers=auth_headers,
        )
        assert response.status_code == 201
        body = response.json()
        assert body["status"] == "success"
        assert body["data"]["name"] == "test-name"

    @pytest.mark.asyncio
    async def test_post_missing_field_returns_422(self, async_client: AsyncClient, auth_headers):
        response = await async_client.post(
            "/v1/{resource}s/",
            json={},   # missing required fields
            headers=auth_headers,
        )
        assert response.status_code == 422

    @pytest.mark.asyncio
    async def test_post_unauthorized_returns_401(self, async_client: AsyncClient):
        response = await async_client.post("/v1/{resource}s/", json={"name": "x"})
        assert response.status_code == 401

    @pytest.mark.asyncio
    async def test_list_returns_paginated(self, async_client: AsyncClient, auth_headers):
        response = await async_client.get(
            "/v1/{resource}s/",
            params={"limit": 10, "offset": 0},
            headers=auth_headers,
        )
        assert response.status_code == 200
        assert "data" in response.json()
```

Rules:
- Test the **HTTP contract**, not the service internals — mock the service layer
- Always test: success (2xx), validation failure (422), auth failure (401), not-found (404)
- Verify the response envelope: `{"status": "success", "data": {...}, "meta": {...}}`
- For list endpoints, always test the paginated response shape

---

## Step 5 — Mocking LLM / external calls

```python
@pytest.fixture
def mock_llm(mocker):
    mock = mocker.patch("services.llm_client.complete", new_callable=AsyncMock)
    mock.return_value = '{"answer": "42"}'   # structured string — service must parse
    return mock

@pytest.mark.asyncio
async def test_insight_generation(mock_llm, request_id):
    # LLM is always mocked — never hit real endpoints in tests
    ...
```

Rules:
- **Always** mock LLM calls — never hit real endpoints in tests
- Mock at the **call-site module path**, not the import path
- Return realistic structured strings so downstream parsing is tested too

---

## Step 6 — ContextVar setup in tests

Any test that calls code reading `request_id_var` must seed it first. Use the `request_id` fixture defined in conftest — it handles setup and teardown:

```python
@pytest.mark.asyncio
async def test_service_logs_correctly(request_id, mock_db):
    # request_id fixture has already set request_id_var = "test-req-123"
    service = MyService(db=mock_db)
    result = await service.do_something()
    # logging inside do_something() will get "test-req-123" — no crash, no "unknown"
    assert result is not None
```

For tests that need to verify concurrent isolation:

```python
@pytest.mark.asyncio
async def test_contextvar_isolation():
    import asyncio
    from middleware.logging import request_id_var

    async def task(req_id):
        token = request_id_var.set(req_id)
        await asyncio.sleep(0)              # yield to event loop
        val = request_id_var.get()
        request_id_var.reset(token)
        return val

    results = await asyncio.gather(task("req-A"), task("req-B"))
    assert set(results) == {"req-A", "req-B"}   # no bleed-through
```

---

## Step 7 — Output format & checklist

Present files in this order:
1. `tests/conftest.py` (or additions to it)
2. Unit test file(s)
3. Integration test file(s)

Before finalising, verify:

- [ ] Every async test has `@pytest.mark.asyncio`
- [ ] No `TestClient` — only `httpx.AsyncClient`
- [ ] `request_id` fixture used wherever services are called
- [ ] LLM / external API calls are mocked
- [ ] Both happy path and failure cases covered
- [ ] Integration tests verify the `{ status, data, meta }` envelope
- [ ] 401 / 403 / 404 / 422 / 500 cases have at least one test each
- [ ] `AsyncMock` used for all async dependencies

End with a one-liner showing how to run with coverage:

```bash
pytest tests/ --cov=app --cov-report=term-missing --cov-fail-under=80
```
