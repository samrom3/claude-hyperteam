# adr-wizard

> ADR lifecycle management for Claude Code.

<!-- mdformat-toc start --slug=github --maxlevel=3 --minlevel=2 -->

- [Overview](#overview)
- [Installation](#installation)
- [Skills](#skills)
- [ADR Location Convention](#adr-location-convention)
- [Compatibility](#compatibility)

<!-- mdformat-toc end -->

## Overview<a name="overview"></a>

`adr-wizard` is a Claude Code plugin that provides focused skills for every stage of an
Architecture Decision Record (ADR) lifecycle: **create**, **supersede**, **deprecate**, and
**validate**. It supports multiple ADR directories in a single repository via a CLAUDE.md
convention, and integrates gracefully with the `hyperloop` gate when both plugins are installed.

## Installation<a name="installation"></a>

```bash
claude plugin install adr-wizard@hyper-plugs
```

## Skills<a name="skills"></a>

| Skill           | Command          | Description                                                    |
| --------------- | ---------------- | -------------------------------------------------------------- |
| `adr-create`    | `/adr-create`    | Create a new ADR with auto-numbering and index updates         |
| `adr-supersede` | `/adr-supersede` | Supersede an existing ADR, with bidirectional cross-references |
| `adr-deprecate` | `/adr-deprecate` | Deprecate an ADR with a recorded reason                        |
| `adr-check`     | `/adr-check`     | Validate all ADRs for structural integrity and index sync      |

## ADR Location Convention<a name="adr-location-convention"></a>

`adr-wizard` skills discover ADR directories by reading a `### ADR Locations` subsection from
the project's `CLAUDE.md`. Add this subsection wherever it makes sense in your `CLAUDE.md`:

```markdown
### ADR Locations
- docs/adrs/           # project-wide decisions
- hyperloop/docs/adrs/ # hyperloop plugin decisions
```

Each bullet is a relative path to an ADR directory. Inline `# comments` are ignored by the
skills.

If no `### ADR Locations` section is present, skills fall back to scanning for `docs/adrs/`,
`decisions/`, and `architecture/decisions/` at the repository root.

## Compatibility<a name="compatibility"></a>

`adr-wizard` works standalone — no other plugins required. When used alongside `hyperloop`,
the gate template automatically delegates ADR validation to `/adr-check` if the skill is
available, providing multi-directory ADR support in the gate. If `adr-wizard` is not installed,
`hyperloop` falls back to its built-in basic ADR scanning.
