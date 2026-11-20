# Maestro Control Plane Implementation Handoff

This document is an implementation brief for a future Copilot CLI session. The instruction-only
improvements are already in `SKILL.md`, `agent-prompts.md`, `director-protocol.md`, and
`lifecycle.md`, with the pinned instruction-only model catalog in `compute-routing.md`. This
roadmap covers the automation that instructions alone cannot enforce reliably.

## Outcome

Build an installable Copilot CLI Maestro plugin/workflow that delivers a feature from research
through reviewed draft PR while keeping the main session as the user-facing orchestrator. It must:

- schedule dependency-ready work with maximum safe parallelism;
- isolate writers in branches/worktrees;
- collect agent, external CLI, shell, git, test, review, and PR state deterministically;
- supervise workers without consuming the orchestrator's context;
- recover and resume from durable state;
- resolve model availability/capabilities and enforce task/risk compute routes deterministically;
- enforce integration, validation, review, approval, and PR gates;
- produce concise status and escalation events rather than raw worker transcripts.

The target architecture is a workflow/control plane, not one parent agent with an ordinary child
director that is expected to see its siblings.

## Current problems to solve

- Agent, shell, external CLI, worktree, test, todo, and review state is fragmented.
- A child director cannot reliably discover or inspect sibling agents and side-chained CLI workers.
- Monitoring and status reconstruction consume orchestrator context.
- Silent workers can run for too long without a code/test artifact.
- Todo state becomes stale after commits, failed tests, takeovers, and review results.
- Independent work is sometimes launched serially even when ownership is non-overlapping.
- Approval-heavy shell usage creates unnecessary interaction cost.
- Review PASS results are not bound to a frozen commit and can become stale after later fixes.
- Fix verification can create tunnel vision; fresh defects are still found during PR review.
- Repeated full-review/fix loops can fail to converge, rediscover the same findings, and consume
  agent time without increasing confidence.
- Findings lack deterministic semantic fingerprints, introduced/pre-existing classification,
  bug-class grouping, and targeted adjudication for disputed or expensive corrections.
- Personal review quality is not measured across runs, and repeated/escaped defect classes are not
  automatically proposed for promotion into tests, checks, instructions, or durable lessons.
- Static compute routing cannot inspect live model availability, validate model/effort/context
  compatibility, persist cost/latency/quality outcomes, or tune routes from representative evals.
- A hardcoded model catalog will drift as models are added, renamed, restricted, or removed.
- Resume depends too heavily on conversation summaries rather than a durable run ledger.

## User experience

Provide commands equivalent to:

```text
/maestro <objective>
/maestro research <objective>
/maestro plan <objective>
/maestro execute <spec-or-plan>
/maestro status
/maestro resume [run-id]
/maestro pause
/maestro cancel
```

`/maestro status` should read structured state directly and return:

- current phase and frozen candidate SHA;
- critical path and ready queue;
- active workers with last verified evidence;
- planned and actual compute route, model, effort, context, fallback, and escalation;
- integration queue;
- validation and review gates;
- blockers and decisions;
- next automatic action.

## Architecture

### 1. Deterministic runtime collector

Implement a non-LLM collector that periodically or event-driven reads:

- task/dependency state;
- known agent status and responses;
- external Copilot CLI session/resume IDs;
- shell/background process status and bounded logs;
- branch/worktree status and commit heads;
- changed files and diff hashes;
- build/test/lint command results;
- review candidate and finding ledger state;
- PR state and review feedback.

Normalize this into compact events. Never require the director to infer progress from process
uptime, tool-call counts, or raw logs.

### 2. Durable run ledger

Use SQLite or an equivalent local durable store. Suggested entities:

- `runs`: objective, mode, repositories, phase, status, timestamps;
- `tasks`: objective, owner, status, dependencies, risk, evidence deadline, task class,
  verification strength, latency class, context breadth;
- `workers`: type, agent/shell/session/resume ID, worktree, branch, log, status, planned/actual
  compute profile;
