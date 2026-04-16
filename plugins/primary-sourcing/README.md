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
| `kickoff-role`             | Kick off a new search inside a scaffolded Project. Creates Slack channel + Airtable record + tasks. | User, per search                                   |
| `run-sourcing-batch`       | Run one sourcing batch. Dedup-aware. Posts profile cards to Slack.                                  | Cowork scheduled tasks (and manual)                |
| `run-weekly-summary`       | Post the weekly digest to a search's Slack channel.                                                 | Cowork scheduled tasks (and manual)                |

## Architecture

- **Cowork Project per subject**: shared across the team. Holds `CLAUDE.md`, `project-context/`, `roles/{search_slug}/`, and `templates/`.
- **Per-user Airtable base**: each teammate connects their own base with Searches, Candidates, Served Leads, Feedback tables. Dedup history is naturally user-scoped.
- **Lovelace MCP for Apify**: shared Primary Apify account is wrapped as a Lovelace MCP tool, so teammates don't need their own Apify key.
- **Scheduled tasks per search**: one for sourcing cadence, one for the weekly digest. Prompts invoke the skills in this plugin.
- **Slack bot**: extends the existing Lovelace Slack bot to route channel events (buttons, messages) to the right Search record via Airtable lookup.

## Dedup rules (universal across use cases)

1. Never serve the same lead twice on the same search.
2. At most one cross-search repeat per batch per user.
3. Cross-search repeats are burned on serve — permanent exclusion for that user.
4. Scope is per-user — teammates' histories are independent.

Full algorithm: `skills/run-sourcing-batch/references/dedup-algorithm.md`.

## Connector prerequisites

See `CONNECTORS.md` for the full list and which use case needs which connector.

Required for sourcing to actually run: Slack, Airtable, scheduled-tasks, Lovelace MCP.
Optional for richer kickoff context: Granola, Notion, Google Drive, Gmail, Affinity.

## Bundled templates

When the plugin scaffolds a Project, it drops in these templates:

- `templates/company-project/CLAUDE.md` — Project instructions for every Cowork conversation.
- `templates/role/SEARCH-TEMPLATE.md` — starting structure for each search's brain.
- `templates/role/config-template.json` — per-search config schema.

## Versioning

Semantic versioning. See `CHANGELOG.md` for release notes.

## Issues and contributions

Issues live in the Linear "Team Recruiting Sourcing Agent" project. Plugin source is in `Primary-OS/primary-os-skills` under `plugins/primary-sourcing/`.

Pull requests welcome — keep the plugin self-contained and version-bumped.

## License

Proprietary to Primary Venture Partners. See marketplace `LICENSE`.
