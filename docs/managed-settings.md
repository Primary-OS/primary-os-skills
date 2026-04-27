# Managed Settings for the Primary OS Marketplace

This doc is for Primary IT / Hi-Tech. It explains how to roll out the `primary-os` plugin marketplace to every teammate's Cowork automatically through Claude managed settings, so teammates don't have to run `/plugin marketplace add` individually.

For background on how Claude managed settings work, see [Claude's settings docs](https://code.claude.com/docs/en/settings).

## What "managed settings" lets us do

1. **Auto-register the marketplace.** Every teammate's Cowork picks up the marketplace on startup — no `/plugin marketplace add` needed.
2. **Auto-enable specific plugins.** Brand-new Cowork sessions for Primary employees come with `primary-sourcing` installed and active.
3. **Lock the allowlist.** Prevent teammates from adding random third-party marketplaces if that's a policy requirement.

## Recommended managed settings JSON

Add this to Primary's Claude managed settings file (the location depends on the MDM or config management tool you use to push managed settings to employee machines):

```json
{
  "extraKnownMarketplaces": {
    "primary-os": {
      "source": {
        "source": "github",
        "repo": "Primary-OS/primary-os-skills"
      }
    }
  },
  "enabledPlugins": {
    "primary-sourcing@primary-os": true
  }
}
```

### What each key does

- `extraKnownMarketplaces.primary-os` — tells Cowork that the `primary-os` marketplace lives at `github.com/Primary-OS/primary-os-skills`. Cowork will fetch `.claude-plugin/marketplace.json` from that repo on startup.
- `enabledPlugins["primary-sourcing@primary-os"]` — enables the plugin by default for every teammate. Set to `false` to install-but-not-enable, or remove the entry entirely to let teammates choose.

## Optional: strict marketplace allowlist

If Primary policy requires locking down which marketplaces teammates can add:

```json
{
  "strictKnownMarketplaces": [
    {
      "source": "github",
      "repo": "Primary-OS/primary-os-skills"
    },
    {
      "source": "github",
      "repo": "anthropics/claude-code-plugins"
    }
  ]
}
```

This blocks any marketplace add that doesn't match the allowlist. Add Anthropic's official marketplace to the list unless there's a reason not to — it's how teammates get first-party skills.

## Private repo authentication

`Primary-OS/primary-os-skills` is a private repository. Teammates' Cowork needs git credentials to clone it. Options:

### Option A — personal GitHub auth (simple, default)

Teammates run `gh auth login` once on their machines. Cowork uses the GitHub CLI's credential helper.

**Upside**: zero extra config.
**Downside**: background auto-updates won't run if the user hasn't logged in. Teammates would need to `/plugin marketplace update` manually to get new versions.

### Option B — managed `GITHUB_TOKEN` (recommended for auto-updates)

Push a `GITHUB_TOKEN` env var to every teammate's shell via MDM. The token needs `repo` scope on the `Primary-OS` org. Cowork uses it for background marketplace pulls on startup.

```bash
# in ~/.zshrc or ~/.bashrc, managed by MDM
export GITHUB_TOKEN=ghp_xxxxxxxxxxxxxxxxxxxx
```

**Upside**: auto-updates work silently on every Cowork launch.
**Downside**: requires a managed token with scope to the org.

Anthropic docs recommend Option B for enterprise deployments. See: https://code.claude.com/docs/en/plugin-marketplaces § Private repositories.

## Rollout playbook

1. Verify `Primary-OS/primary-os-skills` repo exists and contains this marketplace.
2. Add the `extraKnownMarketplaces` + `enabledPlugins` config to managed settings.
3. Roll out `GITHUB_TOKEN` (Option B above) or verify all teammates have `gh auth` set up (Option A).
4. Push managed settings to a small pilot group first (Logan + Annabelle + one other recruiter). Verify the plugin appears in their Cowork.
5. Expand to the whole team.

## How to verify it worked for a teammate

On their machine, they should be able to run:

```
/plugin marketplace list
```

…and see `primary-os` in the list. And:

```
/plugin list
```

…should show `primary-sourcing@primary-os` as installed and enabled.

They should also be able to say in any Cowork conversation "kick off a new search" and have the `primary-sourcing:kickoff-search` skill trigger.

## Updates

Once managed settings is rolled out, **no further IT action is required for plugin updates**. We push new versions by:

1. Updating the plugin in `Primary-OS/primary-os-skills`.
2. Bumping the version in `primary-sourcing/.claude-plugin/plugin.json`.
3. Merging to `main`.

Teammates pick up the new version on next Cowork startup (or sooner if they run `/plugin marketplace update primary-os`).

## Contact

Questions on this rollout: `#ai-tools` Slack or ping Logan Nash (`logan@primary.vc`) directly.