- `model_profiles`: stable route ID, capability tier, preferred/fallback models, effort, context
  mode, risk floor, version/effective dates;
- `model_catalog`: provider/family/model ID, availability, supported effort/context/tool features,
  cost/credit metadata, observed latency, source and refresh time;
- `routing_decisions`: task, selected profile, matched rules, risk-floor override, rationale,
  fallback/escalation, decision source;
- `task_attempts`: task, worker, profile/model/effort/context, start/end, result, insufficiency
  signal, retry/escalation sequence;
- `compute_metrics`: task attempt, wall time, input/output/cache/reasoning tokens, cost or AI
  credits, first-pass success, validation result, rework, human correction, accepted artifact;
- `model_evals`: representative case, acceptance criteria, candidate profile, result, latency,
  cost, repair effort, approval state;
- `artifacts`: commit, diff hash, file, design/spec, build/test result;
- `events`: normalized progress, failure, takeover, decision, completion;
- `decisions`: question, options, selected answer, source;
- `validation_runs`: command, scope, commit, result, output reference;
- `review_candidates`: base, head, diff hash, risk tier, current/stale;
- `review_runs`: reviewer role/model/session, candidate, scope, verdict;
- `findings`: stable ID, semantic fingerprint, bug class, introduced/pre-existing/uncertain origin,
  severity, confidence, invariant, location, disposition, fix/test, first/last seen candidate;
- `review_rounds`: candidate, kind (full/delta/closure/adjudication/checkpoint), sequence, scope,
  model/session, finding IDs, result;
- `review_metrics`: full/delta round counts, accepted findings, false positives, pre-existing
  findings, accepted risks, validation failures, escaped defects, rounds-to-pass;
- `learning_promotions`: source finding/incident, bug class, proposed test/check/instruction/lesson,
  acceptance state and resulting artifact;
- `review_evals`: escaped defect/revert case, expected finding, reviewer configuration, result;
- `approvals`: action, scope, requester, decision, timestamp;
- `pull_requests`: repository, target, ID, draft/ready state, feedback status.

Every event must be attributable to a run, task, worker, and candidate commit where applicable.

### 3. Deterministic compute router

Replace the instruction-only hardcoded lookup with a deterministic router that:

- imports the approved route policy from `compute-routing.md` without letting an LLM rewrite it;
- discovers the runtime's currently available models and supported reasoning/context options;
- resolves stable route IDs to a compatible model, effort, and context tier;
- applies review/security risk floors before latency or cost preferences;
- distinguishes focused prompt scope, fresh-session isolation, and long-context runtime support;
- validates that a requested agent type permits the selected model override;
- uses the approved static fallback only when the preferred model is unavailable;
- emits `ROUTING_INVALID`, `MODEL_UNAVAILABLE`, `PROFILE_INCOMPATIBLE`, and
  `COMPUTE_ESCALATION_EXHAUSTED` events instead of improvising;
- permits one evidence-backed change to context, effort, or model capability per task attempt;
- never silently downshifts a review/safety task below its approved risk floor.

The initial automated router must reproduce the static matrix exactly. Adaptive selection is a
later, separately approved stage after representative eval evidence exists.

### 4. Dependency-aware scheduler

The scheduler should:

- select all ready tasks whose dependencies are complete;
- enforce file/component ownership and one active writer per worktree;
- launch independent tasks in one batch;
- avoid speculative fan-out;
- keep shared contracts, composition roots, migrations, and integration with the orchestrator
  unless ownership is unambiguous;
- transfer command-only tails to the orchestrator;
- pause only for explicit decisions, external dependencies, approvals, or active asynchronous
  operations that will notify.

### 5. Workflow-level director

Run the director as a workflow supervisor or isolated session that consumes collector snapshots.
Do not make it an ordinary child that must discover siblings.

The director:

- evaluates progress against artifact/evidence deadlines;
- detects silence, drift, duplicate ownership, stale review candidates, failed validation, and idle
  critical paths;
