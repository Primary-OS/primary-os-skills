---
name: run-sourcing-batch
description: Runs a single sourcing batch for one role. Invoked programmatically by Cowork scheduled tasks — prompts that say "run the primary-sourcing:run-sourcing-batch skill for role slug X". Also usable manually when the user types "run sourcing for {search_slug}", "source now for {role}", "trigger a sourcing batch for this role", or similar. Processes feedback, updates the search brain, queries Apify via the Lovelace MCP, applies per-user dedup rules (never repeat on same role; at most one cross-role repeat per batch; repeats are burned), records deliveries via `record_sourcing_deliveries`, and posts profile cards with Yes/Maybe/Pass/Details buttons to the role's Slack channel. The Slack bot handles feedback writes when users click buttons.
---

# Run a sourcing batch

Execute one full sourcing pipeline for a single role. Most invocations come from a scheduled task; manual invocations work the same way.

## Required arguments

- `search_slug` — the search's slug (folder name under `roles/`). Passed in the scheduled task prompt or asked interactively if missing.

## Required MCPs

- Slack
- Lovelace MCP (for LinkedIn search + sourcing state via `get_sourcing_status`)

If any are missing, abort with a clear message telling the user what to connect.

## Procedure

### Step 1 — Load search state

- Read `./roles/{search_slug}/config.json` → captures `slack_channel_id`, `subject_name`, `search_title`, `use_case`, `sourcing_search_id`, etc.
- Read `./roles/{search_slug}/SEARCH.md` → the search brain.
- Call `get_sourcing_status(search_id: sourcing_search_id)` to confirm the search is active. If `search.status` is `paused` or `closed`, abort.
- **Capture the `use_case` field** — this determines which scoring rubric to use in step 4, which Slack phrasing to use in step 7, and which context sources to re-check in step 2.

### Step 2 — Process feedback since last run

Before sourcing new candidates, ingest feedback that accumulated since `last_run_at`:

1. Read `deliveries` from the `get_sourcing_status` response (step 1) — each entry has `decision` (yes/maybe/no/null) and `feedback_notes`.
2. Call `get_sourcing_criteria(search_id)` to read the current criteria version from Lovelace.
3. Fetch recent Slack channel messages since `last_run_at` via the Slack MCP.
4. Fetch Granola meetings mentioning the subject or search anchor in the same window. For recruiting the "subject" is the PortCo; for investment sourcing it's the thesis; etc.
5. If there are any signals, decide whether the signals warrant criteria changes. If yes, rewrite `SEARCH.md`, append a Change Log entry dated today with the signal sources cited, and call `update_sourcing_criteria(search_id, content)` to persist the new version to Lovelace. This auto-increments the version number.

### Step 3 — Query Apify via Lovelace MCP

Parse SEARCH.md into structured search parameters (titles, locations, seniority, companies, industries). Then invoke the Lovelace MCP's Apify search tool:

- Shared Primary API key is server-side — no per-user key needed.
- Run multiple archetype-specific searches in parallel. LinkedIn keyword matching is AND — keep individual queries to 2-3 words max and lean on title+location filters for the rest.
- Target max 10 profiles per archetype-search so the final pool across archetypes is ~30 raw profiles.

### Step 4 — Score and rank

Score every raw profile 1-10 against SEARCH.md. Use the rubric in `references/scoring-rubric.md` — specifically the section matching the Search's `use_case`. Each use case has its own calibration of what a 7+ means (a "put this in front of the HM" candidate for recruiting vs. a "pitch this as a portfolio fit" founder for investment sourcing, etc.).

Be harsh regardless of use case — most profiles from a broad LinkedIn search should score 1-3. Filter out scores below 6. Sort descending by score. Keep the top 25.

### Step 5 — Apply per-user dedup rules

This is the heart of the team-version logic. See `references/dedup-algorithm.md` for the full algorithm. Summary:

- Call `get_sourcing_status(search_id, scope: "search")` for same-search deliveries.
- Call `get_sourcing_status(search_id, scope: "global")` for all LinkedIn URLs across the user's searches.
- **Hard exclude**: candidates already delivered on this search. Never repeat.
- **At most one cross-search repeat per batch**: if the user has seen a candidate on a different search (and the candidate isn't burned), they can appear once more.
- **Prefer brand-new candidates**: fill the batch with people this user has never seen; only tap the eligible-repeat pool for one slot if needed.
- **Never more than one repeat per batch**: if fewer than N brand-new candidates are available, post fewer cards, don't pad with repeats.

### Step 6 — Record deliveries and post profile cards to Slack

1. Call `record_sourcing_deliveries` via the Lovelace MCP with the final batch (person_id, score, rationale per candidate). This creates `sourcing_deliveries` rows in Supabase and returns the rows including `delivery_id`s.
2. Post a batch header to the role's Slack channel, then one profile card per candidate with Yes/Maybe/Pass/Details buttons already attached. Format per `references/slack-formatting.md`. Each button's `value` encodes the `delivery_id`.

### Step 7 — Update search state

- Call `update_sourcing_search(search_id, last_run_at: now)` to set the current ISO timestamp.
- If the batch was empty (nothing brand-new and no eligible repeats), post a short Slack note explaining why and suggesting broader criteria.

## Failure modes

- **Apify down or rate-limited**: abort with a clear Slack note to the channel: "Couldn't source this run — Apify returned an error. Will retry on the next schedule."
- **Lovelace API down**: abort. Delivery recording requires the API.
- **Slack posting partially fails**: log which cards were posted, mark the run as partial on `last_run_at` and note in a next-run queue.

## Non-goals

- This skill does not create new Cowork scheduled tasks.
- This skill does not write feedback records — the Slack bot handles that when users click buttons.
