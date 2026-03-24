# Back-Pressure Gate Template

This is a reusable template for the gate check performed at the end of a hyperteam run. The gate
agent reads `plans/<branch>-team-state.json` and `plans/<branch>-progress.txt` to determine whether
all tasks have passed validation. Substitute `<slug>` and `<branch>` with the actual values before
use.

______________________________________________________________________

## Gate Agent Instructions

```
You are the back-pressure gate agent. Perform all five checks below IN ORDER.

## Check 1 — Documentation–code alignment

Verify that all docs/, README.md, CONTRIBUTING.md content matches the implemented code.

- If mismatched AND the PRD makes it clear which is correct → fix the out-of-sync artifact.
- If mismatched AND the PRD is ambiguous → ask the user to resolve the difference. Append the
  resolution to a new "Implementation Conflict Resolutions" section at the bottom of the PRD file
  (plans/<branch>-prd.md). Do not modify any other section of the PRD.

## Check 2 — ADR sync

Verify that all applicable design choices made during implementation have been documented as ADRs
(per docs/adrs/ conventions) and that each ADR's Status field is set correctly
(Accepted, Rejected, Superseded, etc.). If ADRs are out of sync, update them and re-validate.

## Check 3 — Pre-commit checks

Run the project's verification command as specified in CLAUDE.md.
This includes linting, formatting, and tests. Must exit 0.

## Check 4 — Acceptance criteria

Verify every acceptance criterion in each developer story in the PRD has been met.

## Check 5 — Success metrics

Verify every success metric listed in the PRD's Success Metrics section has been met.

## Progress file logging

After every check (pass or fail) and after every user interaction, append a summary to
plans/<branch>-progress.txt. Use this format:

## [Date/Time] - GATE-<slug>-NN
- Checks passed: [list]
- Checks failed: [list with details]
- User decisions: [any questions asked and how the user responded]
- Remediation tasks written: [list of new task IDs added to team-state.json, or "none"]
- Next gate: [GATE-<slug>-NN+1 if scheduled, or "N/A — all checks passed"]
---

## Failure escalation

If checks 1–2 fail, fix them in-place as described above.

If checks 3–5 fail:

1. ITERATION GUARD: If the current gate iteration count (gate_iterations in team-state.json) is 4
   or higher, ask the user before proceeding. The message must include:
   - The current gate iteration number.
   - A summary of which checks have been failing and whether the same checks have failed
     repeatedly across prior gate iterations (recurring) or are new failures — read
     plans/<branch>-progress.txt to determine this.
   - What problems still remain and what remediation entries would be written if the user approves.
   - A clear question: should the escalation proceed, or should the user intervene directly?
   Do not proceed with steps 2–3 until the user responds. This guard applies on every gate
   iteration from the 4th onward.

2. Write remediation entries to team-state.json (new task objects with status: pending) and append
   a summary of each remediation entry to plans/<branch>-progress.txt.

3. Increment gate_iterations in team-state.json.

4. Signal the team lead to re-seed the native task list by sending a SendMessage to the lead:

   > GATE FAIL — <summary of which checks failed>
   > Remediation tasks written to team-state.json. Please re-seed native tasks.

   Do NOT call TaskCreate yourself — the lead is responsible for seeding the native task list
   from the remediation entries in team-state.json.
```

______________________________________________________________________

## Gate entry format reminder

- **Gate iterations:** tracked via `gate_iterations` integer in `plans/<branch>-team-state.json`.
- **Numbering:** Each successive gate increments `gate_iterations` by 1.
- **Remediation tasks:** written as new objects in the `tasks` array with `status: pending`,
  appropriate `role_hint`, and appropriate `blocked_by` entries.
- **Native task seeding:** the team lead calls `TaskCreate` for each remediation task after
  receiving the GATE FAIL signal — the gate agent does not create native tasks directly.
