# Installing the mikado-skills plugin

Three install paths, depending on what you want.

## 1. Marketplace install (recommended)

In Claude Code, run:

```
/plugin marketplace add vordimous/mikado-skills
/plugin install mikado@mikado-skills
```

Claude Code resolves the marketplace, registers it, and installs the `mikado` plugin from it. The three skills (`/mikado`, `/mikado-loop`, `/mikado-mr`) are immediately available.

To update later:

```
/plugin marketplace update mikado-skills
/plugin update mikado@mikado-skills
```

To uninstall:

```
/plugin uninstall mikado@mikado-skills
```

## 2. Manual install (no marketplace)

If you prefer to manage skills as plain files:

```bash
# User-scope (skills available in every project)
git clone https://github.com/vordimous/mikado-skills.git ~/Code/mikado-skills
cp -r ~/Code/mikado-skills/plugins/mikado/skills/mikado ~/.claude/skills/
cp -r ~/Code/mikado-skills/plugins/mikado/skills/mikado-loop ~/.claude/skills/
cp -r ~/Code/mikado-skills/plugins/mikado/skills/mikado-mr ~/.claude/skills/
```

Or for a single project:

```bash
mkdir -p .claude/skills
cp -r ~/Code/mikado-skills/plugins/mikado/skills/mikado .claude/skills/
cp -r ~/Code/mikado-skills/plugins/mikado/skills/mikado-loop .claude/skills/
cp -r ~/Code/mikado-skills/plugins/mikado/skills/mikado-mr .claude/skills/
```

You miss out on `/plugin update`, but the skills work identically.

## 3. Development install

Clone the repo somewhere convenient and add it as a local marketplace so you can edit and test:

```
/plugin marketplace add /Users/you/Code/mikado-skills
/plugin install mikado@mikado-skills
```

Edits to `plugins/mikado/skills/<skill>/SKILL.md` are picked up on next invocation. No publish step required. Useful when you're iterating on a wrapper skill and want fast feedback.

## Verifying

After install, in a Claude Code session, run any of the slash commands:

```
/mikado
```

If it shows up in your slash-command list, you're done. The first invocation with no argument prints usage; the second with a goal kicks off the workflow.

## What gets installed

The plugin registers three skills:

| Skill | Slash command | What it does |
|---|---|---|
| `mikado` | `/mikado <goal>` | Drives the full method |
| `mikado-loop` | `/mikado-loop` | Executes one leaf and stops |
| `mikado-mr` | `/mikado-mr` | Synthesizes a GitLab MR proposal |

No additional dependencies are installed. The skills use Claude Code's built-in tools (`Bash`, `Read`, `Write`, `Edit`, `Glob`, `Grep`, `Agent`, `EnterWorktree`, `ExitWorktree`, `TodoWrite`, `AskUserQuestion`).

The `mikado-mr` skill calls `glab` (GitLab CLI). If you're on GitHub or another forge, the skill still synthesizes the MR title and body; you'll just need to substitute your own command at the proposal step.
