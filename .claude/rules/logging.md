# Logging Rules

## Approach: Centralized Middleware + ContextVars

Logging is centralized via FastAPI middleware. A `request_id` is injected **once** at the request boundary using a `ContextVar` — no `ctx` or `request_id` argument is passed through function signatures.

### Why ContextVars (not a plain global)
Each `asyncio` task gets its own **isolated copy** of a `ContextVar`. Two concurrent requests setting `request_id_var` never bleed into each other, unlike a plain module-level global.

---

## Setup

### `middleware/logging.py`
```python
import uuid
import logging
from contextvars import ContextVar
from starlette.middleware.base import BaseHTTPMiddleware

request_id_var: ContextVar[str] = ContextVar("request_id", default="unknown")

logger = logging.getLogger(__name__)

class RequestLoggingMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request, call_next):
        request_id = request.headers.get("X-Request-ID", str(uuid.uuid4()))
        token = request_id_var.set(request_id)          # isolated per async task
        try:
            logger.info("Request started", extra={"path": request.url.path})
            response = await call_next(request)
            response.headers["X-Request-ID"] = request_id
            logger.info("Request completed", extra={"status": response.status_code})
            return response
        finally:
            request_id_var.reset(token)                 # clean up after request
```

### `core/main.py`
```python
from middleware.logging import RequestLoggingMiddleware
app.add_middleware(RequestLoggingMiddleware)
```

---

## Logging in Services / Helpers

Any module can read `request_id` without accepting it as an argument:

```python
from middleware.logging import request_id_var
import logging

logger = logging.getLogger(__name__)

def log(level: str, message: str, **extra):
    extra["request_id"] = request_id_var.get()
    getattr(logger, level)(message, extra=extra)

# Usage — no ctx, no argument threading:
async def _execute_query(query_string: str):
    log("info", "Executing query", query=query_string)
    ...
```

---

## Log Format

Use **structured JSON logging** in production:

```python
# core/logging_config.py
import logging.config

LOGGING = {
    "version": 1,
    "formatters": {
        "json": {"()": "pythonjsonlogger.jsonlogger.JsonFormatter",
                 "format": "%(asctime)s %(levelname)s %(name)s %(message)s"}
    },
    "handlers": {
        "default": {"class": "logging.StreamHandler", "formatter": "json"}
    },
    "root": {"level": "INFO", "handlers": ["default"]},
}
logging.config.dictConfig(LOGGING)
```

Every log line automatically includes `request_id` via the `extra` dict above.

---

## Rules
- **Never** pass `request_id` or a logging context as a function argument
- **Never** use `print()` — always use `logger`
- **Never** log secrets, tokens, or PII
- Log levels: `DEBUG` (dev only), `INFO` (normal flow), `WARNING` (recoverable issues), `ERROR` (failures with stack trace)
- Always use `logger.error(..., exc_info=True)` for exceptions so the stack trace is captured
- In async code, always `await` — never use `asyncio.run()` inside a running loop
