# Lovelace Migration: Airtable → Supabase

This document describes the changes made in the Lovelace platform (Apr 2026) that replace Airtable with Supabase for all sourcing state management. Use this to update every skill, reference, and connector doc in this plugin.

## What Changed and Why

Airtable was the source of truth for sourcing history, dedup state, and feedback. This created problems:
- Every user needed their own Airtable base with a specific schema
- The Claude agent had to write structured Airtable records correctly every time — fragile
- Feedback from Slack buttons had to round-trip through the agent to reach Airtable

Now Lovelace owns all sourcing state server-side. Claude records deliveries via `record_sourcing_deliveries` and reads state via MCP tools. The Slack bot handles feedback writes when users click buttons — no agent involvement for feedback.

## New Lovelace MCP Tools

These four tools replace all Airtable reads and writes:

### `create_sourcing_project`

Replaces: creating an Airtable Search record.

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| company_name | string | yes | |
| role_title | string | yes | |
| slug | string | yes | URL-safe unique slug. 409 on conflict. |
| use_case | string | no | recruiting (default), gtm, investment, fund_lp, advisor, other |
| slack_channel_id | string | no | Set after channel creation |
| slack_channel_name | string | no | |
| search_criteria | string | no | SEARCH.md content |

Returns the full project row including `id`. **Store this `id` in the role's `config.json` as `sourcing_project_id`** — it replaces the Airtable record ID everywhere (button values, dedup queries, feedback writes).

### `get_sourcing_status`

Replaces: reading Airtable Feedback records, Served Leads, Candidates, and burned flags.

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| project_id | string | yes | UUID from create_sourcing_project |
| scope | string | no | `"project"` (default) or `"global"` |

**scope=project** returns:
- `project` — id, slug, status, last_run_at
- `deliveries` — each delivered person with: person_id, linkedin_url, full_name, score, decision (yes/maybe/no/null), feedback_notes, delivered_at
- `stats` — total, yes, maybe, no, pending counts

**scope=global** returns:
- `global_linkedin_urls` — sorted list of every LinkedIn URL delivered across ALL of this user's projects

Use `scope=project` for feedback processing (Step 2) and `scope=global` for dedup (Step 5).

### `update_sourcing_project`

Replaces: updating the Airtable Search record.

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| project_id | string | yes | |
| search_criteria | string | no | Updated SEARCH.md content |
| status | string | no | active, paused, closed |
| last_run_at | string | no | ISO 8601 timestamp |
| slack_channel_id | string | no | |
| slack_channel_name | string | no | |

### `record_sourcing_deliveries`

Records a batch of deliveries for a sourcing project. Claude calls this **before** posting Slack cards so it has `delivery_id`s for the button actions.

| Parameter | Type | Required | Notes |
|-----------|------|----------|-------|
| project_id | string | yes | Sourcing project UUID |
| deliveries | array | yes | Each: `{ person_id, score?, rationale? }` |

Returns the created delivery rows including `id` (the `delivery_id` to embed in Slack button actions).

## What the Slack Bot Now Handles (Not Claude)

The Lovelace Slack bot handles user interactions on posted cards:

1. **Recording feedback** — when a user clicks Yes/Maybe/Pass, the bot calls `POST /sourcing/feedback` with decision, rejection_reason, notes
2. **Updating Slack cards** — the bot replaces the action buttons with a status line (e.g. "Marked as *Yes* by @logan")
3. **Direction messages** — non-command channel text is stored as feedback with `decision="direction"` via `POST /sourcing/feedback/direction`

## What Claude Still Does

1. Search LinkedIn via `search_linkedin_profiles` (unchanged)
2. Score and rank candidates (unchanged)
3. Create projects via `create_sourcing_project`
4. Read state via `get_sourcing_status` (feedback + dedup)
5. Update search criteria via `update_sourcing_project`
6. Record deliveries via `record_sourcing_deliveries` (returns `delivery_id`s)
7. Post profile cards to Slack with Yes/Maybe/Pass/Details buttons (using `delivery_id`s from step 6)
8. Update `last_run_at` via `update_sourcing_project` after a batch completes

## LinkedIn Search Autopersist