- detects planned/actual compute mismatch, risk-floor violations, unsupported route combinations,
  and exhausted escalation budgets;
- emits structured `TAKEOVER_REQUIRED`, `QUARANTINE_REQUIRED`, `REVIEW_STALE`, and
  `DECISION_REQUIRED` events plus compute-router events;
- recommends one next orchestration action;
- never edits code or makes product decisions.

The orchestrator remains the only component that talks to the user, integrates work, and executes
takeover or approval actions.

### 6. Orchestrator adapter

The adapter should turn director events into bounded actions:

- stop/yield request;
- one status check;
- quarantine and fresh worktree creation;
- todo ownership transfer;
- bounded context/effort/model escalation or task reframe from deterministic router evidence;
- integration and validation commands;
- structured user update;
- continuation scheduling.

It should pass compact summaries to the language model rather than raw transcripts.

### 7. Review engine

Automate the instruction-defined review pipeline:

1. Run optional lightweight checkpoint reviews at coherent implementation slices and risky
   integration boundaries after executable verification; retain them as advisory evidence only.
2. Freeze exact target base, head, and diff hash for the integrated candidate.
3. Build an impact map of changed files, symbols, callers, contracts, configuration, persistence,
   deployment, tests, and docs.
4. Classify risk:
   - low: local behavior, narrow blast radius, no state/security/API/deployment impact;
   - medium: shared behavior, state, integration, recovery, accessibility, or public contract;
   - high: auth/security/privacy, concurrency, persistence/migration, money/data loss, deployment,
     broad public API, or irreversible behavior.
5. Select only applicable reviewers, preserving Maestro's broad-plus-specialist and cross-model
   controls rather than replacing them with a generic reviewer swarm.
6. Resolve the approved static review profiles: balanced-high for low risk; deep-fast plus
   clean-medium-gemini for medium; deep-high plus clean-high-gemini for high; security-critical plus
   clean-critical-gemini for critical security/privacy/data loss.
7. Launch independent reviewers in parallel against the same immutable candidate.
8. Fingerprint and deduplicate findings, classify introduced/pre-existing/uncertain origin, and
   group them by bug class. Existing unresolved findings retain their IDs.
9. Adjudicate disputed, weakly evidenced, or broad/high-cost findings with one independent
   verifier before corrective scope expands. Do not use multi-agent voting by default.
10. Require fix commits, same-bug-class impact-surface search, and regression evidence.
11. Run targeted closure verification.
12. Run a fresh clean-room review for medium/high risk without exposing the old finding list first.
13. Enforce convergence: one parallel batch against the same complete frozen candidate is one full
    round regardless of reviewer count; allow maximum three full rounds per candidate lineage;
    after broad and clean-room scans use delta/closure review unless the impact map changes
    materially; prohibit repeated findings and merge clean-room rediscoveries after ledger access.
14. Invalidate review PASS automatically whenever the head/diff hash changes.
15. Record per-run review and compute metrics and detect repeated bug classes, reverted fixes, and escaped
    defects. Create proposed learning promotions and reviewer eval cases; do not mutate normative
    instructions without user acceptance.
16. When approved, open a draft PR and ingest repository-native/Copilot findings into the same
    ledger.
17. Require final PR head to equal the reviewed/validated head.

Reuse immutable build/test results across reviewers. Permit a reviewer to run only a narrow
reproducer or missing test.

### 8. Approval manager

Reduce approval volume without weakening safety:

- prefer built-in read/edit tools;
- group independent shell reads or validations into bounded batches;
- show the user the exact batch and purpose once;
- cache approval only for the explicit scope allowed by the host;
- keep destructive, deployment, grant, secret, production, merge, and PR-ready actions separately
  gated.

This personal control plane may batch execution approvals but must not define repository/team
governance such as CODEOWNERS, required human approval counts, branch protection, PR templates,
merge queues, or deployment policy. It consumes those policies when present.

### 9. PR gate

After user agreement on proposed changes:

