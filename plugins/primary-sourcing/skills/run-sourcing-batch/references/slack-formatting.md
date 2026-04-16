# Slack Profile Card Formatting

Every sourcing batch posts a header + one profile card per candidate. This reference defines the exact Block Kit structure.

## Batch header

```
🔍 *Sourcing run — {Month Day, Year}*
{N} new candidate{s}. Use the buttons to give feedback, or type `surface my maybes` anytime to revisit.
```

Posted as a simple `chat.postMessage` with the text above (no blocks).

## Profile card (one per candidate)

Matches the Lovelace lead card design:

```
*1. Jane Chen*  ·  Score *9*  ·  <https://linkedin.com/in/janechen|LinkedIn>
Staff ML Engineer  ·  DeepMind  ·  San Francisco Bay Area
> Enterprise AI, 20+ yrs eng leadership. Strong infrastructure-to-startup bridge.

[ Yes (primary) ]  [ Maybe ]  [ Pass ]  [ Details ]
```

## Two-stage posting flow

Claude posts the card content **without buttons**. The Slack bot detects the posted card, creates a `sourcing_deliveries` row in Supabase, obtains the `delivery_id`, and **updates the message** to inject action buttons with the `delivery_id` embedded. This two-stage flow exists because the `delivery_id` does not exist until the bot creates the delivery row.

### Stage 1: Claude posts (no actions block)

Use Block Kit:

```json
{
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*{index}. {name}*  ·  Score *{score}*  ·  <{linkedin_url}|LinkedIn>\n{current_title}  ·  {current_company}  ·  {location}"
      }
    },
    {
      "type": "context",
      "elements": [
        { "type": "mrkdwn", "text": "> {rationale}" }
      ]
    },
    { "type": "divider" }
  ],
  "text": "{name} — {current_title} @ {current_company}"
}
```

The fallback `text` field is required for notifications and accessibility.

### Stage 2: Bot injects buttons (automatic — not Claude's responsibility)

The bot updates the message to add the actions block:

```json
{
  "type": "actions",
  "elements": [
    { "type": "button", "text": { "type": "plain_text", "text": "Yes" }, "style": "primary", "action_id": "sourcing_yes_{delivery_id}", "value": "{delivery_id}" },
    { "type": "button", "text": { "type": "plain_text", "text": "Maybe" }, "action_id": "sourcing_maybe_{delivery_id}", "value": "{delivery_id}" },
    { "type": "button", "text": { "type": "plain_text", "text": "Pass" }, "action_id": "sourcing_pass_{delivery_id}", "value": "{delivery_id}" },
    { "type": "button", "text": { "type": "plain_text", "text": "Details" }, "action_id": "sourcing_details_{delivery_id}", "value": "{delivery_id}" }
  ]
}
```

## Button conventions

- **Yes** = primary style (green). All others default (no danger style — Lovelace convention).
- **Pass** opens a modal with structured reason select + optional notes.
- **Maybe** opens a modal with optional notes.
- **Details** shows person snapshot.

## Button `value` encoding

Each button value is the `delivery_id` (UUID string). Slack limits button values to 2000 chars — a single UUID is well under that.

## Status updates after feedback

The bot replaces buttons with a status line:
- `"Marked as *Yes* by <@U123>"`
- `"*Maybe* — "notes" by <@U123>"`
- `"*Passed* geography by <@U123>"`

## If the candidate has no rationale

Skip the context block entirely. Don't render empty rationale — it's visual noise.

## Pacing between posts

Insert a 300ms delay between posts to maintain message order and stay under Slack's rate limit. Do not post more than 20 cards in under 10 seconds.

## Empty-batch note

If the final batch is zero candidates, post:

```
🔍 *Sourcing run — {date}*
No new candidates this run. Either we're caught up on the pipeline or the criteria need to broaden. Type any guidance into this channel and I'll adjust.
```
