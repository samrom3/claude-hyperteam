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

When the skill is invoked with a file path argument (e.g., `/adr-check docs/adrs/0001-foo.md`),
it enters **scoped mode**:

- Only the specified file is validated.
- Only structural checks (Section 2.1) and style checks (Section 2.1e) run against that file.
- Section 2.2 (Index Sync) and Section 2.3 (Cross-Reference Integrity) are skipped.
- Section 4 (Diff-Based Warnings) is skipped entirely.
- The report scope is limited to the single file rather than a directory.

Scoped mode is designed for use by lifecycle skills (`adr-create`, `adr-supersede`,
`adr-deprecate`) that need to validate only the file they just wrote, without noise from
unrelated ADRs or diff patterns.

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

Style Warnings (advisory only — not gate-blocking)
===================================================
  - [ADR-NNNN] <style warning message>

Overall: PASS | FAIL
```

On failure, each issue entry MUST include:
- Which ADR file is affected (by number and filename)
- What check failed
- A specific remediation step (e.g., "Add a non-empty ## Context section to docs/adrs/003-foo.md")

The `Style Warnings` section appears only when style checks (Section 2.1e) emit at least one
warning. It is omitted entirely when no style warnings are found. Style warnings are distinct from
`Diff-Based Warnings` (Section 4): style warnings reflect the quality of an individual ADR's
content; diff-based warnings reflect patterns in the current git diff.

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
| Style warnings present (whole-directory mode) | **PASS** (warnings are informational) |
| Scoped mode — structural check fails | **FAIL** |
| Scoped mode — style warning emitted | **PASS** (warning is informational) |
| Scoped mode — diff-based warnings | Skipped (not evaluated) |

Gate consumers MUST treat a **FAIL** result as a gate-blocking failure. Gate consumers MUST
treat diff-based warnings as informational only — they must be logged in the progress file and
presented to the user, but they do not block the gate.

---

## 6. Invocation

- **User-invocable (whole-directory mode):** `/adr-check`
  Validates all ADR directories discovered via Section 1.1. Runs all checks (Sections 2.1–2.3
  and Section 4).

- **User-invocable (scoped mode):** `/adr-check path/to/NNNN-adr-file.md`
  Validates only the specified ADR file. Runs structural checks (Section 2.1) and style checks
  (Section 2.1e) only. Skips index sync (Section 2.2), cross-reference checks (Section 2.3),
  and diff-based warnings (Section 4).

- **Model-invocable:** The skill description includes phrases matching ADR validation context
  so the model can auto-trigger it when relevant.

- **Lifecycle skill invocation (scoped):** Lifecycle skills (`adr-create`, `adr-supersede`,
  `adr-deprecate`) invoke the skill in scoped mode against the file they just wrote:
  `adr-check <path>`. This validates only the newly written ADR without noise from other files.

- **Gate invocation:** The gate template calls the skill by name in whole-directory mode. If the
  skill is not installed, the gate falls back to basic directory scanning (see FR-011 in the PRD).
