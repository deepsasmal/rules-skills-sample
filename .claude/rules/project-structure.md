# Project Structure Rules

## Folder Layout
```
app/
├── api/          # FastAPI routers (orchestration only)
├── schemas/      # Pydantic request/response/DTO models
├── services/     # Business logic
├── workers/      # Celery / background tasks
├── db/           # DB connections, repositories
├── core/         # Config, settings, lifespan
├── middleware/   # Logging, auth, request_id middleware
└── utils/        # Pure helper functions
```

## Separation of Concerns
- **API** → validate input, call service, return response or `job_id`
- **Services** → all business logic; no HTTP concerns
- **Workers** → heavy / async background execution
- **Middleware** → cross-cutting concerns (logging, auth, request_id injection)

## Naming Conventions
- Files & functions: `snake_case`
- Classes: `PascalCase`
- Constants: `UPPER_SNAKE_CASE`

## Imports
- Use **absolute imports** only
- No circular dependencies — enforce with `import-linter` or `ruff`

## Config Management
- All config via environment variables
- Centralize in `core/config.py` using `pydantic-settings`:
  ```python
  class Settings(BaseSettings):
      db_url: str
      redis_url: str
      model_config = SettingsConfigDict(env_file=".env")
  ```

## Scalability
- API and workers must be independently deployable services
- Avoid shared in-process state between requests
