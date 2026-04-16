# Custom Tool Reference

Complete parameter and behavior reference for all 8 custom tools in Primary's Affinity MCP. These tools are pipeline-aware and replace the equivalent upstream tools with enriched versions.

## `log_company` — Add Company to Pipeline

Add a new company to the Master Deal Pipeline.

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `name` | string | One of `name` or `company_id` | — | Company name to search in Affinity's global database |
| `company_id` | integer | One of `name` or `company_id` | — | Affinity company ID (from `search_company`) for direct lookup |
| `domain` | string | No | — | Company domain (e.g., `newco.ai`). Helps with matching and required for `create_if_not_found` |
| `status` | string | No | `"First Meeting"` | Initial pipeline status. Must match a dropdown option (case-insensitive). Call `get_status_options` to see valid values |
| `note` | string | No | — | Note to attach (max 10000 chars). E.g., meeting context, source of introduction |
| `round` | string | No | — | Funding round: `Seed`, `Series A`, `Series B`, etc. Non-fatal warning if value doesn't match a dropdown option |
| `segment` | string | No | — | Business segment: `B2B`, `B2C`, `B2B2C`, etc. Non-fatal warning if no match |
| `owner_id` | integer | No | — | Team member ID to assign as deal owner. Get from `get_current_user` (self) or `get_owners` (others) |
| `create_if_not_found` | boolean | No | `false` | If true, creates a new company record when not found in Affinity's global database. Use for stealth startups |

**Response states:**

| State | `success` | Key field | What happened |
|---|---|---|---|
| Logged | `true` | `data.company`, `data.list_entry` | Company added to pipeline |
| Not found | `false` | `not_found: true` | Name not in Affinity's database. Retry with `create_if_not_found: true` |
| Already exists | `false` | `status: "exists_on_pipeline"` | Company already tracked. Use `update_company_status` instead |
| Disambiguation | `false` | `data.matches[]` | Multiple companies matched. Pick one by `company_id` and retry |
| Invalid status | `false` | `data.available_statuses[]` | Status string didn't match. Shows valid options |

**Notes:**
- Status is validated BEFORE any company operations. If the status is invalid, no search or creation happens.
- `round` and `segment` mismatches are non-fatal — the company is still logged, but a warning is included.
- When `create_if_not_found: true`, include `domain` for best results.

---

## `update_company_status` — Change Deal Status

Update a company's status on the Master Deal Pipeline.

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `company` | string | One of `company` or `company_id` | — | Company name or domain |
| `company_id` | integer | One of `company` or `company_id` | — | Affinity company ID |
| `status` | string | **Yes** | — | New status value (case-insensitive match against dropdown options) |
| `note` | string | No | — | Note explaining the status change (max 10000 chars) |

**Response states:**

| State | `success` | Key field | What happened |
|---|---|---|---|
| Updated | `true` | `data.previous_status`, `data.new_status` | Status changed |
| Not on pipeline | `false` | message contains "not on the Master Deal Pipeline" | Company exists in Affinity but has no pipeline entry. Use `log_company` first |
| Disambiguation | `false` | `data.matches[]` | Multiple companies matched. Pick one by `company_id` |
| Invalid status | `false` | `data.available_statuses[]` | Status string didn't match |
| Not found | `false` | message contains "not found" | No company matched the search term |

---

## `get_status_options` — List Valid Pipeline Statuses

Returns valid status values for the Master Deal Pipeline, sorted by rank (pipeline stage order).

**Parameters:** None.

**Response:** `data.options[]` — array of `{ id, text, rank }` objects sorted by rank ascending.

Example values (order may change): New, Reached Out, First Meeting, Active Deal, Passed, Portfolio, etc.

---

## `search_company` — Search with Pipeline Enrichment

Search for companies by name or domain. This is the custom version that replaces the upstream `search_companies` — it adds pipeline status enrichment to each result.

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `term` | string | **Yes** | — | Company name or domain (max 500 chars) |
| `include_pipeline_status` | boolean | No | `true` | Check whether each result is on the Master Deal Pipeline |

**Response:** `data.companies[]` — each company includes:
- `id`, `name`, `domain`, `domains`, `personIds`, `global`
- `onDealPipeline` (boolean or null if lookup failed)
- `pipelineEntryId` (number or null)

Pipeline enrichment is batched (5 concurrent lookups) and non-fatal per company — if one lookup fails, that company gets `onDealPipeline: null`.

---

## `get_company` — Get Company Details with Pipeline Context

Get detailed info about a company by ID. This is the custom version that replaces upstream `get_company_info` — it adds pipeline status and optional notes.

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `company_id` | integer | **Yes** | — | Affinity company ID |
| `include_pipeline_status` | boolean | No | `true` | Include pipeline entry status |
| `include_notes` | boolean | No | `false` | Include interaction notes |
| `max_notes` | integer | No | `10` | Max notes to return |

**Response:** `data.company` includes `onDealPipeline`, `pipelineEntryId`, `pipelineStatus`. If `include_notes: true`, also includes `data.notes[]` and `data.total_notes`.

---

## `get_owners` — Find Deal Owners

Search for internal team members who can be assigned as deal owners.

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `term` | string | No | — | Search by email, first name, or last name. Omit to list all team members |

**Response:** `data.owners[]` — array of `{ id, firstName, lastName, email }` sorted alphabetically. Only internal team members (Affinity `type=1`) are returned.

---

## `get_recent_deals` — View Pipeline Deals

Get recent deals filtered by owner. Calls the Primary OS Deals API (not Affinity directly).

**Parameters:**

| Parameter | Type | Required | Default | Description |
|---|---|---|---|---|
| `owner_id` | integer | **Yes** (runtime) | — | Affinity person ID. Call `get_current_user` first to get your `user_id` |
| `limit` | integer | No | `20` | Max deals to return (max 100) |
| `offset` | integer | No | `0` | Skip N deals for pagination |
| `include_deleted` | boolean | No | `false` | Include soft-deleted deals |

**Important:** The tool will fail if `owner_id` is missing, with a hint to call `get_current_user` first.

**Response:** `data.deals[]` — each deal has `company_name`, `company_domain`, `status`, `round`, `segment`, `priority`, `date_added`, `days_in_pipeline`. Also `data.pagination` (has_more, count) and `data.summary` (status breakdown, avg_days_in_pipeline).

---

## `get_current_user` — Get Authenticated User Info

Returns the authenticated user's ID, email, name, and organization. Call this before `get_recent_deals` or when the user wants to set themselves as deal owner via `log_company`.

**Parameters:** None.

**Response:** Flat object (not wrapped in `data`):
```json
{
  "success": true,
  "user_id": 12345,
  "first_name": "Logan",
  "last_name": "Bartlett",
  "email": "logan@primary.vc",
  "organization": { "id": 67890, "name": "Primary Venture Partners" }
}
```

**Key field:** `user_id` — pass this as `owner_id` to `log_company` and `get_recent_deals`.
