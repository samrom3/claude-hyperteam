# Phase 1 — Fresh Start

Taken when `plans/<branch>-team-state.json` does **not** exist.

---

## Step 1 — Read and parse the spec

1. Read `plans/<branch>-session-spec.md` in full.
   > **Metadata table:** If `/session-spec` invoked with GitHub issue URL, metadata table appears **immediately after H1 heading** and **before first `##` section heading**:
   >
   > ```markdown
   > | Field        | Value        |
   > | ------------ | ------------ |
   > | Source Issue | owner/repo#N |
   > ```
   >
   > Locate H1, scan forward collecting all `| Source Issue |` rows before any `##` line. Extract second cell from each matching row → `<source_issues>` list (e.g. `["samrom3/claude-hyper-plugs#13"]`). No rows before first `##` → `<source_issues> = null`.
2. Extract all steps. Headings matching:
   - `### STEP-*` — default; maps to FEAT task
   - `### FEAT-*` — explicit FEAT (also accepted)
   - `### DOC-*` — documentation task
3. Per step, capture:
   - **Type** — `DOC` if heading prefix is `DOC-`; `FEAT` otherwise
   - **Title** — heading text after `### ` prefix (strip leading prefix+ID if present, e.g. `### STEP-foo-01: Bar` → `Bar`)
   - **Description** — all body text under heading until next `###`
   - **Skills** — scan description for `> skills: <value>` line. `<value>` is comma-separated skill names → split to array. `none` or absent → `[]`.
4. Preserve spec step order — this is dependency order.

> **Note:** Scaffold-first pattern applied by `/session-spec` at authoring time. Phase 1 maps each spec step to exactly one task entry; no re-splitting.

---

## Step 2 — Assign task IDs

Sequential IDs in spec order, two-digit zero-padded counters:

- FEAT steps (includes `STEP-*` and `FEAT-*`) → `FEAT-<slug>-01`, `FEAT-<slug>-02`, … (independent counter)
- DOC steps → `DOC-<slug>-01`, `DOC-<slug>-02`, … (independent counter)
- GATE → `GATE-<slug>-01` (always exactly one, created last)

---

## Step 3 — Infer dependencies

1. Build ordered list of all FEAT and DOC task IDs in spec order.
2. Per task, set `blocked_by` to IDs of tasks appearing **before** it that it logically requires:
   - Each FEAT task blocked by immediately preceding FEAT (linear chain by default).
   - DOC tasks documenting a feature area block FEAT tasks covering same area.
   - Clearly independent tasks (different modules, no shared interfaces) → parallel, omit dependency.
   - Prefer minimal necessary gates; allow parallelism where possible.
3. `GATE-<slug>-01` **always** blocked by **all** FEAT and DOC tasks.

---

## Step 4 — Render ASCII DAG

Render tree: tasks, titles, dependency arrows. `└──` for children (blocked by parent). `[parallel]` for parallel tasks. `[blocked by all]` for gate.

Example:

```
Branch: <branch>

FEAT-<slug>-01 — <title>
└── FEAT-<slug>-02 — <title>
    └── FEAT-<slug>-03 — <title>
        └── ...

DOC-<slug>-01 — <title> [parallel]

GATE-<slug>-01 — Back-pressure gate [blocked by all]
```

Adjust tree to reflect actual dependency structure. Tasks blocked only by root → root's children. Independent parallel chains → separate top-level entries.

---

## Step 5 — Ask for user approval

Use `AskUserQuestion`:

> Here is the proposed task plan for `<branch>`:
>
> ```
> <rendered ASCII DAG>
> ```
>
> Does this plan look correct? Approve to proceed, or describe any changes you'd like to make.

Wait for response.

- **Approved** → proceed to Step 6.
- **Changes requested** → apply changes, re-render DAG, ask again. Repeat until approved.

---

## Step 6 — Write team-state.json

On user approval, write `plans/<branch>-team-state.json` (schema: `references/team-state-schema.md`):

```json
{
  "metadata": {
    "branch": "<branch>",
    "slug": "<slug>",
    "spec_path": "plans/<branch>-session-spec.md",
    "status": "running",
    "source_issues": "<array from spec metadata table, or null>",
    "created_at": "<ISO 8601 timestamp>"
  },
  "tasks": [
    {
      "id": "FEAT-<slug>-01",
      "title": "<step title>",
      "description": "<full step text including acceptance criteria>",
      "type": "FEAT",
      "skills": ["<skill_name>"],
      "status": "pending",
      "blocked_by": [],
      "started_at": null,
      "completed_at": null,
      "reviewed": false,
      "review_result": null,
      "review_notes": null,
      "reviewed_at": null
    },
    {
      "id": "DOC-<slug>-01",
      "title": "<step title>",
      "description": "<full step text>",
      "type": "DOC",
      "skills": ["hyperwork-tech-writing"],
      "status": "pending",
      "blocked_by": ["FEAT-<slug>-01"],
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
      "skills": [],
      "status": "pending",
      "blocked_by": ["<all FEAT and DOC task IDs>"],
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

- `metadata.source_issues` — array from spec metadata table Step 1 (e.g. `["owner/repo#N"]`, or `null` if none). **MUST NOT be mutated after first write.**
- `metadata.created_at` — current UTC timestamp, ISO 8601 (e.g. `"2026-03-14T10:00:00Z"`).
- All tasks: `"status": "pending"`, all timestamp/result fields `null`.
- All FEAT tasks: `"reviewed": false`.
- Task order: FEAT (spec order) → DOC (spec order) → GATE.
- `blocked_by` arrays: exact task ID strings from Step 3.
- `skills` arrays: from Step 1 annotation parsing (`skills: none` → `[]`).

**After writing, per issue in `metadata.source_issues` (skip if null/empty):**

1. Parse `owner/repo#N` → `<owner>`, `<repo>`, `<N>`.
2. Run:
   ```
   gh issue view <N> --repo <owner>/<repo> --json assignees
   ```
   Check if authenticated `gh` user (from `gh auth status`) appears in `assignees`.
3. If not in assignees:
   ```
   gh issue edit <N> --repo <owner>/<repo> --add-assignee @me
   ```
4. Command fails (unauthenticated, network error, permissions) → print warning (`⚠ Warning: could not verify/assign issue — <error>`) and continue. Do **NOT** block team creation.

After processing all issues (or immediately if `source_issues` null) → return to SKILL.md, proceed to Phase 2.
