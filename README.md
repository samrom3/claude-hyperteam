# Hyperteam

AI-augmented development for engineers who stay in the loop.

Hyperteam interviews you to build a PRD (with requirement analysis and conflict resolution), then
runs an autonomous specialist agent team with back-pressure gates to implement it — while you
review, guide, and approve at every milestone.

## Prerequisites

- **Claude Code** with Agent Teams enabled:
  ```bash
  export CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1
  ```
- **`gh` CLI** installed and authenticated (for PR creation)
- A **`CLAUDE.md`** at your project root that documents:
  - Your project's verification command (lint + format + tests), e.g. `uv run pre-commit run`,
    `npm run lint && npm test`, `cargo clippy && cargo test`
  - Source directory layout and conventions

## Installation

Add to your project's `.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "hyperteam@samrom3/claude-hyperteam": true
  }
}
```

Or enable globally in `~/.claude/settings.json` to use across all projects.

## Usage

### 1. Generate a PRD

```
/prd Add user authentication with OAuth2 support
```

or from a seedling document:

```
/prd plans/auth-seedling.md
```

The PRD skill will:
- Interview you to gather requirements (3 phases of refinement)
- Cross-check against existing ADRs and codebase patterns
- Surface conflicts and ambiguities before any code is written
- Output `plans/<branch>-prd.md`

### 2. Run the team

```
/hyperteam
```

Hyperteam will:
- Parse the PRD into a dependency-ordered task DAG
- Show you the plan and wait for approval
- Create a specialist agent team
- Execute all tasks with TDD, code review, and a back-pressure gate
- Offer to create a PR when everything passes

### Resuming interrupted runs

If a session ends mid-run (quota exhaustion, network drop, etc.), just run `/hyperteam` again.
It detects the existing `team-state.json`, shows you what's done and what's left, and picks up
where it stopped.

## Architecture

### Skills

| Skill | Purpose |
|-------|---------|
| `/prd` | Multi-phase PRD generator with conflict detection and requirement analysis |
| `/hyperteam` | Converts a PRD into an autonomous agent team with back-pressure gates |

### Agents

| Agent | Role |
|-------|------|
| `hyperteam-lead` | Orchestrator — monitors the run, handles failures, detects gate readiness |
| `hyperteam-reviewer` | Reviews completed work against acceptance criteria; runs the 5-check gate |
| `hyperteam-techwriter` | Claims DOC tasks, keeps documentation in sync with implementation |
| `hyperteam-worker` | Fallback implementer for any task without a matching specialist |

### Python pack agents

| Agent | Role |
|-------|------|
| `hyperteam-py-api-scaffolder` | Creates dataclasses, ABCs, API stubs with `NotImplementedError` bodies |
| `hyperteam-py-builder` | Implements business logic via TDD on top of existing scaffolds |

### State management

- **`plans/<branch>-team-state.json`** — Authoritative task registry. Tracks status, blockers,
  review results, and gate iterations. Enables mid-run resume.
- **`plans/<branch>-progress.txt`** — Append-only audit log with timestamps.
- **Native task list** — Live coordination bus for agent self-claiming.

## Language packs

Hyperteam ships with a **Python pack** (`agents/packs/python/`) that provides specialized
scaffolder and builder agents for Python projects.

Unlike language-specific forks, a single hyperteam installation works across mixed-stack repos.
The role-hint system activates only the language-pack agents relevant to each task — if no
language-specific agent matches, the generic `hyperteam-worker` handles it.

### Adding your own language pack

1. Create agent definitions following the naming convention `hyperteam-<lang>-<role>.md`
2. Place them in your project's `.claude/agents/` directory
3. The phase-1 role-hint assignment will detect installed agents and route tasks accordingly

For example, a TypeScript pack might include:
- `hyperteam-ts-scaffolder.md` — Creates interfaces, type definitions, and API stubs
- `hyperteam-ts-builder.md` — Implements business logic with test-first development

## Customization

### Verification commands

Hyperteam agents read `CLAUDE.md` to find your project's lint/test command. Ensure your
`CLAUDE.md` has a clear "Tooling" section, for example:

```markdown
## Tooling

```bash
npm run lint          # lint + format
npm test              # run tests
npm run check         # both (use this as the verification command)
`` `
```

### Overriding the PRD template

Place your own `example-prd.md` at `.claude/skills/prd/references/example-prd.md` in your
project. Project-level files take precedence over plugin files.

### Project-specific agents

Add agents to your project's `.claude/agents/` directory. They supplement the plugin's agents
and take precedence when names overlap.

## Inspiration & prior art

**Inspired by [hyperworker](https://github.com/joseph-ravenwolfe/hyperworker)** by Joseph Ravenwolfe.

**Key architectural differences in hyperteam:**

| | hyperworker | hyperteam |
|---|---|---|
| **Language support** | Pick the language-specific version of the repo | Single install; activate language packs per task — works in mixed-stack repos |
| **Requirement gathering** | Prompt-driven | Structured user interview with explicit requirement analysis and conflict deconfliction before any code is written |
| **Back-pressure gate** | Intended but consistently absent in most forks | First-class GATE tasks with a dedicated reviewer agent; the lead blocks new work until the gate clears |
| **Re-entrant execution** | Single run | Designed for mid-run quota exhaustion: resume picks up exactly where the last session left off via `team-state.json` |

## License

MIT
