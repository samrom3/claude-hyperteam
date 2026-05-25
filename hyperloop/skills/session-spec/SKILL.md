---
name: session-spec
description: "Single-pass session-spec generator. Outputs plans/<branch>-session-spec.md ready for /hyperteam."
argument-hint: "<feature description or external sources (e.g., plans/auth-seedling.md, GitHub/JIRA issue, URL(s), etc.)>"
user-invocable: true
disable-model-invocation: true
---

# Session-Spec Generator

Produces `plans/<branch>-session-spec.md` in step→verify format. Single-pass: one interview round, one conflict check, one output.

______________________________________________________________________

## Adversarial Review Mandate

**Primary job: critique requirements, not agree with them.** Before accepting any requirement, actively search for:

1. **Explicit conflicts** — contradictory requirements, mutually exclusive acceptance criteria.
2. **Implicit conflicts vs. codebase** — contradictions with existing ADRs, `CLAUDE.md` patterns, domain model, `CONTRIBUTING.md`. Read `CLAUDE.md` for ADR locations, scan ADR dirs, search project source dirs before accepting any requirement.
3. **Ambiguity-hidden conflicts** — vague requirements that seem compatible but force contradictory impl choices.

Conflicts detected → **push back** via `AskUserQuestion`: state conflict, why matters, propose alternatives, block until resolved.

> **Seedling philosophy:** Seedling doc gives head start — not sacred. Challenge it same as any input.

______________________________________________________________________

## The Job

### Step 1 — Environment Setup

1. Read `$ARGUMENTS`. Empty → `AskUserQuestion` to gather input before proceeding.
2. **Detect input type:**
   - `$ARGUMENTS` is path to existing `.md` file → **seedling mode** (use as baseline draft).
   - Otherwise → **text description mode** (generate from scratch).
3. Derive `<slug>` as short lowercase-kebab-case label from title or description.
4. Generate `<branch>` as `feat-<slug>`.
5. **Detect GitHub issue references:**
   - Scan `$ARGUMENTS` for URLs matching `https://github.com/{owner}/{repo}/issues/{N}`.
   - Matches → `<source_issues>`: list of `owner/repo#N`. No match → `<source_issues>` null.
   - Per issue in `<source_issues>`:
     ```
     gh issue edit <N> --repo <owner>/<repo> --add-assignee @me
     ```
     Fails → print `⚠ Warning: could not assign issue — <error>`. Do **NOT** block spec creation.
6. **Sync main from origin:**
   1. Run `git fetch origin main`.
   2. `git log main..origin/main --oneline` — commits listed → main behind. `AskUserQuestion` to surface; **stop** until user confirms.
7. **Create and checkout branch:**
   ```
   git checkout -B <branch> main
   ```
   Verify `git branch --show-current` equals `<branch>`. Mismatch → `AskUserQuestion` and stop.
8. Create `plans/` if absent: `mkdir -p plans`
9. Create symlink:
   ```
   ln -sf ~/.claude/tasks/<branch> plans/<branch>
   ```
   Verify `test -L plans/<branch>` and `readlink` ends in `.claude/tasks/<branch>`. Fails → `AskUserQuestion` and stop.

### Step 2 — Conflict Scan

Read `CLAUDE.md` for ADR locations. Scan ADR dirs and relevant source dirs for existing code related to requested feature. Identify contradictions between requested and existing. **Mandatory in both modes.**

**Versioning detection (same pass):** Check for `CHANGELOG.md` + version manifest (`plugin.json`, `package.json`, `pyproject.toml`, `Cargo.toml`, etc.) in the repo. If both present → set `<has_versioning>: true`. Read `CLAUDE.md` for bump-type rules (major/minor/patch criteria). Determine `<bump_type>` from those rules + nature of changes being spec'd.

### Step 3 — Single Focused Interview

Max 2–3 `AskUserQuestion` calls total. Rules:

- At least one question must address conflicts found in Step 2 (if any).
- No conflicts found → ask about goals, scope boundaries, success criteria.
- Do not re-ask content user already provided.

**Seedling mode:** ask 2–3 targeted questions on gaps, conflicts, ambiguities — not repeating seedling content.

**Text description mode:** ask 2–3 questions on problem/goal, scope/boundaries, success criteria.

### Step 4 — Pre-Generation Checkpoint

1. Summarize in 1–3 sentences: goals gathered, scope boundaries, conflicts resolved in Steps 2–3.
2. `AskUserQuestion`: "Proceed to generate spec, or refine further?"
   - **Proceed** → continue to Step 5.
   - **Refine** → return to Step 3 for one additional targeted interview round (budget: 1–2 questions, separate from initial Step 3 budget). Cap: max 2 refinement iterations total — if Refine selected twice, proceed to Step 5 regardless on next pass.

### Step 5 — Generate Spec

1. Draft spec content in step→verify format per `references/example-session-spec.md` — do not write to disk yet; Step 6 sweep runs first.
3. Spec structure:
   - `<source_issues>` non-null → write metadata table **immediately after H1 and before `## Goal`**, one `| Source Issue |` row per issue.
   - `<source_issues>` null → omit table. H1 followed directly by `## Goal`.
   - Steps framing: each deliverable = `### STEP-<slug>-NN: <name>` with `**Acceptance Criteria:**` checklist.
   - **One step = one commit.** Scope each step so it can be implemented and committed independently (assuming prior steps already on branch). Steps that cannot be committed in isolation must be merged or re-scoped.
   - AC per step: `- [ ]` items — concrete, independently falsifiable checks. Include: artifact exists, behavior correct, project verification command passes. For new API surface, first step creates stubs with failing tests; subsequent steps implement against stable contracts.

