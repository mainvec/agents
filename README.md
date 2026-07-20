# agents

Shared Copilot skills, instructions, and prompts for all mainvec projects.

## Structure

```
instructions/   — .instructions.md files (Copilot instruction rules)
skills/         — skill folders, each with a SKILL.md and optional assets/
```

## Setup (new machine)

Clone the repo and symlink it to `~/.mainvec`:

```bash
git clone https://github.com/mainvec/agents ~/Development/mainvec/agents
ln -s ~/Development/mainvec/agents ~/.mainvec
```

> Use the full `~/Development/mainvec/agents` path (not a relative path) so the symlink resolves correctly from any working directory.

Add the shared directories to your VS Code **User** settings so they are available in every workspace opened with that profile:

```json
{
	"chat.instructionsFilesLocations": {
		"~/.mainvec/instructions": true
	},
	"chat.agentSkillsLocations": {
		"~/.mainvec/skills": true
	}
}
```

Merge these entries with any existing values for the same settings. Settings Sync can carry the configuration to another machine, but the repository and symlink must also exist there.

## Discovery behavior

- Instructions without an `applyTo` pattern load when their description matches the current task or when they are attached manually. This keeps specialized workflows out of unrelated requests.
- Skills load on demand when their description matches the task and are also available as slash commands, such as `/feature-workflow`.
- Repository instructions may load alongside these personal instructions. Personal instructions take precedence if they conflict.

## Verify setup

1. Run **Chat: Open Customizations** from the Command Palette.
2. Confirm the shared instructions and skills appear without errors.
3. To troubleshoot loading, right-click the Chat view, select **Diagnostics**, and inspect the customization sources for the current request.

## Contributing

- Edit skills and instructions directly in this repo.
- Commit and push — all machines sharing the symlink get the update on next `git pull`.
