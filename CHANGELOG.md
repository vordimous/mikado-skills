# Changelog

All notable changes to this plugin are documented here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versioning follows [SemVer](https://semver.org/spec/v2.0.0.html).

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
