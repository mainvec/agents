# agents

Shared Copilot skills, instructions, and prompts for all mainvec projects.

## Structure

```
instructions/   — .instructions.md files (Copilot instruction rules)
skills/         — skill folders, each with a SKILL.md and optional assets/
```

## Setup (new machine)

Clone the repo and symlink it to `~/.mainvec` so VS Code Copilot picks it up automatically:

```bash
git clone https://github.com/mainvec/agents ~/Development/mainvec/agents
ln -s ~/Development/mainvec/agents ~/.mainvec
```

> Use the full `~/Development/mainvec/agents` path (not a relative path) so the symlink resolves correctly from any working directory.

## Contributing

- Edit skills and instructions directly in this repo.
- Commit and push — all machines sharing the symlink get the update on next `git pull`.