Every result from `search_linkedin_profiles` is now automatically persisted to the `persons` table in Supabase with `upload_type='sourcing_linkedin'`. This happens server-side — no extra MCP call needed. Every Apify result is captured whether or not it's later delivered.

## Files That Need Updating

### `CONNECTORS.md`

**Remove Airtable from required connectors.** New required table:

| Connector | Required? | Purpose |
|-----------|-----------|---------|
| **Slack** | Yes | Channels, profile cards, feedback buttons, surface commands |
| **scheduled-tasks** | Yes | Recurring sourcing batches + weekly digests |
| **Lovelace MCP** | Yes | LinkedIn search, project CRUD, delivery/feedback state |

Remove the Airtable privacy/scope bullet. Add:
- Lovelace: all sourcing state (projects, deliveries, feedback) stored in Supabase. Each user's projects are scoped to their account. The Slack bot writes via internal API key.

### `skills/start-sourcing-project/references/mcp-prereqs.md`

- Remove Airtable from the required MCPs table
- Remove the "Airtable missing" user message
- Update the status display example to remove the Airtable line
- The Lovelace MCP entry should note it now provides project CRUD + dedup, not just LinkedIn search

### `skills/kickoff-role/SKILL.md`

These steps change:

| Old Step | New Step |
|----------|----------|
| **Prerequisites #3**: Required MCPs = Slack, Airtable, scheduled-tasks, Lovelace MCP | Required MCPs = Slack, scheduled-tasks, Lovelace MCP |
| **Prerequisites #4**: owner_email on Airtable Search record | owner is the JWT user who creates the project — Lovelace tracks `created_by` automatically |
| **Step 7.5** (duplicate check): query Airtable Searches table | Call `create_sourcing_project` with the slug — if 409, it already exists. Offer to use existing or suffix. |
| **Step 8** (create Airtable Search record): Use Airtable MCP | Call `create_sourcing_project` MCP tool. Store returned `id` as `sourcing_project_id` in config.json |
| **Step 9** (update Airtable with channel ID): Update Airtable Search record | Call `update_sourcing_project` with `slack_channel_id` and `slack_channel_name` |
| **Step 10** (config.json): Store airtable info | Store `sourcing_project_id` (the UUID) instead |
| **Step 11** (store task IDs): Store on Airtable Search record | Store in config.json (task IDs are local state, not needed in Supabase) |
| **Step 13** (summary): Airtable record ID | Show `sourcing_project_id` instead |
| **Description frontmatter**: mentions "Airtable Search record" | Replace with "Lovelace sourcing project" |

### `skills/run-sourcing-batch/SKILL.md`

These steps change:

