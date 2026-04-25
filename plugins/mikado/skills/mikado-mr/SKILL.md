---
name: mikado-mr
description: Prepare a GitLab merge request from a completed Mikado goal. Reads .mikado/<slug>.md, extracts the graph, synthesizes an MR title and body from the graph and the commits produced on the feature branch, then proposes (does not run) the glab mr create command. Use when the user says "open the MR", "prep the MR", "/mikado-mr", or signals that a Mikado goal is done and ready to ship. Pairs with the mikado skill. Requires glab to be installed and authenticated.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, AskUserQuestion
---

# Mikado MR

Turns a completed Mikado goal into a merge request proposal. This skill never runs `glab mr create` itself — it assembles the title, body, and command for the user to execute.

## When to use

- The user finished a Mikado goal and wants to open the MR
- The user invokes `/mikado-mr` with or without an argument
- The user says "open the MR", "prep the MR", "ship it", "create MR for the mikado goal"

If multiple `.mikado/*.md` files exist, the user can pass a slug: `/mikado-mr remove-redisson`. Otherwise, detect by most recently modified, or ask if ambiguous.

## Preflight

1. `glab auth status` — must be authenticated. If not, tell the user to run `glab auth login` and stop.
2. `git status --short` — working tree must be clean. If dirty, ask the user to commit or stash first.
3. `git rev-parse --abbrev-ref HEAD` — capture the source branch.
4. Confirm the branch is fully pushed:
   - `git rev-parse --abbrev-ref --symbolic-full-name @{u}` — must resolve to an upstream (e.g. `origin/feat/…`). If not, propose `git push -u origin <branch>` and stop.
   - `git rev-list --count @{u}..HEAD` — must be `0`. A nonzero count means the branch is ahead of its upstream and the local commits aren't on GitLab yet. Propose `git push` and stop.
   - Both checks are required. The first only confirms a tracking ref exists; the second confirms HEAD matches.
5. Find the goal file at `.mikado/<slug>.md` (or as supplied). Must exist.

## Phase 1: Load and validate the goal

Read `.mikado/<slug>.md`. Extract:
- Goal statement (first heading)
- Ticket (front-matter, if present)
- Base commit (front-matter)
- Mermaid graph
- Prerequisite checklist (with done/pending status)
- Notes and learnings

**Completeness check.** Walk the checklist:
- If any prereq is unchecked, list them and ask: proceed with an incomplete goal (draft MR) or wait and finish first?
- If the goal itself is not marked Complete in the Status line, ask the same question.

Default recommendation: if any prereq is unchecked, create the MR as **Draft** (`glab mr create --draft`). Note this in the MR body.

## Phase 2: Gather commit context

```bash
git log --oneline <base-commit>..HEAD
git log --pretty=format:'%H%n%s%n%b%n---' <base-commit>..HEAD
git diff --stat <base-commit>..HEAD
```

From these, produce:
- Ordered list of leaf commits (subject line + short SHA)
- List of touched paths grouped by top-level directory
- Total insertion/deletion stats

Filter out commits whose subject starts with `mikado:` (graph-management commits) from the user-visible "Commit sequence" list, but keep their count as a parenthetical — `(plus N graph-management commits)`.

## Phase 3: Determine target branch

Default to `develop` (x1 convention) when present on the remote. Otherwise `master` or `main`. Confirm with the user before composing if the default isn't obviously right — ask with a single prompt showing the detected default.

## Phase 3.5: Pick the MR shape

Three options; offer them explicitly when the goal is non-trivial (≥5 prereqs, or touches ≥3 subsystems by commit scope):

### Single MR (default)

One MR for the whole goal. Covers every leaf commit. Use for:
- Small-to-medium goals
- Single-subsystem changes
- Squash-merge teams with no stacked tooling

No extra branch surgery. This is what `/mikado-mr` has always produced; everything below this section applies unchanged.

### Clustered MRs

Split the goal into N independent MRs, each targeting the main branch, clustered by commit scope or subsystem.

**When to offer this:** the goal touches clearly separate subsystems (e.g. `refactor(common)` + `feat(ui)` + `chore(migration)` commits). Reviewers for each subsystem can work in parallel.

**Cluster detection heuristic:** group commits by the scope in the Conventional Commits header. Example groupings:
- `common` / `integrations` commits → backend cluster
- `ui` commits → frontend cluster
- `migration` commits → migration cluster

