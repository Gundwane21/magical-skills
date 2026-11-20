---
name: maestro
description: "Explicitly invoked end-to-end feature delivery orchestration: research, architecture, specification, dependency-aware planning, parallel implementation agents, a director agent, independent review, integration, validation, and PR handoff. Use only when the user invokes /maestro or explicitly asks for the Maestro workflow; do not auto-invoke for ordinary coding tasks."
---

# Maestro Feature Delivery

Orchestrate substantial feature delivery from an uncertain request to a validated,
reviewed implementation. Keep the main agent as the user-facing orchestrator and source
of truth. Delegate bounded work, not accountability.

Read these resources before starting:

- `references/lifecycle.md`
- `references/agent-prompts.md`
- `references/director-protocol.md`
- `references/compute-routing.md`

## Invocation modes

Interpret the text following `/maestro` as the feature objective unless it starts with
one of these modes:

- `/maestro <feature>`: run the complete lifecycle.
- `/maestro research <feature>`: stop after research findings and open decisions.
- `/maestro plan <feature>`: produce research, design, specification, and execution plan;
  do not implement.
- `/maestro execute <feature-or-plan>`: implement from an existing approved design/spec.
- `/maestro resume`: reconstruct state from session todos, tracking artifacts, branches,
  worktrees, agent IDs, and recent history, then continue the critical path.
- `/maestro status`: have the director produce one consolidated status report; do not
  start unrelated work.

If the requested mode is unclear but changes whether code will be modified, ask once
using the structured user-question tool. Otherwise default to the complete lifecycle.

## Non-negotiable operating rules

1. Verify repository freshness before investigating or changing code.
2. Define the root outcome and observable completion criteria.
3. Use dependency-aware session todos for every Maestro run.
4. Research current implementation and source-of-truth documentation before designing.
5. Convert findings into a testable design and specification before implementation.
6. Maximize safe parallelism, not agent count. Parallelize only independent work with
   explicit ownership and integration boundaries.
7. Keep one orchestrator responsible for decisions, integration, user communication, and
   final validation.
8. Use a dedicated director agent during multi-agent implementation. The director does
   not edit code or replace the orchestrator.
9. Use independent reviewer agents that did not implement the reviewed code. A reviewer owns its
   assigned review directly and must not launch a nested reviewer, security reviewer, research
   agent, or other child delegation unless the orchestrator explicitly assigned a separate,
   registered sub-review.
10. Do not declare completion until the integrated branch passes required validation and
    every review finding is fixed, explicitly accepted, or recorded as blocked.
11. While any delegated agent, background command, build, test, or validation remains active,
    send the user a concise progress update at least every three minutes, or more frequently when
    the host environment requires a stricter cadence. Report even when nothing has changed.
    Director check-ins do not satisfy this rule until the orchestrator relays the consolidated
    status to the user. Report blockers, failed validation, material scope changes, phase
    completion, and decisions needing user input immediately rather than waiting for the cadence.
12. Bound every delegated task with exact ownership, a first code-producing action, expected
    artifacts, and the narrow validation commands it may run. For implementation agents, planning,
    file discovery, and a proposed approach are not concrete progress after the design is accepted.
    By the first director cadence, require an owned-file diff, focused test result, or a
    source-evidenced blocker. A silent agent with no artifact triggers takeover immediately. An
    agent that reports exact evidence but has not produced an artifact may receive at most one
    final cadence with exploration frozen. Review deadlines are risk-scaled rather than copied from
    implementation. Low/medium reviews still owe an evidence-bearing artifact at the first
    cadence. A high-risk integrated, specialist, assumption-challenger, or clean-room review
    declared large-scale in the execution plan gets up to twelve minutes for its complete
    finding/PASS/missing-evidence report. The three-minute cadence remains a user heartbeat and
    runtime-health check; do not require a partial reviewer turn when the subagent runtime does not
    stream one. Intervene earlier only for failure, tool/scope error, delegation drift, or no
    observable activity across two cadences.
13. Delegation is not for a small command-only tail. Once implementation is complete and the
    remaining work is at most three known reads, checks, test commands, formatting commands, or
    commit steps, the orchestrator MUST execute them directly. Do not keep an agent alive merely to
    wait for or retry validation.
