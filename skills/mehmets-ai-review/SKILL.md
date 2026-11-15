---
name: "mehmets-ai-review"
description: "Run a high-signal, multi-model review of a branch or pull request before it merges. Use whenever someone asks to review a branch or a PR, check whether a branch is ready for a PR, self-review their changes, or run a deep, multi-model, or parallel review. Applies the same quality standards to every branch-to-PR flow so any colleague can review a PR consistently."
---

# Review a branch or pull request (multi-model)

Structured workflow for AI-assisted review of branches and pull requests in this repository. The goal is a consistent, high-signal review that any colleague can run before a change merges, so quality does not depend on who is reviewing.

Synthesised from the Microsoft Code-with-Engineering Playbook, Microsoft-internal agentic reviewer prompts (Loop Bebop, Windows ES, Azure Core), and Google's eng-practices. See [References](#references) for the public canonical sources (the internal prompts are not publicly linkable).

---

## When to invoke

Apply this workflow in these situations:

1. **Pre-PR self-review.** Before someone raises a PR for their own branch (phrasings include "review my branch", "ready for PR?", "self-review", "critique my changes"), or immediately before a create-PR flow runs.
2. **Reviewing someone else's PR.** Any time you are asked to review an existing PR (by PR ID, branch name, or Azure DevOps link) authored by another person.
3. **Reviewing documentation or architecture changes.** Any time the branch touches `docs/**`, any `ReadMe.md`, a wiki page, or any design or architecture document.
4. **On explicit request.** Any time the request mentions a "deep review", "multi-model review", or "parallel review".

If the request is ambiguous (for example "look at the branch"), ask whether the user wants a review or a summary before running this workflow.

If the request is to actually post comments, votes, or PR-description updates to Azure DevOps, draft locally first and require explicit per-call approval before invoking any `ado-repo_*` write tool. Never auto-post.

---

## Operating principles

These apply to every reviewer (single-model or any of the parallel models) on top of the dimension prompts.

1. **Two-pass agentic review** (Loop Bebop pattern):
   - **Pass 1, Reviewer:** walk the diff and the surrounding code; surface findings using the dimensions below.
   - **Pass 2, Auditor:** re-read your own findings against the diff a second time. Ask "what did I miss?" Focus on cross-file impact, dead branches, and silent-failure paths the reviewer pass tends to overlook.
2. **Signal over noise.** Only surface findings that would block, change, or meaningfully improve the PR. Do not comment on style, formatting, or matters the linter handles. (Google: "approve once it definitively improves overall code health, not when it is perfect".)
3. **Distinguish blocking from non-blocking.** Every finding you intend to post as an ADO comment must use the Conventional Comments form defined in "Conventional Comments and commit messages" below: a bracket label (`[issue]`, `[suggestion]`, `[question]`, `[nitpick]`, and so on) with an explicit decoration when it is not the label's default (`[blocking]`, `[non-blocking]`, `[if-minor]`, `[security]`). In-chat findings may use a lighter shorthand, but anything destined for ADO must carry the bracket label.
4. **Cite the line and file** in every finding. Use absolute paths. Quote the offending fragment.
5. **Praise the good things** at least once when warranted (Google "Good Things"). Reviewing is a relationship, not a gate.
6. **Challenge architecture when warranted.** Reviewers are authorized to question any architectural decision, even one documented as accepted. State the flaw, the risk, the prior art that contradicts it (if any), and a concrete alternative. The author decides what to do with the feedback.

---

## Review scope (what to examine)

A full review covers all of the following. Do not limit the review to code diffs only.

1. **Code changes.** All staged and unstaged changes and commits between the base and head. Include the full diff, not just touched lines, when context is needed.
2. **Tests.** Unit, integration, end-to-end. Coverage, edge cases, brittleness, determinism.
3. **Configuration and infrastructure.** ScopeBindings, ConfigSpecs, parameter files, Liquid templates, YAML pipelines, Helm charts, Bicep, ARM, EV2 rollout specs.
4. **Documentation.** Code comments, XML doc and JSDoc, in-repo docs under `docs/**`, ReadMe files, and any `*.md` touched by the branch.
5. **Architecture decisions.** Any architectural or design decision surfaced by the branch (explicit in a doc, or implicit in the code). Challenge decisions you believe are unsound, outdated, or inconsistent with the repo's prior art. Silence is not acceptable when a decision is questionable: raise it.
6. **PR description and change doc.** If a bug or design doc for this change exists, cross-check it against the code: does the description match what actually changed? Are there scope drifts, missing mitigations, or outdated sections?
7. **Consistency with prior art.** Cross-reference against the canonical architecture and design docs for whatever feature area the branch touches, and against comparable prior PRs.

