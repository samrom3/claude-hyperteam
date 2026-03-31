---
name: adr-create
description: "This skill should be used when the user asks to 'create an ADR', 'document an architectural decision', 'record a design decision', 'write an architecture decision record', or when the model identifies an architectural decision in context that warrants formal documentation. Evaluates decision worthiness before authoring and guides section drafting."
argument-hint: "Brief description of the architectural decision (e.g., 'Use PostgreSQL for primary storage')"
user-invocable: true
---

# adr-create

Creates a new ADR file with auto-numbering, fills it from the Nygard template, and updates the
directory's README.md index.

---

## Step 0 — Evaluate decision worthiness

### Phase A — Silent evaluation (always runs first)

Using domain knowledge and the full conversation context, assess the decision across these
axes before engaging the user at all:

| Axis | What to assess |
|------|---------------|
| **Alternatives** | Do obvious alternatives exist in this domain? Would a knowledgeable engineer have weighed options, or was this effectively forced? |
| **Scope** | Does the decision cross module, team, or service boundaries — or is it local to one function or file? |
| **Reversibility** | Is the cost of undoing this significant (data migration, API contract change, team retraining)? |
| **Tradeoffs** | Does the conversation context already surface what was given up? |
| **Existing ADRs** | Does this decision extend, contradict, or supersede a prior architectural choice? |

**If Phase A yields a clear verdict**, act on it directly — no debate needed:

- **Clear yes** (cross-cutting, meaningful alternatives existed, non-trivial reversal cost, or
  intersects an existing ADR): proceed to Step 1 without engaging the user.
- **Clear no** (routine implementation detail, derivable from code, no alternatives were
  ever in play, entirely local): inform the user plainly, suggest a lighter alternative (inline
  comment, team discussion note), and stop. No escape hatch — the assessment is confident.

**If Phase A is inconclusive** — alternatives are ambiguous, tradeoffs are not surfaced in
context, or the decision boundaries are unclear — proceed to Phase B.

---

### Phase B — Adversarial debate (only when Phase A is inconclusive)

Engage the user to extract the information Phase A could not resolve. The debate loads the
agent's context with the substance Steps 4–5 will need; it also surfaces the decision's true
shape, which may not match the user's original framing.

**Your posture is adversarial-constructive.** Argue against the ADR's necessity — or against the
framing — until the picture is clear. State challenges directly; do not soften them into
questions with obvious answers.

**Challenge axes** (raise 1–3 based on what Phase A left unresolved):

| Axis | Counterpoint to raise |
|------|-----------------------|
| **Scope** | "This sounds contained to [X]. Why does someone working on a different part of the system six months from now need to know about it?" |
| **Obviousness** | "An experienced engineer reading the code would likely infer this from [Y]. What would they miss that the ADR adds?" |
| **Reversibility** | "What is the real cost of undoing this? If it is low, a permanent record may not be warranted." |
| **Alternatives** | "What alternatives did you actually consider? If only one option was ever on the table, this may be recording an outcome rather than a decision." |
| **Novelty** | "Is this consistent with an existing ADR? If so, it may be implementing an already-approved pattern rather than making a new decision." |

Conduct 1–2 exchanges. Steelman the user's responses, then counter if the case is still
unconvincing. The debate ends when the picture is clear enough to route to an outcome.

---

### Outcome routing (after Phase A or Phase B)

| Outcome | Action |
|---------|--------|
| **Single ADR warranted** | Proceed to Step 1. |
| **Multiple ADRs warranted** | The decision is compound. Surface each distinct decision to the user: "This looks like two separate decisions: [A] and [B]. I'll create them in sequence — starting with [A]." Proceed to Step 1 for the first; loop back to Step 0 for each subsequent. |
| **Supersede or deprecate warranted** | The decision changes a prior architectural choice. Surface this: "This appears to supersede ADR-NNNN ([title]). I'll hand off to `/adr-supersede`." Stop and invoke the appropriate lifecycle skill. |
| **Decision does not warrant an ADR** (Phase A clear no, or debate concludes no) | Explain why confidently. If Phase A was the source (clear no), stop without offering an escape hatch. If the debate concludes no, offer the escape hatch: "I'm not convinced this clears the bar for a permanent ADR. That said, you're the author — proceed anyway, or revisit the framing?" Use `AskUserQuestion` with **Proceed anyway** and **Revisit framing**. Revisit restarts Phase B with the updated framing; proceed continues to Step 1. |

