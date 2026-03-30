# Forge

Forge is a local-first workflow controller for parallel coding agents.

The goal is to speed up software delivery without forcing a team into a
heavyweight agent platform, paid model APIs, or repo-specific workflow glue.
Forge should work across projects by orchestrating the tools a developer
already uses: git, worktrees, GitHub issues and pull requests, and coding
agents such as Codex, Claude Code, and optional local models.

## Design Goals

- Local-first operation
- Repo-agnostic workflow contracts
- Parallel issue lanes with clear overlap rules
- Human approval on material trade-offs
- Public-safe project docs and policies
- No required paid API dependencies beyond tools the user already subscribes to

## Non-Goals

- Replacing git, GitHub, or CI
- Acting as a general-purpose autonomous agent platform
- Storing private session transcripts inside the repo by default
- Requiring LangGraph or another workflow runtime in v1

## Planned Shape

The first version should be a thin CLI that:

- reads a small per-repo config file
- manages issue worktrees and branch conventions
- routes work to configured agent providers
- enforces review and merge gates
- normalizes GitHub post formatting

See [docs/architecture.md](docs/architecture.md) and
[docs/roadmap.md](docs/roadmap.md) for the current working plan, and
[docs/mvp-design.md](docs/mvp-design.md) for the explicit MVP scope and
north-star relationship.

## Project Docs

- `AGENTS.md` / `CLAUDE.md` — mirrored agent policy files (CI-enforced);
  contain coding standards, project commands, PR authoring rules, and review
  format templates
- `.github/pull_request_template.md` — GitHub auto-fills this into every PR
- `docs/decisions/` — durable architectural and workflow decisions
  (see `template.md` for the expected format)
