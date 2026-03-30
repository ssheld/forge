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
| **Reviewer** | Reviews implementation, scores, produces ReviewReport | Executor completion or PR detection |
| **Fixer** | Routes findings to implementing agent, re-verifies | Reviewer findings |
| **Merger** | Checks gates, merges, rebases remaining PRs | Manual or periodic |

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
