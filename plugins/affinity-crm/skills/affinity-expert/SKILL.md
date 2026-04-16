---
name: affinity-expert
description: >
  This skill MUST be loaded before calling ANY Affinity MCP tool —
  log_company, update_company_status, search_company, get_company,
  get_recent_deals, get_status_options, get_current_user, search_persons,
  create_note, get_lists, and all other Affinity tools. Contains critical
  workflow chains the tool descriptions do not convey: disambiguation,
  required sequences (get_current_user→get_recent_deals,
  get_status_options→status writes), and error recovery
  (exists_on_pipeline, not_found, invalid status). Covers requests like
  "log this company," "update deal status," "show my deals," "look up a
  company," "who is this person in our CRM," "add a note," "what lists is
  X on." Always load when working with Affinity, CRM, deal pipeline,
  companies, people, or relationships — even if tools seem self-explanatory.
---

# Affinity Expert

Single entry point for all Primary Affinity MCP operations. This skill provides the procedural knowledge that MCP tool descriptions alone do not convey — prerequisite chains, disambiguation handling, error recovery patterns, and domain-specific business logic.

## Reference files (load as needed for deep detail)

| When | File |
|------|------|
| Full parameter tables, response shapes, edge cases for the 8 custom tools | `${CLAUDE_PLUGIN_ROOT}/skills/affinity-expert/references/custom-tools.md` |
| Step-by-step pipeline workflows with every error branch | `${CLAUDE_PLUGIN_ROOT}/skills/affinity-expert/references/pipeline-workflow.md` |
| Search tool selection, semantic search, domain vs name search tips | `${CLAUDE_PLUGIN_ROOT}/skills/affinity-expert/references/search-strategies.md` |
| Affinity data model: entities, lists, fields, field values, relationships | `${CLAUDE_PLUGIN_ROOT}/skills/affinity-expert/references/data-model.md` |
| Note creation, HTML formatting, threading, entity linking patterns | `${CLAUDE_PLUGIN_ROOT}/skills/affinity-expert/references/note-operations.md` |
| Meetings, transcripts, reminders | `${CLAUDE_PLUGIN_ROOT}/skills/affinity-expert/references/meetings-and-reminders.md` |
| Field value types, write formats, dropdowns, multi-value replacement rules | `${CLAUDE_PLUGIN_ROOT}/skills/affinity-expert/references/field-types.md` |
| List browsing, entries, pagination, saved views | `${CLAUDE_PLUGIN_ROOT}/skills/affinity-expert/references/list-navigation.md` |

## Critical tool replacements

Three upstream Affinity MCP tools are **excluded** and replaced by custom versions. Never call the upstream names — they do not exist.

| Excluded upstream name | Use instead | Why |
|---|---|---|
| `search_companies` | `search_company` | Custom version adds pipeline status enrichment to every result |
| `get_company_info` | `get_company` | Custom version adds pipeline status + optional notes |
| upstream `get_current_user` | `get_current_user` (custom version, same name) | Custom version returns `user_id` as a tool response instead of an MCP resource |

---

## Procedure A — Deal Pipeline Operations

Applies when the intent is: log a company, update deal status, view recent deals, check valid statuses, find deal owners, or assign ownership.

### Identify the operation

| Intent | Tool(s) |
|---|---|
| "Log X", "Add X to pipeline", "Track X" | `log_company` |
| "Update status", "Move to X", "Mark as passed" | `update_company_status` |
| "My deals", "Recent deals", "Show pipeline" | `get_current_user` → `get_recent_deals(owner_id: user_id)` |
| "What statuses are there" | `get_status_options` |
| "Who can own deals", "Assign to X" | `get_owners` (others) or `get_current_user` (self) |

### Prerequisite chains

- **Before any status write**: Call `get_status_options` to validate the status string. Status values must match a dropdown option (case-insensitive). Guessing fails.
- **Before `get_recent_deals`**: Call `get_current_user` → extract `user_id` → pass as `owner_id`. The tool fails without it.
- **Before setting self as owner**: Call `get_current_user` → pass `user_id` as `owner_id` to `log_company`.

### Response handling (critical — this is what tool descriptions omit)

**Disambiguation**: `log_company` and `update_company_status` search by name. If multiple companies match, the tool returns a disambiguation response with up to 10 matches including `company_id`, `name`, `domain`. Present ALL matches to the user. Let them pick. Retry with the chosen `company_id`. **Never auto-select.**

**`exists_on_pipeline`**: `log_company` returns this when the company is already tracked. Report the current status to the user. If they want a change, switch to `update_company_status`. Do NOT retry `log_company`.

**`not_found`**: `log_company` returns this when the company doesn't exist in Affinity's global database (stealth startup, very new company). Confirm with the user before retrying with `create_if_not_found: true`. Include `domain` if available.

**Invalid status**: Both pipeline tools return the list of valid options when the status string doesn't match. Present the valid options. Ask the user to pick.

**Company not on pipeline**: `update_company_status` returns this when the company exists in Affinity but has no pipeline entry. Offer to use `log_company` instead.

### Reporting results

- **Log**: "Logged [Company] to the pipeline with status [Status]." Include round, segment, owner if set. Relay any round/segment warnings (non-fatal dropdown mismatches).
- **Status update**: "Updated [Company] from [Old] to [New]."
- **Recent deals**: Present deals with status, round, days in pipeline. Include the summary (status breakdown, avg days).

