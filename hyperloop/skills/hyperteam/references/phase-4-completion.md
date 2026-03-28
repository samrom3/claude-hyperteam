# Phase 4 — Completion and PR Offer

This phase runs after the team lead returns and the GATE task has passed.

______________________________________________________________________

## Step 1 — Compute completion summary

Read `plans/<branch>-progress.txt` and `plans/<branch>-team-state.json`. Report:

- Total tasks completed, broken down by type: FEAT, DOC, GATE.
- Number of FEAT tasks validated by the validator agent.
- Number of gate iterations (from `gate_iterations` in `team-state.json`).
- Time elapsed: timestamp of the first `progress.txt` entry to the last.

______________________________________________________________________

## Step 2 — Mark run complete

Update `plans/<branch>-team-state.json`: set `metadata.status` to `"complete"`.

Append a final entry to `plans/<branch>-progress.txt`:

```
## [ISO timestamp] - COMPLETE
- All tasks validated and gate passed.
- Total tasks: N (FEAT: X, DOC: Y, GATE: Z)
- Gate iterations: N
- Time elapsed: [computed duration]
---
```

______________________________________________________________________

## Step 3 — Offer PR creation

Use `AskUserQuestion` to ask:

> GATE passed. All tasks complete. Create a PR for branch `<branch>`?
> (Title will be derived from the PRD title. Answering 'no' leaves the branch open for a manual PR.)

______________________________________________________________________

## Step 4a — On confirmation: create the PR

Run `gh pr create`:

- `--title`: The first H1 heading in `plans/<branch>-prd.md` after the frontmatter, with any
  backtick-wrapped skill name stripped (use the plain prose title).
- `--body`: A summary including:
  1. **Goals** section from the PRD (verbatim or abbreviated).
  2. Linked stories: list of `FEAT-*` and `DOC-*` task IDs with their titles.
  3. **Source issue close links** (if `metadata.source_issues` in `team-state.json` is non-null and non-empty):
     - Run `gh repo view --json nameWithOwner --jq '.nameWithOwner'` to get the current repo in
       `owner/repo` format.
     - For **each** entry in `source_issues`, emit one `Closes` line:
       - If `owner/repo` **matches** the current repo: `Closes #N`
       - If `owner/repo` **does not match** (cross-repo): `Closes https://github.com/owner/repo/issues/N`
     - Place all `Closes` lines **after** the linked stories section and **before** the `---` footer
       separator, each on its own line, the block surrounded by blank lines.
     - If `source_issues` is `null` or empty, omit this section entirely — no regression.
  4. Standard footer:
     ```
     ---
     🤖 Generated with [Claude Code](https://claude.com/claude-code)
     ```
- `--base main` (or the repo's default branch).

______________________________________________________________________

## Step 4b — On decline

Print: "Branch `<branch>` is ready. Run `gh pr create` manually when you are ready."

Exit cleanly.