---

## Default workflow: 3-model parallel review

For any branch or PR review, use three different strong models in parallel to reduce wall-clock time and increase coverage diversity. Do not serialize unless the user explicitly asks for a single-model review.

### Step 0. Resolve the review target and set up context (main agent, once)

Run this before gathering the diff. It establishes which branch is under review, isolates a named branch into its own worktree, and pulls in the work-item and PR context that grounds the review. Cache the following variables for later steps: `REVIEW_BRANCH` (the branch being reviewed), `REVIEW_REF` (the ref reviewers diff against; `HEAD`, evaluated inside the worktree when one is created), `BASE_REF` (the branch this change merges into: the PR target when a PR exists, otherwise the repo default such as `origin/main`), and `WORKTREE_PATH` (empty when reviewing the current branch).

1. **Determine the target branch.**
   - If the user gave a PR ID or PR link, resolve the PR first (search the deferred ADO catalogue for `repo_get_pull_request_by_id`): set `REVIEW_BRANCH` from its `sourceRefName` and `BASE_REF` from its `targetRefName`, then continue with the fetch and worktree substeps below. This takes precedence over the current checkout, so a "review PR 123" request never reviews the wrong branch.
   - If the user named a branch (by branch name or Azure DevOps link) and it differs from the currently checked-out branch, that named branch is the target. Set `REVIEW_BRANCH` to it.
   - If no branch was named, or the named branch is already checked out, the target is the current branch: `REVIEW_BRANCH = git rev-parse --abbrev-ref HEAD`, `REVIEW_REF = HEAD`, `WORKTREE_PATH` empty. Skip to substep 4. (If that command returns the literal `HEAD`, the checkout is itself detached: use the commit SHA as `REVIEW_REF` and skip the PR-by-branch lookup in substep 5.)

2. **Fetch the named branch.** Normalise the name first (strip any leading `refs/heads/`), then `git fetch origin <REVIEW_BRANCH>:refs/remotes/origin/<REVIEW_BRANCH>` so the local `origin/<REVIEW_BRANCH>` ref is current. Fetching never disturbs the developer's working tree.

