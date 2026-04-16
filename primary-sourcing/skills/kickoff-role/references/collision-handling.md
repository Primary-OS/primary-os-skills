# Slug and Channel Collision Handling

Role slugs drive the Lovelace search key, the Slack channel name, and the role folder name. Collisions are rare but must be handled deterministically.

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

No existing Lovelace sourcing search with this slug. Use it directly — no suffix needed.

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

**Important**: call `create_sourcing_search` (step 8 in the main flow) with the final slug. If 409 conflict, the slug is taken — apply suffix and retry.

## Slack channel naming

Slack channel creation and naming is handled server-side by Lovelace when `create_sourcing_search` is called. The channel name follows the pattern `sourcing-{search_slug}`. Collision handling (email prefix, then sequential numbers) is automatic — the agent does not need to manage this.

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

Do not use UUIDs, timestamps, or database IDs in the slug — they defeat the readability goal.
