# plan-server

Local planning server — captures structured planning items as versioned Markdown files.

## Status

`[MVP]` — In design. Not yet built.

## Spec

See [`SYSTEM-PROMPT.md`](./SYSTEM-PROMPT.md) for the full system prompt and build specification.

## Release stages

| Stage | Scope |
|---|---|
| `[MVP]` | Project wizard + Idea capture form |
| `[MMP]` | Discussion + PRD forms, archiving, changelog, list view |
| `[MLP]` | OKF v0.2, Seven QC Tools view actions, optional classifications |
| `[PMF]` | Multi-project, agent scaffolding, SRS/TDD |

## Tech stack

- Node.js + Express
- Plain HTML + CSS
- Local filesystem (no database)
- Git (auto-initialized per project)
- Port 3000, localhost only