- create the draft PR after local integration and initial review, not after all work is over;
- run PR-native review concurrently with final validation;
- ingest review threads/findings into the ledger;
- reply to review threads in the same pass as fixes;
- rerun affected tests and delta review;
- mark ready only when the PR head is the final reviewed head.

## End-to-end workflow

```text
frame/reconcile
  -> research
  -> design
  -> specification
  -> dependency/file-ownership plan + compute profiles
  -> deterministic route validation
  -> parallel isolated implementation
  -> executable verification + lightweight review checkpoints
  -> integration checkpoints
  -> frozen candidate + impact/risk map
  -> parallel applicable reviews
  -> deduplication/adjudication + bug-class fixes + finding closure
  -> clean-room final review
  -> convergence check + delta-only closure
  -> draft PR-native review
  -> final validation/delta review
  -> review/compute metrics + proposed learning promotions
  -> ready PR/handoff
```

At every transition, reconcile todos, workers, artifacts, commits, validation, findings, and
external blockers.

## Implementation stages

### Stage 1: Local MVP

- Copilot CLI plugin/skill command surface.
- SQLite run ledger.
- Task/dependency/worker registry.
- Persist the approved static compute profile catalog and planned/actual route fields without
  changing route selection.
- Git worktree and commit tracking.
- Shell/session registration and status collector.
- `/maestro status` and `/maestro resume`.
- Instruction-compatible director snapshots and takeover events.

### Stage 2: Automated execution

- Runtime model catalog discovery with supported model/effort/context/tool metadata.
- Deterministic compute router that initially reproduces `compute-routing.md` exactly.
- Risk-floor and route-compatibility validation before worker launch.
- One approved static fallback and one evidence-backed compute escalation per task attempt.
- Ready-task scheduler.
- Isolated worktree creation.
- Worker launch adapters for agents and external Copilot CLI sessions.
- Evidence deadlines and automatic quarantine workflow.
- Phase-gate and todo reconciliation.
- Approval batching.

### Stage 3: Review automation

- Diff/symbol impact mapper using available code intelligence.
- Risk classifier and reviewer selector.
- Frozen candidate/diff hash.
- Lightweight checkpoint reviewer trigger after executable evidence.
- Finding fingerprinting, deduplication, origin classification and bug-class grouping.
- Targeted finding adjudicator.
- Finding ledger, closure verifier, clean-room reviewer.
- Full/delta round counter and three-round convergence controller.
- Review invalidation on head change.
- Personal review and compute metrics, escaped-defect/revert eval corpus and proposed learning
  promotions.

### Stage 4: PR integration

- Draft PR creation after explicit approval.
- Repository-native/Copilot review trigger or monitoring.
- Thread/finding ingestion and reply tracking.
- Final reviewed-head enforcement.

### Stage 5: Eval-driven compute optimization

Only after Stage 2 has captured representative outcomes:

- build a corpus spanning lookup, command, edit, research, implementation, architecture, review,
  adjudication, security, and clean-room tasks;
- predeclare acceptance criteria for correctness, evidence completeness, review precision, and
  validation;
- compare candidate profiles on the same inputs and frozen source state;
- measure wall time, cost/AI credits, token categories, retries, repair effort, human correction,
  false positives, and escaped defects;
- optimize cost and wall time per accepted result rather than per-call price;
- run route changes in shadow mode before enforcing them;
- propose downshifts only when a cheaper/faster profile consistently preserves the quality bar;
- propose upshifts only when a route has repeated capability failures after adequate context and
  effort;
- require user acceptance before changing the normative route catalog or risk floors;
- retain route version/effective dates so historical runs remain interpretable.

The router must not learn directly from one successful or failed run, vendor benchmarks, or model
marketing claims.

### Stage 6: Runtime enhancements if needed

Only pursue Copilot CLI/runtime changes when plugin APIs cannot provide:

- event subscriptions for all worker types;
- sibling/global task visibility;
- durable workflow timers;
- detached supervisor lifecycle;
- direct PR-review event integration.

