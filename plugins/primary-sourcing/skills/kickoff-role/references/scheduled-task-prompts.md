# Scheduled Task Prompts and Airtable Schema

This reference defines the exact prompt text used when creating Cowork scheduled tasks, plus the Airtable schema the plugin expects to find.

## Airtable schema (per-user base)

Each teammate connects their own Airtable base. The base should contain four tables. The plugin assumes these exact table names — if the user's base doesn't have them, provide setup instructions before proceeding.

The table is named **Searches** for generality — a "Search" is one kicked-off sourcing effort of any use case (a role for recruiting, a buyer persona for GTM, a founder archetype for investment sourcing, etc.).

### Searches
One record per kicked-off search.

| Field                | Type          | Notes                                                                                              |
| -------------------- | ------------- | -------------------------------------------------------------------------------------------------- |
| `search_id`          | Auto          | Airtable record ID.                                                                                |
| `use_case`           | Single select | `recruiting-portco` / `gtm-sourcing-portco` / `investment-sourcing` / `fund-lp-sourcing` / `advisor-board-sourcing` / `other` |
| `subject_name`       | Text          | Project subject — PortCo name, investment thesis, fundraising theme, etc.                          |
| `subject_slug`       | Text          | Hyphenated lowercase.                                                                              |
| `search_title`       | Text          | As written by the user (role title, persona, archetype, etc.).                                     |
| `search_slug`        | Text          | Hyphenated. May include Airtable record ID suffix on collision.                                    |
| `owner_email`        | Email         | Cowork user email.                                                                                 |
| `status`             | Single select | `active` / `paused` / `closed`                                                                     |
| `slack_channel_id`   | Text          | Populated after Slack channel creation.                                                            |
| `slack_channel_name` | Text          | Populated after Slack channel creation.                                                            |
| `search_md_path`     | Text          | Relative path inside Cowork Project, e.g. `roles/foo-bar/SEARCH.md`.                               |
| `schedule`           | Long text     | JSON-stringified schedule object.                                                                  |
| `sourcing_task_id`   | Text          | Cowork scheduled task ID.                                                                          |
| `weekly_task_id`     | Text          | Cowork scheduled task ID.                                                                          |
| `last_run_at`        | Date/Time     | Updated after each sourcing batch.                                                                 |
| `created_at`         | Date/Time     | ISO-8601 UTC.                                                                                      |

### Candidates
One record per LinkedIn profile ever sourced for this user.

| Field               | Type       | Notes                                                               |
| ------------------- | ---------- | ------------------------------------------------------------------- |
| `candidate_id`      | Auto       | Airtable record ID.                                                 |
| `linkedin_url`      | URL        | Normalized — strip trailing slash, lowercase.                       |
| `name`              | Text       |                                                                     |
| `headline`          | Text       | LinkedIn headline at time of first sourcing.                        |
| `current_title`     | Text       |                                                                     |
| `current_company`   | Text       |                                                                     |
| `location`          | Text       |                                                                     |
| `first_sourced_at`  | Date/Time  |                                                                     |
| `burned`            | Checkbox   | True = never serve again to this user on any role.                  |
| `burned_reason`     | Text       | Populated when burned flips to true.                                |

### Served Leads
Junction table — which candidates went to which searches for this user.

| Field             | Type      | Notes                                                             |
| ----------------- | --------- | ----------------------------------------------------------------- |
| `served_id`       | Auto      |                                                                   |
| `candidate_id`    | Link      | → Candidates                                                      |
| `search_id`       | Link      | → Searches                                                        |
| `served_at`       | Date/Time |                                                                   |
| `was_repeat`      | Checkbox  | True if this was a cross-search repeat.                           |
| `slack_message_ts`| Text      | Slack message timestamp (for follow-up feedback linking).         |
| `feedback_status` | Single select | null / `yes` / `maybe` / `no`                                 |
| `score`           | Number    | 1-10 AI score at time of serving.                                 |
| `rationale`       | Long text | AI scoring rationale.                                             |

### Feedback
Per-event log.

| Field            | Type          | Notes                                                              |
| ---------------- | ------------- | ------------------------------------------------------------------ |
| `feedback_id`    | Auto          |                                                                    |
| `served_id`      | Link          | → Served Leads (null if general channel message)                   |
| `search_id`      | Link          | → Searches                                                         |
| `type`           | Single select | `button_click` / `channel_message`                                 |
| `value`          | Long text     | yes/maybe/no or raw message text.                                  |
| `user_email`     | Email         | Who sent the feedback.                                             |
| `created_at`     | Date/Time     |                                                                    |

## Scheduled task prompts

Cowork scheduled tasks run a natural language prompt. Keep prompts concise — the skills themselves hold the actual logic. The prompt is identical regardless of use case; the skill reads the use case from the Search record's `use_case` field and adapts.

### Sourcing task prompt

When creating the sourcing task, use this prompt (substitute the bracketed values):

```
Run the primary-sourcing:run-sourcing-batch skill for search slug [SEARCH_SLUG].
The search folder lives in this Cowork Project at roles/[SEARCH_SLUG]/.
Use connected MCPs: Airtable (for dedup state and writes), Slack (for posting cards), Lovelace (for Apify LinkedIn search).
```

### Weekly summary task prompt

```
Run the primary-sourcing:run-weekly-summary skill for search slug [SEARCH_SLUG].
Post the weekly digest to the search's Slack channel using the schedule in Airtable.
```

## Cadence options to offer

When asking the user about sourcing cadence, offer these defaults via AskUserQuestion:

- **"Mon/Wed/Fri 9:30am ET"** — highest-volume option.
- **"Tue/Thu 9:30am ET"** — the current default Annabelle uses.
- **"Weekly, Monday 9am ET"** — for slower-moving searches.
- **"Manual only — I'll trigger runs myself"** — skips the scheduled task entirely.

For the weekly summary, offer Friday 4pm ET or skip.
