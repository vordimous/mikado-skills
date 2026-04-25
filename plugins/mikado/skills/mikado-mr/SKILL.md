---
name: mikado-mr
description: Prepare a merge/pull request from a completed Mikado goal. Reads .mikado/<slug>.md, extracts the graph, synthesizes a title and body, detects the forge from the git remote (GitHub via gh, GitLab via glab), then proposes (does not run) the appropriate create command. Falls back to manual instructions when the forge is unknown or no CLI is installed. Use when the user says "open the MR", "open the PR", "prep the MR", "/mikado-mr", or signals that a Mikado goal is done and ready to ship. Pairs with the mikado skill.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, AskUserQuestion
---

# Mikado MR

Turns a completed Mikado goal into a merge or pull request proposal. The synthesis (title, body, graph) is forge-agnostic; the command shape adapts to the detected forge. This skill never runs the create command itself; it assembles the title, body, and command for the user to execute.

## When to use

- The user finished a Mikado goal and wants to open the request
- The user invokes `/mikado-mr` with or without an argument
- The user says "open the MR", "open the PR", "prep the MR", "ship it", "create MR for the mikado goal"

If multiple `.mikado/*.md` files exist, the user can pass a slug: `/mikado-mr remove-redisson`. Otherwise, detect by most recently modified, or ask if ambiguous.

## Preflight

1. **Detect the forge** from `git remote get-url origin`:
   - Host is `github.com` → forge is GitHub; CLI is `gh`.
   - Host contains `gitlab` (e.g. `gitlab.com`, self-hosted `gitlab.example.com`) → forge is GitLab; CLI is `glab`.
   - Environment variable `MIKADO_FORGE` is set to `github` or `gitlab` → use that, regardless of host. This is the override for self-hosted GitLab on a vanity domain (e.g. `code.company.com`) where the host string doesn't contain `gitlab`.
   - Anything else → forge is unknown; fall through to manual mode (skip CLI auth check; render synthesis only). When falling through, log "forge unknown for host `<host>`; using manual mode. Set `MIKADO_FORGE=gitlab` (or `github`) to override." so the user knows why.
   If a CLI is selected, run `<cli> auth status`. If not authenticated, tell the user to run `<cli> auth login` and stop.
2. `git status --short`. Working tree must be clean. If dirty, ask the user to commit or stash first.
3. `git rev-parse --abbrev-ref HEAD`. Capture the source branch.
4. Confirm the branch is fully pushed:
   - `git rev-parse --abbrev-ref --symbolic-full-name @{u}`. Must resolve to an upstream (e.g. `origin/feat/…`). If not, propose `git push -u origin <branch>` and stop.
   - `git rev-list --count @{u}..HEAD`. Must be `0`. A nonzero count means the branch is ahead of its upstream and the local commits aren't on the remote yet. Propose `git push` and stop.
   - Both checks are required. The first only confirms a tracking ref exists; the second confirms HEAD matches.
5. Find the goal file at `.mikado/<slug>.md` (or as supplied). Must exist.

## Phase 1: Load and validate the goal

