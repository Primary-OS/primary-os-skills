---
name: run-sourcing-batch
description: Runs a single sourcing batch for one role. Invoked programmatically by Cowork scheduled tasks — prompts that say "run the primary-sourcing:run-sourcing-batch skill for role slug X". Also usable manually when the user types "run sourcing for {search_slug}", "source now for {role}", "trigger a sourcing batch for this role", or similar. Processes feedback, updates the search brain, queries Apify via the Lovelace MCP, applies per-user dedup rules (never repeat on same role; at most one cross-role repeat per batch; repeats are burned), writes to Airtable, and posts profile cards to the role's Slack channel with Yes/Maybe/No buttons.
---

# Run a sourcing batch

Execute one full sourcing pipeline for a single role. Most invocations come from a scheduled task; manual invocations work the same way.

## Required arguments

- `search_slug` — the search's slug (folder name under `roles/`). Passed in the scheduled task prompt or asked interactively if missing.

## Required MCPs

- Airtable (user's own base)
- Slack
- Lovelace (for Apify LinkedIn search)

If any are missing, abort with a clear message telling the user what to connect.

## Procedure

### Step 1 — Load search state

- Read `./roles/{search_slug}/config.json` → captures `slack_channel_id`, `subject_name`, `search_title`, `use_case`, etc.
- Read `./roles/{search_slug}/SEARCH.md` → the search brain.
- Fetch the Airtable Search record by `search_slug` to confirm the search is active. If status is `paused` or `closed`, abort.
- **Capture the `use_case` field** — this determines which scoring rubric to use in step 4, which Slack phrasing to use in step 7, and which context sources to re-check in step 2.

### Step 2 — Process feedback since last run

Before sourcing new candidates, ingest feedback that accumulated since `last_run_at`:

1. Fetch Feedback records from Airtable where `search_id = {this search}` and `created_at > last_run_at`.
2. Fetch recent Slack channel messages since `last_run_at` via the Slack MCP.
3. Fetch Granola meetings mentioning the subject or search anchor in the same window. For recruiting the "subject" is the PortCo; for investment sourcing it's the thesis; etc.
4. If there are any signals, call the SEARCH.md live-update logic — decide whether the signals warrant criteria changes. If yes, rewrite `SEARCH.md` and append a Change Log entry dated today with the signal sources cited.

### Step 3 — Query Apify via Lovelace MCP

Parse SEARCH.md into structured search parameters (titles, locations, seniority, companies, industries). Then invoke the Lovelace MCP's Apify search tool:

- Shared Primary API key is server-side — no per-user key needed.
- Run multiple archetype-specific searches in parallel. LinkedIn keyword matching is AND — keep individual queries to 2-3 words max and lean on title+location filters for the rest.
- Target ~35 profiles per archetype-search so the final pool across archetypes is 70-120 raw profiles.

### Step 4 — Score and rank

Score every raw profile 1-10 against SEARCH.md. Use the rubric in `references/scoring-rubric.md` — specifically the section matching the Search's `use_case`. Each use case has its own calibration of what a 7+ means (a "put this in front of the HM" candidate for recruiting vs. a "pitch this as a portfolio fit" founder for investment sourcing, etc.).

Be harsh regardless of use case — most profiles from a broad LinkedIn search should score 1-3. Filter out scores below 6. Sort descending by score. Keep the top 25.

### Step 5 — Apply per-user dedup rules

This is the heart of the team-version logic. See `references/dedup-algorithm.md` for the full algorithm. Summary:

- **Hard exclude**: candidates already served on this role for this user. Never repeat.
- **At most one cross-role repeat per batch**: if the user has seen a candidate on a different role (and the candidate isn't burned), they can appear once more — but this burns them. `burned = true` means never serve again to this user on any role.
- **Prefer brand-new candidates**: fill the batch with people this user has never seen; only tap the eligible-repeat pool for one slot if needed.
- **Never more than one repeat per batch**: if fewer than N brand-new candidates are available, post fewer cards, don't pad with repeats.

### Step 6 — Write to Airtable

For every candidate in the final batch, write:

1. **Candidates** table: if `linkedin_url` doesn't exist for this user, create a new Candidates record. If it does, skip.
2. **Served Leads** table: always create a new record linking the Candidates record and the Search record, with `was_repeat`, `score`, `rationale`, and a placeholder `slack_message_ts` to be filled in step 7.
3. **Burned flag**: if this candidate was a cross-role repeat, flip `burned = true` on the Candidates record and set `burned_reason = "cross-role repeat on {this role title}"`.

### Step 7 — Post profile cards to Slack

Post a batch header to the role's Slack channel, then one profile card per candidate. Format per `references/slack-formatting.md`.

Each card has Yes / Maybe / No buttons whose `value` encodes `{served_id, candidate_id, search_id}` so the Slack bot can record feedback scoped to the right record.

After each post, update the corresponding Served Leads record's `slack_message_ts` with the posted message timestamp.

### Step 8 — Update Role state

- Set `last_run_at` on the Airtable Search record to the current ISO timestamp.
- If the batch was empty (nothing brand-new and no eligible repeats), post a short Slack note explaining why and suggesting broader criteria.

## Failure modes

- **Apify down or rate-limited**: abort with a clear Slack note to the channel: "Couldn't source this run — Apify returned an error. Will retry on the next schedule."
- **Airtable API errors**: don't post to Slack without writing to Airtable first. If Airtable is down, abort.
- **Slack posting partially fails**: log which cards were posted, update Served Leads for posted ones only, mark the run as partial on `last_run_at` and note in a next-run queue.

## Non-goals

- This skill does not create new Cowork scheduled tasks.
- This skill does not modify the Airtable schema.
- This skill does not touch other users' Airtable bases — scope is strictly the current user's base.
