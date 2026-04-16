---
name: run-weekly-summary
description: Posts a weekly sourcing digest to one search's Slack channel. Invoked programmatically by Cowork scheduled tasks (prompts that say "run the primary-sourcing:run-weekly-summary skill for role slug X"). Also usable manually when the user says "post the weekly digest for {role}", "run the weekly summary for {search_slug}", or "digest the last week for this role". Aggregates the past 7 days of deliveries and feedback, identifies yes/maybe/pass patterns, extracts recent SEARCH.md change log entries, and posts a Friday-style summary to the search's channel.
---

# Run the weekly summary for one role

Post a concise weekly recap to the search's Slack channel: counts, patterns, and an AI analysis of what's working vs. what's not.

## Required arguments

- `search_slug` — which role to run the summary for. Passed in the scheduled task prompt or asked interactively.

## Required MCPs

- Slack
- Lovelace MCP (for `get_sourcing_status`)

## Procedure

### Step 1 — Load role state

- Read `./roles/{search_slug}/config.json` and `./roles/{search_slug}/SEARCH.md`.
- Call `get_sourcing_status(search_id: sourcing_search_id)` to confirm the role is active. Skip paused/closed roles unless the user explicitly requested the summary.

### Step 2 — Fetch the past 7 days

From the `get_sourcing_status` response:

- Read the `deliveries` array — each entry has `full_name`, `score`, `decision` (yes/maybe/no/null), `feedback_notes`, `delivered_at`.
- Filter to deliveries from the past 7 days.

Count buckets from the `stats` object or compute from deliveries:

- `yes_count` — deliveries with `decision = "yes"`.
- `maybe_count` — deliveries with `decision = "maybe"`.
- `pass_count` — deliveries with `decision = "no"` (displayed as "Pass" in the UI).
- `pending_count` — deliveries with `decision = null`.
- `total` — sum of all above.

### Step 3 — Extract recent Change Log entries from SEARCH.md

Read SEARCH.md, find the `## Change Log` section, extract entries dated within the past 7 days. These are the criteria shifts that happened this week.

### Step 4 — AI pattern analysis

Only run the analysis if there are at least 3 reviewed candidates (yes + maybe + no ≥ 3). Otherwise skip — not enough signal.

Load the Search's `use_case` from `config.json`. The analysis prompt adapts slightly so phrasing reads naturally (e.g. "candidates" vs. "founders" vs. "LPs") — but the shape of the analysis is consistent.

Prompt:

```
You are helping a Primary teammate review one week of {use_case} sourcing for
{subject_name} — {search_title}. Be concise, specific, direct.

YES CANDIDATES ({yes_count}):
{formatted list: name, title @ company, feedback notes}

MAYBE CANDIDATES ({maybe_count}):
{formatted list}

PASSED CANDIDATES ({pass_count}):
{formatted list}

SEARCH CRITERIA SHIFTS THIS WEEK (from SEARCH.md change log + Slack direction):
{change log entries or "(none)"}

Provide these sections. Each bullet is 1-2 sentences max. Cite names and
concrete details, not generic observations. Skip any section with no data.
Use natural phrasing for the use case — "candidates" for recruiting,
"founders" or "companies" for investment, "prospects" for GTM, "LPs" for
fundraising, "advisors" for advisor search.

1. **Yes Trends** — What do the approved entries have in common?
2. **Maybe Themes** — What's holding back Maybes? What would tip them to Yes?
3. **Pass Patterns** — Top 2-3 reasons for rejection, with counts.
4. **Criteria Shifts** — Any changes to sourcing direction this week and why.

Do NOT include action items or suggestions — just the analysis.
```

### Step 5 — Format and post the Slack digest

Post a Block Kit message to the search's Slack channel:

```
📊 Weekly Sourcing Recap — {Month Day, Year}

*{subject_name} — {search_title}*

• *{total} profiles* surfaced this week
• ✅ {yes_count} Yes  ·  🤔 {maybe_count} Maybe  ·  ❌ {pass_count} Pass  ·  ⏳ {pending_count} Pending

{analysis section from step 4, if present}

─────

Type `surface my maybes` or `surface nos because [reason]` anytime to revisit candidates.
```

Use the analysis text as the body of a `section` block. Separate header, stats, analysis, and footer with `divider` blocks.

### Step 6 — Log the digest

Log the digest post in the search's Slack channel thread. The Slack bot captures direction-type feedback from channel messages automatically. No separate write to Supabase is needed from Claude.

## Edge cases

- **Zero activity this week**: post a short note: "No candidates this week for {role}. Either sourcing paused or criteria are too narrow — let me know if you want to adjust."
- **Fewer than 3 reviewed**: post counts only, skip the analysis section.
- **Analysis fails (API error)**: post counts + a note that the analysis couldn't run. Don't block the digest on analysis.

## Non-goals

- Don't modify SEARCH.md from this skill.
- Don't touch other roles' data.
- Don't create scheduled tasks.
