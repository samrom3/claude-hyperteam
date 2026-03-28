# ADR-002: Two-Location `source_issues` Storage

**Status:** Accepted

**Date:** 2026-03-27

______________________________________________________________________

## Context

When a user invokes `/prd` with a GitHub issue URL, hyperloop needs to track the originating issue
so that:

1. The PRD is self-describing — a reader opening the `.md` file can see at a glance which issue
   it addresses.
1. Phase 1 and Phase 4 can access the issue reference without re-parsing the PRD at runtime.
1. The PR auto-closes the issue via a `Closes` keyword when merged.

Two candidate storage locations exist:

- **PRD metadata table** — a Markdown table written immediately after the H1 heading of
  `plans/<branch>-prd.md`. Human-readable, travels with the PRD file, and visible in GitHub's
  rendered Markdown view.
- **`team-state.json` `metadata.source_issues`** — a structured array field in the durable JSON
  state file that all agents read at runtime.

A single-location design was considered:

- **PRD only:** Phase 1 and Phase 4 would each need to parse Markdown to extract the issue
  reference, coupling them to the PRD format and making the parsing contract implicit.
- **`team-state.json` only:** The PRD would not be self-describing; a reader could not tell from
  the document which issue it addresses.

______________________________________________________________________

## Decision

Store `source_issues` in **both** locations, with a clear authority hierarchy:

| Location                                   | Role                                                     | Mutability                     |
| ------------------------------------------ | -------------------------------------------------------- | ------------------------------ |
| PRD metadata table                         | Human-visible record; written once by `/prd`             | Immutable after PRD is written |
| `team-state.json` `metadata.source_issues` | Authoritative runtime value; read by Phase 1 and Phase 4 | Immutable after first write    |

**Write path:** `/prd` (Phase 0) detects all issue URLs in `$ARGUMENTS`, writes one `| Source Issue |`
row per issue in the PRD metadata table, and assigns each issue. Phase 1 collects all rows and
copies the list into `team-state.json` as `source_issues`.

**Read path:** All agents (Phase 1 assignment verification, Phase 4 PR creation) read
`metadata.source_issues` from `team-state.json`. No agent re-parses the PRD after Phase 1.

### Array type rationale

`source_issues` is typed `string[] | null` rather than `string | null` because a single PR
commonly closes more than one issue. Changing `string` → `string[]` post-release would be a
breaking schema change requiring a major version bump; adopting the array type from the outset
is lower cost. A single-issue PRD produces a one-element array; the `null` case (no issues)
is preserved for full backwards compatibility with no-issue PRDs.

### Canonical metadata table format

The metadata table format is a binding contract between `/prd` (writer) and Phase 1 (reader).
It must be followed exactly:

```markdown
# <Title>

| Field        | Value                         |
| ------------ | ----------------------------- |
| Source Issue | owner/repo#N                  |
| Source Issue | owner/repo#M                  |

## 1. Introduction/Overview
```

For a single issue, the table contains exactly one `Source Issue` row.

**Placement rules:**

- The table appears **immediately after the H1 heading** (one blank line separating them).
- The table appears **before the first `##` section heading**.
- If no issue URL was provided, the table is **omitted entirely** — the H1 heading is followed
  directly by `## 1.`.

**Parsing contract for agents:**

1. Locate the H1 heading (`# ...`).
1. Scan forward line by line.
1. Collect **all** `| Source Issue |` rows found **before** the first `##` line; extract the
   second cell value from each row and build the `source_issues` list.
1. If a `##` line is reached before any `| Source Issue |` row, `source_issues` is `null`.

**Immutability:** `source_issues` MUST NOT be mutated after `team-state.json` is first written.
It records the originating issues at the moment work began and must remain stable for the lifetime
of the run.

______________________________________________________________________

## Consequences

**Positive:**

- **PRDs are self-describing.** Any reader — human or agent — can open the `.md` file and
  immediately see the linked issues without consulting `team-state.json`.
- **Agents have a stable, structured read path.** Phase 1, Phase 4, and any future phase read
  `metadata.source_issues` from JSON. No Markdown parsing after the initial propagation step.
- **Single point of mutation.** The PRD table is written once by `/prd`; `team-state.json` is
  written once by Phase 1. Immutability after those writes prevents drift.
- **Multi-issue native.** The array type accommodates PRs that close more than one issue without
  any future schema change.

**Neutral:**

- Two locations must stay consistent. Because `team-state.json` is derived from the PRD (not
  vice versa), and both are immutable after creation, there is no synchronisation burden — the
  value is written once in each location and never updated.

**Negative / Trade-offs:**

- Phase 1 bears a one-time parsing step to copy the value from Markdown to JSON. This is bounded
  and low-risk because the format is tightly specified above.
