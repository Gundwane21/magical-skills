# Maestro Static Compute Routing

Use this instruction-only router for every delegated AI task. It selects a fixed model,
reasoning effort, and context mode from the task's purpose and risk. It does not query model
availability, optimize from historical feedback, or change routes autonomously.

The model catalog below is pinned as of 2026-11-20. Update it deliberately when the runtime
catalog changes. Routes are optimized for output quality and development speed; cost is not a
selection criterion. Core Maestro safety, review, ownership, and evidence rules take precedence
over speed.

## Compute dimensions

Treat these as independent controls:

- **Model capability**: whether the task needs fast execution, balanced engineering ability, or
  deep judgment.
- **Reasoning effort**: how thoroughly the selected model should investigate and verify.
- **Context mode**:
  - `focused`: launch with `context_tier: default` and a bounded prompt containing only relevant
    sources;
  - `fresh`: launch a new agent/session rather than reusing an implementation or prior-review
    conversation;
  - `long`: launch with `context_tier: long_context` only when targeted retrieval cannot cover the
    required dependency surface.

When the matrix says `default` effort, omit the `reasoning_effort` override and use the model's
runtime default.

## Static routing matrix

| Route | Task or scenario | Model | Reasoning effort | Context |
|---|---|---|---|---|
| `orchestrator-interactive` | Main user-facing orchestrator | `gpt-6.2-sol` | default | focused |
| `fast-read` | Exact search, summary, routing, bounded read-only exploration | `claude-haiku-5.5` | `low` | focused |
| `fast-command` | Build/test/command runner | `gpt-6-luna` | `low` | focused |
| `fast-edit` | Mechanical edit or tightly scoped fix with strong verification | `claude-haiku-5.5` | `medium` | focused |
| `balanced` | Normal research, implementation, tests, or documentation | `claude-sonnet-5.5` | default | focused |
| `balanced-high` | Difficult bounded debugging/refactor or low-risk final review | `claude-sonnet-5.5` | `high` | focused; fresh for review |
| `local-review-high` | High-risk review of an exact frozen local git diff | `gpt-6.2-sol` | `high` | fresh, focused local worktree |
| `director-balanced` | Director monitoring and structured worker-state synthesis | `claude-sonnet-5.5` | `low` | focused on registry/task evidence |
| `deep-fast` | Architecture review, integration review, medium-risk broad review | `gpt-6.2-sol` | `medium` | focused; fresh for review |
| `deep-standard` | Architecture creation, conflicting evidence, complex cross-cutting work | `gpt-6.2-sol` | `high` | focused or curated broad context |
| `deep-high` | High-risk architecture or broad review | `gpt-6.2-sol` | `max` | fresh, focused impact map |
| `checkpoint-gemini` | Lightweight checkpoint review | `gemini-4-hydrogen` | `medium` | fresh and focused |
| `clean-medium-gemini` | Medium-risk clean-room review | `gemini-4-hydrogen` | `high` | fresh frozen diff |
| `clean-high-gemini` | High-risk clean-room review | `gemini-4-carbon` | `medium` | fresh impact map |
| `clean-critical-gemini` | Critical security/data-loss clean-room review | `gemini-4-carbon` | `high` | fresh complete impact map |
| `security-deep` | Security/privacy/data-loss review | `gpt-6.2-sol` | `high` | fresh complete impact map |
| `security-critical` | Credible critical security/privacy/data-loss review | `gpt-6.2-sol` | `max` | fresh complete impact map |
| `fix-verifier` | Finding closure or changed-line delta review | `claude-sonnet-5.5` | default | fresh, focused corrected diff |

## Suggested context budgets

Vendor window sizes are inconsistently reported and change between releases, so budgets are
prompt-size targets kept below the most conservative reported default window for the route's
model. Quality degrades before the window fills; scope and retrieve first.

