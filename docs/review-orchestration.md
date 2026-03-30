# Review Orchestration

How Forge detects, dispatches, and manages code review across local agents
and (eventually) multiple hardware nodes.

## Problem Statement

When an AI coding agent finishes implementing a feature and creates a PR,
someone or something needs to review it. Today, developers either:

- Manually tell another agent to review (defeats the purpose of automation)
- Rely on cloud services like
  [Codex Cloud](https://developers.openai.com/codex/pricing)
  or [Claude Code Action](https://github.com/anthropics/claude-code-action)
  (adds API billing or depends on free tiers that may shrink)
- Skip automated review entirely

Forge should automate review dispatch using local agents that run on
subscriptions the user already pays for, without requiring separate API
billing.

## Research

Extensive landscape research was conducted to inform this design. See:

- [Agent Orchestration Landscape](research/agent-orchestration-landscape.md) --
  tools surveyed, subscription auth mechanisms, platform comparison
- [Pipeline Architecture](research/pipeline-architecture.md) -- job
  structures, review scoring, iterative fix loops, quality gates

## Design Decision: Phased Review

See [DEC-001](decisions/DEC-001-review-stage-phasing.md) for the formal
decision record.

### Phase 1: Pre-PR Review (Local)

```
Agent implements in worktree
  -> Forge runs local review (same machine)
  -> Forge acts on review findings (exact behavior TBD -- see below)
```

- No event infrastructure needed (no webhooks, no polling)
- Forge controls the full loop -- the implementing agent signals done,
  Forge dispatches the reviewer locally
- Easy to swap reviewer models (point at a different provider)
- Inspired by [Overstory's](https://github.com/jayminwest/overstory)
  pre-PR review model with SQLite mail coordination

Whether Phase 1 review blocks PR creation or is advisory-only is an
open design question tracked in [issue #6](https://github.com/ssheld/forge/issues/6).

### Phase 2: Post-PR Review (Multi-Node, GitHub Provenance)

```
Agent creates PR (from any node)
  -> Forge detects PR (polling or webhook)
  -> Forge dispatches reviewer on available node
  -> Review posted to GitHub with full provenance
  -> Human approves or requests changes
```

- Review trail visible on GitHub (comments, severity labels, approvals)
- Enables multi-node hardware: a Jetson Orin implements, an MBP reviews
- GitHub becomes the coordination substrate between nodes
- Inspired by [ide-agent-kit's](https://github.com/ThinkOffApp/ide-agent-kit)
  webhook dispatch and
  [night-watch-cli's](https://github.com/jonit-dev/night-watch-cli)
  polling + worktree pattern
- Aligns with the north-star "node-aware control plane" from
  [docs/mvp-design.md](mvp-design.md)

### Why Both

Phase 1 is simpler to ship (no event infrastructure) and catches issues
before they hit GitHub. Phase 2 adds GitHub provenance (visible review
trail) and multi-node hardware support.

Phase 1 requires implementer and reviewer on the same machine. Phase 2
uses GitHub as the coordination layer between nodes, which is where the
hardware modularity kicks in.

Open design questions:
- Whether Phase 1 review blocks PR creation or is advisory-only
- How the two phases interact when both are active (e.g., does Phase 2
  repeat work that Phase 1 already did?)
- Whether the reviewer's authority level is policy-configurable per repo

These will be resolved during implementation rather than prematurely
locked down.

## Event Detection (Phase 2)

Two viable approaches for detecting PRs that need review:

### Polling (simpler, no infrastructure)
- Periodic `gh pr list` to find unreviewed PRs
- Filter by labels, CI status, review state
- 30-60 second intervals, configurable
- Used by: [Agent Orchestrator](https://github.com/ComposioHQ/agent-orchestrator),
  [night-watch-cli](https://github.com/jonit-dev/night-watch-cli),
  [better-symphony](https://github.com/SabatinoMasala/better-symphony)

### Webhooks (faster, requires inbound endpoint)
- GitHub POSTs `pull_request` events to a local HTTP server
- Requires public endpoint or tunnel (ngrok, Tailscale, Hookdeck)
- Used by: [ide-agent-kit](https://github.com/ThinkOffApp/ide-agent-kit),
  [claude-hub](https://github.com/claude-did-this/claude-hub)

### Recommendation

Start with polling in Phase 2 -- it's simpler to operate, requires no
public endpoint, and fits the "thin CLI" principle. Add optional webhook
support later if polling latency becomes a problem.

## Reviewer Isolation

The reviewer agent should be **read-only** with respect to the codebase
being reviewed. Overstory demonstrates this well:

- Mechanically enforce tool guards (block Write, Edit, destructive Bash)
  via Claude Code hooks, not just prompt instructions
- Reviewer can read files, run `git diff`, run quality gates
- Reviewer posts findings but cannot modify the code
- This prevents a reviewer from "fixing" issues and silently changing the
  implementation

## Open Questions

- Should Phase 1 pre-PR review block PR creation, or just annotate the PR
  with findings and let the human decide?
- How should Forge handle reviewer disagreements (pre-PR review passes but
  post-PR review finds issues)?
- What's the right default polling interval for Phase 2? (30s like Agent
  Orchestrator, or longer?)
- Should Forge support reviewer model preferences per repo (e.g., "use
  Qwen for this repo, Claude for that one")?
