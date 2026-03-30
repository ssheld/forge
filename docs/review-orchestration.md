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

## Landscape Research (March 2026)

We surveyed the existing tools to understand what's been built and where
the gaps are.

### Tools That Partially Solve This

**[ide-agent-kit](https://github.com/ThinkOffApp/ide-agent-kit)** (37 stars)
- Receives GitHub webhooks, normalizes events into a JSONL queue, nudges
  tmux agent sessions via `tmux send-keys`
- Zero API billing by design: the agent uses its own subscription
- Deliberately thin -- "message delivery layer, not an autoresponder"
- **Takeaway for Forge:** The webhook -> queue -> agent nudge dispatch
  pattern. Forge's event layer could work similarly.

**[claude-hub](https://github.com/claude-did-this/claude-hub)** (427 stars)
- Full webhook-to-agent pipeline with Docker container isolation
- Supports subscription auth by capturing `.credentials.json` from a
  `claude` login session and mounting it into containers
- Most mature project in this space
- **Takeaway for Forge:** Docker isolation per review task. Auth approach
  is a grey area (credentials extracted and reused in automated containers,
  not an officially sanctioned pattern like `setup-token`).

**[night-watch-cli](https://github.com/jonit-dev/night-watch-cli)** (32 stars)
- Cron-based polling via `gh pr list`, dispatches local Claude/Codex CLI
  in isolated worktrees
- Parallel review workers across multiple PRs
- `night-watch review` + `night-watch-pr-reviewer-cron.sh` is a working
  implementation of "find PRs needing attention, dispatch a reviewer"
- **Takeaway for Forge:** The polling + worktree isolation + parallel
  dispatch pattern, and the specific detection logic (failed CI, low review
  score, merge conflicts).

**[better-symphony](https://github.com/SabatinoMasala/better-symphony)** (12 stars)
- Polls GitHub PRs via `gh` CLI with Markdown workflow templates
  (YAML frontmatter + Liquid prompt templates)
- Has a specific `pr-review.md` workflow for reviewing PRs
- Spawns local Claude CLI using subscription
- **Takeaway for Forge:** Workflow-as-markdown-config is a clean pattern
  for defining stage-specific behavior.

### Orchestration Frameworks (Don't Solve the Dispatch Problem)

**[Overstory](https://github.com/jayminwest/overstory)** (multi-agent orchestrator)
- Review happens entirely locally, *before* a PR is created
- Builder finishes -> sends `worker_done` via SQLite mail -> lead agent
  spawns a read-only reviewer in a separate tmux session
- Reviewer has mechanically enforced tool guards (not just prompt
  instructions -- actual tool blocks via Claude Code hooks)
- No GitHub integration for agent coordination
- **Takeaway for Forge:** The local coordination protocol (SQLite mail
  with typed messages), tool-enforced read-only reviewer, and the pre-PR
  review concept.

**[Agent Orchestrator](https://github.com/ComposioHQ/agent-orchestrator)** (Composio)
- No reviewer agent. Routes external review feedback (from humans and bots
  like [Cursor Bugbot](https://www.cursor.com/bugbot)) back to the
  implementing agent via `tmux send-keys`
- Lifecycle manager polls every 30 seconds, detects state transitions,
  fires configured YAML reactions
- **Takeaway for Forge:** The declarative reaction engine for state
  transitions. Forge's policy layer could use a similar pattern:
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

**[CrewAI](https://github.com/crewAIInc/crewAI)**
- No built-in PR review. A
  [demo template](https://github.com/crewAIInc/demo-pull-request-review)
  exists but has placeholder logic. Enterprise adds webhook triggers but
  not PR review actions.
- Cannot wrap CLI agents (Claude Code, Codex CLI) -- API-only models.
- **Takeaway for Forge:** Limited. CrewAI's model (API-based LLM calls)
  is fundamentally different from Forge's model (local CLI agent dispatch).

**[AutoGen](https://github.com/microsoft/autogen)** (Microsoft)
- No built-in PR review in Python. A .NET
  [dev-team sample](https://github.com/microsoft/autogen/tree/main/dotnet/samples/dev-team)
  demonstrates the pattern with GitHub App webhooks.
- Cannot wrap CLI agents. Now in maintenance mode; new features going to
  [Microsoft Agent Framework](https://github.com/microsoft/agent-framework).
- **Takeaway for Forge:** The Swarm handoff and GraphFlow patterns are
  well-designed multi-agent primitives, but the framework doesn't fit
  Forge's local-CLI-first model.

**[Composio PR Review Cookbook](https://docs.composio.dev/cookbooks/pr-review-agent)**
- GitHub Actions-triggered PR review using OpenAI API + Composio toolkit
- Runs on GitHub-hosted runners, not locally
- Requires per-token API billing
- **Takeaway for Forge:** The three-step review structure
  (gather context -> analyze diff -> post feedback) is a good pattern
  regardless of where the agent runs.

### The OpenClaw Cautionary Tale

[OpenClaw](https://github.com/openclaw/openclaw) (342k stars, formerly
Clawdbot) has the most complete architecture: webhooks, heartbeat polling,
CLI backends for Claude Code/Codex, multi-agent routing, skills marketplace.

However, Anthropic
[blocked subscription OAuth](https://midens.com/articles/2026-02-anthropic-oauth-ban-ai-tool-alternatives/)
in January 2026 for any tool except Claude.ai and Claude Code CLI. Google
followed suit for AI Ultra subscribers. The February 2026 ToS update
formally bans subscription OAuth in third-party tools.

The economics drove the bans: a $200/month Max subscription becomes deeply
unprofitable when an always-on agent burns millions of tokens per day.

**Lesson for Forge:** Don't build around subscription OAuth extraction.
Use officially sanctioned mechanisms (`claude setup-token`, direct CLI
invocation) or local models.

## Subscription Auth Mechanisms

How to run agents without per-token API billing:

### Officially Supported

| Mechanism | Provider | How It Works | Risk |
|---|---|---|---|
| Direct CLI invocation | Claude Code, Codex CLI | Run `claude` or `codex` on your machine -- uses your subscription login | None (indistinguishable from human use) |
| `claude setup-token` | Claude Code | Generates OAuth token (`sk-ant-oat01-*`) for CI/CD, tied to Max plan | Low (officially designed for automation, tokens expire daily) |
| Codex ChatGPT auth | Codex CLI | `codex logout` then re-auth via subscription | Low for local use; unsupported for CI/CD |
| Local models (Ollama, etc.) | Any | Run inference on your own hardware | None (fully self-hosted) |

### Grey Area

| Mechanism | Risk |
|---|---|
| Capturing `.credentials.json` into Docker containers | Credentials obtained legitimately but reused in automated context outside the CLI |

### Banned

| Mechanism | Status |
|---|---|
| Subscription OAuth in third-party tools (OpenClaw pattern) | Blocked by Anthropic Jan 2026, Google Feb 2026 |

## Design Decision: Phased Review

See [DEC-001](decisions/DEC-001-review-stage-phasing.md) for the formal
decision record.

### Phase 1: Pre-PR Review (Local)

```
Agent implements in worktree
  -> Forge runs local review (same machine)
  -> Review passes: Forge creates PR
  -> Review fails: Forge routes feedback to implementing agent
```

- No event infrastructure needed (no webhooks, no polling)
- Forge controls the full loop -- the implementing agent signals done,
  Forge dispatches the reviewer locally
- Easy to swap reviewer models (point at a different provider)
- Inspired by [Overstory's](https://github.com/jayminwest/overstory)
  pre-PR review model with SQLite mail coordination
- PRs arrive on GitHub already reviewed, higher quality

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