Consult `references/pipeline-workflow.md` for full step-by-step workflows with every error branch, and `references/custom-tools.md` for exact parameter tables.

---

## Procedure B — Entity Research

Applies when the intent is: look up a company or person, check pipeline membership, find relationship strength, or do a fuzzy search.

### Choose the search tool

| Scenario | Tool |
|---|---|
| Company by exact name or domain | `search_company` (custom — adds pipeline enrichment) |
| Company by fuzzy/natural language description | `semantic_search` (companies only, beta) |
| Person by name or email | `search_persons` (email is most precise) |
| Deep company lookup with notes | `get_company(company_id, include_pipeline_status: true, include_notes: true)` |
| Deep person lookup | `get_person_info(person_id)` |
| Relationship strength | `get_relationship_strengths(person_id)` — scores 0.0–1.0; 0.8+ strong, 0.3–0.6 moderate, <0.3 weak |
| Meeting history | `get_meetings_for_entity(entity_type, entity_id)` |
| List memberships | `get_company_list_entries` or `get_person_list_entries` |

### Key behaviors

- Domain searches are more precise than name searches. Prefer domain when available.
- `search_company` returns `onDealPipeline` per result. Treat `null` as "enrichment failed" (unknown), NOT "off pipeline".
- If company search returns zero results, suggest `semantic_search` with a natural language description.
- After search, use IDs for all subsequent calls — avoid repeated name searches.
- Company and meeting lookups can run in parallel (independent calls).
- Summarize notes rather than dumping raw content.

Consult `references/search-strategies.md` for search tips and `references/data-model.md` for entity relationships and field types.

---

## Procedure C — Notes and Activities

Applies when the intent is: create a note, find notes, view meeting history, set a reminder, or access transcripts.

### Entity resolution first

Most operations require an entity ID. Resolve by name:
- Company: `search_company(term: "X")` → `company_id`
- Person: `search_persons(query: "X")` → `person_id`

### Creating notes

`create_note` requires `content` (supports HTML) and at least one of `person_ids`, `company_ids`, or `opportunity_ids`. Attach to ALL relevant entities — both the company and people present. A meeting note should link to the company AND the attendees.

**Atomic pipeline + note**: `log_company` and `update_company_status` accept a `note` parameter. Prefer this when creating a note as part of a pipeline operation — it keeps the note creation atomic with the status change.

### Finding notes

- For a specific entity: `get_notes_for_entity(entity_type, entity_id)`
- By creator or date: `get_notes(creator_id, min_created_at)`
- Via company lookup: `get_company(company_id, include_notes: true)` — returns notes alongside company details in one call
- Reverse lookup: `get_entities_attached_to_note(note_id)` — which entities are linked

### Meetings, transcripts, reminders

- `get_meetings_for_entity(entity_type, entity_id)` — past and upcoming meetings
- `get_transcript_fragments(transcript_id)` — dialogue from recorded meetings; summarize rather than dump
- `create_reminder(entity_type, entity_id, remind_at)` — ISO 8601 datetime; requires entity link
- `get_reminders(assignee_id)` — filterable by assignee and due date

Consult `references/note-operations.md` for HTML formatting, threading, and entity linking patterns. Consult `references/meetings-and-reminders.md` for meeting prerequisites and reminder workflows.

---

## Procedure D — Lists and Fields

Applies when the intent is: browse lists, read/write field values, manage dropdowns, work with saved views, or create list entries on non-pipeline lists. For Master Deal Pipeline operations, prefer Procedure A with the custom pipeline tools.

### Navigation

- `get_lists()` → find `list_id` by name. Types: persons (0), companies (1), opportunities (8).
- `get_list_info(list_id)` → detailed metadata.
- `get_list_entries(list_id)` → paginated (max 100 per page, cursor-based).
- `get_single_list_entry(list_id, list_entry_id)` → specific entry.

### Reading fields

- `get_list_fields(list_id)` → field definitions for a list.
- `get_entity_fields(entity_type)` → global fields for companies or persons.
- `get_list_field_dropdown_options(list_id, field_id)` → valid dropdown values. **Always check before writing to a dropdown field.**
- `get_entity_field_values(entity_type, entity_id)` → all populated field values.

### Writing fields

- `upsert_entity_field_value(entity_type, entity_id, field_id, value)` — creates if absent, updates if present.
- `upsert_list_entry_field_values(list_id, list_entry_id, field_values)` — batch up to 100 per request.
- **Multi-value fields**: Provide the COMPLETE array. The API replaces, not appends. Read current values first, then write the merged array.
- **Enriched fields** (`affinity-data-*`): Read-only. Writing fails.
- `create_list_entry(list_id, entity_id)` — entity must already exist in Affinity.

Consult `references/field-types.md` for value formatting per type (text, number, datetime, location, dropdown, person reference) and `references/list-navigation.md` for pagination and saved view patterns.

---

## Guardrails (all procedures)

- Call `get_status_options` before using an uncertain pipeline status string.
- Call `get_current_user` before `get_recent_deals`; pass `user_id` as `owner_id`.
- Never auto-select from disambiguation results. Present options to the user.
- Use custom `search_company` / `get_company` — upstream `search_companies` / `get_company_info` are excluded.
- `create_if_not_found` on `log_company` creates a permanent company record. Use only with explicit user confirmation.
- Notes require at least one associated entity. Never create orphan notes.
- Always check dropdown options before writing to a dropdown field. Invalid values fail.
- Multi-value field updates replace the entire array. Read first, merge, then write.
