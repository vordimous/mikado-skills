---
name: mikado-loop
description: Execute ONE leaf from a Mikado goal end-to-end (pick leaf → implement → commit → mark done) and stop. Designed to be composed with the built-in /loop skill (/loop /mikado-loop) for automated leaf-by-leaf progression with fresh context per leaf, matching the blog's "one leaf per session" recommendation. Use when the user says "do the next leaf", "advance mikado", "/mikado-loop", or wants to drive a long-running Mikado goal without re-issuing the full workflow prompt each time. Requires an existing .mikado/<slug>.md.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent, EnterWorktree, ExitWorktree, TodoWrite, AskUserQuestion
---

# Mikado Loop: Single-Leaf Driver

One invocation = one leaf. When all leaves are done, exits with a signal so `/loop` can stop automatically.

## Composition

- Manual single step: `/mikado-loop`
- Auto-pace through all remaining leaves: `/loop /mikado-loop` (recommended)
- Workaround for harnesses that ignore the exit-signal contract: `/loop 60s /mikado-loop` (caps idle time at 60s per leaf even if dynamic pacing kicks in long delays)

When running inside `/loop`, fresh context each invocation is the point: it matches the Mikado-for-agents recommendation of one leaf per fresh session. Do NOT try to preserve session state between invocations via memory or conversation context; rely only on the graph file and git log.

## Locate the goal

1. `ls .mikado/*.md 2>/dev/null`. If empty, fall through to step 4 (worktree scan). Otherwise continue at step 2.
2. If one file, use it.
3. If multiple, prefer the most recently modified. Display it in the opening status line so the user can override via argument: `/mikado-loop <slug>`.
4. **No `.mikado/` in the current checkout.** The goal may have been started with `Workspace: worktree`, in which case the goal file lives in a sibling worktree, not here. Run `git worktree list --porcelain` and inspect each listed worktree path for a `.mikado/<slug>.md`. If any are found, surface them and stop:

   ```
   No goal in current checkout. Found goal worktrees:
     <path>  goal: <slug>  branch: <branch>
   Run /mikado-loop from inside one of these:
     cd <path>
   ```

   If no sibling worktrees contain a goal file, stop with the original message: `No Mikado goal found. Run /mikado <goal> first.`

## Exit-condition check (before doing any work)

Read the goal file. If:
- **All prerequisites checked AND goal's Status is not `Complete`**: the remaining work is the goal itself. Execute that as Phase 5 of the global `mikado` skill (final implementation), mark Status `Complete`, propose the final commit, and emit `MIKADO_LOOP_DONE: goal '<slug>' complete; run /mikado-mr to open MR` in the final output so `/loop` can detect completion.
- **All prerequisites checked AND goal already Complete**: emit `MIKADO_LOOP_DONE: goal '<slug>' already complete; nothing to do` and exit.
- **Unchecked prereqs exist**: continue to the leaf loop below.

## Preflight (lightweight)

Skip the full Phase 0 from the global skill. This skill runs often, so keep it tight:

1. `git status --short`. Must be empty. If not, stop: "Working tree dirty. Commit or stash before running /mikado-loop."
2. `git rev-parse --abbrev-ref HEAD`. Must not be a protected branch. If it is, stop.
3. **Workspace check.** Read `Workspace` and `Worktree path` from the goal file's front-matter.
   - If `Workspace: worktree`: run `git rev-parse --show-toplevel` and compare against `Worktree path`. If they don't match, the operator launched the loop from outside the goal worktree (e.g. via a nested checkout). Stop with: `Run /mikado-loop from inside the goal worktree at <path>. cd there and re-run.`
   - If `Workspace: in-place` or the field is missing (older goal file): no check; continue.
