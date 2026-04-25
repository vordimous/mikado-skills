# mikado-skills

Mikado Method skills for [Claude Code](https://docs.claude.com/en/docs/claude-code/overview), packaged as a plugin.

The Mikado Method turns large, scary refactors into a graph of small, reversible commits. You state a goal, attempt it naively in an isolated worktree, watch what breaks, and use those failures as your prerequisite list. You discard the experiment, record the prerequisites, and execute leaves one at a time on a real feature branch. The graph is the artifact; experiment code is throwaway.

These skills make that workflow native to Claude Code.

## What's in here

Three composable skills under one plugin:

- **`mikado`** — Start a goal. Run the naive experiment in a git worktree, analyze failures, record prerequisites in `.mikado/<slug>.md`, drive the leaf loop. Reverts are free; commits are small and explicit.
- **`mikado-loop`** — Execute exactly one leaf end-to-end (pick → implement → commit → mark done) and stop. Designed to compose with the built-in `/loop` skill so you can run `/loop /mikado-loop` and let Claude advance leaf-by-leaf with fresh context per leaf.
- **`mikado-mr`** — When the goal is complete, synthesize a GitLab merge request from the graph and commit history. Proposes the `glab mr create` command; never runs it without confirmation.

The skills are designed to be composed with thin wrapper skills for project-specific conventions (build commands, ticket key formats, known flakes, MR template). See [`docs/writing-a-wrapper.md`](docs/writing-a-wrapper.md) and the [`examples/wrapper-template/`](examples/wrapper-template/) directory.

## Installation

Add this repo as a marketplace and install the `mikado` plugin from inside Claude Code:

```
/plugin marketplace add vordimous/mikado-skills
/plugin install mikado@mikado-skills
```

That's it. The three skills are now available as `/mikado`, `/mikado-loop`, and `/mikado-mr`.

## Quick start

```
/mikado remove Redisson from x1-app
```

Claude will:

1. Run preflight (clean tree, feature branch, build/test detection).
2. Write `.mikado/remove-redisson.md` with the goal.
3. Open a throwaway worktree and attempt the change naively.
4. Discard the worktree; record every failure as a prerequisite in the Mermaid graph.
5. Pick a leaf and implement it on your real feature branch.
6. Commit one leaf, mark it checked, repeat.

To auto-pace through all leaves with fresh context per leaf:

```
/loop /mikado-loop
```

When all prerequisites are checked, the goal itself becomes the final leaf. Then:

```
/mikado-mr
```

builds the MR title and body from the graph and the commits, and shows you the `glab mr create` command. You decide when to run it.

## Why this exists

The Mikado Method ([Daniel Brolund and Ola Ellnestam, 2014](https://mikadomethod.info/)) was designed for humans pair-programming on hard refactors. With AI agents, the parts that humans found tedious — committing the discipline of "revert and record" rather than "fix and continue" — become a strength: agents have no ego attached to the experiment, and `git revert` is cheap.

I wrote about this in detail at [wellaged.dev/posts/mikado-method-ai-agents](https://wellaged.dev/posts/mikado-method-ai-agents/). These skills are the working implementation that backs that post.

## Repository layout

```
mikado-skills/
├── .claude-plugin/
│   └── marketplace.json          Marketplace manifest (lists the mikado plugin)
├── plugins/
│   └── mikado/
│       ├── .claude-plugin/
│       │   └── plugin.json       Plugin manifest
│       └── skills/
│           ├── mikado/SKILL.md
│           ├── mikado-loop/SKILL.md
│           └── mikado-mr/SKILL.md
├── examples/
│   ├── wrapper-template/         Generic project wrapper skeleton
│   └── worked-goal/              Real .mikado/<slug>.md from a completed goal
├── docs/
│   ├── method.md                 The method, in 5 minutes
│   ├── installing.md             Install paths (marketplace, manual, dev)
│   └── writing-a-wrapper.md      How to layer project conventions on top
├── LICENSE
└── README.md
```

## Status

`v0.1.0`. The skills are in production use on a Java + Vue monorepo. The shape is stable; sharp edges are still being filed down. Issues and PRs welcome at [github.com/vordimous/mikado-skills](https://github.com/vordimous/mikado-skills).

## License

MIT. See [LICENSE](LICENSE).
