# Authentication & Authorization Rules

## Authentication
- JWT-based auth; support OAuth (Google, Microsoft)
- Tokens must be **short-lived** (access: 15m, refresh: 7d)
- All endpoints require auth except `/health` and `/metrics`
- Service-to-service communication uses API keys

## Authorization
- Implement RBAC: `Admin`, `Analyst`, `Viewer`
- Enforce **row-level and column-level** data access
- Users must only access permitted datasets

## Secrets Management
- Store secrets in environment variables or a secrets manager (e.g. AWS Secrets Manager, Vault)
- **Never hardcode credentials** — enforced via CI lint checks

## Audit Logging
- Log: user actions, queries executed, data accessed
- Always include `request_id` and `user_id` (provided by logging middleware)
