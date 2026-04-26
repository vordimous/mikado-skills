# The rhythm of a Mikado session

The skill is interactive by design. It runs autonomously between checkpoints, then pauses at each phase boundary to surface a small decision. Once you learn the rhythm, you can predict where the pauses are and whether you need to be at the keyboard for them.

## Before the experiment (Phases 0 to 2)

After mechanical preflight (clean tree, valid branch), Claude prints the **Mikado ethos preamble** (3 sentences on the discipline: graph leads code, revert is free, one leaf per commit) and asks for three configuration choices: cadence (where the leaf loop pauses), MR strategy (when MRs open), and implementation (who writes the code: AI or you). The choices are persisted to the goal file's front-matter and apply for the rest of the goal. Phase 0.5 suppresses the preamble and the prompts on resume; the values are read from the existing front-matter. Edit the front-matter directly if you want to change a value mid-goal.

Then Claude pauses to derive and confirm the **testing plan** for this goal. The plan has three tiers (fast, targeted, regression), an optional affinity map, and a skip list. It is derived from existing repo context (`CLAUDE.md`, README, `package.json` scripts, `build.gradle` task names, etc.) so you do not need a permanent testing-strategy artifact in the repo. You sign off, amend, or add instructions before the naive experiment runs. Whatever you confirm is written into the goal file as a durable `## Testing plan` section that the leaf loop consults afterward.

Confidence labels (high, medium, low) are the skill's honest read of how well the derived plan matches reality. Low confidence means the plan was inferred from ecosystem conventions with no project-specific signal. That is the moment to add testing instructions if you have any.

If your goal file had an `Acceptance` or `Tasks` section, you will also see those echoed back so you can confirm anticipated prerequisites before they show up in the graph.

The naive worktree experiment runs without further input.

## At the leaf-loop handoff (Phase 3 to Phase 4)

The graph is built and the experiment is discarded. Cadence and MR strategy were already chosen in Phase 0.3, so the only remaining question at this handoff is:

- **Commit strategy.** When a leaf is checked off, do you want the graph update in a *separate* follow-up commit (cleaner history, more commits), or *folded* into the leaf's commit (fewer commits, slightly noisier diffs)? The choice is written to the goal file's front-matter and reused for every remaining leaf without re-asking.

Claude will also propose a **suggested first leaf** with the reasoning (true leaf, unblocks the most parents, lowest risk). Confirm or redirect. Nothing is written until you say go.

## Within the leaf loop (Phase 4)

Most leaves run quietly: pick, implement, verify, commit, mark done. The pauses you will see are the ones that matter:

- **Sub-prerequisite discovery.** If a leaf turns out to have its own prerequisites (compile errors that are not in scope, a missing migration), Claude records them as deeper graph nodes and stops to ask which to take next rather than guessing.
- **Subagent delegation.** For larger or cross-boundary leaves, Claude may propose delegating to a subagent. You will see the rationale (file count, subsystem boundary, abstraction risk) before the delegation runs.
- **Flaky tests.** A test that fails once then passes gets appended to the goal file's `## Testing plan` → `### Flaky tests` list and the leaf continues. A test that fails twice is treated as a real failure and triggers a sub-prerequisite or a fix-in-leaf decision.
- **Tier escalation.** If a leaf changes logic in a path that has no affinity rule, Claude proposes adding one (path → tier with filter) and confirms before extending the testing plan. The plan grows with the goal rather than being predicted up front.

Between these prompts, expect the same rhythm: short status updates, focused commits, no surprises.

## At the finish (Phase 5)

When every prerequisite is checked, the goal itself becomes the final leaf. Claude implements it, runs the regression tier from the testing plan (the only point in the session that tier runs), proposes the final commit message, and offers a short retro. The retro is yours to keep, edit, or discard. It goes in the goal file alongside the graph as the durable record.

## Why the pauses are there

The Mikado discipline depends on small, reversible decisions made at the right time. Hiding the commit strategy or the cadence behind defaults would save typing and lose the option to course-correct. The pauses are the feature.
