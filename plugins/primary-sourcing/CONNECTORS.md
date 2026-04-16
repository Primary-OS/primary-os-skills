# Connectors

The `primary-sourcing` plugin coordinates several MCP connectors. This document lists every connector the plugin interacts with, whether it's required or optional, and which use cases rely on it.

The plugin itself does not bundle or ship an MCP server — all connectors are provided by Claude Cowork, Claude Code, or first-party Primary tooling (Lovelace). Users connect them through their Cowork settings.

## Required for core sourcing flow

| Connector         | Purpose in this plugin                                                                                                                                                            | All use cases? |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------- |
| **Slack**         | Create per-search private channels. Post profile cards with Yes/Maybe/No buttons. Read feedback (channel messages). Post weekly digests.                                          | Yes            |
| **Airtable**      | User-owned base with four tables (Searches, Candidates, Served Leads, Feedback). Source of truth for the user's sourcing history and dedup state.                                 | Yes            |
| **scheduled-tasks** | Create recurring Cowork scheduled tasks (one per search for sourcing batches; one for weekly digests).                                                                          | Yes            |
| **Lovelace MCP**  | Shared Apify-backed LinkedIn Profile Search tool using Primary's Apify account server-side. Without this, `run-sourcing-batch` cannot fetch profiles.                             | Yes            |

## Recommended for kickoff context gathering

These are optional — kickoff works without them, but the initial SEARCH.md is much richer when they're connected.

| Connector         | Purpose in this plugin                                                                                                           | Use case priority                                                          |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| **Affinity**      | Portfolio company metadata, founder relationship strength, recent meetings. Best for auto-validating the Project subject.        | High for: recruiting, investment, LP/fund. Medium for: GTM, advisor.       |
| **Granola**       | Past meeting transcripts — HM syncs, founder calls, board meetings.                                                              | High for all use cases.                                                    |
| **Notion**        | Portfolio company pages, thesis docs, hiring plans.                                                                              | High for recruiting & investment. Medium for others.                       |
| **Google Drive**  | Job briefs, board decks, outreach templates, ICP docs.                                                                           | High for recruiting & GTM. Medium for others.                              |
| **Gmail**         | Founder threads, candidate intros, external referrals.                                                                           | Medium for all use cases.                                                  |

## Behavior when connectors are missing

- **Required connectors**: If any are missing at kickoff time (or when a scheduled task runs), the skill aborts with a clear message telling the user what to connect and why. The plugin never silently skips required steps.
- **Recommended connectors**: The plugin gracefully degrades. If Granola isn't connected, the kickoff flow just skips Granola queries and leans harder on user-provided context.

## Privacy and scope

- Airtable: each teammate's base holds only their own sourcing history. No cross-user reads.
- Apify (via Lovelace MCP): shared API key is server-side; the Lovelace MCP logs caller identity and credit usage per call for internal monitoring.
- Slack: the plugin only reads and writes to channels it creates (named `sourcing-{search_slug}`), never broader workspace data.
- Context gathering connectors (Granola/Notion/Drive/Gmail/Affinity): reads only. No writes.

## Future connectors

Planned or under consideration:

- **LinkedIn Recruiter** (direct API if/when available) — reduce Apify dependency.
- **Clay** — optional structured enrichment layer. Already supported in the single-user sourcing agent; porting is straightforward.
- **HubSpot / Salesforce** — for GTM use case, avoid double-touching accounts already in the PortCo's CRM.

None of these are required. The plugin is designed to add connectors without breaking existing users.
