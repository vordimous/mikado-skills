# mikado plugin

Three composable skills that implement the [Mikado Method](https://mikadomethod.info/) for Claude Code.

## Skills

### `/mikado <goal>`

Starts a Mikado goal. Drives the full method:

- Phase 0: preflight (clean tree, feature branch, build/test detection).
- Phase 0.5: resume detection if `.mikado/<slug>.md` already exists.
- Phase 1: record the goal in `.mikado/<slug>.md` with frontmatter, Mermaid graph stub, prereq checklist.
- Phase 2: naive experiment in a git worktree. Run, capture failures, discard.
- Phase 3: convert failures into prerequisites; commit `mikado: record prerequisites from naive attempt`.
- Phase 4: leaf loop. Pick a leaf, implement on the feature branch, commit, mark checked, repeat.
- Phase 5: when all leaves are checked, attempt the goal itself; mark Status `Complete`.

Invoke as `/mikado <goal-statement>` or `/mikado <path-to-spec.md>`.

### `/mikado-loop`

Executes exactly one leaf end-to-end and exits. Designed for `/loop /mikado-loop` so you can let Claude work leaf-by-leaf with fresh context each invocation — the Mikado-for-agents recommendation of "one leaf per session."

Emits a status line at the end of each invocation that `/loop` can parse:
- `MIKADO_LOOP_ADVANCE` — leaf done, more remain
- `MIKADO_LOOP_EXPAND` — leaf was too big; expanded into sub-prereqs
- `MIKADO_LOOP_DONE` — goal complete, run `/mikado-mr`
- `MIKADO_LOOP_BLOCKED` — needs human input

### `/mikado-mr`

Reads the completed `.mikado/<slug>.md` and synthesizes a GitLab MR proposal: title, body (with the Mermaid graph and commit sequence inline), and the `glab mr create` command. Never runs the command itself without confirmation.

Supports three MR shapes:
- **Single MR** (default) — one MR for the whole goal.
- **Clustered MRs** — one MR per subsystem cluster (e.g. `common` / `ui` / `migration`), each targeting the main branch independently. Recommended for goals that span ≥3 subsystems.
- **Stacked MRs** — chain of MRs where each targets its parent. Only for teams that don't squash-merge.

## How they compose

```
/mikado <goal>           ── start the goal, run naive experiment, record prereqs
   │
   ▼
/loop /mikado-loop       ── auto-pace through leaves, fresh context each iteration
   │
   ▼
/mikado-mr               ── synthesize the MR from graph + commits
```

Each skill operates on the same `.mikado/<slug>.md` file. The graph is the source of truth; the skills read and write it but never push state to memory or external systems.

## Operating rules these skills follow

These are hard-coded into the skill instructions; users don't need to remember them, but reviewers might want to know:

- **Never push or rewrite history.** The skills create branches, stage specific files, commit, revert, cherry-pick, stash. They do not push, pull, merge, rebase, reset, force-delete branches, or clean.
- **One leaf per commit.** Bundling prerequisites defeats the method.
- **Stage specific files, never `git add -A`.** The user sees each commit's diff stat before it lands.
- **Resist fixing during the naive experiment.** The experiment's job is to surface failures, not to fix them. If you start fixing, you lose the signal.
- **Revert is free.** If a leaf needs more than a handful of changes before tests pass, you're probably fixing instead of experimenting. Exit and re-scope.
- **Use the narrowest verification.** Compile-only for pure refactors; single-class `--tests` filter for logic changes. Module-level test suites are a yellow flag and reserved for the commit boundary before opening the MR.

The full set of agent-safety properties (adapted from [GitButler's framework](https://blog.gitbutler.com/agentic-safety)) is documented in the `mikado` skill's SKILL.md.

## Wrapper skills

These three skills are deliberately project-agnostic. To layer project conventions on top — Gradle module names, Jira key prefixes, known flakes, MR templates — write a thin wrapper skill that includes a "read the global mikado skill first, then apply these adaptations" instruction. See [`../../docs/writing-a-wrapper.md`](../../docs/writing-a-wrapper.md) for the pattern, and [`../../examples/wrapper-template/`](../../examples/wrapper-template/) for a runnable skeleton.
