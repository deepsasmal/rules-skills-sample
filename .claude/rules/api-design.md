---
description: "API design standards: versioning, naming, async rules, request/response envelope, pagination."
alwaysApply: false
globs: "api/**"
---

# API Design Rules

## Versioning & Naming
- All APIs must be versioned: `/v1/`, `/v2/`
- Use **nouns, not verbs** and plural resources:
  - ✅ `/v1/insights` ❌ `/v1/getInsights`
- Follow RESTful design unless building LLM-specific endpoints

## LLM-Friendly Endpoints
- Expose structured NL-compatible endpoints: `/v1/query`, `/v1/semantic-search`, `/v1/insights/generate`
- Accept both structured and natural language inputs
- Responses must be deterministic and structured

## Request / Response
- Use Pydantic schemas for all inputs and outputs — strict validation required
- Standard response envelope:
  ```json
  { "status": "success|error", "data": {}, "meta": {} }
  ```
- Add `request_id` to every request (injected by middleware — see logging rules)

## Async Rules
- All endpoints **must be `async def`**
- No blocking I/O — use `asyncio`-compatible libraries or run blocking code via `asyncio.run_in_executor`
- Never use `time.sleep` — always `await asyncio.sleep`

## Orchestration
- API layer: validate input → trigger service/task → return response or `job_id`
- Heavy computation must **not** happen in the API handler; delegate to services or workers

## Pagination & Filtering
- Required for all list endpoints: `limit`, `offset`, `filters`