4. Reconcile if out-of-band commits exist since Base commit (Phase 0.5 of the global skill). If reconciliation would require user input, stop and ask; do not silently alter the graph.
5. Read `Cadence`, `MR strategy`, and `Implementation` from the goal file's front-matter (set in `mikado/SKILL.md` Phase 0.3). The current release fully wires only `Cadence: per-leaf` + `MR strategy: at-goal` + `Implementation: ai-implements`. For any other combination, print a one-line fallback warning naming the field (e.g. "Implementation: coach is not yet wired in this release; falling back to ai-implements") and proceed with the supported behavior. Do not block. Subsequent phases (B for `coach`, C for `per-cluster` / `continuous`, D for sub-MR strategies) wire the remaining combinations.

## Pick ONE leaf

A leaf is a prerequisite with no unchecked children. Strategy:
- Prefer the leaf that unblocks the most parents.
- Tie-break with "lowest estimated risk" (small surface area, well-isolated module).
- If the user supplied `/mikado-loop <leaf-phrase>`, use fuzzy-match to that phrase and confirm before proceeding.

Announce the chosen leaf up front: `Leaf: <text>`.

## Main-session vs. subagent

The prior default was "always delegate." In practice that spawns a subagent for trivial work (pure deletions, graph bookkeeping, companion commits), which costs minutes of overhead per leaf. Refined rule:

**Delegate to a subagent when ANY of these holds:**
- The leaf crosses subsystem boundaries (backend ↔ frontend, different build modules with independent test suites)
- The leaf introduces a new abstraction or public class
- Estimated change is >100 lines of non-mechanical edits
- The leaf requires interactive verification (browser testing, live API calls)

**Prefer main-session for:**
- Single-file deletions with grep-verified zero references
- Constants-only refactors
- Companion commits that are direct byproducts of a just-completed leaf
- Graph-bookkeeping and reconciliation
- Checkoff of a leaf already folded into an earlier commit

When in doubt, delegate. But don't delegate a trivial deletion just because the skill used to say "always."

When delegating, the subagent prompt must include:

1. The full current contents of `.mikado/<slug>.md`
2. The exact leaf text to implement
3. Build/test commands from the goal's front-matter, **leading with the narrowest target** (compile-only or single-class `--tests` filter). Forbid module-level test runs unless explicitly justified.
4. Acceptance criteria:
   - Narrowest verification passes (compile for refactors, targeted test for logic changes)
   - No new failures anywhere touched
   - Change scoped strictly to this leaf
   - Do NOT fix unrelated issues encountered along the way
5. Escape hatch: "If the leaf reveals its own prerequisites you cannot satisfy within its scope, STOP, report the sub-prerequisites as a list, and do not commit anything. The loop driver will record them and pick a deeper leaf next invocation."
6. Commit instructions: the subagent shows `git diff --stat` + `git status --short`, composes a conventional commit message including any ticket key from the branch name, stages the specific changed files (not `git add -A`), commits, and includes any auto-generated byproduct files (OpenAPI specs, API clients) in the same commit. Never `git push`.
7. Any known-flake / environment-quirk notes documented in the project (e.g. `CLAUDE.md`), if any.

When the subagent returns:
- If it reported sub-prerequisites: add them to the graph under the current leaf, commit `mikado: expand '<leaf>' into sub-prerequisites`, and exit. The next `/mikado-loop` invocation will pick the deepest leaf.
- If it completed and committed the leaf: verify the commit landed (`git log -1 --oneline`), then update the graph to check the leaf off.
- If it returned without committing (timeout, mid-test return, silent exit): verify its uncommitted work in the main session. If the acceptance criteria are met (modifications are coherent, narrow verification passes), commit the staged work directly with a conventional message and proceed to checkoff. If the work is incomplete or incoherent, `git restore` the uncommitted changes and re-delegate with a more constrained prompt. Do not spin a fresh subagent on top of half-finished work.

## Handle the commit