14. A delegated command may receive at most one targeted correction after a tooling/argument
    failure. If it still cannot run, the agent must stop and report the exact command, error, and
    current worktree state. It must not invent flags, widen scope, bootstrap unrelated modules, or
    substitute a broader command without orchestrator approval.
15. Treat stop messages as cooperative requests, not process termination. Delivery is not
    acknowledgment, and a running status is not safe for shared-worktree takeover. After one stop
    request, use an explicit runtime cancellation control when one is actually available to the
    orchestrator; the interactive CLI's interrupt/task controls do not imply that an agent-tool API
    can terminate a child. Perform one status check, then quarantine any still-running agent and its
    worktree. Continue from the last trusted commit in a fresh worktree; do not wait, poll
    indefinitely, or edit the quarantined worktree concurrently. Future output from an abandoned
    agent is ignored unless the orchestrator explicitly reviews and accepts it.
16. A director `TAKEOVER REQUIRED` signal is binding orchestration input. The orchestrator must act
    in the same turn, before unrelated work: stop or abandon the delegation, isolate its worktree,
    transfer todo ownership, and tell the user what action was taken. A `REVIEW GATE REQUIRED`
    signal is also binding: mark the candidate incomplete and obtain the missing integrated review,
    specialist, contract, adversarial-test, or coverage evidence before accepting PASS. Do not
    merely acknowledge either signal or wait for another cadence.
17. Do not delegate implementation that still requires broad architecture discovery or traversal
    across unresolved shared surfaces. Return that work to research/design, narrow it to one
    testable slice, or keep it with the orchestrator.
18. An active Maestro run with ready work is self-driving. User status questions, process
    questions, and other side questions are interruptions, not pause signals. Answer briefly, then
    continue the critical path in the same turn without requiring the user to say "continue."
    Do not end an active run with a status-only final response.
19. End or yield an active Maestro turn only when the outcome is complete, an explicit user
    decision or external dependency blocks all ready work, an active asynchronous operation will
    generate a completion notification, or a continuation schedule has been armed. If the runtime
    cannot execute between turns and ready work remains, arm a continuation schedule before
    yielding and stop it only when complete, blocked, or explicitly paused by the user.
20. Progress reporting is not execution. Heartbeats, director reports, todo updates, and status
    checks must not replace the next ready implementation action.
21. Reconcile the task graph at every phase boundary and after every commit, takeover, failed
    validation, or review result. A todo may be marked done only when its referenced artifact and
    evidence exist. Correct stale status immediately; do not carry inaccurate pending/in-progress
    tasks into status reports or handoff.
22. Track every worker, including external CLI side-chains and background commands, in one worker
    registry. Record its task, owner, agent/shell/session or resume ID, worktree/branch, log path,
    expected artifact, last verified evidence, and intervention deadline. A process that is not
    visible to the director must still be represented through this registry and monitored by the
    orchestrator.
23. Minimize approval friction. Prefer built-in read/edit tools, batch independent reads and
    unavoidable shell operations, and describe the bounded command batch before requesting
    approval. Do not trade away inspection or validation quality merely to reduce approvals.
24. Freeze every review candidate to an exact target base, head commit, and diff. Any code or
    configuration change after a review invalidates that review for the changed impact surface;
    record the new head and run the required delta or full review before treating the candidate as
    reviewed.
25. Select reviews by risk, not by a fixed reviewer count. Run independent applicable reviewers in
    parallel, reuse shared build/test evidence, and avoid serial generic reviews. Low-risk changes
    receive one independent review and a final delta check; medium/high-risk changes add only the
    specialists justified by the impact map.
26. A fix verification is not a fresh review. After findings are corrected, run a targeted closure
    verification and then a new clean-room review of the final integrated diff for medium/high-risk
    work. The clean-room reviewer must not receive the previous finding list before its independent
    scan.
27. Treat an approved draft PR as a review environment, not the end of delivery. When PR creation
    is in scope and the user has agreed to the proposed changes, open the draft after the local
    candidate is integrated, run repository-native/Copilot review while final validation continues,
    reconcile valid findings, and repeat the final delta review before marking the PR ready.
