# ADR-001: Runtime Task List Scoping via `export`

**Status:** Accepted

**Date:** 2026-03-26

______________________________________________________________________

## Context

Claude Code's native task list is scoped by the `CLAUDE_CODE_TASK_LIST_ID` environment variable.
When this variable is set, all `TaskCreate`, `TaskList`, and `TaskUpdate` calls operate on the
named task list rather than the default one. This allows multiple independent task lists to
coexist in a single installation.

Hyperteam previously relied on the `/prd` skill to write `CLAUDE_CODE_TASK_LIST_ID` into
`.claude/settings.local.json` under `env`. This approach has two significant problems:

1. **Concurrent runs clobber each other.** `.claude/settings.local.json` is a shared file on
   disk. If a user runs `/prd` to create a second PRD while a hyperteam session is already
   executing, the file is overwritten with the new branch name. The in-flight session then reads
   the wrong task list ID on its next task operation.

1. **Agent Teams do not get isolated environments.** Teammates spawned via `TeamCreate` inherit
   the environment of the parent session at team creation time — they do not re-read
   `settings.local.json` on each tool call. If the shared file is mutated after team creation,
   teammate task operations silently target the wrong list.

The root cause is that `.claude/settings.local.json` is a single file shared across all sessions
and all time. It is not a suitable store for per-session, per-run configuration.

______________________________________________________________________

## Decision

Hyperteam sets `CLAUDE_CODE_TASK_LIST_ID` via a runtime `export` statement immediately after
PRD selection, instead of reading it from `.claude/settings.local.json`.

The export happens in Phase 0 of the `/hyperteam` skill, after the user selects (or confirms)
the PRD to execute and before any `TaskCreate` calls. Because the export is a shell environment
mutation scoped to the current process, it affects only the running session and its spawned
teammates — not any other concurrent session.

The `/prd` skill's responsibility is reduced to creating the branch, the `plans/` directory, and
the `plans/<branch>` → `~/.claude/tasks/<branch>` symlink. It no longer writes
`CLAUDE_CODE_TASK_LIST_ID` to `.claude/settings.local.json`.

______________________________________________________________________

## Consequences

**Positive:**

- **Concurrent runs work correctly.** Each `/hyperteam` invocation in a separate Claude Code
  session exports its own `CLAUDE_CODE_TASK_LIST_ID`. The sessions are fully isolated from each
  other at the process level.
- **No manual configuration required.** Users no longer need to ensure `settings.local.json` is
  correct before running `/hyperteam`. The skill derives the correct value from the selected PRD
  filename and sets it automatically.
- **`/prd` is decoupled from `/hyperteam`.** Users can create several PRDs upfront and run
  `/hyperteam` against any of them in any order, in any session.

**Neutral:**

- The `plans/<branch>` symlink (created by `/prd`) is still required. Hyperteam verifies the
  symlink exists after PRD selection and creates it if missing.

**Negative / Trade-offs:**

- The env var is no longer persisted between sessions. If a user resumes a hyperteam run in a
  new terminal, the export must happen again — which it does, automatically, because Phase 0 runs
  at the start of every `/hyperteam` invocation.
