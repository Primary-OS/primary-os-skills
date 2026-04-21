# SEARCH.md template

Use this structure for every `roles/{slug}/SEARCH.md`. Section names stay generic across use cases; content is use-case-specific. Cite the source of every criterion inline (e.g. "per HM sync 2026-04-15", "per Notion thesis doc", "per user during kickoff").

---

```markdown
# {Search Title}

**Subject:** {subject_name}
**Use case:** {use_case}
**Owner:** {owner_name}
**Created:** {ISO date}

## Summary

One paragraph: what this search is for, why it matters, and what a "great" outcome looks like.

## Archetypes

The distinct shapes of fit to look for. For each archetype:

### Archetype 1 — {short name}

- **Profile:** 1–2 sentences describing this shape.
- **Signals:** specific titles, companies, experiences, or patterns that indicate this archetype.
- **Priority:** high / medium / low.

### Archetype 2 — {short name}

- …

(2–4 archetypes total. If you can only think of one, dig deeper with the user before writing.)

## Must-haves

Non-negotiables. Any candidate / founder / LP / advisor missing these is auto-rejected.

- {must-have 1} — *source: …*
- {must-have 2} — *source: …*

## Dealbreakers

Hard exclusions. Do not surface anyone matching these.

- {dealbreaker 1} — *source: …*
- {dealbreaker 2} — *source: …*

## Search Parameters

LinkedIn-searchable fields used by the Apify query.

- **Titles:** [list of current/past title variations]
- **Seniority:** [IC / Lead / Manager / Director / VP / C-level / Founder]
- **Locations:** [cities, regions, or "remote OK"]
- **Company size:** [employee count ranges]
- **Industries:** [LinkedIn industry tags]
- **Exclusions:** [companies or industries to filter out]

## Stakeholders

Who's involved and how they interact with this search.

- **{Name}** — {role, e.g. "hiring manager", "thesis owner", "LP relationship lead"}. {1-line note.}

## Notes

Anything else not captured above: context, nuances, prior attempts, related searches.

## Change Log

Append an entry every time criteria shift. Each entry:

- **YYYY-MM-DD** — {what changed and why}. *Signal source: {Slack message / Granola meeting / user direction}.*
```
