# Slack Profile Card Formatting

Every sourcing batch posts a header + one profile card per candidate. This reference defines the exact Block Kit structure.

## Batch header

```
🔍 *Sourcing run — {Month Day, Year}*
{N} new candidate{s}. Use the buttons to give feedback, or type `surface my maybes` anytime to revisit.
```

Posted as a simple `chat.postMessage` with the text above (no blocks).

## Profile card (one per candidate)

Use Block Kit. Structure:

```json
{
  "blocks": [
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "*{name}*\n{current_title} @ {current_company}"
      }
    },
    {
      "type": "section",
      "text": {
        "type": "mrkdwn",
        "text": "<{linkedin_url}|LinkedIn Profile>"
      }
    },
    {
      "type": "context",
      "elements": [
        { "type": "mrkdwn", "text": "_{rationale}_" }
      ]
    },
    {
      "type": "actions",
      "elements": [
        {
          "type": "button",
          "text": { "type": "plain_text", "text": "✓  Yes" },
          "style": "primary",
          "action_id": "candidate_yes",
          "value": "{encoded_meta}"
        },
        {
          "type": "button",
          "text": { "type": "plain_text", "text": "◌  Maybe" },
          "action_id": "candidate_maybe",
          "value": "{encoded_meta}"
        },
        {
          "type": "button",
          "text": { "type": "plain_text", "text": "✗  No" },
          "style": "danger",
          "action_id": "candidate_no",
          "value": "{encoded_meta}"
        }
      ]
    },
    { "type": "divider" }
  ],
  "text": "{name} — {current_title} @ {current_company}"
}
```

The fallback `text` field is required for notifications and accessibility.

## Button `value` encoding

Each button value is a JSON-stringified object:

```json
{
  "served_id": "recAbcd123",
  "candidate_id": "recEfgh456",
  "role_id": "recIjkl789",
  "user_email": "logan@primary.vc"
}
```

The Slack bot (Lovelace side) reads this on click and writes a Feedback record to Airtable. Slack limits button values to 2000 chars — the object above is well under that.

## If the candidate has no rationale

Skip the context block entirely. Don't render `_(no rationale)_` or similar — it's visual noise.

## Pacing between posts

Insert a 300ms delay between posts to maintain message order and stay under Slack's rate limit. Do not post more than 20 cards in under 10 seconds.

## Empty-batch note

If the final batch is zero candidates, post:

```
🔍 *Sourcing run — {date}*
No new candidates this run. Either we're caught up on the pipeline or the criteria need to broaden. Type any guidance into this channel and I'll adjust.
```

## Rationale for the format

- Two sections (name/title line + LinkedIn link) instead of one big block: makes the card skimmable — the reviewer can glance at 25 cards in 60 seconds.
- `context` block for rationale: visually lighter than a full section, signals "supporting info".
- Three buttons: Yes is primary (green), No is danger (red), Maybe is neutral. The styling matters — Yes/No stand out, Maybe fades.
- Divider between cards: clear visual separation at scale.
