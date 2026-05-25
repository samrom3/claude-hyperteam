# Changelog

All notable changes to the hyperloop plugin will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [3.1.1] - 2026-05-25

### Changed

- `team-state.json` no longer holds `in_progress` status or `native_task_id` field. Live
  execution state (`pending` → `in_progress`) is owned exclusively by the native task list
  (via `TaskUpdate`); JSON is updated only on terminal transitions (`completed`, `failed`,
  `blocked`, `validated`). Eliminates the JSON/native-task sync hazard on mid-task crashes.
- Resume flow (Phase 1) now stashes uncommitted/unstaged changes before reconciling state.
  Orphaned native tasks from the prior session are cancelled via `TaskStop` before re-seeding,
  preventing duplicate task claims.
- `split ownership` table added to `team-state-schema.md` documenting which system owns each
  state category and why.

### Added

- ADR-004: documents the revert-uncommitted-on-resume decision and its rationale.
- `session-spec` skill detects `CHANGELOG.md` + version manifest at plan time and auto-appends
  a version bump + CHANGELOG step (blocked by all preceding FEAT steps, skill: `hyperwork-tech-writing`).
  Prevents version bump omissions at the source — the spec — rather than relying on GATE enforcement.

## [3.1.0] - 2026-05-08

### Fixed

- Confirmed all three required `TaskUpdate` call sites present in `hyperteam-worker.md`:
  `in_progress` on claim, `completed` on finish, `pending` rollback on commit failure or
  unresolvable blocker. "Always update BOTH native task and team-state.json" invariant present
  in Rules. (Issue #28 item 1)
- Restored `## Open Questions` resolution gate to `session-spec` Step 6 (Final Conflict Sweep)
  and Before Saving checklist. Gate was present in the original `prd` skill and was lost during
  the v2.0.0 rename. (Issue #28 item 2)
- Removed stale `prd` references from `hyperteam/SKILL.md` frontmatter description and
  `team-state-schema.md`; updated "On scaffold missing" handler in `hyperteam-lead.md` from
  "builder" to "worker" to match v3.0.0 self-claim terminology. (Issue #28 item 3)

### Added

- Pre-generation idea-refinement checkpoint (Step 4) in `session-spec` skill. After the
  interview, skill now summarizes gathered goals/scope/resolved conflicts and asks the user to
  Proceed or Refine. Capped at 2 refinement iterations — proceeds regardless on the third pass.
  Added corresponding item to Before Saving checklist. (Issue #22)
- Two-condition gate for `hyperwork-api-scaffold` as a standalone step in session-spec skill
  assignment. Standalone step assigned only when BOTH: (a) no existing structure — target
  modules/files absent and shape indeterminate from existing code; (b) parallelism unlock —
  scaffolded stubs allow ≥2 workers to proceed independently. Gate not met → structure
  definition bundled into the first FEAT task that requires it. (Issue #26)

## [3.0.0] - 2026-05-08

### Added

- Five loadable worker skills at `skills/hyperwork-*/`: `hyperwork-tdd`, `hyperwork-python`,
  `hyperwork-typescript`, `hyperwork-api-scaffold`, `hyperwork-tech-writing`. Workers load
  assigned skills at task-claim time via the `Skill` tool.
- `skills: none` sentinel in session-spec annotations — explicit signal that a step requires
  no skill loading (config, env setup, non-code steps).

### Changed

- `hyperteam-worker` promoted to primary executor — claims all FEAT and DOC tasks by `type`
  field; loads skills from `skills:` front-matter before work.
- `hyperteam-lead` extended with consult-arbiter protocol — receives worker blockers/questions
  via `SendMessage`, decides to unblock or escalate upstream. Workers never contact user directly.
- `hyperteam-reviewer` updated to emit structured PASS/FAIL output — `result`, `task_id`,
  `findings` (FEAT review) and per-check status (GATE); no prose narratives.
- `/hyperteam` Phase 2 team composition — N workers inferred from parallel-eligible task count
  at kickoff (clamped 1–4).
- `session-spec` skill assigns `skills:` array per step using five match rules; task routing
  driven by `type` field — `role_hint` field removed entirely.

### Removed

- `hyperteam-py-builder` agent
- `hyperteam-py-api-scaffolder` agent
- `hyperteam-techwriter` agent

## [2.0.0] - 2026-05-07

### Added

- `session-spec` skill — single-pass replacement for `prd`: one interview round, step→verify
  output format, adversarial conflict detection preserved.
- Gate discretion in `/hyperteam`: recommends skipping back-pressure gate for small specs
  (≤3 steps, no cross-step deps, human in verification loop). User can override either way.
- ADR-003 supersedes ADR-002 — updates contract writer from `prd` to `session-spec` skill and
  first section heading from `## 1. Introduction/Overview` to `## Goal`.

### Changed

- Output files renamed: `plans/<branch>-session-spec.md` (was `-prd.md`).
- `/hyperteam` warns on legacy `-prd.md` files — will not process silently.
- All internal cross-references updated: ADR-001, ADR-002 (superseded), README, skill tables,
  reference files.

### Removed

- `prd` skill (breaking). Use `/session-spec` going forward.

## [1.3.1] - 2026-05-07

### Changed

- Ultra caveman compression applied to `prd/SKILL.md` (201→~130 lines) and
  `hyperteam/SKILL.md` (203→~130 lines) — articles/conjunctions/hedging dropped; arrows for
  causality; "Seedling philosophy" blockquote condensed to 1–2 lines; Critical Review Mandate
  multi-sentence rationale → 1–2 line imperatives; Phase notes → one-line callouts; all
  phase/step structure, `AskUserQuestion` call specs, `git` command sequences, and conditional
  logic branches preserved intact
- Ultra caveman compression applied to all six reference files:
  `phase-1-fresh-start.md`, `phase-1-resume.md`, `phase-4-completion.md` (numbered steps
  preserved in order); `gate-task-template.md` (gate check sequence Check 1–5 and
  failure-escalation logic preserved); `team-state-schema.md` (JSON examples remain valid,
  field names/status values/timestamps unchanged, field description cells compressed);
  `example-prd.md` (`> **[Guidance]**` blockquotes compressed, PRD body fully preserved)

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
