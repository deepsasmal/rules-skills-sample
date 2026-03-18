---
name: code-review
description: Review Python/FastAPI code against this project's standards and flag violations with actionable fixes. Use this skill whenever the user asks to review code, check a PR, audit a file, review a diff, check if something follows conventions, or asks "is this right / does this look good". Triggers on: "review this", "check this code", "PR review", "does this follow our standards", "audit this file", "what's wrong with this", or pasting code and asking for feedback. Always use this skill for code review — it checks against the real project rules for async correctness, logging (ContextVar), Pydantic v2, error handling, structure, and testing coverage.
---

# Code Review

Review the provided code against this project's standards. Produce a structured report: blocking issues first, then warnings, then suggestions. Every finding must include the rule violated and an exact fix.

The user provides code, a diff, a file path, or a description of a PR.

---

## Review Checklist

Work through every section below. Skip a section only if it's clearly not applicable (e.g. no DB code → skip DB section). Do not silently skip — note "N/A" for the section if so.

---

### 1. Async correctness ❗ (blocking)

These are hard failures — merge blockers.

| Check | What to look for |
|---|---|
| Sync def in async context | `def` instead of `async def` on any route handler or service method |
| Blocking I/O | `requests.get`, `open()`, `psycopg2`, any sync DB driver used directly in async code |
| `time.sleep` | Must be `await asyncio.sleep(...)` |
| `asyncio.run()` inside running loop | Raises `RuntimeError` — use `await` instead |
| Missing `await` | Calling a coroutine without `await` returns a coroutine object, not the result |
| Sync fixtures in async tests | Must use `pytest_asyncio.fixture`, not `pytest.fixture` for async fixtures |

Flag any of these as **BLOCKING**.

---

### 2. Logging & ContextVar ❗ (blocking)

| Check | What to look for |
|---|---|
| `ctx` / `request_id` passed as argument | Any function signature containing `ctx`, `request_id`, or similar logging context — this is the old pattern |
| `print()` used | Must use `logger` |
| Logger created correctly | Each module must have `logger = logging.getLogger(__name__)` |
| `request_id_var.get()` missing `extra=` | Logging calls must include `extra={"request_id": request_id_var.get()}` |
| `request_id_var.reset(token)` missing | Every `.set()` must have a `.reset(token)` in a `finally` block |
| PII / secrets in logs | Flag any log line that could emit tokens, passwords, or personal data |

Example of the **correct** pattern:
```python
from middleware.logging import request_id_var
logger = logging.getLogger(__name__)

async def my_service_method():
    logger.info("Doing work", extra={"request_id": request_id_var.get()})
```

Example of the **wrong** pattern (must be flagged as BLOCKING):
```python
async def my_service_method(self, ctx, ...):   # ❌ ctx threaded as arg
    log_with_request_id(ctx, "Doing work")
```

---

### 3. Pydantic v2 & schemas ⚠️ (warning)

| Check | What to look for |
|---|---|
| Pydantic v1 patterns | `class Config:` instead of `model_config = ConfigDict(...)` |
| Missing `extra="forbid"` on request models | Request schemas must reject unknown fields |
| Missing `Field(description=...)` | Required on every public / LLM-facing field |
| Schema reuse across roles | A single model used as both Request and Response — must be split |
| Deeply nested response | Flag if response schema nests more than 2 levels — suggest flattening |
| `DTO` used in API response | DTOs are internal; never return them from a router |

---

### 4. API / Router layer ⚠️ (warning)

| Check | What to look for |
|---|---|
| Business logic in router | DB calls, computation, or conditionals beyond input validation in the handler |
| `HTTPException` raised in service | Services must raise domain exceptions (`ValueError`, `PermissionError`); only routers/middleware map to HTTP codes |
| Missing response envelope | Response must be `{ "status", "data", "meta" }` |
| Non-versioned path | Route must start with `/v1/` or `/v2/` |
| List endpoint missing pagination | `limit`, `offset`, `filters` required |
| Verb in URL | `/getInsights` → `/insights` |
| Heavy work in handler | Should delegate to service/worker and return `job_id` |

---

### 5. Error handling ⚠️ (warning)

| Check | What to look for |
|---|---|
| Stack trace returned to client | Any `traceback`, `exc_info`, or raw exception string in a response body |
| Bare `except:` or `except Exception:` without logging | Silent swallowing of errors |
| `logger.error` without `exc_info=True` | Stack trace won't be captured |
| Missing global exception handler | Ensure `@app.exception_handler(Exception)` is registered |
| LLM output used without validation | Raw LLM string returned directly — must be parsed and validated |

---

### 6. Project structure 💡 (suggestion)

| Check | What to look for |
|---|---|
| Relative imports | Must use absolute imports only |
| Circular imports | Flag if visible from the diff/code |
| Config not in `core/config.py` | Hardcoded values or env reads scattered across modules |
| Wrong layer for code | Business logic in `api/`, HTTP logic in `services/`, etc. |
| `snake_case` violations | Files, functions, variables must be `snake_case`; classes `PascalCase` |

---

### 7. Testing coverage 💡 (suggestion)

| Check | What to look for |
|---|---|
| `TestClient` used | Must be replaced with `httpx.AsyncClient` |
| Sync test for async code | Missing `@pytest.mark.asyncio` |
| Missing `request_id` fixture | Test calls async service without seeding `request_id_var` |
| No failure case tests | Only happy-path tests — must also cover 422, 401, 404 |
| Real LLM / external API called | Must be mocked |
| No coverage threshold | `pytest --cov-fail-under=80` must be enforced in CI |

---

## Output format

Structure the review as:

```
## 🔴 Blocking Issues  (merge blockers)
## 🟡 Warnings         (should fix before merge)
## 🔵 Suggestions      (nice to have / tech debt)
## ✅ Looks Good        (what was done well)
```

For every finding:

1. **Rule** — which standard is violated (link to the relevant `.md` rule file by name)
2. **Location** — file + line number if known, or code snippet
3. **Problem** — one sentence
4. **Fix** — concrete corrected code snippet, not just a description

If there are zero issues in a category, write `None found.` — never skip the heading.

End with a **Summary** line:

> `X blocking · Y warnings · Z suggestions`

---

## Severity guide

| Severity | Meaning | Action |
|---|---|---|
| 🔴 Blocking | Correctness, security, or standards violation | Must fix before merge |
| 🟡 Warning | Degrades maintainability or violates a convention | Should fix; flag in PR |
| 🔵 Suggestion | Improvement or cleanup | Optional; track as tech debt |
