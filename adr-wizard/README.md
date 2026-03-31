# adr-wizard

> ADR lifecycle management for Claude Code.

<!-- mdformat-toc start --slug=github --maxlevel=3 --minlevel=2 -->

- [Overview](#overview)
- [Installation](#installation)
- [Skills](#skills)
  - [adr-create](#adr-create)
  - [adr-supersede](#adr-supersede)
  - [adr-deprecate](#adr-deprecate)
  - [adr-check](#adr-check)
- [ADR Location Convention](#adr-location-convention)
- [Compatibility](#compatibility)

<!-- mdformat-toc end -->

## Overview<a name="overview"></a>

`adr-wizard` is a Claude Code plugin that provides focused skills for every stage of an
Architecture Decision Record (ADR) lifecycle: **create**, **supersede**, **deprecate**, and
**validate**. It supports multiple ADR directories in a single repository via a CLAUDE.md
convention, and integrates gracefully with the `hyperloop` gate when both plugins are installed.

## Installation<a name="installation"></a>

Make sure you have the [`hyper-plugs`](/README.md) marketplace installed, then install `adr-wizard`:

```bash
claude plugin install adr-wizard@hyper-plugs
```

## Skills<a name="skills"></a>

| Skill           | Command             | Description                                                                                                     |
| --------------- | ------------------- | --------------------------------------------------------------------------------------------------------------- |
| `adr-create`    | `/adr-create`       | Create a new ADR with auto-numbering and index updates                                                          |
| `adr-supersede` | `/adr-supersede`    | Supersede an existing ADR, with bidirectional cross-references                                                  |
| `adr-deprecate` | `/adr-deprecate`    | Deprecate an ADR with a recorded reason                                                                         |
| `adr-check`     | `/adr-check [file]` | Validate ADRs for structural integrity, index sync, and advisory style checks; supports scoped single-file mode |

### adr-create<a name="adr-create"></a>

Creates a new Nygard-style ADR file with auto-numbering, drafts all sections from conversation
context, and updates the directory index. If no ADR directory exists, offers to bootstrap one.

```
/adr-create <decision summary>

# Examples:
/adr-create Use PostgreSQL for primary storage
/adr-create Adopt hexagonal architecture for the payments service

# Or: just discuss an architectural decision — the model will offer to
# create an ADR automatically based on what you've described.
```

The skill first evaluates whether the decision warrants an ADR (four qualifying signals: cross-cutting
concern, non-obvious to a new contributor, costly to reverse, or multiple alternatives considered).
If no signal applies it will ask for confirmation before proceeding. It then discovers your ADR
directory from CLAUDE.md, finds the next available number, and writes `NNNN-<slug>.md` with
Context, Decision, and Consequences drafted from your conversation. It interviews you via
`AskUserQuestion` only for details it cannot infer. After writing the file it runs `/adr-check`
in scoped mode to validate the new ADR and surfaces any issues before confirming completion.

### adr-supersede<a name="adr-supersede"></a>

Supersedes an existing ADR with a new one. Reads the old ADR and conversation context to draft
all sections of the replacement, then links both ADRs with bidirectional cross-references.
Delegates new ADR creation to `adr-create`.

```
/adr-supersede <old>[->new] <new decision or reason>

# Examples:
/adr-supersede 3 Switching from PostgreSQL to CockroachDB for horizontal scaling
/adr-supersede 3->7 CockroachDB was chosen — link existing ADR-0007 as the replacement
/adr-supersede 3->7   # ADR-0007 already exists, just link the two
```

### adr-deprecate<a name="adr-deprecate"></a>

Marks an existing ADR as deprecated. Infers the reason from conversation context, writes a
concise `**Deprecated:**` metadata line, and appends a full `## Deprecation Note` section so
future readers understand what changed and why.

```
/adr-deprecate <ADR number> <reason>

# Examples:
/adr-deprecate 5 No longer applicable after migration to microservices
/adr-deprecate 2   # skill will infer reason from conversation context
```

### adr-check<a name="adr-check"></a>

Validates all ADR directories for structural integrity, index sync, and cross-reference
consistency. Also scans `git diff` for patterns that may indicate undocumented architectural
decisions (advisory warnings only). Emits advisory style warnings when the `## Consequences`
section of an ADR contains no clearly adverse consequence or trade-off.

```
# Whole-directory mode — validates all ADR directories
/adr-check

# Scoped mode — validates a single file only (structural + style checks; skips index/cross-ref/diff)
/adr-check path/to/NNNN-adr-file.md
```

Exit result: **PASS** (all checks clear) or **FAIL** (with a remediation report listing exactly
what needs fixing and in which files). Style warnings are advisory only and do not affect the
pass/fail result.

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
