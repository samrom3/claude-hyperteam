# Changelog

All notable changes to the `adr-wizard` plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-03-27

### Added

- Plugin scaffold: `.claude-plugin/plugin.json`, `agents/` directory, skill directory structure
- `adr-create` skill — creates new ADRs with auto-numbering, CLAUDE.md-based directory
  discovery, multi-directory context-aware selection, directory bootstrapping, and README.md
  index updates; accepts a decision summary argument and drafts all sections (Context, Decision,
  Consequences) from conversation context, interviewing via `AskUserQuestion` only when needed
- `adr-supersede` skill — supersedes an existing ADR with a new one via `<old>[->new]` argument
  syntax; delegates new ADR creation to `adr-create`; drafts all sections from the old ADR
  content and conversation context; maintains bidirectional cross-references
- `adr-deprecate` skill — marks an ADR as deprecated; infers the rationale from conversation
  context; writes a full `## Deprecation Note` section into the file; updates the index
- `adr-check` skill — validates all ADR directories for structural integrity, index sync, and
  cross-reference correctness; surfaces diff-based advisory warnings for undocumented decisions
- `adr-check` contract specification (`skills/adr-check/references/adr-check-contract.md`)
- Nygard-style ADR template (`skills/adr-create/references/0000-adr-template.md`) with 4-digit
  zero-padded numbering (`NNNN`)
- ADR directory README template (`skills/adr-create/references/README-template.md`)
- CLAUDE.md `### ADR Locations` convention for multi-directory ADR support