28. Review must converge. One parallel batch reviewing the same complete frozen candidate counts as
    one full review round regardless of reviewer count; targeted finding closure and changed-line
    delta checks are not full rounds. Permit at most three full rounds for one candidate lineage.
    A stopped/quarantined attempt that produced no usable review artifact does not count as a full
    round. It may receive at most one fresh replacement only when the assignment is materially
    narrowed or reframed around the same frozen candidate; do not relaunch the identical prompt or
    create an unbounded retry loop.
    After the initial broad review and any required clean-room review, use targeted closure and
    delta review unless the impact map changes materially. At the cap, stop the review/fix loop and
    reconsider the design, test strategy, or candidate scope rather than starting another broad
    scan.
29. Deduplicate findings by stable ID and semantic fingerprint. Classify each finding as introduced
    by the candidate, pre-existing, or uncertain, and name the underlying bug class. A reviewer must
    reference an unresolved existing ID instead of reposting the same defect. A clean-room reviewer
    may rediscover a defect because it cannot see the ledger before its scan; merge that duplicate
    into the existing ID afterward. A fix must search the changed impact surface for the same bug
    class and either correct every in-scope occurrence or ledger the remainder explicitly without
    silently widening scope.
30. Adjudicate findings before expensive or risky correction when confidence is disputed, direct
    evidence is missing, or the proposed fix has broad impact. Use one independent verifier by
    default rather than a reviewer swarm. Credible critical security or data-loss findings may
    trigger immediate containment while verification continues.
31. Run lightweight review continuously at coherent implementation slices and risky integration
    boundaries: executable verification first, then a fresh-context bug/simplification/behavior
    check when justified. These checkpoints are advisory evidence and never replace the frozen
    risk-tiered final review, targeted closure, or required clean-room review.
32. Record personal review-learning evidence per run: full and delta round counts, accepted
    findings, false positives, pre-existing findings, accepted risks, validation failures, and
    escaped defects when later known. Repeated bug classes, reverted fixes, and escaped defects must
    produce a proposed promotion into a test, lint/check, Maestro instruction, or durable lesson.
    Normative instruction changes still require user acceptance.
33. Maestro may impose stricter personal evidence and review controls, but it does not invent or
    replace repository/team governance such as CODEOWNERS, required human approval counts, branch
    protection, PR templates, merge queues, or deployment policy. Repository rules remain
    authoritative and belong in the separate engineering-excellence flow.
34. Classify every delegated AI task with the static route in `references/compute-routing.md`
    before launch. Record the route ID, exact model, reasoning effort, context mode, task/risk
    class, rationale, escalation trigger, and fallback in the execution plan and worker registry.
    Model selection is task-based, not merely agent-type-based.
35. Treat model capability, reasoning effort, and context as independent controls. Use focused
    context by default; launch a fresh agent for independent review/adjudication; use
    `long_context` only after targeted retrieval cannot cover a genuinely broad dependency
    surface. A large repository alone does not justify long context or a larger model.
36. Review and safety risk floors override latency or cost preferences. Follow the exact low,
    medium, high, and critical review routes in the compute matrix. Cross-model diversity must use
    a fresh model family at the required capability tier and never replace source evidence,
    executable validation, or applicable specialists.
37. Diagnose failure before changing compute: missing evidence expands context; incomplete
    investigation raises effort; adequate investigation with unresolved judgment raises model
    capability; an unbounded task returns to research/design. Permit at most one compute
    escalation per delegated attempt, then stop and reconsider scope, design, or validation.
38. Do not query or probe live model availability during instruction-only routing. If a selected
    model launch fails specifically because it is unsupported/unavailable, use the one static
    fallback in the compute matrix. A second launch/configuration failure stops with exact
    evidence; do not invent another model or effort setting.
39. Record the actual compute profile at handoff, including fallback/escalation and available
    wall-time/token/cost evidence. These observations do not authorize autonomous route changes;
    model-catalog or normative-policy changes remain user-approved until a future deterministic
    control plane is accepted.
40. Classify as high risk any change that combines a cross-service side effect with authentication
    delegation, non-idempotent writes, large untrusted payloads, shared host middleware, or
    ambiguous outcomes. A disabled feature flag, missing grant, or delayed rollout does not lower
    the implementation's review risk.
