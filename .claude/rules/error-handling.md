# Error Handling Rules

## Standard Error Response
```json
{
  "status": "error",
  "error": {
    "code": "ERROR_CODE",
    "message": "Human-readable message"
  }
}
```

## Principles
- **Never** expose internal stack traces in API responses
- Use centralized exception middleware to catch and convert all exceptions
- Stack traces are logged internally (with `request_id`) but never returned to clients

## HTTP Status Codes
| Code | Meaning |
|------|---------|
| 400 | Validation error |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not found |
| 422 | Unprocessable entity (Pydantic) |
| 429 | Rate limited |
| 500 | Internal server error |

## Exception Middleware (FastAPI)
```python
@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    # request_id already in context via ContextVar (see logging.md)
    logger.error("Unhandled exception", exc_info=exc)
    return JSONResponse(status_code=500, content={"status": "error", "error": {"code": "INTERNAL_ERROR", "message": "An unexpected error occurred"}})
```

## Retryable Errors
- Mark errors as retryable in metadata for Celery/worker consumers
- Include `retry_after` hint where applicable

## LLM Output Validation
- Always validate LLM responses against expected schema before returning
- Handle malformed/hallucinated outputs gracefully — return a structured error, never a raw LLM string
