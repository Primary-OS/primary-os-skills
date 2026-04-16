# Per-User Dedup Algorithm

The dedup rules are the most load-bearing logic in the plugin. Get them right or the whole team experience breaks down.

## The rules (stated precisely)

Let U = the search owner (current user). Let S = the sourcing project being sourced. Let C = a candidate the pipeline just surfaced.

1. **Hard exclusion on same-project repeat.** If C has been delivered on project S (appears in `get_sourcing_status(project_id=S, scope="project")` deliveries), candidate is a hard no. Never serve C to U on S twice.
2. **Cross-project repeats are permitted, once, per person, ever.** If C's LinkedIn URL appears in `get_sourcing_status(project_id=S, scope="global")` but NOT in the project-scoped deliveries, C is eligible as a cross-project repeat. Using C in the current batch burns C permanently for user U.
3. **At most one cross-project repeat per batch.** Even if multiple eligible repeats exist, only one is served per run.
4. **Brand-new candidates are always preferred.** Use the eligible-repeat pool only to fill a slot when strictly necessary, and only one slot ever per batch.
5. **Burned candidates never return.** Once burned, filtered out of every future batch for U, across every project.
6. **Per-user scope.** A candidate delivered to user A's project S can still be served fresh to user B's project S' — two teammates' histories are independent.

## The implementation

Given batch size N (default 25):

### Phase 1 — Pull state from Lovelace

Fetch for the current user U:

- `project_deliveries` = call `get_sourcing_status(project_id=S, scope="project")`. Extract `linkedin_url` from each delivery.
- `global_urls` = call `get_sourcing_status(project_id=S, scope="global")`. This returns all LinkedIn URLs delivered across ALL of this user's projects.
- `cross_project_urls` = `global_urls` minus `project_deliveries` — LinkedIn URLs the user has seen on other projects.

All comparisons should use a normalized `linkedin_url` (strip trailing slash, lowercase, strip query strings).

### Phase 2 — Bucket the ranked candidate list

Take the scored, filtered, ranked candidate list from step 4 of `run-sourcing-batch`. Iterate in score order. For each candidate C:

```
if C.linkedin_url in project_deliveries:
    drop                                # hard exclusion — never repeat on same project
elif C.linkedin_url in cross_project_urls:
    bucket = eligible_repeat
else:
    bucket = brand_new
```

**Note on burned state:** The burned flag is tracked via the global URL list. A candidate who has been delivered on two different projects (once as original, once as cross-project repeat) will appear in the global set and be ineligible as a repeat again. Track explicit burn state in `config.json` per-project until a dedicated `burned` column is added to `sourcing_deliveries`.

### Phase 3 — Assemble the batch

1. Sort `brand_new` by score descending. Take up to N candidates.
2. If `len(brand_new) < N` AND `eligible_repeat` is non-empty:
   - Pick the single highest-scoring `eligible_repeat` candidate.
   - Add it to the batch.
   - Mark this candidate as burned in the project's `config.json`.
3. If `len(brand_new) == N`, do NOT pull from `eligible_repeat`. Prefer a batch of all-new over including a repeat unnecessarily.
4. Never include more than one repeat per batch — even if `brand_new` is small and `eligible_repeat` has many candidates.
5. If the final batch is empty, post an empty-batch note to Slack (see `run-sourcing-batch` step 7) and exit.

### Phase 4 — Post to Slack

Claude posts profile cards to the Slack channel. The Slack bot creates `sourcing_deliveries` rows automatically in Supabase after each card is posted. The bot sets `was_repeat` on the delivery record.

Claude does NOT write delivery records, candidate records, or burned flags to Supabase directly. The bot handles all persistence.

## Edge cases

### The user has zero prior history

No deliveries for this user. Every sourced candidate is brand_new. Standard batch assembly.

### A candidate appears in multiple archetype searches within one run

Dedupe by `linkedin_url` before bucket assignment. One candidate, one bucket assignment.

### A candidate's LinkedIn URL format varies across runs

Normalize aggressively: strip `?` query strings, trailing slashes, case. If LinkedIn returned the same person via two URLs (rare but possible), dedupe by public identifier extracted from the URL path.

## Why these specific rules?

1. **Never repeat on same project** — protects the reviewer's trust. A tool that shows the same person twice loses credibility instantly.
2. **Cross-project repeats permitted once** — reflects real life: a great person for one search sometimes fits a second differently, but not a third. We'd rather give the user one chance to reconsider than hide them forever.
3. **Max one repeat per batch** — keeps the signal fresh. A 25-card batch where 5 are rehashes is demoralizing to review.
4. **Burning is terminal** — once the user has had two chances, we're done. Their attention is finite.

Do not soften these rules without a user-facing conversation. They're the contract — and they apply uniformly across every use case (recruiting, GTM, investment, LP, advisor, other).
