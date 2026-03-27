---
name: adr-check
description: "Validate that all Architecture Decision Records (ADRs) are in sync with the codebase — checks ADR structure, status fields, index consistency, and cross-references. Surfaces git-diff-based warnings for undocumented architectural decisions. Invocable via /adr-check or by gate agents performing ADR validation."
user-invocable: true
---

# adr-check

Validates all ADR directories against the contract defined in
`references/adr-check-contract.md`. Outputs a structured pass/fail report per directory, plus
advisory diff-based warnings.

---

## Step 1 — Discover ADR directories

1. Read the project's `CLAUDE.md`.
2. Search for any heading containing `ADR Locations` (case-insensitive, any heading level).
3. If found, collect each bullet list item as a relative path, stripping inline `# comments`.
   Store as `adr_dirs`.
4. If not found, fall back: scan the repository root for `docs/adrs/`, `decisions/`, and
   `architecture/decisions/`. Use whichever exist.
5. If `adr_dirs` is empty after both methods, output:
   ```
   ADR Check Report
   ================
   WARNING: No ADR directories found. Skipping validation.

   Overall: PASS
   ```
   Stop. (No ADR directories is not a failure — nothing to validate.)

## Step 2 — Validate each directory

For each directory in `adr_dirs`, run all checks below. Track failures as a list of `issues`.

### Check 2.1 — ADR file structure

1. List all files in the directory matching `NNNN-*.md` (1+ digits, hyphen, any chars, `.md`).
   These are the ADR files.
2. For each ADR file, verify:
   a. **Status present and non-empty:** The file contains a line matching `**Status:**` (or
      `## Status`) with a non-empty value after the colon. If missing or empty, add issue:
      `[ADR-NNNN] Missing or empty Status field`
   b. **Context non-trivial:** The file contains a `## Context` section whose body is at least
      one non-blank line that is not a placeholder (does not consist solely of whitespace, `TODO`,
      `TBD`, or the guidance text from the template). If missing or trivial, add issue:
      `[ADR-NNNN] ## Context section is empty or contains only placeholder text`
   c. **Decision non-trivial:** Same check for `## Decision`. If failing, add issue:
      `[ADR-NNNN] ## Decision section is empty or contains only placeholder text`
   d. **Consequences present:** The file contains a `## Consequences` section. Content may be
      brief. If entirely absent, add issue:
      `[ADR-NNNN] ## Consequences section is missing`

### Check 2.2 — Index sync

1. Check if `<dir>/README.md` exists.
   - If it does not exist, add issue: `README.md index is missing from <dir>`
   - Skip index sync checks for this directory.
2. Parse the README.md table: extract all ADR filenames or numbers linked in the table.
3. For each ADR file in the directory, check it has a corresponding entry in the table. If not,
   add issue: `[ADR-NNNN] ADR file exists but has no entry in README.md index`
4. For each entry in the README.md table, check the referenced ADR file exists. If not, add
   issue: `[index entry] README.md references <filename> but file does not exist (orphaned entry)`

### Check 2.3 — Cross-reference integrity

1. For each ADR file with a status of `Superseded by ADR-NNNN`:
   a. Verify `NNNN-*.md` exists in the directory. If not, add issue:
      `[ADR-MMMM] Status says "Superseded by ADR-NNNN" but ADR-NNNN does not exist`
   b. Read the referenced ADR (NNNN). Verify it contains `Supersedes: ADR-MMMM` (where MMM is
      the superseded ADR number). If not, add issue:
      `[ADR-NNNN] Missing "Supersedes: ADR-MMMM" back-reference`
2. For each ADR file with a `Supersedes: ADR-NNNN` field:
   a. Verify `NNNN-*.md` exists. If not, add issue:
      `[ADR-MMMM] References "Supersedes: ADR-NNNN" but ADR-NNNN does not exist`
   b. Read ADR NNNN. Verify its status is `Superseded by ADR-MMMM`. If not, add issue:
      `[ADR-NNNN] Expected status "Superseded by ADR-MMMM" but found different status`

## Step 3 — Build the validation report

For each directory in `adr_dirs`:
- If `issues` is empty: status = PASS
- If `issues` is non-empty: status = FAIL

Output:

```
ADR Check Report
================

Directory: <path>
  Status: PASS | FAIL
  Issues:
    - <issue 1>
    - <issue 2>
    ...

Directory: <path>
  Status: PASS
  Issues: none

Overall: PASS | FAIL
```

If any directory has status FAIL, `Overall` is FAIL. Otherwise PASS.

On FAIL, append a `Remediation` section listing each issue with the specific file and the action
needed to fix it.

## Step 4 — Diff-based warnings (advisory only)

1. Run `git diff HEAD` to capture staged + unstaged changes.
2. Scan the diff output for these patterns:

   | Pattern | Warning |
   |---------|---------|
   | Lines starting with `+` that define a new abstract class or interface (Python `class Foo(ABC)`, `abstractmethod`, Java/TypeScript/Go `interface`, TypeScript `abstract class`) | "New interface/ABC detected in `<file>` — consider documenting the design decision with `/adr-create`" |
   | Lines starting with `+` in a new file whose name matches `*Config*`, `*Settings*`, `*Configuration*` (case-insensitive) | "New configuration file `<file>` — consider documenting configuration decisions with `/adr-create`" |
   | Lines starting with `+` in `package.json` (`dependencies`/`devDependencies`), `pyproject.toml` (`[tool.poetry.dependencies]` or `[project]`), `Cargo.toml` (`[dependencies]`), or `go.mod` (`require`) | "New dependency added in `<file>` — consider documenting the technology choice with `/adr-create`" |
   | New directory created at the root or as a top-level subdirectory named `service`, `module`, `component`, `pkg`, `lib`, `api`, or `gateway` | "New service/module boundary `<dir>` — consider documenting the architectural boundary with `/adr-create`" |

3. If any warnings found, append to the report:

   ```
   Diff-Based Warnings (advisory only — not gate-blocking)
   ========================================================
     - [WARNING] <message>
   ```

   If no warnings, omit this section entirely.

## Step 5 — Output and exit

Print the full report. The overall result determines the exit signal to callers:

- **PASS** (with or without warnings): success. Gate consumers should continue.
- **FAIL**: failure. Gate consumers should block and present the remediation steps to the user.
