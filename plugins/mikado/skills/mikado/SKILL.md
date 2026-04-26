---
name: mikado
description: Structured Mikado Method workflow for substantial codebase changes (refactors, migrations, dependency swaps, architectural shifts) that touch many places. Use when the user invokes /mikado, says "apply the mikado method" or "mikado this", or describes a goal clearly too big to implement in one naive pass. Builds a prerequisite graph in .mikado/<slug>.md by running naive experiments in isolated git worktrees, then executes leaf prerequisites on the feature branch with small commits. Do NOT use for single-file tweaks, obvious bug fixes, or changes where the implementation path is already clear.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent, EnterWorktree, ExitWorktree, TodoWrite, AskUserQuestion
---

# Mikado Method

The Mikado Method makes large codebase changes by discovery, not planning. The loop:

1. State a concrete goal.
2. Attempt it naively in an isolated worktree.
3. Whatever breaks reveals prerequisites.
4. Discard the worktree; record prerequisites in a Mermaid graph.
5. Pick a leaf prerequisite, implement it on the feature branch, commit.
6. Repeat until the goal can be implemented cleanly.

The prerequisite graph is the artifact. Experiment code is throwaway.

## Inputs

Invoked as `/mikado <goal>` or `/mikado <path-to-spec.md>`. The argument is either:

- **A direct prose goal:** `/mikado remove Redisson and migrate to Lettuce`. The skill writes a minimal goal file and discovers all prerequisites from the naive experiment.
- **A path to a spec file (preferred for substantive goals):** `/mikado docs/specs/remove-redisson.md`. The skill ingests the file in Phase 0.75 and seeds the graph with **anticipated** prerequisites that the naive experiment then confirms, refines, or invalidates.

Plans from other sessions (Claude plan-mode output, ChatGPT, hand-written) must be saved to a file first. The skill does not accept inline multi-line plans; that's deliberate to keep the graph reproducible from disk state.

Derive a short kebab-case slug from the goal for filenames. Example: "Remove Redisson and migrate to Lettuce" → `remove-redisson`. When ingesting a spec file, prefer the file's basename (minus extension) as the slug if it's already kebab-case.

### Spec file schema (loose)

The skill scans for these headings case-insensitively. Synonyms in parentheses; any one match is enough. Sections not found are skipped without complaint.

- **Goal** (Goal Statement, Objective, Summary): the goal sentence. Fall back to the document's first H1 if absent.
- **Why** (Motivation, Background, Context): copied verbatim into the goal file's `Notes and learnings` as the opening paragraph.
- **Acceptance** (Acceptance Criteria, Done When, Success Criteria): copied into a new `## Acceptance` section in the goal file. Reviewers and the leaf loop both consult this.
- **Tasks** (Steps, Plan, Subtasks, Prerequisites, Work Items, Implementation Plan): each top-level bullet becomes an **anticipated** prerequisite (dashed orange `classDef anticipated` in the graph). Nested bullets stay as sub-notes on the parent prereq, not as separate nodes; expansion happens through experiments, not indentation.
- **Open Questions** (Decisions Needed, TBD, Unknowns): each item becomes an entry in `Notes and learnings` under an `### Open questions` subsection, marked `(pending decision)` until resolved.

If none of these headings match and the file is non-empty, treat its full contents as the goal statement.

## Operating rules (read before acting)

