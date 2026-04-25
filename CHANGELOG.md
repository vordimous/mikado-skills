# Changelog

All notable changes to this plugin are documented here. Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/); versioning follows [SemVer](https://semver.org/spec/v2.0.0.html).

## [0.1.0] (2026-04-25)

Initial public release.

### Added

- `mikado` skill: drives the full Mikado Method workflow (preflight, naive worktree experiment, prerequisite recording, leaf loop, goal completion).
- `mikado-loop` skill: executes one leaf per invocation and exits with a parseable status line, so it composes with the built-in `/loop` skill for fresh-context-per-leaf progression.
- `mikado-mr` skill: synthesizes a merge or pull request from the completed goal. Detects the forge from `git remote get-url origin` and proposes the appropriate `gh pr create` or `glab mr create` command. Falls back to manual instructions for unknown forges. Supports single, clustered, and stacked request shapes.
- Marketplace manifest at `.claude-plugin/marketplace.json`. Plugin manifest at `plugins/mikado/.claude-plugin/plugin.json`.
- Worked example at `examples/worked-goal/remove-redisson.md` showing what a completed `.mikado/<slug>.md` looks like.
- Docs: `docs/method.md` (the method in 5 minutes), `docs/installing.md` (marketplace, manual, dev install paths).
