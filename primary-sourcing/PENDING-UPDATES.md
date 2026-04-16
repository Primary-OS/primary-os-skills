# Pending Plugin Updates (April 2026)

## ~~1. Rename: `sourcing_projects` -> `sourcing_searches`~~ DONE

## ~~2. New MCP tools for criteria versioning~~ DONE

## ~~3. Slack channel auto-creation~~ DONE

## 4. Delivery flow simplification (coming soon)

`record_sourcing_deliveries` will be updated to also post Block Kit cards to Slack and record `slack_message_ts` — all server-side. The agent will no longer need to:
- Post cards to Slack manually via Slack MCP
- Call `PATCH /deliveries/{id}` to attach message timestamps

### Where to update (after Lovelace change lands)

- **`run-sourcing-batch/SKILL.md`** Step 6 — Simplify to just calling `record_sourcing_deliveries`. Remove the "post each card to Slack" substep.
- **`run-sourcing-batch/references/slack-formatting.md`** — The Block Kit format moves server-side. This reference becomes informational (describes what the system posts) rather than instructional (tells the agent how to post).
- **`kickoff-role/references/scheduled-task-prompts.md`** — Update `record_sourcing_deliveries` description to note it handles Slack posting automatically.
- **`start-sourcing-project/references/mcp-prereqs.md`** — Slack MCP becomes optional (only needed for reading messages / surface commands, not for posting cards).
