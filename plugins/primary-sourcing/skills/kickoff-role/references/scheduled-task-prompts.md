# Scheduled Task Prompts and Lovelace Sourcing Tools

This reference defines the exact prompt text used when creating Cowork scheduled tasks, plus the Lovelace MCP tools the plugin uses for sourcing state management.

## Lovelace MCP sourcing tools

All sourcing state (projects, deliveries, feedback) lives in Supabase via the Lovelace platform. Claude records deliveries via `record_sourcing_deliveries` and reads state via `get_sourcing_status`. The Slack bot handles feedback writes when users click buttons.

### `create_sourcing_project`

Creates a new sourcing project.

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| company_name | string | yes | |
| role_title | string | yes | |
| slug | string | yes | URL-safe unique slug. Returns 409 on conflict. |
| use_case | string | no | recruiting (default), gtm, investment, fund_lp, advisor, other |
| slack_channel_id | string | no | Set after channel creation via `update_sourcing_project` |
| slack_channel_name | string | no | |
| search_criteria | string | no | SEARCH.md content |

Returns the full project row including `id`. **Store this `id` in the role's `config.json` as `sourcing_project_id`**.

### `get_sourcing_status`

Reads project state, deliveries, and feedback.

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| project_id | string | yes | UUID from `create_sourcing_project` |
| scope | string | no | `"project"` (default) or `"global"` |

**scope=project** returns:
- `project` — id, slug, status, last_run_at
- `deliveries` — each delivered person with: person_id, linkedin_url, full_name, score, decision (yes/maybe/no/null), feedback_notes, delivered_at
- `stats` — total, yes, maybe, no, pending counts

**scope=global** returns:
- `global_linkedin_urls` — sorted list of every LinkedIn URL delivered across ALL of this user's projects

Use `scope=project` for feedback processing and `scope=global` for cross-project dedup.

### `update_sourcing_project`

Updates a sourcing project's metadata.

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| project_id | string | yes | |
| search_criteria | string | no | Updated SEARCH.md content |
| status | string | no | active, paused, closed |
| last_run_at | string | no | ISO 8601 timestamp |
| slack_channel_id | string | no | |
| slack_channel_name | string | no | |

### `record_sourcing_deliveries`

Records a batch of deliveries for a sourcing project. Call this before posting Slack cards so you have `delivery_id`s for the button actions.

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| project_id | string | yes | Sourcing project UUID |
| deliveries | array | yes | Each: `{ person_id, score?, rationale? }` |

Returns the created delivery rows including `id` (the `delivery_id` to embed in Slack button actions).

## What the Slack bot handles (not Claude)

The Lovelace Slack bot handles user interactions on posted cards:

1. **Recording feedback** — when a user clicks Yes/Maybe/Pass, the bot writes the decision to Supabase
2. **Updating Slack cards** — the bot replaces action buttons with a status line after feedback

## Scheduled task prompts

Cowork scheduled tasks run a natural language prompt. Keep prompts concise — the skills hold the actual logic. The prompt is identical regardless of use case; the skill reads the use case from config.json and adapts.

### Sourcing task prompt

```
Run the primary-sourcing:run-sourcing-batch skill for search slug [SEARCH_SLUG].
The search folder lives in this Cowork Project at roles/[SEARCH_SLUG]/.
Use connected MCPs: Slack (for posting cards), Lovelace (for LinkedIn search and sourcing state).
```

### Weekly summary task prompt

```
Run the primary-sourcing:run-weekly-summary skill for search slug [SEARCH_SLUG].
Post the weekly digest to the search's Slack channel.
```

## Cadence options to offer

When asking the user about sourcing cadence, offer these defaults via AskUserQuestion:

- **"Mon/Wed/Fri 9:30am ET"** — highest-volume option.
- **"Tue/Thu 9:30am ET"** — the current default Annabelle uses.
- **"Weekly, Monday 9am ET"** — for slower-moving searches.
- **"Manual only — I'll trigger runs myself"** — skips the scheduled task entirely.

For the weekly summary, offer Friday 4pm ET or skip.
