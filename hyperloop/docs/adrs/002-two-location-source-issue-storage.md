# ADR-002: Two-Location `source_issue` Storage

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
- **`team-state.json` `metadata.source_issue`** — a structured field in the durable JSON state
  file that all agents read at runtime.

A single-location design was considered:

- **PRD only:** Phase 1 and Phase 4 would each need to parse Markdown to extract the issue
  reference, coupling them to the PRD format and making the parsing contract implicit.
- **`team-state.json` only:** The PRD would not be self-describing; a reader could not tell from
  the document which issue it addresses.

______________________________________________________________________

## Decision

Store `source_issue` in **both** locations, with a clear authority hierarchy:

| Location                                  | Role                                                     | Mutability                     |
| ----------------------------------------- | -------------------------------------------------------- | ------------------------------ |
| PRD metadata table                        | Human-visible record; written once by `/prd`             | Immutable after PRD is written |
| `team-state.json` `metadata.source_issue` | Authoritative runtime value; read by Phase 1 and Phase 4 | Immutable after first write    |

**Write path:** `/prd` (Phase 0) detects the issue URL, writes the PRD metadata table, and
assigns the issue. Phase 1 reads the PRD metadata table and copies the value into
`team-state.json`.

**Read path:** All agents (Phase 1 assignment verification, Phase 4 PR creation) read
`metadata.source_issue` from `team-state.json`. No agent re-parses the PRD after Phase 1.

### Canonical metadata table format

The metadata table format is a binding contract between `/prd` (writer) and Phase 1 (reader).
It must be followed exactly:

```markdown
# <Title>

| Field        | Value        |
| ------------ | ------------ |
| Source Issue | owner/repo#N |

## 1. Introduction/Overview
```

**Placement rules:**

- The table appears **immediately after the H1 heading** (one blank line separating them).
- The table appears **before the first `##` section heading**.
- If no issue URL was provided, the table is **omitted entirely** — the H1 heading is followed
  directly by `## 1.`.

**Parsing contract for agents:**

1. Locate the H1 heading (`# ...`).
1. Scan forward line by line.
1. If a `| Source Issue |` row is found **before** the first `##` line, extract the second cell
   value as `source_issue` (e.g. `samrom3/claude-hyper-plugs#13`).
1. If a `##` line is reached before any `| Source Issue |` row, `source_issue` is `null`.

**Immutability:** `source_issue` MUST NOT be mutated after `team-state.json` is first written.
It records the originating issue at the moment work began and must remain stable for the lifetime
of the run.

______________________________________________________________________

## Consequences

**Positive:**

- **PRDs are self-describing.** Any reader — human or agent — can open the `.md` file and
  immediately see the linked issue without consulting `team-state.json`.
- **Agents have a stable, structured read path.** Phase 1, Phase 4, and any future phase read
  `metadata.source_issue` from JSON. No Markdown parsing after the initial propagation step.
- **Single point of mutation.** The PRD table is written once by `/prd`; `team-state.json` is
  written once by Phase 1. Immutability after those writes prevents drift.

**Neutral:**

- Two locations must stay consistent. Because `team-state.json` is derived from the PRD (not
  vice versa), and both are immutable after creation, there is no synchronisation burden — the
  value is written once in each location and never updated.

**Negative / Trade-offs:**

- Phase 1 bears a one-time parsing step to copy the value from Markdown to JSON. This is bounded
  and low-risk because the format is tightly specified above.
