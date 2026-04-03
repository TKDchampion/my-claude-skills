# Build with FE Agent Team

A full-cycle Frontend Engineer Agent that executes the complete workflow from requirement analysis to delivery in React 19 + TypeScript projects.

## When to Use

Invoke `/frontend-agent` when you need to:
- Implement a new frontend feature end-to-end
- Scaffold a new React project
- Build UI from a Figma design (paste a Figma URL to auto-trigger design extraction)

## Figma Detection

Before Step 01, the agent checks for a Figma URL in your input:
- **With Figma URL** → runs `figma-agent` Steps 0–1 to extract a Design Summary, then uses it as Step 01 input
- **No Figma URL** → proceeds directly to Step 01

## Workflow Overview

The agent follows a 6-step workflow with mandatory user confirmation between each step:

| Step | Name | Description |
|---|---|---|
| 01 | Analyze & Clarify | Break down requirements, clarify pages, APIs, forms, routing, state, and responsive needs |
| 02 | Plan & Branch | Create git branch, generate `agents-plan/[feature]/plan-step-N.md` per layer |
| 03 | Implement | Execute each plan step in layer order with shared component audit (Step 03-B) |
| 04 | Test Loop | Type-check, lint, build, runtime state testing, auth edge cases + visual review (Step 04-B) |
| 05 | Code Review | Security audit: XSS, hardcoded secrets, unvalidated input, protected routes, npm audit |
| 06 | Final Delivery | Route list, new shared components, API hooks, env var changes, bundle size impact |

## Layer Implementation Order

```
Types → Constants → API layer → Hooks → Presentational components → Form components → Page components → Route registration → Barrel export
```

## Key Rules

- **Styling**: Tailwind CSS only — no inline styles, no CSS modules
- **Forms**: React Hook Form + Zod, schema in `[Feature]Form.schema.ts`
- **API layer**: extend `HttpClient`, never use axios directly; use key factories in `[feature].keys.ts`
- **Mutations**: `onSuccess` must call `invalidateQueries`
- **Shared components**: any UI element used 2+ times must go to `shared/components/`
- **Props naming**: boolean props use `is/has/can` prefix; event handlers use `on` prefix

## Architecture Reference

See `.claude/commands/frontend-feature.md` for layer boundaries and naming conventions.
See `.claude/commands/frontend-architecture.md` for new project scaffolding.
