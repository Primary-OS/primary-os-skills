# Slack Intro Message Templates Per Use Case

When a new search is kicked off, the skill posts an intro message to the newly created Slack channel. Phrasing adapts per use case so the message reads naturally.

## Common structure

Every intro includes:

1. Headline — what was launched.
2. "How to interact with me" — buttons, channel messages, surface commands.
3. Schedule summary.
4. Owner tag.

## Templates

### Recruiting (`recruiting-portco`)

```
🚀 *Sourcing search launched: {role_title} at {subject_name}*

I'll source candidates and post them here with Yes / Maybe / Pass buttons.

*How to interact with me:*
• Click the buttons on each profile card to give feedback
• Type notes or direction changes directly in this channel — I'll update the search criteria in real time
• Type `surface my maybes` to revisit Maybe candidates
• Type `surface nos because [reason]` to re-evaluate specific passes

*Schedule:*
• Sourcing runs: {schedule description}
• Weekly digest: {Friday 4pm or skipped}

Owner: <@{owner_slack_id}>
```

### GTM sourcing (`gtm-sourcing-portco`)

```
🎯 *Prospect sourcing launched: {persona_title} for {subject_name}'s GTM*

I'll surface buyer prospects and post them here with Yes / Maybe / Pass buttons.

*How to interact with me:*
• Click buttons on each card to give feedback
• Type notes or account targeting changes in this channel — I'll adjust the ICP in real time
• Type `surface my maybes` to revisit Maybe prospects
• Type `surface nos because [reason]` to re-evaluate specific passes

*Schedule:*
• Prospect batches: {schedule description}
• Weekly report: {Friday 4pm or skipped}

Owner: <@{owner_slack_id}>
```

### Investment sourcing (`investment-sourcing`)

```
🧭 *Investment search launched: {archetype_title}*
_Thesis: {subject_name}_

I'll surface founders and companies matching this thesis and post them here with Yes / Maybe / Pass buttons.

*How to interact with me:*
• Click buttons on each profile to give feedback on fit
• Type thesis adjustments or direction changes directly in this channel — I'll update the search in real time
• Type `surface my maybes` to revisit Maybes
• Type `surface nos because [reason]` to re-evaluate passes

*Schedule:*
• Sourcing batches: {schedule description}
• Weekly digest: {Friday 4pm or skipped}

Owner: <@{owner_slack_id}>
```

### Fund/LP sourcing (`fund-lp-sourcing`)

```
💼 *LP / fund search launched: {archetype_title}*
_For: {subject_name}_

I'll surface LPs, GPs, or ecosystem partners matching this mandate and post them here with Yes / Maybe / Pass buttons.

*How to interact with me:*
• Click buttons on each profile to give feedback
• Type mandate changes or targeting adjustments directly in this channel — I'll adjust the search
• Type `surface my maybes` to revisit Maybes
• Type `surface nos because [reason]` to re-evaluate passes

*Schedule:*
• Sourcing batches: {schedule description}
• Weekly digest: {Friday 4pm or skipped}

Owner: <@{owner_slack_id}>
```

### Advisor / board (`advisor-board-sourcing`)

```
🧠 *Advisor search launched: {expertise_title}*
_For: {subject_name}_

I'll surface advisors and subject matter experts matching this shape and post them here with Yes / Maybe / Pass buttons.

*How to interact with me:*
• Click buttons on each profile to give feedback
• Type notes or shape changes directly in this channel — I'll update the search
• Type `surface my maybes` to revisit Maybes
• Type `surface nos because [reason]` to re-evaluate passes

*Schedule:*
• Sourcing batches: {schedule description}
• Weekly digest: {Friday 4pm or skipped}

Owner: <@{owner_slack_id}>
```

### Other (`other`)

```
🔍 *Sourcing search launched: {title}*
_Project: {subject_name}_

I'll surface matches and post them here with Yes / Maybe / Pass buttons.

*How to interact with me:*
• Click buttons on each card to give feedback
• Type direction changes in this channel — I'll adjust the search in real time
• Type `surface my maybes` to revisit Maybes
• Type `surface nos because [reason]` to re-evaluate passes

*Schedule:*
• Sourcing batches: {schedule description}
• Weekly digest: {Friday 4pm or skipped}

Owner: <@{owner_slack_id}>
```

## Principles

- Lead with an emoji that matches the use case. Recognizable at a glance when a teammate has multiple sourcing channels open.
- Keep it under 15 lines. Long intros get ignored.
- Always include the surface commands so users remember they exist.
- Always tag the owner explicitly so notifications route correctly.
- No marketing language. Direct and practical.
