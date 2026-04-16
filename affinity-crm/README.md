# Affinity CRM Plugin

Expert Affinity CRM guidance for the Primary team. Install this plugin so any Claude agent loads the `affinity-expert` skill before using Primary's custom Affinity MCP — deal pipeline, company/person research, notes, relationship intelligence, and list/field operations.

## What's inside

| Skill | What it does |
|-------|-------------|
| `affinity-expert` | Single entry point: routing table, prerequisite chains, error recovery, and links to reference docs for pipeline, research, notes, meetings, lists, and fields |

Reference files under `skills/affinity-expert/references/` hold detailed material: `custom-tools.md`, `pipeline-workflow.md`, `search-strategies.md`, `data-model.md`, `note-operations.md`, `meetings-and-reminders.md`, `field-types.md`, `list-navigation.md`.

## Required connector: Affinity MCP

This plugin requires a single MCP connection. It does not bundle or ship an MCP server — the Affinity MCP is deployed as a Cloudflare Worker and connected through Cowork, Claude Desktop, or Claude Code settings.

**How it works:**

1. **8 custom tools** optimized for Primary's deal pipeline — `log_company`, `update_company_status`, `search_company`, `get_company`, `get_status_options`, `get_owners`, `get_recent_deals`, `get_current_user`.
2. **~24 proxied official Affinity MCP tools** — person search, notes, lists, fields, meetings, relationship strengths, reminders, transcripts.
3. **3 upstream tools excluded** and replaced by pipeline-enriched custom versions: `search_companies` → `search_company`, `get_company_info` → `get_company`, upstream `get_current_user` → local `get_current_user`.

Authentication is via OAuth — enter an Affinity API key once (generate at Settings > Manage Apps in Affinity). The MCP handles both v1 (Basic auth) and v2 (Bearer auth) calls. Each user authenticates with their own key. No cross-user data access.

If the Affinity MCP is not connected, the skill cannot run Affinity operations. There is no graceful degradation.

## Install

```bash
claude plugin install affinity-crm@primary-os
```

## How it works

The plugin teaches agents how to use Primary's Affinity MCP effectively. The `affinity-expert` skill is written to load before tool calls, with tool-name-first description text so it competes with bare tool self-descriptions. Reference docs cover parameters, disambiguation, and workflows so agents can operate the CRM confidently without guessing.
