# Per-User Dedup Algorithm

The dedup rules are the most load-bearing logic in the plugin. Get them right or the whole team experience breaks down.

## The rules (stated precisely)

Let U = the search owner (current user). Let S = the sourcing search being sourced. Let C = a candidate the pipeline just surfaced.

1. **Hard exclusion on same-search repeat.** If C has been delivered on search S (appears in `get_sourcing_status(search_id=S, scope="search")` deliveries), candidate is a hard no. Never serve C to U on S twice.
2. **Cross-search repeats are permitted, once, per person, ever.** If C's LinkedIn URL appears in `get_sourcing_status(search_id=S, scope="global")` but NOT in the search-scoped deliveries, C is eligible as a cross-search repeat. Using C in the current batch burns C permanently for user U.
3. **At most one cross-search repeat per batch.** Even if multiple eligible repeats exist, only one is served per run.
4. **Brand-new candidates are always preferred.** Use the eligible-repeat pool only to fill a slot when strictly necessary, and only one slot ever per batch.
5. **Burned candidates never return.** Once burned, filtered out of every future batch for U, across every search.
6. **Per-user scope.** A candidate delivered to user A's search S can still be served fresh to user B's search S' — two teammates' histories are independent.

## The implementation

Given batch size N (default 25):

### Phase 1 — Pull state from Lovelace

Fetch for the current user U:

- `search_deliveries` = call `get_sourcing_status(search_id=S, scope="search")`. Extract `linkedin_url` from each delivery.
- `global_urls` = call `get_sourcing_status(search_id=S, scope="global")`. This returns all LinkedIn URLs delivered across ALL of this user's searches.
- `cross_search_urls` = `global_urls` minus `search_deliveries` — LinkedIn URLs the user has seen on other searches.

All comparisons should use a normalized `linkedin_url` (strip trailing slash, lowercase, strip query strings).

### Phase 2 — Bucket the ranked candidate list

Take the scored, filtered, ranked candidate list from step 4 of `run-sourcing-batch`. Iterate in score order. For each candidate C:

```
if C.linkedin_url in search_deliveries:
    drop                                # hard exclusion — never repeat on same search
elif C.linkedin_url in cross_search_urls:
    bucket = eligible_repeat
else:
    bucket = brand_new
```

**Note on burned state:** The burned flag is tracked via the global URL list. A candidate who has been delivered on two different searches (once as original, once as cross-search repeat) will appear in the global set and be ineligible as a repeat again. Track explicit burn state in `config.json` per-search until a dedicated `burned` column is added to `sourcing_deliveries`.

### Phase 3 — Assemble the batch

1. Sort `brand_new` by score descending. Take up to N candidates.
2. If `len(brand_new) < N` AND `eligible_repeat` is non-empty:
   - Pick the single highest-scoring `eligible_repeat` candidate.
   - Add it to the batch.
   - Mark this candidate as burned in the search's `config.json`.
3. If `len(brand_new) == N`, do NOT pull from `eligible_repeat`. Prefer a batch of all-new over including a repeat unnecessarily.
4. Never include more than one repeat per batch — even if `brand_new` is small and `eligible_repeat` has many candidates.
5. If the final batch is empty, post an empty-batch note to Slack (see `run-sourcing-batch` step 7) and exit.

### Phase 4 — Record deliveries and post to Slack

Claude calls `record_sourcing_deliveries` to create `sourcing_deliveries` rows with `was_repeat` set appropriately, then posts profile cards to Slack with buttons already attached (using the returned `delivery_id`s).

## Edge cases

### The user has zero prior history

No deliveries for this user. Every sourced candidate is brand_new. Standard batch assembly.

### A candidate appears in multiple archetype searches within one run

Dedupe by `linkedin_url` before bucket assignment. One candidate, one bucket assignment.

### A candidate's LinkedIn URL format varies across runs

Normalize aggressively: strip `?` query strings, trailing slashes, case. If LinkedIn returned the same person via two URLs (rare but possible), dedupe by public identifier extracted from the URL path.

## Why these specific rules?

1. **Never repeat on same search** — protects the reviewer's trust. A tool that shows the same person twice loses credibility instantly.
2. **Cross-search repeats permitted once** — reflects real life: a great person for one search sometimes fits a second differently, but not a third. We'd rather give the user one chance to reconsider than hide them forever.
3. **Max one repeat per batch** — keeps the signal fresh. A 25-card batch where 5 are rehashes is demoralizing to review.
4. **Burning is terminal** — once the user has had two chances, we're done. Their attention is finite.

Do not soften these rules without a user-facing conversation. They're the contract — and they apply uniformly across every use case (recruiting, GTM, investment, LP, advisor, other).
