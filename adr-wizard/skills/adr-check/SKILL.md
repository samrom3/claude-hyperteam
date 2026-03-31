---
name: adr-check
description: "This skill should be used when the user asks to 'validate ADRs', 'check ADR structure', 'run adr-check', 'check for undocumented decisions', or when lifecycle skills and gate agents need to validate ADR files. Supports Global mode (all discovered ADR directories, including diff-based warnings), Scoped mode (single file, directory, or natural-language query — no diff noise), and future Diff mode."
user-invocable: true
argument-hint: "[optional: path/to/NNNN-adr-file.md | path/to/adrs/ | natural language query]"
---

# adr-check

Validates ADRs against the contract defined in `references/adr-check-contract.md`. Supports
three modes:

* **Global** (all discovered ADR directories, including diff-based warnings),
* **Scoped** (a single file, a directory, or a natural-language query — no diff-based noise), and
* **Diff** (ADRs changed in the current branch).

Outputs a structured pass/fail report with a consistent header, regardless of mode.

---

## Step 1 — Determine mode and discover inputs

1. Inspect the skill argument (the text following `/adr-check`, if any):

   - **No argument** → `mode = global`. Continue with directory discovery (steps 2–6).

   - **Argument resolves to a `.md` file** → `mode = scoped_file`, `target = <path>`.
     Skip directory discovery. Proceed to Step 2 with only that file as target.
     Checks 2.2, 2.3, and Step 4 are skipped entirely.

   - **Argument resolves to a directory** → `mode = scoped_dir`, `target = <directory>`.
     Skip CLAUDE.md discovery. Use `target` as the sole entry in `adr_dirs`.
     Run Checks 2.1, 2.1e, 2.2, and 2.3 for that directory. Skip Step 4.

   - **Any other non-empty argument** → `mode = scoped_query`, `query = <argument>`.
     Run directory discovery (steps 2–6) to locate all ADR files. Then use model judgment
     to identify which ADR files match the natural-language query. Use those files as the
     validation target. Run only Checks 2.1 and 2.1e against each matched file. Skip 2.2,
     2.3, and Step 4. If no ADRs match the query, emit an advisory warning and exit PASS.

2. *(global and scoped_query only)* Read the project's `CLAUDE.md`.
3. *(global and scoped_query only)* Search for any heading containing `ADR Locations`
   (case-insensitive, any heading level).
4. *(global and scoped_query only)* If found, collect each bullet list item as a relative path,
   stripping inline `# comments`. Store as `adr_dirs`.
5. *(global and scoped_query only)* If not found, fall back: scan the repository root for
   `docs/adrs/`, `decisions/`, and `architecture/decisions/`. Use whichever exist.
6. *(global only)* If `adr_dirs` is empty after both methods, output a report with
   `Mode: Global` and emit: `WARNING: No ADR directories found. Skipping validation.`
   Set `Overall: PASS` and stop.

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
   e. **Consequences quality (style check):** Using model judgment — not keyword matching —
      assess whether the `## Consequences` section contains at least one genuinely adverse
      outcome. Qualifying examples: a known risk, a trade-off, a migration cost, an increase in
      complexity, a performance regression, reduced flexibility, or an explicit statement of what
      is being given up. If no such adverse consequence is found, add a **style warning** (not a
      structural issue):
      `[ADR-NNNN] Consequences section contains no clearly adverse consequence or trade-off — consider adding one for credibility`
      Style warnings do not affect the pass/fail result and are reported separately (see Step 3).

### Check 2.2 — Index sync *(whole-directory mode only)*

1. Check if `<dir>/README.md` exists.
   - If it does not exist, add issue: `README.md index is missing from <dir>`
   - Skip index sync checks for this directory.
2. Parse the README.md table: extract all ADR filenames or numbers linked in the table.
3. For each ADR file in the directory, check it has a corresponding entry in the table. If not,
   add issue: `[ADR-NNNN] ADR file exists but has no entry in README.md index`
4. For each entry in the README.md table, check the referenced ADR file exists. If not, add
   issue: `[index entry] README.md references <filename> but file does not exist (orphaned entry)`

### Check 2.3 — Cross-reference integrity *(whole-directory mode only)*

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

Separate all items collected from Check 2.1e into a `style_warnings` list. These are NOT counted
as structural issues and do NOT affect status.

For each validated target (directory or file):
- If structural issues (2.1a–2.1d) are present: status = FAIL
- Otherwise: status = PASS

Output the report using the structure below. The header rows (`Mode`, `Target`, `ADRs`) are
**always present** regardless of mode — they provide context for all rows that follow.

```
ADR Check Report
================

Mode:    Global | Scoped (file) | Scoped (directory) | Scoped (query)
Target:  <path, directory, or natural-language query>
ADRs:    <comma-separated list of all validated ADR filenames>

Results
-------
<path/to/directory-or-file>:
  Status: PASS | FAIL
  Issues:
    - [ADR-NNNN] <issue description>

<path/to/next-item>:
  Status: PASS
  Issues: none

Style Warnings (advisory only — not gate-blocking)
===================================================
  - [ADR-NNNN] <style warning message>

Overall: PASS | FAIL
```

**Results grouping:** In `global` and `scoped_dir` modes, group results by directory path. In
`scoped_file` and `scoped_query` modes, list each validated file as its own result entry.

The `Style Warnings` section appears only when `style_warnings` is non-empty; omit it otherwise.

If any result entry has status FAIL, `Overall` is FAIL. Otherwise PASS.

On FAIL, append a `Remediation` section listing each issue with the specific file and the action
needed to fix it.

## Step 4 — Diff-based warnings *(Global mode only — skip in all scoped sub-modes)*

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

- **PASS** (with or without diff-based warnings or style warnings): success. Gate consumers
  should continue.
- **FAIL**: failure. Gate consumers should block and present the remediation steps to the user.

## Additional Resources

- **`references/adr-check-contract.md`** — Full contract specification: discovery rules, validation checks (2.1–2.3, 2.1e), report format, pass/fail semantics, and invocation modes.
