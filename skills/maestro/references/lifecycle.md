# Maestro Lifecycle Deliverables

Use repository-native documentation locations and naming conventions. Do not create
duplicate planning systems when the repository already has SDD, ADR, spec, or tracking
conventions.

## 1. Feature brief

Capture:

- problem and target users;
- observable outcome;
- in-scope and out-of-scope behavior;
- environments and rollout target;
- constraints and owner decisions;
- acceptance criteria;
- known external dependencies.

## 2. Research report

The report must distinguish:

- current implemented behavior;
- declared/documented behavior;
- reusable components and prior art;
- implementation gaps;
- conflicting or stale documentation;
- external contracts and owners;
- verified facts versus inference.

Every material claim should identify its source: path/symbol, commit, authoritative
documentation, work item, telemetry query, or owner statement.

For every external side-effecting API, add a contract-evidence matrix with:

- operation and side effect;
- authoritative source or owner;
- authentication and authorization mode;
- stable identifiers and ownership enforcement;
- idempotency, retry and lost-response behavior;
- consistency and status semantics;
- timeout, size, rate and retention limits;
- evidence state: documented, empirically verified, inferred, or unknown.

Unknown security, identity, replay, or consistency behavior is a readiness blocker. The
implementation must remain unavailable or fail closed until the evidence exists.

## 3. High-level design

Include:

- component and trust boundaries;
- request, data, state, and identity flows;
- API/events/payload contracts;
- persistence and concurrency model;
- retries, idempotency, cancellation, and ambiguous outcomes;
- a per-stage mutation failure matrix: possible side effect, timeout/lost response, safe retry,
  reconciliation, and customer action;
- the complete identity chain for each hop: issuer, audience, scope/role, client app, user/tenant
  claims, fallback and ownership-enforcement point;
- composed timeout budgets, including token acquisition, sequential downstream calls and
  response-body reads;
- synchronous acceptance versus asynchronous completion when downstream work can outlive the
  request;
- data-plane suitability for large/binary payloads, including encoding expansion, copies,
  buffering, streaming, resumability, model-context exposure and token cost;
- security, privacy, accessibility, reliability, and observability;
- deployment, compatibility, migration, and rollback;
- alternatives considered and decisions still open.

Use a diagram when it materially improves comprehension.

## 4. Specification

Translate the design into requirements that can fail a test:

- user scenarios and state transitions;
- input/output schemas;
- source precedence and authority;
- validation and authorization rules;
- exact failure behavior;
- telemetry categories and prohibited data;
- compatibility and feature-flag behavior;
- performance and availability expectations;
- outer and per-stage timeout budgets with measurable margin;
- maximum payload size, encoded size, copy/buffering budget and concurrency behavior;
- whether any request, payload or tool result enters model context;
- fail-closed behavior for missing delegation, grants, ownership evidence or external-contract
  decisions;
- acceptance tests.

Avoid aspirational language such as "support," "robust," or "secure" without measurable
behavior.

## 5. Execution plan

Each task must state:

- bounded objective;
- owned files/components;
- dependencies;
- first expected code/test artifact by the first director cadence;
- expected artifacts or behavior;
- focused validation;
- stop/handoff condition;
- worker registry fields when delegated: owner, agent/shell/session/resume ID, worktree/branch,
  log path, expected artifact, evidence deadline, and takeover route;
- static compute profile from `compute-routing.md`: route ID, exact model, reasoning effort,
  focused/fresh/long context mode, task/risk class, rationale, escalation trigger, and fallback;
- approval-efficient command batches and which operations require separate authorization;
- review risk tier, expected impact map, and applicable independent specialists.
- each reviewer's artifact timing: normal first-cadence evidence, or an explicitly justified
  large-scale high-risk twelve-minute complete-report deadline; the three-minute user heartbeat
  never changes;

For medium/high-risk cross-service work, the plan must also name:

- the contract-evidence matrix and unresolved owner decisions;
- the identity-chain and negative app-only/missing-assertion tests;
- the mutation failure/reconciliation matrix;
- the timeout-composition and asynchronous-lifecycle tests;
- the payload/data-plane evaluation;
- applicable auth, resilience, performance, platform-impact and privacy reviewers;
- the adversarial test matrix and test-double realism audit;
- the final acceptance-and-risk coverage table.

Create a file-ownership matrix before parallel implementation. Shared contracts,
composition roots, central configuration, and migrations should usually integrate through
the orchestrator. If the first artifact cannot be named without more broad investigation,
the task is still research/design work and is not ready for an implementer.

## 6. Implementation evidence

