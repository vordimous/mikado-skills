# wrapper-template

A minimal project-scoped wrapper skill for the Mikado plugin. Drop it into any repo at `.claude/skills/mikado-<projectname>/SKILL.md`, replace the `<PLACEHOLDER>` tokens with your project's specifics, and you're done.

## Why a wrapper

The three skills in the `mikado` plugin (`mikado`, `mikado-loop`, `mikado-mr`) are intentionally project-agnostic. They know how to drive the method, but they don't know:

- What your build and test commands look like.
- What ticket key format your branches use.
- Which tests are known to flake.
- Which paths trigger cross-module impact.
- What your MR template requires.

A wrapper skill captures all of that in one place. The wrapper says "read the global mikado skill, then apply these adaptations" — so the conventions stay in one file and the global skill stays clean.

## How to use

1. Copy `.claude/skills/mikado-myproject/SKILL.md` into your project at the same path.
2. Rename the directory: `mikado-myproject` → `mikado-<your-project-slug>`.
3. Open `SKILL.md` and replace every `<PLACEHOLDER>` token. Search for the angle brackets to find them.
4. Test it: in your project, run `/mikado-<your-project-slug> <some goal>`.

## Naming convention

We recommend `mikado-<projectname>` so it's discoverable and obvious. If your project has a short codename, use it. Examples:
- `mikado-x1` for an internal platform called X1.
- `mikado-frontend` if you only want the wrapper for one part of a monorepo.
- `mikado-acme` if you work on multiple Acme products and want one shared wrapper.

The skill name only affects the slash-command surface (`/mikado-<projectname>`); the global skills (`/mikado`, `/mikado-loop`, `/mikado-mr`) remain available regardless.

## See also

- [`docs/writing-a-wrapper.md`](../../docs/writing-a-wrapper.md) for the full guide on what to put in a wrapper and why.
- The plugin's own skills under [`plugins/mikado/skills/`](../../plugins/mikado/skills/) to see what surface area the wrapper layers on top of.
