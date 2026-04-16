---
name: kickoff-role
description: Kicks off a new search inside an already-scaffolded Primary sourcing Cowork Project, regardless of use case (recruiting, GTM, investment sourcing, LP/fund sourcing, advisor sourcing, etc.). Use when the user says things like "kick off a role", "kick off a new search", "start a search for {title}", "new search for {role}", "start sourcing for {prospect / deal / founder / advisor}", "kick off an investment search", "kick off an LP search", or any phrasing that implies beginning one specific search inside the current Project. Reads the Project's use case from the scaffold marker and adapts every follow-up question, SEARCH.md section, and downstream scheduled task to that use case. Creates the Slack channel, Airtable Search record, role folder, and two recurring scheduled tasks (sourcing batch + weekly digest).
---

# Kick off a new search

Run the full kickoff flow for a single search inside a Primary sourcing Cowork Project. At the end of this skill, the search has a Slack channel, an Airtable Search record, a role folder under `roles/{search_slug}/`, and two Cowork scheduled tasks (sourcing + weekly digest).

The skill adapts to the Project's use case — recruiting, GTM sourcing, investment sourcing, LP/fund sourcing, advisor sourcing, or a custom "other" case. Terminology, follow-up questions, and SEARCH.md generation all branch based on what the Project was scaffolded for.

## Read before running

- `${CLAUDE_PLUGIN_ROOT}/skills/kickoff-role/references/use-cases.md` — exactly how each use case adapts.
- `${CLAUDE_PLUGIN_ROOT}/skills/start-sourcing-project/references/context-intake.md` — the intake principles apply here too.

## Prerequisites

