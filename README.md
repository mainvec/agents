# agents

Public Copilot instructions and skills for all mainvec projects: the
development workflow, test-first rules, and technology skills. Internal-only
material lives in the private `mainvec/agents-internal` repo.

This repo is an [Agent Plugin](https://agent-plugins.org/) named `mainvec`, so VS Code, the Copilot CLI, and the Copilot app load its skills without copying files.

## Structure

```
plugin.json                  — Agent Plugins 1.0 manifest (name, version)
skills/                      — skill folders, each with a SKILL.md and optional assets/
com.github.copilot/rules/    — Copilot instruction files (.instructions.md)
```

## Setup (new machine)

Install the plugin; it provides both the skills and the instructions. Pick one:

- **No clone (simplest):** in VS Code run **Chat: Install Plugin From Source** and enter `https://github.com/mainvec/agents`. VS Code clones it and checks for updates about every 24 hours.
- **From a clone (to edit skills or rules):** clone the repo and register the folder in your VS Code **User** settings. Changes apply on `git pull`.

  ```bash
  git clone https://github.com/mainvec/agents ~/Development/mainvec/agents
  ```

  ```json
  "chat.pluginLocations": {
  	"/Users/<you>/Development/mainvec/agents": true
  }
  ```

- **Copilot CLI:** `copilot plugin install mainvec/agents`.

Team members install the private `mainvec/agents-internal` plugin the same way.

Skills from a plugin are invoked with the plugin prefix, for example `/mainvec:feature-workflow`.

Other agent tools read the same `SKILL.md` format ([agentskills.io](https://agentskills.io/)); point them at `skills/`. The `com.github.copilot/` folder is Copilot-specific and other tools ignore it.

## Discovery behavior

- Instructions without an `applyTo` pattern load when their description matches the current task or when they are attached manually. This keeps specialized workflows out of unrelated requests.
- Skills load on demand when their description matches the task and are also available as slash commands, such as `/mainvec:feature-workflow`.
- Repository instructions may load alongside these personal instructions. Personal instructions take precedence if they conflict.

## Verify setup

1. Run **Chat: Open Customizations** from the Command Palette.
2. Confirm the shared instructions and skills appear without errors.
3. To troubleshoot loading, right-click the Chat view, select **Diagnostics**, and inspect the customization sources for the current request.

## Contributing

- Edit skills and rules directly in this repo, through a pull request.
- **Bump `version` in `plugin.json`** in any PR that changes skills or rules; plugins installed from the Git URL only update when the version changes. Clone-based setups pick up changes on `git pull`.
- Content here is public. Anything you would not want a competitor to read goes to `mainvec/agents-internal`.

## How the rules reach everyone

Nothing from this repo is copied into project repositories. Each audience gets
the rules through a channel it already uses:

| Audience | Mechanism |
|---|---|
| Teammates and their local agents | The `mainvec` and `mainvec-internal` plugins |
| Outside contributors | Org defaults in the public `mainvec/.github` repo: `CONTRIBUTING.md`, issue and PR templates |
| Every pull request, human or agent | Reusable checks in `mainvec/.github` (PR title, plan numbering), called from each repo |
| Cloud and outside agents | Each repo's `AGENTS.md`: project rules plus one line linking to this repo's workflow |

When a workflow rule changes here, update `mainvec/.github` `CONTRIBUTING.md`
and its checks in the same change set. The `AGENTS.md` template is at
`skills/feature-workflow/assets/AGENTS-template.md`.
