# Agent Orchestration Landscape

Research into existing agent orchestration tools, automated PR review
dispatch, and subscription-based authentication. Conducted March 2026 to
inform Forge's design.

## Problem Statement

When an AI coding agent finishes implementing a feature, someone or
something needs to review it. The core challenge: **how to automatically
trigger a local CLI coding agent to review a PR without paying for
separate cloud API billing**, using existing subscriptions (Claude Max,
ChatGPT Plus) instead.

## Key Finding

No existing tool handles the full issue-to-PR-to-review-to-merge loop
using subscription-authenticated local agents. The pieces exist but are
fragmented. This validates Forge's reason to exist.

## Tools Surveyed

### Orchestration Frameworks

**[Overstory](https://github.com/jayminwest/overstory)**
- Review happens entirely locally, pre-PR, via SQLite mail + tmux
- Hierarchical agent tree: coordinator -> lead -> builder/reviewer/merger
- Builder finishes -> sends `worker_done` via SQLite mail -> lead spawns
  read-only reviewer in separate tmux session
- Reviewer has mechanically enforced tool guards (Claude Code hooks block
  Write, Edit, destructive Bash) -- not just prompt instructions
- 3-round revision cap before escalation to coordinator
- 4-tier merge conflict resolution (clean merge -> auto-resolve ->
  AI-resolve -> reimagine from spec)
- 11 runtime adapters (Claude Code, Pi, Gemini CLI, Aider, Goose, etc.)
- No GitHub integration for agent coordination
- Uses API keys, not subscriptions

**[Agent Orchestrator](https://github.com/ComposioHQ/agent-orchestrator)** (Composio)
- No reviewer agent. Routes external review feedback (humans, bots like
  [Cursor Bugbot](https://www.cursor.com/bugbot)) back to implementing
  agent via `tmux send-keys`
- 17-state formal session lifecycle with typed state transitions
- Lifecycle manager polls every 30 seconds, detects state transitions,
  fires configured YAML reactions
- Reaction engine is configurable per event type:
  ```yaml
  reactions:
    ci-failed:
      auto: true
      action: send-to-agent
      retries: 2
    changes-requested:
      auto: true
      action: send-to-agent
      escalateAfter: 30m
  ```
- Batch GraphQL queries for PR enrichment (N sessions in 1 request)
- Plugin architecture: 8 swappable slots (runtime, agent, workspace,
  tracker, SCM, notifier, terminal, lifecycle)
- Uses API keys, not subscriptions

**[CrewAI](https://github.com/crewAIInc/crewAI)**
- No built-in PR review. A
  [demo template](https://github.com/crewAIInc/demo-pull-request-review)
  exists but has placeholder logic
- Cannot wrap CLI agents (Claude Code, Codex CLI) -- API-only models
- Hierarchical delegation has documented
  [ping-pong infinite loop bugs](https://azguards.com/technical/the-delegation-ping-pong-breaking-infinite-handoff-loops-in-crewai-hierarchical-topologies/)
- Flow system uses decorator-based DAG (@start, @listen, @router) with
  and_/or_ composition operators
- Structured state via Pydantic models with persistence
- Enterprise adds webhook `/kickoff` endpoint but not PR review actions
- Uses API keys; supports local models via Ollama/LiteLLM

**[AutoGen](https://github.com/microsoft/autogen)** (Microsoft)
- No built-in PR review in Python. A .NET
  [dev-team sample](https://github.com/microsoft/autogen/tree/main/dotnet/samples/dev-team)
  exists using GitHub App webhooks
- Cannot wrap CLI agents
- Strong multi-agent primitives (Swarm handoffs, AgentTool delegation,
  GraphFlow, MagenticOne)
- Now in maintenance mode; new features going to
  [Microsoft Agent Framework](https://github.com/microsoft/agent-framework)
- Supports local models via Ollama and llama.cpp as first-class citizens
- Uses API keys

**[Composio PR Review Cookbook](https://docs.composio.dev/cookbooks/pr-review-agent)**
- GitHub Actions-triggered PR review using OpenAI API + Composio toolkit
- Three-step review structure: gather context -> analyze diff -> post
  feedback (useful pattern regardless of where the agent runs)
- Runs on GitHub-hosted runners, not locally
- Requires per-token API billing

### Local Agent Dispatch Tools

**[ide-agent-kit](https://github.com/ThinkOffApp/ide-agent-kit)** (37 stars)
- Receives GitHub webhooks, normalizes events into a JSONL queue, nudges
  tmux agent sessions via `tmux send-keys`
- Zero API billing by design -- "message delivery layer, not an
  autoresponder"
- Agent uses its existing subscription
- Supports multiple event sources (GitHub, Discord, GroupMind, ACP)
- AGPL-3.0 licensed

**[claude-hub](https://github.com/claude-did-this/claude-hub)** (427 stars)
- Full webhook-to-agent pipeline with Docker container isolation
- Supports subscription auth via captured `.credentials.json` from
  interactive `claude` login (grey area -- see Auth section below)
- On @-mention in PR, spawns Docker container running Claude Code CLI
- Most mature project in this space

**[night-watch-cli](https://github.com/jonit-dev/night-watch-cli)** (32 stars)
- 8 job types: executor, merger, reviewer, PR resolver, slicer, QA,
  audit, analytics
- Cron-based polling via `gh pr list`, dispatches local Claude/Codex CLI
  in isolated worktrees
- Review scoring system: AI posts `**Overall Score:** XX/100` comments,
  bash parses scores back out for retry decisions
- Iterative fix loop: up to 3 attempts to reach `minReviewScore` (80)
- Parallel review workers (one per PR, staggered 60s)
- Session checkpointing on timeout + context-exhaustion recovery
- Full kanban board (web UI + TUI + CLI) with GitHub Projects V2 or
  local SQLite backend
- Auto-merge with cascade rebase of remaining PRs
- Uses subscription auth when invoking `claude` CLI locally

**[better-symphony](https://github.com/SabatinoMasala/better-symphony)** (12 stars)
- Polls GitHub PRs via `gh` CLI with Markdown workflow templates
  (YAML frontmatter + Liquid prompt templates)
- Has a specific `pr-review.md` workflow for reviewing PRs
- 6 workflow types: PRD, dev, PR review, Ralph Loop (sequential subtask
  execution), smoke test, GitHub Issues
- Hot-reloading of workflow configs
- Uses subscription auth when invoking `claude` CLI locally

**[Merlin](https://github.com/Arunachalamkalimuthu/merlin-ai-code-review)**
- Supports GitHub, GitLab, Bitbucket, Azure DevOps, and Gitea
- Lists Claude Code CLI (subscription-based) as a supported provider
  alongside Ollama (fully local)
- RAG pipeline, 20+ slash commands, security scanning, reflect-and-review
- Most platform-comprehensive tool found

### The OpenClaw Cautionary Tale

[OpenClaw](https://github.com/openclaw/openclaw) (342k stars, formerly
Clawdbot) had the most complete architecture: webhooks, heartbeat polling,
CLI backends for Claude Code/Codex, multi-agent routing, skills marketplace.

Anthropic
[blocked subscription OAuth](https://midens.com/articles/2026-02-anthropic-oauth-ban-ai-tool-alternatives/)
in January 2026 for any tool except Claude.ai and Claude Code CLI. Google
followed suit for AI Ultra subscribers. The February 2026 ToS update
formally bans subscription OAuth in third-party tools.

The economics drove the bans: a $200/month Max subscription becomes deeply
unprofitable when an always-on agent burns millions of tokens daily.

[Nine CVEs in four days](https://openclawai.io/blog/openclaw-cve-flood-nine-vulnerabilities-four-days-march-2026)
(March 2026). 150+ total security advisories.
[820+ malicious skills](https://www.digitalocean.com/resources/articles/openclaw-security-challenges)
in the marketplace.

**Lesson for Forge:** Don't build around subscription OAuth extraction.
Use officially sanctioned mechanisms (`claude setup-token`, direct CLI
invocation) or local models.

## Subscription Auth Mechanisms

### Officially Supported

| Mechanism | Provider | How It Works | Risk |
|---|---|---|---|
| Direct CLI invocation | Claude Code, Codex CLI | Run `claude` or `codex` on your machine | None |
| `claude setup-token` | Claude Code | Generates OAuth token for CI/CD, tied to Max plan ([docs](https://github.com/anthropics/claude-code-action/blob/main/docs/setup.md)) | Low (tokens expire daily) |
| Codex ChatGPT auth | Codex CLI | Authenticate via subscription ([docs](https://developers.openai.com/codex/auth)) | Low for local; unsupported for CI/CD |
| Local models (Ollama) | Any | Run inference on your own hardware | None |

### Grey Area

| Mechanism | Risk |
|---|---|
| Capturing `.credentials.json` into Docker containers (claude-hub) | Credentials reused in automated context |

### Banned

| Mechanism | Status |
|---|---|
| Subscription OAuth in third-party tools (OpenClaw) | Blocked by Anthropic Jan 2026, Google Feb 2026 |

**Key discovery:**
[claude-setup-token-proxy](https://github.com/hugoharman/claude-setup-token-proxy)
reverse-engineered direct API calls using setup-token. Shows how to
authenticate with subscription credentials programmatically without the
full CLI.

## Platform Comparison (GitHub vs Alternatives)

The problem is platform-agnostic. Switching to GitLab, Gitea, or Forgejo
doesn't solve the core issue -- the bottleneck is the LLM provider's auth
model, not the git platform.

Self-hosted platforms (Gitea, Forgejo) have the strongest story for fully
local, zero-cost review using local models:
- [AiReviewPR](https://github.com/kekxv/AiReviewPR) (Gitea + Ollama)
- [AuditLM](https://github.com/ellenhp/auditlm) (Forgejo + llama.cpp)

Forge is right to be platform-agnostic at the adapter level (GitHub-first
with clean boundaries for future adapters).

## Gas Town (steveyegge/gastown) -- Closest Direct Peer

[Gas Town](https://github.com/steveyegge/gastown) (13k stars) is the
most directly comparable project to Forge. Uses git worktrees, local
agent dispatch, issue tracking as workflow substrate, and human gates.
See [pipeline-architecture.md](pipeline-architecture.md) for the full
deep dive.

Key differences from Forge's approach:
- 200k+ lines of Go vs Forge's planned thin Python CLI
- Dolt (version-controlled SQL) vs Forge's SQLite + files
- Custom issue tracker (Beads, 20k stars) vs GitHub Issues
- Elaborate metaphor system vs conventional terminology
- Heavy install (Go + Dolt + beads; tmux recommended) vs lighter install path

Worth trying before implementation to understand the UX tradeoffs.

## Implications for Forge

See [DEC-001](../decisions/DEC-001-review-stage-phasing.md) for the
phased review decision this research informed.

The dispatch mechanism (polling vs webhook) and auth strategy
(setup-token vs direct CLI invocation) are material design decisions.
For MVP (Phase 1), direct CLI invocation on the local machine is the
simplest and most ToS-compliant path.