The MVP must not depend on these enhancements.

## Acceptance criteria

- A run resumes after CLI restart from durable state without reconstructing work from chat.
- Every active agent, shell, and external CLI worker appears in one status view.
- No worker remains silent beyond its evidence deadline without takeover/quarantine.
- Independent ready tasks launch concurrently without overlapping writer ownership.
- Todo status matches referenced commits/tests/reviews.
- Every AI worker has a planned and actual route; launch is blocked when the profile is missing,
  incompatible, or below the task's risk floor.
- The runtime catalog resolves model availability and supported effort/context values without
  allowing an LLM to invent a replacement.
- One static fallback and one evidence-backed compute escalation are enforced per task attempt.
- Focused/fresh/long context modes are distinguishable in durable evidence; independent reviewers
  are always fresh.
- Review PASS is tied to an exact base/head/diff hash and becomes stale automatically after change.
- Medium/high-risk work receives closure verification and a clean-room final review.
- No candidate lineage exceeds three full review rounds without stopping for approach
  reconsideration or a material impact-map change.
- Semantically duplicate findings keep one stable ID; disputed or broad/high-cost findings are
  adjudicated before corrective scope expands.
- Fixes search the changed impact surface for the recorded bug class and ledger remaining
  occurrences explicitly.
- Review metrics persist across runs, and repeated/escaped defect classes produce proposed
  learning promotions without silently editing normative instructions.
- Compute metrics persist across runs, and an approved route change can be traced to representative
  eval evidence rather than one anecdotal run.
- Draft-PR findings use the same ledger and cannot be silently dropped.
- Final PR head equals the final reviewed and validated head.
- User-facing progress continues on cadence without flooding the orchestrator context with raw logs.

## Non-goals

- The director does not edit code or replace the orchestrator.
- The scheduler does not make product, security, rollout, cost, or data-ownership decisions.
- The compute router does not lower review/safety risk floors, change normative routes, or optimize
  solely from vendor benchmarks without user-approved representative eval evidence.
- The plugin does not deploy, grant access, merge, or mark PRs ready without the required approval.
- The personal Maestro plugin does not define team/repository review governance, CODEOWNERS,
  required human approval counts, branch protection, PR templates, merge queues, or deployment
  policy; a separate engineering-excellence flow owns those standards.
- Anthropic-inspired controls are adopted only where they improve this personal workflow; they do
  not replace Maestro's frozen candidates, finding ledger, clean-room review, cross-model
  diversity, worker registry, takeover/quarantine, or stale-PASS guarantees.
- More agents is not a success metric; safe wall-clock reduction and evidence quality are.

## Suggested prompt for the implementation CLI

```text
Implement the Maestro control plane described in:
C:\Users\sinansar\.copilot\skills\maestro\references\control-plane-roadmap.md

Start by inspecting the current Copilot CLI plugin, skill, workflow, task, session, worktree, and
history APIs. Produce an architecture and staged implementation plan before editing. Build the
Stage 1 local MVP first with a durable SQLite run ledger, unified worker registry, deterministic
collector, persisted static compute profiles, /maestro status, and /maestro resume. Preserve the
existing instruction behavior and do not require Copilot runtime changes for the MVP. In Stage 2
add deterministic model-catalog discovery and route resolution that initially reproduces
compute-routing.md exactly, validates model/effort/context compatibility and risk floors, and
enforces one fallback/one escalation. Validate resume, worker visibility, routing mismatch,
stale-worker takeover signals, and review-candidate invalidation end to end. Preserve Maestro's
stronger frozen-candidate, clean-room, cross-model, and takeover controls. In Stage 3 add finding
fingerprints/deduplication, origin and bug-class classification, targeted adjudication, a
three-full-round convergence budget, delta-only re-review, personal review/compute metrics, eval
cases, and user-approved learning promotions. Add adaptive route proposals only in Stage 5 after
representative shadow evals; never self-modify normative routes or risk floors. Do not implement
team/repository governance in this plugin.
```
