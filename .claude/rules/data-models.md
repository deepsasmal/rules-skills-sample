# Data Models Rules

## General
- Use **Pydantic v2** for all schemas
- Separate model types:
  - `XxxRequest` — incoming API payloads
  - `XxxResponse` — outgoing API responses
  - `XxxDTO` — internal service-layer transfer objects

## Naming & Conventions
- All field names: `snake_case`
- Always add `Field(description=...)` for LLM-facing or public schemas

## Validation
- Use Pydantic field constraints: `min_length`, `max_length`, `pattern`, `Enum`
- Validate strictly — use `model_config = ConfigDict(strict=True)` where appropriate
- Reject unknown fields: `model_config = ConfigDict(extra="forbid")`

## LLM Compatibility
- Keep schemas **flat, simple, and self-descriptive**
- Avoid deeply nested structures in responses
- Add `description` to every field exposed to LLMs

## Example
```python
class InsightRequest(BaseModel):
    model_config = ConfigDict(extra="forbid")

    query: str = Field(..., description="Natural language query from the user")
    filters: dict | None = Field(default=None, description="Optional key-value filters")
    user_context: dict = Field(default_factory=dict)
```

## Versioning
- Never break existing schema contracts
- Introduce a new schema version when a breaking change is required
