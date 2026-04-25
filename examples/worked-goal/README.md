# worked-goal example

A sanitized example of what `.mikado/<slug>.md` looks like for a real, completed Mikado goal.

[`remove-redisson.md`](remove-redisson.md) is the artifact produced by running:

```
/mikado remove Redisson, replace with Spring Data Redis (Lettuce)
/loop /mikado-loop
/mikado-mr
```

against a hypothetical Spring Boot platform. It shows:

- **Frontmatter** — slug, ticket, base commit, commit strategy.
- **Goal statement** — concrete, measurable, with implementation direction.
- **Acceptance criteria** — what "done" looks like, in language reviewers can verify.
- **Status** — current state of the goal.
- **MR** — link back to the merge request once opened.
- **Mermaid graph** — the prerequisite DAG, with `classDef` styling that distinguishes observed (from the naive experiment) vs anticipated (from the plan, confirmed later).
- **Prerequisites** — checked-off list with file references, decisions made, and notes about what was folded together.
- **Notes and learnings** — non-obvious facts about the codebase, pre-existing flakes, open questions and how they were resolved.

## What to copy from this example

When you start your own goal, you don't need to write any of this from scratch. The `mikado` skill writes the initial scaffolding (frontmatter + goal statement + Mermaid stub + empty checklist) for you. Then the leaf loop fills in the rest as it discovers prerequisites and lands commits.

What you do want to copy is **the level of detail in the prerequisite entries and the Notes section**. Specifically:

- File paths next to each prereq, so reviewers can find the change.
- Decisions made during the leaf, so future-you knows why.
- Pre-existing flakes called out explicitly, so they don't get blamed on this work.
- Open questions tracked through resolution, so the MR description can summarize them honestly.

## What NOT to copy

- The exact prerequisite count. Real goals can have 3 or 30 leaves; the method scales either way.
- The exact graph shape. The DAG is whatever the codebase tells you it is, not what looks tidy.
- Style decisions like emoji checkmarks or `✓` in node labels. Use whatever your team finds readable.
