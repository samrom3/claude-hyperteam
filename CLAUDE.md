# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

This repository is a **Claude Code plugin marketplace** — a curated collection of Claude Code plugins, each in its own subdirectory. Each plugin is self-contained with its own `.claude-plugin/plugin.json`, agents, skills, and documentation.

This is **not** a traditional software project with build/test/lint commands. It is a collection of Markdown-based agent definitions and skill specifications that form Claude Code plugins.

## Repository Structure

```
claude-hyper-plugs/
├── CLAUDE.md                  # This file (repo-level guidance)
├── README.md                  # Marketplace index and installation guide
├── .pre-commit-config.yaml    # Shared formatting/linting hooks
└── <plugin-name>/             # One directory per plugin
    ├── .claude-plugin/
    │   └── plugin.json        # Plugin metadata (name, version, description, author)
    ├── agents/                # Agent role definitions (Markdown with YAML front-matter)
    ├── skills/                # Skill definitions (SKILL.md + references/)
    └── README.md              # Plugin-specific documentation
```

## Current Plugins

| Plugin      | Directory    | Description                                                         |
| ----------- | ------------ | ------------------------------------------------------------------- |
| `hyperteam` | `hyperteam/` | Autonomous specialist agent team with PRD-driven planning and gates |

## Verification

Pre-commit hooks handle formatting, linting, and plugin validation:

```bash
pre-commit run --all-files    # trailing whitespace, EOF fixer, YAML check, merge conflict check, mdformat, plugin validate
```

- mdformat (with GFM, footnote, config, ruff, and toc plugins) runs on all Markdown files **except** those under `skills/` and `agents/` (excluded because agent/skill prompts use intentional formatting).
- `claude plugin validate .` validates the marketplace manifest and all plugin manifests/frontmatter.

You can also run validation manually:

```bash
claude plugin validate .
```

## Adding a New Plugin

1. Create a new directory at the repo root: `<plugin-name>/`
1. Add `.claude-plugin/plugin.json` with the plugin metadata
1. Add `agents/` and/or `skills/` directories with Markdown definitions
1. Add a `README.md` documenting the plugin's purpose, installation, and usage
1. Update the "Current Plugins" table above and the root `README.md`

## Plugin Anatomy

Each plugin follows the Claude Code plugin spec:

- **`.claude-plugin/plugin.json`** — Entry point. Declares name, version, description, author, license.
- **`agents/<name>.md`** — Agent definitions with YAML front-matter (`name`, `description`, `model`, `permissionMode`).
- **`skills/<name>/SKILL.md`** — Skill definitions with YAML front-matter (`name`, `description`, `user-invocable`). Supporting files go in `references/`.
- **`agents/packs/<lang>/`** — Optional language-specific agent packs.