Ask the user to confirm the clusters before proceeding. If they'd rather hand-pick which leaves go with which cluster, show a menu with a suggested grouping.

**Branch surgery per cluster:**
```bash
# For cluster X with source commits <sha1> <sha2> ...
git switch -c feat/<ticket>-<cluster-slug>  # new branch off main/develop
git cherry-pick <sha1> <sha2> ...
# user pushes, then open MR for this cluster against main/develop
```

The original goal branch stays intact as a working-tree record; cluster branches derive from it via cherry-pick. Each cluster MR is independent — merge order is flexible as long as logical dependencies are respected.

**Limitation:** cherry-picks can conflict if commits touch overlapping files. If a cluster needs commits from another cluster to apply cleanly, either merge the clusters or switch to stacked mode.

### Stacked MRs

One MR per cluster, each branch targeting its parent branch instead of main. The graph becomes a chain of MRs.

**When to offer this:** the team doesn't squash-merge, uses tooling like `git-branchless` / Graphite / `stack-pr`, and has the appetite for rebase-on-merge.

**Branch layout:**
```
feat/<ticket>-p1  → main
feat/<ticket>-p2  → feat/<ticket>-p1
feat/<ticket>-p3  → feat/<ticket>-p2
```

**MR creation:**
```bash
glab mr create --source-branch feat/<ticket>-p1 --target-branch main
glab mr create --source-branch feat/<ticket>-p2 --target-branch feat/<ticket>-p1
glab mr create --source-branch feat/<ticket>-p3 --target-branch feat/<ticket>-p2
```

Link them explicitly so the review UI shows the chain:
```bash
glab api projects/:id/merge_requests/<iid>/dependencies --method POST --field depends_on=<parent-iid>
```

**Merge policy warning:** if the team squash-merges, stacked MRs break — when the parent MR squashes into main, the child MR's base branch still points at the (now-orphan) parent branch. The fix is `git rebase --onto main <old-parent> feat/<ticket>-p2`, but rebase is denied by default. Tell the user this will require manual rebase between merges.

**Do NOT choose stacked mode on a squash-merge team unless the user explicitly opts in and accepts the manual rebase cost.**

### Asking the user

When the goal qualifies (≥5 prereqs OR ≥3 subsystems), show:

```
This goal has <N> leaves across <M> subsystems. How do you want to ship it?

  1. Single MR (default) — one MR, all leaves included. Simplest review.
  2. Clustered MRs — <K> independent MRs by subsystem: <list>. Parallel review, works with squash-merge.
  3. Stacked MRs — <K> dependent MRs, each targeting its parent. Not recommended for squash-merge teams.
```

If the user picks 2 or 3, the subsequent phases produce multiple MR proposals instead of one.

## Phase 4: Synthesize the MR

### Title

If the goal has a ticket key (e.g. `PROJ-1234`), use:
```
PROJ-1234 // <type>: <goal statement in imperative, ≤60 chars>
```

If no ticket, use plain Conventional Commits:
```
<type>: <goal statement in imperative, ≤60 chars>
```

`<type>` is derived from the dominant commit type across the leaf commits (`fix`, `feat`, `chore`, `refactor`). If mixed, default to `refactor` for Mikado-style goals.

### Body template

````markdown
## Why

<one paragraph. Pull from the goal statement and Notes section. If a ticket is linked, cross-reference it.>

## Mikado Graph

<paste the final Mermaid block verbatim from .mikado/<slug>.md>

## Commit sequence

<numbered list of leaf commits in chronological order; exclude `mikado:` graph-management commits but note the count>

1. `<short-sha>` <subject>
2. `<short-sha>` <subject>
...

(plus N graph-management commits)

## Test plan

- [ ] <derived from notes + commit types; e.g. "Unit tests pass for api module">
- [ ] <affected module verification — pulled from cross-module warnings in the graph if present>
- [ ] Manual smoke test on <relevant flow> if applicable

## Cross-module impact

<copy any ⚠️ warnings from the graph's prerequisites, or "None." if absent>

## Notes and learnings

<copy non-empty Notes section from the graph verbatim, so reviewers see what was discovered>

---

