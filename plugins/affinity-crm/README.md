# Affinity CRM Plugin

Expert Affinity CRM guidance for the Primary team. Install this plugin so any Claude agent loads **one** consolidated skill (`affinity-expert`) before using Primary’s custom Affinity MCP — deal pipeline, company/person research, notes, relationship intelligence, and list/field operations.

## What's inside

| Skill | What it does |
|-------|-------------|
| `affinity-expert` | Single entry point: routing table, prerequisite chains, error recovery, and links to reference docs for pipeline, research, notes, meetings, lists, and fields |

Reference files under `skills/affinity-expert/references/` hold the same detailed material previously split across four skills (`custom-tools.md`, `pipeline-workflow.md`, `search-strategies.md`, `data-model.md`, `note-operations.md`, `meetings-and-reminders.md`, `field-types.md`, `list-navigation.md`).

## Prerequisites

- **Affinity MCP** connected in your Cowork or Claude settings. See `CONNECTORS.md` for details.
- An **Affinity API key** with appropriate permissions (generate at Settings > Manage Apps in Affinity).

## Install

```bash
claude plugin install affinity-crm@primary-os
```

## How it works

The plugin teaches agents how to use Primary's Affinity MCP effectively. The MCP provides 8 custom pipeline-aware tools and proxies ~24 official Affinity tools through a single connection. The `affinity-expert` skill is written to load before tool calls, with tool-first description text so it competes with bare tool self-descriptions. Reference docs cover parameters, disambiguation, and workflows so agents can operate the CRM confidently without guessing.