41. For every external side-effecting API, maintain a contract-evidence matrix covering source
    authority, authentication, authorization, identifiers, idempotency, retries, consistency,
    status semantics, timeouts, limits, and owner decisions. Label each claim documented,
    empirically verified, inferred, or unknown. Unknown security or replay behavior blocks
    readiness and must fail closed in code.
42. Map the complete identity chain for every side-effecting operation: issuer, audience, scopes
    or roles, client application, user claims, tenant, fallback behavior, and the component that
    enforces resource ownership. When user delegation is required, negative tests must prove that
    a missing assertion sends zero downstream mutation requests.
43. Compose timeout budgets explicitly. Record the outer request budget and every sequential or
    nested stage budget, including token acquisition and response-body reads. If final completion
    can exceed the synchronous request lifetime, separate durable acceptance from asynchronous
    processing status instead of holding the request open.
44. Any payload larger than 1 MiB, binary content, or Base64 transport requires a data-plane
    suitability review covering encoding expansion, copies, buffering, memory, concurrency,
    streaming, resumability, request limits, model-context exposure, and model-token consumption.
    Compare structured tool protocols with purpose-built binary HTTP transport.
45. Before implementing a non-idempotent mutation, produce a per-stage failure matrix covering
    side effects, cancellation, timeout, lost response, safe retry, reconciliation, and customer
    action. A reconciliation round trip must either change durable behavior or be explicitly
    justified as information-only.
46. Slice or component PASS results never aggregate into a final candidate PASS. Medium/high-risk
    work requires one independent review of the complete integrated base-to-head diff, including
    cross-slice behavior, host pipeline ordering, shared configuration, and unrelated sibling
    traffic.
47. Select mandatory specialists from the impact map. Authentication delegation requires an
    auth/authorization reviewer; non-idempotent distributed writes require a resilience reviewer;
    large payloads require performance/memory/timeout review; shared middleware requires a
    platform-impact reviewer; sensitive telemetry requires privacy review.
48. Reviewers must audit test-double realism and require applicable adversarial scenarios:
    maximum-size input, reordered or duplicate fields, malformed envelopes, slow headers/body,
    cancellation at each stage, unauthorized concurrency pressure, disabled-feature behavior,
    unrelated traffic, and failures immediately before and after each mutation boundary.
49. An unresolved security, authorization, identity, replay, or external-contract decision cannot
    be deferred only in documentation. The implementation must reject, remain unavailable, or fail
    startup until the unresolved path is safe; reviewers must verify that fail-closed behavior.
50. High-risk candidates require one fresh assumption-challenger review that initially ignores the
    proposed solution and tests whether the protocol, trust boundary, identity flow, synchronous
    lifecycle, persistence, and recovery model are appropriate. It complements rather than
    replaces spec-conformance review.
51. Before final PASS, maintain an acceptance-and-risk coverage table mapping every applicable
    invariant to code evidence, executable tests, reviewer evidence, and external contract or
    owner evidence. A row without direct evidence remains open and blocks completion.

## Lifecycle

Execute these gates in order. A later phase may expose missing earlier work; update the
task graph and return to the required gate rather than hiding the gap.

1. **Frame and reconcile**
   - Load applicable instructions, durable project context, repository state, open PRs,
     and existing design/spec artifacts.
   - Identify goals, non-goals, users, environments, constraints, external approvals,
     and measurable acceptance criteria.

2. **Research**
   - Split independent questions across research/explore agents.
   - Classify each question with the static compute matrix: exact lookup and bounded read-only
     exploration use the fast-read route; normal synthesis uses balanced; conflicting evidence or
     broad cross-cutting reasoning uses the applicable deep route.
   - Require file paths, symbols, commits, source links, observed behavior, uncertainty,
     and gaps between documentation and implementation.
   - Reconcile agent findings against current source before treating them as facts.

3. **High-level design**
   - Describe components, boundaries, data/control flow, identity, state, failure modes,
     rollout, compatibility, observability, and alternatives.
   - Record material decisions and unresolved owner choices.

4. **Specification**
   - Convert the design into testable functional and non-functional requirements,
     contracts, state transitions, validation rules, failure behavior, and acceptance
     criteria.
   - Treat repository SDD/spec conventions as the source of truth.

