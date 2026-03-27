# <PROJECT_NAME>: Architecture Decision Records (ADRs)

This directory contains the Architecture Decision Records (ADRs) for <PROJECT_NAME>.

ADRs are short documents that capture an important architectural or design decision, the context that motivated it, and its consequences.

Martin Fowler explains:

> Software architecture is those decisions which are both important and hard to change.

An Architecture Decision Record is a document that captures an important architectural decision made along with its context and consequences.

It’s a way to capture the important decisions that shape a software's [quality attributes](https://en.wikipedia.org/wiki/List_of_system_quality_attributes).

## What is an ADR?

An ADR is a lightweight record of a significant design decision made during the project's development. It is **not** a
design document for the whole system — that role belongs to the top-level [README.md](../../README.md). Instead, ADRs
serve as **"designs in the small"**: focused, numbered records of a single consequential choice.

**Further reading:**

- [Documenting Architecture Decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions) —
  Michael Nygard's original post that popularised ADRs
- [adr.github.io](https://adr.github.io/) — community tooling, templates, and guidance

______________________________________________________________________

## When to Fetch Individual ADR Files (Agent Guidance)

At session start, read **only this README index** — not the individual ADR files. The index gives you enough topic
awareness to decide when a deeper read is warranted.

Fetch an individual ADR file on-demand when:

- You are about to make a **cross-cutting design choice** (affects multiple modules or layers) and the index suggests an
  existing ADR may constrain it.
- The index reveals an ADR whose topic **directly overlaps** the current task — e.g., you are working on the transaction
  model and ADR-0003 covers the double-entry model.
- You are in a **design or PRD session** where full architectural context is worth the token cost (the `/prd` skill's
  full ADR scan is a deliberate exception, not an oversight).
- You encounter a "why was this done this way?" moment where an ADR title in the index suggests the answer.

Do **not** fetch ADRs speculatively or as a blanket startup step. If no ADR title in the index is relevant to your task,
no individual ADR file needs to be opened.

______________________________________________________________________

## When to Write an ADR

Create an ADR when a decision meets one or more of the following criteria:

- **Cross-cutting**: affects multiple modules or layers (e.g. turn granularity, transaction model)
- **Non-obvious**: future contributors would likely ask "why was this done this way?"
- **Costly to reverse**: changing it later would require significant refactoring
- **Alternative approaches were seriously considered**: the rejected options are worth remembering

You generally do **not** need an ADR for routine implementation choices, stylistic preferences, or decisions that are
entirely local to one file or function.

______________________________________________________________________

## Process

1. **Draft** — copy `0000-adr-template.md`, assign the next available number, and fill in Context and Decision. Set
   status to `Draft`.
1. **Discuss** — open a PR or discussion thread. Collect feedback and revise.
1. **Resolve** — set status to one of:
   - `Accepted` — the decision is adopted
   - `Rejected` — the decision was considered but not adopted; document why
   - `Superseded by ADR-XXXX` — a later ADR replaces this one; update both documents
1. **Commit** — merge the ADR into `main` alongside (or just before) the code it describes.

ADRs are **always additive** — never delete or heavily edit an accepted record. If circumstances change, write a new ADR
that supersedes the old one.

______________________________________________________________________

## File Naming Convention

```
NNNN-short-hyphenated-title.md
```

`NNNN` is a zero-padded, monotonically increasing integer. Example: `0005-intra-turn-causality-and-execution-order.md`.

Start from `0000-adr-template.md` as your template (it is reserved and never represents an actual decision).

______________________________________________________________________

## Index

| ADR | Title | Status |
| --- | ----- | ------ |
