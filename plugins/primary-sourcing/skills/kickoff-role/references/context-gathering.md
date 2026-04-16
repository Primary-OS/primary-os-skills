# Context Gathering Playbook

The kickoff flow is only as strong as the context it gathers before asking the user follow-up questions. This reference lays out which tools to query, what to search for, and how to synthesize the results.

## Goals

1. **Prove to the user you did homework.** Coming into the AskUserQuestion batch with a grounded synthesis beats asking generic questions.
2. **Avoid wasting the user's time re-stating what's already in the system.**
3. **Surface contradictions.** If Notion says "we want a VP" but a Granola meeting says "pivot to Director", flag that.

## What to query and how

Run these queries **in parallel** — issue all the tool calls in a single turn. Waiting on each sequentially turns a fast kickoff into a slow one.

### Granola

Use `query_granola_meetings` or `get_meetings` filtered by the company name and the role title. Look for:

- Hiring manager syncs (title contains "sync" or "1:1" + company name).
- Board meetings mentioning hiring plans.
- Any meeting in the past 60 days with the company's founders.

Pull transcript fragments for the most relevant matches — don't load full transcripts of every meeting.

### Slack

Use the Slack MCP's search tools to check both public and private channels. Queries:

- `"{company_name}"` limited to last 30 days.
- `"{role_title} {company_name}"` limited to last 90 days.
- Any channel explicitly named for this company.

Look for: direction changes, dealbreakers named in passing, founder updates, reference checks that happened informally.

### Notion

Search for pages mentioning the company's name. Look for:

- Dedicated portfolio company pages.
- Hiring plans or org charts.
- Board decks (when they're in Notion rather than Drive).

### Google Drive

Search for files. Look for:

- Job descriptions or "briefs" with the company name or role in the title.
- Outreach templates previously used.
- Past candidate packets.

### Gmail

Search threads. Look for:

- Recent threads with the company's founder email domain.
- Threads where other teammates mention the role by name.
- External referrals or candidate intros that came in via email.

### Affinity

Use `get_company` and `get_meetings_for_entity` to pull the portfolio company's Affinity record. Capture:

- Lead partner / deal owner.
- Relationship strength snapshot for key founders.
- Recent meeting notes if they're not already in Granola.

## Synthesis

After you have all the raw data, write the synthesis as inline context for yourself before replying to the user. Keep the summary you show the user focused — 3 to 8 bullets max. Format:

```
From context gathering:
• [Granola, Mar 28 HM sync] Founder wants someone with specific Infra FinOps experience, not general FP&A.
• [Slack, #pf-lighttable, Mar 31] Logan noted "we've already passed on 3 AE candidates from Procore — vertical fit isn't enough if deal size is wrong."
• [Drive, "LightTable Founding AE brief v2"] Target: IC closer with $300-500k ACV experience, not an enterprise leader.
• [Affinity] Lead partner: Ben Sun. Most recent founder meeting: Apr 2 (candidates discussed).
• [Gmail] Founder forwarded 2 candidate intros last week — assume some pipeline already exists.
```

Cite sources inline (bracketed). That's how the user builds trust that the synthesis is grounded, and it's how future iterations trace signals back.

## What to do when tools are empty

If a tool returns nothing for a company, say so — don't invent content. An explicit "No Granola meetings in the past 60 days" is better than silence or a fabricated bullet.

## Token budget

If the raw context gathered exceeds a few thousand tokens, summarize it before stuffing into the SEARCH.md generation step. Don't copy entire meeting transcripts into the role folder.