| Route | Tier | Prompt budget | Long-context allowed |
|---|---|---|---|
| `orchestrator-interactive` | default | session-managed; compact at about 60% usage | session-level only |
| `fast-read` | default | ≤ 32k | no |
| `fast-command` | default | ≤ 16k | no |
| `fast-edit` | default | ≤ 48k | no |
| `balanced` | default | ≤ 100k | no |
| `balanced-high` | default | ≤ 128k | no |
| `local-review-high` | default | ≤ 200k | only if the impact map exceeds budget |
| `director-balanced` | default | ≤ 64k | no |
| `deep-fast` | default | ≤ 200k | no |
| `deep-standard` | default | ≤ 250k | yes, for a broad dependency surface |
| `deep-high` | default | ≤ 250k | yes, for a broad dependency surface |
| `checkpoint-gemini` | default | ≤ 64k | no |
| `clean-medium-gemini` | default | ≤ 64k | no; use `clean-high-gemini` if the diff is larger |
| `clean-high-gemini` | default | ≤ 200k | yes |
| `clean-critical-gemini` | default | ≤ 300k | yes |
| `security-deep` | default | ≤ 250k | yes |
| `security-critical` | default | ≤ 300k | yes |
| `fix-verifier` | default | ≤ 64k | no |

Rules:

- Launch with `context_tier: default` unless the supplied prompt would exceed the route budget
  after targeted retrieval.
- `gemini-4-hydrogen` has no long-context tier. A diff or impact map above its budget moves to
  the corresponding Carbon route rather than long context.
- Fast and `fix-verifier` routes never use long context. Oversized input means the task is
  `TASK_SCOPE_INVALID`: split it or move it up one capability step.
- Check the runtime catalog's `long_context` support for a model before requesting the tier.
  If a launch rejects the tier, retry once at `default` with a reduced, prioritized prompt.

## Agent launch mapping

Use the route's exact model even when a built-in agent type has a different default:

- `fast-read` -> read-only explore agent;
- `fast-command` -> task/command agent;
- implementation/research/architecture routes -> general-purpose agent with the applicable
  bounded prompt;
- `local-review-high` -> specialized code-review agent; supply the exact local worktree,
  base/head and changed-file impact map, prohibit nested delegation, and require name/stat
  discovery followed by bounded per-file diffs rather than one large diff payload;
- director -> persistent general-purpose read-only director prompt;
- high-risk assumption challenger -> fresh general-purpose agent using the `deep-high` route and
  the assumption-challenger prompt;
- non-security review, clean-room, adjudication, and fix-verification routes -> fresh
  general-purpose agent with the corresponding read-only reviewer prompt;
- security/privacy vulnerability review -> the specialized security-review agent first, using
  `security-deep` or `security-critical`, then the Gemini clean-room route when required.

Use a specialized code-review agent only when its enforced model matches the selected route.
Otherwise use the independent-review prompt with a fresh general-purpose agent so the route's
model remains explicit. Prompt-level read-only restrictions remain mandatory.

Per-call model routing requires the selected agent type to allow an explicit model override. If
the current `/subagents` configuration pins that agent type to a conflicting required model, mark
the launch `ROUTING_INVALID` and stop rather than silently using the wrong model. Resolve the
one-time configuration conflict through `/subagents` or a route-specific custom agent before the
run continues. A specialized security-review pin is acceptable only when it matches the
`security-*` route model.

Use the opposite-family deep route for finding adjudication:

- a GPT-originated finding is adjudicated with `gemini-4-carbon` at `medium`;
- a Gemini- or Claude-originated finding is adjudicated with `gpt-6.2-sol` at `medium`;
- raise the adjudicator to `high` only when the evidence remains conflicting and the correction is
  high risk or broad.

## Classification rules

Apply the first matching rule, then apply risk floors:

1. Exact lookup, summary, or bounded read-only exploration -> `fast-read`.
2. Deterministic command execution -> `fast-command`.
3. Mechanical edit with explicit files, behavior, and executable verification -> `fast-edit`.
4. Ordinary bounded engineering work -> `balanced`.
5. Difficult but still bounded work -> `balanced-high`.
6. Review of an existing architecture or medium-risk integrated candidate -> `deep-fast`.
7. High-risk review whose exact frozen candidate is available in a local worktree ->
   `local-review-high`.
8. Architecture creation, conflicting evidence, broad cross-cutting reasoning, or weak
   verification -> `deep-standard`.
9. High-risk architecture or non-local system review -> `deep-high`.
10. Review routes follow the candidate-risk matrix below and override generic task routes.

