---
name: kickoff-search
description: Kicks off a new sourcing search inside the current Cowork Project. Use when the user says things like "kick off a search", "start a search for {title}", "new search for {role}", "start sourcing for {prospect / deal / founder / advisor / LP}", "kick off an investment search", or any phrasing that implies beginning one specific search inside the current Project. Handles both first-time setup (writes a tiny marker file) and subsequent searches — never touches CLAUDE.md or existing Project files. Creates the Lovelace sourcing search (which auto-creates the Slack channel), a role folder, and two recurring scheduled tasks (sourcing batch + weekly digest). Adapts every question and SEARCH.md section to the use case: recruiting, GTM, investment sourcing, LP/fund sourcing, advisor sourcing, or other.
---

# Kick off a search

Run one end-to-end kickoff for a single sourcing search inside the current Cowork Project.

This skill works in any Project — whether freshly created or one you've already built up with context. It does not overwrite your `CLAUDE.md` or rewrite Project files. It relies on the context that's already there (CLAUDE.md, attached files, connected MCP signals) and asks you directly when something is unclear.

## What this skill writes

- `.primary-sourcing-project` marker at Project root (first run only) — stores subject + default use case so future runs don't re-ask.
- `roles/{slug}/SEARCH.md` — the search brain.
- `roles/{slug}/KICKOFF.md` — audit trail of user input + raw tool findings.
- `roles/{slug}/config.json` — per-search state (use case, Lovelace search ID, Slack channel ID, scheduled task IDs).

Everything else in the Project stays untouched.

## Prerequisites

- **MCPs connected**: Slack, scheduled-tasks, Lovelace MCP (for LinkedIn search, sourcing search CRUD, and weekly digest posting). Abort clearly if any required MCP is missing.
- **Owner**: the current Cowork user. Lovelace tracks `created_by` automatically from the authenticated session.

All user-facing questions use AskUserQuestion, never plain text. Batch up to 4 per call.

## Available MCP tools (sourcing)

The Lovelace MCP provides these sourcing tools:

- `create_sourcing_search` — create a new search (returns ID, auto-creates Slack channel)
- `get_sourcing_searches` — list all searches (no args) or get full detail for one search (pass `identifier`: UUID, slug, or fuzzy name). Returns search info, criteria, stats, deliveries, directions, slack_channel_id, slack_channel_name.
- `update_sourcing_search` — update search fields
- `record_sourcing_deliveries` — record delivered profiles
- `get_sourcing_deliveries` — filtered delivery listing with pagination and rationale
- `submit_sourcing_feedback` — submit yes/maybe/no feedback on a delivery (upserts)
- `submit_sourcing_direction` — submit free-text direction to steer the search brain
- `get_sourcing_criteria` / `update_sourcing_criteria` — read/write versioned search criteria
- `get_sourcing_criteria_versions` / `set_sourcing_criteria_version` — manage criteria versions
- `post_sourcing_weekly_summary` — post a formatted weekly digest to the search's Slack channel (Claude does not need the Slack MCP for this)

## Procedure

### Step 1 — Detect Project state

Read `./.primary-sourcing-project`.

- **Missing**: first run in this Project → do Step 2.
- **Present**: carry forward `subject_name`, `subject_slug`, and `default_use_case` as defaults → skip to Step 3.

### Step 2 — First-run setup (only if marker missing)

Ask the user with a single AskUserQuestion batch:

1. **Project subject** — the anchor this whole Project is about. Examples per use case:
   - Recruiting / GTM: which portfolio company.
   - Investment: thesis or theme (e.g. "Vertical SaaS ops tools — Seed/A").
   - LP/fund: fundraise or LP ecosystem category.
   - Advisor: PortCo (if PortCo-specific) or theme.
   - Other: user's own description.
2. **Default use case** — one of: `recruiting-portco`, `gtm-sourcing-portco`, `investment-sourcing`, `fund-lp-sourcing`, `advisor-board-sourcing`, `other`. This is just a default for future kickoffs in this Project; it's always overridable per-search.

Derive `subject_slug` from the subject name (lowercased, hyphenated). Confirm with the user if non-trivial.

