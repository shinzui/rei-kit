# rei-kit

Claude Code skills and subagents for Rei end-users.

## Installation

```bash
rei kit list                          # see all available content
rei kit install <skill-name>          # install a skill (user scope)
rei kit install <skill-name> --project  # install to project scope
rei kit status                        # see what's installed
rei kit update                        # pull latest versions
rei kit uninstall <skill-name>        # remove a skill
```

## Available Skills

| Name | Description |
|------|-------------|
| `rei-bootstrap` | Interactively bootstrap intentions, habits, and reflections |
| `rei-bootstrap-habit` | Interactively bootstrap a new habit |

## How It Works

Skills and agents are installed to either user scope (`~/.config/rei/agents/`) or project scope (`.rei/agents/`). Installed content is automatically discovered by `rei agent` sessions via Claude Code's `--add-dir` mechanism.
