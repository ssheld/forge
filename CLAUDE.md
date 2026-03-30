# CLAUDE.md

Follow [AGENTS.md](AGENTS.md) as the primary repo policy.

This file exists because some agents reliably read tool-specific instruction
files before or instead of `AGENTS.md`. Keep it short and keep it aligned.

## Claude-Specific Working Rules

- Keep the repository public-safe by default.
- Do not create session logs, daily notes, or handoff artifacts in the repo.
- Prefer direct, small, verifiable changes.
- Ask for owner direction when the choice would materially affect the tool's
  architecture, workflow model, or dependency policy.
- When posting to GitHub as an agent, clearly attribute the post with model
  name and client tool.

## Default Build Direction

Until the repo records a different decision:

- Build a thin local CLI first.
- Prefer plain Python and straightforward subprocess/file orchestration.
- Use git worktrees and GitHub as the durable workflow substrate.
- Keep provider integrations behind small adapters.
- Do not introduce LangGraph or another workflow engine unless the simpler
  controller demonstrably stops scaling.

## Docs Discipline

- Update public docs when behavior or architecture changes.
- Keep private experimentation notes outside the repo.
- Prefer concise architecture and roadmap documents over verbose process logs.
