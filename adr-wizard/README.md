# adr-wizard

ADR lifecycle management for Claude Code. Create, supersede, deprecate, and validate
Architecture Decision Records across multiple directories.

## Overview

`adr-wizard` provides four focused skills for managing ADRs throughout their lifecycle:

- **`/adr-create`** — Create a new numbered ADR using the Nygard template and update the index.
- **`/adr-supersede`** — Replace an existing ADR with a new one, preserving bidirectional references.
- **`/adr-deprecate`** — Mark an ADR as deprecated with a recorded reason.
- **`/adr-check`** — Validate ADR structural integrity, index sync, cross-references, and surface diff-based warnings.

All skills are both **user-invocable** (via `/adr-*` slash commands) and **model-invocable** (the
model auto-triggers them when it detects architectural decision context).

## Installation

```bash
claude plugin install adr-wizard
```

## Skills

### `/adr-create`

Creates a new ADR in the appropriate directory. The skill:

1. Discovers ADR directories from `CLAUDE.md` or common patterns (see [ADR Location Convention](#adr-location-convention)).
1. If multiple directories exist, auto-suggests the most likely one based on recent file context and asks for confirmation.
1. Auto-numbers the ADR (highest existing + 1, zero-padded to 3 digits).
1. Fills the Nygard-style template (`references/adr-template.md`).
1. Updates the `README.md` index in the target directory.

### `/adr-supersede`

Supersedes an existing ADR with a new one. The skill:

1. Asks which existing ADR is being superseded.
1. Creates a new ADR with a "Supersedes: ADR-NNN" field.
1. Updates the superseded ADR's status to "Superseded by ADR-NNN".
1. Updates the index for both changes.

### `/adr-deprecate`

Marks an existing ADR as deprecated. The skill:

1. Asks which ADR to deprecate and the reason.
1. Updates the ADR's status to "Deprecated" and adds a "Deprecated: \<reason>" field.
1. Updates the directory index.

### `/adr-check`

Validates all ADR directories. The skill:

- Checks template structure, non-empty status, minimal content quality (Context/Decision non-empty).
- Verifies the `README.md` index is in sync with actual files.
- Validates "Supersedes"/"Superseded by" cross-references are bidirectional.
- Scans `git diff` for patterns suggesting undocumented architectural decisions (new interfaces,
  config files, dependency additions, new service boundaries) — surfaced as **warnings**, not failures.
- Outputs a per-directory pass/fail report with remediation steps on failure.

See [`skills/adr-check/references/adr-check-contract.md`](skills/adr-check/references/adr-check-contract.md)
for the full contract specification.

## ADR Location Convention

All skills discover ADR directories using the following priority order:

**1. `### ADR Locations` section in `CLAUDE.md` (recommended)**

Add a section to your project's `CLAUDE.md` listing each ADR directory as a bullet:

```markdown
### ADR Locations

- docs/adrs/            # project-wide decisions
- hyperloop/docs/adrs/  # hyperloop plugin
```

The skill searches for any heading containing "ADR Locations" at any nesting level. Inline
`# comment` annotations are ignored when parsing paths.

**2. Common-pattern fallback**

If no `### ADR Locations` section exists, skills scan for:

- `docs/adrs/`
- `decisions/`
- `architecture/decisions/`

## Compatibility

- **Standalone:** Works in any project with or without hyperloop.
- **With hyperloop:** The hyperloop gate's Check 2 (ADR sync) will automatically invoke `/adr-check`
  if adr-wizard is installed, using the full validation contract. Without adr-wizard, the gate
  falls back to basic directory scanning.
