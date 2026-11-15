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

## 3. High-level design

Include:

- component and trust boundaries;
- request, data, state, and identity flows;
- API/events/payload contracts;
- persistence and concurrency model;
- retries, idempotency, cancellation, and ambiguous outcomes;
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
- acceptance tests.

Avoid aspirational language such as "support," "robust," or "secure" without measurable
behavior.

## 5. Execution plan

Each task must state:

- bounded objective;
- owned files/components;
- dependencies;
- expected artifacts or behavior;
- focused validation;
- stop/handoff condition.

Create a file-ownership matrix before parallel implementation. Shared contracts,
composition roots, central configuration, and migrations should usually integrate through
the orchestrator.

## 6. Implementation evidence

Each implementer returns:

- branch/worktree and head commit;
- files changed;
- behavior implemented;
- tests/builds run and exact results;
- assumptions and unresolved risks;
- integration notes;
- confirmation that no unrelated changes were included.

## 7. Review evidence

Review findings must include:

- severity and confidence;
- file and line range;
- concrete failure scenario;
- why existing tests do not prevent it;
- minimal recommended correction.

Track false positives and accepted risks explicitly. Do not silently drop findings.

## 8. Completion report

Lead with:

- delivered capability;
- branch/PR/deployment state;
- validation results;
- unresolved external gates.

Then summarize meaningful architecture and behavior changes. Do not narrate the entire
process.
