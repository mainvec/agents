# agents

Public Copilot instructions and skills for all mainvec projects: the
development workflow, test-first rules, and technology skills. Internal-only
material lives in the private `mainvec/agents-internal` repo.

## Structure

```
instructions/   — .instructions.md files (Copilot instruction rules)
skills/         — skill folders, each with a SKILL.md and optional assets/
```

## Setup (new machine)

Clone the repos and link each one under `~/.mainvec/`:

```bash
git clone https://github.com/mainvec/agents ~/Development/mainvec/agents
mkdir -p ~/.mainvec
ln -s ~/Development/mainvec/agents ~/.mainvec/agents

# team members only (private)
git clone https://github.com/mainvec/agents-internal ~/Development/mainvec/agents-internal
ln -s ~/Development/mainvec/agents-internal ~/.mainvec/agents-internal
```

> Use absolute targets (not relative paths) so the symlinks resolve from any working directory.

Add the directories to your VS Code **User** settings so they are available in every workspace opened with that profile:

```json
{
	"chat.instructionsFilesLocations": {
		"~/.mainvec/agents/instructions": true,
		"~/.mainvec/agents-internal/instructions": true
	},
	"chat.agentSkillsLocations": {
		"~/.mainvec/agents/skills": true,
		"~/.mainvec/agents-internal/skills": true
	}
}
```

Merge these entries with any existing values for the same settings. Settings Sync can carry the configuration to another machine, but the clones and symlinks must also exist there. Adding another repo later is one more symlink and one more line per setting.

Other agent tools: link the skill folders into that tool's skills directory, for example `ln -s ~/.mainvec/agents/skills/feature-workflow ~/.claude/skills/`.

## Discovery behavior

- Instructions without an `applyTo` pattern load when their description matches the current task or when they are attached manually. This keeps specialized workflows out of unrelated requests.
- Skills load on demand when their description matches the task and are also available as slash commands, such as `/feature-workflow`.
- Repository instructions may load alongside these personal instructions. Personal instructions take precedence if they conflict.

## Verify setup

1. Run **Chat: Open Customizations** from the Command Palette.
2. Confirm the shared instructions and skills appear without errors.
3. To troubleshoot loading, right-click the Chat view, select **Diagnostics**, and inspect the customization sources for the current request.

## Contributing

- Edit skills and instructions directly in this repo, through a pull request.
- Machines pick up changes on the next `git pull`.
- Content here is public. Anything you would not want a competitor to read goes to `mainvec/agents-internal`.

## How the rules reach everyone

Nothing from this repo is copied into project repositories. Each audience gets
the rules through a channel it already uses:

| Audience | Mechanism |
|---|---|
| Teammates and their local agents | This repo and `agents-internal`, linked under `~/.mainvec/` |
| Outside contributors | Org defaults in the public `mainvec/.github` repo: `CONTRIBUTING.md`, issue and PR templates |
| Every pull request, human or agent | Reusable checks in `mainvec/.github` (PR title, plan numbering), called from each repo |
| Cloud and outside agents | Each repo's `AGENTS.md`: project rules plus one line linking to this repo's workflow |

When a workflow rule changes here, update `mainvec/.github` `CONTRIBUTING.md`
and its checks in the same change set. The `AGENTS.md` template is at
`skills/feature-workflow/assets/AGENTS-template.md`.
