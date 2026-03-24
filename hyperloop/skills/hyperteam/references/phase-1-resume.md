# Phase 1 — Resume

This path is taken when `plans/<branch>-team-state.json` already exists.

______________________________________________________________________

## Step 1 — Read authoritative state

- Read `plans/<branch>-team-state.json` (authoritative source of truth).
- Read `plans/<branch>-progress.txt` if it exists (for context only).

______________________________________________________________________

## Step 2 — Reconcile state

- Tasks with `status: completed` or `status: validated` are done — leave them.
- Any task with `status: in_progress` was interrupted — reset it to `pending`.
- Clear `native_task_id` to `null` on **every** task (both pending and reset tasks) — the
  native task list will be re-seeded in Step 5.
- `team-state.json` wins over `progress.txt` if they disagree.

______________________________________________________________________

## Step 3 — Render resume summary

Prepare a summary listing:

- Tasks already done: IDs and titles of all `completed` / `validated` tasks.
- Tasks remaining: IDs and titles of all `pending` tasks (after the reset in Step 2).
- Current `gate_iterations` count.

______________________________________________________________________

## Step 4 — Confirm with user

Use `AskUserQuestion` to present the summary and ask:

> Found existing run for `<branch>`.
> Completed: N tasks — <list of completed task IDs and titles>
> Remaining: M tasks — <list of remaining task IDs and titles>
> Gate iterations so far: G
>
> Continue with remaining tasks?

______________________________________________________________________

## Step 5 — Proceed or stop

- **If user confirms:**
  1. Write the updated `team-state.json` (with `in_progress` tasks reset to `pending` and all
     `native_task_id` values cleared to `null`).
  2. Re-seed the native task list: for every task with `status: pending`, call `TaskCreate`
     with the task's YAML front-matter block and full story text as the `description`. The YAML
     front-matter format is:

     ```
     ---
     id: <task_id>
     type: <FEAT|DOC|GATE>
     role_hint: <role_hint>
     blocked_by:
       - <blocker_id_1>
     ---

     <full story text and acceptance criteria>
     ```

  3. Store each returned task UUID as `native_task_id` in the corresponding task object in
     `team-state.json`.
  4. Write the updated `team-state.json` to disk.
  5. Return to SKILL.md and proceed to Phase 2.

- **If user declines** — stop here. Leave state files unchanged.
