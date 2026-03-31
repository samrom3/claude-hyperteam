---
name: adr-supersede
description: "This skill should be used when the user asks to 'supersede an ADR', 'replace an architectural decision', 'update an ADR with a new decision', 'mark an ADR as superseded', or when an existing architectural decision has changed and needs to be replaced while preserving the historical record."
argument-hint: "Old ADR number, optional arrow to existing new ADR number, and the new decision or reason (e.g., '3 Switching from PostgreSQL to CockroachDB for horizontal scaling' or '3->7 CockroachDB chosen to replace PostgreSQL')"
user-invocable: true
---

# adr-supersede

Creates a new ADR that supersedes an existing one, updates the old ADR's status, and maintains
bidirectional cross-references in both files and the directory index.

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

If multiple directories exist, auto-suggest the most relevant based on recent file context
(same logic as `adr-create` Step 2). Present the suggestion and wait for confirmation.

Use the confirmed path as `target_dir`.

## Step 3 — Parse the argument and identify ADRs

Parse the argument (text after `/adr-supersede`) using this format:

```
<old_num>[->new_num] [new decision text or reason]
```

Examples:
- `3 Switching from PostgreSQL to CockroachDB` → old=3, new ADR does not exist yet, decision text provided
- `3->7 CockroachDB chosen to replace PostgreSQL` → old=3, new=7 (ADR-0007 already exists), rationale provided
- `3->7` → old=3, new=7 already exists, no extra text

Resolution:

1. List all files in `target_dir` matching `NNNN-*.md`.
2. **Parse `old_num`** from the argument. Zero-pad to 4 digits and find the matching file as
   `old_adr`. If no argument was given, display the ADR list and ask:
   > Which ADR is being superseded? (enter the number)
3. **Parse `new_num`** if an arrow (`->`) was present:
   - If `new_num` is provided and the file `<new_num_padded>-*.md` exists, set `new_adr` to
     that file. The new ADR already exists — skip Steps 4–6 and go directly to Step 7 to link
     the two ADRs together.
   - If `new_num` is provided but the file does not exist, treat it as a title/number hint for
     the new ADR to be created in Step 6.
   - If no `->` was given, the new ADR must be created (proceed through all steps).
4. Confirm:
   > Superseding: ADR-NNNN — <title>. Proceed? (y/n)
5. If `old_adr`'s current status is already `Superseded by ADR-MMMM`, warn:
   > ADR-NNNN is already superseded by ADR-MMMM. Supersede again? (y/n)
   Proceed only on confirmation.

## Step 4 — Understand the supersession

The skill must understand **why** the old decision is being replaced and **what** the new
decision is. Gather this from multiple sources:

1. **Read the old ADR in full** — its Context, Decision, and Consequences sections provide the
   baseline understanding of what is changing.
2. **Check conversation context** — the user may have just discussed why the old approach no
   longer works or what the replacement should be.
3. **Check the argument** — if the user provided text after `/adr-supersede` (beyond the ADR
   number), use it as the supersession rationale.

From these sources, derive:
- `new_adr_title`: a concise title for the replacement decision.
- `what_changed`: why the original decision is being replaced (new constraints, better
  alternatives, lessons learned, etc.).
- `new_decision`: what the new decision is, stated specifically and declaratively.

If any of these cannot be confidently inferred, interview the user with `AskUserQuestion`.
Batch questions together — for example, if both the new title and rationale are needed:

> I'm superseding ADR-NNNN (<old_title>). To write the replacement ADR, I need:
>
> 1. What is the new decision? (e.g., "Use Redis for session storage instead of PostgreSQL")
> 2. What changed that makes the old decision no longer appropriate?

Proceed only after all three pieces are clear.

## Step 5 — Draft the new ADR sections

Using the old ADR as a foundation and the supersession context from Step 4, draft all sections:

### Context

Write a paragraph explaining the situation. Start from the old ADR's Context (what was the
original problem?), then explain what has changed since — new constraints, growth, incidents,
technology shifts, or lessons learned that invalidate the original decision. Reference the old
ADR by number.

### Decision

Write the new decision clearly and declaratively. Contrast with the old decision where helpful
(e.g., "We will migrate from X to Y because..."). Include the `**Supersedes:** ADR-<old_num>`
cross-reference.

### Consequences

Write 3–7 bullet points covering:
- Benefits of the new approach over the old one
- Migration or transition costs
- Risks or trade-offs of the new decision
- Any follow-up work required (e.g., "migrate existing data", "update deployment configs")
- What becomes easier vs. harder compared to the superseded approach

If consequences cannot be inferred confidently, ask:
> What are the key benefits of the new approach, and what migration or transition costs are
> anticipated?

## Step 6 — Create the new ADR via adr-create

Invoke the `adr-create` skill to create the new ADR. Pass it:
- The `new_adr_title` as the decision summary argument
- The full supersession context (old ADR content + what changed + new decision) so adr-create
  can draft all sections correctly

After adr-create completes, it will have written the new file and updated the index. Capture
the new ADR's path and number as `new_adr` and `next_num`.

Then open the new ADR file and add the cross-reference metadata line after `**Date:**`:
```
**Supersedes:** ADR-<old_num_padded>
```

Save the file.

## Step 7 — Link the superseded ADR

1. Open `old_adr`.
2. Find the `**Status:**` line. Replace its value with:
   `Superseded by ADR-<next_num_padded>`
3. If `new_adr` already existed (the `->` path), also add a `**Superseded by:**` metadata line
   after `**Date:**` if it is not already present.
4. Save the file.

## Step 8 — Update the index

1. Open `<target_dir>/README.md`.
2. Find the row for the old ADR. Update its `Status` column to
   `Superseded by ADR-<next_num_padded>`.
3. If the new ADR was just created by adr-create, its index row was already added — skip adding
   a duplicate row. If the new ADR already existed and is not yet in the index, add it now.
4. Save `README.md`.

## Step 9 — Confirm

Inform the user:

> ADR-<old_num_padded> → ADR-<next_num_padded>: <new_adr_title>
> Updated ADR-<old_num_padded>: Status → "Superseded by ADR-<next_num_padded>"
> Updated index: `<target_dir>/README.md`
>
> Review the new ADR and change the Status to `Accepted` when finalised.

## Step 10 — Post-write validation

Invoke `adr-check` in scoped mode against the superseded ADR (the file this skill directly
modified in Step 7):

```
/adr-check <old_adr_path>
```

Display all output to the user.

- If `adr-check` returns a structural **FAIL**: block completion and prompt the user to resolve
  the issue before proceeding:
  > The superseded ADR has a structural validation failure. Please fix the issue above before
  > confirming this supersession is complete.
- If `adr-check` emits style warnings only: display them and continue. Style warnings are
  informational and do not block completion.

Note: the new ADR (created via `adr-create` in Step 6) is already validated by `adr-create`'s
own post-write step — do not run `adr-check` on it again here.
