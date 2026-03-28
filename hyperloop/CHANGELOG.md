# Changelog

All notable changes to the hyperloop plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- `/prd` accepts one or more GitHub issue URLs anywhere in its arguments. When detected, hyperloop:
  - Assigns each issue to the authenticated `gh` user immediately (`gh issue edit --add-assignee @me`).
    On failure, a visible warning is printed but PRD creation is not blocked.
  - Writes one `| Source Issue | owner/repo#N |` metadata table row per issue immediately after
    the PRD's H1 heading and before `## 1.`, so all issue references travel with the PRD.
  - Stores `metadata.source_issues` (array) in `team-state.json` during Phase 1 (parsed from the
    PRD metadata table); `null` when no issues were provided. This field is immutable after first write.
  - Appends a `Closes #N` (same-repo) or `Closes <URL>` (cross-repo) line per issue to the PR
    body in Phase 4, so all linked issues auto-close on merge.
- ADR-002: Two-location `source_issues` storage — documents the decision to store issue references
  in both the PRD metadata table (for human visibility) and `team-state.json` (as the authoritative
  runtime value), along with the canonical metadata table format spec and the array-type rationale.

### Changed

- `metadata.source_issue` (singular string) renamed to `metadata.source_issues` (string array) in
  `team-state.json`. A single-issue PRD produces a one-element array; `null` is preserved for
  no-issue PRDs.
- Phase 1 (`phase-1-fresh-start.md`) now collects all `| Source Issue |` rows from the PRD metadata
  table and populates `metadata.source_issues` in `team-state.json`. Assignment verification runs
  for each issue.
- Phase 4 (`phase-4-completion.md`) PR body now emits one `Closes` line per entry in
  `metadata.source_issues` when the array is non-null.
- `references/team-state-schema.md` documents `source_issues` as a formally specified optional
  field (`string[] | null`) in the `metadata` block.
- `references/example-prd.md` guidance updated to show both the table-present and table-absent
  PRD states.

## [1.2.0] - 2026-03-27

### Changed

- Gate Check 2 (ADR sync) now delegates to the `/adr-check` skill when adr-wizard is installed,
  falling back to manual directory scanning when it is not. The check no longer hardcodes a
  single `docs/adrs/` path — it reads `### ADR Locations` from `CLAUDE.md` or scans common
  fallback directories. Backwards-compatible: hyperteam works without adr-wizard installed.

## [1.1.0] - 2026-03-26

### Added

- PRD selection at `/hyperteam` startup — scans `plans/` for `*-prd.md` files, categorises them
  as unstarted or in-progress, and prompts the user to choose; supports creating several PRDs
  upfront and executing them in any order
- Concurrent session support — each `/hyperteam` session creates its own agent team via
  `TeamCreate`, which automatically scopes the native task list by team name; multiple sessions
  can run against different PRDs without interfering with each other.

### Changed

- Phase 0 of `/hyperteam` now scans `plans/` for PRDs instead of reading
  `CLAUDE_CODE_TASK_LIST_ID` from `.claude/settings.local.json`
- Phase 0 git branch step is now automatic: if the selected PRD's branch differs from the current
  branch, `/hyperteam` checks it out locally or creates it from `origin/main`
- `/prd` no longer writes `CLAUDE_CODE_TASK_LIST_ID` to `.claude/settings.local.json`; task list
  scoping is handled automatically by `TeamCreate` via team name.

## [1.0.1] - 2026-03-26

### Added

- This CHANGELOG

### Changed

- Flattened `agents/packs/python/` into `agents/` for automatic discovery by Claude Code

### Fixed

- Agent discovery for Python pack agents (`hyperteam-py-builder`, `hyperteam-py-api-scaffolder`) —
  previously invisible due to subdirectory nesting (Claude Code only scans the top-level `agents/` directory)

## [1.0.0] - 2026-03-23

### Added

- PRD generator skill (`/hyperloop:prd`) — multi-phase structured interview with requirement
  analysis and conflict deconfliction before any code is written
- Autonomous agent team skill (`/hyperloop:hyperteam`) — converts a PRD into a dependency-ordered
  task DAG and runs a specialist agent team with back-pressure gates
- Back-pressure gate (`hyperteam-reviewer`) — dedicated GATE task type; the lead blocks new work
  until the reviewer clears all acceptance criteria
- Re-entrant execution — `team-state.json` enables mid-run resume after quota exhaustion or
  network interruptions
- Python language pack — `hyperteam-py-api-scaffolder` (scaffold-first interface definitions) and
  `hyperteam-py-builder` (TDD business logic implementation)
- Core agents: `hyperteam-lead`, `hyperteam-reviewer`, `hyperteam-techwriter`, `hyperteam-worker`
