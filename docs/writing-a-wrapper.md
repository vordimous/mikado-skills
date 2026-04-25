# Writing a project-specific wrapper

The three skills in the `mikado` plugin are project-agnostic by design. They know how to drive the method but they don't know your build commands, your ticket key format, or which tests are flaky.

A **wrapper skill** captures all of that. It lives in your project at `.claude/skills/mikado-<projectname>/SKILL.md` and tells Claude "read the global mikado skill first, then apply these adaptations."

## When to write one

Write a wrapper when any of these are true:

- Your build command isn't auto-detectable (custom Gradle/Maven invocations, monorepo subprojects, exotic toolchains).
- You have specific test-narrowing patterns the agent should prefer over module-level test runs.
- You have known flakes the agent rediscovers every session if you don't tell it.
- Your branch names follow a ticket-key convention that should appear in commits and MR titles.
- Your MR template has required sections that need to be populated from the graph.

If none of these apply, the global `/mikado` works fine without a wrapper. You can always add one later.

## What goes in a wrapper

The [wrapper-template](../examples/wrapper-template/) directory has a runnable skeleton. The structure to follow:

```markdown
---
name: mikado-<projectname>
description: <one-line description that says when to use this wrapper>
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent, EnterWorktree, ExitWorktree, TodoWrite, AskUserQuestion
---

# Mikado Method — <Project> Wrapper

## How this skill runs

1. Read the global `mikado` skill first.
2. Layer the project adaptations below on top.

## <Project> adaptations

### Build and test commands
<narrowest-first list of compile / test commands>

### Known flakes and environment quirks
<list of flaky tests, required env vars, etc.>

### Codegen byproducts
<which auto-generated files belong with the leaf commit>

### Ticket and branch
<branch-name pattern, where to extract the ticket key>

### Cross-module impact warnings
<which paths require ⚠️ warnings in the graph>

### Commit message conventions
<scope vocabulary, examples>

### MR shape recommendation
<single | clustered | stacked, and why>

### Subagent delegation hints
<what to put in subagent prompts so they don't rediscover conventions>
```

Keep it concise. Long wrappers are skipped; tight wrappers get followed.

## What to leave OUT

A wrapper is **not** the place for:

- The full Mikado method. The global skill already has it. Re-stating it just creates drift.
- Project-specific code style or linting rules. Those belong in `CLAUDE.md` or a separate skill.
- Anything the agent can derive from the project structure (e.g. "this is a Java project" — `build.gradle` already says so).
- Inspirational text about why the method is good. Reviewers don't read it; agents waste tokens on it.

If you find yourself explaining the method, you're writing the wrong skill. Cut it.

## Naming

Convention: `mikado-<projectname>`. Reasons:

- Discoverable. `mikado-` prefix shows up next to `mikado`, `mikado-loop`, `mikado-mr` in any skill listing.
- Doesn't shadow. The global skills remain `/mikado`, `/mikado-loop`, `/mikado-mr`. The wrapper is a sibling, not a replacement.
- Greppable. `git grep mikado-` finds every wrapper across an org's repos.

## Composition with `/loop`

`/mikado-loop` reads `.mikado/<slug>.md`, picks a leaf, implements it, commits, marks it done, and exits with a status line. It does **not** know about your wrapper — it follows the global rules.

That's intentional. The wrapper's adaptations live in the leaf-implementation step (Phase 4d in the global skill), which is delegated to a subagent. The subagent prompt should include your wrapper's "Subagent delegation hints" verbatim — that's how project conventions reach the leaf without `/mikado-loop` itself needing to know about them.

## Maintaining a wrapper

Treat wrappers as code. They drift:

- New flakes appear; old ones get fixed.
- Test commands change with build-system upgrades.
- Branch conventions evolve.

When a session uncovers a project quirk that the wrapper missed, add it to the wrapper. The next session should not have to rediscover the same thing.

## Real-world example

The X1 platform wrapper (the one this plugin was extracted from) is structured exactly like the template above. It has:

- Six narrow build/test commands plus three commit-boundary commands (Gradle multi-module).
- Four known flakes including a Spring-context loading quirk and a pre-existing TypeScript error.
- Cross-module warnings for `common/`, `integrations/shared/`, `x1-ui/src/api/`, and `openapi-spec.json`.
- Ticket key extraction for `X1-\d+` from branch names.
- Conventional commit scope vocabulary: `api`, `ui`, `adapter`, `corsair`, `migration`.
- MR shape: single-MR default with clustered-MR preference for goals spanning ≥3 subsystems (X1 squash-merges).
- Subagent hints that codify the test-narrowing rules so subagents don't run module-level suites by default.

That single file replaces ~30 minutes of session boilerplate per Mikado goal in that repo.