| Old Step | New Step |
|----------|----------|
| **Description frontmatter**: "writes to Airtable" | "writes to Lovelace via Slack bot" |
| **Required MCPs**: Airtable (user's own base) | Remove. Lovelace MCP covers it. |
| **Step 1** (load state): Fetch Airtable Search record by search_slug | Call `get_sourcing_status` with `sourcing_project_id` from config.json. Check `project.status`. |
| **Step 2** (process feedback): Fetch Airtable Feedback records | Read `deliveries` from `get_sourcing_status` response — each entry has `decision` and `feedback_notes` |
| **Step 5** (dedup): Query Airtable Candidates + Served Leads + burned | Call `get_sourcing_status?scope=global` for cross-project LinkedIn URLs. Call `get_sourcing_status?scope=project` for same-project deliveries. See dedup section below. |
| **Step 6** (write to Airtable + post to Slack): Create Candidates + Served Leads records, then post cards | Claude calls `record_sourcing_deliveries` to create delivery rows (returns `delivery_id`s), then posts profile cards to Slack with buttons already attached using those IDs. |
| **Step 7** (post to Slack): Button value = {served_id, candidate_id, search_id} | Button value = `delivery_id` (UUID string). Claude embeds it in each button's `action_id` and `value`. |
| **Step 8** (update role state): Set last_run_at on Airtable | Call `update_sourcing_project` with `last_run_at` |
| **Failure modes**: "Airtable API errors" section | Replace with: if Lovelace API is down, abort. The Slack bot handles delivery recording, so Slack + Lovelace API must both be up. |
| **Non-goals**: "does not modify Airtable schema" / "does not touch other users' Airtable bases" | Replace with: does not write feedback records — the Slack bot handles that when users click buttons. |

### `skills/run-sourcing-batch/references/dedup-algorithm.md`

The dedup rules are unchanged — same logic, different data source.

Replace:

| Old | New |
|-----|-----|
| **Phase 1 — Pull state from Airtable** | **Phase 1 — Pull state from Lovelace** |
| Fetch Served Leads, Candidates, burned flags from Airtable | Call `get_sourcing_status?scope=project` for same-search deliveries. Call `get_sourcing_status?scope=global` for all LinkedIn URLs across projects. |
| `served_on_this_search` = Served Leads where search_id = S | `served_on_this_search` = deliveries from the project-scoped response |
| `served_on_other_searches` = Served Leads where search_id != S | `served_on_other_searches` = global LinkedIn URLs minus the project-scoped ones |
| `burned_candidates` = Candidates where burned = true | **Note:** The burned flag is not yet implemented in Lovelace. For now, use `served_on_other_searches` as the cross-project set. A candidate who appears in `served_on_other_searches` can be used as a cross-search repeat once (same rule), but there is no persistent `burned` column to flip yet. Track burn state in `config.json` per-project until a `burned` column is added to `sourcing_deliveries`. |
| **Phase 4 — Write state back**: create Candidates/Served Leads records, update burned flag | **Phase 4 — Record deliveries and post to Slack**: Claude calls `record_sourcing_deliveries` to create `sourcing_deliveries` rows with `was_repeat` set, then posts profile cards to Slack with buttons attached. |

### `skills/run-sourcing-batch/references/slack-formatting.md`

Update the button value format. Old:

```json
{"served_id": "...", "candidate_id": "...", "search_id": "..."}
```

New: the button `value` is just the `delivery_id` (UUID string). The Slack bot handles the rest.

Update action_id patterns:
- `sourcing_yes_{delivery_id}`
- `sourcing_maybe_{delivery_id}`
- `sourcing_pass_{delivery_id}`
- `sourcing_details_{delivery_id}`

### `plugin.json`

- Remove `airtable` from keywords
- Add `supabase`
- Update description to reference Lovelace platform instead of Airtable

## New DB Schema (for reference)

Three tables in Supabase:

### `sourcing_projects`
- id (UUID PK), created_by (FK profiles), company_name, role_title, slug (unique), use_case, slack_channel_id, slack_channel_name, search_criteria (text), status, last_run_at, created_at, updated_at

### `sourcing_deliveries`
- id (UUID PK), sourcing_project_id (FK), person_id (FK persons), delivered_by (FK profiles), score, rationale, slack_message_ts, slack_channel_id, was_repeat, created_at
- Unique constraint: (sourcing_project_id, person_id) — can't deliver same person twice to same project

### `sourcing_feedback`
- id (UUID PK), sourcing_project_id (FK), sourcing_delivery_id (FK, nullable for direction), user_id (FK profiles), decision (yes/maybe/no/direction), rejection_reason, notes, created_at
- Constraint: button feedback (yes/maybe/no) requires a delivery_id; direction feedback requires notes and no delivery_id

## Slack Bot Profile Card Format

Matches existing Lovelace lead card design:

```
*1. Jane Chen*  ·  Score *87*  ·  <https://linkedin.com/in/janechen|LinkedIn>
Staff ML Engineer  ·  DeepMind  ·  San Francisco Bay Area
> Enterprise AI, 20+ yrs eng leadership. Strong infrastructure-to-startup bridge.

[ Yes (primary) ]  [ Maybe ]  [ Pass ]  [ Details ]
```

- Yes = primary style, all others default (no danger style — Lovelace convention)
- Pass opens a modal with structured reason select + optional notes
- Maybe opens a modal with optional notes
- Details shows person snapshot

Status updates replace buttons with: `"Marked as *Yes* by <@U123>"`, `"*Maybe* — "notes" by <@U123>"`, `"*Passed* — Wrong geography by <@U123>"`

## What's NOT Changing

- SEARCH.md format and brain-update logic
- Scoring rubric and archetype-specific search patterns  
- Use-case branching in kickoff and batch skills
- Slack channel naming convention (sourcing-{search_slug})
- Schedule config and weekly digest logic
- Context gathering from optional connectors (Granola, Notion, etc.)
- The dedup rules themselves (just the data source changes)
