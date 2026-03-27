# Phase 1 — Fresh Start

This path is taken when `plans/<branch>-team-state.json` does **not** exist.

______________________________________________________________________

## Step 1 — Read and parse the PRD

1. Read `plans/<branch>-prd.md` in full.
2. Extract all developer stories. Stories are headings that match:
   - `### FEAT-*` — feature implementation stories
   - `### DOC-*` — documentation stories
3. For each story, capture:
   - **Title** — the heading text after the `### ` prefix (strip the leading ID prefix if present,
     e.g. `### FEAT-hyperteam-skill-04: Phase 1` → title is `Phase 1`)
   - **Description** — all body text under that heading until the next `###` heading
4. Preserve the order stories appear in the PRD — this is the dependency order.

> **Note:** The scaffold-first pattern is applied by the `/prd` skill when the PRD is authored.
> Phase 1 does not re-split stories; it maps each PRD story to exactly one task entry.

______________________________________________________________________

## Step 2 — Assign task IDs

Assign IDs sequentially in the order stories appear in the PRD, using two-digit zero-padded
counters:

- FEAT stories → `FEAT-<slug>-01`, `FEAT-<slug>-02`, … (independent counter)
- DOC stories → `DOC-<slug>-01`, `DOC-<slug>-02`, … (independent counter)
- GATE → `GATE-<slug>-01` (always exactly one, created last)

______________________________________________________________________

## Step 3 — Infer dependencies

1. Build an ordered list of all FEAT and DOC task IDs in PRD order.
2. For each task, set `blocked_by` to the IDs of all tasks that appear **before** it in the PRD
   and that it logically requires. Use the following heuristic:
   - Each FEAT task is blocked by the FEAT task immediately preceding it (linear chain by default).
   - DOC tasks that document a feature area block the FEAT tasks covering that same area.
   - If two tasks are clearly independent (different modules, no shared interfaces), they may run
     in parallel — omit the dependency between them.
   - Prefer the minimal set of necessary gates; allow parallelism where possible.
3. The `GATE-<slug>-01` task is **always** blocked by **all** FEAT and DOC tasks.

______________________________________________________________________

## Step 4 — Render ASCII DAG

Render a tree showing tasks, their titles, and dependency arrows. Use `└──` for children (tasks
that are blocked by a parent). Mark parallel tasks with `[parallel]` and the gate with
`[blocked by all]`.

Example format:

```
Branch: <branch>

FEAT-<slug>-01 — <title>
└── FEAT-<slug>-02 — <title>
    └── FEAT-<slug>-03 — <title>
        └── ...

DOC-<slug>-01 — <title> [parallel]

GATE-<slug>-01 — Back-pressure gate [blocked by all]
```

Adjust the tree to reflect the actual dependency structure derived in Step 3. Tasks blocked only
by the root appear as children of that root. Truly independent parallel chains appear as separate
top-level entries.

______________________________________________________________________

## Step 5 — Ask for user approval

Use `AskUserQuestion` to present the full rendered DAG and ask:

> Here is the proposed task plan for `<branch>`:
>
> ```
> <rendered ASCII DAG>
> ```
>
> Does this plan look correct? Approve to proceed, or describe any changes you'd like to make.

Wait for the user's response.

- **If approved** — proceed to Step 5b.
- **If changes requested** — apply the requested changes, re-render the DAG, and ask again. Repeat
  until the user approves.

______________________________________________________________________

## Step 5b — Assign role hints

For each task, assign a `role_hint` using the following rules:

**Core roles (always available):**

| Condition | `role_hint` |
|-----------|-------------|
| DOC tasks | `hyperteam-techwriter` |
| GATE tasks | `hyperteam-reviewer` |

**Language-pack roles (when matching agents are installed):**

Check whether language-specific agents exist in the project's `.claude/agents/` or the
plugin's `agents/` directory (all plugin agents live flat in `agents/` for auto-discovery).
If language-pack agents are present, apply their heuristics BEFORE falling back to `hyperteam-worker`.

For example, with the Python pack installed:

| Condition | `role_hint` |
|-----------|-------------|
| FEAT task first in its chain, OR title contains: scaffold, stub, api, dataclass, interface, abc | `hyperteam-py-api-scaffolder` |
| Other FEAT tasks | `hyperteam-py-builder` |

**Fallback (no language-pack match):**

| Condition | `role_hint` |
|-----------|-------------|
| Any unmatched task | `hyperteam-worker` |

______________________________________________________________________

## Step 6 — Write team-state.json

Once the user approves the plan, write `plans/<branch>-team-state.json` (schema:
`references/team-state-schema.md`):

```json
{
  "metadata": {
    "branch": "<branch>",
    "slug": "<slug>",
    "prd_path": "plans/<branch>-prd.md",
    "status": "running",
    "created_at": "<ISO 8601 timestamp>"
  },
  "tasks": [
    {
      "id": "FEAT-<slug>-01",
      "title": "<story title>",
      "description": "<full story text including acceptance criteria>",
      "type": "FEAT",
      "role_hint": "hyperteam-py-api-scaffolder",
      "status": "pending",
      "blocked_by": [],
      "native_task_id": null,
      "started_at": null,
      "completed_at": null,
      "reviewed": false,
      "review_result": null,
      "review_notes": null,
      "reviewed_at": null
    },
    {
      "id": "DOC-<slug>-01",
      "title": "<story title>",
      "description": "<full story text>",
      "type": "DOC",
      "role_hint": "hyperteam-techwriter",
      "status": "pending",
      "blocked_by": ["FEAT-<slug>-01"],
      "native_task_id": null,
      "started_at": null,
      "completed_at": null,
      "reviewed": false,
      "review_result": null,
      "review_notes": null,
      "reviewed_at": null
    },
    {
      "id": "GATE-<slug>-01",
      "title": "Back-pressure gate",
      "description": "Run all five gate checks per references/gate-task-template.md.",
      "type": "GATE",
      "role_hint": "hyperteam-reviewer",
      "status": "pending",
      "blocked_by": ["<all FEAT and DOC task IDs>"],
      "native_task_id": null,
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

Rules:

- `metadata.created_at` — current UTC timestamp in ISO 8601 format (e.g. `"2026-03-14T10:00:00Z"`).
- All tasks have `"status": "pending"` and all timestamp/result fields `null`.
- All tasks have `"native_task_id": null` — native tasks are seeded by Phase 2 Step 3.
- All FEAT tasks have `"reviewed": false`.
- Task order: FEAT tasks first (in PRD order), then DOC tasks (in PRD order), then the GATE task.
- `blocked_by` arrays contain the exact task IDs (strings) derived in Step 3.
- `role_hint` values are assigned per Step 5b.

After writing the file, return to SKILL.md and proceed to Phase 2.
