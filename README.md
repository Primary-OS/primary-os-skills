# Primary OS — Skills Marketplace

Official plugin marketplace for Primary Venture Partners. Internal workflows, automation, and AI-assisted tools built for the Primary team.

This repo holds the catalog of plugins (`.claude-plugin/marketplace.json`) plus each plugin's source under `plugins/`. Teammates add the marketplace once in Cowork or Claude Code, and new plugins become available automatically as they're published here.

## Install the marketplace

Once, from any Cowork or Claude Code session:

```
/plugin marketplace add Primary-OS/primary-os-skills
```

Or from the CLI:

```
claude plugin marketplace add Primary-OS/primary-os-skills
```

For private repositories, make sure `gh auth login` (or an equivalent credential helper) is configured so the clone can authenticate.

Updates come automatically — the agent refreshes the marketplace on startup. To pull updates on demand:

```
/plugin marketplace update primary-os
```

## Install a plugin from the marketplace

```
/plugin install <plugin-name>@primary-os
```

For example:

```
/plugin install primary-sourcing@primary-os
```

## Plugins currently available

| Plugin              | What it does                                                                                                                                |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| `primary-sourcing`  | Team-wide sourcing agent. Scaffold a portfolio company Project, kick off role searches, run automated LinkedIn sourcing with Slack loops.    |

New plugins will be added here as they ship.

## Managed distribution

For a fully hands-off rollout, IT can add the marketplace and enabled plugins to Primary's Claude managed settings so every teammate's Cowork picks them up automatically. See `docs/managed-settings.md` for the recommended configuration.

## Contributing a new plugin

1. Add your plugin folder under `plugins/<plugin-name>/` with a valid `.claude-plugin/plugin.json` manifest.
2. Register it in the top-level `.claude-plugin/marketplace.json` under `plugins`.
3. Open a pull request. Keep the plugin self-contained — no references to files outside its directory.

## Support

Open an issue in this repo, or ping `#ai-tools` in Slack.
