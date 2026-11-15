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

See the [official Copilot CLI skill documentation](https://docs.github.com/en/copilot/how-tos/copilot-cli/customize-copilot/add-skills).