3. **Create a worktree for the named branch (do not switch the developer's working tree).** Reviewing a named branch must never touch the developer's current checkout or uncommitted work, so isolate it in a dedicated worktree:
   - Sanitise the branch name for a folder (replace `/` and other path-unsafe characters with `-`), for example `alias/36227935/fix-x` becomes `alias-36227935-fix-x`.
   - `WORKTREE_PATH = ..\ExtensionStudio-review-<sanitized-branch>` (a sibling of the repo root; worktrees cannot live inside `.git`). On Windows this repo has a deep source tree, so keep the path short: prefer the task ID or a short hash over the full branch name if the sanitised name is long, and enable long paths once with `git config --global core.longpaths true`. If the path already exists, first confirm it is a registered review worktree with `git worktree list --porcelain` (never `reset --hard` an unrelated directory): if it is registered and clean (`git -C <WORKTREE_PATH> status --porcelain` is empty), reuse it (`git -C <WORKTREE_PATH> fetch` then `git -C <WORKTREE_PATH> reset --hard origin/<REVIEW_BRANCH>`); otherwise remove it with `git worktree remove <WORKTREE_PATH>` (run `git worktree prune` to clear stale metadata) or pick a unique suffix.
   - `git worktree add --detach <WORKTREE_PATH> origin/<REVIEW_BRANCH>` to check the branch tip out there in detached HEAD (no local branch needed for a read-only review). `REVIEW_REF` stays `HEAD`, evaluated as `git -C <WORKTREE_PATH> ...`; run all later git commands with that `-C` prefix.
   - **Inform the developer explicitly**, in the response, of the worktree name and path, that their current working tree is untouched, and how to clean it up after the review: `git worktree remove <WORKTREE_PATH>`.

4. **Establish work-item context (optional, never a blocker).**
   - Try to extract a work-item ID from the branch name (branches often follow an `<alias>/<taskID>/<desc>` convention that puts a numeric task ID in the second segment, but conventions vary by author). If found, load it with the ADO work-item tool (search the tool catalogue for `wit_get_work_item` first, since ADO tools are deferred) to pull the title and description as context.
   - If no work-item ID is discoverable from the branch name or the PR, ask the user for one with `ask_user`, framed as optional context. **Proceed with the review regardless of the answer**: a missing work item is not a reason to block or delay the review.

5. **Pull the PR description for history and context, and set `BASE_REF`.**
   - Look up any open PR whose source branch is `REVIEW_BRANCH` (search the deferred ADO catalogue for `repo_list_pull_requests_by_repo_or_project`; scope the call with the repository name or `repositoryId`, the `project`, `status: "Active"`, and `sourceRefName: "refs/heads/<REVIEW_BRANCH>"`). When the user gave a PR link or ID, this was already resolved in substep 1 via `repo_get_pull_request_by_id`.
   - If a PR exists, set `BASE_REF` from the PR's `targetRefName` (for example `refs/heads/dev` becomes `origin/dev`), read its description, and pass it to the reviewers as background history: what the author says changed, prior review rounds, and any stated scope or risk. Cross-check it against the diff in the Documentation and readability dimension.
   - If no PR exists, treat this as a pre-PR self-review, note that in the summary, and resolve `BASE_REF` in substep 6.

6. **Detect merge conflicts against the base branch (warn, do not block).**
   - Resolve `BASE_REF` if not already set from the PR target: use an explicit user-supplied base, else the repo default from `git symbolic-ref refs/remotes/origin/HEAD --short` (commonly `origin/main`; may be `origin/dev` in repos that integrate on `dev`). If that lookup errors because `origin/HEAD` is unset, run `git remote set-head origin -a` once or default to `origin/main`. Refresh the base: `git fetch origin <BASE_REF without the origin/ prefix>`.
   - Test the merge non-destructively without altering any working tree (requires Git 2.38+; on older clients skip this check and note it): `git merge-tree --write-tree --name-only <BASE_REF> <MERGE_HEAD>`, where `<MERGE_HEAD>` is `origin/<REVIEW_BRANCH>` for a named-branch review (worktree-independent, so no `-C` prefix is needed) and `HEAD` for a current-branch review. Exit `0` means clean, exit `1` means conflicts (the `--name-only` output then lists the conflicting files), exit `>1` means the command itself errored.
   - If conflicts exist, **warn the developer prominently** at the top of the review: state that `REVIEW_BRANCH` has merge conflicts with `<BASE_REF>`, list the conflicting files from the `--name-only` output, and recommend a rebase or merge before the PR proceeds. This is a warning, not a hard stop: continue the review.

### Step 1. Gather context (main agent, once)

Using the variables resolved in Step 0 (`REVIEW_REF` is `HEAD`, evaluated inside the worktree when one exists; substitute your resolved `BASE_REF` for `origin/main`; and prefix every git command with `git -C <WORKTREE_PATH>` when a worktree was created), collect and cache:

- `git --no-pager log --no-merges <BASE_REF>..<REVIEW_REF> --stat` for the full commit list and per-commit file stats.
- `git --no-pager diff --stat <BASE_REF>...<REVIEW_REF>` for the aggregate diff stat.
- `git --no-pager diff <BASE_REF>...<REVIEW_REF>` for the full diff. If large, save it to a file in your git-ignored review workspace (the `reviews/` scratch area from Step 5) and reference it by path; treat the file as disposable.
- **For a current-branch pre-PR self-review** (no worktree), also capture uncommitted work so it is not missed: `git status --short`, `git --no-pager diff --cached` (staged), and `git --no-pager diff` (unstaged). Commit-only diffs skip staged and unstaged changes. List brand-new files with `git ls-files --others --exclude-standard` and read each one's content, since `git status --short` shows untracked files by name only, not their contents.
- The work-item title and description (from Step 0), the PR description (from Step 0), any bug or design doc for the branch, any architecture docs the branch touches, and the canonical architecture docs relevant to the feature area.
- Any merge-conflict warning from Step 0, so it can be surfaced in the final summary.
- The target review dimensions (see "Review dimensions" below). Default to a combined full review unless the user asks for a specific dimension.

### Step 2. Launch three parallel `rubber-duck` agents

Invoke the `task` tool three times **in the same response** (parallel), each with:

- `agent_type: "rubber-duck"` (the built-in reviewer agent; if a given environment does not expose it, fall back to `general-purpose`).
- `mode: "background"` so the main agent can continue while they run.
- A distinct `model` override so the three reviewers run on three different models. Do not pin specific model versions here: they change often and would force a skill edit every time. Instead, at review time read the current list from the `task` tool's `model` parameter description and pick one model from each of three different vendors (for example Claude, GPT, and Gemini), preferring each vendor's flagship or most-capable tier over its mini or flash tier, at a high reasoning-effort setting. When a vendor offers several same-generation variants with no clear ranking, pick any one. Never use the same model twice in one review. Degrade gracefully: if fewer than three vendors are available, use the strongest available models even if two share a vendor; if the environment exposes no `model` override, run all three reviewers on the default model and note that in the review record.
- A comprehensive prompt containing:
  - The branch name (`REVIEW_BRANCH`), base (`BASE_REF`, usually `origin/main`), and the head commit SHA of `REVIEW_REF`.
  - The full diff (or path to the saved diff file) and the list of changed files.
  - The PR description (from Step 0) as history and background, plus any change-doc content.
  - The work-item title and description (from Step 0) when available, for intent context.
  - Any merge-conflict warning from Step 0.
  - Links or paths to relevant architecture docs the reviewer should load.
  - The selected review dimension prompt(s) from this file, pasted verbatim.
  - Explicit instruction: "You are authorized to challenge architectural decisions if you believe they are incorrect, inconsistent with prior art, or create undue risk. State the risk and propose an alternative."
  - The required output format (see Step 4).

Each reviewer runs in its own context window, so give each one the complete context it needs to review independently without relying on prior conversation.

Order each reviewer prompt for long context (Anthropic long-context guidance): put the longform inputs first (the full diff or its saved file path, the changed-file list, the PR description, and the work-item context), then the dimension prompts and the challenge-architecture instruction, and put the specific review question and the required output format last. Placing the query after the long inputs measurably improves response quality. Instruct each reviewer to first quote the exact file-and-line fragments it will comment on, then write its findings, so every finding is grounded in the diff rather than in a paraphrase.

### Step 3. Collect and triage (main agent)

When all three reviewers complete (notifications fire), read their responses with `read_agent`. Then:

1. **Deduplicate.** Merge findings that are the same issue phrased differently.
2. **Classify agreement.** For each finding, note how many reviewers raised it (1/3, 2/3, 3/3). Findings raised by two or more reviewers are higher priority.
3. **Resolve contradictions.** When reviewers disagree, present both positions to the user and state your recommendation with reasoning.
4. **Exercise judgment.** Adopt findings that clearly prevent bugs, security issues, or regressions. Set aside findings that would significantly complicate the implementation without clear benefit. Never silently drop a finding without justification.

### Step 4. Present the consolidated review

Deliver to the user:

1. **Summary.** One-paragraph overall verdict: ship, ship-with-fixes, wait-for-author, or block. Mention any architectural concerns raised.
2. **Review setup note.** When a worktree was created in Step 0, state its path and the `git worktree remove <WORKTREE_PATH>` cleanup command. When Step 0 detected a merge conflict against the integration branch, surface that warning prominently here with the list of conflicting files.
3. **Findings table.** Columns (matching the Step 5 template order): `#`, `Severity` (Critical/High/Medium/Low), `Reviewers` (for example 3/3), `Dimension` (from the list below), `Finding`, `Suggested fix`, `Blocker?` (Yes/No). For any finding you intend to post to ADO, phrase the `Finding` in Conventional Comments form so Step 5 and the thread reply can reuse it verbatim.
4. **Architecture challenges (if any).** A separate subsection listing any architectural decisions the reviewers questioned, with the stated risk and proposed alternatives.
5. **Model-by-model notes.** Brief per-model highlights only for findings unique to one reviewer. Do not repeat the consolidated table.
6. **Next actions.** A concrete list: what to fix before PR, what to defer to follow-ups, what to confirm with the user.

### Step 5. Persist the review as a canonical review doc (main agent, always)

After presenting the review in chat, always write the consolidated review to a markdown file so it survives the session and can drive follow-up (thread replies, re-review, hand-off). This applies to every review, whether the branch is the author's own or a colleague's.

**Location and lifecycle.** Save the doc in your local, git-ignored review workspace (for example a `reviews/` folder at the repo root that `git check-ignore` confirms is ignored), never in a tracked path, and never commit it. Use a two-stage lifecycle: an `in-progress` folder while the review or its thread replies are open, and a `completed` folder once the review is closed out. Because the doc lives in an untracked local workspace that teammates cannot see, never reference its path in any PR description, PR thread, commit message, work item, or Teams message; copy the relevant content inline instead.

**Naming.** Pick the first identifier that exists, lower-kebab-case, `.md`:
- PR under review: `pr-<prId>-<author>-<slug>.md` (for example `pr-15633666-daoyilu-generalize-core-kv.md`).
- No PR yet but a work item is known: `<taskId>-<author>-<slug>.md`.
- Neither: `<author>-<slug>.md` derived from the branch name.

`<author>` is the branch author's alias; `<slug>` is a short declarative summary from the branch name.

**Canonical template.** Populate these sections in order (they mirror the Step 4 output plus the Step 0 context, so most content is already gathered):

<pre style="white-space: pre-wrap; word-wrap: break-word;">
# PR Review: &lt;title&gt;

Consulting the `mehmets-ai-review` skill (workflow: 3-model parallel, models: &lt;trio&gt;, dimensions: &lt;list&gt;).

## Metadata
| Field | Value |
|-------|-------|
| PR | &lt;id + link, or "none (pre-PR self-review)"&gt; |
| Title | &lt;pr/branch title&gt; |
| Author | &lt;name (alias)&gt; |
| Branch | `&lt;REVIEW_BRANCH&gt;` -> `&lt;BASE_REF&gt;` |
| HEAD SHA | &lt;REVIEW_REF sha&gt; |
| Work item | &lt;id + title, or "not provided"&gt; |
| Worktree | `&lt;WORKTREE_PATH&gt;` or "n/a (current branch)" |
| Merge conflicts vs base | &lt;none / list of conflicting files&gt; |
| Reviewed | &lt;yyyy-mm-dd&gt; |
| Verdict | ship / ship-with-fixes / wait-for-author / block |

## Scope
&lt;diff --stat and commit list&gt;

## What the PR does
&lt;short factual summary&gt;

## Cross-check vs prior art / architecture docs
&lt;how the change lines up with the canonical docs for the feature area&gt;

## Consolidated findings
| # | Severity | Reviewers | Dimension | Finding (+ Conventional Comment) | Suggested fix | Blocker? |
|---|----------|-----------|-----------|----------------------------------|---------------|----------|

## Architectural challenges (if any)
| # | Reviewer(s) | Decision questioned | Risk | Proposed alternative |
|---|-------------|---------------------|------|----------------------|

## Validated claims (no action needed)
&lt;checked items&gt;

## Per-model unique findings
&lt;items raised by only one reviewer&gt;

## Disagreements
&lt;where reviewers conflicted + main-agent recommendation&gt;

## Suggested next actions
&lt;fix-before-PR / defer / confirm-with-user&gt;

## Files reviewed for context (not in diff)
&lt;paths&gt;

## Drafted replies to open threads
&lt;one ready-to-paste block per thread in Conventional Comments form (see "Conventional Comments and commit messages"); "Do not post to ADO without explicit per-call approval"&gt;
</pre>

Omit a section only when it is genuinely empty (for example no open threads, no architectural challenges). Keep the Conventional Comments format inside findings so drafted replies can be lifted straight into ADO after approval.

---

## Review dimensions

Use individually when the user asks for a targeted review, or combine for comprehensive reviews. These prompts are passed verbatim to each reviewer. The bullet checklists are reference material the reviewer walks through; they are not exhaustive but cover the failure modes most often missed in this codebase.

### General

> Review the pull request comprehensively as a senior principal or staff engineer.
>
> Focus on:
> - Correctness: does the logic do what the PR claims?
> - Maintainability: will the next engineer understand this in 6 months?
> - Regression risk: could this break existing functionality?
> - Unclear assumptions: are there implicit dependencies or undocumented constraints?
> - Rollout concerns: is this safe to deploy incrementally?
>
> Prioritize concrete findings over style commentary. For each issue, state the risk and suggest a fix.

### Release risk

> Review the pull request for deployment and release safety.
>
> Focus on:
> - Migration and rollback safety: can this be reverted without data loss?
> - Operational blast radius: how many users or services are affected if this fails?
> - Observability gaps: are failures detectable via logs, metrics, or alerts?
> - Backwards compatibility: do existing clients or consumers break?
> - Deployment sequencing risks: does this depend on other changes landing first?
>
> Highlight any reason the change should not ship immediately.

### Security

> Review the pull request with a security mindset.
>
> Focus on:
> - Authentication and authorization flaws: missing or bypassable checks.
> - Secret handling: hardcoded secrets, secrets in logs, insecure storage.
> - Unsafe input handling: injection, deserialization, path traversal.
> - Privilege escalation: can a lower-privilege user gain access?
> - Logging or disclosure: PII in logs, verbose error messages.
>
> Call out exploit paths and suggest mitigations where possible. Reference OWASP Top 10 categories if applicable.

### Testing

> Review the pull request for testing sufficiency.
>
> Focus on:
> - Missing unit coverage: are new code paths tested?
> - Missing integration coverage: are cross-component interactions validated?
> - Edge cases and failure modes: null inputs, boundary values, timeouts, concurrency.
> - Brittle or low-signal tests: tests that pass or fail for the wrong reasons.
> - Validation gaps before merge: what could break that is not tested?
>
> Prefer practical tests over exhaustive boilerplate. Flag any untested behavior that carries production risk.

### Performance

> Review the pull request for performance impact.
>
> Focus on:
> - Algorithm complexity: O(n^2) loops, unnecessary allocations, repeated work.
> - Database and network calls: N+1 queries, missing caching, unbounded result sets.
> - Memory pressure: large object graphs, missing disposal, string concatenation in loops.
> - Concurrency: thread safety, deadlock potential, async/await misuse.
> - Cold start and hot path: does this affect latency-sensitive code paths?
>
> Quantify impact where possible (for example, "this adds an API call per item in a list of up to 10K items").

### Architecture

> Review the pull request for architectural soundness. You are explicitly authorized to challenge architectural decisions if you believe they are unsound, inconsistent with prior art, or create undue long-term risk. State the decision, the risk, the prior art that contradicts it (if any), and a concrete alternative.
>
> Focus on:
> - Separation of concerns: is business logic mixed with infrastructure or UI?
> - Dependency direction: do dependencies flow toward abstractions or concrete implementations?
> - API design: are contracts clear, versioned, and backward-compatible?
> - Coupling: does this change create tight coupling between unrelated modules?
> - Scalability: will this design hold at 10x the current load?
> - Consistency with the repo's architecture and design docs for whatever feature area the branch touches.
>
> Flag architectural drift from established patterns. Suggest alternatives if the current approach introduces technical debt.

### Correctness (deep checklist)

In addition to the General prompt above, walk the following when reviewing C# or TypeScript changes in this repo:

- Off-by-one, integer overflow, signed vs unsigned, floating-point equality.
- Null, empty, and missing-config handling. **Distinguish** missing key vs present-but-empty vs present-but-malformed. Each should fail loudly with a message that names the key and the value.
- Time-zone, DST, leap-year, and leap-second behaviour for any datetime arithmetic. Use `CultureInfo.InvariantCulture` for parsing and formatting non-display values.
- Concurrency: shared mutable state, async/await deadlocks (`.Result` or `.Wait()` on UI or sync contexts), missing `ConfigureAwait`, race windows around lazy initialisation.
- Resource lifetime: `using` and `IAsyncDisposable`, HttpClient reuse, stream disposal, cancellation-token propagation.
- Exception handling: catch what you can act on; never swallow; preserve the stack via `throw;` not `throw ex;`. Wrap external failures with context (which key, which URL).

### Error handling

> Review the pull request for error handling completeness.
>
> Focus on:
> - Silent failures: errors caught and swallowed without logging or rethrowing.
> - Partial state: operations that can leave data inconsistent on failure.
> - User-facing errors: are error messages actionable without leaking internals?
> - Retry logic: is it idempotent? Does it have backoff and circuit-breaking?
> - Resource cleanup: are connections, streams, and locks released on all paths (including exceptions)?
>
> For each gap, describe the failure scenario and its user or system impact.

### Configuration and infrastructure

> Review the pull request for configuration and infrastructure correctness.
>
> Focus on:
> - Environment-specific values: are dev, staging, and prod configs properly separated?
> - Feature flags: are new features gated? Can they be disabled without redeployment?
> - Resource naming: do names follow project conventions and avoid collisions?
> - Template correctness: are generated artifacts (ARM, Bicep, Liquid, YAML) structurally valid?
> - Idempotency: can this change be applied multiple times safely?
> - Feature-area isolation and naming gating: where an area gates resource names or behaviour behind a flag (for example MPAC's `UseLegacyNaming`), confirm the gating is respected and cross-environment or cross-cloud isolation is preserved.
> - EV2: run `tools\ev2\Validate-EV2Infra.ps1` from a clean working tree; confirm rollout spec, scope-binding, region list, and ATM or DNS implications.
>
> Flag any configuration that could cause a deployment failure or environment leak.

### Observability

> Review the pull request for observability and supportability.
>
> Focus on:
> - Telemetry: new code paths emit metrics, traces, and structured logs at the right level. No `Console.WriteLine`. No `string.Format` interpolation that defeats structured logging.
> - Correlation IDs propagated across async boundaries and out-of-process calls.
> - Health endpoints reflect any new dependency.
> - Alerts and dashboards updated when new SLIs or SLOs are introduced.
> - Failure scenarios: can on-call diagnose a production failure with only the new logs and metrics?
>
> Flag any new code path that would be invisible to on-call.

### Style and consistency

> Review the pull request for stylistic consistency, but only flag deviations that hurt readability or maintainability.
>
> Focus on:
> - Match the surrounding code's idioms. The existing code is the style guide.
> - Naming: pronounceable, searchable, no Hungarian, no abbreviations except the established project ones (PortalRP, MPAC, EV2, and similar).
> - No dead code. No commented-out blocks. No TODOs without a tracked work item.
> - Punctuation: do not use a hyphen as a sentence dash in prose or code comments; use a colon or restructure.
>
> Do NOT flag formatter-handled issues. Prefix all findings with `nit:` unless they materially impair readability.

### Documentation and readability

> Review the pull request for documentation and code readability, including standalone documentation files changed or added by this branch.
>
> Focus on:
> - Naming: are variables, functions, and types named clearly and consistently?
> - Comments: are complex algorithms or non-obvious decisions explained?
> - Public API documentation: are new public interfaces documented?
> - PR description: does it explain the "why" not just the "what"? Does it match the actual diff?
> - Change-doc accuracy: if a bug or design doc for this change exists, does it still reflect reality?
> - Architecture-doc accuracy: are any architecture or design docs updated or contradicted by this change?
> - Dead code and stale docs: commented-out blocks, unused imports, outdated diagrams, stale examples.

---

## Conventional Comments and commit messages (ADO replies and recommended commits)

Every PR review comment, every reply to an ADO thread, and every commit message or PR title you draft in a review must follow the repo's bracketed conventions below. This is self-contained here on purpose: apply it whether you are the **reviewer** commenting on a colleague's branch, or the **author** replying to comments on your own branch.

### Conventional Comments (all review comments and thread replies)

Format, one separable concern per comment:

<pre style="white-space: pre-wrap; word-wrap: break-word;">
[&lt;label&gt;][&lt;decoration&gt;] &lt;subject&gt;

[optional discussion, code snippet, or suggested diff]
</pre>

- **`[<label>]`** (required): one of `praise`, `nitpick`, `suggestion`, `issue`, `todo`, `question`, `thought`, `chore`, `note`, `typo`, `polish`.
- **`[<decoration>]`** (optional): `blocking`, `non-blocking`, `if-minor`, `security`. Decorations stack, for example `[issue][blocking][security]`.
- **`<subject>`**: one sentence, imperative or declarative, no trailing period.
- **Defaults:** `issue` and `question` default to `[blocking]`; make exceptions explicit (`[issue][non-blocking]`). `praise`, `nitpick`, `thought`, `chore`, `note`, `typo`, `polish` default to `[non-blocking]`. `suggestion` defaults to `[blocking]` unless paired with `[if-minor]` or `[non-blocking]`.
- Do not stack multiple labels in one comment; post separate comments. No emojis as a substitute for a label.

**Reviewer comments (commenting on a branch you review):**

<pre style="white-space: pre-wrap; word-wrap: break-word;">
[issue][blocking] GetEnvironmentVariable returns null here; the downstream Dimensions.Add throws ArgumentNullException
[suggestion][if-minor] Rename local var c to count for readability
[question][blocking] Why is reloader excluded but ingress.yaml in scope? Neither injects pod env
[praise] Great boundary coverage in MetricSourceExtensionsTests
[nitpick] Two-space indent here, but the surrounding file uses tabs
[issue][blocking][security] Connection string with credentials logged at Information level on line 142
</pre>

**Author replies (replying to comments on your own branch):** author replies are usually `[chore]`, `[note]`, or `[question]`. Cite the commit SHA when claiming a fix is applied, the new test name when claiming coverage, and the follow-up work-item ID when deferring.

<pre style="white-space: pre-wrap; word-wrap: break-word;">
[chore] Pushed f3a9c0d fixing the null check
[note] Intentional per prior art; renaming would invalidate the predecessor schema, leaving as is
[question] Did you mean wrap in try/catch? I kept the explicit null check to preserve the stack trace
[chore] Deferred to work item 12345678; out of scope for this PR
</pre>

**Self-check before posting any comment or reply:** (1) opens with a bracket label; (2) decoration is explicit when it is not the label default; (3) subject is one sentence with no trailing period; (4) for a reply, the label still fits (an author reply is usually `[chore]`, `[note]`, or `[question]`, not the original `[suggestion]` or `[issue]`).

### Conventional Commits (recommended fix commits and PR titles)

When a review recommends how the author should commit a fix, when you cite a fix commit, or when you draft the PR title in the review doc, use the repo's bracketed Conventional Commits form:

<pre style="white-space: pre-wrap; word-wrap: break-word;">
[&lt;type&gt;][&lt;scope&gt;] [&lt;series name&gt;][&lt;seq&gt;/&lt;total&gt;] &lt;description&gt;
</pre>

- **`[<type>]`** (required): `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`. Append `!` for a breaking change (`[feat]!`) and add a `BREAKING CHANGE:` footer.
- **`[<scope>]`** (encouraged): shortest noun for the area, for example `obs`, `ev2`, `helm`, `mpac`, `geneva`, `pipeline`, `runtime`, `deps`, `docs`.
- **`[<series name>][<seq>/<total>]`** (mandatory only for a multi-PR series): identical series name across the set, sequence from 1, include the total.
- **`<description>`**: imperative mood, no trailing period, target under 72 chars (up to 100 when a series tag is present), no file paths or line numbers.

Examples:

<pre style="white-space: pre-wrap; word-wrap: break-word;">
[fix][ev2] Add missing Bleu exclusion checks for PortalRPObservability dependencies
[feat][obs] [Add Geo Dimension to Monitoring][1/2] Add Geography dimension to Geneva metric defs
[chore][deps] Bump System.Drawing.Common from 4.6.0 to 4.7.2
</pre>

### ADO thread workflow

When asked to respond to existing Azure DevOps threads on a PR:

1. Read every active thread on the PR (`ado-repo_list_pull_request_threads` filtered by `status: "Active"`).
2. For each thread, read the full comment chain (`ado-repo_list_pull_request_thread_comments`). Never reply based on only the first comment.
3. Decide one of: **Agree and apply**, **Agree and defer (with a follow-up work item)**, **Disagree (with reasoning)**, or **Clarify (ask the reviewer)**.
4. Draft the reply text in Conventional Comments form and show it to the user. Group replies by thread ID with a ready-to-paste block so the human can copy and paste without re-reading the analysis.
5. Show the user the draft and the planned post-action (Resolve, Mark as Fixed, or Leave Active) and wait for explicit approval before calling `ado-repo_reply_to_comment` or `ado-repo_update_pull_request_thread`.

Tone: respectful, brief, specific.

---

## References

Canonical industry sources:

- Microsoft: [Code-with-Engineering Playbook: Code Reviews](https://microsoft.github.io/code-with-engineering-playbook/code-reviews/)
- Microsoft Learn: [Review pull requests in Azure Repos](https://learn.microsoft.com/azure/devops/repos/git/review-pull-requests)
- Google: [What to look for in a code review](https://google.github.io/eng-practices/review/reviewer/looking-for.html)
- Google: [The standard of code review](https://google.github.io/eng-practices/review/reviewer/standard.html)
- Google: [How to write code review comments](https://google.github.io/eng-practices/review/reviewer/comments.html)
- Google: [Handling pushback in code reviews](https://google.github.io/eng-practices/review/reviewer/pushback.html)
- Conventional Comments: [conventionalcomments.org](https://conventionalcomments.org/)
- Conventional Commits: [conventionalcommits.org](https://www.conventionalcommits.org/en/v1.0.0/)

AI-assisted review technique:

- Anthropic: [Claude prompting best practices](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices), especially [long-context prompting](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices#long-context-prompting) (put longform inputs such as the diff at the top of the prompt with the query last, and ground responses in quoted fragments) and the guidance on clear role-setting, few-shot examples, and parallel tool calls that this workflow relies on.
