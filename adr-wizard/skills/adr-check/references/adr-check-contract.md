# ADR-Check Contract

This document defines the generic contract that the `adr-check` skill implements and that gate
agents (such as the hyperloop gate) consume. The contract is convention-based — no runtime
dependency between plugins. Any skill that satisfies this interface can serve as the ADR
validator for a gate.

---

## 1. Discovery — Inputs

The skill MUST discover ADR directories using the following priority order:

1. **CLAUDE.md convention (primary):** Search the project's `CLAUDE.md` for any heading
   containing the text `ADR Locations` (case-insensitive, any heading level). Under that heading,
   read each bullet list item as a relative path to an ADR directory. Inline `# comment`
   annotations after the path are ignored when parsing. Example:

   ```markdown
   ### ADR Locations
   - docs/adrs/           # project-wide decisions
   - hyperloop/docs/adrs/ # hyperloop plugin decisions
   ```

2. **Fallback (when no ADR Locations heading exists):** Scan the repository root for directories
   matching these common patterns: `docs/adrs/`, `decisions/`, `architecture/decisions/`.

If no ADR directories are found via either method, the skill MUST report this as a **warning**
(not a failure) and exit with a passing result. A project with no ADR directories has nothing to
validate.

---

## 2. Validation — What Is Checked

For each discovered ADR directory, the skill runs these checks:

### 2.1 ADR File Structure

Each file matching `NNNN-*.md` (where NNNN is a zero-padded 4-digit number) in the directory is
an ADR file. For each ADR file, verify:

- The file contains a `## Status` section (or equivalent heading) with a non-empty value.
- The file contains a `## Context` section with non-trivial content (more than one blank line
  or placeholder text like "TODO").
- The file contains a `## Decision` section with non-trivial content.
- The file contains a `## Consequences` section (may be brief; existence is sufficient).

### 2.2 Index Sync

The ADR directory MUST contain a `README.md` file serving as the index. Verify:

- Every ADR file in the directory has a corresponding entry in the `README.md` index.
- Every entry in the `README.md` index has a corresponding ADR file in the directory.
  (No orphaned index entries pointing to missing files.)

### 2.3 Cross-Reference Integrity

For ADRs with a `Superseded by ADR-NNNN` status:

- The referenced ADR (NNNN) must exist in the directory.
- The referenced ADR must contain a `Supersedes: ADR-MMMM` reference back to the superseded ADR.

For ADRs with a `Supersedes: ADR-NNNN` field:

- The referenced ADR (NNNN) must exist.
- The referenced ADR's status must be `Superseded by ADR-<this>`.

---

## 3. Output — Validation Report

The skill MUST output a structured report in the following format:

```
ADR Check Report
================

Directory: <path>
  Status: PASS | FAIL
  Issues:
    - [ADR-NNNN] <description of issue>
    - ...

Directory: <path>
  Status: PASS
  Issues: none

Overall: PASS | FAIL
```

On failure, each issue entry MUST include:
- Which ADR file is affected (by number and filename)
- What check failed
- A specific remediation step (e.g., "Add a non-empty ## Context section to docs/adrs/003-foo.md")

---

## 4. Diff-Based Warnings

After structural validation, the skill MUST scan `git diff HEAD` (staged + unstaged) for
patterns that suggest undocumented architectural decisions. These are surfaced as **warnings**
— they do NOT affect the pass/fail result and do NOT block any gate.

Patterns to detect:

| Pattern | Warning message |
|---------|----------------|
| New abstract class or interface definition (Python ABC, Java `interface`, Go `interface{}` with multiple methods, TypeScript `interface`/`abstract class`) | "New interface/ABC detected in \<file\> — consider documenting the design decision" |
| New file matching `*Config*`, `*Settings*`, `*Configuration*` | "New configuration file detected — consider documenting configuration decisions" |
| Addition of a new dependency in `package.json`, `pyproject.toml`, `Cargo.toml`, or `go.mod` | "New dependency added — consider documenting the technology choice" |
| New top-level directory or new subdirectory named `service`, `module`, `component`, `pkg`, `lib` | "New service/module boundary detected — consider documenting the architectural boundary" |

Warnings are appended to the report after the main validation output:

```
Diff-Based Warnings (advisory only — not gate-blocking)
========================================================
  - [WARNING] <pattern description>: <file or location>
```

If no diff-based warnings are found, this section is omitted from the report.

---

## 5. Pass / Fail Semantics

| Condition | Result |
|-----------|--------|
| All structural checks pass for all directories | **PASS** |
| Any structural check fails in any directory | **FAIL** |
| No ADR directories found | **PASS** (warning emitted) |
| Diff-based warnings present | **PASS** (warnings are informational) |

Gate consumers MUST treat a **FAIL** result as a gate-blocking failure. Gate consumers MUST
treat diff-based warnings as informational only — they must be logged in the progress file and
presented to the user, but they do not block the gate.

---

## 6. Invocation

- **User-invocable:** `/adr-check`
- **Model-invocable:** The skill description includes phrases matching ADR validation context
  so the model can auto-trigger it when relevant.
- **Gate invocation:** The gate template calls the skill by name. If the skill is not installed,
  the gate falls back to basic directory scanning (see FR-011 in the PRD).
