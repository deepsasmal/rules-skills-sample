# Database Rules

## Connections
- Use async DB drivers: `asyncpg` for Postgres, `motor` for MongoDB, `aiomysql` for MySQL
- **Never use sync drivers** in async code — they block the event loop
- Use connection pooling; configure pool size per environment
- All DB calls must be `await`ed

## Query Patterns
- Use parameterized queries or ORM — never string-interpolated SQL
- Keep queries in the service layer, not in API handlers
- Use `EXPLAIN ANALYZE` during development for slow queries
- Add DB indexes for all foreign keys and frequently filtered columns

## Migrations
- Use Alembic for schema migrations
- Migrations must be **reviewed** before merging — never auto-run in production without approval
- Each migration must be reversible (`upgrade` + `downgrade`)

## Transactions
- Wrap multi-step writes in explicit transactions
- Use `try/except/rollback` or context managers — never leave transactions open on error

## Naming Conventions
- Tables: `snake_case`, plural (e.g. `user_events`)
- Columns: `snake_case`
- Avoid reserved SQL keywords as column names

## Semantic Layer
- DB schema must not be exposed directly in API responses
- Map through the semantic / dbt layer
