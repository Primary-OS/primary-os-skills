# Connectors

The `affinity-crm` plugin requires a single MCP connector: the Primary Affinity MCP server. This server provides 8 custom deal-pipeline tools and proxies ~24 official Affinity MCP tools through a single connection.

The plugin itself does not bundle or ship an MCP server. The Affinity MCP is deployed as a Cloudflare Worker and connected through Cowork, Claude Desktop, or Claude Code settings.

## Required

| Connector              | Purpose                                                                                                                                                          |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Affinity MCP**       | All Affinity operations: pipeline management, company/person search, notes, lists, fields, meetings, relationship intelligence. Single connection covers everything. |

### How the MCP works

Primary's Affinity MCP is a Cloudflare Worker that:

1. **Exposes 8 custom tools** optimized for Primary's deal pipeline workflow — `log_company`, `update_company_status`, `search_company`, `get_company`, `get_status_options`, `get_owners`, `get_recent_deals`, `get_current_user`.
2. **Proxies ~24 official Affinity MCP tools** from the upstream server — person search, notes, lists, fields, meetings, relationship strengths, reminders, transcripts, and more.
3. **Excludes 3 upstream tools** that the custom versions replace with pipeline-enriched alternatives: `search_companies` → `search_company`, `get_company_info` → `get_company`, upstream `get_current_user` → local `get_current_user`.

Authentication is via OAuth — the user enters their Affinity API key once and the MCP handles both v1 (Basic auth) and v2 (Bearer auth) Affinity API calls.

## Behavior when connector is missing

If the Affinity MCP is not connected, the skill cannot run Affinity operations. There is no graceful degradation — the plugin depends on the Affinity MCP.

## Privacy and scope

- **Authentication is per-user.** Each user authenticates with their own Affinity API key. The MCP never stores keys in the Worker environment — they're held in encrypted OAuth tokens.
- **Read and write access mirrors Affinity permissions.** If the user can't see a record in Affinity's web UI, they can't access it via the MCP.
- **Deal pipeline operations** target a specific Affinity list configured per environment (production vs staging).
- **No cross-user data access.** Each user sees only what their Affinity API key grants.
