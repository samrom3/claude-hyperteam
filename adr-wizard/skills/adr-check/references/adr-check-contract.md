# ADR-Check Contract

This document defines the generic contract that the `adr-check` skill implements and that gate
agents (such as the hyperloop gate) consume. The contract is convention-based — no runtime
dependency between plugins. Any skill that satisfies this interface can serve as the ADR
validator for a gate.

---

## 1. Discovery — Inputs

### 1.1 Whole-Directory Mode (default)

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

### 1.2 Scoped Mode

When invoked with an argument, the skill enters **scoped mode**. The argument determines the
target scope — three sub-modes are supported:

- **Scoped (file):** The argument is a path to a single `.md` file
  (e.g., `/adr-check docs/adrs/0001-foo.md`). Only that file is validated. Runs Section 2.1
  (structural) and Section 2.1e (style check) only. Sections 2.2, 2.3, and 4 are skipped.

- **Scoped (directory):** The argument is a path to a directory
  (e.g., `/adr-check docs/adrs/`). All ADR files in that directory are validated. Runs
  Sections 2.1, 2.1e, 2.2, and 2.3 for that directory only. Section 4 is skipped.

- **Scoped (query):** The argument is a natural-language description
  (e.g., `/adr-check "validate only ADRs impacting the Zip Event management components"`).
  The skill uses model judgment to identify which ADR files across all discovered directories
  match the query. Runs Section 2.1 and Section 2.1e against each matched file. Sections 2.2,
  2.3, and 4 are skipped.

All scoped sub-modes skip Section 4 (Diff-Based Warnings). Section 4 is exclusive to Global mode.

Scoped mode is designed for intentional, targeted validation: lifecycle skills use
`Scoped (file)` for post-write validation; users and tools can use `Scoped (directory)` or
`Scoped (query)` to narrow a check to a relevant subset without diff-based noise.

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

### 2.1e Style Check — Consequences Quality

This check fires on every validated ADR file (in both whole-directory and scoped mode).

The skill uses model judgment — not keyword matching — to assess whether the `## Consequences`
section contains at least one genuinely adverse outcome. Qualifying examples include: a known
risk, a trade-off, a migration cost, an increase in complexity, a performance regression, reduced
flexibility, or an explicit statement of what is being given up.

If no such adverse consequence is found, the skill emits a **style warning** (not a structural
failure):

```
[ADR-NNNN] Consequences section contains no clearly adverse consequence or trade-off — consider adding one for credibility
```

Style warnings are informational. They do not affect the pass/fail result.

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

The skill MUST output a structured report. The format is identical regardless of mode; the
`Mode` and `ADRs` header rows provide context for all rows that follow.

```
ADR Check Report
================

Mode:    Global | Scoped (file) | Scoped (directory) | Scoped (query)
Target:  <path, directory path, or natural-language query>
ADRs:    <comma-separated list of all ADR filenames validated, e.g. "0001-foo.md, 0002-bar.md">

Results
-------
<path/to/directory-or-file>:
  Status: PASS | FAIL
  Issues:
    - [ADR-NNNN] <description of issue>

<path/to/next-item>:
  Status: PASS
  Issues: none

Style Warnings (advisory only — not gate-blocking)
===================================================
  - [ADR-NNNN] <style warning message>

Diff-Based Warnings (advisory only — not gate-blocking)
========================================================
  - [WARNING] <message>

Overall: PASS | FAIL
```

**Results grouping:**
- **Global** and **Scoped (directory):** Group results by directory path.
- **Scoped (file)** and **Scoped (query):** List each validated file as its own result entry.

**Conditional sections:**
- `Style Warnings` — appears only when Section 2.1e emits at least one warning; omitted otherwise.
- `Diff-Based Warnings` — appears only in Global mode when Section 4 emits warnings; omitted in
  all scoped sub-modes and when no diff warnings are found.

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

| Mode | Condition | Result |
|------|-----------|--------|
| Global | All structural checks pass | **PASS** |
| Global | Any structural check fails | **FAIL** |
| Global | No ADR directories found | **PASS** (warning emitted) |
| Global | Diff-based warnings present | **PASS** (informational only) |
| Global | Style warnings present | **PASS** (informational only) |
| Scoped (any) | Structural check fails on target | **FAIL** |
| Scoped (any) | Style warning emitted | **PASS** (informational only) |
| Scoped (any) | Diff-based warnings | Skipped — not evaluated |
| Scoped (query) | No ADRs match the query | **PASS** (advisory warning emitted) |

Gate consumers MUST treat a **FAIL** result as a gate-blocking failure. Gate consumers MUST
treat diff-based warnings as informational only — they must be logged in the progress file and
presented to the user, but they do not block the gate.

---

## 6. Invocation

- **Global mode:** `/adr-check`
  No argument. Validates all ADR directories discovered via Section 1.1. Runs all checks
  (Sections 2.1–2.3 and Section 4).

- **Scoped (file):** `/adr-check path/to/NNNN-adr-file.md`
  Argument is a `.md` file path. Validates only that file. Runs Sections 2.1 and 2.1e; skips
  2.2, 2.3, and 4.

- **Scoped (directory):** `/adr-check path/to/adrs/`
  Argument is a directory path. Validates all ADR files in that directory. Runs Sections 2.1,
  2.1e, 2.2, and 2.3 for that directory only; skips Section 4.

- **Scoped (query):** `/adr-check "natural language description"`
  Argument is a quoted or unquoted natural-language string that is not a resolvable path. The
  model identifies matching ADRs and runs Sections 2.1 and 2.1e against them; skips 2.2, 2.3,
  and 4.

- **Lifecycle skill invocation:** Lifecycle skills (`adr-create`, `adr-supersede`,
  `adr-deprecate`) invoke `Scoped (file)` mode against the file they just wrote. This validates
  only the newly written ADR without noise from other files or diff patterns.

- **Gate invocation:** The gate template calls the skill in Global mode. If the skill is not
  installed, the gate falls back to basic directory scanning (see FR-011 in the PRD).
