# team-state.json Schema

`plans/<branch>-team-state.json` is the authoritative task state registry for a hyperteam run. Written at Phase 2 start; mutated by lead, reviewer, and teammates throughout. Native task list (via `TaskCreate`/`TaskUpdate`/`TaskList`) is the live coordination bus; `team-state.json` is the durable mirror for re-entrancy and blocker resolution.

## Split ownership

| State | Owner | Rationale |
| ----- | ----- | --------- |
| Live execution status (`pending` → `in_progress`) | Native task list only | Atomic `TaskUpdate`; no file write race |
| Terminal status (`completed` / `failed` / `blocked`) | JSON, appended on transition | Durable; needed for reviewer discovery and re-entrancy |
| Terminal review status (`validated`) | JSON, written by reviewer | Durable; reviewer is authoritative |
| DAG (`blocked_by`, `id`, `title`, `type`, `skills`) | JSON, write-once at Phase 2 start | Structured data native tasks cannot store |
| Review results (`reviewed`, `review_result`, `review_notes`, `reviewed_at`) | JSON, append-only by reviewer | No native task fields for structured review data |
| `gate_iterations` | JSON, incremented by lead | Single counter; not duplicated |

---

## Top-level structure

```json
{
  "metadata": { ... },
  "tasks": [ ... ],
  "gate_iterations": 0
}
```

---

## `metadata`

| Field           | Type             | Description                                                                                                                                                                                                                   |
| --------------- | ---------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `branch`        | string           | Git branch name (e.g. `"feat-user-auth"`).                                                                                                                                                                                    |
| `slug`          | string           | Short identifier derived from branch (e.g. `"user-auth"`).                                                                                                                                                                    |
| `spec_path`      | string           | Relative path to session-spec file (e.g. `"plans/feat-user-auth-session-spec.md"`).                                                                                                                                          |
| `status`        | string           | Overall run status. One of: `"running"`, `"complete"`.                                                                                                                                                                        |
| `source_issues` | string[] \| null | GitHub issues originating this work, each in `"owner/repo#N"` format (e.g. `["samrom3/claude-hyper-plugs#13"]`). `null` or empty when none provided. **MUST NOT be mutated after `team-state.json` is first written.** |
| `created_at`    | string           | ISO 8601 timestamp when file first written.                                                                                                                                                                                   |

### Example

```json
"metadata": {
  "branch": "feat-user-auth",
  "slug": "user-auth",
  "spec_path": "plans/feat-user-auth-session-spec.md",
  "status": "running",
  "source_issues": null,
  "created_at": "2026-03-14T10:00:00Z"
}
```

---

## `tasks`

Array of task objects. Each represents one unit of work from session-spec DAG or gate remediation.

| Field            | Type                    | Default | Description                                                                                                                                                                                                                              |
| ---------------- | ----------------------- | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `id`             | string                  |         | Unique task identifier (e.g. `"FEAT-user-auth-01"`).                                                                                                                                                                                    |
| `title`          | string                  |         | Short human-readable title.                                                                                                                                                                                                              |
| `description`    | string                  |         | Full task description including acceptance criteria.                                                                                                                                                                                     |
| `type`           | string                  |         | One of: `"FEAT"`, `"DOC"`, `"GATE"`.                                                                                                                                                                                                   |
| `skills`         | array of string         | `[]`    | Skills loaded by worker at claim time (names from `hyperloop/skills/hyperwork-*/`). `[]` for no-skill steps (config, env setup, gate).                                                                                                                 |
| `status`         | string                  |         | One of: `"pending"`, `"completed"`, `"validated"`, `"failed"`, `"blocked"`. JSON never holds `"in_progress"` — workers skip that write; `in_progress` lives only in the native task list. |
| `blocked_by`     | array of string         |         | IDs of tasks that must reach `"validated"` (FEAT) or `"completed"` (DOC) before this task unblocks.                                                                                                                                     |
| `started_at`     | string \| null          | `null`  | ISO 8601 timestamp when agent picked up task, or `null`.                                                                                                                                                                                |
| `completed_at`   | string \| null          | `null`  | ISO 8601 timestamp when task reached `"completed"`, or `null`.                                                                                                                                                                          |
| `reviewed`       | boolean                 | `false` | Whether reviewer claimed this task for review. Set to `true` as mutex before review begins. FEAT tasks only.                                                                                                                             |
| `review_result`  | string \| null          | `null`  | Reviewer verdict: `"PASS"`, `"FAIL"`, `"in_progress"`, or `null` if not yet reviewed.                                                                                                                                                  |
| `review_notes`   | array of string \| null | `null`  | Specific findings from reviewer. `null` until review runs; `[]` for clean PASS; list of strings for FAIL findings.                                                                                                                      |
| `reviewed_at`    | string \| null          | `null`  | Timestamp when reviewer completed; `null` until reviewer runs.                                                                                                                                                                          |

### Status transitions

```
pending → completed → validated
        ↘ failed
                    ↘ blocked  (after third reviewer FAIL)
```

Note: `in_progress` exists only in the native task list (via `TaskUpdate`). JSON transitions directly from `pending` to `completed`.

- `pending`: not yet started; blockers may or may not be complete.
- `completed`: agent finished and committed work.
- `validated`: reviewer confirmed work meets acceptance criteria (FEAT tasks only).
- `failed`: worker cannot complete after retries. Lead attempts one re-dispatch before escalating to `blocked`.
- `blocked`: failed review three times; cannot proceed without manual intervention.

### Example task objects

Pending task (not yet started):

```json
{
  "id": "FEAT-user-auth-02",
  "title": "Implement AuthService business logic",
  "description": "As a developer, I want ...\n\nAcceptance Criteria:\n- [ ] ...",
  "type": "FEAT",
  "skills": ["hyperwork-tdd", "hyperwork-python"],
  "status": "pending",
  "blocked_by": ["FEAT-user-auth-01"],
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
  "skills": ["hyperwork-api-scaffold"],
  "status": "validated",
  "blocked_by": [],
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
  "skills": ["hyperwork-tdd", "hyperwork-python"],
  "status": "completed",
  "blocked_by": ["FEAT-user-auth-02"],
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
  "skills": ["hyperwork-tech-writing"],
  "status": "completed",
  "blocked_by": ["FEAT-user-auth-01"],
  "started_at": "2026-03-14T10:50:00Z",
  "completed_at": "2026-03-14T11:30:00Z",
  "reviewed": false,
  "review_result": null,
  "review_notes": null,
  "reviewed_at": null
}
```

---

## `gate_iterations`

Integer. Starts at `0` when file first written. Incremented by 1 each time gate iteration fails (checks 3–5). Iteration guard: if `gate_iterations` is **4 or higher**, reviewer uses `AskUserQuestion` before creating more remediation work.

---

## Full example

```json
{
  "metadata": {
    "branch": "feat-user-auth",
    "slug": "user-auth",
    "spec_path": "plans/feat-user-auth-session-spec.md",
    "status": "running",
    "source_issues": null,
    "created_at": "2026-03-14T10:00:00Z"
  },
  "tasks": [
    {
      "id": "FEAT-user-auth-01",
      "title": "Scaffold UserAuth model and AuthService API stubs",
      "description": "As a developer, I want ...",
      "type": "FEAT",
      "skills": ["hyperwork-api-scaffold"],
      "status": "validated",
      "blocked_by": [],
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
      "skills": [],
      "status": "pending",
      "blocked_by": ["FEAT-user-auth-01"],
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