- **Safe git operations allowed; never push or rewrite history.** Claude owns local git state while the goal is in progress: create feature branches, stage specific files, commit, revert, cherry-pick, stash, restore, rename or delete merged branches. Claude does NOT push, pull, merge, rebase, reset, force-delete branches, or clean. Show `git status --short` and `git diff --stat` before each commit so the user can interject. `EnterWorktree` / `ExitWorktree` are allowed because they don't mutate the feature branch.
- **Resist fixing during experiments.** The naive attempt's only job is to surface failures. If you start fixing things in the worktree, you lose the signal. Exit and record.
- **One leaf per commit.** Do not bundle unrelated prerequisites.
- **Update the graph before touching code.** The graph should always lead the code, not trail it.
- **Revert is free.** If an experiment needs more than a handful of changes before you can run tests, you're probably fixing instead of experimenting. Exit and re-scope the leaf.
- **Use the goal's Testing plan to scope verification.** Phase 0.4 derives a tiered plan (fast / targeted / regression, plus optional affinity rules) from existing repo context and gets the user's signoff. The agreed plan is the `## Testing plan` section of the goal file. Default to the fast tier after every leaf. Escalate to the targeted tier when the leaf changes logic (not pure refactor) or an affinity rule fires. The regression tier runs at Phase 5 only, never per-leaf. Treat unsanctioned module-level runs in the leaf loop as a yellow flag requiring explicit justification.
- **Fold codegen byproducts into the producing leaf.** Auto-generated files (OpenAPI specs, API clients, lockfiles) regenerate deterministically from their source. When a leaf causes them to regenerate, stage and include them in the same commit as the leaf; do not create separate `chore: regenerate X` commits. The exception is when codegen regenerates on its own schedule (unrelated churn); that's the only case where a standalone regen commit is correct.
- **Subagent timeout recovery.** If a subagent returns without committing (timed out mid-test, returned control prematurely, hit a silent error), verify its work in the main session before redelegating. If acceptance criteria are met (modified files are coherent, narrow test suite is green), commit the staged work directly. If the work is incomplete or incoherent, revert the uncommitted changes and re-delegate. Do not spin a fresh subagent on top of half-finished work.

## Agent-safe git: six properties this skill upholds

