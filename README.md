# magical-skills

Personal, reusable agent skills shared across devices. Add new skills under
`skills/<skill-name>/`, with a `SKILL.md` file and any required supporting files.

## Maestro

[`skills/maestro/SKILL.md`](skills/maestro/SKILL.md) orchestrates feature delivery:
research, design, specification, dependency-aware implementation, independent review,
and validation. Keep its `references` directory with the skill.

### Install on Windows

Requires GitHub CLI and GitHub Copilot CLI. This repository is public.
Run in PowerShell:

```powershell
gh repo clone Gundwane21/magical-skills "$HOME\magical-skills"
New-Item -ItemType Directory -Force "$HOME\.copilot\skills" | Out-Null
Copy-Item "$HOME\magical-skills\skills\maestro" "$HOME\.copilot\skills\" -Recurse -Force
```

In Copilot CLI:

```text
/skills reload
/skills info maestro
/maestro plan <feature>
```

Other modes: `/maestro <feature>`, `/maestro research <feature>`,
`/maestro execute <plan>`, `/maestro resume`, and `/maestro status`.

### Update

```powershell
git -C "$HOME\magical-skills" pull --ff-only
Copy-Item "$HOME\magical-skills\skills\maestro" "$HOME\.copilot\skills\" -Recurse -Force
```

Run `/skills reload` afterward. The copy updates existing files but does not remove
files deleted upstream.

### Requirements

Maestro uses subagents, session task tracking, file/shell tools, and the target
repository's build/test tooling. It declares no mandatory MCP servers or additional
packages. Configure model preferences, authentication, and project dependencies on each
device separately. Director scheduling is optional; the protocol includes a fallback.

## Mehmet's AI Review

[`skills/mehmets-ai-review/SKILL.md`](skills/mehmets-ai-review/SKILL.md) defines a
multi-model branch and pull-request review workflow. It combines independent reviewers,
evidence-backed findings, a second-pass audit, and a local review report.

### Install on Windows

After cloning this repository as shown above:

```powershell
New-Item -ItemType Directory -Force "$HOME\.copilot\skills" | Out-Null
Copy-Item "$HOME\magical-skills\skills\mehmets-ai-review" "$HOME\.copilot\skills\" -Recurse -Force
```

In Copilot CLI:

```text
/skills reload
/skills info mehmets-ai-review
Use /mehmets-ai-review to review my current branch
```

### Update

```powershell
git -C "$HOME\magical-skills" pull --ff-only
Copy-Item "$HOME\magical-skills\skills\mehmets-ai-review" "$HOME\.copilot\skills\" -Recurse -Force
```

Run `/skills reload` afterward.

### Requirements and portability

The original skill is included unchanged. It uses Git, subagents, and file/shell tools,
and prefers three models from different vendors when available. It includes Azure
DevOps-specific tool names and repository-specific examples; those tools, paths, and
conventions must be adapted to the repository being reviewed. External authentication
and model availability are configured separately, not installed by copying the skill.

## PR Quality Review and Fix Loop

These skills are copied unchanged from
[`Gundwane21/ai-jobs`](https://github.com/Gundwane21/ai-jobs/tree/903dd7d00ea1f3aadf1b8588c0c7f784a25c6317/.agents/skills)
at commit `903dd7d00ea1f3aadf1b8588c0c7f784a25c6317`.
The original repository retains its copies.

- [`pr-quality-review`](skills/pr-quality-review/SKILL.md) reviews a captured PR
  revision with independent reviewers and verified findings. By default, it posts
  one PR summary. Request report-only output to prevent publication.
- [`pr-review-fix-loop`](skills/pr-review-fix-loop/SKILL.md) performs an explicitly
  requested review, fix, and verification loop for at most five rounds. By default,
  it commits and pushes only a clean result. Request local-only changes to prevent
  commits and pushes.

Install both directories together. The fix loop uses
`pr-quality-review/reviewers.md`. Each directory includes the source MIT license.
The skills reference AI jobs standards and contracts, including
`CODING_STANDARDS.md`, `PYTHON_CODING_REFERENCES.md`, and `docs/README.md`.
Adapt those document references when using the skills in another repository.
Automatic post-create or post-push invocation requires a trusted repository
instruction; copying the skills does not configure that instruction.

### Install on macOS or Linux

From your clone of this repository:

```bash
mkdir -p "$HOME/.copilot/skills"
cp -R skills/pr-quality-review skills/pr-review-fix-loop "$HOME/.copilot/skills/"
```

### Install on Windows

```powershell
New-Item -ItemType Directory -Force "$HOME\.copilot\skills" | Out-Null
Copy-Item "$HOME\magical-skills\skills\pr-quality-review" "$HOME\.copilot\skills\" -Recurse -Force
Copy-Item "$HOME\magical-skills\skills\pr-review-fix-loop" "$HOME\.copilot\skills\" -Recurse -Force
```

Start a new Copilot session after installation. In Copilot CLI, you can also run
`/skills reload`.

See the [official Copilot CLI skill documentation](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-skills).