# Claude Hyper Plugs

A curated marketplace of custom Claude Code plugins built by @samrom3 with help from his friends and colleagues.

## Available Plugins

### [Hyperloop](hyperloop/)

AI-augmented development for engineers who stay in the loop. Interviews you to build a PRD (with requirement analysis and conflict resolution), then runs an autonomous specialist agent team with back-pressure gates to implement it — while you review, guide, and approve at every milestone.

- **Skills:** `/prd`, `/hyperteam`.
- **Agents:** lead, reviewer, worker, techwriter + Python language pack

Refer to the [Hyperloop guide](/hyperloop/README.md) for more usage and details.

### [adr-wizard](adr-wizard/)

ADR lifecycle management for Claude Code. Create, supersede, deprecate, and validate Architecture
Decision Records across single or multiple directories using a CLAUDE.md convention for discovery.
Integrates with `hyperloop`'s gate for automated ADR validation.

- **Skills:** `/adr-create`, `/adr-supersede`, `/adr-deprecate`, `/adr-check`

Refer to the [adr-wizard guide](/adr-wizard/README.md) for more usage and details.

## Installation

### 1. Add the marketplace

```
/plugin marketplace add samrom3/claude-hyper-plugs
```

### 2. Install a plugin

```
/plugin install hyperloop@hyper-plugs
```

See the [Claude Code plugin marketplaces documentation](https://code.claude.com/docs/en/plugin-marketplaces) for details.

## Prerequisites

- **Claude Code** — see the [official installation guide](https://docs.anthropic.com/en/docs/claude-code/setup)
- Some plugins require **Agent Teams** to be enabled — see individual plugin READMEs for specific requirements

## Contributing a Plugin

1. Create a directory at the repo root named after your plugin
1. Add a `.claude-plugin/plugin.json` with metadata:
   ```json
   {
     "name": "your-plugin",
     "version": "1.0.0",
     "description": "What your plugin does.",
     "author": {
       "name": "Your Name",
       "url": "https://github.com/you"
     },
     "license": "MIT"
   }
   ```
1. Add `agents/` and/or `skills/` with Markdown definitions
1. Add a `README.md` documenting usage
1. Open a PR

## License

MIT
