---
name: prd
description: "Multi-phase PRD generator. Outputs plans/<branch>-prd.md ready for /hyperteam."
argument-hint: "<feature description or external sources (e.g., plans/auth-seedling.md, GitHub/JIRA issue, URL(s), etc.)>"
user-invocable: true
disable-model-invocation: true
---

# PRD Generator

Create detailed Product Requirements Documents that are clear, actionable, and suitable for implementation by junior
developers or AI agents.

______________________________________________________________________

## Critical Review Mandate

> **Your primary job is to _critique_ the requirements you receive — not simply agree with them.** The user is paying
> for your judgement, not your compliance. An agent that rubber-stamps every input is worse than useless: it lets
> conflicting requirements slip through and causes implementing agents to get stuck with irreconcilable constraints.

Before accepting any requirement at face value, actively search for and surface:

1. **Explicit conflicts within the input** — contradictory requirements, goals that work against each other, acceptance
   criteria that are mutually exclusive.
2. **Implicit conflicts against the existing codebase** — requirements that contradict existing ADRs, break established
   patterns in `CLAUDE.md`, conflict with the current domain model in the project source tree, or violate conventions in
   `CONTRIBUTING.md`. Read `CLAUDE.md`, scan `docs/adrs/`, and search the project source directories (as identified in
   `CLAUDE.md`) before accepting any requirement.
   _(Full ADR scan is intentional here: PRD work is design work and warrants full architectural context.)_
3. **Ambiguities that hide conflicts** — vague requirements that seem compatible but would force contradictory
   implementation choices once an agent tries to write code.

If ambiguities are hiding conflicts, interview the user instead with clarifying questions via `AskUserQuestion` to
disambiguate and reveal the missing requirements and/or new conflicts.

When conflicts are detected, **push back** — use `AskUserQuestion` to clearly state the conflict, why it matters, and
propose concrete alternatives, blocking until the user resolves it.

**Do not proceed to the next phase until all ambiguities and conflicts are resolved.**

> **Seedling philosophy:** A seedling PRD or external document source such as a GitHub or JIRA issue is the operator's
> intent distilled into a draft document — it gives the skill a head start with more details. But a seedling is not sacred:
> challenge them just as you would any other input.

______________________________________________________________________

## The Job

### Phase 0: Environment Setup

1. Read `$ARGUMENTS` (the text typed after `/prd`). If empty, use `AskUserQuestion` to gather input.
2. **Detect input type:**
   - If `$ARGUMENTS` is a path to an existing `.md` file → **seedling mode** (use the file as a baseline draft).
   - Otherwise → **text description mode** (generate from scratch).
3. Derive `<slug>` as a short, lowercase-kebab-case label:
   - Seedling mode: derive from the seedling document's title.
   - Text mode: derive from the feature description (e.g., "Account Rollover" → `account-rollover`).
4. Generate `<branch>` as `feat-<slug>` (e.g., `feat-account-rollover`).
5. **Detect GitHub issue references:**
   - Scan `$ARGUMENTS` for **all** URLs matching the pattern `https://github.com/{owner}/{repo}/issues/{N}` (where
     `{N}` is a positive integer).
   - If one or more matches are found, collect all of them as `<source_issues>`, a list of `owner/repo#N` references
     (e.g., `["samrom3/claude-hyper-plugs#13"]`).
   - If no match is found, set `<source_issues>` to `null` — the metadata table will be omitted from the PRD.
   - For each issue in `<source_issues>`, immediately run:
     ```
     gh issue edit <N> --repo <owner>/<repo> --add-assignee @me
     ```
     If this command fails (e.g., unauthenticated `gh`, insufficient permissions), print a visible warning line
     (`⚠ Warning: could not assign issue — <error>`) but do **NOT** block PRD creation.
6. **Sync main from origin:**
   1. Run `git fetch origin main` to pull latest remote state.
   2. Run `git log main..origin/main --oneline` — if any commits are listed, main is behind; use `AskUserQuestion` to
      surface this to the user and **stop** until they confirm how to proceed (typically `git merge origin/main` or
      `git rebase origin/main`).
   3. Only proceed once `main` is up to date with `origin/main`.
7. **Create and checkout branch from main:**
   ```
   git checkout -B <branch> main
   ```
   (If the branch already exists, `-B` resets it to `main` — this is intentional when re-running `/prd` to avoid
   inheriting stale state.)
   After running, verify with `git branch --show-current` — the output must equal `<branch>`. If it doesn't, use
   `AskUserQuestion` to surface the error and stop.
8. Create the `plans/` directory if it does not exist:
   ```
   mkdir -p plans
   ```
9. Create a symlink so task files are accessible under `plans/<branch>`:
   ```
   ln -sf ~/.claude/tasks/<branch> plans/<branch>
   ```
   After running, verify the symlink:
   1. Confirm `plans/<branch>` exists and is a symlink: `test -L plans/<branch>`
   2. Confirm `readlink plans/<branch>` returns a path ending in `.claude/tasks/<branch>`.
   3. If either check fails, use `AskUserQuestion` to surface the error and stop — do not proceed to Phase 1.

### Phase 1: Draft PRD (baseline)

1. If `plans/<branch>-prd.md` already exists, move it to `plans/archive/<branch>-prd.md` before continuing.

