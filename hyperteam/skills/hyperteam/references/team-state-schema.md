# team-state.json Schema

`plans/<branch>-team-state.json` is the authoritative task state registry for a hyperteam run. It
is written at the start of Phase 2 and mutated by the lead agent, reviewer, and teammates
throughout the run. The native task list (managed via `TaskCreate`/`TaskUpdate`/`TaskList`) is the
live coordination bus; `team-state.json` is the durable mirror used for re-entrancy and as the
single source of truth for blocker resolution.

______________________________________________________________________

## Top-level structure

```json
{
  "metadata": { ... },
  "tasks": [ ... ],
  "gate_iterations": 0
}
```

______________________________________________________________________

## `metadata`

| Field           | Type   | Description                                                      |
|-----------------|--------|------------------------------------------------------------------|
| `branch`        | string | Git branch name (e.g. `"feat-user-auth"`).                      |
| `slug`          | string | Short identifier derived from branch (e.g. `"user-auth"`).      |
| `prd_path`      | string | Relative path to the PRD file (e.g. `"plans/feat-user-auth-prd.md"`). |
| `status`        | string | Overall run status. One of: `"running"`, `"complete"`.          |
| `created_at`    | string | ISO 8601 timestamp when the file was first written.             |

### Example

```json
"metadata": {
  "branch": "feat-user-auth",
  "slug": "user-auth",
  "prd_path": "plans/feat-user-auth-prd.md",
  "status": "running",
  "created_at": "2026-03-14T10:00:00Z"
}
```

______________________________________________________________________

## `tasks`

Array of task objects. Each object represents one unit of work derived from the PRD DAG or created
as a gate remediation entry.

| Field              | Type                   | Default | Description                                                        |
|--------------------|------------------------|---------|---------------------------------------------------------------------|
| `id`               | string                 |         | Unique task identifier (e.g. `"FEAT-user-auth-01"`).               |
| `title`            | string                 |         | Short human-readable title.                                         |
| `description`      | string                 |         | Full task description including acceptance criteria.                |
| `type`             | string                 |         | One of: `"FEAT"`, `"DOC"`, `"GATE"`.                              |
| `role_hint`        | string                 |         | Which specialist should claim this task. Matches a registered agent name. Core: `"hyperteam-reviewer"`, `"hyperteam-techwriter"`, `"hyperteam-worker"`. Language packs add more (e.g. `"hyperteam-py-api-scaffolder"`, `"hyperteam-py-builder"`). |
| `status`           | string                 |         | One of: `"pending"`, `"in_progress"`, `"completed"`, `"validated"`, `"failed"`, `"blocked"`. |
| `blocked_by`       | array of string        |         | IDs of tasks that must reach `"validated"` (FEAT) or `"completed"` (DOC) before this task unblocks. |
| `native_task_id`   | string \| null         | `null`  | UUID returned by `TaskCreate` for the corresponding native task entry. Cleared to `null` on resume (re-seeded on start). |
| `started_at`       | string \| null         | `null`  | ISO 8601 timestamp when an agent picked up this task, or `null`.   |
| `completed_at`     | string \| null         | `null`  | ISO 8601 timestamp when the task reached `"completed"`, or `null`. |
| `reviewed`         | boolean                | `false` | Whether the reviewer has claimed this task for review. Set to `true` as a mutex before review begins. FEAT tasks only. |
| `review_result`    | string \| null         | `null`  | Reviewer verdict: `"PASS"`, `"FAIL"`, `"in_progress"`, or `null` if not yet reviewed. |
| `review_notes`     | array of string \| null| `null`  | Array of specific findings from reviewer. `null` until review runs; `[]` for clean PASS; list of strings for FAIL findings. |
| `reviewed_at`      | string \| null         | `null`  | Timestamp when reviewer completed; `null` until reviewer runs.   |

### Status transitions

```
pending → in_progress → completed → validated
                      ↘ failed
                                  ↘ blocked  (after third reviewer FAIL)
```

- `pending`: not yet started; all blockers may or may not be complete.
- `in_progress`: an agent has claimed the task.
- `completed`: the agent finished and committed its work.
- `validated`: the reviewer confirmed the work meets acceptance criteria (FEAT tasks only).
- `failed`: set by a worker that cannot complete its task after retries. The lead attempts one re-dispatch before escalating to `blocked`.
- `blocked`: task failed review three times and cannot proceed without manual intervention.

### Example task objects