5. **Execution plan**
   - Build a dependency graph of deliverable-level tasks.
   - Assign non-overlapping file/component ownership.
   - Define each task's inputs, expected output, completion evidence, integration order,
     and focused validation.
   - Identify the critical path and the maximum safe parallel batch.
   - Define the worker registry entries, approval-efficient command batches, review risk tier,
     expected specialists, exact integration/review branch, and static compute profile from
     `references/compute-routing.md`.

6. **Parallel implementation**
   - Use isolated branches/worktrees when agents can produce independent commits.
   - Give every implementer a bounded objective and stop condition using the prompt in
     `references/agent-prompts.md`.
   - Define the first expected diff, test, or source-evidenced blocker for each implementer before
     launch. If that first artifact cannot be named, the task is not ready to delegate.
   - Launch with the exact model, effort, and context tier from the recorded compute profile.
     Agents may report insufficiency but must not choose a stronger model for themselves.
   - Start all ready independent tasks in one batch.
   - Keep shared-contract and integration work with the orchestrator unless ownership is
     unambiguous.

7. **Director oversight**
   - Start one persistent director agent when two or more implementation agents run.
   - Use the protocol in `references/director-protocol.md`.
   - Default check-in cadence is three minutes while agents are active.
   - Relay each consolidated director update to the user in the same orchestration turn.
   - If no new evidence exists at the reporting deadline, send a heartbeat stating the active
     phase, current critical-path work, last verified evidence, and what is still running.
   - At the first cadence, immediately take over a silent implementation agent with no artifact.
     Give at most one final cadence only when the agent supplied exact source/command evidence and
     freezes further exploration. Planning-only updates do not qualify.
   - When takeover is required, apply the stop-and-quarantine procedure immediately; never wait for
     a non-yielding agent before continuing from a clean worktree.
   - Transfer command-only validation and commit tails back to the orchestrator immediately.
   - Monitor registered external CLI side-chains and background commands through orchestrator-fed
     evidence; never assume a child director can discover sibling processes or sessions.
   - Reconcile every AI worker's actual model/effort/context against its registered static route
     and apply routing intervention in the same cadence when mismatched or below a risk floor.
   - Reconcile worker registry and todo state at each check-in before reporting progress.
   - Stop monitoring when no implementation/review agent remains active.

   Director monitoring and execution liveness are separate. Stopping the director does not pause
   the Maestro run. If direct orchestrator work remains ready after takeover, continue it
   immediately or arm a continuation schedule before yielding.

8. **Integration**
   - The orchestrator reviews agent evidence and integrates in dependency order.
   - Resolve conflicts semantically against the design/spec, never by blindly choosing
     one side.
   - Run focused validation after each risky integration boundary.
   - At coherent implementation slices and risky integration boundaries, run the smallest
     executable verification and an optional fresh-context checkpoint review. Record its exact
     head and findings as advisory evidence; do not treat it as the final candidate PASS.
   - Produce one integrated review candidate and record target base, head commit, changed files,
     changed public symbols/contracts, dependent callers/configuration, and validation evidence.

9. **Independent review**
   - Classify the candidate as low, medium, or high risk from the actual impact map. State why.
   - Run one broad adversarial review plus only the architecture, accessibility, database,
     performance, security, concurrency, persistence, or resilience specialists justified by the
     changed surfaces. Launch independent reviewers together.
   - Use fresh reviewer sessions that did not implement the code and follow the static review
     routes: low-risk broad review uses balanced-high; medium uses deep-fast plus
     clean-medium-gemini; high uses deep-high plus clean-high-gemini; critical security/privacy/data
     loss uses security-critical plus clean-critical-gemini. Model diversity supplements but never
     replaces source evidence and tests.
   - Require reviewers to inspect the exact base-to-head diff and affected callers/contracts, map
     acceptance criteria to evidence, and challenge retries, cancellation, stale state,
     concurrency, duplicate requests, partial failure, invalid configuration, reload/recovery,
     authorization, privacy, and rollout behavior where applicable.
   - Create a finding ledger. Every finding has an ID, severity, confidence, invariant/acceptance
     criterion, semantic fingerprint, bug class, introduced/pre-existing/uncertain origin,
     file/lines, concrete reproducer or failure scenario, owner, disposition, fix commit, and
     regression-test evidence. Track false positives and accepted risks explicitly.
   - Deduplicate against existing finding IDs/fingerprints. Adjudicate disputed, weakly evidenced,
     or broad/high-cost findings with one independent verifier before correction.
   - After fixes, run a targeted closure verifier for every finding. Then run a fresh clean-room
     full review for medium/high-risk candidates; this review searches for new defects and
     fix-induced regressions rather than receiving the old finding list first.
   - Count full review rounds and enforce the three-round convergence budget. After the broad and
     clean-room scans, use closure/delta review unless the changed impact map justifies another full
     scan. At the cap, reconsider the approach instead of continuing an agent review loop.
   - If the head changes after any PASS, invalidate the affected PASS and run the required delta or
     full review. A review is complete only for the recorded head commit.