Adapted from [GitButler's agent-safety framework](https://blog.gitbutler.com/agentic-safety). Useful as a check when adding a new operation or loosening a rule.

1. **Task isolation.** One goal = one feature branch (and, for experiments, one throwaway worktree). Never mix two goals on the same branch.
2. **Clear branch boundaries.** The skill never silently switches branches. Branch creation is allowed; branch switching is not. If a goal needs to move to a different base, stop and ask.
3. **Explicit commit selection.** Stage specific files; never `git add -A` or `git add .`. The user sees each commit's diff stat before it lands.
4. **Easy pre-push review.** The user owns `git push`. Every leaf commit is local until the user decides to publish. MR creation (`/mikado-mr`) also requires explicit confirmation.
5. **Cheap rollback.** `git revert`, `git restore --staged`, and `git stash push`/`pop`/`apply` are all allowed so the skill can undo its own work without human friction. Destructive undo (`git reset --hard`, `git clean`, `git branch -D`, `git stash drop/clear`) stays denied; those lose data silently.
6. **Cross-branch damage prevention.** `git checkout -- <file>` is denied (use `git restore` instead, which is explicit about what it touches). `git switch -C` is denied (force-overwrites a branch pointer). `git clean` is denied (kills untracked files).

## Phase 0: Preflight

Run in sequence:

1. `git status --short`. Must be empty. If not, ask the user to stash or commit before continuing.
2. `git rev-parse --abbrev-ref HEAD`. Must not be a protected branch (`main`, `master`, `develop`, `release/*`, `hotfix/*`). If it is, ask the user to create a feature branch first.
3. Identify the build system(s) so Phase 0.4 has something to derive from:
   - `build.gradle` / `settings.gradle` → Gradle
   - `package.json` → npm/pnpm/yarn (check lockfile)
   - `pyproject.toml` / `Pipfile` → Python (ask user which runner)
   - Polyglot repos: ask the user which subsystem the goal targets.
4. Ensure `.mikado/` exists; create it if not.

## Phase 0.3: Goal configuration

Three configuration choices control rhythm and integration for this goal. Surface the **ethos preamble** first (so the user understands the discipline the choices sit on top of), then prompt for the three values, then persist them to the goal file's front-matter for Phase 4, mikado-loop, and mikado-mr to consult.

If the goal file already exists (Phase 0.5 detects a resume), skip the preamble and the prompts entirely. Read the values from the existing front-matter and proceed.

### Ethos preamble

Print verbatim before the prompts:

```
Mikado in 30 seconds:
  1. The graph leads the code; we never write a leaf without first observing
     its prerequisite as a real failure.
  2. Revert is free. The naive experiment is a sensor, not a draft.
  3. One leaf per commit. The graph is the durable artifact, not the code.

Three configuration choices follow. They control rhythm and integration,
not the discipline above.
```

### Prompts

Ask three questions, in this order. Annotate each option with its release status: **(supported)** for the fully-wired path, **(experimental, falls back to <default> in this release)** for options whose downstream behavior is deferred to Phase B/C/D.

1. **Cadence** (`continuous` | `per-leaf` | `per-cluster`): where the leaf loop pauses for review.
   - `per-leaf` **(supported)**: pause after every leaf commit. Best for review-heavy workflows.
   - `per-cluster` **(experimental, falls back to per-leaf in this release)**: pause after the last leaf in each graph cluster (subtree under a direct goal-child). Natural cohesion boundary; full behavior arrives in Phase C.
   - `continuous` **(experimental, falls back to per-leaf in this release)**: no per-leaf or per-cluster pauses; only stop for sub-prereq discoveries, test failures requiring fix-vs-revert, or goal completion. Best paired with `MR strategy: at-goal` and `/loop /mikado-loop`. Full behavior arrives in Phase C.
2. **MR strategy** (`per-leaf` | `per-cluster` | `at-goal`): when the skill opens MRs.
   - `at-goal` **(supported)**: single MR at goal completion targeting main/develop. The default for small-to-medium goals.
   - `per-cluster` **(experimental, falls back to at-goal in this release)**: each cluster gets its own sub-branch and sub-MR targeting the goal branch. Full behavior arrives in Phase D.
   - `per-leaf` **(experimental, falls back to at-goal in this release)**: each leaf gets its own sub-branch and sub-MR targeting the goal branch. Full behavior arrives in Phase D.
3. **Implementation** (`ai-implements` | `coach`): who writes the code.
   - `ai-implements` **(supported)**: Claude writes, runs tests, and commits each leaf.
   - `coach` **(experimental, falls back to ai-implements in this release)**: Claude builds the graph, runs the naive experiment, runs the testing-plan tiers, and prepares a research brief per leaf. The user writes and commits the code. Full behavior arrives in Phase B.

When `Implementation: coach` is selected, omit `Cadence: continuous` from the cadence options (coach mode requires user input at each leaf, so unbroken running is incoherent).

### Persistence

Write the agreed values to the goal file's front-matter (added in Phase 1):

```
**Cadence:** <value>
**MR strategy:** <value>
**Implementation:** <value>
```

The values are read by:
- Phase 4 (leaf loop) for cadence and implementation branching.
- mikado-loop for cadence honoring under `/loop /mikado-loop`.
- mikado-mr for sub-MR targeting and final-MR shape.

### Phase A scope (current release)

The configuration prompts are wired in, but only one combination is fully implemented in the current release:

- **Fully supported:** `Cadence: per-leaf` + `MR strategy: at-goal` + `Implementation: ai-implements` (today's behavior).
- **Captured in front-matter, prompt allowed, downstream behavior not yet wired:** every other combination.

When the leaf loop, mikado-loop, or mikado-mr encounters a combination outside the supported scope, print a one-line warning at the first relevant boundary (e.g. "Cadence: per-cluster is not yet wired; falling back to per-leaf for this release") and proceed with today's behavior. Subsequent phases activate the remaining combinations.

## Phase 0.4: Testing plan

The leaf loop runs verification after every leaf. Without an explicit plan for what runs when, the skill either burns minutes on broad suites per leaf or under-tests and ships regressions. Derive a tiered plan from existing repo context, show it to the user, and get signoff before any experiment runs.

### Sources to consult (in order)

1. `CLAUDE.md` (project root and any nested), looking for sections named "Testing", "Test", "Verification", "CI", or similar.
2. `README.md` (project root), same heading patterns.
3. Build manifests: `package.json` scripts, `build.gradle` / `settings.gradle` task names, `pyproject.toml` `[tool.*]` sections, `Makefile` targets.
4. Ecosystem convention. Use only when nothing more specific was found. Examples: Gradle → `./gradlew compileJava` (fast), `./gradlew test` (regression). npm → `npm run typecheck` (fast), `npm test` (regression). pytest → `pytest --collect-only` (fast), `pytest` (regression).

If a resumed goal already has a `## Testing plan` section in `.mikado/<slug>.md`, read it from there instead of re-deriving. The user signs off, amends, or replaces the stored plan.

### Plan structure

A plan has three tiers. Every plan must populate at least the fast and regression tiers. Targeted is optional but strongly recommended for any project with a non-trivial test suite.

- **fast** runs after every leaf by default. Compile, typecheck, or lint. Sub-30s. Cheap enough that "always run" is the right policy.
- **targeted** runs on leaves that change logic, scoped to the affected module/class. 1-5min. The leaf loop selects this tier when the leaf changes business logic, not when it's a pure refactor.
- **regression** is the full relevant suite. Runs at Phase 5 (commit boundary before MR), never per-leaf.

Optionally, an **affinity map** ("if a leaf touches `src/billing/`, escalate to targeted with filter `Billing*`") and a **skip list** for long-running tasks the user explicitly wants kept out of the leaf loop (e2e, integration suites tied to external services, etc.).

### Signoff prompt

Show the derived plan and ask the user to confirm or amend:

```
Testing plan (derived from <source summary>):

  fast       <command>            (per-leaf default)
  targeted   <command>            (logic changes; supply pattern)
  regression <command>            (Phase 5 only)

  Affinity:
    <bullet>  |  (none specified)

  Skip in the leaf loop:
    <bullet>  |  (none)

  Source: <CLAUDE.md section / README / build files / convention>
  Confidence: <high | medium | low>

Sign off, amend, or add instructions:
```

The user can confirm as-is, edit any line, add affinity rules, or override the source-derived defaults entirely. Whatever they sign off becomes the durable plan.

The agreed plan is written into the goal file in Phase 1 as the `## Testing plan` section. For resumed goals, write any amendments back to that section and propose a single commit (`mikado: update testing plan`) if the section changed.

Confidence is the skill's honest read of how well the derived plan matches reality:
- **high** explicit testing documentation exists and the plan was lifted from it
- **medium** build manifests provided commands but no documented strategy
- **low** pure ecosystem convention; no project-specific signal

Low confidence is not a blocker. It's a signal to the user that this is the moment to add testing instructions if they have any.

## Phase 0.5: Resume detection and reconciliation

If `.mikado/<slug>.md` already exists, this is a resumed session. Before picking a leaf, **reconcile the graph with the actual branch state**. The user may have committed out-of-band since last session.

Phase 0.3 (ethos preamble and configuration prompts) is suppressed on resume; the values for `Cadence`, `MR strategy`, and `Implementation` are read from the existing front-matter. If the user wants to change a value mid-goal, they edit the front-matter directly and the skill picks up the new value at the next leaf boundary.

1. Read `.mikado/<slug>.md`. Note the `Base commit`, `Commit strategy`, `Cadence`, `MR strategy`, and `Implementation`.
2. `git log --oneline <base-commit>..HEAD`. List every commit since the goal started.
3. For each commit, match its subject against the graph's prerequisite list:
   - If a commit looks like it implemented a still-unchecked prereq, flag it.
   - If a commit doesn't match any prereq, note it as an out-of-band change.
4. Show the user:
   - Current status line and remaining unchecked prerequisites
   - Commits since start, with matches/mismatches highlighted
   - Any suspected unchecked-but-implemented leaves
   - The next suggested leaf
5. Ask: resume as-is, update graph to mark flagged prereqs done, or start over?

If the user resumes, apply any confirmed check-offs to the graph before Phase 4, and propose a single reconciliation commit (`mikado: reconcile graph with branch state`) if the graph changed.

If the branch `HEAD` no longer contains the `Base commit` in its ancestry (force-pushed, rebased onto a different base), stop and ask the user how to proceed. Do not silently mutate the graph.

## Phase 0.75: Plan ingestion (new goals only)

Skip this phase entirely if the argument is direct prose, or if Phase 0.5 detected a resume. Run only when the argument resolves to an existing markdown file.

1. Read the file.
2. Walk the document looking for the section headings listed in the [spec file schema](#spec-file-schema-loose). Match case-insensitively. Stop scanning a section when the next H2 (or higher) heading is reached.
3. For each section found, capture the content for use in Phase 1's template:
   - **Goal** sentence → goal statement (first H1 fallback if absent)
   - **Why** body → opening paragraph of `Notes and learnings`
   - **Acceptance** bullets → `## Acceptance` section
   - **Tasks** top-level bullets → anticipated prereqs (one node per bullet, named `P1`, `P2`, ...)
   - **Open Questions** items → `Notes and learnings` → `### Open questions` subsection
4. If the document is non-empty but no recognized section was found, log "No recognized headings; treating full document as goal statement." and continue.
5. Show the user a one-line summary of what was ingested:
   ```
   Ingested plan from <path>: <N> anticipated prereqs, <M> open questions, <K> acceptance criteria.
   ```
   No confirmation prompt; ingestion is non-destructive (the naive experiment in Phase 2 still runs and corrects any wrong anticipations).

Anticipated prereqs are not the same as observed prereqs. They start the graph with a hypothesis that the naive experiment will validate. The Mermaid graph distinguishes them via `classDef`: solid green for observed, dashed orange for anticipated.

## Phase 1: Record the goal

Write `.mikado/<slug>.md` using this template (the outer four-backtick fence is only here so the inner three-backtick mermaid fence renders; write the inner version to the file):

````markdown
# Mikado Goal: <goal statement>

**Slug:** <slug>
**Started:** <ISO 8601 date>
**Ticket:** <Jira/issue key if discoverable, else omit>
**Source plan:** <relative path to ingested spec file; omit this line entirely if direct prose goal>
**Skill version:** <semver of the mikado plugin at goal-creation time, e.g. 0.2.0>
**Cadence:** <continuous|per-leaf|per-cluster>
**MR strategy:** <per-leaf|per-cluster|at-goal>
**Implementation:** <ai-implements|coach>
**Commit strategy:** <unset; to be set on first leaf: separate|folded>
**Base commit:** <SHA of feature branch HEAD when goal started>

## Acceptance

<one bullet per criterion from the spec file's Acceptance section. If direct prose goal or no Acceptance section was ingested, write `_(none specified; populate as the goal progresses)_`>

## Testing plan

**Source:** <e.g., "CLAUDE.md > Testing section + package.json scripts" or "Inferred from build.gradle (no testing docs found)">
**Confidence:** <high | medium | low>

### Tiers

- **fast** (per-leaf default): `<command>` — <one-line purpose, e.g., "compile-only typecheck">
- **targeted** (logic changes): `<command>` — <one-line purpose; often takes a pattern argument>
- **regression** (Phase 5 only): `<command>` — full suite

### Affinity

<bullets in the form `touched path → tier with filter`, or `_(none specified; rely on tier defaults)_`>

### Skip in the leaf loop

<bullets for long-running tasks the user excluded, or `_(none)_`>

### Flaky tests

<bullets in the form `<test name>: <symptom>`, populated as discovered, or `_(none yet)_`>

## Status
Discovering prerequisites.

## Mikado Graph

```mermaid
graph TD
  G((Goal: <short restatement>))
  <if anticipated prereqs were ingested in Phase 0.75, add `G --> P1[<task 1>]`, `G --> P2[<task 2>]`, etc., one per ingested task>

  classDef observed fill:#e8f5e9,stroke:#2e7d32
  classDef anticipated fill:#fff3e0,stroke:#e65100,stroke-dasharray: 5 5
  <if anticipated prereqs were ingested, add `class P1,P2,... anticipated` listing all ingested node IDs>
```

## Prerequisites

<one entry per anticipated prereq, formatted as `- [ ] **P<N>.** <task text> (anticipated)`. If none ingested, write `_(none yet; will populate after the naive experiment)_`>

## Notes and learnings

<if a Why section was ingested, paste its content as the opening paragraph; otherwise leave blank>

<if Open Questions were ingested, add an `### Open questions` subsection with one bulleted entry per question, each tagged `(pending decision)`>
````

`Base commit` is the `git rev-parse HEAD` at the moment of goal creation. It anchors resume-time reconciliation (Phase 0.5) against the commits produced since.

`Skill version` records the mikado plugin's semver at goal-creation time. Read it at resume to detect schema mismatches: if the current plugin version is newer than the stored value, log "goal file written with v<old>; current plugin is v<new>" and proceed. If a future migration is required, the version field is the trigger.

`Commit strategy` stays `unset` until the first leaf is completed; on that leaf, ask the user whether to keep graph updates as a separate commit (`separate`) or fold them into the leaf's commit (`folded`), then update this field. For every subsequent leaf, follow the recorded strategy without re-asking.

Commit: `mikado: start goal '<slug>'`. Show the diff summary, then commit directly.

## Phase 2: Naive experiment

1. `EnterWorktree`. Creates an isolated worktree off the current branch.
2. Inside the worktree, attempt the goal the most obvious way. Do not try to be clever about prerequisites. The experiment is a sensor, not an implementation.
3. Run the regression tier from the goal file's `## Testing plan` (the naive experiment wants maximum surface, not minimum). Capture ALL failures: compile errors, test failures, runtime errors, lint warnings that would gate a commit. Skip-listed tasks remain skipped; the user already opted them out for the loop, and the experiment respects that.
4. Do NOT attempt to fix anything. Collect signal only.
5. `ExitWorktree`. Discards all changes.

If the naive attempt passed on first try, the goal is itself a leaf. Jump to Phase 5.

## Phase 3: Analyze failures into prerequisites

This is where human judgment is most valuable. For each failure cluster, ask:

- **What is the root cause?** One missing parameter in `createUser(..)` might cause 50 test failures. That is one prerequisite, not 50.
- **Is this prerequisite actionable?** "Fix null pointer in BillingService" is actionable. "Make it faster" is not; decompose further.
- **Does this prerequisite obviously have its own prerequisites?** If so, note them as children, but do NOT expand them yet. Expansion happens through experiments, not speculation.

### Three-way reconciliation (when a plan was ingested)

If Phase 0.75 seeded the graph with anticipated prereqs, every failure cluster falls into one of three buckets:

- **Confirmed:** the failure matches an anticipated prereq. Promote that node from `anticipated` to `observed` (move its ID from the `class ... anticipated` line to a `class ... observed` line). Keep the same node ID so existing references stay valid.
- **New:** the failure has no matching anticipated prereq. Add a new `observed` node with a fresh ID.
- **Stale:** an anticipated prereq did not surface as a failure. Keep it as `anticipated` for now and add a note in `Notes and learnings` flagging that the naive experiment did not exercise it. Stale prereqs often surface only after observed leaves land; revisit during the leaf loop. If after the entire goal is complete a stale prereq was never exercised, it was probably wrong, and the retro should call it out.

Record the reconciliation tally in `Notes and learnings` once Phase 3 completes:

```
Reconciliation: <X> confirmed, <Y> new, <Z> stale.
```

If no plan was ingested (direct prose goal), skip reconciliation. All prereqs are observed by definition; just add them to the graph and the `class ... observed` line.

### Updating the graph

Update `.mikado/<slug>.md`:
- Add new observed prereqs to the Mermaid graph; promote confirmed anticipated prereqs by moving their IDs between `class` lines
- Add each new prereq to the checklist below the graph (drop the `(anticipated)` tag from any prereq that gets promoted)
- Use short imperative phrases: "Extract UserService from RequestHandler"

Example (mixed graph after reconciliation):

```mermaid
graph TD
  G((Goal: Remove Redisson))
  G --> P1[Replace RedissonClient beans with Lettuce]
  G --> P2[Migrate distributed locks]
  G --> P3[Drop Redisson cache manager]
  P2 --> P2a[Choose lock replacement]

  classDef observed fill:#e8f5e9,stroke:#2e7d32
  classDef anticipated fill:#fff3e0,stroke:#e65100,stroke-dasharray: 5 5
  class P1,P2,P2a observed
  class P3 anticipated
```

Here P1, P2, and P2a were observed in the naive experiment; P3 was anticipated from the plan and didn't surface yet (so it stays anticipated until a re-experiment after the observed leaves land).

Commit: `mikado: record prerequisites from naive attempt`. Show the diff summary, then commit directly.

## Phase 4: Leaf loop

Repeat until every prerequisite is checked.

Before entering the loop, read `Cadence`, `MR strategy`, and `Implementation` from the goal file's front-matter (set in Phase 0.3). The current release fully wires only the `per-leaf` + `at-goal` + `ai-implements` combination. For any other combination, print a one-line warning at the first relevant boundary (e.g. "Cadence: per-cluster is not yet wired in this release; falling back to per-leaf") and proceed with the fully-supported behavior. Do not block; do not re-prompt. Subsequent phases (B for `coach`, C for `per-cluster` / `continuous`, D for sub-MR strategies) wire the remaining combinations.

### 4a. Pick a leaf

A leaf has no unchecked children. If multiple leaves exist, prefer one that unblocks the most parents or carries the least risk.

Use `TodoWrite` to mirror the current leaf (and its siblings) as in-session todos. Do not mirror the entire graph; the graph file is the durable record.

### 4b. Size the leaf: main vs. subagent

Delegate to a subagent via the `Agent` tool when ANY of the following are true:
- The leaf crosses subsystem boundaries (backend ↔ frontend, or different build modules with independent test suites)
- The leaf introduces a new abstraction or public class that will ripple to many callers
- Estimated change is >100 lines of non-mechanical edits (pure type-widening edits across many files, even if file count is high, are mechanical and often faster inline)
- The leaf requires interactive verification (browser test, live API call) that benefits from a fresh context summarizing the test plan
- You have already implemented two non-trivial leaves in this session (context saturation risk)

Implement in the main session otherwise. Specifically prefer main session for:
- Single-file deletions with verified zero references
- Constants-only refactors
- Graph bookkeeping and reconciliation
- Companion commits that are direct byproducts of a just-completed leaf

File count alone is a weak signal; 10 files of single-line type swaps may be faster inline than one 200-line state-management change. Judge by cognitive load, not LOC.

When delegating, the subagent prompt must include:
1. The full current contents of `.mikado/<slug>.md`
2. The specific leaf to implement, verbatim
3. Acceptance criteria: "fast tier from the Testing plan passes; targeted tier passes for the affected scope (per affinity rules, if any); no new failures anywhere; change is scoped strictly to this leaf"
4. A directive: "If this leaf turns out to have its own prerequisites (new compile/test errors you can't resolve within this leaf's scope), STOP and report them as sub-prerequisites. Do not try to fix them."

### 4c. Experiment if uncertain

If it's not obvious the leaf will apply cleanly, run a worktree experiment scoped to the leaf (repeat Phase 2 logic for the leaf). Failures become sub-prerequisites; record them in the graph and pick a deeper leaf.

### 4d. Implement cleanly on the feature branch

On the main tree, make the focused change. Run the appropriate tier from the goal file's `## Testing plan`:

- **Fast tier** by default. Always runs after every leaf.
- **Targeted tier** when (a) the leaf changes logic rather than mechanical refactor, or (b) an affinity rule in the plan matches a path the leaf touched. Apply the narrowest filter the targeted command supports (single class, single test pattern) so the run stays focused.
- **Regression tier** is reserved for Phase 5; do not run it per leaf.

If the leaf touches a path with no affinity rule and changes logic, propose escalating to targeted with a specific filter and confirm with the user before adding a permanent affinity rule to the plan.

**Flaky-test rule.** If a test fails on first run, re-run *only that one test* once. If it passes on the second run, append it to the goal file's `## Testing plan` → `### Flaky tests` list (`<test name>: <symptom>`) and continue. If it fails again, treat it as a real failure and either fix within this leaf (if trivial and in scope) or record a new sub-prerequisite and revert. Do not re-run more than once; chasing flakes at the leaf level burns the session.

### 4e. Commit

Commit the leaf directly:
- Show `git diff --stat` and `git status --short` first so the user sees what's about to commit
- Compose a conventional commit message including any ticket key discoverable from the branch name
- Stage the specific files changed (not `git add -A`) and commit
- If the leaf triggered regeneration of auto-generated files (OpenAPI specs, API clients, lockfiles, etc.), stage and include those in the same commit. Do not create a separate chore commit for codegen byproducts.

Mark the leaf checked in `.mikado/<slug>.md`:
- Checkbox `[x]` in the checklist
- Append ` ✓` or strikethrough to the Mermaid node label (e.g. `P2a[Choose lock replacement ✓]`)

Consult the `Commit strategy` front-matter field:
- If `unset`: ask the user once. Separate graph commit (`mikado: check off '<leaf>'`) or folded into the leaf's commit? Then write the choice (`separate` or `folded`) to the front-matter and proceed.
- If `separate`: commit the leaf, then a follow-up graph commit.
- If `folded`: single commit that includes both the code change and the graph update.

Subsequent leaves must honor the recorded strategy without re-asking. Never push; the user owns remote operations.

## Phase 5: Finish the goal

When all prerequisites are checked, attempt the original goal:

1. Optional final worktree experiment to confirm nothing regressed.
2. Implement on the feature branch.
3. Run the regression tier from the goal file's `## Testing plan`. Capture any new flakes in the plan's flaky tests list before opening the MR.
4. Propose the final commit: something like `<type>: <goal>` with the ticket key.
5. Update the graph status to `Complete`, dated.

Offer a short retro:
- Any prerequisites that turned out unnecessary?
- Any discovered late (suggesting the naive attempt under-sampled the failure surface)?
- What you'd do differently next time.

Propose an MR/PR body summarizing the graph and commits. Do NOT open the MR; tell the user the command to run.

### MR shape: single, clustered, or stacked

The Mikado graph is itself a dependency DAG; it often maps cleanly to one of three MR shapes. `/mikado-mr` supports all three; pick based on the goal and the team's merge policy.

- **Single MR (default).** One MR per goal, covering every leaf. Fine for small-to-medium goals with a single reviewer. Matches squash-merge policies without friction.
- **Clustered MRs.** One MR per coherent subsystem cluster (e.g. backend refactor, FE changes, DB migration). Each targets the main branch independently; dependencies between clusters are managed by MR merge order, not by branch stacking. Fits squash-merge policies and gives subject-matter reviewers smaller, focused MRs. Recommended for goals that touch ≥3 subsystems.
- **Stacked MRs.** Each cluster gets its own branch; each branch targets its parent (not the main branch). The graph becomes a chain of MRs. Best for teams that don't squash-merge and have tooling like git-branchless, Graphite, or stack-pr. Incurs rebase pain when intermediate MRs merge under squash-merge policies; use only if the team has embraced stacked workflows.

Default to single for small goals, clustered for large ones. Only use stacked when the team explicitly runs a stacked workflow.

## Common failure modes to avoid

- **Planning the graph up front.** The graph is discovered through experiments. If you find yourself writing prerequisites you haven't actually observed as failures, stop and experiment.
- **Fixing during the naive attempt.** The urge is strong. Resist. Discard and record.
- **Leaves that are too big.** A leaf that crosses clear subsystem boundaries or introduces a new abstraction is an unexpanded node; re-experiment on it. Pure mechanical edits that happen to span many files are not necessarily too big; judge by cognitive load rather than LOC or file count.
- **Skipping reverts.** "I'll just keep this working code and clean up later" is how merges become disasters. The method's value comes from always returning to a known-good state between leaves.
- **Graph drift.** The graph and code must match. If the user adds a commit out-of-band, reconcile the graph before the next leaf.
- **Scope leaks between leaves.** A leaf that solves one prerequisite's symptom with a shim that touches an adjacent prerequisite's territory creates latent bugs (the FE shim displays legacy values correctly but the backend still sees them as unknown). If a leaf's implementation reaches into another prerequisite's scope, either fold the two leaves or stop and surface the leak as a sub-prerequisite.
- **Running broad tests per leaf.** Regression-tier suites burn minutes and compound across leaves. Stick to the fast tier from the goal's Testing plan by default; escalate to targeted only when the leaf changes logic or an affinity rule fires. Regression runs once, at Phase 5.
- **Separate codegen chore commits.** Regenerated OpenAPI specs, API clients, and lockfiles belong in the commit of the leaf that caused them to regenerate. Don't split them out.
