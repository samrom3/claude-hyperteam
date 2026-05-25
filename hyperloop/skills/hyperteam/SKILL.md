---
name: hyperteam
description: "Reads a session-spec, derives a task DAG, gets user approval, writes team-state.json, seeds the native task list, creates a specialist team, and monitors until the back-pressure gate passes."
user-invocable: true
disable-model-invocation: true
---

# Hyperteam

Converts session-spec into autonomous agent team. Executes full task DAG, tracks state in `plans/<branch>-team-state.json`, coordinates via native task list, offers PR when gate passes.

______________________________________________________________________

## Phase 0: Pre-Flight

> **Prerequisites:** Requires Agent Teams feature + `gh` CLI. Set `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`. Verify `gh` CLI installed and authenticated for PR creation.

Run checks in order. **Stop and surface each issue as encountered.**

### Step 1 — Scan `plans/` and select a session-spec

1. **Legacy check:** List files matching `*-prd.md` in `plans/`. Any found → stop:
   > Legacy `-prd.md` plans found: `<list>`. Rename to `-session-spec.md` to proceed, or create a new spec with `/session-spec`. Do not silently process old-format files.
2. List all files in `plans/` matching `*-session-spec.md`.
3. For each `plans/<name>-session-spec.md`, determine state:
   - No `plans/<name>-team-state.json` → **unstarted**.
   - `metadata.status = "running"` → **in-progress**.
   - `metadata.status = "complete"` → **complete**.
   - Any other value → **in-progress**.
4. Exclude **complete** specs from selection.
5. No incomplete specs → stop:
   > No incomplete session-specs found in `plans/`. Create a session-spec first with `/session-spec`.
6. Build ordered selection list: unstarted first, then in-progress. Within each group, sort by file modification time (most recent first). Format each entry: `<n>. plans/<name>-session-spec.md`. Append for in-progress entries: `⚠ This spec may have an in-flight hyperteam run. Ensure no other session is working on it before proceeding.`
7. **Single spec:** Use `AskUserQuestion`:
   > Only one incomplete session-spec found:
   >
   > `plans/<name>-session-spec.md` [warning if in-progress]
   >
   > Proceed with this spec?

   User confirms → select it. Otherwise stop.
8. **Multiple specs:** Use `AskUserQuestion`:
   > Multiple session-specs found. Choose one to run:
   >
   > <numbered selection list>

   Wait for user's choice.
9. Derive `<branch>` from selected filename: strip `plans/` prefix and `-session-spec.md` suffix.
10. Derive `<slug>` from `<branch>` by stripping leading `feat-` prefix if present. If `<branch>` does not start with `feat-`, use `<branch>` as `<slug>` unchanged.

### Step 2 — Checkout git branch

1. Run `git branch --show-current`.
2. Result matches `<branch>` → proceed to Step 3.
3. Mismatch:
   a. Run `git branch --list <branch>`.
   b. **Branch exists locally** → `git checkout <branch>`.
   c. **Branch absent** → `git fetch origin main && git checkout -b <branch> origin/main`.
   d. Verify `git branch --show-current` equals `<branch>`. Mismatch → `AskUserQuestion` and stop.

### Step 3 — Verify symlink

1. Run `test -L plans/<branch>`.
2. Absent or not symlink → create:
   ```
   mkdir -p plans && ln -sf ~/.claude/tasks/<branch> plans/<branch>
   ```
3. Verify: `readlink plans/<branch>` must return path ending in `.claude/tasks/<branch>`. Fails → `AskUserQuestion` and stop.

> **Note:** Task list scoping handled automatically by `TeamCreate` in Phase 2, Step 2. `TeamCreate` with `team_name: "<branch>"` creates task list at `~/.claude/tasks/<branch>/` and sets `CLAUDE_CODE_TEAM_NAME` on all teammates. No manual `export` of `CLAUDE_CODE_TASK_LIST_ID` needed.

### Step 4 — Detect fresh start vs. resume