## Step 1 — Discover ADR directories

1. Read the project's `CLAUDE.md`.
2. Search for any heading containing the text `ADR Locations` (case-insensitive, any heading
   level, e.g., `## ADR Locations`, `### ADR Locations`).
3. If found, read every bullet list item under that heading as a relative path. Ignore any inline
   `# comment` annotation after the path (strip from `#` to end of line). Collect all paths into
   `adr_dirs`.
4. If **no** such heading exists in CLAUDE.md, fall back: scan the repository root for
   directories named `docs/adrs`, `decisions`, or `architecture/decisions`. Use whichever exist
   as `adr_dirs`.
5. If `adr_dirs` is empty, offer to bootstrap one for the user using `AskUserQuestion`:
   > No ADR directories found. Would you like me to create one?

   Options:
   - `docs/adrs/` (recommended — standard location)
   - Other (let the user type a custom path)

   If the user confirms:
   a. Create the chosen directory.
   b. Copy the README template from [`references/README-template.md`](references/README-template.md)
      into `<chosen_dir>/README.md`, replacing the `<PROJECT_NAME>` placeholder with the actual
      project name (inferred from the repo root directory name or `package.json`/`pyproject.toml`
      if available).
   c. Copy [`references/0000-adr-template.md`](references/0000-adr-template.md) into the new
      directory as the template reference file.
   d. Add a `### ADR Locations` section to the project's `CLAUDE.md` (or create one if absent)
      with the new directory path.
   e. Set `adr_dirs` to the newly created directory and continue to Step 2.

## Step 2 — Select target directory

1. If `adr_dirs` has exactly one entry, use it as `target_dir`.
2. If `adr_dirs` has multiple entries:
   a. Identify the directory whose path best matches the user's recent file context. "Recent
      file context" means: any files mentioned in the current conversation, files opened in the
      editor, or the directory containing the file most recently mentioned. Choose the ADR
      directory whose path shares the longest common prefix with that context.
   b. Present this as the suggested default:
      > Suggested ADR directory: `<target_dir>` (based on recent file context)
      > Other options: `<list remaining dirs>`
      > Press Enter to accept, or type the path of another directory.
   c. Wait for user confirmation or override.
   d. Use the confirmed path as `target_dir`.

## Step 3 — Determine next ADR number

1. List all files in `target_dir` matching the pattern `NNNN-*.md` (where NNNN is one or more digits).
2. Find the highest number among all matching filenames.
3. `next_num = highest + 1` (or `1` if no matching files exist).
4. Zero-pad `next_num` to 4 digits: e.g., `1` → `0001`, `12` → `0012`.

## Step 4 — Parse the decision context

The user may provide the decision as an argument to `/adr-create <decision summary>`, or it may
be available from the conversation context (e.g., the user just discussed an architectural choice).

1. **Argument provided** (e.g., `/adr-create Use PostgreSQL for primary storage`): use the
   argument text as `decision_summary`.
2. **No argument, but conversation context** contains a clear architectural decision: extract the
   decision from context as `decision_summary`.
3. **No argument and no clear context**: ask:
   > What architectural decision are you recording? (e.g., "Use PostgreSQL for primary storage")

From `decision_summary` and the full conversation context, derive:
- `adr_title`: a concise title for the ADR (the summary itself, or a shortened form if verbose).

Convert `adr_title` to a kebab-case slug for the filename: lowercase, spaces and special
characters replaced with hyphens.

## Step 5 — Draft all ADR sections

The skill is responsible for authoring every section of the ADR — do not leave template
placeholders for the user. Infer content from the conversation context, codebase state, and
`decision_summary`. When information is insufficient to write a section confidently, use
`AskUserQuestion` to interview the user before proceeding.

For each section, gather information as follows:

### Context