Follow the goal's `Commit strategy` front-matter:
- `unset`: default to `folded` when the invocation is part of `/loop /mikado-loop` (continuous auto-pacing implies the user doesn't want a graph-update commit after every leaf). Default to `separate` for explicit single-step `/mikado-loop` invocations. Write the inferred choice to the front-matter and proceed without asking.
- `separate`: the subagent commits the leaf's code change; this skill then marks the leaf checked in `.mikado/<slug>.md` and commits that as `mikado: check off '<leaf>'`.
- `folded`: mark the leaf checked in `.mikado/<slug>.md` before the subagent stages, so both changes land in one commit.

Mark the leaf with `[x]` in the checklist and append ` ✓` to its Mermaid node label.

## Open questions during the loop

If the graph lists an open question with a clearly marked default (e.g. "Open Q #X → suggested default: option (a)") or the goal file's Notes section contains a resolution for it, proceed with that default and note the choice in the leaf's commit body. Do not stop the loop to re-confirm.

Stop the loop and surface to the user only when:
- The graph marks the question as needing input ("pending decision" / "requires user input")
- The options have materially different blast radii and no default is recorded
- The question affects a leaf's commit in ways the user hasn't pre-approved

A "stop on genuine ambiguity" rule beats "stop on any open question"; the latter interrupts end-to-end flow for decisions the user already made in the plan.

## Exit-signal output

End every invocation with one of these status lines on its own line, so `/loop` can parse:

- `MIKADO_LOOP_ADVANCE: leaf '<leaf>' completed; <N> prereqs remaining`
- `MIKADO_LOOP_EXPAND: leaf '<leaf>' expanded into <M> sub-prereqs; <N> total remaining`
- `MIKADO_LOOP_DONE: goal '<slug>' complete; run /mikado-mr to open MR`
- `MIKADO_LOOP_BLOCKED: <reason>` (dirty tree, reconcile conflict, genuine open question with no default, etc.)

### Pacing contract for `/loop`

The dynamic `/loop` self-pacing heuristic defaults to long idle delays (20 to 30 minutes) for indeterminate work. Mikado leaves are determinate: there is either more graph work or there is not. The exit signal tells you which. Honor this:

- `MIKADO_LOOP_ADVANCE` / `MIKADO_LOOP_EXPAND` → fire the next iteration immediately. Do not insert an idle delay. If the harness requires a `delaySeconds` value, use the minimum it accepts (typically 60s).
- `MIKADO_LOOP_DONE` / `MIKADO_LOOP_BLOCKED` → stop. Do not schedule another iteration.

Include a clear "stop looping" signal in the final output when emitting `DONE` or `BLOCKED` so the orchestrator parses the intent unambiguously.

Alongside the status line, emit a short summary block so the user can see progress at a glance:

```
Commits this iteration:
  <sha>  <subject>
  <sha>  <subject>
Files touched: <count> (<primary subsystem>)
Mode: main-session | subagent
Open questions resolved: <list or "none">
Scope-leak flags: <list or "none">
```

Keep it terse. The summary is read fast between iterations; long reports get ignored.

## Operating rules

- **Commit the leaf; never push.** Stage only the changed files (not `git add -A`), commit with a conventional message that includes any ticket key from the branch name, and stop there. The user owns `git push` and destructive history operations.
- **Never run more than one leaf per invocation.** That defeats the purpose of the loop driver.
- **Never skip the reconciliation check.** Out-of-band commits between invocations are normal when the user is reviewing; the graph must stay honest.
- **Delegate when the leaf's surface warrants it; stay inline when it doesn't.** See the main-session vs subagent section. The old "always delegate" rule was too absolute.
- **Stop on ambiguity, not on every open question.** Proceed with recorded defaults from the graph's open questions. Stop only for genuine unresolved decisions.
- **Watch for scope leaks.** If a leaf solves its prerequisite by reaching into an adjacent prerequisite's territory (FE display shim that touches backend semantics, migration logic leaking into the FE), surface it in the iteration summary under "Scope-leak flags" and either fold the leaves or expand the graph to reflect the actual work.

## Related skills

- `mikado`: start the goal this skill advances
- `mikado-mr`: call after `MIKADO_LOOP_DONE` to open the MR
- `loop` (built-in): the orchestrator. Run `/loop /mikado-loop` for auto-pacing.
- `ralph-loop:ralph-loop`: NOT recommended here. Ralph re-feeds the same prompt into the same session, which accumulates context; that violates the fresh-session-per-leaf principle. Use the built-in `/loop` instead.