Write `./.primary-sourcing-project`:

```json
{
  "plugin_version": "0.2.0",
  "subject_name": "...",
  "subject_slug": "...",
  "default_use_case": "...",
  "scaffolded_at": "{ISO-8601 UTC}"
}
```

### Step 3 — Confirm the use case for this specific search

Default = marker's `default_use_case` (or the most recent role's `config.json` if newer). Confirm with a single AskUserQuestion:

> Use case for this search: **{default}**. Keep it or change?
> - Keep as {default}
> - Change to {other option} …

The chosen `use_case` goes into `roles/{slug}/config.json` for this search only. It does not mutate the marker.

### Step 4 — Collect the starting signal

Ask what's being searched for this run. Adapt phrasing to the use case. Single AskUserQuestion batch, 2–3 items:

- **Recruiting**: role title; one-paragraph role description; hiring manager or founder driving it.
- **GTM**: buyer persona title; one-paragraph persona description; sales lead or AE driving outreach.
- **Investment**: target archetype (e.g. "repeat infra-SaaS founder"); one-paragraph thesis or shape; lead investor.
- **LP/fund**: LP/fund archetype; one-paragraph mandate or relationship goal; Primary driver.
- **Advisor**: expertise or role; one-paragraph shape of the advisory engagement; who they'd report to.
- **Other**: free-text description.

If the user already provided these upfront, skip.

### Step 5 — Use existing context; ask when it's thin

Instead of synthesizing a durable project brief, rely on context that already exists:

- The Project's `CLAUDE.md` and any attached files.
- Quick parallel MCP queries relevant to the use case. Run them all in one turn:
  - **Recruiting / GTM / Advisor**: Granola (recent meetings mentioning the subject), Slack (subject or search-anchor mentions), Affinity (PortCo record), Notion (company/role docs), Gmail (threads with stakeholders).
  - **Investment**: Granola (thesis discussions, pitch meetings), Notion (thesis docs, deal memos), Slack (IC channels), Affinity (pipeline).
  - **LP/fund**: Granola, Affinity (LP records), Gmail, Notion.

If, after reading the Project and running these queries, you still don't have what the SEARCH.md needs (company stage, pipeline gaps, target geographies, dealbreakers, etc.), **ask the user directly via AskUserQuestion**. Never invent criteria. Generic SEARCH.md content is the #1 failure mode.

Summarize findings in 3–8 crisp bullets with inline source citations and invite corrections before moving on.

### Step 6 — Follow-up question batches (use-case-aware)

Two AskUserQuestion batches, 3–4 questions each. Adapt options based on what you already know from Step 5.

**Recruiting (`recruiting-portco`):**
- B1: archetype priorities; candidate paths to include/exclude; industry depth; location.
- B2: biggest pipeline gap; companies/people to avoid; sourcing cadence; weekly digest opt-in.

**GTM (`gtm-sourcing-portco`):**
- B1: target account size; buyer seniority; geography; persona variations.
- B2: existing-customer overlap to avoid double-touch; competitor taint; outreach cadence; weekly digest opt-in.

**Investment (`investment-sourcing`):**
- B1: stage; geography; traction signals to look for; technical depth required.
- B2: already-seen / passed to exclude; sector boundaries; sourcing cadence; weekly digest opt-in.

**LP/fund (`fund-lp-sourcing`):**
- B1: LP type/size; geography; mandate fit; relationship depth.
- B2: already-committed / declined to exclude; LP categories to avoid; outreach cadence; weekly digest opt-in.

**Advisor (`advisor-board-sourcing`):**
- B1: expertise area; seniority; commitment level; location flexibility.
- B2: active competitors to avoid; anti-patterns; sourcing cadence; weekly digest opt-in.

**Other (`other`):**
- Generic batch: must-have; dealbreaker; geo constraint; sourcing cadence.
- Plus 2–3 custom questions improvised from the Step 4 description.

### Step 7 — Generate SEARCH.md

Write `SEARCH.md` using `references/search-template.md` as the structure. Section names are generic across use cases; content is specific:

