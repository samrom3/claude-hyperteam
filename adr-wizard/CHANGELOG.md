# Changelog

All notable changes to the adr-wizard plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- `adr-create` skill: create numbered ADRs with Nygard template and auto-index updates
- `adr-supersede` skill: supersede an existing ADR with bidirectional cross-references
- `adr-deprecate` skill: deprecate an ADR with recorded reason
- `adr-check` skill: full ADR validation (structure, index sync, cross-references, diff-based warnings)
- ADR-check contract specification (`skills/adr-check/references/adr-check-contract.md`)
- Nygard ADR template (`skills/adr-create/references/adr-template.md`)
- `### ADR Locations` CLAUDE.md convention for multi-directory ADR discovery