**Quality criteria:** Describe the problem and the forces acting on the decision — not the
solution. Avoid advocating for the chosen approach in this section. The reader should understand
why a decision was needed, not be pre-sold on the answer.

Write a short paragraph explaining **why** this decision is needed. Draw from:
- The conversation history (what problem was being discussed?)
- Recent code changes or files under discussion
- Any constraints, requirements, or trade-offs mentioned

If the conversation does not provide enough context, ask:
> What problem or situation prompted this decision? What constraints are in play?

### Decision

**Quality criteria:** Use active voice — "We decided to X" or "We will use Y". Be specific and
declarative. Avoid passive constructions such as "it was felt that" or "it was agreed". State
exactly what was chosen, not why (that belongs in Context) or what happens next (that belongs in
Consequences).

Write a few sentences stating **what** was decided. Be specific and declarative (e.g., "We will
use PostgreSQL 16 as the primary data store for all user-facing services"). Draw from:
- The `decision_summary` argument
- Any explicit choices made in the conversation

If the decision is ambiguous (e.g., the user only gave a vague title), ask:
> Can you state the decision more precisely? What exactly are we choosing to do?

### Consequences

**Quality criteria:** Must include at least one genuinely adverse consequence or trade-off. An
ADR that only lists benefits is less credible — every architectural choice involves giving
something up. Target 200–400 words total for the consequences section. If substantially more is
needed, consider whether the decision should be split into two ADRs.

Write 3–7 bullet points covering both **positive** and **negative** consequences of this
decision. Include:
- Benefits the team expects to gain
- Trade-offs or risks being accepted (at least one — required for credibility)
- Any follow-up work this decision creates
- Migration or compatibility implications, if applicable

If consequences cannot be inferred from context, ask:
> What are the main benefits and trade-offs of this decision? Any follow-up work it creates?

### Interview flow

Batch questions — if clarity is needed on multiple sections, ask them together in a single
`AskUserQuestion` call rather than one at a time. Proceed to file creation only after all
sections have substantive content.

## Step 6 — Create the ADR file

1. Construct filename: `<target_dir>/<NNNN>-<slug>.md` where NNNN is the zero-padded number and
   slug is the kebab-case title.
2. Fill in the template from [`references/0000-adr-template.md`](references/0000-adr-template.md):
   - Replace `NNNN` with the zero-padded number.
   - Replace `Title` with `adr_title`.
   - Set `**Status:** Proposed` (new ADRs always start as Proposed unless the user specifies
     otherwise).
   - Set `**Date:**` to today's date in `YYYY-MM-DD` format.
   - Write the drafted Context, Decision, and Consequences content into their respective sections.
3. Write the file to disk.

## Step 7 — Update the index

1. Check if `<target_dir>/README.md` exists.
   - If it does not exist, create it with this structure:
     ```markdown
     # Architecture Decision Records

     | ADR | Title | Status |
     |-----|-------|--------|
     ```
2. Add a new row to the table in `README.md`:
   ```
   | [ADR-NNNN](<NNNN>-<slug>.md) | <adr_title> | Proposed |
   ```
   Insert it at the end of the table (after the last existing row).
3. Save `README.md`.

## Step 8 — Confirm

Inform the user:

> Created `<target_dir>/<NNNN>-<slug>.md` (ADR-NNNN: <adr_title>)
> Updated index: `<target_dir>/README.md`
>
> Review the ADR and change the Status to `Accepted` when the decision is finalised.
>
> Tip: commit this ADR in the same PR as (or just before) the code it describes so the decision
> and implementation are traceable together.

## Step 9 — Post-write validation

1. Invoke `adr-check` in scoped mode against the newly created ADR file:
   `adr-check <target_dir>/<NNNN>-<slug>.md`
2. Review the result:
   - **Structural FAIL:** Block completion and present the issues to the user. Prompt them to
     resolve the structural problems before the skill confirms completion. Offer to fix the issues
     directly if they are straightforward (e.g., a missing section).
   - **Style warnings:** Display the warnings to the user but do not block. The ADR is considered
     complete; the user may choose to address the warnings or not.
   - **PASS (no warnings):** The skill completes successfully with no further action.
