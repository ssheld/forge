# Pipeline Architecture

Research into how CI/CD systems and AI agent orchestrators structure work
into stages, jobs, and pipelines. Conducted March 2026 to inform Forge's
job architecture.

## Key Insight: Cycles, Not DAGs

Traditional CI/CD pipelines are directed acyclic graphs -- each stage runs
once. AI agent workflows are fundamentally **cyclic**. The implement ->
review -> fix -> re-review loop is the core value proposition, not a
failure mode.

[LangGraph](https://docs.langchain.com/oss/python/langgraph/workflows-agents)
explicitly embraces this: it models agent workflows as cyclic graphs.
As their documentation states,
["Agents aren't pipelines. Agents need cycles."](https://medium.com/@creativeaininja/langgraph-why-the-future-of-ai-agents-looks-like-a-state-machine-not-a-chatbot-c4562fa148cb)

This is the single most important architectural distinction between
traditional CI/CD and AI agent orchestration.

## The Pattern Every Working Tool Converges On

```
Plan/Slice → Implement → Review ⟲ Fix → Merge
                           ↑__________|
                          (bounded cycle)
```

Every tool that handles the full loop has some version of this cycle with
termination conditions (score threshold, iteration cap, timeout, or human
escalation).

## Night-Watch CLI: The Most Complete Job Architecture

[Night-watch-cli](https://github.com/jonit-dev/night-watch-cli) has the
most fully realized job-based pipeline. Deep source code analysis revealed
patterns worth studying closely.

### Job Registry (8 types, priority-ordered)

| Priority | Job | Default Schedule | Purpose |
|----------|-----|-----------------|---------|
| 50 | Executor | Every 2h | Implements work items in isolated worktrees |
| 45 | Merger | Every 4h (disabled) | Merges PRs that pass all gates |
| 40 | Reviewer | Every 3h | Scores PRs, iteratively fixes until threshold |
| 35 | PR Resolver | 3x/day | Resolves merge conflicts via AI rebase |
| 30 | Slicer | Every 6h | Generates PRDs from roadmap items |
| 20 | QA | 3x/day | Runs Playwright e2e tests against PRs |
| 10 | Audit | Weekly | Scans codebase for quality issues, creates board issues |
| 10 | Analytics | Weekly (disabled) | Processes Amplitude data into actionable issues |

### Review Scoring System

The scoring mechanism is elegantly simple:

1. The reviewer prompt template instructs the AI to post comments with
   `**Overall Score:** XX/100`
2. The bash script parses scores back out with regex:
   `grep -oP '(?:Overall\s+)?Score:\*?\*?\s*\d+/100'`
3. Takes the most recent (last) match from all PR comments
4. Compares against `minReviewScore` (default 80, configurable 0-100)

**The scores are AI-generated, not computed by the system.** The rubric
lives in the prompt template, not in code. Changing what "80/100" means
is a prompt change, not a code change.

### Iterative Fix Loop

The reviewer's retry loop per PR:

1. Check PR status: merge conflicts? CI failures? Score below threshold?
2. If needs work: construct prompt with:
   - Preflight data (merge state, failing checks, current score)
   - Full latest review comment body (truncated to 6000 chars) as
     `## Latest Review Feedback`
   - PRD context from linked issues (truncated to 4500 chars)
3. Dispatch AI provider with timeout budget
4. After execution: re-check score
   - Score >= threshold → done, mark ready for review
   - Score < threshold + attempts remain → sleep 30s, retry
   - Score < threshold + no attempts → label `needs-human-review`
5. Total attempts = `reviewerMaxRetries + 1` (default: 3 total)

**Parallel workers:** When multiple PRs need work, fans out one worker
process per PR, staggered 60s apart, each with its own lock file.

### Session Checkpointing

On timeout (exit code 124):
1. Stages all changes, commits as `"chore: checkpoint timed-out progress"`
2. Moves issue back to Ready column
3. Next run picks up the same branch in the worktree

**Context-exhaustion recovery:** When the AI hits the context window limit,
night-watch checkpoints progress and restarts with a fresh context using a
"continue" prompt that instructs the AI to review existing work and
continue from where it stopped.

### Merger Cascade Rebase

After each successful merge, the merger rebases ALL remaining open PRs
against the new base branch. This prevents cascading merge conflicts when
running parallel issue lanes. FIFO merge order (oldest PR first).

### Auto-Merge Gates

All must pass (conjunctive):
1. CI: all status checks completed and passing
2. Review score: >= `minReviewScore` (parsed from PR comments)
3. Not draft
4. Optional: rebase against base branch, wait up to 5 min for CI to pass

### End-to-End Pipeline

```
ROADMAP.md
    |
    v
[Slicer] (every 6h)
    | Reads roadmap, generates PRD, creates board issue in "Ready"
    v
Board Issue (Ready)
    |
    v
[Executor] (every 2h)
    | Picks next Ready issue (priority-sorted)
    | Creates worktree + branch, dispatches AI with PRD
    | AI implements, opens PR
    | Moves issue to Done
    v
Open PR
    |
    +---> [Reviewer] (every 3h)
    |       Checks: conflicts, CI, score
    |       Fix loop: up to 3 attempts to reach score >= 80
    |       Labels ready-for-review or needs-human-review
    |
    +---> [QA] (3x/day)
    |       Writes + runs Playwright e2e tests
    |       Labels e2e-validated on pass
    |
    +---> [PR Resolver] (3x/day)
    |       Resolves merge conflicts via AI rebase
    |       Labels ready-to-merge
    |
    +---> [Merger] (every 4h, disabled by default)
            All gates pass → merge (squash)
            Cascade rebase remaining PRs
```

### Provider Configuration

Per-job provider assignment + time-based schedule overrides:
```json
{
  "jobProviders": {
    "executor": "glm-5",
    "reviewer": "claude",
    "qa": "claude-sonnet-4-6"
  },
  "providerScheduleOverrides": [{
    "label": "Night Surge",
    "presetId": "glm-5",
    "days": [0,1,2,3,4,5,6],
    "startTime": "23:00",
    "endTime": "04:00"
  }]
}
```

## Cross-Orchestrator Phase Patterns

### How Each Tool Structures Work

**Overstory** -- Hierarchical role-based, 3-phase lead workflow:
1. **Scout:** Delegate exploration to scouts (parallel). Collect findings.
2. **Build:** Write spec files from scout findings. Spawn builders with
   non-overlapping file scopes (parallel).
3. **Review & Verify:** Monitor builders via mail. On `worker_done`,
   self-verify (simple) or spawn reviewer (complex). Handle PASS/FAIL.
   3-cycle revision cap.

Complexity-based pipeline selection: simple tasks skip scouts, moderate
tasks use single builder with lead self-verification, complex tasks run
the full Scout -> Build -> Review pipeline.

**Agent Orchestrator** -- Formal state machine with reaction engine:
```
spawning -> working -> pr_open -> [ci_failed | review_pending]
  -> changes_requested -> working (fix) | approved -> mergeable
  -> merged -> cleanup -> done
```
17 explicit states. Reactions are configurable YAML per event type.
Differentiated retry strategies: CI failure vs review rejection vs
timeout each get different handling.

**better-symphony** -- Markdown workflow templates:
- Each workflow is a `.md` file with YAML frontmatter defining tracker,
  workspace, hooks, and agent config
- 6 workflow types: PRD, dev, PR review, Ralph Loop, smoke test, GitHub
  Issues
- Ralph Loop: sequential subtask execution with fresh Claude session per
  subtask
- Revision mode: when PR comments exist, dev workflow template injects
  "you are iterating on existing work" instructions
- Hot-reloading via chokidar file watcher

**CrewAI** -- Decorator-based DAG with Flow system:
- `@start()` marks entry points (parallel by default)
- `@listen("method")` chains stages (fires on completion)
- `@router()` enables conditional branching
- `and_()` / `or_()` for synchronization barriers
- No built-in review-fix loop -- must be implemented manually
- `@human_feedback` decorator for human-in-the-loop gates

### Common Patterns Across Tools

1. **All use git worktrees (or clones) for isolation.** Universal pattern.
2. **All that have pipelines include implement -> review/verify.**
3. **Issue tracker as state machine driver** (Agent Orchestrator,
   better-symphony, night-watch). Tracker state is the source of truth.
4. **Label-based dispatch** for routing issues to workflows.

### State Machine Formality

| Tool | Approach |
|------|----------|
| Agent Orchestrator | Most formal -- 17 typed states, terminal state guards |
| Overstory | Semi-formal -- typed mail messages enforce transitions |
| better-symphony | Semi-formal -- RunAttempt status enum + tracker reconciliation |
| CrewAI | DAG-based -- decorators build compile-time graph |
| Night-watch | Job-based -- each job has its own state, GitHub is source of truth |

## The Review-Fix Loop: Best Practices

The research points to a **layered termination strategy**:

1. **Quality threshold** (primary): Define "good enough" numerically.
   Night-watch uses 80/100. LangGraph uses 0.8 on 0-1 scale. Forge
   could use review severity (no Critical findings = pass).
2. **Fixed iteration cap** (safety valve): 2-3 revision rounds before
   escalation. Overstory defaults to 3. Night-watch defaults to 3.
3. **Timeout** (secondary safety): Composio's `escalateAfter: 30m`.
   Important for async dispatch (Phase 2), less critical for local
   (Phase 1).
4. **Human escalation** (terminal): When cap or timeout is hit, escalate
   with the agent's best attempt and accumulated feedback. Never
   silently give up.
5. **Differentiated handling by failure type**: CI failure vs review
   rejection vs stall deserve different retry strategies.

## Quality Gates and Scoring

### How Traditional CI/CD Composes Gates

Quality gates compose through conjunction (AND): all must pass.
- Build/compilation -- binary pass/fail
- Test coverage -- percentage threshold
  ([JaCoCo](https://www.jacoco.org/),
  [Codecov](https://about.codecov.io/))
- Static analysis -- composite score
  ([SonarQube quality gates](https://www.sonarsource.com/resources/library/integrating-quality-gates-ci-cd-pipeline/))
- Security scanning -- severity threshold (no Critical vulnerabilities)
- Peer review -- N-of-M approvals
- Manual approval -- explicit human action

### How AI Agent Systems Score

Evolving toward multi-dimensional scoring:
- **LLM-as-Judge:** Secondary model evaluates primary agent's output
  against rubrics. Becoming the
  [standard pattern for 2026](https://www.qodo.ai/blog/5-ai-code-review-pattern-predictions-in-2026/)
- **Confidence indicators:** Not just "is this a problem?" but "how sure
  am I?" Forge's existing model (High/Medium/Low) aligns with industry.
- **Severity-based thresholds:** More actionable than raw numeric scores.

### Recommended Composition for Forge

```
merge_ready = (
    no Critical findings with High confidence
    AND all verification commands passed
    AND human approval granted
)
```

Maps to both CI/CD quality gates (conjunctive pass/fail) and AI review
scoring (severity-based thresholds).

## Recommended Forge Job Architecture

Based on all research, Forge should use **independently schedulable jobs**
that compose into a pipeline:

| Job | What it does | Trigger |
|---|---|---|
| **Planner** | Reads roadmap/issues, generates implementation briefs | Manual or periodic |
| **Executor** | Picks ready work item, dispatches agent in worktree | Manual or periodic |
| **Reviewer** | Reviews implementation, scores, runs fix loop, produces ReviewReport | Executor completion or PR detection |
| **Merger** | Checks gates, merges, rebases remaining PRs | Manual or periodic |

Note: earlier research considered a separate **Fixer** job, but the v1
decomposition folds the fix loop into the Reviewer. Review and fix are
tightly coupled in every working implementation studied (night-watch
handles fix retries within the reviewer, Overstory handles fix cycles
within the lead's Phase 3). The fix logic can be extracted into a
separate job later if needed.

### Why Jobs, Not a Rigid Pipeline

Jobs are independently schedulable units. This is more flexible than a
rigid pipeline because:
- Run the reviewer without the executor (review an existing PR)
- Run the executor without the reviewer (implement, skip auto-review)
- Run the merger independently (batch-merge approved PRs)
- Each job has its own retry logic, timeouts, and provider config
- Jobs still compose into the full pipeline when chained

### Two-Layer Architecture

Validated by Tekton's Task/Pipeline model,
[OpenAI Symphony's](https://github.com/openai/symphony) WORKFLOW.md
approach, and Composio's plugin architecture:

1. **Fixed infrastructure** managed by Forge: workspace setup, worktree
   creation, provider dispatch, state tracking, cleanup, stage transitions
2. **User-configurable behavior** in `.forge.yaml`: which stages to run,
   verification commands, review model, retry limits, quality thresholds,
   provider selection per job

### What to Borrow From Each Tool

| Tool | Pattern | Detail |
|---|---|---|
| Night-watch | Review scoring + iterative fix | AI-generated scores parsed by system; inject review feedback into fix prompt; configurable threshold and retry count |
| Night-watch | Session checkpointing | On timeout: commit checkpoint, resume from same branch next run |
| Night-watch | Context-exhaustion recovery | Detect context limit, checkpoint, restart with "continue" prompt |
| Night-watch | Merger cascade rebase | After each merge, rebase all remaining PRs |
| Night-watch | Per-job provider config | Different models for different jobs + time-based overrides |
| Overstory | Read-only reviewer | Tool guards mechanically enforced via Claude Code hooks, not just prompts |
| Overstory | Typed coordination messages | 8 protocol types for structured agent communication |
| Overstory | Complexity-based pipeline depth | Skip phases for simple tasks |
| Agent Orchestrator | Declarative reaction engine | YAML-configured responses to state transitions with retries and escalation |
| Agent Orchestrator | Batch PR enrichment | Single GraphQL query for all active sessions |
| better-symphony | Workflow-as-markdown templates | Prompt construction from YAML frontmatter + Liquid templates |
| better-symphony | Hot-reload config | File watcher for workflow changes without restart |
| CrewAI | Conditional routing | @router pattern for branching stage transitions |

### Recommended Forge Vocabulary

| Term | Meaning | Analogues |
|------|---------|-----------|
| **WorkItem** | A GitHub issue being processed | Night-watch: work item |
| **Run** | A single end-to-end attempt to process a WorkItem | Tekton: PipelineRun |
| **Stage** | A bounded phase (intake, brief, implement, review, fix, merge-ready) | CI/CD: stage |
| **StageResult** | The structured output of a stage | Tekton: Task result |
| **Provider** | The agent/tool executing a stage | Overstory: runtime adapter |

Already established in `docs/mvp-design.md` and validated by the research.

## Additional Tool Analysis (March 2026)

### codex-plugin-cc (OpenAI Official)

[codex-plugin-cc](https://github.com/openai/codex-plugin-cc) (4.9k stars)
is OpenAI's official plugin that lets Claude Code delegate work to Codex
CLI. Released March 2026. One-way bridge -- Claude Code dispatches
tasks/reviews to Codex running on the same machine. Codex doesn't know
it's being orchestrated.

**Patterns relevant to Forge:**

- **Broker/multiplexer:** A TCP/Unix socket multiplexer shares a single
  Codex runtime across multiple concurrent plugin invocations. Serializes
  requests, tracks stream ownership, passes interrupts through. Directly
  applicable to Forge managing parallel agent sessions.
- **Stop gate (quality gate before session end):** The `Stop` hook runs
  a Codex review of Claude's work before allowing the session to end. If
  the review finds issues, it blocks with
  `{"decision": "block", "reason": "..."}`. This is exactly the pre-PR
  review gate pattern Forge designed for Phase 1.
- **"Thin forwarder" pattern:** When delegating to an external agent, the
  forwarding layer is deliberately dumb -- no interpretation, no
  summarization, just pass-through. Prevents the orchestrator from
  corrupting agent results.
- **Prompt-as-configuration:** Commands, agents, and skills are all
  markdown files. Behavior changes by editing text, not code. Similar to
  better-symphony's workflow-as-markdown pattern.
- **Zero dependencies:** Entire runtime uses only Node.js built-ins
  (`node:child_process`, `node:net`, `node:fs`). Strong signal about
  what's achievable without frameworks.
- **Prompt contract tests:** Tests verify that prompt markdown files
  contain specific phrases and instructions. Novel approach to testing
  prompt engineering as contracts.

**Limitations:** No DAG/pipeline or fanout-merge orchestration, no GitHub
integration at the plugin layer, no worktree management. Supports
background jobs and concurrent broker invocations, but lacks structured
multi-step workflow coordination.

### AgentFlow (shouc/agentflow)

[AgentFlow](https://github.com/shouc/agentflow) (~253 stars, ~715 commits)
is a Python DAG orchestrator for local CLI coding agents. Think Apache
Airflow but for Codex/Claude/Kimi. Substantial codebase with a clean
layered architecture: DSL -> Specs (Pydantic) -> Orchestrator (async) ->
Agents (adapters) -> Runners (execution targets).

**Patterns relevant to Forge:**

- **`on_failure` back-edges for iterative cycles:** `review.on_failure >>
  write` creates a bounded cycle with `max_iterations`. When review fails,
  upstream nodes reset to PENDING and re-execute. The cleanest expression
  of the review-fix loop found across all tools studied.
- **Declarative success criteria:** Post-conditions evaluated after
  execution, independent of exit code: `output_contains: "LGTM"`,
  `file_exists`, `file_contains`, `file_nonempty`. A node can exit 0 but
  still "fail" if output doesn't meet criteria. More robust than exit
  codes for non-deterministic agents.
- **Fanout/merge as first-class primitives:** `fanout(node, 128)` expands
  one template into N parallel copies. `merge(node, source, by=["field"])`
  aggregates results. Handles parallel review across files without manual
  loop construction.
- **Pipeline-as-JSON intermediate:** Python DSL generates JSON. Orchestrator
  only consumes JSON. Pipeline is fully inspectable and validatable before
  execution (`agentflow validate`, `agentflow inspect`). Clean separation
  of definition from execution.
- **Scratchboard:** A shared markdown file all agents can read/append to
  with line-level deduplication. Simple weak coordination without a
  message bus. Useful for lightweight inter-agent context sharing.
- **Git worktree per node:** Each parallel agent gets an isolated worktree.
  Diffs captured per-node as artifacts. Critical for parallel coding
  agents editing files concurrently.
- **PreparedExecution / Runner separation:** Clean split between "what to
  run" (agent adapter: Codex, Claude, Kimi) and "where to run" (runner:
  local, SSH, EC2, ECS, Docker). Add new agents or execution targets
  independently. Maps to Forge's Provider + future Node concepts.
- **Normalized trace events:** Each agent's output stream is parsed into
  a common `NormalizedTraceEvent` model. Makes monitoring agent-agnostic.
- **File-based cancellation:** `cancel.requested` sentinel file plus
  in-memory threading events. Survives process crashes, can be triggered
  externally (e.g., by a monitoring script touching the file).
- **Periodic controller nodes:** A monitor node runs on a timer, inspects
  fanout group status, and emits cancel/rerun commands for individual
  members. Runtime adaptive control within a running pipeline.

**Limitations:** No GitHub integration (PRs, issues, reviews). No
structured data flow between nodes (only Jinja2 string interpolation in
prompts). No diff merging or conflict resolution across parallel agents.
Very large monolithic files suggesting rapid development without
decomposition.

### Top 5 Ideas Across Both Repos

1. **`on_failure` back-edges with `max_iterations`** (AgentFlow) -- The
   cleanest review-fix loop expression. Forge's reviewer job should work
   this way.
2. **Stop gate / quality gate hooks** (codex-plugin-cc) -- Agent B reviews
   Agent A's work before allowing completion. Maps directly to Forge's
   Phase 1 pre-PR review.
3. **Declarative success criteria** (AgentFlow) -- Let users define "done"
   as post-conditions rather than just exit codes. More robust for
   non-deterministic agents.
4. **Broker pattern for shared runtime** (codex-plugin-cc) -- Multiplexing
   concurrent invocations through one agent process. Relevant when Forge
   manages parallel issue lanes.
5. **PreparedExecution / Runner separation** (AgentFlow) -- "What to run"
   vs "where to run" as independent axes. Forge already has Provider
   adapters; adding Runner adapters (local, SSH, Docker) gives hardware
   modularity cleanly.

## Gas Town (steveyegge/gastown) -- Closest Direct Peer

[Gas Town](https://github.com/steveyegge/gastown) (13k stars, Go, MIT)
is a 200k+ line CLI orchestrator by Steve Yegge. Uses the same core
primitives as Forge: git worktrees for isolation, issue tracking as
workflow substrate, human gates, multi-agent orchestration.

**Key architectural patterns:**

- **Bare repo + worktree:** Each project ("rig") has a bare repo
  (`.repo.git`) as a shared object store. All worktrees are cheap
  branches off this bare repo. More efficient than standard worktrees.
- **Three-layer agent model:** Identity (permanent bead in database) /
  Sandbox (persistent worktree, reused across sessions) / Session
  (ephemeral tmux session). Clean lifecycle separation.
- **Bors-style merge queue ("Refinery"):** Batches MRs, tests the merged
  stack, bisects on failure to isolate the culprit. Can fix inline or
  re-dispatch. The most complete AI merge queue found.
- **TOML workflow templates ("Formulas"):** 48 built-in templates with
  step dependencies, variable substitution, two modes (lightweight inline
  vs checkpointed sub-tasks that survive crashes). Key formula:
  `mol-polecat-work` (7 steps: load-context -> branch-setup -> implement
  -> commit -> self-review -> build-check -> submit).
- **Three-tier health monitoring:** Boot dog (5min heartbeat) -> Deacon
  (cross-project patrol) -> Witness (per-project agent monitor). Auto-
  recovery via nudge, handoff, or nuke.
- **GUPP propulsion principle:** "If it's on your hook, you MUST run it."
  Simple rule that drives autonomous operation.
- **Seance:** Agents query previous sessions for context recovery.
- **Mail + nudge system:** Persistent messages via Dolt database + real-
  time delivery via tmux keystroke injection.

**Companion: [Beads](https://github.com/steveyegge/beads)** (20k stars)
is a standalone git-backed issue tracker built on Dolt (version-controlled
SQL database). Full lifecycle, dependency tracking, formulas, compaction,
cross-project dependencies.

**Where Forge can differentiate:**
- **Install friction:** Gas Town requires Go 1.25+, Dolt 1.82+, beads,
  tmux, sqlite3. Forge could be `pip install forge-cli`.
- **Complexity:** The metaphor system (GUPP, MEOW, NDI, Polecats, Hooks,
  Convoys, Wasteland) is a steep learning curve. Forge uses conventional
  terminology.
- **Dependencies:** Dolt is a non-trivial operational dependency. Forge's
  SQLite + files approach is lighter.
- **tmux-only:** Gas Town is deeply tied to tmux. Forge could be session-
  manager-agnostic.
- **Config format:** JSON-only is verbose. YAML is friendlier.

## Additional Tools Surveyed (March 2026)

### Git-Native Peers

**[lalph](https://github.com/tim-smart/lalph)** (106 stars, TypeScript)
- Issue-driven orchestrator with GitHub Issues and Linear backends
- Key pattern: **prd.yml as agent-editable state** -- agents communicate
  state changes by editing a watched YAML file, filesystem watcher syncs
  bi-directionally with the issue source
- **AI-driven task selection:** A "chooser" agent picks the most important
  task, not FIFO
- **Stall timeout + auto-decomposition:** If an agent stalls, a separate
  agent breaks the task into smaller pieces
- Three git flow strategies: PR, direct commit (rebase+push), spec-driven

**[operator](https://github.com/untra/operator)** (10 stars, Rust)
- TUI for supervising parallel agent lanes via tmux/zellij
- **Terminal multiplexer as supervision layer:** Launch agents in panes,
  monitor via screen capture. Natural human observability.
- **Ticket-as-file:** Markdown with YAML frontmatter in `.tickets/queue/`
- **Same-project sequential, cross-project parallel:** Simple conflict-
  prevention rule

**[parallel-worktrees](https://github.com/SpillwaveSolutions/parallel-worktrees)** (6 stars)
- Claude Code skill (SKILL.md + 3 bash scripts)
- **Non-determinism exploitation:** Run N agents on same task, pick best
  result. Treats LLM randomness as an advantage.
- `.agent-status/*.json` for zero-dependency coordination
- Proves the concept works with zero dependencies

### Major Platforms

**[OpenHands](https://github.com/All-Hands-AI/OpenHands)** (70k stars)
- Formerly OpenDevin. AI software development platform.
- **EventStream as central bus:** Typed Action/Observation events with
  serialization, enabling replay and persistence
- **Issue Resolver pipeline:** Complete issue-to-PR pipeline with multi-
  platform support (GitHub, GitLab, Bitbucket, Azure DevOps)
- **Stuck detection:** Dedicated 21KB module that programmatically detects
  agent loops
- **Runtime abstraction:** Pluggable runtimes (Docker, local, remote,
  Modal). Does NOT use worktrees -- uses containers for isolation.

**[MetaGPT](https://github.com/geekan/MetaGPT)** (66k stars)
- Models a "software company" as a multi-agent system.
- **Publish-subscribe role routing:** Roles produce messages tagged with
  `cause_by`; other roles `_watch()` for specific action types. Creates
  implicit DAGs without hard-coded sequencing.
- **Three react modes:** BY_ORDER (sequential), REACT (LLM chooses),
  PLAN_AND_ACT (plan first, then execute)
- **LGTM/LBTM code review loop:** Structured review with clear pass/fail
  signaling and configurable iteration count (default 2 rounds)
- **Team serialization:** Save/restore multi-agent workflows for
  resumability. Does NOT use worktrees.

### Lightweight / Policy-Driven

**[Atomic Agents](https://github.com/BrainBlend-AI/atomic-agents)** (5.8k stars)
- Entire agent runtime in ~1100 lines. Built on Instructor + Pydantic.
- **Schema chaining:** Output Pydantic schema of one agent IS the input
  schema of the next. Catches integration bugs at definition time.
- **No orchestration layer by design:** "Orchestration is your code."
  Closest to Forge's "thin controller" principle.
- **Context providers:** Dynamic system prompt injection at runtime.

**[Orloj](https://github.com/OrlojHQ/orloj)** (62 stars, Go, v0.4.0)
- Kubernetes-style declarative runtime for multi-agent AI systems.
- **YAML manifests for everything:** Agents, tools, model endpoints,
  governance policies declared as `apiVersion/kind/metadata/spec`.
- **Model endpoint indirection:** Agents reference models by name, not
  provider details. One YAML change swaps all providers.
- **Governance as data:** AgentPolicy, ToolPermission, ToolApproval as
  declarative resources. Full RBAC.
- Over-built for Forge's current stage but the concepts are sound.

**[OpenAI Swarm](https://github.com/openai/swarm)** (21k stars, deprecated)
- **Handoff-as-return-value:** Agent routing by having functions return
  the next Agent. No routing table, no orchestrator. ~400 lines total.
- **Context variables as hidden state:** Shared dict readable by tools
  but invisible to the LLM.
- Now deprecated in favor of OpenAI Agents SDK.

### Research Papers

**[CAID](https://github.com/JiayiGeng/CAID)** (CMU, arXiv 2603.21489)
- Dependency-aware task delegation with parallel git worktree execution
- Central manager builds dependency graph, dispatches when prerequisites
  met. Built on OpenHands. 14-27% gains depending on benchmark.

**[AgentConductor](https://arxiv.org/abs/2602.17100)** (Feb 2026)
- RL-optimized dynamic communication topology generation. 68% token
  savings. Adaptive difficulty-based parallelism.

**[Agyn](https://arxiv.org/abs/2602.01465)** (Feb 2026)
- Manager/Researcher/Engineer/Reviewer team roles with isolated sandboxes
  and native GitHub PR/review flows. Custom `gh-pr-review` CLI extension.

### Anthropic 2026 Agentic Coding Trends Report

[18-page report](https://resources.anthropic.com/2026-agentic-coding-trends-report)
(January 2026) identifies 8 trends. Most relevant to Forge:
- Trend #2: Single agents evolve into coordinated teams
- Trend #3: Long-running agents build complete systems
- Trend #4: Human oversight scales through intelligent collaboration
Validates Forge's direction: local-first, multi-agent, human-gated.

### Protocol Landscape (March 2026)

- **MCP** (Model Context Protocol): 97M monthly SDK downloads. Donated to
  Linux Foundation. The standard for agent-to-tool connectivity.
- **A2A** (Agent-to-Agent): Google protocol, v0.3, Linux Foundation, 50+
  partners. Focused on enterprise agent interop, not coding worktree
  handoffs. Not needed for Forge v1.

## Tools Worth Trying

Before borrowing patterns, these tools are worth actually using to
understand how they feel in practice, not just how they look in source:

1. **Gas Town** -- Closest peer. Try the full flow: install, create a rig,
   sling work to a polecat, watch the review-merge cycle.
2. **Night-watch-cli** -- Most complete job system. Try: queue a PRD,
   watch executor run, see the reviewer score and iterate.
3. **lalph** -- Try the prd.yml bidirectional sync pattern. See how agent-
   editable state files feel vs explicit API calls.
4. **Overstory** -- Try spawning a reviewer with tool guards. See how
   read-only enforcement works in practice.
5. **AgentFlow** -- Try the `on_failure` back-edge pattern for a review-
   fix loop. See how declarative success criteria feel.

## Sources

### CI/CD Pipeline Patterns
- [GitHub Actions Workflow Syntax](https://docs.github.com/actions/using-workflows/workflow-syntax-for-github-actions)
- [GitLab CI DAG](https://about.gitlab.com/blog/directed-acyclic-graph/)
- [Jenkins Pipeline Syntax](https://www.jenkins.io/doc/book/pipeline/syntax/)
- [Buildkite Pipeline Design](https://buildkite.com/docs/pipelines/best-practices/pipeline-design-and-structure)
- [Tekton Tasks and Pipelines](https://tekton.dev/docs/pipelines/)
- [Dagger Programmable Pipelines](https://docs.dagger.io/features/programmable-pipelines/)
- [SonarQube Quality Gates in CI/CD](https://www.sonarsource.com/resources/library/integrating-quality-gates-ci-cd-pipeline/)

### AI Agent Orchestration
- [Azure AI Agent Design Patterns](https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/ai-agent-design-patterns)
- [Agent Orchestration with Feedback Loops](https://loopjar.ai/blog/agent-orchestration-feedback-loop)
- [LangGraph Architecture](https://medium.com/@shuv.sdr/langgraph-architecture-and-design-280c365aaf2c)
- [Google ADK Workflow Agents](https://google.github.io/adk-docs/agents/workflow-agents/)
- [AWS Strands Agents Orchestration](https://aws.amazon.com/blogs/machine-learning/customize-agent-workflows-with-advanced-orchestration-techniques-using-strands-agents/)
- [Self-Improving Coding Agents](https://addyosmani.com/blog/self-improving-agents/) (Addy Osmani)
- [Agent-Operated CI/CD](https://alexlavaee.me/blog/agent-operated-cicd-pipelines/)
- [Elastic Self-Correcting Monorepos](https://www.elastic.co/search-labs/blog/ci-pipelines-claude-ai-agent)
- [AI Code Review Predictions 2026](https://www.qodo.ai/blog/5-ai-code-review-pattern-predictions-in-2026/)

### Specific Tools
- [Night-Watch CLI](https://github.com/jonit-dev/night-watch-cli)
- [Overstory](https://github.com/jayminwest/overstory)
- [Agent Orchestrator](https://github.com/ComposioHQ/agent-orchestrator)
- [better-symphony](https://github.com/SabatinoMasala/better-symphony)
- [CrewAI](https://github.com/crewAIInc/crewAI)
- [OpenAI Symphony](https://github.com/openai/symphony)
- [ide-agent-kit](https://github.com/ThinkOffApp/ide-agent-kit)
- [claude-hub](https://github.com/claude-did-this/claude-hub)
- [Merlin](https://github.com/Arunachalamkalimuthu/merlin-ai-code-review)
- [codex-plugin-cc](https://github.com/openai/codex-plugin-cc)
- [AgentFlow](https://github.com/shouc/agentflow)
- [Gas Town](https://github.com/steveyegge/gastown)
- [Beads](https://github.com/steveyegge/beads)
- [CAID](https://github.com/JiayiGeng/CAID) (CMU paper: arXiv 2603.21489)
- [AgentConductor](https://arxiv.org/abs/2602.17100)
- [Agyn](https://arxiv.org/abs/2602.01465)
- [lalph](https://github.com/tim-smart/lalph)
- [operator](https://github.com/untra/operator)
- [parallel-worktrees](https://github.com/SpillwaveSolutions/parallel-worktrees)
- [OpenHands](https://github.com/All-Hands-AI/OpenHands)
- [MetaGPT](https://github.com/geekan/MetaGPT)
- [Atomic Agents](https://github.com/BrainBlend-AI/atomic-agents)
- [Orloj](https://github.com/OrlojHQ/orloj)
- [OpenAI Swarm](https://github.com/openai/swarm) (deprecated)
- [smolagents](https://github.com/huggingface/smolagents)
- [Anthropic 2026 Agentic Coding Trends Report](https://resources.anthropic.com/2026-agentic-coding-trends-report)
