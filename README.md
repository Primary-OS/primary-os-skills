<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset=".github/assets/primary-logo.svg">
    <source media="(prefers-color-scheme: light)" srcset=".github/assets/primary-logo-dark.svg">
    <img alt="Primary" src=".github/assets/primary-logo-dark.svg" width="320">
  </picture>
</p>

<h3 align="center">Skills Marketplace</h3>

<p align="center">
  Internal workflows, automation, and AI-assisted tools for the Primary team.<br>
  Model-agnostic — works with Claude Code, Cowork, Codex, and other AI agents.
</p>

---

## Overview

This repo is the skills marketplace for [Primary Venture Partners](https://primary.vc). It contains a catalog of plugins (`.claude-plugin/marketplace.json`) and each plugin's source under `plugins/`.

Add the marketplace once and new plugins become available automatically as they ship.

## Available plugins

| Plugin | Version | Description |
|--------|---------|-------------|
| **primary-sourcing** | `0.1.0-alpha` | Team-wide sourcing agent. Scaffold a project per portfolio company or sourcing target, kick off searches, run automated LinkedIn sourcing with Slack feedback loops and weekly digests. Supports recruiting, GTM, investment, fund/LP, advisor, and custom use cases. |

## Installation

### Cowork

1. Open a Cowork session and click **Customize** in the left sidebar.
2. Click **+** next to "Personal plugins", then **Create plugin** > **Add marketplace**.
3. Paste `Primary-OS/primary-os-skills` and confirm.
4. Go to **Browse plugins** and click **Install** on any plugin.

### Claude Code CLI

```bash
# Add the marketplace (one-time)
claude plugin marketplace add Primary-OS/primary-os-skills

# Install a plugin
claude plugin install primary-sourcing@primary-os
```

Or inside an interactive session:

```
/plugin marketplace add Primary-OS/primary-os-skills
/plugin install primary-sourcing@primary-os
```

Browse all available plugins interactively with `/plugin`. Updates pull automatically on startup — force-refresh with `/plugin marketplace update primary-os`.

### VS Code and JetBrains

Open the Claude Code panel and run the same slash commands as the CLI:

```
/plugin marketplace add Primary-OS/primary-os-skills
/plugin install primary-sourcing@primary-os
```

### Codex and other agents

The `.claude-plugin/` format is specific to Claude Code and Cowork. However, all skill content in this repo is model-agnostic — the procedures, scoring rubrics, and reference docs work with any LLM-based agent.

1. Clone this repo.
2. Point your agent at the relevant skill files under `plugins/<plugin>/skills/<skill-name>/`.
3. Each skill contains a `SKILL.md` (the full procedure) and a `references/` directory with supporting docs.
4. The skills reference MCP connectors (Airtable, Slack, etc.) — map these to whatever tool-use mechanism your agent supports.

**Key paths for direct integration:**

```
primary-sourcing/
  skills/
    start-sourcing-project/SKILL.md    # Scaffold a new project
    kickoff-role/SKILL.md              # Kick off a search
    run-sourcing-batch/SKILL.md        # Run a sourcing batch
    run-weekly-summary/SKILL.md        # Post a weekly digest
  templates/                           # Project and search templates
  CONNECTORS.md                        # Required and optional integrations
```

## Managed distribution

For hands-off team rollout, IT can push the marketplace and enabled plugins through Claude managed settings — no manual install needed per teammate. See [`docs/managed-settings.md`](docs/managed-settings.md).

## Contributing

1. Add your plugin under `<plugin-name>/` with a `.claude-plugin/plugin.json` manifest.
2. Register it in `.claude-plugin/marketplace.json`.
3. Open a pull request. Keep plugins self-contained.

## Support

Open an issue in this repo or reach out in `#ai-tools` on Slack.

---

<p align="center">
  <a href="https://primary.vc">primary.vc</a>
</p>
