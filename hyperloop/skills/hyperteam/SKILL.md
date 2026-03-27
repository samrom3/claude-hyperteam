---
name: hyperteam
description: "Reads a PRD, derives a task DAG, gets user approval, writes team-state.json, seeds the native task list, creates a specialist team, and monitors until the back-pressure gate passes. Replaces the /prd-tasks + /hyperworker two-step workflow."
user-invocable: true
disable-model-invocation: true
---

# Hyperteam

Converts a PRD into an autonomous agent team that executes the full task DAG, tracks state in
`plans/<branch>-team-state.json`, coordinates via the native task list, and offers PR creation
when all tasks pass the back-pressure gate.

______________________________________________________________________

## Phase 0: Pre-Flight

> **Prerequisites:** This skill requires the Agent Teams feature and `gh` CLI installed.
>
> 1. Ensure `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` is set in your environment.
> 2. Check that `gh` CLI is installed and authenticated for PR creation.

Run these checks in order.

> **IMPORTANT: Stop and surface each issue as you encounter it.**

### Step 1 — Scan `plans/` and select a PRD

1. List all files in `plans/` matching `*-prd.md`. These are the candidate PRDs.
2. For each candidate `plans/<name>-prd.md`, determine its state:
   - Check if `plans/<name>-team-state.json` exists.
   - If absent → **unstarted**.
   - If present, read `metadata.status`:
     - `"running"` → **in-progress**
     - `"complete"` → **complete**
     - Any other value → treat as **in-progress**.
3. Exclude **complete** PRDs from the selection list.
4. If no incomplete PRDs remain:
   > No incomplete PRDs found in `plans/`. Create a PRD first with `/prd`.

   Stop here.
5. Build the ordered selection list:
   - Unstarted PRDs first, then in-progress PRDs.
   - Within each group, sort by file modification time (most recent first).
   - Format each entry as: `<n>. plans/<name>-prd.md`
   - Append the following warning on the same line for every in-progress entry:
     `⚠ This PRD may have an in-flight hyperteam run. Ensure no other session is working on it before proceeding.`
6. **Single PRD:** If exactly one incomplete PRD exists, use `AskUserQuestion` to confirm:
   > Only one incomplete PRD found:
   >
   > `plans/<name>-prd.md` [warning if in-progress]
   >
   > Proceed with this PRD?

   If the user confirms, select it. Otherwise, stop.
7. **Multiple PRDs:** If more than one incomplete PRD exists, use `AskUserQuestion`:
   > Multiple PRDs found. Choose one to run:
   >
   > <numbered selection list from step 5>

   Wait for the user's choice.
8. Derive `<branch>` from the selected filename: strip the `plans/` prefix and the `-prd.md` suffix.
   - Example: `plans/feat-auth-flow-prd.md` → `feat-auth-flow`
9. Derive `<slug>` from `<branch>` by stripping the leading `feat-` prefix if present.
   - Example: `feat-auth-flow` → `auth-flow`
   - If `<branch>` does not start with `feat-`, use `<branch>` as `<slug>` unchanged.

### Step 2 — Checkout git branch

1. Run `git branch --show-current`.
2. If the result matches `<branch>` — proceed to Step 3.
3. If not:
   a. Run `git branch --list <branch>` to check whether the branch exists locally.
   b. **Branch exists locally** → run `git checkout <branch>`.
   c. **Branch does not exist locally** → run `git fetch origin main && git checkout -b <branch> origin/main`.
   d. Verify with `git branch --show-current` — the output must equal `<branch>`. If it doesn't,
      use `AskUserQuestion` to surface the error and stop.

### Step 3 — Verify symlink

1. Verify the symlink `plans/<branch>` → `~/.claude/tasks/<branch>` exists:
   - Run `test -L plans/<branch>`.
   - If absent or not a symlink, create it:
     ```
     mkdir -p plans && ln -sf ~/.claude/tasks/<branch> plans/<branch>
     ```
   - Verify: `readlink plans/<branch>` must return a path ending in `.claude/tasks/<branch>`.
     If it doesn't, use `AskUserQuestion` to surface the error and stop.

