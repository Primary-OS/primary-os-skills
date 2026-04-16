# Slug and Channel Collision Handling

Role slugs drive the Airtable key, the Slack channel name, and the role folder name. Collisions are rare but must be handled deterministically.

## Base slug construction

Base slug = `{company_slug}-{role_slug}`.

Rules:

- Lowercase everything.
- Replace any non `[a-z0-9]` with `-`.
- Collapse repeated hyphens.
- Strip leading/trailing hyphens.
- Drop stopwords from the role title: `a`, `an`, `the`, `of`, `for`, `to`, `in`, `at`. Keep meaningful words — "Head of Sales" becomes "head-sales", not "sales".
- Example: `"LightTable"` + `"Founding AE"` → `lighttable-founding-ae`.

## When the base slug is free

No existing Airtable Roles record with this slug. Use it directly.

## When the base slug is taken

Sources of collision:

1. Same teammate already kicked off a role with this slug (probably re-running a kickoff). Before suffixing, ask the user: "You already have a role with slug `lighttable-founding-ae`. Are you re-running that kickoff, or is this a second role for the same title?"
   - If re-running: abort and tell them to use the existing role.
   - If genuinely second role: proceed to suffix.
2. A different teammate has this slug. Proceed to suffix.
3. Same slug, role is closed. Still suffix — closed roles stay for audit.

## Suffix algorithm

After creating the Airtable Role record (step 7 in the main flow), append `-{rec_suffix}` where `rec_suffix` is the last 8 characters of the Airtable record ID, lowercased, and sanitized to `[a-z0-9]`.

Example:

- Airtable returns `recAB1cD2eF3gH4iJ5`.
- `rec_suffix` = `f3gh4ij5`.
- Final slug: `lighttable-founding-ae-f3gh4ij5`.

Write the suffixed slug back to the Airtable Role record's `role_slug` field.

## Slack channel naming

Channel name = `sourcing-{role_slug}`. Slack has a 80-character limit and disallows certain characters.

If the resulting channel name exceeds 80 chars, truncate the role part (not the company part) until it fits, keeping the record-ID suffix intact for uniqueness.

## What happens if Slack reports `name_taken`

Slack collision independent of Airtable — another workspace channel has this name (maybe from an old run before the Airtable record was deleted). Log a warning, then:

1. Try `sourcing-{role_slug}-v2`.
2. If still taken, try `-v3`, etc., up to `-v5`.
3. If still taken, abort and surface the issue to the user.

Update `slack_channel_name` on the Airtable record with whatever name actually got created.

## Role folder naming

The workspace folder name matches `role_slug`:

```
roles/
  lighttable-founding-ae-f3gh4ij5/
    SEARCH.md
    KICKOFF.md
    config.json
```

The folder name must match `role_slug` exactly. Downstream skills (`run-sourcing-batch`, `run-weekly-summary`) depend on this.

## Why the Airtable record ID as suffix?

Two reasons:

1. It's guaranteed unique — the Airtable API generates it.
2. It's human-scannable — 8 characters at the end is tolerable; teammates reading `#sourcing-lighttable-founding-ae-f3gh4ij5` can still tell at a glance what search it's for.

Do not use UUIDs, timestamps, or sequential counters — they defeat the scannability goal.
