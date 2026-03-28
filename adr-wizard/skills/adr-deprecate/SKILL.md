---
name: adr-deprecate
description: "Deprecate an existing Architecture Decision Record (ADR) that is no longer relevant, recording the reason while preserving history. Use when an architectural decision, design choice, or technical decision is obsolete or no longer applicable. Invocable via /adr-deprecate <ADR number> <reason for deprecation>."
argument-hint: "ADR number to deprecate and why (e.g., '5 No longer applicable after migration to microservices')"
user-invocable: true
---

# adr-deprecate

Marks an existing ADR as deprecated, records the reason in the file, and updates the directory's
README.md index.

> ADRs are **additive only**: never delete or heavily rewrite an accepted ADR. This skill exists
> so that the history of decisions is preserved — the old ADR remains readable, and the new ADR
> explains what changed and why.

---

## Step 1 — Discover ADR directories

Follow the same discovery procedure as `adr-create` (Step 1):

1. Search `CLAUDE.md` for any heading containing `ADR Locations`. Collect bullet paths into
   `adr_dirs`, stripping inline `# comments`.
2. If not found, fall back to scanning for `docs/adrs`, `decisions`, `architecture/decisions`.
3. If empty, inform the user and stop.

## Step 2 — Select target directory

If multiple directories exist, ask the user which directory contains the ADR to deprecate, or
auto-suggest the most relevant directory based on recent file context (same logic as `adr-create`
Step 2). Wait for confirmation.

## Step 3 — Identify the ADR to deprecate

1. List all files in `target_dir` matching `NNNN-*.md`.
2. If the user invoked the skill with an ADR number (e.g., `/adr-deprecate 3`), use it
   directly: find the file matching `0003-*.md`.
3. Otherwise, display the list of current ADRs with their numbers, titles, and statuses, and
   ask:
   > Which ADR do you want to deprecate? (enter the ADR number or filename)
4. Confirm the selection:
   > Deprecating: ADR-NNNN — <title>. Proceed? (y/n)
5. If the ADR's current status is already `Deprecated`, warn the user and stop:
   > ADR-NNNN is already deprecated.

## Step 4 — Understand and draft the deprecation

The skill must understand **why** this ADR is being deprecated and produce a clear, informative
deprecation note — not just a one-line reason. Gather context from multiple sources:

1. **Read the ADR in full** — understand what decision it records and its current consequences.
2. **Check conversation context** — the user may have just discussed why the decision is
   obsolete, what replaced it, or what changed in the project.
3. **Check the argument** — if the user provided text after the ADR number
   (e.g., `/adr-deprecate 3 migrated to microservices`), use it as the deprecation rationale.

From these sources, draft:
- `deprecation_reason`: a concise one-line summary for the `**Deprecated:**` metadata field
  (e.g., "No longer applicable after migration to microservices architecture").
- `deprecation_note`: a 2–4 sentence paragraph to append to the ADR's `## Context` section (or
  add as a new `## Deprecation Note` section) explaining:
  - What changed that makes this decision no longer relevant
  - Whether a replacement decision exists (reference by ADR number if so)
  - Any cleanup or migration that resulted from this deprecation

If the reason cannot be confidently inferred from context, interview the user with
`AskUserQuestion`:

> I'm deprecating ADR-NNNN (<title>). To write a proper deprecation record, I need:
>
> 1. Why is this decision no longer relevant? (What changed?)
> 2. Has it been replaced by another decision, or is it simply obsolete?
> 3. Was any cleanup or migration done as a result?

Batch all needed questions into a single `AskUserQuestion` call. Proceed only after the
deprecation reason and note are clear.

## Step 5 — Update the ADR file

1. Open the ADR file.
2. Find the `**Status:**` line. Replace its value with `Deprecated`.
3. After the `**Status:**` line (or after `**Date:**` if Status comes before Date), add:
   ```
   **Deprecated:** <deprecation_reason>
   ```
4. Append a `## Deprecation Note` section at the end of the file (before any trailing blank
   lines) with `deprecation_note`. This section provides future readers with the full context of
   why this decision was deprecated.
5. Save the file.

## Step 6 — Update the index

1. Open `<target_dir>/README.md`.
2. Find the row for this ADR in the table (match by filename or ADR number).
3. Update the `Status` column value in that row to `Deprecated`.
4. Save the file.

## Step 7 — Confirm

Inform the user:

> Deprecated ADR-NNNN: <title>
> Reason: <deprecation_reason>
> Updated: `<target_dir>/<NNNN>-<slug>.md`
> Updated index: `<target_dir>/README.md`

## Step 8 — Post-write validation

Invoke `adr-check` in scoped mode against the deprecated ADR file:

```
/adr-check <deprecated_adr_path>
```

Display all output to the user.

- If `adr-check` returns a structural **FAIL**: block completion and prompt the user to resolve
  the issue before proceeding:
  > The deprecated ADR has a structural validation failure. Please fix the issue above before
  > confirming this deprecation is complete.
- If `adr-check` emits style warnings only: display them and continue. Style warnings are
  informational and do not block completion.