_Generated from `.mikado/<slug>.md`. Method: [Mikado](https://wellaged.dev/posts/mikado-method-ai-agents/)._
````

Keep it terse. Reviewers scan MRs; long bodies get ignored.

## Phase 5: Propose the command(s)

### Single-MR output

Show the user:

```
# Proposed MR

Title: <title>
Target: <target-branch>
Source: <source-branch>
Draft: <yes/no>

# Body:
<rendered body>

# Command to run:
glab mr create \
  --title "<title>" \
  --target-branch <target> \
  --source-branch <source> \
  [--draft] \
  --description "$(cat <<'EOF'
<body>
EOF
)"
```

### Clustered / stacked output

Show all N MRs in a single block, ordered by dependency (parents first):

```
# Proposed MRs (<mode>): <N> total

## MR 1/<N>: <cluster-title-1>
Source: <source-branch-1>
Target: <target-branch-1>
Commits: <sha1> <sha2> ...

Title: <title-1>
Body: <body-1>
Command: <glab mr create ... for MR 1>

## MR 2/<N>: <cluster-title-2>
...

## Branch-creation commands (run before opening MRs):
git switch -c <source-branch-1>
git cherry-pick <sha1> <sha2>  # clustered mode
  # or: git reset --hard <parent-sha>; git cherry-pick <sha3> <sha4>  # stacked mode setup
git switch -c <source-branch-2>
...

## MR-creation commands (run after pushing each branch):
glab mr create ... # for each MR
```

Each body is the same template as single-MR, but:
- `## Why` mentions which cluster this MR covers and which sibling MRs exist
- `## Commit sequence` lists only that cluster's commits
- `## Test plan` scopes to that cluster's subsystems
- A new `## Related MRs` section links to the other clusters (use `<repo>!<iid>` placeholders; fill in after opening)

### Actions offered to the user

1. **Run the commands now** — execute branch-creation, then MR-creation. Requires explicit confirmation; `glab mr create` is state-changing. For multi-MR, confirm once for the whole batch.
2. **Copy for manual execution** — default safer path when the user wants to inspect each command.
3. **Revise** — change the cluster grouping, title, target, body, or collapse back to single-MR.

If the user chooses (1), execute the commands sequentially, reporting each MR URL as it opens. If a command fails mid-batch, stop and report which succeeded.

## Phase 6: Post-creation housekeeping (only if MR was created)

After a successful `glab mr create`:
- Append the MR URL to `.mikado/<slug>.md` under a new `## MR` section
- Commit `mikado: record MR link for '<slug>'` as a standalone commit. Don't amend the goal-complete commit — amending rewrites history and the user owns that path. A separate housekeeping commit is the safe default regardless of the goal's `folded`/`separate` setting.
- Remind the user the housekeeping commit isn't pushed yet, since the skill never pushes.

## Draft MR + review cycle

For goals that benefit from reviewer feedback mid-flow (most nontrivial Mikado goals do), encourage the two-phase pattern:

1. Open the MR as `--draft` even if all prereqs are checked, when the goal is substantive enough to warrant review before merge prep.
2. Run a code review (`/r-mr-review-glab` or similar) against the draft.
3. Land fixes as additional commits on the same branch.
4. Push, then re-run `/mikado-mr` which detects the existing MR and offers to mark it ready (`glab mr update <iid> --ready`) rather than opening a new one.

This pattern keeps the review record transparent: the MR's history shows "opened draft → reviewed → fixed → marked ready" instead of "opened ready → merged." Especially valuable for refactors that ran through the leaf loop, where each leaf was individually green but the gestalt of the change deserves a fresh look.

Skip the draft cycle only for purely mechanical goals (e.g. dependency renames) where review is mostly a formality.

## Operating rules

- **Never run `glab mr create` without explicit confirmation.** The default path is to propose the command, not execute it. Creating an MR is state-changing and visible to the team.
- **Never push.** The user owns `git push`. If the branch is behind upstream, stop and ask.
- **Never force-push.** If the branch diverges from its upstream, surface the conflict and stop. Do not auto-rebase.
- **Draft over incomplete.** If any prereq is unchecked, default to `--draft`. For complete goals, still prefer draft when the goal is substantive enough to merit review before merge.
- **Don't re-run the MR preflight during a loop.** If `/mikado-loop` is orchestrating and calls this skill on every leaf completion, skip straight to an early exit when prereqs remain.
- **Never amend.** The post-creation housekeeping commit is standalone; do not amend the goal's final commit to add the MR link.

## Related skills

- `mikado` — produces the goal file this skill consumes
- `r-mr-review-glab` — review the MR after it's open
- `r-mr-post-comments-glab` — post follow-up review threads
