# Meetings, Transcripts, and Reminders

## Meetings

Affinity tracks meetings (calendar events) and their attendees automatically via email/calendar integration.

### `get_meetings` — All meetings

Paginate through all meetings visible to the authenticated user.

**Filters available:**
- Date range (min/max datetime)
- Participant filters

**What it returns per meeting:**
- Meeting ID, subject/title
- Start and end time
- Attendees (person IDs, names, emails)
- Whether it's a past or future meeting

### `get_meetings_for_entity` — Meetings with a specific entity

Get all meetings involving a specific company, person, or opportunity.

```
get_meetings_for_entity(entity_type: "company", entity_id: 123)
get_meetings_for_entity(entity_type: "person", entity_id: 456)
get_meetings_for_entity(entity_type: "opportunity", entity_id: 789)
```

**Prerequisite:** The "unified events" feature must be enabled in the user's Affinity account. If meetings return empty but the user knows they've had meetings, this feature may not be onboarded.

### Common meeting queries

**"When did we last meet with X?"**
```
1. Resolve entity (search_company or search_persons)
2. get_meetings_for_entity(entity_type: ..., entity_id: ...)
3. Sort by date, present the most recent one
```

**"Do we have upcoming meetings with X?"**
```
1. Resolve entity
2. get_meetings_for_entity(entity_type: ..., entity_id: ...)
3. Filter for future dates
```

**"Who attended the meeting with X?"**
```
1. Resolve entity
2. get_meetings_for_entity → meeting data includes attendee details
```

## Transcripts

Affinity can capture meeting transcripts (from supported recording tools). Transcripts are broken into **fragments** — individual dialogue segments with speaker attribution.

### `get_transcript_fragments`

Get the dialogue fragments from a meeting transcript.

```
get_transcript_fragments(transcript_id: 555)
```

**Returns:**
- Speaker name/identifier
- Dialogue text
- Timestamp within the meeting

**Finding transcript IDs:** Transcripts are associated with meetings. You may need to browse transcripts via the Affinity v2 API endpoints (`/v2/transcripts`). This is not directly exposed as a separate MCP tool — use the general list browsing approach or check meeting data for transcript references.

**Tips:**
- Transcripts can be very long. Summarize rather than displaying raw fragments.
- When the user asks "what was discussed," extract the key topics, decisions, and action items.
- Speaker attribution lets you identify who said what — useful for "what did [person] say about..."

## Reminders

Reminders are follow-up tasks tied to Affinity entities. They can be one-time or recurring.

### `create_reminder`

```
create_reminder(
  entity_type: "company",      # or "person", "opportunity"
  entity_id: 123,
  remind_at: "2026-05-01T10:00:00Z",
  note: "Follow up on term sheet"
)
```

**Required:**
- `entity_type` and `entity_id` — which entity the reminder is about
- `remind_at` — when to trigger (ISO 8601 format)

**Optional:**
- `note` — descriptive text
- `list_entry_id` — scope the reminder to a specific list entry
- Recurring options may be available depending on MCP version

### `get_reminders`

List reminders visible to the authenticated user.

```
get_reminders()
get_reminders(assignee_id: 789)           # By assignee
get_reminders(min_due_date: "2026-04-01") # By due date
```

**Returns per reminder:**
- Reminder ID
- Entity info (type, ID)
- Owner/assignee
- Due date (`remind_at`)
- Note text
- Created/updated timestamps

### Common reminder patterns

**"Remind me to follow up with X next week"**
```
1. Resolve entity (search_company or search_persons)
2. Calculate next week's date
3. create_reminder(
     entity_type: "company",
     entity_id: ...,
     remind_at: "2026-04-23T09:00:00Z",
     note: "Follow up on partnership discussion"
   )
```

**"What reminders do I have?"**
```
1. get_current_user() → user_id
2. get_reminders(assignee_id: {user_id})
3. Sort by due date, present upcoming ones first
```

**"Any overdue follow-ups?"**
```
1. get_current_user() → user_id
2. get_reminders(assignee_id: {user_id}, max_due_date: "{today}")
3. Present any with due dates in the past
```
