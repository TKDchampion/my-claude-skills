# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

A collection of custom Claude Code slash commands, skills, and workflow guides. Files here are deployed to `.claude/` in target projects — they define how Claude should behave in those projects, not this one.

## Repository Structure

```
commands/         # Slash command definitions → deployed to .claude/commands/
skills/           # Skill definitions with README + SKILL.md
common-workflow/  # Six-step numbered workflow fragments referenced by agent skills
```

### `commands/`

Each `.md` file becomes a slash command (`/filename`) in target projects:

| File | Purpose |
|---|---|
| `git-commit.md` | Conventional commit flow with confirmation gate |
| `git-review.md` | Pre-commit review: tests, security, code quality |
| `backend-feature.md` | FastAPI feature implementation with layered architecture rules |
| `backend-architecture.md` | New backend project scaffold (Flat/Feature-based/DDD selection) |
| `frontend-feature.md` | React/TypeScript feature implementation rules |
| `frontend-architecture.md` | New frontend project scaffold and architecture plan |
| `security-review.md` | Deep security review: OWASP, Trail of Bits, DB, race conditions, AI leakage |
| `owasp-security.md` | OWASP Top 10:2025 + ASVS 5.0 + Agentic AI security reference |
| `hello.md` | Test command |

### `common-workflow/`

Six sequential step files referenced by agent skills. Steps must execute in order with user confirmation between each:

1. `01-analyze-qa.md` — Analyze requirements, clarify ambiguities, produce requirement summary for confirmation
2. `02-plan-and-split-steps.md` — Determine project/task type, create git branch, write `agents-plan/[feature]/plan-step-N.md` files
3. `03-implement-plan-step.md` — Implement one plan step at a time
4. `04-test-loop.md` — Test, fix bugs, repeat until green
5. `05-code-review-security-and-improvements.md` — Deep code review + security audit
6. `06-explain-changes-and-final-review.md` — Final summary, delivery confirmation

### `skills/`

- `SKILL.md` — Entry point: reads all files under `.claude/commands/` and executes them as individual skills
- `backend-agent/SKILL.md` — Full backend delivery agent: runs all 6 common-workflow steps with FastAPI-specific additions (migration review in Step 03-B, API contract verification in Step 04-B)
- `frontend-agent/SKILL.md` — Full frontend delivery agent: runs all 6 common-workflow steps with React-specific additions (shared component audit in Step 03-B, visual review in Step 04-B)

## Architecture Conventions Encoded in Commands

### Backend (FastAPI + PostgreSQL)

Layer order for feature implementation: **Entity → DTO → Repository → Domain → Service → Router → main.py registration**

Layer boundaries (enforced by `backend-feature.md`):
- Router: HTTP in/out only, calls Service, always has `@router_try()` + `verify_user_permission`
- Service: business logic only, uses `@db_tx` for transactions, calls Repository + Domain
- Repository: DB CRUD only, uses `db.flush()` never `db.commit()`
- Domain: pure functions, zero external dependencies
- DTO: Pydantic types only, no logic
- Entity: SQLAlchemy ORM models only, no logic

Core decorators: `@router_try()`, `@db_tx`, `@external_api`. External services extend `BaseHTTPService`.

Error handling: `DomainException(msg, type, code)` raised in Domain/Service, caught by `@router_try()` in Router.

### Frontend (React 19 + TypeScript)

Layer order for feature implementation: **types → constants → API layer → hooks → presentational components → form components → page components → route registration → barrel export**

Layer boundaries (enforced by `frontend-feature.md`):
- API layer: extends `HttpClient`, uses query key factories, no business logic
- Hooks: one hook per concern — query/mutation/UI state separated
- Presentational components: pure UI, props only, no direct API calls
- Page components: data fetching and assembly only, no UI logic
- Shared components: any UI element used 2+ times must go to `shared/components/`

Styling: Tailwind CSS exclusively — no inline styles, no CSS modules.
Forms: React Hook Form + Zod, schema defined in `[Feature]Form.schema.ts`.
Mutations: `onSuccess` must call `invalidateQueries`.

## Workflow Rules

**`02-plan-and-split-steps.md` creates `agents-plan/[feature]/plan-step-N.md` files** in the target project before any code is written. Never skip this step.

**Each step requires user confirmation** before proceeding to the next. The agent must not auto-advance through the 6-step workflow.

**`git-commit.md`**: Never commits without user confirmation. Never uses `--no-verify`.

**`git-review.md`**: Never auto-fixes or commits. Always asks what to do after presenting the report.

**`security-review.md`**: Always asks for scope (full/diff/files) before starting. Never auto-fixes without confirmation.