4. **Skill assignment per step.** Every step must declare `skills`. Parsed by hyperteam Phase 1 → `skills` array in task YAML front-matter.

   Apply all rules that match.

   - Python code involved → add `hyperwork-python`
   - TypeScript code involved → add `hyperwork-typescript`
   - Step implements logic against existing contracts (not pure scaffolding) → add `hyperwork-tdd`
   - Step is docs, README, changelog, ADR, or user-facing writing → add `hyperwork-tech-writing`
   - `hyperwork-api-scaffold` as a **standalone** step only when BOTH hold:
     (a) **No existing structure** — target modules/files absent and shape indeterminate from existing code.
     (b) **Parallelism unlock** — scaffolded stubs allow ≥2 workers to proceed independently in next wave; if work remains serial after scaffolding, skip standalone step.
     **Gate not met** → bundle structure-definition into first FEAT task requiring it: add `hyperwork-api-scaffold` to that task's `skills:` array + note in description that worker creates structure inline before proceeding.

   No matches → `skills: none` (explicit sentinel; worker skips loading).

   Each step annotation (embedded in spec body, parsed by hyperteam Phase 1):
   ```
   > skills: hyperwork-tdd, hyperwork-python
   ```

   Example step with skill annotation:
   ```markdown
   ### STEP-auth-01: Implement JWT validation middleware

   > skills: hyperwork-tdd, hyperwork-python

   **Acceptance Criteria:**
   - [ ] `src/auth/middleware.py` exists with `validate_jwt(token: str) -> Claims` function
   - [ ] Unit tests in `tests/test_middleware.py` cover valid, expired, malformed token cases
   - [ ] `pre-commit run --all-files` exits 0
   ```

   The hyperteam skill (Phase 2, Step 3) reads these annotations and produces the native task YAML:
   ```yaml
   ---
   id: FEAT-auth-01
   type: FEAT
   skills:
     - hyperwork-tdd
     - hyperwork-python
   blocked_by: []
   ---
   ```

   > **Reading note for agents:** Metadata table (if present) appears **immediately after H1** and **before first `##` section**. Parsers: locate H1, scan forward collecting all `| Source Issue |` rows before `##`; none found → `source_issues` is `null`.

5. **Version bump step (conditional).** `<has_versioning>` true → append final step after all FEAT/DOC steps:

   ```markdown
   ### STEP-<slug>-NN: Bump version and update CHANGELOG

   > skills: hyperwork-tech-writing

   **Acceptance Criteria:**
   - [ ] Version manifest (`plugin.json` / `package.json` / etc.) bumped to next <bump_type> version per CLAUDE.md rules
   - [ ] `CHANGELOG.md` entry added under correct version header covering all changes in this spec
   - [ ] Project verification command exits 0
   ```

   `<bump_type>` from Step 2 detection. Step always last — blocked by all preceding FEAT steps.

### Step 6 — Final Conflict Sweep

Verify: no step contradicts another; no step conflicts with codebase findings from Step 2. Conflict found → raise with user via `AskUserQuestion` and resolve before saving.

**Open-questions gate:** Every item in `## Open Questions` must be: answered inline, explicitly deferred (note rationale in item), or removed. Any unresolved item remaining → raise via `AskUserQuestion` and resolve before saving.

All sweep checks pass → (1) if `plans/<branch>-session-spec.md` exists, move to `plans/archive/<branch>-session-spec.md`; (2) write drafted spec to disk.

> **Do NOT start implementing. Create spec only.**

______________________________________________________________________

## Before Saving

- [ ] Step 1 complete: `<branch>` chosen, main synced, branch checked out and verified, `plans/` exists, symlink created and validated
- [ ] GitHub issue URLs scanned: `<source_issues>` set to `["owner/repo#N", ...]` or `null`; `gh issue edit` assignment attempted per issue (warning on failure, spec not blocked)
- [ ] Input mode detected: seedling or text description
- [ ] `CLAUDE.md` ADR locations, ADR dirs, project source dirs searched for conflicts (Step 2)
- [ ] Interview complete: ≤3 `AskUserQuestion` calls; ≥1 addressed Step 2 conflicts (if any)
- [ ] Pre-generation checkpoint (Step 4) presented; user selected Proceed, or refinement cap (2 iterations) reached
- [ ] `<source_issues>` non-null → metadata table present immediately after H1 and before `## Goal`; null → no table
- [ ] Spec uses `## Goal / ## Context / ## Non-goals / ## Steps / ## Open Questions` structure
- [ ] Each step has `**Acceptance Criteria:**` checklist with ≥1 concrete, independently falsifiable `- [ ]` item
- [ ] Each step has `> skills:` annotation (`skills: none` if no rules match)
- [ ] Old `plans/<branch>-session-spec.md` archived to `plans/archive/` if existed
- [ ] Final conflict sweep complete — no intra-spec contradictions, no codebase conflicts
- [ ] All `## Open Questions` items answered, deferred with rationale, or removed
- [ ] `<has_versioning>` true → final step is version bump + CHANGELOG; `<bump_type>` matches CLAUDE.md rules; step blocked by all preceding FEAT steps
