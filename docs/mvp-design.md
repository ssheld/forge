# MVP Design

## Summary

Forge v1 should be a local-first workflow controller for AI-assisted
software delivery. The MVP should solve a narrow but real problem:
standardizing and speeding up the issue-to-PR loop across repositories without
requiring committed session logs, paid orchestration APIs, or a heavyweight
runtime.

The MVP is intentionally smaller than the long-term vision. It should prove
that Forge can replace the most painful manual orchestration steps while
preserving room to grow into a more capable node-aware control plane later.

## Why This Exists

Current AI coding workflows are fragmented and manual. A developer often has
to:

- choose an issue
- explain the task repeatedly to different agents
- route implementation and review manually
- track worktrees and branches manually
- reconcile review feedback manually
- remember repo-specific workflow rules manually

This is slow, inconsistent across repos, and difficult to parallelize safely.

## North Star

The long-term direction is a node-aware, environment-aware software delivery
control plane that can orchestrate work across local and remote nodes using a
mix of cloud coding agents and local models.

That long-term direction matters because it explains why Forge should keep
clean boundaries between:

- workflow orchestration
- agent/provider adapters
- node or execution environment routing
- durable state
- repo-specific policy

However, the MVP should not try to implement the whole control plane.

## MVP Goal

Forge v1 should make one developer materially faster at the real
issue-to-PR loop across multiple repositories.

It should do that by standardizing:

- issue intake
- implementation brief generation
- worktree and branch setup
- provider assignment
- review and fix routing
- merge-readiness checks

## What The MVP Must Prove

1. A thin controller can save time immediately without requiring a new
   platform.
2. Repo workflow rules can be normalized into machine-readable policy instead
   of long natural-language instruction files.
3. The useful parts of `agent-vault` can be preserved without committing
   operational memory sprawl into each repository.
4. Parallel issue lanes can be managed more safely through worktrees and
   explicit state.

## Non-Goals

The MVP should not attempt to provide:

- general-purpose autonomous agent orchestration
- task-source agnosticism beyond GitHub
- code-host agnosticism beyond GitHub
- unbounded multi-agent debate loops
- automatic prioritization of all incoming work
- full remote fleet orchestration
- semantic memory or RAG as a core dependency
- automatic merge without explicit human approval

## Primary Users

Initial target user:

- a solo engineer or very small team already using coding agents such as
  Codex and Claude Code

Initial target environment:

- one main development machine
- one or two repositories
- optional later validation node, but not required for MVP success

## Core Product Decisions

### 1. Local-First

Forge should run as a local CLI. It should not require a hosted service in
v1.

### 2. GitHub-First

The MVP should assume:

- GitHub Issues for intake
- GitHub Pull Requests for code review and delivery
- git worktrees for isolated execution lanes

Other trackers and code hosts can be added later through adapters.

### 3. Explicit Workflow Stages

Forge should model bounded stages rather than free-form agent chat:

1. intake
2. brief
3. implement
4. review
5. fix
6. merge-ready

### 4. Policy Over Prompting

Important workflow rules should be machine-readable and enforced where
possible.

Examples:

- required verification commands
- GitHub attribution rules
- merge gates
- overlap rules
- default provider selection

### 5. Private Operational Memory

Forge should not require repositories to commit daily logs, handoff notes,
or session transcripts.

Instead:

- durable shared knowledge stays in repo docs
- operational run state stays local
- selected artifacts can be promoted into repo docs when they become durable

## MVP Architecture

### Repo Policy

Each repository should expose a small config file such as `.forge.yaml`.

This file should describe:

- branch naming convention
- worktree root or pattern
- verification commands
- overlap policy
- GitHub posting requirements
- provider defaults

### Local State Store

Forge should persist workflow state locally using SQLite plus an artifact
directory.

SQLite is for structured run metadata such as:

- work items
- runs
- stages
- approvals
- worktrees
- provider assignments
- node assignments later

Artifacts should remain file-based:

- briefs
- review reports
- logs
- patches
- command transcripts when useful

### Provider Adapters

Provider adapters should wrap tools the user already pays for or runs locally.

Initial providers:

- Codex
- Claude Code

Later optional providers:

- local Qwen or similar sidecar model
- other coding agents

### GitHub Adapter

The GitHub adapter should manage:

- issue reads
- PR creation or metadata capture
- issue and PR comments
- review state capture

### Worktree Manager

The worktree manager should:

- create isolated worktrees per work item
- map work items to branches
- detect stale or merged branches
- support safe cleanup

## Structured Artifacts

The MVP should use explicit artifact types rather than loose chat history.

Initial artifacts:

- `ImplementationBrief`
- `ReviewReport`
- `FixRequest`
- `RunDecision`
- `StageResult`

These can be JSON or Markdown-backed, but their shape should be stable.

## MVP Workflow

### Intake

- select a GitHub issue
- normalize it into a `WorkItem`
- assign repo policy and default provider

### Brief

- generate a concise implementation brief
- include scope, constraints, verification expectations, and known risks
- allow human adjustment before implementation if needed

### Implement

- create worktree and branch
- dispatch to selected provider
- capture outputs and artifacts

### Review (MVP: Pre-PR Local Review)

The MVP review stage is local and pre-PR:

- implementing agent signals done
- Forge dispatches a local reviewer on the same machine
- reviewer is read-only (tool guards enforce this, not just prompts)
- reviewer produces a structured `ReviewReport`
- reviewer model is configurable per repo (Claude, Codex, Qwen, etc.)
- no event infrastructure needed -- Forge controls the full loop

Open design question: whether pre-PR review blocks PR creation or is
advisory-only. This will be resolved during implementation. See
[DEC-001](decisions/DEC-001-review-stage-phasing.md) for context.

Post-PR review with GitHub provenance and multi-node dispatch is the
planned Phase 2 extension. See
[docs/review-orchestration.md](review-orchestration.md) for the full
design, research context, and landscape analysis.

The human owner remains the final merge gatekeeper regardless of phase.

### Fix

- route review findings back to the implementation provider
- rerun checks as needed
- update run state
- capped revision rounds (e.g., 3 attempts before escalation to human)

### Merge-Ready

- require explicit human approval
- confirm policy gates passed
- mark cleanup actions

## Agent-Vault Replacement Direction

Forge should keep the best parts of `agent-vault` and replace the rest.

Keep:

- architecture docs
- roadmap docs
- decision records
- verification discipline
- human decision gates

Replace:

- committed session logs
- daily notes
- handoff files
- scratchpads
- large repo-local operational memory

The replacement model should be:

- repo-shared durable context in docs
- local private operational state in Forge
- machine-readable repo policy for enforcement

## Success Criteria

The MVP is successful if it can:

1. manage at least one repository end-to-end
2. reduce manual work in the issue-to-PR loop
3. run at least two independent issue lanes safely through worktrees
4. enforce repo workflow rules more consistently than prompt-only guidance
5. do all of the above without committed session-memory sprawl

## Deferred Decisions

These are intentionally deferred until after the MVP proves value:

- LangGraph as the orchestration runtime
- SSH or remote worker transport
- non-GitHub task sources
- non-GitHub code hosts
- semantic retrieval or RAG
- automatic issue prioritization
- auto-merge for safe subsets

## Why The North Star Still Matters

The long-term node-aware control-plane direction should be recorded, but only
briefly.

It is useful because it explains:

- why the abstractions should stay clean
- why provider and execution concerns should stay separate
- why local run state matters

It should not be used as permission to overbuild the MVP.
