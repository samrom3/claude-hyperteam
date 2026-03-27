# Changelog

All notable changes to the `adr-wizard` plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.1.0] - 2026-03-27

### Added

- Plugin scaffold: `.claude-plugin/plugin.json`, `agents/` directory, skill directory structure
- `adr-create` skill — creates new ADRs with auto-numbering, CLAUDE.md-based directory
  discovery, multi-directory context-aware selection, and README.md index updates
- `adr-supersede` skill — supersedes an existing ADR with a new one, maintaining bidirectional
  cross-references and updating both ADRs' Status fields and the directory index
- `adr-deprecate` skill — marks an ADR as deprecated with a recorded reason and updates the index
- `adr-check` skill — validates all ADR directories for structural integrity, index sync, and
  cross-reference correctness; surfaces diff-based advisory warnings for undocumented decisions
- `adr-check` contract specification (`skills/adr-check/references/adr-check-contract.md`)
- Nygard-style ADR template (`skills/adr-create/references/adr-template.md`)
- CLAUDE.md `### ADR Locations` convention for multi-directory ADR support