Read `.mikado/<slug>.md`. Extract:
- Goal statement (first heading)
- Ticket (front-matter, if present)
- Source plan path (front-matter, if present; goes into the MR body's Why section as "Plan: `<path>`" so reviewers can find the source doc)
- Base commit (front-matter)
- Acceptance section (bullets, if present)
- Mermaid graph
- Prerequisite checklist (with done/pending status)
- Notes and learnings

**Completeness check.** Walk the checklist:
- If any prereq is unchecked, list them and ask: proceed with an incomplete goal (draft request) or wait and finish first?
- If the goal itself is not marked Complete in the Status line, ask the same question.

Default recommendation: if any prereq is unchecked, create the request as a **draft** (the CLI's `--draft` flag, where supported). Note this in the body.

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

Filter out commits whose subject starts with `mikado:` (graph-management commits) from the user-visible "Commit sequence" list, but keep their count as a parenthetical: `(plus N graph-management commits)`.

## Phase 3: Determine target branch

Default to whatever branch the remote treats as default. Usually `main` or `master`, but `develop` is common in teams that use a separate integration branch. Detect with:

```bash
git remote show origin | sed -n '/HEAD branch/s/.*: //p'
```

Confirm with the user before composing if the detected default isn't obviously right. Ask with a single prompt showing the candidate.

## Phase 3.5: Pick the request shape

Three options; offer them explicitly when the goal is non-trivial (≥5 prereqs, or touches ≥3 subsystems by commit scope):

### Single request (default)

One MR/PR for the whole goal. Covers every leaf commit. Use for:
- Small-to-medium goals
- Single-subsystem changes
- Squash-merge teams with no stacked tooling

No extra branch surgery. Everything below this section applies unchanged.

### Clustered requests

Split the goal into N independent MRs/PRs, each targeting the main branch, clustered by commit scope or subsystem.

**When to offer this:** the goal touches clearly separate subsystems (e.g. `refactor(common)` + `feat(ui)` + `chore(migration)` commits). Reviewers for each subsystem can work in parallel.

**Cluster detection heuristic:** group commits by the scope in the Conventional Commits header. Example groupings:
- `common` / `integrations` commits → backend cluster
- `ui` commits → frontend cluster
- `migration` commits → migration cluster

Ask the user to confirm the clusters before proceeding. If they'd rather hand-pick which leaves go with which cluster, show a menu with a suggested grouping.

**Branch surgery per cluster:**
```bash
# For cluster X with source commits <sha1> <sha2> ...
git switch -c feat/<ticket>-<cluster-slug>  # new branch off the default branch
git cherry-pick <sha1> <sha2> ...
# user pushes, then opens the request for this cluster against the default branch
```

The original goal branch stays intact as a working-tree record; cluster branches derive from it via cherry-pick. Each cluster's request is independent. Merge order is flexible as long as logical dependencies are respected.

**Limitation:** cherry-picks can conflict if commits touch overlapping files. If a cluster needs commits from another cluster to apply cleanly, either merge the clusters or switch to stacked mode.

### Stacked requests

One request per cluster, each branch targeting its parent branch instead of the default branch. The graph becomes a chain of requests.

**When to offer this:** the team doesn't squash-merge, uses tooling like `git-branchless` / Graphite / `stack-pr`, and has the appetite for rebase-on-merge.

**Branch layout:**
```
feat/<ticket>-p1  → main
feat/<ticket>-p2  → feat/<ticket>-p1
feat/<ticket>-p3  → feat/<ticket>-p2
```

**Request creation:** GitHub uses `gh pr create --base <parent-branch>` directly; the chain is implicit in the base branch. GitLab uses `glab mr create --target-branch <parent-branch>` and an explicit dependency link via the GitLab API.

**Merge policy warning:** if the team squash-merges, stacked requests break. When the parent merges into the default branch, the child's base branch still points at the (now-orphan) parent branch. The fix is `git rebase --onto <default> <old-parent> feat/<ticket>-p2`, but rebase is denied by default. Tell the user this will require manual rebase between merges.

**Do NOT choose stacked mode on a squash-merge team unless the user explicitly opts in and accepts the manual rebase cost.**

### Asking the user

When the goal qualifies (≥5 prereqs OR ≥3 subsystems), show:

```
This goal has <N> leaves across <M> subsystems. How do you want to ship it?

  1. Single request (default): one MR/PR, all leaves included. Simplest review.
  2. Clustered requests: <K> independent requests by subsystem: <list>. Parallel review, works with squash-merge.
  3. Stacked requests: <K> dependent requests, each targeting its parent. Not recommended for squash-merge teams.
```

If the user picks 2 or 3, the subsequent phases produce multiple proposals instead of one.

## Phase 4: Synthesize the request

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

<one paragraph. Pull from the goal statement and Notes section. If a ticket is linked, cross-reference it. If a Source plan is recorded in the goal file's front-matter, append "Plan: `<path>`" on its own line so reviewers can read the original spec.>

## Acceptance

<copy the goal file's `## Acceptance` section verbatim. If the goal file has no Acceptance section, omit this heading entirely.>

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
- [ ] <affected module verification (pulled from cross-module warnings in the graph if present)>
- [ ] Manual smoke test on <relevant flow> if applicable

## Cross-module impact

<copy any ⚠️ warnings from the graph's prerequisites, or "None." if absent>

## Notes and learnings

<copy non-empty Notes section from the graph verbatim, so reviewers see what was discovered>

---

_Generated from `.mikado/<slug>.md`. Method: [Mikado](https://wellaged.dev/posts/mikado-method-ai-agents/)._
````

Keep it terse. Reviewers scan; long bodies get ignored.

## Phase 5: Propose the command(s)

The synthesis (title, body, target/source branches) is identical across forges; only the command shape changes. Render the command for the detected forge.

### GitHub (`gh`)

```
# Proposed PR

Title: <title>
Target: <target-branch>
Source: <source-branch>
Draft: <yes/no>

# Body:
<rendered body>

# Command to run:
gh pr create \
  --title "<title>" \
  --base <target> \
  --head <source> \
  [--draft] \
  --body "$(cat <<'EOF'
<body>
EOF
)"
```

### GitLab (`glab`)

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

### Unknown forge / no CLI installed

The skill still produces the synthesis. Show the title and body, give the user the manual steps:

```
# Proposed request (manual mode; no forge CLI detected)

Title: <title>
Target: <target-branch>
Source: <source-branch>
Draft: <yes/no>

# Body (paste into the forge UI):
<rendered body>

# Steps:
1. Confirm the branch is pushed: `git push -u origin <source>`
2. Open the new-request page on the forge:
   - GitHub:        https://<host>/<owner>/<repo>/compare/<target>...<source>
   - GitLab:        https://<host>/<owner>/<repo>/-/merge_requests/new?merge_request[source_branch]=<source>&merge_request[target_branch]=<target>
   - Other forges:  navigate to the project, open a new request from <source> → <target>
3. Paste the title and body above. Mark as draft if applicable.
```

### Clustered / stacked output

Show all N requests in a single block, ordered by dependency (parents first). Each entry uses the appropriate per-forge command from the sections above.

```
# Proposed requests (<mode>): <N> total

## Request 1/<N>: <cluster-title-1>
Source: <source-branch-1>
Target: <target-branch-1>
Commits: <sha1> <sha2> ...

Title: <title-1>
Body: <body-1>
Command: <forge-appropriate create command for request 1>

## Request 2/<N>: <cluster-title-2>
...

## Branch-creation commands (run before opening requests):
git switch -c <source-branch-1>
git cherry-pick <sha1> <sha2>  # clustered mode
  # or: git reset --hard <parent-sha>; git cherry-pick <sha3> <sha4>  # stacked mode setup
git switch -c <source-branch-2>
...

## Request-creation commands (run after pushing each branch):
<forge-appropriate create command for each request>
```

For **stacked GitLab requests**, follow each `glab mr create` with a dependency link so the review UI shows the chain. The exact endpoint depends on the GitLab version (`/blocks` on recent versions, `/dependencies` on older ones); check the [GitLab MR API docs](https://docs.gitlab.com/api/merge_requests/) for the syntax that matches your instance:
```bash
glab api projects/:id/merge_requests/<iid>/blocks --method POST --field blocking_merge_request_id=<parent-mr-id>
```
This API requires GitLab Premium or Ultimate. On GitLab Free or self-hosted CE, the call returns 403; in that case, fall back to a comment on each child MR linking to its parent ("Depends on !<parent-iid>") and skip the API call. GitHub stacked requests don't need a separate linking step. The chain is expressed by `--base <parent-branch>`.

Each body is the same template as the single-request case, but:
- `## Why` mentions which cluster this request covers and which sibling requests exist
- `## Commit sequence` lists only that cluster's commits
- `## Test plan` scopes to that cluster's subsystems
- A new `## Related requests` section links to the other clusters (use placeholder URLs; fill in after opening)

### Actions offered to the user

1. **Run the commands now.** Execute branch-creation, then the create command(s). Requires explicit confirmation; create is state-changing. For multi-request, confirm once for the whole batch.
2. **Copy for manual execution.** Default safer path when the user wants to inspect each command.
3. **Revise.** Change the cluster grouping, title, target, body, or collapse back to single-request mode.

If the user chooses (1), execute the commands sequentially, reporting each request URL as it opens. If a command fails mid-batch, stop and report which succeeded.

## Phase 6: Post-creation housekeeping (only if a request was created)

After a successful create:
- Append the request URL to `.mikado/<slug>.md` under a new `## MR` section (heading kept as `MR` for stable anchor; the URL itself disambiguates GitHub vs GitLab).
- Commit `mikado: record request link for '<slug>'` as a standalone commit. Don't amend the goal-complete commit; amending rewrites history and the user owns that path. A separate housekeeping commit is the safe default regardless of the goal's `folded`/`separate` setting.
- Remind the user the housekeeping commit isn't pushed yet, since the skill never pushes.

## Draft + review cycle

For goals that benefit from reviewer feedback mid-flow (most nontrivial Mikado goals do), encourage the two-phase pattern:

1. Open the request as a draft even if all prereqs are checked, when the goal is substantive enough to warrant review before merge prep.
2. Run a code review against the draft.
3. Land fixes as additional commits on the same branch.
4. Push, then re-run `/mikado-mr` which detects the existing request and offers to mark it ready rather than opening a new one. The "mark ready" command depends on the forge:
   - GitHub: `gh pr ready <number>`
   - GitLab: `glab mr update <iid> --ready`
   - Manual: open the request in the forge UI and click "Mark ready" (or equivalent).

This pattern keeps the review record transparent: the request's history shows "opened draft → reviewed → fixed → marked ready" instead of "opened ready → merged." Especially valuable for refactors that ran through the leaf loop, where each leaf was individually green but the gestalt of the change deserves a fresh look.

Skip the draft cycle only for purely mechanical goals (e.g. dependency renames) where review is mostly a formality.

## Operating rules

- **Never run the create command without explicit confirmation.** The default path is to propose the command, not execute it. Creating a request is state-changing and visible to the team.
- **Never push.** The user owns `git push`. If the branch is behind upstream, stop and ask.
- **Never force-push.** If the branch diverges from its upstream, surface the conflict and stop. Do not auto-rebase.
- **Draft over incomplete.** If any prereq is unchecked, default to draft. For complete goals, still prefer draft when the goal is substantive enough to merit review before merge.
- **Don't re-run preflight during a loop.** If `/mikado-loop` is orchestrating and calls this skill on every leaf completion, skip straight to an early exit when prereqs remain.
- **Never amend.** The post-creation housekeeping commit is standalone; do not amend the goal's final commit to add the request link.

## Related skills

- `mikado`: produces the goal file this skill consumes
- `mikado-loop`: drives the leaf loop that fills the goal file before this skill is invoked
