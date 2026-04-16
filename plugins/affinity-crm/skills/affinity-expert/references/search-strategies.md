# Search Strategies

How to choose the right search tool and get the best results from Affinity.

## Company search tools

### `search_company` (custom — use this by default)

The primary company search tool. Searches Affinity's database by name or domain and enriches each result with whether it's on the Master Deal Pipeline.

**When to use:**
- Any time you're looking up a company by name or domain.
- When checking if a company is in the pipeline.
- Before `log_company` to see if the company already exists.

**Tips:**
- Domain searches are more precise than name searches. If you have a domain, use it.
- The search is fuzzy on names — "Acme" will match "Acme Inc", "Acme Labs", etc.
- Pipeline enrichment is batched with a concurrency limit of 5. For large result sets, the first 5 are enriched first.
- If pipeline enrichment fails for a specific company, that company gets `onDealPipeline: null` (not `false`). Treat `null` as "unknown" rather than "not on pipeline".

**Common patterns:**
```
search_company(term: "acme.com")          # By domain — most precise
search_company(term: "Acme Inc")          # By name — may return multiple matches
search_company(term: "Acme", include_pipeline_status: false)  # Skip enrichment for speed
```

### `semantic_search` (upstream — for fuzzy/natural language)

Natural language search across companies. Use when you don't know the exact name or domain.

**When to use:**
- The user describes a company vaguely: "that AI company from the YC batch"
- The user gives a partial or possibly misspelled name.
- Standard search returned no results and you want to try a broader match.

**Limitations:**
- Companies only — does not search persons.
- Beta feature — may change.
- No pipeline enrichment. If you get results, follow up with `get_company` for pipeline status.

**Pattern:**
```
semantic_search(query: "infrastructure startup in SF that does GPU orchestration")
```

### `search_companies` — EXCLUDED

This upstream tool is excluded from the MCP. The custom `search_company` replaces it with pipeline enrichment. Do not try to call `search_companies` — it won't exist.

## Person search tools

### `search_persons` (upstream)

Search for people by name or email address, with optional interaction date filtering.

**When to use:**
- Looking up a specific person.
- Finding someone by email address.
- Checking if a person is in Affinity.

**Tips:**
- Email searches are most precise. "jane@acme.com" will return an exact match if the person exists.
- Name searches are fuzzy. "Jane" will return many results; "Jane Smith" narrows it.
- Use interaction date filters to find people you've been in contact with recently.

**Pattern:**
```
search_persons(query: "Jane Smith")
search_persons(query: "jane@acme.com")
search_persons(query: "Smith", min_first_email_date: "2025-01-01")  # Recent contacts only
```

### `get_person_info` (upstream)

Get comprehensive profile data for a specific person by ID.

**When to use:**
- After a search, when you need full details on a specific person.
- When you have a person ID from another tool's response (e.g., from `data.company.personIds`).

**What it returns:**
- Name, emails, phone numbers
- Job titles and current organization
- LinkedIn URL
- Connected lists
- Interaction dates (first/last email, first/last meeting, etc.)
- Location

### `get_company_info` — EXCLUDED

This upstream tool is excluded. The custom `get_company` replaces it. Do not try to call `get_company_info`.

## Choosing the right strategy

### "Is X in our pipeline?"
```
search_company(term: "X", include_pipeline_status: true)
→ Check onDealPipeline in results
```

### "What do we know about X?"
```
1. search_company(term: "X") → get company_id
2. get_company(company_id: ..., include_pipeline_status: true, include_notes: true)
```

### "Who is X?" (person)
```
1. search_persons(query: "X") → get person_id
2. get_person_info(person_id: ...)
3. Optionally: get_relationship_strengths(person_id: ...)
```

### "Do we know anyone at X?" (company → people)
```
1. search_company(term: "X") → company has personIds
2. For key people: get_person_info(person_id: ...) for each
3. Optionally: get_relationship_strengths for each
```

### "I'm not sure of the name, something like..."
```
1. semantic_search(query: "natural language description")
2. If match found: get_company(company_id: ..., include_pipeline_status: true)
```

### "What meetings have we had with X?"
```
1. Search for the entity first (company or person)
2. get_meetings_for_entity(entity_type: "company"|"person", entity_id: ...)
```

## Tips for better results

1. **Try domain first.** If you have a company URL, extract the domain and search by that. It's almost always a unique match.
2. **Narrow with IDs.** Once you have an entity ID from search, always use it for subsequent calls. Avoid repeated name-based searches.
3. **Parallel lookups.** If you need both company details and meeting history, you can call `get_company` and `get_meetings_for_entity` in parallel — they're independent.
4. **Notes tell the story.** When researching a company, always check notes (`include_notes: true`). They contain meeting takeaways, deal context, and relationship history that profiles alone don't capture.
5. **Relationship strengths for warm intros.** Before reaching out to someone, check `get_relationship_strengths` to find which team member has the strongest connection. Always route through the warmest path.
