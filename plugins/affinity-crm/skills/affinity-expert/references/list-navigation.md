# List Navigation

How to browse lists, entries, saved views, and handle pagination in Affinity.

## Discovering lists

### `get_lists`

Returns all lists accessible to the authenticated user.

Each list has:
- `id` — the list ID
- `name` — display name
- `type` — entity type: persons (0), organizations/companies (1), opportunities (8)
- `owner` — who created/owns the list
- `is_public` — whether the list is visible to all team members

**Example:**
```
get_lists()
→ [
    { id: 12673, name: "Master Deal Pipeline", type: 1 },
    { id: 13001, name: "LP Tracker", type: 0 },
    { id: 13555, name: "Board Opportunities", type: 8 }
  ]
```

### `get_list_info`

Get detailed metadata for a specific list, including field definitions and links.

```
get_list_info(list_id: 12673)
```

## Browsing list entries

### `get_list_entries`

Paginate through all entries on a list. Each entry includes the entity data and list-specific field values.

```
get_list_entries(list_id: 12673)
```

**Pagination:**
- Default page size: 100
- Max page size: 100
- Cursor-based: use `nextUrl` from the response to get the next page
- First page: no cursor needed. Subsequent pages: the MCP handles cursor passing, or use the pagination URLs.

**Include field data:** By default, entries come with their field values. This is the main way to see what's in each row of the list.

### `get_single_list_entry`

Get a specific entry by its list entry ID.

```
get_single_list_entry(list_id: 12673, list_entry_id: 456)
```

Useful when you already have the entry ID (e.g., from `search_company`'s `pipelineEntryId` or from `get_company_list_entries`).

## Finding an entity's list memberships

### `get_company_list_entries`

Get all lists that contain a specific company, with the list entry IDs.

```
get_company_list_entries(company_id: 123)
```

Returns one entry per list membership, with the list metadata and entry ID.

### `get_person_list_entries`

Same for persons:

```
get_person_list_entries(person_id: 456)
```

These are useful for understanding where an entity lives across the organization's lists.

## Working with saved views

Saved views are filtered/sorted subsets of a list that team members create and share.

### Finding saved views

```
get_list_info(list_id: 12673)
```

The list info response includes saved view metadata, or you can browse views separately. Saved views have:
- `id` — the view ID
- `name` — display name
- `type` — filter type
- `creator` — who created the view

### Viewing saved view entries

Saved views provide a filtered view of list entries. To get entries from a saved view, use the list's saved view endpoint:

```
# Browse via list entry endpoints with the saved view ID
```

This returns only the entries matching the saved view's filters, with the same entry format as `get_list_entries`.

## Field discovery

### `get_list_fields`

Get all field definitions for a specific list:

```
get_list_fields(list_id: 12673)
```

Returns field metadata:
- `id` — field ID (e.g., `field-1234`)
- `name` — display name
- `value_type` — text, number, dropdown, etc.
- `allows_multiple` — whether the field accepts multiple values
- `is_required` — whether a value is required on new entries
- `dropdown_options` — for dropdown fields, the list of valid options

### `get_entity_fields`

Get global field definitions for a specific entity type:

```
get_entity_fields(entity_type: "company")   # or "person"
```

These are account-wide fields that apply to all entities of that type, regardless of which list they're on.

### `get_list_field_dropdown_options`

Get the valid dropdown options for a specific field on a specific list:

```
get_list_field_dropdown_options(list_id: 12673, field_id: "field-456")
```

Returns an array of option objects with `id`, `text`, and optionally `rank` and `color` (for ranked dropdowns).

**Always call this before writing to a dropdown field** to ensure you're using a valid value.

## Pagination patterns

### Standard pagination

Most list endpoints use cursor-based pagination:

```
1. get_list_entries(list_id: 123)
   → results + pagination.nextUrl

2. Follow nextUrl for next page
   → more results + pagination.nextUrl (or null if done)

3. Repeat until nextUrl is null
```

### Large lists

For lists with thousands of entries:
- Stick to the default page size (100) — don't try to fetch everything at once.
- If looking for a specific entity, search first (e.g., `search_company`) rather than paginating through the whole list.
- Use saved views to narrow down to relevant subsets.

### Counting

Some endpoints support `totalCount=true` to get the total entry count:
- Useful for showing "page 1 of 15" type information.
- Adds overhead — only use when the count is needed.

## Common patterns

### "Show me everything on list X"
```
1. get_list_info(list_id: ...) → get list metadata and field definitions
2. get_list_entries(list_id: ...) → first page of entries
3. Present entries with key field values
4. Offer to show more if there are additional pages
```

### "Is X on list Y?"
```
1. search_company(term: "X") → get company_id
2. get_company_list_entries(company_id: ...) → check if list Y appears
```

### "What fields can I set on list Y?"
```
1. get_list_fields(list_id: Y)
2. Present field names, types, and whether they're required
3. For dropdown fields, offer to show the valid options
```

### "Add X to list Y with field values"
```
1. search_company(term: "X") → get entity_id
2. create_list_entry(list_id: Y, entity_id: ...)
3. get_list_fields(list_id: Y) → get field IDs
4. upsert_list_entry_field_values(list_id: Y, list_entry_id: ..., field_values: [...])
```
