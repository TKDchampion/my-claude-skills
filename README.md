# my-claude-skills

A collection of custom Claude Code slash commands, agent skills, and workflow guides. Files here are deployed to `.claude/` in target projects to enforce consistent architecture, workflow, and quality standards.

## What's Inside

```
.claude/
├── .claude/agents-plan/      # Agent will create plans to here automatically
├── commands/         # Slash command definitions
├── skills/           # Agent skill entry points (SKILL.md per agent)
└── common-workflow/  # Six-step workflow fragments shared by agent skills
```

## Slash Commands

Deploy to `.claude/commands/` in a target project. Each file becomes a `/command`.

| Command                  | Description                                                                            |
| ------------------------ | -------------------------------------------------------------------------------------- |
| `/git-commit`            | Conventional commit flow with confirmation gate — never skips hooks                    |
| `/git-review`            | Pre-commit review: tests, security, code quality — presents report, asks before fixing |
| `/backend-feature`       | FastAPI feature implementation enforcing layered architecture                          |
| `/backend-architecture`  | New backend project scaffold (Flat / Feature-based / DDD)                              |
| `/frontend-feature`      | React 19 + TypeScript feature implementation rules                                     |
| `/frontend-architecture` | New frontend project scaffold and architecture plan                                    |
| `/security-review`       | Deep security review: OWASP, Trail of Bits, DB, race conditions, AI leakage            |
| `/owasp-security`        | OWASP Top 10:2025 + ASVS 5.0 + Agentic AI security reference                           |
| `/figma-agent`           | Extract Figma designs and translate them into production-ready UI code                 |
| `/hello`                 | Smoke-test command                                                                     |

## Agent Skills

Full-cycle delivery agents that orchestrate the 6-step common workflow end-to-end.

### `/backend-agent` — ex. FastAPI + PostgreSQL

Invoke when implementing a backend feature or scaffolding a new service.

Layer order: `Entity → DTO → Repository → Domain → Service → Router → main.py`

Key rules:

- Repository: `db.flush()` only — `@db_tx` handles commits
- Service: business logic wrapped in `@db_tx`
- Router: every endpoint requires `@router_try()` + `verify_user_permission`
- External services: extend `BaseHTTPService`, decorate with `@external_api`
- Run `alembic revision --autogenerate` after every Entity change

### `/frontend-agent` — ex. React + TypeScript

Invoke when implementing a frontend feature or building UI from a Figma design. Paste a Figma URL to auto-trigger design extraction before Step 01.

Layer order: `Types → Constants → API layer → Hooks → Presentational → Form → Page → Route → Barrel export`

Key rules:

- Styling: Tailwind CSS only — no inline styles, no CSS modules
- Forms: React Hook Form + Zod, schema in `[Feature]Form.schema.ts`
- API layer: extend `HttpClient`; use key factories in `[feature].keys.ts`
- Mutations: `onSuccess` must call `invalidateQueries`
- Shared components: any UI element used 2+ times goes to `shared/components/`

## Common Workflow (6 Steps)

Both agent skills follow this ordered workflow. Each step requires user confirmation before advancing — the agent never auto-advances.

| Step | File                                          | Description                                                                   |
| ---- | --------------------------------------------- | ----------------------------------------------------------------------------- |
| 01   | `01-analyze-qa.md`                            | Analyze requirements, clarify ambiguities, produce requirement summary        |
| 02   | `02-plan-and-split-steps.md`                  | Create git branch, write `.claude/agents-plan/[feature]/plan-step-N.md` files |
| 03   | `03-implement-plan-step-loop.md`              | Implement one plan step at a time in layer order                              |
| 04   | `04-test-loop.md`                             | Test, fix bugs, repeat until green                                            |
| 05   | `05-code-review-security-and-improvements.md` | Deep code review + security audit                                             |
| 06   | `06-explain-changes-and-final-review.md`      | Final summary and delivery confirmation                                       |

## Usage

Copy `.claude/` into the root of any target project:

```bash
cp -r .claude/ /path/to/your-project/.claude/
```

Then use the commands and skills in Claude Code within that project.
