# Hyperloop

AI-augmented development for engineers who stay in the loop.

Hyperloop interviews you to build a PRD (with requirement analysis and conflict resolution), then
runs an autonomous specialist agent team with back-pressure gates to implement it — while you
review, guide, and approve at every milestone.

## Prerequisites

- **Claude Code** — see the [official installation guide](https://docs.anthropic.com/en/docs/claude-code/setup).
- **Agent Teams** must be enabled. This is an experimental feature — see [Claude Code: Enable Agent Teams](https://code.claude.com/docs/en/agent-teams#enable-agent-teams) for details.
- **`gh` CLI** installed and authenticated (for PR creation).
- A **`CLAUDE.md`** at your project root that documents:
  - Your project's verification command (lint + format + tests), e.g. `uv run pre-commit run`,
    `npm run lint && npm test`, `cargo clippy && cargo test`.
  - Source directory layout and conventions
  - If you don't have one, then use `claude --init` to create one.

## Installation

Hyperloop is installed via the [hyper-plugs marketplace](../README.md):

```
/plugin marketplace add samrom3/claude-hyper-plugs
/plugin install hyperloop@hyper-plugs
```

See the [plugin marketplaces documentation](https://code.claude.com/docs/en/plugin-marketplaces) for details.

## Usage

Hyperloop is a two-step process. First you build a PRD through a structured interview, then the agent team executes it autonomously against a back-pressure gate:

```
                    ┌─────────────────────────────────────────────────────────┐
  Step 1 (/prd)     │                                                         │
                    │   You ──► Interview ──► Conflict ──► plans/<branch>     │
                    │            (3 phases)    analysis      -prd.md          │
                    └──────────────────────────────┬──────────────────────────┘
                                                   │ PRD
                    ┌──────────────────────────────▼──────────────────────────┐
  Step 2            │              Lead parses PRD → task DAG                 │
  (/hyperteam)      │                                                         │
                    │   ┌─────────┐   ┌─────────┐   ┌─────────────┐          │
                    │   │Worker A │   │Worker B │   │ Tech Writer │  . . .   │
                    │   └────┬────┘   └────┬────┘   └──────┬──────┘          │
                    │        └────────────┬┘               │                  │
                    │                     ▼                 │                  │
                    │               ┌──────────┐           │                  │
                    │               │ Reviewer │◄──────────┘                  │
                    │               │  [GATE]  │                              │
                    │               └────┬─────┘                              │
                    │                    │ pass                                │
                    │                    ▼                                     │
                    │                   PR                                     │
                    └─────────────────────────────────────────────────────────┘
```

### 1. Generate a PRD

```
/hyperloop:prd Add user authentication with OAuth2 support
```

or from a seedling document:

```
/hyperloop:prd plans/auth-seedling.md
```

The PRD skill will:

- Interview you to gather requirements (3 phases of refinement)
- Cross-check against existing ADRs and codebase patterns
- Surface conflicts and ambiguities before any code is written
- Output `plans/<branch>-prd.md`

You can generate multiple PRDs upfront — they accumulate in `plans/` and can be executed in any order.

### 2. Run the team

> BEST-PRACTICE: Run `/clear` first to avoid context drift and costs of auto compaction.

```
/hyperloop:hyperteam
```

At startup, `/hyperloop:hyperteam` scans `plans/` for `*-prd.md` files and prompts you to select one:

- If only one incomplete PRD exists, it confirms and auto-selects it.
- If multiple incomplete PRDs exist, it presents a numbered list for you to choose from.
- In-progress PRDs (those with an existing `team-state.json`) are flagged with a warning so you don't accidentally start a duplicate session.

After PRD selection, hyperloop will:

- Parse the PRD into a dependency-ordered task DAG
- Show you the plan and wait for approval
- Create a specialist agent team
- Execute all tasks with TDD, code review, and a back-pressure gate
- Offer to create a PR when everything passes

### Resuming interrupted runs

If a session ends mid-run (quota exhaustion, network drop, etc.), just run `/hyperloop:hyperteam` again.
It detects the existing `team-state.json`, shows you what's done and what's left, and picks up
where it stopped.

### Concurrent sessions

Each `/hyperloop:hyperteam` session creates its own agent team via `TeamCreate`, which
automatically scopes the native task list by team name. This means you can run multiple sessions
in parallel — open separate Claude Code sessions, then run `/hyperloop:hyperteam` in each, and select a
different PRD in each session. Sessions are fully isolated because each team has its own task list
at `~/.claude/tasks/{team-name}/`.

## Architecture

### Skills

| Skill                  | Purpose                                                                    |
| ---------------------- | -------------------------------------------------------------------------- |
| `/hyperloop:prd`       | Multi-phase PRD generator with conflict detection and requirement analysis |
| `/hyperloop:hyperteam` | Converts a PRD into an autonomous agent team with back-pressure gates      |

### Agents

These are the generalized agents included in the plugin.

| Agent                  | Role                                                                      |
| ---------------------- | ------------------------------------------------------------------------- |
| `hyperteam-lead`       | Orchestrator — monitors the run, handles failures, detects gate readiness |
| `hyperteam-reviewer`   | Reviews completed work against acceptance criteria; runs the 5-check gate |
| `hyperteam-techwriter` | Claims DOC tasks, keeps documentation in sync with implementation         |
| `hyperteam-worker`     | Fallback implementer for any task without a matching specialist           |

#### Python pack agents

The Python pack agents live flat in `agents/` alongside the core agents and are discovered automatically. The `hyperteam-lead` knows when to activate these automatically.

| Agent                         | Role                                                                   |
| ----------------------------- | ---------------------------------------------------------------------- |
| `hyperteam-py-api-scaffolder` | Creates dataclasses, ABCs, API stubs with `NotImplementedError` bodies |
| `hyperteam-py-builder`        | Implements business logic via TDD on top of existing scaffolds         |

### State management

- **`plans/<branch>-team-state.json`** — Authoritative task registry. Tracks status, blockers,
  review results, and gate iterations. Enables mid-run resume.
- **`plans/<branch>-progress.txt`** — Append-only audit log with timestamps.
- **Native task list** — Live coordination bus for agent self-claiming.

## Language packs

Hyperloop ships with a **Python pack** that provides specialized scaffolder and builder agents
for Python projects. These agents live flat in `agents/` for automatic discovery by Claude Code.

Unlike language-specific forks, a single hyperloop installation works across mixed-stack repos.
The role-hint system activates only the language-pack agents relevant to each task — if no
language-specific agent matches, the generic `hyperteam-worker` handles it.

### Adding your own language pack

1. Create agent definitions following the naming convention `hyperteam-<lang>-<role>.md`
1. Place them in your project's `.claude/agents/` directory
1. The phase-1 role-hint assignment will detect installed agents and route tasks accordingly

> **Note for plugin contributors:** Plugin-bundled agents must live flat in `agents/` (no subdirectories).
> Claude Code only scans the top-level `agents/` directory for auto-discovery. Custom pack agents
> added to a user's project go in `.claude/agents/` (unrestricted nesting) as described above.

For example, a TypeScript pack might include:

- `hyperteam-ts-scaffolder.md` — Creates interfaces, type definitions, and API stubs
- `hyperteam-ts-builder.md` — Implements business logic with test-first development

## Customization

### Verification commands

Hyperloop agents read `CLAUDE.md` to find your project's lint/test command. Ensure your
`CLAUDE.md` has a clear "Tooling" section, for example:

````markdown
## Tooling

```bash
npm run lint          # lint + format
npm test              # run tests
npm run check         # both (use this as the verification command)
`` `
````

### Overriding the PRD template

Place your own `example-prd.md` at `.claude/skills/prd/references/example-prd.md` in your
project. Project-level files take precedence over plugin files.

### Project-specific agents

Add agents to your project's `.claude/agents/` directory. They supplement the plugin's agents
and take precedence when names overlap.

## Inspiration & prior art

**Inspired by [hyperworker](https://github.com/joseph-ravenwolfe/hyperworker)** by Joseph Ravenwolfe.

**Key architectural differences in hyperloop:**

|                           | hyperworker                                    | hyperloop                                                                                                            |
| ------------------------- | ---------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| **Language support**      | Pick the language-specific version of the repo | Single install; activate language packs per task — works in mixed-stack repos                                        |
| **Requirement gathering** | Prompt-driven                                  | Structured user interview with explicit requirement analysis and conflict deconfliction before any code is written   |
| **Back-pressure gate**    | Intended but consistently absent in most forks | First-class GATE tasks with a dedicated reviewer agent; the lead blocks new work until the gate clears               |
| **Re-entrant execution**  | Single run                                     | Designed for mid-run quota exhaustion: resume picks up exactly where the last session left off via `team-state.json` |

## License

MIT
