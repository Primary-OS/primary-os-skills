# Field Types and Operations

How to read and write each Affinity field type correctly.

## Field value types and write formats

When writing field values via `upsert_entity_field_value` or `upsert_list_entry_field_values`, the `value` format depends on the field type.

### Text

```json
{ "field_id": "field-123", "value": "Some text content" }
```

### Number

```json
{ "field_id": "field-123", "value": 5000000 }
```

### Datetime

ISO 8601 format:
```json
{ "field_id": "field-123", "value": "2026-06-15T00:00:00Z" }
```

### Location

Object with address components:
```json
{
  "field_id": "field-123",
  "value": {
    "street_address": "123 Main St",
    "city": "San Francisco",
    "state": "CA",
    "country": "US"
  }
}
```

Not all components are required. A location can be just `{ "city": "SF", "state": "CA" }`.

### Dropdown

For dropdown fields, use the option text (string):
```json
{ "field_id": "field-123", "value": "Series A" }
```

The value must exactly match one of the dropdown options (case-insensitive for the custom pipeline tools, but case-sensitive for generic field operations — always match exactly to be safe).

**Always check options first:**
```
get_list_field_dropdown_options(list_id: ..., field_id: "field-123")
```

### Ranked Dropdown

Same as dropdown — provide the option text:
```json
{ "field_id": "field-123", "value": "High Priority" }
```

### Person (reference)

Provide the person ID:
```json
{ "field_id": "field-123", "value": 456 }
```

For multi-person fields:
```json
{ "field_id": "field-123", "value": [456, 789] }
```

### Company (reference)

Provide the company ID:
```json
{ "field_id": "field-123", "value": 123 }
```

## Null and empty values

- Setting a value to `null` clears the field.
- Empty arrays `[]` are returned as `null` by the API.
- To clear a dropdown, set value to `null`.

## Reading field values

`get_entity_field_values` returns all populated field values for an entity. Each value includes:

```json
{
  "field_id": "field-123",
  "field_name": "Round",
  "value": "Series A",
  "value_type": "dropdown",
  "list_id": 12673,
  "list_entry_id": 456
}
```

**Important distinctions:**
- **Global fields** have `list_id: null` — they apply across all lists.
- **List fields** have a `list_id` and `list_entry_id` — they're scoped to a specific list entry.
- **Enriched fields** (e.g., `affinity-data-description`) are read-only. Writing to them will fail.

## Field categories at a glance

| Category | ID format | Writable | Scope |
|---|---|---|---|
| Custom global | `field-1234` | Yes | Account-wide |
| Custom list | `field-1234` | Yes | Specific list |
| Enriched | `affinity-data-*` | No (read-only) | Account-wide |
| Relationship intelligence | kebab-case (`first-email`, etc.) | No (computed) | Account-wide |

## Multi-value fields

Fields with `allows_multiple: true` accept arrays:

```json
{ "field_id": "field-123", "value": ["Tag A", "Tag B", "Tag C"] }
```

When updating multi-value fields, provide the COMPLETE array — not just the new value. The API replaces the entire array, it doesn't append.

**Wrong (overwrites existing tags):**
```json
{ "field_id": "field-123", "value": ["New Tag"] }
```

**Right (preserves existing + adds new):**
```
1. Read current values: get_entity_field_values(...)
2. Append new value to existing array
3. Write full array: { "field_id": "field-123", "value": ["Existing A", "Existing B", "New Tag"] }
```

## The `upsert` pattern

Both `upsert_entity_field_value` and `upsert_list_entry_field_values` use upsert semantics:

- If the field has no value → **creates** the value.
- If the field already has a value → **updates** the value.

This means you don't need to check whether a value exists before writing. Just write. The API handles the create-vs-update decision.

## Batch updates

`upsert_list_entry_field_values` accepts up to 100 field values per request:

```
upsert_list_entry_field_values(
  list_id: 123,
  list_entry_id: 456,
  field_values: [
    { "field_id": "field-001", "value": "Series B" },
    { "field_id": "field-002", "value": 10000000 },
    { "field_id": "field-003", "value": "2026-06-01T00:00:00Z" },
    { "field_id": "field-004", "value": 789 }  // person reference
  ]
)
```

Use batch updates when changing multiple fields on the same list entry — it's more efficient than individual calls.
