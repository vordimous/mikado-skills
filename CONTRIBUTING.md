# Contributing

Thanks for considering a contribution. This plugin is small enough that the bar for contribution is low: bug reports are gold, doc fixes are welcome, and skill-text refinements are how this gets better over time.

## Reporting issues

Open an issue at [github.com/vordimous/mikado-skills/issues](https://github.com/vordimous/mikado-skills/issues) with:

- What you ran (`/mikado <goal>`, `/loop /mikado-loop`, `/mikado-mr`)
- What happened, what you expected
- The relevant excerpt of `.mikado/<slug>.md` if it's a workflow issue
- Your forge (GitHub, GitLab, self-hosted, other) if it's an MR-creation issue

Please scrub ticket keys, internal hostnames, and proprietary code before pasting.

## Development

Clone and add as a local marketplace so you can iterate on skill text without publishing:

```shell
git clone https://github.com/vordimous/mikado-skills.git
cd mikado-skills
```

Then in any Claude Code session:

```shell
/plugin marketplace add /absolute/path/to/mikado-skills
/plugin install mikado@mikado-skills
```

Edits to `plugins/mikado/skills/<skill>/SKILL.md` are picked up on the next invocation. No publish step.

## Style

- Conventional Commits for commit messages: `feat:`, `fix:`, `chore:`, `docs:`, `refactor:`. The plugin practices what it preaches.
- One concern per PR. If you're touching a skill's behavior and its docs, that's fine; if you're touching two skills, split the PR.
- Skill descriptions ride on triggers. When you change a skill's behavior, re-read its `description:` frontmatter and update it so the router still picks the right skill for the right phrase.

## Testing skill changes

There is no test harness. The way to verify a change is to run the skill against a real (or sandbox) repo and observe behavior. For substantive changes:

1. Create a throwaway repo with a small refactor goal.
2. Run the affected skill end-to-end.
3. Confirm the behavior matches the SKILL.md instructions.
4. If the skill produced unexpected commits, attach the commit log to the PR.

## What's in scope

- Refinements to skill instructions (clearer triggers, better edge-case handling, more accurate forge detection).
- Bug fixes in the documented workflow.
- New worked examples under `examples/`.
- Docs improvements.

## What's out of scope (for now)

- Adding new skills to this plugin. The three current skills are intentional. New skills belong in their own plugin.
- Forges beyond GitHub and GitLab as first-class. PRs that improve the manual-mode fallback for other forges are welcome; PRs that hardcode a third forge's CLI are not.
- Runtime dependencies. The skills use Claude Code's built-in tools only.