Do not raise capability solely because the repository or context window is large. Retrieve and
scope first.

## Review risk floors

| Candidate risk | Broad review | Clean-room review | Additional rule |
|---|---|---|---|
| Low | `balanced-high` | Not required unless impact changes | Final delta uses `fix-verifier` |
| Medium | `deep-fast` | `clean-medium-gemini` | Applicable specialists only |
| High | `local-review-high` for an exact local frozen diff; `deep-high` for architecture/system review | `clean-high-gemini` | Use fresh sessions and exact impact map |
| Critical security/privacy/data loss | `security-critical` | `clean-critical-gemini` | Containment may proceed while evidence closes |

Cross-model diversity supplements source evidence, executable validation, and risk-appropriate
specialists. It never replaces them.

## Context rules

- `focused` is the default even when a model supports a very large context window.
- Provide exact design/spec sections, base/head commits, diff, impacted callers/contracts, and
  reusable validation evidence instead of an entire session transcript.
- Use `fresh` for checkpoint reviewers, independent reviewers, clean-room reviewers, adjudicators,
  and fix verifiers. Do not use `write_agent` to repurpose an implementer as a reviewer.
- Use `long_context` only for cross-repository architecture, a genuinely broad dependency graph,
  or a candidate whose complete impact map cannot fit in a focused prompt after targeted
  retrieval.
- Compaction or a new session is preferred when accumulated history is unrelated to the next task.

## Insufficiency diagnosis and one-step escalation

Agents report one of these signals when they cannot complete the bounded task:

- `CONTEXT_INSUFFICIENT`: required evidence or dependency surface is missing;
- `EFFORT_INSUFFICIENT`: the assigned model did not investigate or verify enough;
- `CAPABILITY_INSUFFICIENT`: the model investigated the supplied evidence but could not resolve
  the reasoning or judgment;
- `TASK_SCOPE_INVALID`: the task is not actually bounded enough for delegation.

The orchestrator responds:

1. `CONTEXT_INSUFFICIENT` -> retrieve the missing evidence; use `long_context` only when targeted
   expansion is insufficient.
2. `EFFORT_INSUFFICIENT` -> raise the same model by one supported effort level.
3. `CAPABILITY_INSUFFICIENT` -> move one capability step:
   - fast route -> `balanced`;
   - `balanced`/`balanced-high` -> `deep-fast`;
   - `deep-fast` -> `deep-standard`;
   - `deep-standard` -> `deep-high`;
   - Gemini Hydrogen review -> the corresponding Gemini Carbon route.
4. `TASK_SCOPE_INVALID` -> return to research/design, split the task, or keep it with the
   orchestrator.

Permit at most one compute escalation for one delegated attempt. If the escalated attempt still
cannot satisfy the evidence bar, stop and reconsider the task, design, or validation strategy.
Do not continue a model/effort escalation loop.

## Static launch-failure fallbacks

Do not probe or enumerate the live catalog. If the selected model launch fails specifically
because the model is unsupported or unavailable, retry once:

| Selected model | One fallback |
|---|---|
| `claude-haiku-5.5` | `claude-sonnet-5.5` |
| `gpt-6-luna` | `gpt-6.1-terra` |
| `claude-sonnet-5.5` | `claude-sonnet-5` |
| `gpt-6.2-sol` | `gpt-6.1-sol` |
| `gpt-6.1-sol` | `gpt-5.6-sol` |
| `gemini-4-hydrogen` or `gemini-4-carbon` | `gemini-4-argon` |
| `gemini-4-argon` | `claude-sonnet-5.5` |

Keep the original effort and context mode when supported by the fallback. If the fallback rejects
the requested effort/context or also fails, stop and report the exact launch error. Do not invent
another model.

## Required routing evidence

Before launch, record in the execution plan and worker registry:

- route ID;
- task class and risk;
- exact model;
- reasoning effort or runtime default;
- focused/fresh/long context mode;
- why the route satisfies the task and risk floor;
- observable escalation trigger;
- fallback, if used.

At handoff, record the actual route, any fallback/escalation, completion evidence, and available
wall-time/token/cost data. These records are observational only until a future control plane adds
persistent eval-driven optimization.