2. **Before generating anything:** Read `CLAUDE.md`, scan `docs/adrs/`, and search the project source directories (as
   identified in `CLAUDE.md`) for existing code related to the feature. Identify any conflicts between what is being
   requested and what already exists. This research is mandatory in both modes. _(Full ADR scan is intentional: PRD work
   is design work — the author is actively making architectural decisions and full ADR context is worth the token cost.)_

3. **Branch on input mode:**

   **Seedling mode:**

   - Read the seedling file. Preserve the author's structure, intent, and any existing sections.
   - Review the seedling for internal conflicts and conflicts against the existing codebase (from step 2).
   - Use `AskUserQuestion` to ask 2–3 targeted clarifying questions focused on conflicts, gaps, and ambiguities (not
     repeating what the seedling already says).
   - Expand the seedling into a complete PRD, filling in all missing template sections.

   **Text description mode:**

   - Use `AskUserQuestion` to ask 3–5 clarifying questions (focus on: problem/goal, core functionality,
     scope/boundaries, success criteria). At least one question must probe potential conflicts with existing
     functionality uncovered in step 2.
   - Generate a complete PRD from scratch.

4. The generated PRD must follow the annotated example in `references/example-prd.md` and include **Design
   Considerations** and **Open Questions**.

5. Developer stories should follow the scaffold-first / implement-second pattern for any new or changed API surface:
   the first story creates typed stubs (interfaces, API contracts) with `NotImplementedError` bodies and skeleton tests;
   subsequent stories implement the business logic against those stable contracts via TDD.

6. Save to `plans/<branch>-prd.md`.
   - If `<source_issues>` is non-null and non-empty, write a metadata table **immediately after the H1 heading**
     and **before section `## 1.`**, with **one `| Source Issue |` row per issue**, using exactly this format:

     ```markdown
     # <Title>

     | Field        | Value                          |
     | ------------ | ------------------------------ |
     | Source Issue | owner/repo#N                   |
     | Source Issue | owner/repo#M                   |

     ## 1. Introduction/Overview
     ```

     For a single issue, the table has exactly one `Source Issue` row.

   - If `<source_issues>` is null or empty, omit the metadata table entirely — the H1 heading is followed directly
     by `## 1. Introduction/Overview` with no table in between.

   > **Reading note for agents and future readers:** The metadata table, if present, appears **immediately after
   > the H1 heading** and **before the first `##` section heading**. Parsers should locate the H1, then scan
   > forward collecting **all** `| Source Issue |` rows before encountering a `##` line; if none are found,
   > `source_issues` is `null`.

### Phase 2: Design refinement (questions + expand open questions)

1. Review the PRD's **Design Considerations** and use `AskUserQuestion` to ask a targeted series of design questions.
2. **Explicitly cross-check** each proposed design choice against existing ADRs and `CLAUDE.md` patterns. If a design
   choice contradicts an existing decision, surface this as a conflict requiring resolution (potentially via a new ADR
   that supersedes the old one).
3. Append newly discovered questions to the **bottom of the Open Questions section** (keep existing; add an "Added in
   Phase 2" subsection).
4. Update `plans/<branch>-prd.md` with the refined design considerations and updated open questions.

### Phase 3: Final refinement (answer all open questions + final pass)

1. Use `AskUserQuestion` to ask the user **all remaining Open Questions**.
2. Refine the PRD one final time based on the answers.
3. **Final conflict sweep:** Before saving, verify that no requirement in the PRD contradicts another, and that no
   requirement conflicts with the existing codebase as understood from the Phase 1 research. If any conflict is found,
   raise it with the user and resolve before saving.
4. Save the final version to `plans/<branch>-prd.md`.

> **Important:** Do NOT start implementing. Just create the PRD.

______________________________________________________________________

## Before Saving

- [ ] Phase 0 completed: `<branch>` chosen (`feat-<slug>`), main synced from origin, branch checked out and verified,
  `plans/` directory exists, symlink `plans/<branch>` → `~/.claude/tasks/<branch>` created and validated
- [ ] GitHub issue URLs scanned from `$ARGUMENTS`: `<source_issues>` set to `["owner/repo#N", ...]` if found, `null` otherwise;
  for each issue, `gh issue edit` assignment attempted (warning printed on failure, PRD creation not blocked)
- [ ] Input mode detected: seedling (file path) or text description
- [ ] `CLAUDE.md`, `docs/adrs/`, and project source directories searched for conflicts before generating
- [ ] Phase 1 PRD includes all 9 sections (see `references/example-prd.md`), including Design Considerations and Open
  Questions
- [ ] If `<source_issues>` is non-null: metadata table present immediately after H1 heading and before `## 1.`, one row per issue; if null:
  no metadata table in PRD
- [ ] Seedling mode: author's structure and intent preserved; only gaps/ambiguities questioned
- [ ] User input gathered in each phase as needed
- [ ] Incorporated user's answers into the PRD after each refinement phase
- [ ] Phase 2 cross-checked all design choices against existing ADRs and `CLAUDE.md` patterns
- [ ] Phase 3 final conflict sweep completed — no intra-PRD contradictions, no codebase conflicts
- [ ] Developer stories are small, specific, and follow the scaffold-first pattern
- [ ] Functional requirements are numbered (`FR-###`) and unambiguous
- [ ] Any old `plans/<branch>-prd.md` is archived to `plans/archive/`
- [ ] Non-goals section clarifies Goal section boundaries
- [ ] Any newly discovered questions were appended to the bottom of **Open Questions** (with a phase marker)
