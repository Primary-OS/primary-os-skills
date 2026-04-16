# Affinity MCP (Cloudflare Worker) — implementation spec

The MCP server is **not** bundled with this plugin. Apply the following in the Worker repo so agents get workflow guidance even before the `affinity-expert` skill loads, and so tool descriptions reinforce correct sequences.

## 1. Expand `serverInstructions` (or equivalent) on initialize

Replace a short one-liner with:

```
Affinity CRM integration for deal pipeline management.

Required workflow chains:
- ALWAYS call get_status_options before using status values in log_company or update_company_status when the status is not already known to be valid.
- ALWAYS call get_current_user first to obtain owner_id (user_id) before calling get_recent_deals.
- When search_company or log_company returns multiple matches (disambiguation), present ALL options to the user and retry with the chosen company_id — never auto-select.
- When log_company returns exists_on_pipeline, switch to update_company_status instead of retrying log_company.
- When log_company returns not_found, confirm with the user before retrying with create_if_not_found=true.

If the affinity-crm plugin is available, load it for comprehensive workflow guidance and edge case handling.
```

## 2. Append workflow hints to tool descriptions

### `log_company`

Append to the description:

```
IMPORTANT: Prefer search_company first when unsure of identity to reduce disambiguation errors. If the response is exists_on_pipeline, switch to update_company_status. If not_found, confirm with the user before setting create_if_not_found=true. Validate status via get_status_options when uncertain.
```

### `update_company_status`

Append:

```
IMPORTANT: Validate status via get_status_options when uncertain. If multiple companies match, present options and retry with company_id. If the company is not on the pipeline, use log_company instead.
```

### `get_recent_deals`

Append:

```
IMPORTANT: Requires owner_id — call get_current_user first, then pass user_id as owner_id.
```

### `search_company`

Append:

```
NOTE: Results include onDealPipeline for each company. A value of null means the pipeline status lookup failed (unknown) — do NOT treat null as "not on pipeline." Use company_id from results for subsequent calls instead of repeating name searches.
```

### `get_current_user`

Append:

```
NOTE: The user_id returned here is needed as owner_id for get_recent_deals and log_company. Call this tool first when the user wants to see their deals or set themselves as deal owner.
```

## 3. Optional: `get_affinity_help` meta-tool

Add a tool that returns short workflow text so confused agents can self-serve without loading the plugin skill first.

**Name:** `get_affinity_help`

**Description:**

```
Get workflow guidance for Affinity deal pipeline and CRM operations. Call when unsure about tool order, parameters, disambiguation, status values, or error responses. Returns concise steps; use the affinity-crm plugin skill for full reference docs.
```

**Input schema:**

```json
{
  "type": "object",
  "properties": {
    "topic": {
      "type": "string",
      "description": "What you need help with. Examples: log a company, update status, search, disambiguation, recent deals, notes, fields, lists"
    }
  },
  "required": ["topic"]
}
```

**Behavior:** Map `topic` (keyword match) to canned markdown or JSON snippets covering the same chains as section 1, plus pointers to disambiguation and `exists_on_pipeline` / `not_found` handling.
