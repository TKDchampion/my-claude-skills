# Skills

Skill entry points for Claude Code. Each subdirectory contains a `SKILL.md` that Claude loads and executes when the corresponding skill is invoked.

## Available Skills

| Skill | Trigger | Description |
|---|---|---|
| `backend-agent` | `/backend-agent` | Full-cycle Backend Engineer Agent (FastAPI + PostgreSQL) |
| `frontend-agent` | `/frontend-agent` | Full-cycle Frontend Engineer Agent (React 19 + TypeScript) |

## How Skills Work

`SKILL.md` at this level reads all files under `.claude/commands/` and exposes each as an individual executable skill.

Each agent skill (`backend-agent/SKILL.md`, `frontend-agent/SKILL.md`) orchestrates the full 6-step common workflow with domain-specific additions:

```
01 Analyze → 02 Plan → 03 Implement → 04 Test → 05 Review → 06 Deliver
```

Steps must execute in order. The agent waits for user confirmation before advancing to the next step.

## Related

- `../commands/` — individual slash commands deployed to target projects
- `../common-workflow/` — shared step definitions referenced by agent skills
