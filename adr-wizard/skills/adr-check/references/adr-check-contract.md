# ADR-Check Contract

This document defines the generic contract for any skill that performs ADR validation. Any skill
implementing this contract (e.g., the `adr-check` skill) can be invoked by the hyperloop gate or
any other workflow to validate ADR health without hardcoding paths.

---

## Input

### ADR Directory Discovery

The implementing skill discovers ADR directories in the following priority order:

1. **CLAUDE.md convention (primary):** Read the project's `CLAUDE.md` and locate any heading
   containing the text "ADR Locations" (at any nesting level, e.g., `## ADR Locations`,
   `### ADR Locations`). Parse the bulleted list beneath it — each bullet is a relative path to an
   ADR directory. Inline `# comment` annotations are ignored when parsing.

   Example CLAUDE.md section:
   ```
   ### ADR Locations
   - docs/adrs/            # project-wide decisions
   - hyperloop/docs/adrs/  # hyperloop plugin
   ```

2. **Common-pattern fallback:** If no "ADR Locations" heading exists in CLAUDE.md, scan the
   repository for directories matching these patterns:
   - `docs/adrs/`
   - `decisions/`
   - `architecture/decisions/`

### Invocation Arguments

The skill accepts an optional argument to restrict validation to a single directory:

```
/adr-check [path/to/adrs/]
```

If no argument is provided, all discovered ADR directories are validated.

---

## Output

### Validation Report Format

The skill outputs a structured report in the following format for each directory:

```
## ADR Check Report

### <directory-path>

Status: PASS | FAIL

#### Failures (if any)
- <specific issue with file reference and remediation step>

#### Warnings (advisory, not gate-blocking)
- <advisory hint about potentially undocumented architectural decisions>
```

### Per-Directory Checks

For each discovered ADR directory, the skill validates:

1. **Template structure:** Every `.md` file (excluding `README.md`) must contain these sections:
   a heading matching the ADR title, a `**Status:**` field, a `**Date:**` field, and `## Context`,
   `## Decision`, and `## Consequences` sections.

2. **Non-empty status:** No ADR may have a blank or missing `**Status:**` value.

3. **Minimal content quality:** The `## Context` and `## Decision` sections must be non-empty
   (contain at least one non-whitespace line beyond the heading).

4. **Index sync:** The directory's `README.md` index table must list every ADR file present and
   must not list files that do not exist (orphaned entries).

5. **Cross-reference integrity:** ADRs with "Superseded by ADR-NNN" status must have a
   corresponding new ADR that contains "Supersedes: ADR-NNN". The reverse must also hold.

### Diff-Based Warnings

The skill scans `git diff HEAD` (staged + unstaged) for patterns that commonly indicate
undocumented architectural decisions. These are surfaced as **warnings** — advisory hints only,
not gate failures:

- New abstract classes or interfaces (`abstract class`, `ABC`, `interface `, `Protocol`)
- New configuration or settings files (filenames matching `*config*`, `*settings*`, `*Config*`,
  `*Settings*`)
- Additions to dependency manifests (`package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`)
- New top-level directories (suggesting new service or module boundaries)

---

## Failure Semantics

### PASS

A directory receives a `PASS` result when:

- All ADR files have a valid, non-empty `**Status:**` field.
- All ADR files contain non-empty `## Context` and `## Decision` sections.
- The `README.md` index is in sync with the actual files (no missing entries, no orphaned entries).
- All "Superseded by" / "Supersedes" cross-references resolve correctly.

### FAIL

A directory receives a `FAIL` result when any of the above checks does not pass. The report lists
each failure with the specific file and a remediation step.

### Overall result

The overall skill result is:

- **PASS** — all discovered directories pass all checks (warnings are allowed).
- **FAIL** — one or more directories fail one or more checks.

### Gate consumer contract

Gate agents invoking this skill (or any skill implementing this contract) must:

1. Invoke the skill with no arguments to check all directories, or with a specific path.
2. Parse the `Status: PASS | FAIL` line per directory to determine pass/fail.
3. Treat `#### Warnings` items as advisory — log them in the progress file but do not fail the gate.
4. On `FAIL`, surface the specific failure details to the user as gate failure reasons.

---

## Graceful Degradation

If no skill implementing this contract is installed, gate agents must fall back to:

1. Scanning for ADR directories via the `### ADR Locations` convention in CLAUDE.md.
2. If no convention section exists, scanning for `docs/adrs/` directories.
3. Performing a basic status-field spot-check manually (no full validation).

This ensures hyperloop gates work in projects that do not have adr-wizard installed.