Pending task (not yet started):

```json
{
  "id": "FEAT-user-auth-02",
  "title": "Implement AuthService business logic",
  "description": "As a developer, I want ...\n\nAcceptance Criteria:\n- [ ] ...",
  "type": "FEAT",
  "role_hint": "hyperteam-py-builder",
  "status": "pending",
  "blocked_by": ["FEAT-user-auth-01"],
  "native_task_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "started_at": null,
  "completed_at": null,
  "reviewed": false,
  "review_result": null,
  "review_notes": null,
  "reviewed_at": null
}
```

Completed task (reviewer PASS):

```json
{
  "id": "FEAT-user-auth-01",
  "title": "Scaffold UserAuth model and AuthService API stubs",
  "description": "As a developer, I want ...\n\nAcceptance Criteria:\n- [ ] ...",
  "type": "FEAT",
  "role_hint": "hyperteam-py-api-scaffolder",
  "status": "validated",
  "blocked_by": [],
  "native_task_id": null,
  "started_at": "2026-03-14T10:05:00Z",
  "completed_at": "2026-03-14T10:45:00Z",
  "reviewed": true,
  "review_result": "PASS",
  "review_notes": [],
  "reviewed_at": "2026-03-14T11:00:00Z"
}
```

Failed task (reviewer FAIL):

```json
{
  "id": "FEAT-user-auth-03",
  "title": "Add authentication end-to-end tests",
  "description": "As a developer, I want ...\n\nAcceptance Criteria:\n- [ ] ...",
  "type": "FEAT",
  "role_hint": "hyperteam-py-builder",
  "status": "completed",
  "blocked_by": ["FEAT-user-auth-02"],
  "native_task_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
  "started_at": "2026-03-14T11:10:00Z",
  "completed_at": "2026-03-14T11:50:00Z",
  "reviewed": true,
  "review_result": "FAIL",
  "review_notes": ["Missing test for edge case: expired token refresh", "pre-commit failed: lint error in tests/test_auth.py"],
  "reviewed_at": "2026-03-14T12:05:00Z"
}
```

DOC task (no review step):

```json
{
  "id": "DOC-user-auth-01",
  "title": "Document AuthService public API",
  "description": "As a developer, I want ...",
  "type": "DOC",
  "role_hint": "hyperteam-techwriter",
  "status": "completed",
  "blocked_by": ["FEAT-user-auth-01"],
  "native_task_id": null,
  "started_at": "2026-03-14T10:50:00Z",
  "completed_at": "2026-03-14T11:30:00Z",
  "reviewed": false,
  "review_result": null,
  "review_notes": null,
  "reviewed_at": null
}
```

______________________________________________________________________

## `gate_iterations`

Integer. Starts at `0` when the file is first written. Incremented by 1 each time a gate iteration
fails (checks 3–5). Used by the iteration guard: if `gate_iterations` is **4 or higher**, the
reviewer uses `AskUserQuestion` to ask the user before creating more remediation work.

______________________________________________________________________

## Full example

```json
{
  "metadata": {
    "branch": "feat-user-auth",
    "slug": "user-auth",
    "prd_path": "plans/feat-user-auth-prd.md",
    "status": "running",
    "created_at": "2026-03-14T10:00:00Z"
  },
  "tasks": [
    {
      "id": "FEAT-user-auth-01",
      "title": "Scaffold UserAuth model and AuthService API stubs",
      "description": "As a developer, I want ...",
      "type": "FEAT",
      "role_hint": "hyperteam-py-api-scaffolder",
      "status": "validated",
      "blocked_by": [],
      "native_task_id": null,
      "started_at": "2026-03-14T10:05:00Z",
      "completed_at": "2026-03-14T10:45:00Z",
      "reviewed": true,
      "review_result": "PASS",
      "review_notes": [],
      "reviewed_at": "2026-03-14T11:00:00Z"
    },
    {
      "id": "GATE-user-auth-01",
      "title": "Back-pressure gate check",
      "description": "Run all five gate checks per references/gate-task-template.md.",
      "type": "GATE",
      "role_hint": "hyperteam-reviewer",
      "status": "pending",
      "blocked_by": ["FEAT-user-auth-01"],
      "native_task_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
      "started_at": null,
      "completed_at": null,
      "reviewed": false,
      "review_result": null,
      "review_notes": null,
      "reviewed_at": null
    }
  ],
  "gate_iterations": 0
}
```