Each implementer returns:

- branch/worktree and head commit;
- actual compute route, model, effort, context mode, and any fallback/escalation;
- files changed;
- behavior implemented;
- tests/builds run and exact results;
- assumptions and unresolved risks;
- integration notes;
- confirmation that no unrelated changes were included.

For external CLI side-chains and background commands, record the same evidence plus shell/session
or resume ID, log location, process completion state, and the exact worktree being modified. An
untracked side-chain is not valid Maestro execution.

A plan, file inventory, or description of intended edits is not implementation evidence.
When an agent is abandoned for failing to yield or stop, record the quarantined worktree
and last trusted commit. Resume in a fresh worktree rather than creating a second writer.

## Execution liveness

Once execution starts, the orchestrator continues every ready task without waiting for a
user prompt. Status questions and process discussions do not pause the run. After answering
them, resume the critical path in the same turn.

Do not end with a status-only response while unblocked work remains. Yield only for completion,
an explicit decision/external blocker, an asynchronous operation that will notify on completion,
or an armed continuation schedule. Stopping director monitoring or quarantining an agent does
not stop direct implementation.

## Compute routing evidence

Before launching any AI worker, classify it with `compute-routing.md`. Static routing is the
instruction-only source of truth; do not probe the live catalog or substitute an unlisted model.
The execution-plan entry and worker registry must agree on:

- route ID and task/risk class;
- exact model and reasoning effort (or runtime default);
- focused, fresh, or long context mode;
- why the route satisfies the task and any risk floor;
- the observable trigger for context expansion, effort increase, capability increase, or
  task-scope rejection;
- whether the one static launch fallback was used.

An agent reports `CONTEXT_INSUFFICIENT`, `EFFORT_INSUFFICIENT`,
`CAPABILITY_INSUFFICIENT`, or `TASK_SCOPE_INVALID`; it does not choose or launch its own
replacement model. Permit one compute escalation for that delegated attempt. If the escalated
attempt still cannot satisfy the evidence bar, stop and reconsider the task, design, or validation
strategy.

At handoff, retain the actual route, fallback/escalation, completion result, and available wall
time/token/cost information. These observations may inform a later approved routing change but do
not modify the static matrix automatically.

## 7. Review evidence

### Required evidence for high-risk cross-service work

Before any reviewer may return PASS, freeze and provide:

- the external contract-evidence matrix;
- the end-to-end identity and ownership chain;
- the per-stage mutation failure/retry/reconciliation matrix;
- the timeout-composition table and asynchronous-lifecycle decision;
- the binary/data-plane and model-context analysis when applicable;
- the adversarial test matrix and evidence that mocks/fakes match the relevant wire behavior;
- the acceptance-and-risk coverage table.

A disabled feature, missing grant or future rollout does not excuse unsafe latent behavior. Unknown
security or replay behavior must fail closed in the reviewed implementation.

### Continuous checkpoint review

Before the final candidate exists, review coherent implementation slices and risky integration
boundaries when the expected defect cost justifies it:

- run an executable check first and retain its output;
- freeze the checkpoint head and changed slice;
- use a fresh-context lightweight bug, simplification, or end-to-end behavior review;
- record findings as advisory checkpoint evidence;
- promote valid findings into the same ledger when the final candidate is frozen.

Checkpoint review reduces late rework but is not a final PASS and does not replace the frozen
risk-tiered review, finding closure, or clean-room review.

Slice and checkpoint PASS results never aggregate into a final PASS. Medium/high-risk work requires
one review of the complete integrated base-to-head diff, including middleware ordering, shared
configuration, authentication resolution, dependent callers and unrelated sibling traffic.

Large-scale high-risk reviews may use a twelve-minute complete-report deadline without weakening
monitoring. The orchestrator still reports every three minutes and checks for runtime failure,
nested-delegation drift, wrong repository/tool initialization, or absent activity, but it does not
require an intermediate reviewer turn when the subagent runtime does not stream one. Missing the
twelve-minute report triggers takeover. A quarantined no-artifact attempt may be replaced once
only by a materially narrower/reframed assignment against the unchanged frozen candidate.

Freeze the review candidate before review:

- target branch/ref and exact base commit;
- exact head commit;
- base-to-head changed files and public symbols/contracts;
- affected callers, persistence, configuration, deployment, tests, and documentation;
- risk classification with rationale;
- reusable build/test evidence.

Choose the smallest sufficient review set:

- low risk: `balanced-high` broad review plus a final `fix-verifier` delta check;
- medium risk: `deep-fast` broad review plus applicable specialists in parallel, targeted
  `fix-verifier` closure, and a fresh `clean-medium-gemini` final review;
