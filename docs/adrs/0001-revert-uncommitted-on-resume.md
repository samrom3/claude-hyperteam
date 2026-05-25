# ADR-0001: Revert Uncommitted Work on Hyperteam Resume

**Status:** Accepted

**Date:** 2026-05-25

## Context

When a hyperteam session is interrupted mid-task (crash, timeout, user stop), the worker may leave
uncommitted/unstaged files on disk. The previous resume procedure tried to detect this by checking
whether any task had `status: in_progress` in `team-state.json`, then resetting it to `pending`.

This approach had two problems:

1. The JSON `in_progress` write was a separate operation from the native `TaskUpdate` call, so the
   two could fall out of sync. A crash between them left the state ambiguous.
1. Even when detected correctly, leaving partial implementation files on disk gave the next agent
   session unreliable context — a half-written file is worse than a blank one.

Git commit history is the only authoritative record of completed work. A task that was not
committed was not done.

## Decision

On resume, discard all uncommitted/unstaged changes before reconciling task state:

```bash
git stash --include-untracked
# or, if a clean wipe is preferred:
# git checkout -- . && git clean -fd
```

Any task whose work was discarded will have `status: pending` in `team-state.json` (workers write
`completed` only on successful commit) and will re-enter the queue naturally. Workers claim it
again and start from scratch.

## Consequences

- **Positive:** Resume logic is simple — no `in_progress` tracking, no status resets, no
  `native_task_id` lookups. The queue state is always consistent with the commit graph.
- **Positive:** Eliminates the JSON/native-task sync hazard entirely for in-flight tasks.
- **Negative:** A worker that completed most of a task but had not yet committed loses that work
  and must redo it. This is an acceptable trade-off; partial implementations are small in scope.
- **Neutral:** `status: in_progress` is removed from the JSON state machine. Live execution status
  lives only in the native task list (via `TaskUpdate`); JSON holds terminal states only.

## Alternatives Considered

**Track in-flight tasks in JSON (`status: in_progress`) and resume mid-task.** Rejected. A new
agent session has no reliable way to continue a partial implementation left by a prior session.
The prior session's reasoning and intermediate steps are gone. Starting fresh from `pending` is
safer and produces higher-quality output than trying to patch an unknown partial state.