- **Archetypes**: for recruiting = candidate archetypes; for investment = founder/company archetypes; for LP = LP personality types; for advisor = expertise shapes; etc.
- **Dealbreakers**: universal section; content differs per use case.
- **Search Parameters**: LinkedIn-searchable fields (titles, locations, company size, industries) regardless of use case.

Always cite the source for each criterion (e.g. "per HM sync 2026-04-15", "per Notion thesis doc", "per user during kickoff").

### Step 8 — Generate the search slug + check for collision

Base slug = `{subject_slug}-{search_title_slug}`. Examples:
- Recruiting: `lighttable-founding-ae`
- Investment: `vertical-saas-ops-repeat-founder`
- LP: `fund-v-family-offices`

Before writing any files, call `create_sourcing_search` with the slug. If the response is a **409 conflict**, ask the user:

> A search already exists with slug `{slug}` in this workspace.
> - Open the existing search's role folder
> - Suffix the slug and create a new one (`-2`, `-3`, …)

If they pick existing, abort kickoff and point to `./roles/{existing_slug}/`. If new, suffix and retry until accepted.

### Step 9 — Create the Lovelace sourcing search

Call `create_sourcing_search` with:
- `company_name`: subject name
- `role_title`: search title
- `slug`: final slug from Step 8
- `use_case`: this search's use case (from Step 3)
- `search_criteria`: the SEARCH.md content from Step 7

Store the returned `id` as `sourcing_search_id`.

### Step 10 — Wait for Slack channel auto-creation

Lovelace creates the Slack channel automatically when the sourcing search is created. Poll `get_sourcing_searches(identifier: search_id)` until `slack_channel_id` appears in the detail response (usually a few seconds). The response also includes `slack_channel_name`. If still missing after 30s, warn the user and continue — the channel may still be provisioning.

### Step 11 — Write the role folder

Create `./roles/{slug}/`:

| File          | Contents                                                                            |
| ------------- | ----------------------------------------------------------------------------------- |
| `SEARCH.md`   | The synthesized brain from Step 7.                                                  |
| `KICKOFF.md`  | Snapshot of user input + all raw tool findings from Steps 4–6. Durable audit trail. |
| `config.json` | `use_case`, `sourcing_search_id`, `slack_channel_id`, `subject_name`, `subject_slug`, `search_title`, `slug`, `created_at`, task IDs (filled in Step 12). |

### Step 12 — Create two recurring scheduled tasks

Via the scheduled-tasks MCP:

- **Sourcing batch**: runs on the user's chosen cadence from Step 6.
  - Prompt: `run the primary-sourcing:run-sourcing-batch skill for search_slug "{slug}"`
- **Weekly digest**: Fridays 4pm (or user's choice). Skip if the user opted out.
  - Prompt: `run the primary-sourcing:run-weekly-summary skill for search_slug "{slug}". Post the weekly digest via the Lovelace MCP (post_sourcing_weekly_summary). Do not use the Slack MCP for this.`

Store both task IDs in `config.json`.

### Step 13 — Verify the intro message in Slack

Lovelace posts an intro message in the channel when the sourcing search is created. Read recent messages in the channel via the Slack MCP to confirm. If missing, post a short intro manually summarizing subject, search title, cadence, and how feedback works (Yes/Maybe/Pass buttons on cards; direction-style feedback in the channel updates the brain on the next run).

### Step 14 — Summarize to the user

- Final slug + Slack channel name.
- Sourcing search ID.
- Scheduled task IDs + cadence.
- Reminder: channel feedback → brain updates automatically on the next sourcing run.

## Guardrails

- Never write to or modify `CLAUDE.md` or any Project file other than the marker, `roles/`, and files listed above.
- Always run the duplicate-slug check (Step 8) before creating resources.
- If any MCP call fails mid-flow, stop and report what succeeded vs. what didn't. Prefer partial completion with clear state over silent rollback.
- Respect the per-user dedup model — this skill only **creates** the search. Dedup happens in `run-sourcing-batch`.
- If the user's answers in Steps 4–6 reveal a different use case than chosen in Step 3 (e.g. they're describing an investment deal but picked recruiting), stop and re-confirm before continuing.
