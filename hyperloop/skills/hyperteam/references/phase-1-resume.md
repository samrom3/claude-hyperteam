# Phase 1 — Resume

Taken when `plans/<branch>-team-state.json` already exists.

---

## Step 1 — Read authoritative state

- Read `plans/<branch>-team-state.json` (authoritative source of truth).
- Read `plans/<branch>-progress.txt` if exists (context only).

---

## Step 2 — Reconcile state

Before reconciling, revert uncommitted/unstaged changes:

```bash
git stash --include-untracked
# or: git checkout -- . && git clean -fd
```

Any discarded work was interrupted mid-task. Its JSON `status` will be `pending` (workers write `completed` only on successful commit) → re-enters queue naturally.

Reconcile:

- Tasks with `status: completed` or `status: validated` → done, leave them.
- All other tasks remain `pending` — no resets needed.
- `team-state.json` wins over `progress.txt` on disagreement.

---

## Step 3 — Render resume summary

Summary listing:
- Done: IDs and titles of all `completed` / `validated` tasks.
- Remaining: IDs and titles of all `pending` tasks (after reset in Step 2).
- Current `gate_iterations` count.

---

## Step 4 — Confirm with user

Use `AskUserQuestion`:

> Found existing run for `<branch>`.
> Completed: N tasks — <list of completed task IDs and titles>
> Remaining: M tasks — <list of remaining task IDs and titles>
> Gate iterations so far: G
>
> Continue with remaining tasks?

---

## Step 5 — Proceed or stop

- **User confirms:**
  1. Re-seed native task list: per `status: pending` task, call `TaskCreate` with YAML front-matter and full step text as `description`:

     ```
     ---
     id: <task_id>
     type: <FEAT|DOC|GATE>
     skills:
       - <skill_name>
     blocked_by:
       - <blocker_id_1>
     ---

     <full step text and acceptance criteria>
     ```

  2. Return to SKILL.md, proceed to Phase 2.

- **User declines** — stop. Leave state files unchanged.