- high risk: `deep-high` broad adversarial review plus all justified specialists in parallel,
  targeted `fix-verifier` closure, a fresh `clean-high-gemini` final review, and draft-PR-native
  review when PR creation is in scope and approved;
- critical security/privacy/data-loss risk: `security-critical` review plus a fresh
  `clean-critical-gemini` review, with containment allowed while evidence closes.

Specialist selection is mandatory when its trigger is present:

- user delegation, token fallback or resource ownership -> authentication/authorization;
- non-idempotent cross-service mutation -> distributed resilience;
- payloads over 1 MiB, Base64/binary or streaming -> performance/memory/timeout;
- shared host middleware or routing -> platform-wide impact;
- sensitive payloads, identifiers or telemetry -> privacy.

High-risk candidates also require a fresh assumption-challenger review that evaluates whether the
accepted protocol, trust boundary, identity model, synchronous lifecycle and recovery design are
appropriate before checking implementation conformance.

Review findings must include:

- stable finding ID;
- semantic fingerprint used for deduplication;
- bug class;
- origin: introduced by this candidate, pre-existing, or uncertain;
- severity and confidence;
- violated invariant or acceptance criterion;
- file and line range;
- concrete failure scenario;
- why existing tests do not prevent it;
- minimal recommended correction;
- owner, disposition, fix commit, and regression-test evidence once resolved.

Track false positives and accepted risks explicitly. Do not silently drop findings.

Before acting on a disputed, weakly evidenced, or broad/high-cost finding, assign one independent
adjudicator. The adjudicator confirms, rejects, narrows, or marks the issue uncertain against the
same frozen candidate. Do not launch voting swarms by default. Credible critical security or
data-loss findings may be contained immediately while adjudication runs.

Fixes must search the changed impact surface for the same bug class. Fix all in-scope occurrences
or record why remaining occurrences are pre-existing, unrelated, accepted risk, or deferred. Do
not widen implementation scope silently.

Fix verification closes the finding ledger; it does not replace a fresh review. A clean-room
reviewer must inspect the corrected final diff without seeing the old finding list first. Any
post-PASS change invalidates the affected review evidence and requires a delta or full review based
on impact.

Review must converge:

- one full review round is one parallel batch scanning the same complete frozen candidate,
  regardless of how many broad/specialist reviewers are in that batch;
- targeted closure verification and changed-line delta review do not count as full rounds;
- allow at most three full rounds for one candidate lineage;
- after the initial broad review and required clean-room review, use closure/delta review unless
  the impact map changed materially;
- reviewers reference unresolved existing IDs and do not repost semantically identical findings;
  clean-room duplicates discovered before ledger access are merged afterward;
- at the cap, stop and reconsider design, tests, candidate scope, or review configuration.

When an approved draft PR exists, treat repository-native/Copilot review as another delivery gate.
Reconcile valid findings, add them to the same ledger, and ensure the final reviewed head equals the
PR head.

Before final PASS, verify that every row in the acceptance-and-risk coverage table has direct code,
test, reviewer and external-contract/owner evidence where applicable. Missing evidence is an open
finding, not an implicit PASS.

## 8. Completion report

Lead with:

- delivered capability;
- branch/PR/deployment state;
- validation results;
- unresolved external gates.

Include the final base/head review candidate, risk tier, applicable reviewers, finding ledger
summary, full/delta review-round counts, false-positive/pre-existing/accepted-risk counts,
draft-PR review state, proposed learning promotions, and confirmation that no unreviewed code
delta remains. Include planned versus actual compute routes, any fallback/escalation, and available
wall-time/token/cost evidence.

For high-risk cross-service work, also report unresolved contract-matrix rows, identity-chain
verification, timeout/data-plane decisions, adversarial-test results, test-double realism findings,
and the final acceptance-and-risk coverage table.

Then summarize meaningful architecture and behavior changes. Do not narrate the entire
process.

## Personal review learning

For each completed run, retain compact metrics:

- full review rounds and delta/closure passes;
- accepted findings and severity;
- false positives, pre-existing findings and accepted risks;
- validation failures and escaped defects when later known;
- repeated bug classes.

Repeated bug classes, reverted fixes and escaped defects create a proposed promotion into a
focused test, lint/check, Maestro instruction or durable lesson. Do not change normative
instructions without user acceptance. Repository-wide review policy, CODEOWNERS, approval counts,
branch protection, PR templates, merge queues and deployment governance remain outside this
personal workflow.
