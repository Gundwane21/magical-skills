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
9. Use independent reviewer agents that did not implement the reviewed code.
10. Do not declare completion until the integrated branch passes required validation and
    every review finding is fixed, explicitly accepted, or recorded as blocked.

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

6. **Parallel implementation**
   - Use isolated branches/worktrees when agents can produce independent commits.
   - Give every implementer a bounded objective and stop condition using the prompt in
     `references/agent-prompts.md`.
   - Start all ready independent tasks in one batch.
   - Keep shared-contract and integration work with the orchestrator unless ownership is
     unambiguous.

7. **Director oversight**
   - Start one persistent director agent when two or more implementation agents run.
   - Use the protocol in `references/director-protocol.md`.
   - Default check-in cadence is three minutes while agents are active.
   - Stop monitoring when no implementation/review agent remains active.

8. **Integration**
   - The orchestrator reviews agent evidence and integrates in dependency order.
   - Resolve conflicts semantically against the design/spec, never by blindly choosing
     one side.
   - Run focused validation after each risky integration boundary.

9. **Independent review**
   - Run an independent code review after integration.
   - Add architecture, accessibility, database, performance, or security review when the
     feature touches those concerns.
   - Convert findings into tracked tasks ranked by severity and confidence.
   - Require re-review of fixes for critical/high findings.

10. **Final validation and handoff**
    - Run the smallest complete build/test/lint/security/e2e set proving the acceptance
      criteria, escalating to the full suite where required.
    - Reconcile documentation, deployment/configuration, migrations, telemetry, and
      rollback/runbook surfaces.
    - Verify all Maestro todos, branch state, untracked artifacts, and external blockers.
    - Produce a concise final report and, when requested, a PR with validation evidence.

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
- deployment and rollback implications are documented;
- no unexpected pending/in-progress task remains;
- durable project knowledge is synchronized according to active memory instructions.
