# Architecture

## Summary

Forge should begin as a thin local CLI that coordinates repository work
across issues, worktrees, agents, and GitHub states. The first version should
optimize for clarity and portability rather than autonomy or framework
complexity.

The longer-term direction is a node-aware software delivery control plane, but
the MVP should stay focused on the local-first issue-to-PR loop. See
[docs/mvp-design.md](mvp-design.md) for the full v1 contract.

## v1 Components

### CLI

The main entry point should be a local command-line tool, likely exposed as
`forge`.

Responsibilities:

- read repo config
- inspect issue state
- create and clean up worktrees
- launch agent tasks through provider adapters
- record state back to GitHub

### Repo Config

Each target repository should expose a small machine-readable config file,
likely `.forge.yaml`.

Example concerns:

- branch naming
- worktree root
- issue labels / state mapping
- overlap rules
- verification commands
- provider defaults

### Provider Adapters

Providers should be thin wrappers around tools the user already runs.

Initial provider targets:

- Codex
- Claude Code
- optional local sidecar model later

Providers should not own workflow state. They should execute bounded tasks and
return structured results.

### Workflow Backend

Forge should use existing systems before inventing new ones.

Initial durable state:

- git branches
- git worktrees
- GitHub issues
- GitHub pull requests
- labels and comments
- local SQLite state for runs and stage tracking

No always-on service is required in v1. A local SQLite store is acceptable and
expected for run metadata and resumability.

### Policy Layer

Forge should normalize how repos express workflow expectations.

That policy layer should cover:

- required verification steps
- GitHub attribution format
- merge gates
- parallelism rules
- optional repo-specific instructions

## Non-Goals For v1

- durable queue workers
- resumable background jobs
- generalized multi-agent planning graphs
- semantic memory infrastructure
- mandatory hosted services

## Likely Upgrade Trigger

If the tool later needs:

- long-running resumable jobs
- retries and escalation policies
- human approval checkpoints
- many concurrent lanes across repositories

then a workflow runtime such as LangGraph may become justified.
