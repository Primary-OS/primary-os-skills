# Affinity Data Model

Understanding Affinity's entity model is essential for navigating the API correctly. This reference covers the core concepts, entity types, and how they relate.

## Core entities

### Companies (Organizations)

Companies are the primary entity in Affinity. Key properties:
- `id` — unique identifier
- `name` — company name
- `domain` / `domains` — web domains
- `personIds` — array of associated person IDs
- `global` — boolean. If true, the company comes from Affinity's proprietary database. Global companies' names and domains are immutable and they can't be deleted.

### Persons

People in Affinity. Two types:
- **External** (`type: 0`) — contacts, founders, candidates, etc.
- **Internal** (`type: 1`) — your team members with Affinity accounts.

Key properties:
- `id` — unique identifier
- `firstName`, `lastName`
- `emails`, `primaryEmail`
- `organizationIds` — current company associations
- `type` — 0 (external) or 1 (internal)

### Opportunities

Business opportunity records (deals, investments, etc.).
- `id`, `name`
- `organizationIds`, `personIds` — linked entities

## Lists and list entries

Think of a **List** as a spreadsheet. Each list has:
- A **type**: persons (0), organizations (1), or opportunities (8).
- **Fields** (columns) — both global and list-specific.
- **List entries** (rows) — each linking an entity to the list with its own field values.

The **Master Deal Pipeline** is a specific list of type 1 (organizations). It's the primary pipeline tracked by Primary's custom tools.

### List entries

A list entry is an entity's membership on a list. Key properties:
- `id` — the list entry ID (NOT the entity ID)
- `listId` — which list
- `entityId` — which company/person/opportunity
- `createdAt`, `creatorId`

**Important:** The same company can appear on multiple lists. Each appearance is a separate list entry with its own field values.

## Fields and field values

### Field categories

1. **Enriched fields** — auto-populated by Affinity or data partners (Dealroom, Mailchimp). IDs like `affinity-data-description`. Read-only.
2. **List fields** — scoped to a specific list. Managed via list-entry endpoints. IDs like `field-1234`.
3. **Global fields** — account-wide, not list-specific. IDs like `field-1234`.
4. **Relationship intelligence fields** — computed from email/calendar data. Kebab-case IDs: `first-email`, `last-email`, `first-event`, `last-event`, `next-event`, etc.

### Field value types

| Type | Description | Example |
|---|---|---|
| `text` | Free text | "Series A investor" |
| `number` | Numeric | 5000000 |
| `datetime` | ISO 8601 timestamp | "2026-01-15T00:00:00Z" |
| `location` | Object with city, state, country | `{ "city": "SF", "state": "CA", "country": "US" }` |
| `dropdown` | Single selection from predefined options | "Active Deal" |
| `ranked-dropdown` | Ordered dropdown with rank and color | `{ "text": "High", "rank": 1, "color": "#ff0000" }` |
| `person` | Reference to a person entity | Person ID |
| `company` | Reference to a company entity | Company ID |
| `filterable-text` | Searchable text | Tag values |

Most types have multi-value variants (e.g., `dropdown-multi`, `person-multi`).

### Pipeline-specific fields

The Master Deal Pipeline has these key list fields, referenced by the custom tools:
- **Status** — ranked dropdown. Values like "New", "First Meeting", "Active Deal", "Passed", etc.
- **Owners** — person type. Team members assigned to the deal.
- **Round** — dropdown. "Seed", "Series A", etc.
- **Segment** — dropdown. "B2B", "B2C", etc.
- **Priority** — dropdown. Deal priority level.

## Relationships between entities

```
Company ←→ Person (via personIds / organizationIds)
Company → List Entry → List (field values on the entry)
Person  → List Entry → List
Company ←→ Notes (via entity association)
Person  ←→ Notes
Company ←→ Meetings (via attendees' organizations)
Person  ←→ Meetings (via attendees)
Person  ←→ Person (relationship strengths between external + internal)
```

## Key concepts for the custom tools

### Pipeline enrichment

The custom `search_company` and `get_company` tools add pipeline context that the upstream tools don't:
- `onDealPipeline` — is this company on the Master Deal Pipeline?
- `pipelineEntryId` — the list entry ID if so (needed for field operations)
- `pipelineStatus` — the current status dropdown value (get_company only)

### Disambiguation

When searching by name, multiple companies may match. The custom tools handle this with a disambiguation response containing up to 10 matches with `company_id`, `name`, `domain`, and a `suggestion` for each.

### Global vs local companies

Affinity maintains a global database of companies. When `global: true`, the company came from this database and:
- Name and domain are immutable via API.
- The company can't be deleted.
- Rich enrichment data may be available.

When a company is NOT in the global database (stealth startups, very new companies), `log_company` with `create_if_not_found: true` creates a local-only company record.
