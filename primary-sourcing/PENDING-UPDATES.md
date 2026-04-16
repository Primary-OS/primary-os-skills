# Pending Plugin Updates (April 2026)

Two rounds of Lovelace changes require updates to this plugin before the next sourcing run.

## 1. Rename: `sourcing_projects` -> `sourcing_searches`

The DB table and all FK columns were renamed. MCP tools were renamed. Every reference in this plugin to the old names must be updated.

### Find & replace

| Old | New |
|-----|-----|
| `create_sourcing_project` | `create_sourcing_search` |
| `update_sourcing_project` | `update_sourcing_search` |
| `sourcing_project_id` | `sourcing_search_id` |
| `sourcing_projects` | `sourcing_searches` |

### Files to update

| File | What to change |
|------|---------------|
| `skills/kickoff-role/SKILL.md` | 6 references: tool names on lines ~112, 119, 123, 136; field name on lines ~130, 146 |
| `skills/run-sourcing-batch/SKILL.md` | 3 references: field name line ~25, tool name line ~71, param name line ~27 |
| `skills/run-weekly-summary/SKILL.md` | 1 reference: param name line ~24 |
| `templates/role/config-template.json` | `"sourcing_project_id": ""` -> `"sourcing_search_id": ""` |
| `skills/kickoff-role/references/scheduled-task-prompts.md` | 5 references: tool name headers + table content |
| `skills/kickoff-role/references/collision-handling.md` | 2 references: tool names lines ~53, 69 |
| `skills/start-sourcing-project/references/mcp-prereqs.md` | 2 tool names in the Lovelace MCP row |
| `README.md` | 2 tool names in Architecture section line ~51 |

## 2. New MCP tools for criteria versioning

Four new tools were added. The skills should reference these instead of `search_criteria` on `update_sourcing_search`:

| Tool | Purpose |
|------|---------|
| `get_sourcing_criteria` | Get current criteria/brain doc for a search |
| `get_sourcing_criteria_versions` | List all versions (with optional content) |
| `update_sourcing_criteria` | Create a new version (auto-increments, sets as current) |
| `set_sourcing_criteria_version` | Revert to a previous version number |

### Where to update

- **`run-sourcing-batch/SKILL.md`** — Step 2 (process feedback) should mention reading criteria via `get_sourcing_criteria`. Step 8 (update brain) should use `update_sourcing_criteria` instead of `update_sourcing_search(search_criteria: ...)`.
- **`kickoff-role/references/scheduled-task-prompts.md`** — Add the 4 new tools to the MCP tool reference table.

## 3. Slack channel auto-creation

Channels are now created automatically when a sourcing search is created via `create_sourcing_search`. The kickoff skill's Step 9 (create Slack channel) is now handled by the system.

### Where to update

- **`kickoff-role/SKILL.md`** Step 9 — Remove manual channel creation. Instead, note that the channel is created automatically and the skill should wait/poll for `slack_channel_id` to appear on the search record (call `get_sourcing_status` to check).
- **`kickoff-role/SKILL.md`** Step 12 — The intro message is also posted automatically. This step can be removed or simplified to just verifying the message was posted.
- **`kickoff-role/references/collision-handling.md`** — Channel collision is now handled server-side (email prefix then numbers). Remove the manual collision instructions for channels.

## 4. Delivery flow simplification (coming soon)

`record_sourcing_deliveries` will be updated to also post Block Kit cards to Slack and record `slack_message_ts` — all server-side. The agent will no longer need to:
- Post cards to Slack manually via Slack MCP
- Call `PATCH /deliveries/{id}` to attach message timestamps

### Where to update (after Lovelace change lands)

- **`run-sourcing-batch/SKILL.md`** Step 6 — Simplify to just calling `record_sourcing_deliveries`. Remove the "post each card to Slack" substep.
- **`run-sourcing-batch/references/slack-formatting.md`** — The Block Kit format moves server-side. This reference becomes informational (describes what the system posts) rather than instructional (tells the agent how to post).
- **`kickoff-role/references/scheduled-task-prompts.md`** — Update `record_sourcing_deliveries` description to note it handles Slack posting automatically.
- **`start-sourcing-project/references/mcp-prereqs.md`** — Slack MCP becomes optional (only needed for reading messages / surface commands, not for posting cards).
