# Changelog

All notable changes to the `adr-wizard` plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.1.0] - 2026-03-28

### Added

- `adr-check` scoped mode — invoke with a file path argument
  (`/adr-check path/to/NNNN-adr-file.md`) to validate only that file; runs structural and style
  checks only, skipping index sync, cross-reference checks, and diff-based warnings
- `adr-check` style check (Check 2.1e) — uses model judgment to assess whether the
  `## Consequences` section contains at least one genuinely adverse consequence or trade-off;
  emits an advisory style warning (not a structural failure) when none is found
- `adr-check` Style Warnings report section — distinct from Diff-Based Warnings; appears only
  when style checks fire; does not affect the pass/fail result
- `adr-check` `argument-hint` front-matter field for optional scoped file path
- `adr-check` contract updated with scoped mode interface, Section 1.2, style check subsection
  2.1e, Style Warnings report format, and scoped mode pass/fail semantics rows
- `adr-create` Step 0 — decision worthiness evaluation with qualifying signals, detection
  heuristics, explicit skip cases, and `AskUserQuestion` confirmation before proceeding
- `adr-create` per-section writing quality criteria in Step 5 — Context (problem not solution,
  no advocacy), Decision (active voice, declarative, avoid passive constructions), Consequences
  (must include at least one adverse consequence; 200–400 word target)
- `adr-create` PR co-location reminder in Step 8 — advises committing the ADR in the same PR
  as the code it describes for traceability
- `adr-create` Step 9 — post-write validation; invokes `adr-check` in scoped mode against the
  newly created file; blocks on structural FAIL, surfaces style warnings without blocking
- `adr-supersede` and `adr-deprecate` additive-only principle notes near the top of each skill
- `adr-supersede` Step 10 and `adr-deprecate` Step 8 — post-write validation; invokes
  `adr-check` in scoped mode against the modified ADR; blocks on structural FAIL, surfaces style
  warnings without blocking

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