> **Note:** Task list scoping is handled automatically by `TeamCreate` in Phase 2, Step 2.
> When `TeamCreate` is called with `team_name: "<branch>"`, it creates the task list at
> `~/.claude/tasks/<branch>/` and sets `CLAUDE_CODE_TEAM_NAME` on all teammates. No manual
> `export` of `CLAUDE_CODE_TASK_LIST_ID` is needed.

### Step 4 — Detect fresh start vs. resume

Check whether `plans/<branch>-team-state.json` exists.

- **Absent** → Read `references/phase-1-fresh-start.md` and follow it in full, then return here
  and proceed to Phase 2.
- **Present** → Read `references/phase-1-resume.md` and follow it in full, then return here and
  proceed to Phase 2 (or stop if the user declines).

______________________________________________________________________

## Phase 2: Team Creation and Coordination

### Step 1 — Role analysis

1. Read `plans/<branch>-team-state.json`.
2. Collect the distinct set of `role_hint` values across all tasks with `status: pending`.
   Call this `roles_needed`.
3. Always add `hyperteam-reviewer` and `hyperteam-worker` to `roles_needed` regardless of task
   hints (reviewer is always needed; worker is the fallback for unmatched hints).

### Step 2 — Create the team

Call `TeamCreate` with:
- Team name: `<branch>`
- One teammate per role in `roles_needed`
- The prompt should include the branch name and the paths to:
  - `plans/<branch>-team-state.json`
  - `plans/<branch>-progress.txt`
  - `plans/<branch>-prd.md`

### Step 3 — Seed the native task list

For every task in `team-state.json` with `status: pending`:

1. Call `TaskCreate` with the task's YAML front-matter block and full story text as the
   `description`. The YAML front-matter format is:

   ```
   ---
   id: <task_id>
   type: <FEAT|DOC|GATE>
   role_hint: <role_hint>
   blocked_by:
     - <blocker_id_1>
     - <blocker_id_2>
   ---

   <full story text and acceptance criteria from team-state.json task description>
   ```

2. Store the returned task UUID as `native_task_id` in the corresponding task object in
   `team-state.json`.

3. After processing all pending tasks, write the updated `team-state.json` to disk.

### Step 4 — Broadcast kickoff

Send a broadcast `SendMessage` to the team:

> Hyperteam `<branch>` is starting.
> State file: `plans/<branch>-team-state.json`
> Progress log: `plans/<branch>-progress.txt`
>
> All specialists: claim your tasks from the native task list. Parse the YAML front-matter in
> each task's description to find your `role_hint` and `blocked_by` fields. Resolve blockers via
> `team-state.json` (a blocker is terminal when its status is `validated` or `completed`).
>
> Reviewer: begin scanning `team-state.json` for completed FEAT tasks with `reviewed: false`
> immediately.

### Step 5 — Monitor

The main thread now monitors the run. The lead agent (dispatched as a teammate in Step 2)
handles all coordination: review outcomes, failure resets, blocker broadcasts, and GATE
readiness detection.

React to events:
- **`SendMessage` from the lead signalling GATE PASS** → proceed to Phase 4.
- **`SendMessage` from any teammate requiring main-thread intervention** → address and resume.

The main thread does **not** dispatch individual workers or validators. Teammates self-claim.

______________________________________________________________________

## Phase 3: Back-Pressure Gate

> This phase runs **inside** the reviewer agent — not on the main thread.
> The reviewer claims the GATE native task when the lead broadcasts GATE OPEN.
> See `references/gate-task-template.md` for the full gate agent instructions.

The lead notifies the main thread only after the GATE passes. Proceed to Phase 4.

______________________________________________________________________

## Phase 4: Completion and PR Offer

Read `references/phase-4-completion.md` and follow it in full.

______________________________________________________________________

## Phase 5: Team Cleanup

After Phase 4 completes (summary written and PR offered/created/declined), clean up the team:

1. Call `TeamDelete` for team `<branch>`.

This removes all shared team resources. Must be done after Phase 4, not before, so all
teammates are fully idle before cleanup is attempted.
