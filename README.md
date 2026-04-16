# Primary OS — Skills Marketplace

Official skills marketplace for Primary Venture Partners. Internal workflows, automation, and AI-assisted tools built for the Primary team.

This repo holds the catalog of plugins (`.claude-plugin/marketplace.json`) plus each plugin's source under `plugins/`. All skill content is written in model-agnostic language so it works across Claude Code, Cowork, Codex, and other AI coding agents.

## Installation

### Claude Cowork (web & desktop)

1. Open a Cowork session.
2. Click **Customize** in the left sidebar.
3. Click the **+** next to "Personal plugins".
4. Select **Create plugin** → **Add marketplace**.
5. Paste `Primary-OS/primary-os-skills` and confirm.
6. The `primary-os` marketplace and its plugins are now available under **Browse plugins**.
7. Click **Install** on any plugin you want (e.g. `primary-sourcing`).

To verify: click **Skills** in the sidebar — you should see the installed plugin's skills listed.

### Claude Code (CLI)

Add the marketplace (one-time):

```bash
claude plugin marketplace add Primary-OS/primary-os-skills
```

Or use the slash command inside an interactive session:

```
/plugin marketplace add Primary-OS/primary-os-skills
```

Then install a plugin:

```bash
# CLI
claude plugin install primary-sourcing@primary-os

# Or inside a session
/plugin install primary-sourcing@primary-os
```

To browse all available plugins interactively:

```
/plugin
```

Updates pull automatically on startup. To force-refresh:

```
/plugin marketplace update primary-os
```

### Claude Code (VS Code / JetBrains extensions)

The extensions use the same plugin system as the CLI. Open the Claude Code panel and run:

```
/plugin marketplace add Primary-OS/primary-os-skills
/plugin install primary-sourcing@primary-os
```

### OpenAI Codex / other agents

The `.claude-plugin/` marketplace format is specific to Claude Code and Cowork. However, all skill content in this repo is model-agnostic — the `SKILL.md` files, reference docs, templates, and scoring rubrics work with any LLM-based agent.

To use with Codex or other agents:

1. Clone this repo:
   ```bash
   git clone https://github.com/Primary-OS/primary-os-skills.git
   ```
2. Point your agent at the relevant skill files. Each skill lives under `plugins/<plugin>/skills/<skill-name>/` and contains:
   - `SKILL.md` — the full procedure (steps, prompts, decision logic)
   - `references/` — supporting docs (scoring rubrics, algorithms, schemas)
3. Load the skill content into your agent's context or system prompt. The skills reference MCP connectors (Airtable, Slack, etc.) — map these to whatever tool-use mechanism your agent supports.

Key files for Codex users:

| Path | What it is |
|------|------------|
| `plugins/primary-sourcing/skills/*/SKILL.md` | Step-by-step procedures |
| `plugins/primary-sourcing/skills/*/references/` | Scoring rubrics, dedup algorithm, schemas |
| `plugins/primary-sourcing/templates/` | Project and search templates |
| `plugins/primary-sourcing/CONNECTORS.md` | Required and optional integrations |

## Plugins currently available

| Plugin | Version | What it does |
|--------|---------|-------------|
| `primary-sourcing` | 0.1.0-alpha | Team-wide sourcing agent. Scaffold a portfolio company Project, kick off role searches, run automated LinkedIn sourcing with Slack feedback loops and weekly digests. Supports six use cases: recruiting, GTM, investment, fund/LP, advisor, and custom. |

New plugins will be added here as they ship.

## Managed distribution (IT)

For a fully hands-off rollout, IT can add the marketplace and enabled plugins to Primary's Claude managed settings so every teammate's Cowork picks them up automatically — no manual install needed. See `docs/managed-settings.md` for the recommended configuration.

## Contributing a new plugin

1. Add your plugin folder under `plugins/<plugin-name>/` with a valid `.claude-plugin/plugin.json` manifest.
2. Register it in the top-level `.claude-plugin/marketplace.json` under `plugins`.
3. Open a pull request. Keep the plugin self-contained — no references to files outside its directory.

## Support

Open an issue in this repo, or ping `#ai-tools` in Slack.
