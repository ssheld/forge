# Roadmap

## Phase 0 - Repo Bootstrap

- create public-safe repo guidance
- define the initial architecture
- choose config shape and CLI surface

## Phase 1 - Single-Repo Local CLI

Goal: manage one repository with one or two concurrent issue lanes.

Expected capabilities:

- initialize repo config
- create issue worktrees
- assign issue ownership to a provider
- track implementation and review states
- persist run state locally
- clean up merged branches and worktrees

## Phase 2 - Parallel Issue Lanes

Goal: safely run multiple independent issues in parallel.

Expected capabilities:

- overlap detection by path or tag
- lane status inspection
- merge-readiness checks
- review handoff helpers

## Phase 3 - Policy and Memory Adapters

Goal: normalize per-repo workflow rules without requiring a specific note
system.

Expected capabilities:

- generic policy file support
- optional adapter for richer repo-local instructions
- lightweight retrieval/index support for cold docs

## Phase 4 - Optional Durable Runtime

Goal: only if the thin CLI stops scaling.

Possible future work:

- resumable background tasks
- richer approval gates
- queueing and retries
- framework-backed orchestration
