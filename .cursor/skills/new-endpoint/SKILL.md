---
name: new-endpoint
description: Scaffold a complete, production-ready FastAPI endpoint following this project's standards. Use this skill whenever the user asks to add an endpoint, create a new route, build an API for a feature, add a new resource, or wire up a new service. Triggers on phrases like "add an endpoint", "create a route for", "I need an API to", "scaffold a new resource", or any request to expose new functionality over HTTP. Always use this skill for endpoint work — it ensures correct async patterns, Pydantic v2 schemas, ContextVar-based logging, and project structure conventions are followed every time.
---

# New Endpoint Scaffold

Scaffold a complete FastAPI endpoint that is fully compliant with this project's rules across API design, data models, error handling, logging, and project structure.

The user provides a description of what the endpoint should do. From that, produce all required files in one shot.

---

## Step 1 — Clarify before generating

If any of the following are ambiguous, ask before writing code:

- **Resource name** — what noun is being operated on? (e.g. `insight`, `report`, `dataset`)
- **Operation** — CRUD? Custom action? LLM-triggered query?
- **Auth required?** — assume yes unless told otherwise
- **Background job or sync response?** — if heavy work is involved, the endpoint must return a `job_id`
- **Pagination needed?** — assume yes for list endpoints

---

## Step 2 — Files to generate

For a resource called `{resource}`, generate these files:

```
app/
├── api/v1/{resource}s.py          # Router
├── schemas/{resource}.py          # Request + Response models
├── services/{resource}_service.py # Business logic
└── tests/
    ├── unit/test_{resource}_service.py
    └── integration/test_{resource}_api.py
```

---

## Step 3 — Implementation rules per file

### `api/v1/{resource}s.py` — Router

```python
from fastapi import APIRouter, Depends, HTTPException, status
from schemas.{resource} import {Resource}Request, {Resource}Response
from services.{resource}_service import {Resource}Service

router = APIRouter(prefix="/v1/{resource}s", tags=["{resource}s"])

@router.post("/", response_model={Resource}Response, status_code=status.HTTP_201_CREATED)
async def create_{resource}(
    payload: {Resource}Request,
    service: {Resource}Service = Depends(),
) -> {Resource}Response:
    return await service.create(payload)
```

Rules:
- Always `async def`
- Use `Depends()` for service injection — never instantiate services inside the handler
- Never put business logic here — only validate, delegate, return
- For heavy operations, return `{"status": "success", "data": {"job_id": "..."}, "meta": {}}` and delegate to a worker
- Wrap service calls in `try/except` only if you need endpoint-specific error translation; otherwise let the global exception middleware handle it

### `schemas/{resource}.py` — Pydantic v2 models

```python
from pydantic import BaseModel, ConfigDict, Field

class {Resource}Request(BaseModel):
    model_config = ConfigDict(extra="forbid", strict=True)

    # fields with Field(description=...) for every public field
    name: str = Field(..., min_length=1, max_length=255, description="...")

class {Resource}Response(BaseModel):
    model_config = ConfigDict(extra="forbid")

    id: str
    name: str
    created_at: str  # ISO 8601
```

Rules:
- Separate `Request`, `Response`, and `DTO` models — never reuse one model for all three roles
- `extra="forbid"` on all request models
- `Field(description=...)` on every field — required for LLM-facing schemas
- No deeply nested structures; keep responses flat

### `services/{resource}_service.py` — Business logic

```python
from middleware.logging import request_id_var
import logging

logger = logging.getLogger(__name__)

class {Resource}Service:
    async def create(self, payload: {Resource}Request) -> {Resource}Response:
        logger.info("Creating {resource}", extra={
            "request_id": request_id_var.get(),   # ContextVar — no ctx arg needed
            "name": payload.name,
        })
        # ... business logic ...
```

Rules:
- All methods `async def`
- **Never** accept or pass `request_id` / `ctx` as an argument — read it from `request_id_var.get()`
- No HTTP/FastAPI imports here — services are framework-agnostic
- Raise domain exceptions (e.g. `ValueError`, `PermissionError`) — not `HTTPException`; the API layer or middleware maps those

### Response envelope

All responses must use the standard envelope:

```json
{ "status": "success", "data": {}, "meta": {} }
```

Use a generic wrapper model if the project has one, otherwise define inline:

```python
class APIResponse(BaseModel, Generic[T]):
    status: Literal["success", "error"] = "success"
    data: T
    meta: dict = Field(default_factory=dict)
```

---

## Step 4 — Checklist before finishing

Before presenting code, verify each item:

- [ ] Router uses `async def` on every handler
- [ ] No `time.sleep`, no blocking I/O in handler or service
- [ ] `request_id_var.get()` used for logging — no `ctx` arg threaded anywhere
- [ ] Request schema has `extra="forbid"` and `Field(description=...)` on all fields
- [ ] Service raises domain exceptions, not `HTTPException`
- [ ] Endpoint returns standard `{ status, data, meta }` envelope
- [ ] List endpoint has `limit` / `offset` / `filters` params
- [ ] Auth dependency added unless explicitly excluded
- [ ] Stub test files generated alongside production code

---

## Step 5 — Output format

Present files in this order:
1. Schema
2. Service
3. Router
4. Tests (stubs — full test generation is handled by the `gen-tests` skill)

Show each file with its full path as the code block header. After all files, add a short **"Wire it up"** section showing the two lines needed in `core/main.py` to register the router.
