# Build with BE Agent Team

A full-cycle Backend Engineer Agent that executes the complete workflow from requirement analysis to delivery in FastAPI + PostgreSQL projects.

## When to Use

Invoke `/backend-agent` when you need to:
- Implement a new backend feature end-to-end
- Scaffold a new FastAPI service
- Make multi-layer changes (Entity → DTO → Repository → Service → Router)

## Workflow Overview

The agent follows a 6-step workflow with mandatory user confirmation between each step:

| Step | Name | Description |
|---|---|---|
| 01 | Analyze & Clarify | Break down requirements, clarify API contracts, DB schema, permissions, and async needs |
| 02 | Plan & Branch | Create git branch, generate `agents-plan/[feature]/plan-step-N.md` per layer |
| 03 | Implement | Execute each plan step in layer order with Alembic migration review (Step 03-B) |
| 04 | Test Loop | Static checks (mypy), Swagger manual testing, permission & transaction edge cases + API contract verification (Step 04-B) |
| 05 | Code Review | Security audit: auth guards, parameterized queries, credential exposure, error leakage |
| 06 | Final Delivery | Migration changelog, endpoint list, env var changes, frontend handoff notes |

## Layer Implementation Order

```
Entity → DTO → Repository → Domain → Service → Router → main.py registration
```

## Key Rules

- **Repository**: only `db.flush()`, never `db.commit()` — `@db_tx` handles commits
- **Service**: wrap transactions with `@db_tx`
- **Router**: every endpoint requires `@router_try()` + `verify_user_permission`
- **External services**: extend `BaseHTTPService`, decorate with `@external_api`
- **Migrations**: run `alembic revision --autogenerate` after every Entity change, review before continuing

## Architecture Reference

See `.claude/commands/backend-feature.md` for layer boundaries and decorator conventions.
See `.claude/commands/backend-architecture.md` for new project scaffolding.
