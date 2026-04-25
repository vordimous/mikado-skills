---
name: mikado-myproject
description: <PROJECT_NAME>-specific wrapper around the Mikado Method skill. Use when the user invokes /mikado-myproject in this repo, or when they say "use mikado" / "apply mikado method" while working here. Layers <PROJECT_NAME> build/test/commit conventions on top of the global mikado method. Prefer this over /mikado whenever the user is working in the <PROJECT_NAME> repo.
allowed-tools: Bash, Read, Write, Edit, Glob, Grep, Agent, EnterWorktree, ExitWorktree, TodoWrite, AskUserQuestion
---

# Mikado Method — <PROJECT_NAME> Wrapper

This skill is the <PROJECT_NAME>-aware adapter for the Mikado Method.

## How this skill runs

1. **Read the global method first:** open the `mikado` skill (`SKILL.md`) and follow its phases (Preflight → Record Goal → Naive Experiment → Analyze → Leaf Loop → Finish).
2. **Layer the <PROJECT_NAME> adaptations below** on top of those phases.

## <PROJECT_NAME> adaptations

### Build and test commands (Phase 0 detection)

Default to these instead of auto-detecting. **Always lead with the narrowest target** (compile-only for pure refactors, single-class test filters for logic changes). Module-level invocations are a yellow flag and require explicit justification.

**Narrowest first:**
- `<COMMAND_TO_COMPILE_AFFECTED_MODULE>` — compile-only check
- `<COMMAND_TO_RUN_SINGLE_TEST_CLASS>` — single-class verification

**Module-level (justification required, not per-leaf):**
- `<COMMAND_TO_RUN_AFFECTED_MODULE_SUITE>`

**Commit-boundary only (before opening the MR, never per-leaf):**
- `<COMMAND_TO_RUN_FULL_BUILD>`
- `<COMMAND_TO_RUN_FULL_TEST_SUITE>`

### Known flakes and environment quirks

Subagents rediscover these every session. Codify them here:

- **<FLAKY_TEST_OR_QUIRK_NAME>**: <what fails, when, and how to recognize that this is the known issue rather than a real regression>.
- **<ENV_VAR_REQUIREMENT>**: <which command needs which env var, and an acceptable placeholder value if applicable>.

### Codegen byproducts

When a leaf causes <CODEGEN_FILE_OR_PATH> to regenerate, stage and include those files in the leaf's commit. **Do not create a separate `chore: regenerate <X>` commit** — that pollutes the history with noise that belongs with the cause.

The exception is genuinely non-semantic drift (e.g. tool output reordering with no behavior change) during a build unrelated to any leaf. Run `git restore` to drop that drift instead of committing it.

### Ticket and branch

Parse the current branch with `git rev-parse --abbrev-ref HEAD` and extract a ticket key matching `<TICKET_KEY_PATTERN>` (e.g. `PROJ-1234`).

Use the key in:
- The `Ticket:` header in `.mikado/<slug>.md`
- Every commit message proposal: `<TICKET_KEY> // <type>: <description>`
- The final MR title proposal: `<TICKET_KEY> // <type>: <goal>`

If no key is present in the branch name, ask the user once whether to record a ticket or proceed without one.

### Cross-module impact warnings (Phase 3)

When a prerequisite touches any of the following paths, add a bold warning to its graph node or checklist entry:

- **`<SHARED_MODULE_PATH>`** — `⚠️ Affects all dependent modules. Tests must pass in every affected module before commit.`
- **`<AUTO_GENERATED_PATH>`** — `⚠️ Auto-generated. Do not edit; regenerate from the source.`

### Commit message conventions

Follow Conventional Commits with optional <PROJECT_NAME> scopes:
- Scopes: `<scope1>`, `<scope2>`, `<scope3>` (typically map to top-level module or subsystem names).
- Examples:
  - `<TICKET_KEY> // fix(<scope>): <imperative subject>`
  - `<TICKET_KEY> // chore(<scope>): <imperative subject>`
  - `mikado: record prerequisites from naive attempt` (graph-only commits don't need a ticket scope)

### MR shape recommendation for <PROJECT_NAME>

<PROJECT_NAME> uses <SQUASH | MERGE | REBASE>-merge onto shared branches.

- **Default: single MR** for small-to-medium goals.
- **Preferred for goals spanning ≥3 subsystems: clustered MRs** — each subsystem gets its own MR targeting the main branch, opened in parallel.
- **Avoid stacked MRs** unless the team explicitly runs a stacked workflow with tooling like git-branchless or Graphite.

### Subagent delegation hints (Phase 4b)

When delegating a leaf to a subagent, the prompt MUST include:
- Which <PROJECT_NAME> module(s) the leaf touches
- The specific test command(s) to run for that module, leading with the narrowest target
- The ticket key
- The conventional commit format with scope
- The cross-module impact warning if applicable
- The Known Flakes list above, so the subagent doesn't chase them
- The codegen byproduct rule, so they're folded into the leaf commit

### Reference material

<Optional: link to a project handbook, architecture doc, or runbook the agent can consult.>
