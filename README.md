# mikado-skills

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Claude Code](https://img.shields.io/badge/Claude%20Code-plugin-7E5BEF.svg)](https://docs.claude.com/en/docs/claude-code/overview)
[![Version](https://img.shields.io/badge/version-0.1.0-informational)](CHANGELOG.md)

Mikado Method skills for [Claude Code](https://docs.claude.com/en/docs/claude-code/overview), packaged as a plugin.

The Mikado Method turns large, scary refactors into a graph of small, reversible commits. You state a goal, attempt it naively in an isolated worktree, watch what breaks, and use those failures as your prerequisite list. You discard the experiment, record the prerequisites, and execute leaves one at a time on a real feature branch. The graph is the artifact; experiment code is throwaway.

These skills make that workflow native to Claude Code.

## What's in here

Three composable skills under one plugin:

- **`mikado`**: Start a goal. Run the naive experiment in a git worktree, analyze failures, record prerequisites in `.mikado/<slug>.md`, drive the leaf loop. Reverts are free; commits are small and explicit.
- **`mikado-loop`**: Execute exactly one leaf end-to-end (pick → implement → commit → mark done) and stop. Designed to compose with the built-in `/loop` skill so you can run `/loop /mikado-loop` and let Claude advance leaf-by-leaf with fresh context per leaf.
- **`mikado-mr`**: When the goal is complete, synthesize a merge or pull request from the graph and commit history. Detects the forge (GitHub or GitLab) from the git remote and proposes the appropriate `gh pr create` or `glab mr create` command; falls back to manual instructions for unknown forges. Never runs the create command without confirmation.

For project-specific conventions (build commands, ticket key formats, known flakes), put them in your project's `CLAUDE.md`. The skill reads it during preflight, and also auto-detects Gradle, npm, and pyproject build systems from the project structure.

## Installation

Add this repo as a marketplace and install the `mikado` plugin from inside Claude Code:

```
/plugin marketplace add vordimous/mikado-skills
/plugin install mikado@mikado-skills
```

That's it. The three skills are now available as `/mikado`, `/mikado-loop`, and `/mikado-mr`.

## Quick start

```
/mikado remove Redisson and migrate to Lettuce
```

Claude will:

1. Run preflight (clean tree, feature branch, build/test detection).
2. Print the Mikado ethos and ask for cadence / MR strategy / implementation choices.
3. Write `.mikado/remove-redisson.md` with the goal.
4. Open a throwaway worktree and attempt the change naively.
5. Discard the worktree; record every failure as a prerequisite in the Mermaid graph.
6. Pick a leaf and implement it on your real feature branch.
7. Commit one leaf, mark it checked, repeat.

The skill pauses at each phase boundary to ask for input (cadence and MR strategy chosen up front, then commit strategy, sub-prerequisite triage, and so on as the leaf loop progresses). See [docs/session-rhythm.md](docs/session-rhythm.md) for what to expect.

`/mikado` drives the leaf loop in the same session by default, which is what you want if you'd rather review each commit before the next leaf starts. For power-through runs where you want fresh context per leaf instead of reviewing between commits, use the optional loop variant:

```
/loop /mikado-loop
```

When all prerequisites are checked, the goal itself becomes the final leaf. Then:

```
/mikado-mr
```

builds the title and body from the graph and the commits, detects your forge, and shows you the right `gh pr create` or `glab mr create` command. You decide when to run it.

## Why this exists

The Mikado Method ([Daniel Brolund and Ola Ellnestam, 2014](https://mikadomethod.info/)) was designed for humans pair-programming on hard refactors. With AI agents, the parts that humans found tedious (committing to the discipline of "revert and record" rather than "fix and continue") become a strength: agents have no ego attached to the experiment, and `git revert` is cheap.

I wrote about this in detail at [wellaged.dev/posts/mikado-method-ai-agents](https://wellaged.dev/posts/mikado-method-ai-agents/). These skills are the working implementation that backs that post.

## License

MIT. See [LICENSE](LICENSE).
