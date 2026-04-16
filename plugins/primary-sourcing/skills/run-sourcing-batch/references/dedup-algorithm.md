# Per-User Dedup Algorithm

The dedup rules are the most load-bearing logic in the plugin. Get them right or the whole team experience breaks down.

## The rules (stated precisely)

Let U = the search owner (current user). Let S = the search being sourced. Let C = a candidate the pipeline just surfaced.

1. **Hard exclusion on same-search repeat.** If (C, S) has a prior Served Leads record with `user=U`, candidate is a hard no. Never serve C to U on S twice.
2. **Cross-search repeats are permitted, once, per person, ever.** If C has been served to U on a different search S' ≠ S, and C.burned is false, C is eligible as a cross-search repeat. Using C in the current batch burns C permanently for user U.
3. **At most one cross-search repeat per batch.** Even if multiple eligible repeats exist, only one is served per run.
4. **Brand-new candidates are always preferred.** Use the eligible-repeat pool only to fill a slot when strictly necessary, and only one slot ever per batch.
5. **Burned candidates never return.** `burned = true` is terminal. Burned candidates are filtered out of every future batch for U, across every search.
6. **Per-user scope.** A candidate served to user A's search S can still be served fresh to user B's search S' — two teammates' histories are independent.

## The implementation

Given batch size N (default 25):

### Phase 1 — Pull state from Airtable

Fetch for the current user U:

- `served_on_this_search` = all Served Leads records where `search_id = S` and `user = U`. Extract the set of `linkedin_url`s.
- `served_on_other_searches` = all Served Leads records where `search_id != S` and `user = U`. Extract the set of `linkedin_url`s.
- `burned_candidates` = all Candidates records where `owner = U` and `burned = true`. Extract the set of `linkedin_url`s.

All comparisons should use a normalized `linkedin_url` (strip trailing slash, lowercase).

### Phase 2 — Bucket the ranked candidate list

Take the scored, filtered, ranked candidate list from step 4 of `run-sourcing-batch`. Iterate in score order. For each candidate C:

```
if C.linkedin_url in burned_candidates:
    drop                                # permanent exclusion
elif C.linkedin_url in served_on_this_search:
    drop                                # hard exclusion — never repeat on same search
elif C.linkedin_url in served_on_other_searches:
    bucket = eligible_repeat
else:
    bucket = brand_new
```

### Phase 3 — Assemble the batch

1. Sort `brand_new` by score descending. Take up to N candidates.
2. If `len(brand_new) < N` AND `eligible_repeat` is non-empty:
   - Pick the single highest-scoring `eligible_repeat` candidate.
   - Add it to the batch.
   - Mark this candidate as `burned = true` with reason `"cross-search repeat on {search_title}"`.
3. If `len(brand_new) == N`, do NOT pull from `eligible_repeat`. Prefer a batch of all-new over including a repeat unnecessarily.
4. Never include more than one repeat per batch — even if `brand_new` is small and `eligible_repeat` has many candidates.
5. If the final batch is empty, post an empty-batch note to Slack (see `run-sourcing-batch` step 8) and exit.

### Phase 4 — Write state back

For each candidate in the final batch:

- If the candidate is brand-new: create a Candidates record. Create a Served Leads record with `was_repeat = false`.
- If the candidate is an eligible repeat: skip Candidates creation (record exists). Create a Served Leads record with `was_repeat = true`. Update the existing Candidates record: set `burned = true` and `burned_reason`.

## Edge cases

### The user has zero prior history

No Served Leads or Candidates records for this user. Every sourced candidate is brand_new. Standard batch assembly.

### The user has a massive prior history (100s of candidates)

Paging Airtable reads is slow. Cache the three sets (served_on_this_role, served_on_other_roles, burned) per run — don't refetch mid-scoring.

### A candidate appears in multiple archetype searches within one run

Dedupe by `linkedin_url` before bucket assignment. One candidate, one bucket assignment, one Served Leads record.

### A candidate's LinkedIn URL format varies across runs

Normalize aggressively: strip `?` query strings, trailing slashes, case. If LinkedIn returned the same person via two URLs (rare but possible), dedupe by public identifier extracted from the URL path.

### The user manually adds a burned candidate back via Airtable

Respect user overrides. If `burned = false` when the batch runs, the candidate is eligible again (subject to normal rules). Don't silently re-burn.

## Why these specific rules?

1. **Never repeat on same search** — protects the reviewer's trust. A tool that shows the same person twice loses credibility instantly.
2. **Cross-search repeats permitted once** — reflects real life: a great person for one search sometimes fits a second differently, but not a third. We'd rather give the user one chance to reconsider than hide them forever.
3. **Max one repeat per batch** — keeps the signal fresh. A 25-card batch where 5 are rehashes is demoralizing to review.
4. **Burning is terminal** — once the user has had two chances, we're done. Their attention is finite.

Do not soften these rules without a user-facing conversation. They're the contract — and they apply uniformly across every use case (recruiting, GTM, investment, LP, advisor, other).