10. **Final validation and handoff**
    - Run the smallest complete build/test/lint/security/e2e set proving the acceptance
      criteria, escalating to the full suite where required.
    - Reconcile documentation, deployment/configuration, migrations, telemetry, and
      rollback/runbook surfaces.
    - Verify all Maestro todos, branch state, untracked artifacts, and external blockers.
    - Reconcile planned versus actual compute routes, fallbacks/escalations, and available
      wall-time/token/cost evidence; route observations are evidence, not permission to self-tune.
    - When requested and approved, open a draft PR early enough for repository-native/Copilot review
      to run as a delivery gate. Reconcile valid PR findings, rerun affected validation/review, and
      record the final reviewed head.
    - Produce a concise final report and PR handoff with the review candidate SHA, finding ledger
      disposition, validation evidence, review-round/false-positive summary, proposed learning
      promotions, and unresolved external gates.

## Parallelism policy

- Parallelize independent research questions, non-overlapping components, test work, and
  documentation work.
- Do not split a single continuous trace across agents.
- Do not assign multiple agents to edit the same files or contract unless one is a
  read-only reviewer.
- Prefer a smaller set of high-value agents over speculative fan-out.
- Agents must not integrate each other's work. They return commits/evidence to the
  orchestrator.
- Launch independent agents together. Do not wait for one before starting another ready
  task.
- Launch independent applicable reviewers together. Do not serialize broad, security,
  architecture, accessibility, or other specialist reviews when they can inspect the same frozen
  candidate without shared writes.
- Share immutable build/test evidence with reviewers rather than rerunning identical commands in
  every reviewer. A reviewer may run a narrower reproducer or missing test when needed.
- Delegation is not fire-and-forget. Each agent must checkpoint with files, evidence, current
  command, blocker state, and an owned-file diff, focused test result, or source-evidenced blocker
  by the first director cadence.
- A worktree belongs to exactly one active writer. A stop request does not transfer ownership until
  the agent acknowledges and becomes idle/completed; otherwise quarantine it and use a fresh
  worktree.
- Do not delegate work that has collapsed to at most three known direct commands.

## Decision and safety gates

Stop and ask the user when:

- a design choice materially changes behavior, compatibility, cost, security, data
  ownership, or rollout;
- destructive migration or irreversible state change is proposed;
- external authorization, secrets, production access, or policy approval is required;
- the working tree is dirty and safe integration is ambiguous;
- evidence conflicts and no source is strictly authoritative.

Do not stop for routine implementation choices already determined by repository patterns
or the accepted specification.

## Completion definition

A Maestro run is complete only when:

- research, design, specification, and implementation agree;
- every acceptance criterion has direct evidence;
- integrated code passes required validation;
- independent review is closed or explicitly blocked;
- for medium/high-risk work, slice PASSes are superseded by a complete integrated-candidate review;
- every applicable acceptance-and-risk coverage row has code, test, reviewer and external
  contract/owner evidence, or is explicitly blocked with fail-closed behavior;
- deployment and rollback implications are documented;
- no unexpected pending/in-progress task remains;
- the final reviewed head matches the handoff/PR head, or every later delta has its required review
  and validation evidence;
- durable project knowledge is synchronized according to active memory instructions.