1. **Project is scaffolded.** Read `./.primary-sourcing-project`. If missing or malformed, abort and offer to run `start-sourcing-project` first.
2. **Capture scaffold metadata** from the marker: `use_case`, `subject_name`, `subject_slug`.
3. **Required MCPs connected**: Slack, Airtable (user's own base), scheduled-tasks, Lovelace MCP (for Apify search). See `${CLAUDE_PLUGIN_ROOT}/skills/start-sourcing-project/references/mcp-prereqs.md`. Abort clearly if any required MCP is missing.
4. **Identify the search owner**: the current Cowork user. Their email is the `owner_email` on the Airtable Search record.

## Procedure

All user-facing questions use AskUserQuestion, never plain text. Batch up to 4 questions per call. See `references/context-intake.md` for the intake principles — they are not optional.

### Step 1 — Confirm use case and subject carried forward

Quickly confirm with the user (single AskUserQuestion):

> This Project is set up for **{use_case}** sourcing for **{subject_name}**.
> Is this new search for the same subject, or has something changed?
>
> - Yes, same subject
> - New subject (abort and re-scaffold)
> - Same subject but use case is different (rare — confirm with care)

Most of the time the user says yes and you move on.

### Step 2 — Collect the starting signal

What's being searched for in this run? The question adapts per use case. See `references/use-cases.md` for phrasing examples.

AskUserQuestion batch — 2 to 3 questions:

- **Recruiting**: role title; one-paragraph role description; who's the hiring manager or founder driving this.
- **GTM sourcing**: buyer persona title; one-paragraph description of the buyer; who's the sales lead or AE driving the outreach.
- **Investment sourcing**: target archetype (e.g. "repeat infra-SaaS founder", "Series A proptech company"); one-paragraph thesis or shape; which investor is leading the search.
- **Fund/LP sourcing**: LP/fund archetype; one-paragraph context on the mandate or relationship goal; who at Primary is driving.
- **Advisor/board**: expertise or role; one-paragraph shape of what the advisor would do; who the advisor reports to (PortCo founder, Primary partner).
- **Other**: free-text — let the user describe.

If the user provided these upfront in their initial message, skip and move on.

### Step 3 — Gather context in parallel

For each relevant MCP, query with the subject + search anchor. Run queries **in parallel** in a single turn. See `references/context-gathering.md` for the full playbook and `references/use-cases.md` for which sources matter most per use case.

Also re-read `./project-context/project-brief.md` — that's the Project-level context captured at scaffold time.

### Step 4 — Synthesize and confirm

Summarize findings in 3-8 crisp bullets with inline source citations. Invite the user to correct anything wrong and add anything missing. This is a confirmation step — proceed only after the user confirms.

### Step 5 — Follow-up question batches (two batches, use-case-aware)

Ask with AskUserQuestion, four questions per call. Question sets differ per use case — see below. Adapt options based on what you already know from earlier context.

Read `references/use-cases.md § How skills adapt per use case § kickoff-role` for use-case-specific question sets. Summary:

**Recruiting (`recruiting-portco`):**
- Batch 1: archetype priority, candidate paths to include/exclude, industry depth, location.
- Batch 2: biggest pipeline gap, companies/people to avoid, sourcing cadence, weekly digest opt-in.

**GTM sourcing (`gtm-sourcing-portco`):**
- Batch 1: target account size, buyer seniority, geography, buyer persona variations.
- Batch 2: existing customer overlap (avoid double-touch), competitor taint, outreach cadence, weekly digest opt-in.

**Investment sourcing (`investment-sourcing`):**
- Batch 1: stage, geography, traction signals to look for, technical depth required.
- Batch 2: already-seen / passed companies to exclude, sector boundaries, sourcing cadence, weekly digest opt-in.

**Fund/LP sourcing (`fund-lp-sourcing`):**
- Batch 1: LP type/size, geography, mandate fit criteria, relationship depth needed.
- Batch 2: already-committed or already-declined to exclude, LP categories to avoid, outreach cadence, weekly digest opt-in.

**Advisor/board (`advisor-board-sourcing`):**
- Batch 1: expertise area, seniority, commitment level expected, location flexibility.
- Batch 2: active competitors to avoid, anti-patterns (wrong fit shapes), sourcing cadence, weekly digest opt-in.

**Other (`other`):**
- Generic batch: what's a must-have, what's a dealbreaker, any location/geo constraint, sourcing cadence.
- Ask two or three more use-case-specific questions based on the user's description in step 2.

### Step 6 — Generate SEARCH.md

Write the initial `SEARCH.md` using `${CLAUDE_PLUGIN_ROOT}/templates/role/SEARCH-TEMPLATE.md` as the structure. See `references/search-md-structure.md` for how to fill each section per use case.

Content is use-case-aware even though section names are generic:

- "Archetypes" for recruiting = candidate archetypes; for investment = founder/company archetypes; for LP sourcing = LP personality types.
- "Dealbreakers" apply universally but the content differs.
- "Search Parameters" in all cases map to LinkedIn-searchable fields (titles, locations, company size, industries).

Be **specific**. Generic SEARCH.md content is the #1 failure mode. Always cite the source for each criterion (e.g. "per HM sync 2026-03-28" or "per thesis doc in Notion").

### Step 7 — Generate the search slug and resolve collisions

Base slug = `{subject_slug}-{search_slug}`. See `references/collision-handling.md` for the algorithm. For recruiting this might look like `lighttable-founding-ae`; for investment sourcing `vertical-saas-ops-repeat-founder`; for LP sourcing `fund-v-family-offices`.

### Step 8 — Create the Airtable Search record

Use the Airtable MCP to create a record in the user's Searches table (note: table is named "Searches" for generality, but maps 1:1 to what the legacy system called "Roles"). Fields per `references/scheduled-task-prompts.md`.

Capture the returned Airtable record ID and suffix the slug if there was a collision.

### Step 9 — Create the private Slack channel

Channel name: `sourcing-{search_slug}`. Invite the search owner. Handle name-taken collisions with `-v2` etc. Store channel ID on the Airtable Search record.

### Step 10 — Write the role folder

Create `./roles/{search_slug}/` with:

| File              | Contents                                                                                                   |
| ----------------- | ---------------------------------------------------------------------------------------------------------- |
| `SEARCH.md`       | The synthesized brain from step 6.                                                                         |
| `KICKOFF.md`      | Snapshot of user input + all raw tool findings — durable audit trail.                                      |
| `config.json`     | Filled from `${CLAUDE_PLUGIN_ROOT}/templates/role/config-template.json` with final search data + use case. |

### Step 11 — Create recurring Cowork scheduled tasks

Two tasks using the scheduled-tasks MCP. Prompt templates in `references/scheduled-task-prompts.md`. The prompts are identical across use cases — the logic branches inside `run-sourcing-batch` based on the Search record's stored use case.

- **Sourcing task**: runs on the user's chosen schedule.
- **Weekly digest task**: runs Fridays at 4pm (or user's choice). Skip if user opted out.

Store task IDs on the Airtable Search record.

### Step 12 — Post the intro message to Slack

Post a welcome message. Content adapts slightly per use case (phrasing like "I'll source candidates" vs. "I'll surface founders" vs. "I'll surface LPs"). See `references/intro-messages.md` for templates per use case.

Core elements in every intro:

- What the search is about (subject + title).
- How to interact (buttons, channel messages, `surface my maybes`).
- Schedule summary.
- Owner tag.

### Step 13 — Summarize to the user

Cowork-side summary:

- Search slug and Slack channel name.
- Airtable record ID.
- Scheduled task IDs + cadence.
- Reminder: feedback in Slack → brain updates automatically.

## Guardrails

- Never create duplicate Searches — check by `(owner_email, search_slug)` first.
- If any MCP call fails mid-flow, stop and report what succeeded vs. what didn't. Prefer partial completion with clear state over silent rollback.
- Do not kick off inside a workspace that isn't a scaffolded Project.
- Respect the per-user dedup model. This skill only **creates** the search — dedup happens in `run-sourcing-batch`.
- If the user's answers reveal the use case was miscategorized (e.g. they're describing an investment deal but the Project was scaffolded as recruiting), stop and confirm before continuing. Mis-scaffolded Projects should be re-scaffolded, not patched.
