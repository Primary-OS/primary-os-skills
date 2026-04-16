# MCP Prerequisites for Primary Recruiting Sourcing

The plugin depends on a set of MCP connectors. Not every connector is required up front — some are only needed at specific steps. Use this reference to tell users what to connect and why.

## Required for role kickoff and scheduled sourcing

| MCP              | Purpose                                                                                                                                                                          |
| ---------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Slack            | Create per-role private channels, invite the role owner, post intro messages, post profile cards with Yes/Maybe/No buttons, read feedback messages, post weekly digests.         |
| Airtable         | Per-user Airtable base containing Roles, Candidates, Served Leads (with burned flag), and Feedback tables. Source of truth for the owner's sourcing history and dedup state.     |
| scheduled-tasks  | Create recurring Cowork scheduled tasks for per-role sourcing runs and weekly summaries.                                                                                         |
| Lovelace MCP     | Provides the Apify-backed LinkedIn search tool with the shared Primary API key. Without this, the scheduled sourcing task cannot source candidates.                              |

## Recommended for kickoff context gathering

These are strictly optional, but the more connected, the richer the initial SEARCH.md. Kickoffs still work without them — they just lean harder on the user to provide context verbally.

| MCP          | Purpose                                                                                                                           |
| ------------ | --------------------------------------------------------------------------------------------------------------------------------- |
| Affinity     | Portfolio company metadata, relationship graph, recent meetings with founders. Great for auto-suggesting the target company.      |
| Granola      | Past meeting transcripts — especially HM syncs, board meetings, portfolio check-ins.                                              |
| Notion       | Internal company notes, board decks, hiring plans.                                                                                |
| Google Drive | Job packets, briefs, outreach templates.                                                                                          |
| Gmail        | Founder threads, candidate pipelines, external intros.                                                                            |

## How to report prerequisite status to the user

When the `start-sourcing-project` or `kickoff-role` skill checks prereqs, format the status like this:

```
MCP prerequisites for this Project:
  ✓ Slack          (connected)
  ✓ Airtable       (connected — base: "Primary Sourcing (Logan)")
  ✓ scheduled-tasks (connected)
  ✗ Lovelace MCP    — REQUIRED for sourcing. Ask #ai-tools to add it to your Cowork.
  ✓ Granola        (connected)
  ✓ Affinity       (connected)
  ✗ Gmail          — optional, skips gmail context gathering
```

Use checkmarks (✓) for connected, crosses (✗) for missing. Call out "REQUIRED" on anything that blocks sourcing batches from running — the user needs to know they won't get candidates until those are added.

## What to tell the user if a required MCP is missing

- **Slack missing**: "Ask your workspace admin to grant you the Slack connector. You can still scaffold the Project but can't kick off roles until this is connected."
- **Airtable missing**: "Connect your Airtable base via the Airtable connector in Cowork settings. See the onboarding doc for the base schema you need."
- **scheduled-tasks missing**: "This is part of Cowork itself — it should already be there. If it's missing, message #ai-tools."
- **Lovelace MCP missing**: "The Primary Lovelace MCP gives every teammate access to the shared Apify account for LinkedIn searches. Ask Hi-Tech to install it for your account."
