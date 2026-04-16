# Scheduled Task Prompts and Lovelace Sourcing Tools

This reference defines the exact prompt text used when creating Cowork scheduled tasks, plus the Lovelace MCP tools the plugin uses for sourcing state management.

## Lovelace MCP sourcing tools

All sourcing state (projects, deliveries, feedback) lives in Supabase via the Lovelace platform. Claude reads state via these tools and never writes tracking data directly — the Slack bot handles delivery recording and feedback writes server-side.

### `create_sourcing_search`

Creates a new sourcing search.

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| company_name | string | yes | |
| role_title | string | yes | |
| slug | string | yes | URL-safe unique slug. Returns 409 on conflict. |
| use_case | string | no | recruiting (default), gtm, investment, fund_lp, advisor, other |
| search_criteria | string | no | SEARCH.md content |

Returns the full search row including `id`. **Store this `id` in the role's `config.json` as `sourcing_search_id`**.

### `get_sourcing_status`

Reads search state, deliveries, and feedback.

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| search_id | string | yes | UUID from `create_sourcing_search` |
| scope | string | no | `"search"` (default) or `"global"` |

**scope=search** returns:
- `search` — id, slug, status, last_run_at
- `deliveries` — each delivered person with: person_id, linkedin_url, full_name, score, decision (yes/maybe/no/null), feedback_notes, delivered_at
- `stats` — total, yes, maybe, no, pending counts

**scope=global** returns:
- `global_linkedin_urls` — sorted list of every LinkedIn URL delivered across ALL of this user's searches

Use `scope=search` for feedback processing and `scope=global` for cross-search dedup.

### `update_sourcing_search`

Updates a sourcing search's metadata.

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| search_id | string | yes | |
| status | string | no | active, paused, closed |
| last_run_at | string | no | ISO 8601 timestamp |
| slack_channel_id | string | no | |
| slack_channel_name | string | no | |

### `get_sourcing_criteria`

Gets the current search criteria (brain doc) for a search.

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| search_id | string | yes | Sourcing search UUID |

Returns the current criteria version content and version number.

### `get_sourcing_criteria_versions`

Lists all criteria versions for a search, with optional content.

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| search_id | string | yes | Sourcing search UUID |
| include_content | boolean | no | Include full content for each version (default false) |

Returns an array of versions with version number, created_at, and optionally content.

### `update_sourcing_criteria`

Creates a new criteria version. Auto-increments the version number and sets it as current.

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| search_id | string | yes | Sourcing search UUID |
| content | string | yes | The updated SEARCH.md / criteria content |

Returns the new version row including version number.

### `set_sourcing_criteria_version`

Reverts to a previous criteria version number.

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| search_id | string | yes | Sourcing search UUID |
| version | integer | yes | The version number to revert to |

Returns the now-current version row.

### `record_sourcing_deliveries`

Records a batch of deliveries for a sourcing search. Call this before posting Slack cards so you have `delivery_id`s for the button actions.

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| search_id | string | yes | Sourcing search UUID |
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
