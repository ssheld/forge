# AGENTS.md

Agent policy for this repository. `AGENTS.md` and `CLAUDE.md` are intentional
mirrors (identical after line 1). Neither file is canonical over the other --
edit either one and update the other to match. A CI workflow enforces that
the two files stay in sync.

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

## Default Build Direction

Until the repo records a different decision:

- Build a thin local CLI first.
- Prefer plain Python and straightforward subprocess/file orchestration.
- Use git worktrees and GitHub as the durable workflow substrate.
- Keep provider integrations behind small adapters.
- Do not introduce LangGraph or another workflow engine unless the simpler
  controller demonstrably stops scaling.

## Coding Standards

### Core Principles
- Prefer simple, testable designs.
- Keep changes minimal and coherent.
- Preserve backward compatibility unless explicitly planned.

### Readability
- Prefer explicit, readable structure over compact or clever code.
- Avoid high-density control flow such as nested ternaries.
- Remove comments that only restate what the code makes obvious.
- Keep useful abstractions, but remove accidental or redundant ones when
  doing so clarifies the code without widening scope.

### Language and Framework Conventions
- Primary language: Python (per Default Build Direction).
- Use explicit, domain-oriented names over abbreviations.
- Match terms used in `README.md` and user-facing docs.
- Prefer explicit failure states over silent fallbacks.
- Keep logs actionable, structured when possible, and include failure context.
- Prioritize correctness and maintainability first; add performance budgets
  once critical paths are identified.

### Testing Standards
- Treat test-driven development as the default for behavior-changing code.
- Use RED / GREEN / REFACTOR when practical.
- For bug fixes, start with a reproducing test when practical.
- If test-first is not realistic, state why and add coverage before merge.
- Default coverage floor: 80% when the repo has coverage tooling.

### Code Review Checklist
- [ ] Matches project architecture and naming conventions.
- [ ] Keeps code readable without dense or clever control flow.
- [ ] Includes tests for changed behavior.
- [ ] Updates docs when behavior changes.
- [ ] No unrelated changes.

## Project Commands

Setup, test, and operational commands for this repository. Update this
section when the canonical workflow changes.

### Setup
```bash
# No setup commands yet -- update when the CLI is bootstrapped.
```

### Test Commands
```bash
# No test commands yet -- update when test infrastructure exists.
```

### Development / Run Commands
```bash
# No dev commands yet -- update when the CLI entrypoint exists.
```

### Release / Operations Commands
```bash
# No release commands yet -- update when packaging or deploy exists.
```

## Verification Loop

Before reporting substantive work as complete:

1. Run the most relevant tests for the changed behavior.
2. Run any available static checks or linting relevant to the change.
3. Review the diff for accidental edits, debug leftovers, and doc drift.
4. State clearly what was verified and what was not.
5. If a step is unavailable or not meaningful for the repo, say so explicitly
   instead of implying it ran.

Use the project commands above when available.

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

## PR Authoring Standards

- When creating a pull request, include a complete PR body; do not open with
  a one-line summary only.
- Use `.github/pull_request_template.md` when present.
- Use an imperative, outcome-focused title that reflects the actual change
  scope.
- Keep PR scope coherent; split unrelated work into separate PRs.
- Prefer opening as draft until validation is complete, unless the user asks
  otherwise.
- PR body must include:
  - Summary of problem and approach.
  - Files/areas changed with concise rationale.
  - Verification Loop evidence (commands run, outcomes, skipped steps).
  - Risks, rollback considerations, and any residual gaps.
  - Explicit docs consistency note (`docs/architecture.md`/`README.md`
    updated or no impact).
- If any Verification Loop step was not run, explicitly state why and what
  remains unverified.
- If the PR addresses review feedback, include itemized mapping per the
  review feedback response format below.

## Review Format

### Review Transport
- Prefer a formal GitHub PR review whenever the platform/API allows it.
- Use a standalone PR conversation comment only as fallback when a formal
  review cannot be submitted.
- Use inline review comments only for actual file/line-specific findings.

### Canonical Templates

**Formal PR review body**
```md
> 🤖 **Review by {Model Name}** · via {Client Tool}

1. `path/to/file.py:123`

   **Recommended** · High confidence — Brief finding title.
   Short explanation.

## Strengths
- ...

## Merge Recommendation
{Approve | Approve with Changes | Request Changes}

## Top Risks
- ...

## Suggested Additional Tests
- ...
```

**Inline review comment**
```md
🤖 `{Model Name}`

**Recommended** · High confidence — Brief finding title.
Short explanation.
```

**Review feedback response**
```md
> 🤖 **Feedback response by {Model Name}** · via {Client Tool}

| # | Finding | Status | Evidence / Rationale |
|---|---------|--------|----------------------|
| 1 | ... | Resolved | `abc1234`; `path/to/file`; what changed |
| 2 | ... | Not Changed | why, risk/tradeoff, decision needed |

## Remaining Items
- ...
```

### Severity Labels
- Critical — must fix before merge (blocks merge)
- Recommended — strong improvement (should fix before merge)
- Optional — minor suggestion

### Confidence Indicators
- High confidence (80-100) — certain this is a bug or vulnerability
- Medium confidence (50-79) — likely an issue, but context-dependent
- Low confidence (0-49) — stylistic or speculative

Suppress findings scoring below 80 from the posted review by default.
Include sub-80 findings only if they represent a security or data-safety risk.

### Do Not Flag
- Purely stylistic formatting preferences (handled by linters)
- Import ordering
- Minor naming disagreements that do not affect clarity
- TODOs already tracked in issues
- Changes in files outside the PR diff
- Pre-existing issues not introduced by the PR; use git blame to distinguish
  inherited code from new changes
- Issues that linters, formatters, or type checkers already catch
- Nitpicks with no functional, security, or maintainability impact

### Review Posting Checklist
Before posting any review, verify:
- The correct transport was used (formal review when available)
- The attribution header is present with actual model/version
- Findings appear first, ordered by severity
- Required footer sections are present (Strengths, Merge Recommendation,
  Top Risks, Suggested Additional Tests)
- No local filesystem paths (use repo-relative `path:line`)
- No mixed top-level/inline formatting
- Sub-80 findings are suppressed unless security/data-safety risk

### Review State Fallback
If the platform blocks the intended formal review state:
- Prefer a `COMMENT` review submission if supported.
- Otherwise use a standalone PR conversation comment.
- Keep the same textual merge recommendation in the body.
- Add a one-line note: `Formal review state unavailable: <reason>`.

### GitHub Reference Rules
- Never use local filesystem paths in GitHub review comments.
- Use repo-relative `path:line` references or GitHub links.
- When a finding spans multiple locations, list each affected location
  separately.

## Pre-Commit Documentation Checklist

Before every commit:

1. Check if changes affect `docs/architecture.md`. If yes, update it.
2. Check if `README.md` needs updating (CLI usage, setup, env vars, API
   endpoints, project structure). If yes, update it.
3. Do not finalize a commit until docs are consistent with code changes.
