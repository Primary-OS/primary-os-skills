# Note Operations

Detailed guide for creating, searching, and managing notes in Affinity.

## Note types

Affinity has several note types:

| Type | Description |
|---|---|
| `entities` | Standard user-created notes attached to entities |
| `interaction` | Auto-generated from email/calendar interactions |
| `ai-notetaker` | AI-generated meeting notes (from Affinity's notetaker) |
| `user-reply` | Human replies to existing notes (threading) |
| `ai-notetaker-reply` | AI replies to notes |

When creating notes via the API/MCP, you create `entities` type notes. Interaction and AI notes are system-generated.

## Creating notes

### Tool: `create_note`

**Required parameters:**
- `content` — the note text. Supports plain text and HTML.
- At least one of: `person_ids`, `company_ids`, `opportunity_ids` — which entities to attach the note to.

**Optional parameters:**
- `parent_note_id` — creates a threaded reply under an existing note.

### Entity linking best practices

A single note can and should be linked to multiple entities when relevant:

```
# Good: meeting note linked to company AND attendees
create_note(
  content: "Q1 review with Acme team. Discussed expansion plans and Series B timing.",
  company_ids: [123],
  person_ids: [456, 789]    # Both attendees
)

# Bad: note only linked to company, missing the people
create_note(
  content: "Q1 review with Acme team...",
  company_ids: [123]
)
```

### HTML notes

Notes support HTML formatting for richer content:

```
create_note(
  content: "<h3>Meeting Notes - Acme Inc</h3><ul><li>Raised $5M seed</li><li>Looking for Series A in Q3</li><li>Key metric: 2x ARR growth</li></ul><p><strong>Next steps:</strong> Send term sheet by Friday</p>",
  company_ids: [123]
)
```

### Threaded notes

Create replies to existing notes:

```
create_note(
  content: "Update: term sheet sent. Awaiting response.",
  parent_note_id: 999,
  company_ids: [123]
)
```

## Finding notes

### `get_notes_for_entity` — Notes on a specific entity

The most common lookup. Returns all notes attached to a specific company, person, or opportunity.

```
get_notes_for_entity(entity_type: "company", entity_id: 123)
```

**Entity types:** `"company"`, `"person"`, `"opportunity"`.

**Pagination:** Default page size is 20. Use cursor-based pagination (`nextUrl`) for more.

### `get_notes` — Global note search

Search across all notes with filters:

- `creator_id` — notes created by a specific user
- `min_created_at` / `max_created_at` — date range
- Entity filters (person_id, company_id, opportunity_id)

```
get_notes(creator_id: 789, min_created_at: "2026-03-01")
```

### `get_entities_attached_to_note` — Reverse lookup

Given a note ID, find which entities are linked:

```
get_entities_attached_to_note(note_id: 999)
```

Returns separate arrays for companies, persons, and opportunities.

## Notes via the custom tools

The custom `get_company` tool also returns notes when `include_notes: true`:

```
get_company(company_id: 123, include_notes: true, max_notes: 10)
```

This is often more convenient than a separate `get_notes_for_entity` call because you get company details and notes in a single request. Use this when you're already looking up a company.

The custom `log_company` and `update_company_status` tools accept a `note` parameter that creates a note as part of the operation:

```
log_company(name: "Acme", status: "First Meeting", note: "Intro from Sarah, met at TechCrunch")
update_company_status(company: "Acme", status: "Active Deal", note: "Term sheet sent")
```

This is the preferred way to add context when logging or updating pipeline entries — it keeps the note creation atomic with the pipeline operation.

## Common patterns

### "What's the latest on X?"
```
1. search_company(term: "X") → get company_id
2. get_company(company_id: ..., include_notes: true, max_notes: 5)
3. Summarize the most recent notes
```

### "Log my meeting notes about X"
```
1. search_company(term: "X") → company_id
2. Optionally: search_persons(query: "attendee names") → person_ids
3. create_note(content: "...", company_ids: [...], person_ids: [...])
```

### "What has [team member] been writing about?"
```
1. get_owners(term: "Sarah") → get person_id
2. get_notes(creator_id: {person_id}, min_created_at: "2026-01-01")
```
