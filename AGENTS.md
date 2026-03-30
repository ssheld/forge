# AGENTS.md

This repository intentionally keeps agent guidance lean and public-safe.

## Purpose

Forge is meant to be reusable across repositories and teams. Do not turn
this repo into a dumping ground for verbose private notes, session transcripts,
or project-specific working memory.

## Core Principles

- Keep the repo public-safe by default.
- Prefer small, reversible changes over broad speculative refactors.
- Build a thin controller before introducing a framework.
- Treat git, worktrees, and the issue tracker as the primary workflow
  substrate.
- Keep repo-specific behavior in config, not hard-coded logic.

## Public-Safe Documentation Rule

- Do not commit daily logs, handoff notes, private scratchpads, or long
  session transcripts to this repository.
- If private working notes are needed during development, keep them in ignored
  local paths outside the committed docs set.
- Durable decisions that matter to users or contributors belong in public docs
  such as `README.md`, `docs/architecture.md`, `docs/roadmap.md`, or
  `docs/decisions/`.

## Default Workflow

Use this sequence for non-trivial work:

1. Clarify the task and identify any material trade-offs.
2. Check whether an existing file, command, or design already covers most of
   the need.
3. Write or update a short plan before changing code when the task spans
   multiple steps.
4. Implement in small coherent increments.
5. Run the relevant verification commands before calling the work done.
6. Update public docs when behavior, architecture, or workflow expectations
   changed.

## Human Decision Gate

Do not silently choose material trade-offs.

Raise the decision to the owner when the choice would materially affect:

- architecture
- workflow
- public API or config shape
- dependency policy
- security posture
- future extensibility

Routine local implementation details do not need escalation.

## Verification Loop

Before reporting substantive work as complete:

1. Run the most relevant tests for the changed behavior.
2. Run any available static checks or linting relevant to the change.
3. Review the diff for accidental edits, debug leftovers, and doc drift.
4. State clearly what was verified and what was not.

## GitHub Attribution

When posting GitHub issues, comments, or review feedback through an agent,
identify the agent clearly instead of posting as if the human account owner
wrote it personally.

Preferred format for non-review issue or PR comments:

```md
> 🤖 **Post by {Model Name}** · via {Client Tool}
```

Preferred format for PR review summaries:

```md
> 🤖 **Review by {Model Name}** · via {Client Tool}
```

## Pull Request Expectations

- Keep pull requests coherent and scoped to one logical change.
- Include a real summary, not a one-line placeholder.
- Include verification evidence.
- Note any remaining risks or intentionally deferred follow-up work.

## Default Build Direction

Until the repo records a different decision:

- Build a thin local CLI first.
- Prefer plain Python and straightforward subprocess/file orchestration.
- Use git worktrees and GitHub as the durable workflow substrate.
- Keep provider integrations behind small adapters.
- Do not introduce LangGraph or another workflow engine unless the simpler
  controller demonstrably stops scaling.
