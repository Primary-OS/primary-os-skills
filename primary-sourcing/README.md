# Primary Sourcing Plugin

A team-wide sourcing agent for Primary Venture Partners, distributed as a Cowork / Claude Code plugin. Scaffolds a Cowork Project per sourcing subject (portfolio company, investment thesis, fundraising theme, etc.), kicks off individual searches inside it, and runs scheduled LinkedIn-based sourcing with per-user dedup, Slack feedback loops, and weekly digests.

Built on the single-user sourcing agent Annabelle Kim (@annabelle@primary.vc) built for Primary Talent. Extended to multi-user, multi-use-case.

## Supported use cases

- **Recruiting for a portfolio company** (headline use case)
- Sales / GTM sourcing for a portfolio company's outreach
- Investment sourcing (founders or companies)
- Fund / LP sourcing
- Advisor / board member / SME sourcing
- Custom ("Other")

Use case is picked at Project scaffold time and determines how every subsequent skill behaves — follow-up questions, scoring rubric, Slack phrasing, and weekly-digest analysis all adapt.

## Install

### From the Primary OS marketplace

```
/plugin install primary-sourcing@primary-os
```

If the `primary-os` marketplace isn't added yet, first:

```
/plugin marketplace add Primary-OS/primary-os-skills
```

### For development — run locally

```
claude --plugin-dir /path/to/this/plugin/folder
```

## Skills

| Skill                      | Purpose                                                                                             | Typically invoked by                               |
| -------------------------- | --------------------------------------------------------------------------------------------------- | -------------------------------------------------- |
| `start-sourcing-project`   | Scaffold a fresh Cowork Project for ongoing sourcing at one subject. Pick use case, gather context. | User, once per subject (PortCo, thesis, fundraise) |
| `kickoff-role`             | Kick off a new search inside a scaffolded Project. Creates Lovelace search + Slack channel + tasks.| User, per search                                   |
| `run-sourcing-batch`       | Run one sourcing batch. Dedup-aware. Posts profile cards to Slack.                                  | Cowork scheduled tasks (and manual)                |
| `run-weekly-summary`       | Post the weekly digest to a search's Slack channel.                                                 | Cowork scheduled tasks (and manual)                |

## Architecture

- **Cowork Project per subject**: shared across the team. Holds `CLAUDE.md`, `project-context/`, `roles/{search_slug}/`, and `templates/`.
- **Lovelace platform for sourcing state**: projects, deliveries, and feedback stored in Supabase. Per-user dedup history is naturally user-scoped. The Slack bot writes deliveries and feedback server-side — Claude reads state via `get_sourcing_status` and never writes tracking data directly.
- **Lovelace MCP for LinkedIn search**: shared Primary Apify account wrapped as an MCP tool, plus sourcing search CRUD (`create_sourcing_search`, `get_sourcing_status`, `update_sourcing_search`).
- **Scheduled tasks per search**: one for sourcing cadence, one for the weekly digest. Prompts invoke the skills in this plugin.
- **Slack bot**: posts profile cards with Yes/Maybe/Pass/Details buttons. Records deliveries and feedback to Supabase. Updates card status after user clicks.

## Dedup rules (universal across use cases)

1. Never serve the same lead twice on the same search.
2. At most one cross-search repeat per batch per user.
3. Cross-search repeats are burned on serve — permanent exclusion for that user.
4. Scope is per-user — teammates' histories are independent.

Full algorithm: `skills/run-sourcing-batch/references/dedup-algorithm.md`.

## Required connectors

| Connector | Purpose |
|-----------|---------|
| **Slack** | Per-search channels, profile cards, feedback buttons, surface commands, weekly digests |
| **scheduled-tasks** | Recurring sourcing batches + weekly digests |
| **Lovelace MCP** | LinkedIn search (shared Apify account), sourcing search CRUD, delivery/feedback state |

Optional for richer kickoff context: Granola, Notion, Google Drive, Gmail, Affinity.

### Privacy and scope

- **Lovelace**: all sourcing state (projects, deliveries, feedback) stored in Supabase. Each user's projects are scoped to their account. The Slack bot writes via internal API key.
- **Slack**: the plugin only reads and writes to channels it creates (`sourcing-{search_slug}`), never broader workspace data.
- **Context gathering connectors** (Granola/Notion/Drive/Gmail/Affinity): reads only. No writes.

## Bundled templates

When the plugin scaffolds a Project, it drops in these templates:

- `templates/company-project/CLAUDE.md` — Project instructions for every Cowork conversation.
- `templates/role/SEARCH-TEMPLATE.md` — starting structure for each search's brain.
- `templates/role/config-template.json` — per-search config schema.

## Issues and contributions

Issues live in the Linear "Claude Plugin Marketplace" project. Plugin source is in `Primary-OS/primary-os-skills` under `plugins/primary-sourcing/`.

## License

Proprietary to Primary Venture Partners. See marketplace `LICENSE`.
