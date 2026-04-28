# Changelog

All notable changes to this plugin are documented here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versioning follows [SemVer](https://semver.org/spec/v2.0.0.html).

## [0.3.2] (2026-04-28)

### Fixed

- **Phase 0.1 / Phase 0.4 permission patterns:** prefix patterns are now written as `Bash(<prefix> :*)` (with the space before `:*`) to match Claude Code's expected format. v0.3.0 wrote them without the space, so the patterns the skill auto-added to `.claude/settings.local.json` failed to match at runtime and the operator was still prompted mid-loop for things like `git worktree add`. All known-set entries (code research, git read/write, worktree management) and Phase 0.4 derived examples now use the correct format. Added `Bash(git worktree prune :*)` and `Bash(mkdir :*)` for cleanup and fallback-path creation.
- **Mermaid graph dark-mode contrast:** the `classDef observed` and `classDef anticipated` styles in the Phase 1 goal-file template, the Phase 3 example, and the worked-goal example used pale pastel fills (`#e8f5e9`, `#fff3e0`) with no explicit text color. In dark Mermaid themes (e.g. GitHub dark mode), the default white text rendered illegibly on those pale fills. Switched to dark-saturated fills (`#2e7d32`, `#e65100`) with explicit white text and brighter accent strokes; readable on both light and dark canvases. Existing goal files keep their old colors; only new goals pick up the new template.

## [0.3.1] (2026-04-28)

### Fixed

- Plugin manifest validation error on install: removed the `$schema` field from `plugins/mikado/.claude-plugin/plugin.json`. Claude Code's manifest validator rejects unrecognized keys, and `$schema` is no longer accepted there. No behavior change in the skills themselves.

## [0.3.0] (2026-04-28)

Friction reduction for unattended leaf loops: permission preflights, an opinionated worktree default for `ai-implements`, and a tightened pacing contract for `/loop /mikado-loop`.

### Added

- **Permission preflight (Phase 0.1).** New phase between Phase 0 and Phase 0.3 that diffs the skill's known Bash patterns (code research, git read/write, worktree management) against `.claude/settings.local.json`, `.claude/settings.json`, and `~/.claude/settings.json`. If gaps exist, offers to write the missing patterns to `.claude/settings.local.json` after a single confirmation. Records the outcome as `Permission preflight: granted | declined` in the goal file. **Note:** Claude Code loads settings at session startup, so granting takes effect on the *next* session, not the current one. The preflight is a one-time setup step; once granted, every subsequent goal benefits.
- **Test-command permission check (Phase 0.4).** After the testing-plan signoff, the same diff-and-write flow runs against the chosen tier commands (e.g. `Bash(./gradlew test:*)`, `Bash(npm test:*)`). Phase 0.1 covers code research; Phase 0.4 covers test runners. The two diffs do not overlap; the operator sees at most two short prompts.
- **Goal worktree default (Phase 1.0).** When `Implementation: ai-implements`, the goal now runs in a long-lived sibling worktree (default path `../<repo-name>-mikado-<slug>`, fallback `~/.claude/mikado-worktrees/<repo>-<slug>`) on a new branch (`mikado/<slug>`). The main checkout is untouched for the duration of the goal. `Implementation: coach` keeps the existing in-place behavior. Phase 0.3 announces the workspace decision in one line; it is derived, not prompted.
- **`mikado-loop` and `mikado-mr` workspace check.** Both skills detect when they are invoked from outside the goal worktree, scan `git worktree list` for the goal, and surface an actionable `cd` instruction.
- **MR target choice (Phase 3 of mikado-mr).** When `Workspace: worktree`, mikado-mr offers a one-time choice between the remote default branch (default) and the goal's `Source branch`. The decision is recorded as `MR target` in front-matter and reused on subsequent runs.
- **Front-matter additions.** `Workspace`, `Worktree path`, `Source branch`, `Permission preflight`, and `MR target` are written by Phase 1 (or by mikado-mr for `MR target`). Resumed goal files missing any of these fall back gracefully (see Phase 0.5).

### Changed

- **`mikado-loop` exit-signal contract** explicit about pacing: `MIKADO_LOOP_ADVANCE` / `MIKADO_LOOP_EXPAND` → next iteration fires immediately, no idle delay; `MIKADO_LOOP_DONE` / `MIKADO_LOOP_BLOCKED` → stop. The previous behavior depended on the orchestrator's self-pacing heuristic, which defaulted to 20-30 minute idle delays for indeterminate work.
- **`/loop /mikado-loop` recommendation** unchanged as the primary form, with `/loop 60s /mikado-loop` documented as a workaround for harnesses that ignore the contract.
- **`Bash(git switch -c :*)`** is the narrower pattern written into `permissions.allow` (instead of `Bash(git switch:*)`, which would also permit force-switch and conflict with the operating rules).

### Breaking

- **Default workspace for new `ai-implements` goals is `worktree`, not the operator's current branch.** The goal file lives inside the worktree; the main checkout's `.mikado/` stays empty. Existing goals are unaffected: missing `Workspace` field on resume is treated as `in-place` for backward compatibility. To opt out for a new goal, edit the front-matter `Workspace` field to `in-place` after Phase 1 records the goal, or pick `Implementation: coach`.

### Migration

- Goal files written by 0.2.0 and earlier resume cleanly. Phase 0.5 detects missing fields and falls back: `Workspace` → `in-place`, `Permission preflight` → re-run Phase 0.1 once and write the field, `MR target` → mikado-mr prompts on first run, `Source branch` → mikado-mr falls back to remote default and skips the source-branch alternative.

## [0.2.0] (2026-04-26)

Three feature additions to the `mikado` skill plus the foundation for upcoming workflow modes. All changes are additive; resuming a goal started under 0.1.0 works unchanged (the new front-matter fields default cleanly when missing).

### Added

- **Spec-file ingestion.** `/mikado <path-to-spec.md>` reads a markdown spec (Goal, Why, Acceptance, Tasks, Open Questions sections) and seeds the prerequisite graph with **anticipated** nodes that the naive experiment then confirms, refines, or invalidates via three-way reconciliation (confirmed / new / stale). The Mermaid graph distinguishes anticipated vs observed nodes via `classDef`.
- **Per-goal testing plan (Phase 0.4).** A new pause derives a tiered plan (fast / targeted / regression, plus optional affinity rules and skip list) from existing repo context (`CLAUDE.md`, README, build manifests). The user signs off, amends, or replaces it. The leaf loop, naive experiment, subagent acceptance criteria, flaky-test rule, and Phase 5 finish step now reference the plan's tiers by name. Build/Test front-matter fields removed in favor of a structured `## Testing plan` section in the goal file.
- **Goal configuration choices (Phase 0.3).** Three new front-matter fields prompted at goal start: `Cadence` (continuous / per-leaf / per-cluster), `MR strategy` (per-leaf / per-cluster / at-goal), `Implementation` (ai-implements / coach). A 3-sentence ethos preamble prints first. Phase 0.5 (resume) suppresses both preamble and prompts.
- **Skill version field** in goal-file front-matter, written at goal creation. Resume detects schema mismatches against the running plugin version; future migrations key off this field.
- **`docs/session-rhythm.md`.** A new doc that walks through the interactive checkpoints across a Mikado session (testing-plan signoff, goal configuration, leaf-loop handoff, in-loop pauses, finish).
- **`/mikado-mr` consumes Phase 0.3 `MR strategy`.** When set, the request shape is taken from front-matter rather than re-prompted in Phase 3.5.
- **`/mikado-loop` reads Phase 0.3 fields.** Surfaces a one-line fallback warning when running standalone with a non-default combination.

### Changed

- Goal-file template gains `## Acceptance` and `## Testing plan` sections between front-matter and the Mermaid graph. Front-matter gains `Source plan`, `Cadence`, `MR strategy`, `Implementation`, and `Skill version` fields.
- Operating rule "Use the narrowest verification that proves the change" rewritten to point at the goal's Testing plan tiers.
- README's "Claude will:" list renumbered to include the Phase 0.3 step.

### Phase A scope (this release)

The Phase 0.3 prompts capture all three values, but only one combination is fully wired: `Cadence: per-leaf` + `MR strategy: at-goal` + `Implementation: ai-implements` (today's behavior). Other combinations print a one-line fallback warning at the first relevant boundary and proceed with the supported behavior. Phases B (`coach`), C (`per-cluster` / `continuous` cadence), and D (sub-branches and sub-MRs) wire the remaining combinations in subsequent releases.

## [0.1.0] (2026-04-25)

Initial public release.

### Added

- `mikado` skill: drives the full Mikado Method workflow (preflight, naive worktree experiment, prerequisite recording, leaf loop, goal completion).
- `mikado-loop` skill: executes one leaf per invocation and exits with a parseable status line, so it composes with the built-in `/loop` skill for fresh-context-per-leaf progression.
- `mikado-mr` skill: synthesizes a merge or pull request from the completed goal. Detects the forge from `git remote get-url origin` and proposes the appropriate `gh pr create` or `glab mr create` command. Falls back to manual instructions for unknown forges. Supports single, clustered, and stacked request shapes.
- Marketplace manifest at `.claude-plugin/marketplace.json`. Plugin manifest at `plugins/mikado/.claude-plugin/plugin.json`.
- Worked example at `examples/worked-goal/remove-redisson.md` showing what a completed `.mikado/<slug>.md` looks like.
- Docs: `docs/method.md` (the method in 5 minutes), `docs/installing.md` (marketplace, manual, dev install paths).
