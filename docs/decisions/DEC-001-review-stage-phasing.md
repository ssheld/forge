# Decision: Phase review into pre-PR (local) then post-PR (multi-node)

Status: accepted
Date: 2026-03-30
Decision needed from: Stephen Sheldon

## Decision Needed
- Recommendation: Implement pre-PR review in Phase 1, add post-PR review
  with GitHub provenance in Phase 2
- Default if no decision: Pre-PR only (simpler, no event infrastructure)
- Target date: Phase 1 MVP

## Context

Forge needs automated code review dispatch. Research into existing tools
([Overstory](https://github.com/jayminwest/overstory),
[Agent Orchestrator](https://github.com/ComposioHQ/agent-orchestrator),
[CrewAI](https://github.com/crewAIInc/crewAI),
[AutoGen](https://github.com/microsoft/autogen),
[OpenClaw](https://github.com/openclaw/openclaw),
[ide-agent-kit](https://github.com/ThinkOffApp/ide-agent-kit),
[night-watch-cli](https://github.com/jonit-dev/night-watch-cli),
[better-symphony](https://github.com/SabatinoMasala/better-symphony),
[claude-hub](https://github.com/claude-did-this/claude-hub),
[Composio PR cookbook](https://docs.composio.dev/cookbooks/pr-review-agent))
found that no tool handles the full issue-to-PR-to-review-to-merge loop
using subscription-authenticated local agents.

Two fundamentally different review architectures exist:

- **Pre-PR (Overstory model):** Agent implements, Forge reviews locally,
  PR created only after review passes. Simple, no event infrastructure.
- **Post-PR (ide-agent-kit / night-watch model):** Agent creates PR, Forge
  detects it (polling or webhook), dispatches reviewer. Review visible on
  GitHub, supports multi-node hardware.

## Options Considered
- Option A: Pre-PR review only
  - Benefits: Simpler (no webhooks/polling), Forge controls full loop,
    easy to swap reviewer models, PRs arrive clean
  - Risks / trade-offs: No GitHub review trail, implementer and reviewer
    must share a machine, no multi-node support
- Option B: Post-PR review only
  - Benefits: GitHub provenance (visible review trail), multi-node
    hardware support (Jetson implements, MBP reviews), aligns with how
    teams work
  - Risks / trade-offs: Requires event infrastructure (polling or webhook),
    more complex to implement, PRs hit GitHub before review
- Option C: Both, phased
  - Benefits: Phase 1 is simple to ship, Phase 2 adds provenance and
    multi-node. Pre-PR acts as lint pass, post-PR is the real review.
  - Risks / trade-offs: Two review systems to maintain, potential
    confusion about which review is authoritative

## Decision
- Outcome: Option C -- both, phased. Pre-PR review in Phase 1, post-PR
  review added in Phase 2.
- Accepted by: Stephen Sheldon
- Accepted on: 2026-03-30

## Consequences
- Positive: Phase 1 stays simple (no event infrastructure), Phase 2
  unlocks multi-node hardware modularity and GitHub provenance. Human
  owner remains final gatekeeper in both phases.
- Negative: Two review systems to maintain long-term. Need clear policy
  about which review is authoritative when they disagree.

## Sources
- [Review Orchestration Design Doc](../review-orchestration.md)
- [Overstory](https://github.com/jayminwest/overstory) -- pre-PR review
  with SQLite mail coordination
- [ide-agent-kit](https://github.com/ThinkOffApp/ide-agent-kit) -- webhook
  dispatch to local tmux agents
- [night-watch-cli](https://github.com/jonit-dev/night-watch-cli) --
  cron-based polling + worktree dispatch

## Follow-Up
- [ ] Tighten mvp-design.md Review stage with phased approach
- [ ] Define reviewer isolation mechanism (tool guards vs prompt-only)
- [ ] Decide polling vs webhook for Phase 2 event detection
- [ ] Define reviewer model preferences per repo