Check `plans/<branch>-team-state.json`:
- **Absent** → Read `references/phase-1-fresh-start.md` and follow in full. Return here, proceed to Step 5.
- **Present** → Read `references/phase-1-resume.md` and follow in full. Return here and proceed to Step 5 (or stop if user declines).

### Step 5 — Gate discretion

After spec parsed and task DAG proposed, assess whether back-pressure gate task is needed.

**Gate recommended when any of:**

- Total step count ≥ 4
- Cross-step integration required (one step's output is another's input)
- Output not easily human-verifiable in-session (e.g., data pipelines, infra changes)

**Gate optional when ALL of:**

- Total step count ≤ 3
- Each step has independent verify criteria (no cross-step deps)
- Human already in verification loop (e.g., UI changes requiring manual review, or user will run `claude plugin validate .` themselves)
- Unit/integration tests alone sufficient to confirm correctness

Present recommendation via `AskUserQuestion`:

> Gate task **recommended / not recommended** because [reason].
>
> Proceed with gate? [Yes / No / Override]

User can override either way. If Yes (or override to Yes) → include GATE task in DAG and Phase 2 team creation. If No → omit GATE task.

Proceed to Phase 2.

______________________________________________________________________

## Phase 2: Team Creation and Coordination

### Step 1 — Count parallel-eligible tasks

1. Read `plans/<branch>-team-state.json`.
2. Count tasks where `status: pending` AND `blocked_by` is empty (or all listed blockers already terminal). Call this count `N`.
3. Clamp to `M`: `M = min(max(N, 1), 4)`.

### Step 2 — Create the team

Call `TeamCreate` with:
- Team name: `<branch>`
- Teammates: 1 `hyperteam-lead`, `M` `hyperteam-worker` instances, 1 `hyperteam-reviewer`
- Prompt includes: branch name, paths to `plans/<branch>-team-state.json`, `plans/<branch>-progress.txt`, `plans/<branch>-session-spec.md`

### Step 3 — Seed the native task list

For every task in `team-state.json` with `status: pending`:

1. Call `TaskCreate` with YAML front-matter block + full step text as `description`:
   ```
   ---
   id: <task_id>
   type: <FEAT|DOC|GATE>
   skills:
     - <skill_name>
   blocked_by:
     - <blocker_id_1>
     - <blocker_id_2>
   ---

   <full step text and acceptance criteria from team-state.json task description>
   ```

### Step 4 — Broadcast kickoff

Send broadcast `SendMessage` to team:

> Hyperteam `<branch>` is starting.
> State file: `plans/<branch>-team-state.json`
> Progress log: `plans/<branch>-progress.txt`
>
> All workers: claim tasks from native task list. Parse YAML front-matter in each task's description for `type` (FEAT or DOC) and `blocked_by`. Load `skills:` listed in task front-matter via `Skill` tool before beginning work. Resolve blockers via `team-state.json` (blocker terminal when status `validated` or `completed`).
>
> Reviewer: begin scanning `team-state.json` for completed FEAT tasks with `reviewed: false` immediately.

### Step 5 — Monitor

Main thread monitors run. Lead agent (dispatched in Step 2) handles coordination: review outcomes, failure resets, blocker broadcasts, GATE readiness detection.

React to events:
- **`SendMessage` from lead signalling GATE PASS** → proceed to Phase 4.
- **`SendMessage` from any teammate requiring main-thread intervention** → address and resume.

Main thread does **not** dispatch individual workers or validators. Teammates self-claim.

______________________________________________________________________

## Phase 3: Back-Pressure Gate

> Runs **inside** reviewer agent — not main thread. Reviewer claims GATE native task when lead broadcasts GATE OPEN. See `references/gate-task-template.md` for full gate agent instructions.

Lead notifies main thread only after GATE passes. Proceed to Phase 4.

______________________________________________________________________

## Phase 4: Completion and PR Offer

Read `references/phase-4-completion.md` and follow in full.

______________________________________________________________________

## Phase 5: Team Cleanup

After Phase 4 completes (summary written, PR offered/created/declined), call `TeamDelete` for team `<branch>`. Removes all shared team resources. Must be done after Phase 4 so all teammates fully idle before cleanup.
