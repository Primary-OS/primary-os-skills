# {{SUBJECT_NAME}} — Primary Sourcing Project

This Cowork Project is a **{{USE_CASE}}** sourcing workspace for **{{SUBJECT_NAME}}**. It was scaffolded by the `primary-sourcing` plugin.

Every teammate with access to this Project can kick off new searches inside it. Individual searches are each owned by the teammate who kicked them off — scheduled tasks run on their account, and their Airtable base holds the dedup history.

## Project structure

```
{{SUBJECT_SLUG}}/
├── CLAUDE.md                        # this file — Project instructions
├── .primary-sourcing-project        # scaffold marker (use case, subject, version)
├── project-context/
│   └── project-brief.md             # shared context about {{SUBJECT_NAME}}
├── templates/
│   └── SEARCH-TEMPLATE.md           # starting template for each search's brain
└── roles/
    └── {search-slug}/               # one folder per kicked-off search
        ├── SEARCH.md                # the evolving search brain
        ├── KICKOFF.md               # kickoff context snapshot
        └── config.json              # channel, Airtable IDs, schedule, use case
```

## How to interact with this Project

### Kick off a new search

Ask the agent something like:

- "Kick off a new role for this Project"
- "Start a new search for {{SUBJECT_NAME}}"
- "New search: {title}"

The agent will invoke the `primary-sourcing:kickoff-role` skill and walk through:

1. Gathering context from connected tools (Granola, Slack, Notion, Drive, Gmail, Affinity).
2. Asking targeted follow-up questions with AskUserQuestion.
3. Generating the SEARCH.md brain.
4. Creating a private Slack channel (`sourcing-{slug}`).
5. Creating an Airtable Search record in your base.
6. Writing the `roles/{slug}/` folder.
7. Creating recurring Cowork scheduled tasks for sourcing batches and weekly digests.

### Edit an existing search's brain

Open `roles/{slug}/SEARCH.md` in Cowork and edit directly, or drop a message in the search's Slack channel — the bot will update the brain in real time.

### Trigger a manual sourcing run

Say "run sourcing now for {search_slug}" — the `primary-sourcing:run-sourcing-batch` skill handles it. Otherwise batches run on the schedule configured at kickoff.

## Important behaviors

- **Default behaviors come from the `primary-sourcing` plugin.** Follow its skill instructions whenever the user's request touches sourcing, searches, candidates, prospects, founders, LPs, or advisors inside this Project.
- **Read `.primary-sourcing-project` first** whenever starting a Project-level task. It tells you the use case and subject; skill prompts adapt accordingly.
- **Never create files outside this Cowork Project workspace.** All sourcing state lives inside the Project (for shared context) and in the current teammate's Airtable base (for per-user dedup history).
- **Role ownership is per teammate.** A search created by user A belongs to user A's scheduled tasks and Airtable. Other teammates can read the Project and see the searches but shouldn't edit another user's search folder without coordination.
- **Respect the dedup rules.** A candidate served once is never shown again on the same search; a candidate can appear at most once more on a different search before being burned. See the plugin's dedup algorithm reference.

## Use case: {{USE_CASE}}

The language in Slack messages, AskUserQuestion prompts, and SEARCH.md generation is tuned to this use case. If the use case needs to change, re-run the `primary-sourcing:start-sourcing-project` skill — don't manually edit the marker file.

See the plugin's `skills/kickoff-role/references/use-cases.md` for the full definition of each use case.

## Connectors required for sourcing to actually work

- **Slack** — per-search channels + feedback.
- **Airtable** — per-user base with Searches, Candidates, Served Leads, Feedback tables.
- **scheduled-tasks** — Cowork-native, should be always available.
- **Lovelace MCP** — shared Apify LinkedIn search for Primary.

Optional but recommended: Granola, Notion, Google Drive, Gmail, Affinity — used during kickoff to build a grounded SEARCH.md.

If any required connector is missing, `kickoff-role` will tell you what to connect before it proceeds.

## Where to report issues

File a Linear issue in the "Team Recruiting Sourcing Agent" project (even if your use case isn't recruiting — that Linear project tracks the plugin itself), or ping `#ai-tools` in Slack.
