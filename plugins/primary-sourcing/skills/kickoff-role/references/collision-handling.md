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

No existing Airtable Searches record with this slug. Use it directly — no suffix needed.

## When the base slug is taken

Sources of collision:

1. Same teammate already kicked off a search with this slug (probably re-running a kickoff). Before suffixing, ask the user: "You already have a search with slug `lighttable-founding-ae`. Are you re-running that kickoff, or is this a second search for the same title?"
   - If re-running: abort and tell them to use the existing search.
   - If genuinely second search: proceed to suffix.
2. A different teammate has this slug. Proceed to suffix.
3. Same slug, search is closed. Still suffix — closed searches stay for audit.

## Suffix algorithm

Only apply a suffix when the base slug collides with an existing record.

The suffix is the owner's **email prefix** — the part before `@` in their `owner_email`, lowercased and sanitized to `[a-z0-9-]`. Since all teammates are at the same company domain, no two people share an email prefix.

Example:

- Base slug: `lighttable-founding-ae` (taken by another teammate).
- Owner email: `annabelle@primary.vc`.
- Email prefix: `annabelle`.
- Final slug: `lighttable-founding-ae-annabelle`.

If the email-prefix slug *still* collides (same teammate, genuinely second search with the same title — probably accidental), append a sequential number: `-2`, `-3`, etc.

Example:

- `lighttable-founding-ae-annabelle` already exists.
- Final slug: `lighttable-founding-ae-annabelle-2`.
- If that also exists: `lighttable-founding-ae-annabelle-3`, and so on.

**Important**: always create the Airtable Search record (step 8 in the main flow) before resolving the final slug. The record must exist first so the record ID is safely stored. Then update the record's `search_slug` field with the final resolved slug.

## Slack channel naming

Channel name = `sourcing-{search_slug}`. Slack has an 80-character limit and disallows certain characters.

If the resulting channel name exceeds 80 chars, truncate the search title portion (not the company slug or email-prefix suffix) until it fits, keeping the suffix intact for uniqueness.

## What happens if Slack reports `name_taken`

Slack collision independent of Airtable — another workspace channel has this name (maybe from an old run before the Airtable record was deleted). Log a warning, then:

1. Try `sourcing-{role_slug}-v2`.
2. If still taken, try `-v3`, etc., up to `-v5`.
3. If still taken, abort and surface the issue to the user.

Update `slack_channel_name` on the Airtable record with whatever name actually got created.

## Search folder naming

The workspace folder name matches `search_slug`:

```
roles/
  lighttable-founding-ae-annabelle/
    SEARCH.md
    KICKOFF.md
    config.json
```

The folder name must match `search_slug` exactly. Downstream skills (`run-sourcing-batch`, `run-weekly-summary`) depend on this.

## Why this layered approach?

The suffix strategy prioritizes readability at each tier:

1. **No suffix** (base slug is free) — cleanest possible: `lighttable-founding-ae`.
2. **Email prefix** (cross-teammate collision) — human-readable, instantly shows ownership: `lighttable-founding-ae-annabelle`.
3. **Email prefix + sequential number** (same person, duplicate title) — still scannable: `lighttable-founding-ae-annabelle-2`.

Do not use UUIDs, timestamps, or Airtable record IDs in the slug — they defeat the readability goal.
