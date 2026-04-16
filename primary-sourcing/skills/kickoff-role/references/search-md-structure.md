# SEARCH.md Structure and Authoring Guide

SEARCH.md is the role's evolving brain. It's written at kickoff, rewritten by feedback signals over time, and read by every sourcing batch to ground search parameter generation.

The skeleton lives at `${CLAUDE_PLUGIN_ROOT}/templates/role/SEARCH-TEMPLATE.md`. This reference describes what each section means and how to fill it at kickoff.

## Authoring principles

1. **Specific beats generic.** Real company names, real titles, real exclusion reasons. If you find yourself writing "candidates with strong leadership skills," delete and replace with something scorable.
2. **Scorable beats vague.** Every must-have should be something an AI-scored ranking can evaluate from a LinkedIn profile.
3. **Cite the source.** If a criterion came from a HM meeting, say so inline ("per HM sync 2026-03-28"). This keeps the criterion honest as the search evolves.
4. **Archetypes over checkboxes.** People don't fit checkbox criteria — they fit shapes. Two or three well-drawn archetypes out-sources a 20-item must-have list.

## Section-by-section

### Title line
`# Search: {Role} at {Company}`

### Role Summary
One or two paragraphs. Why this hire matters, what the person does day one, what "good" looks like in 12 months. Not a job description — a sourcing north star.

### Target Profile

Four subsections, in this order:

- **Must-Haves** — If missing, candidate is a hard no.
- **Strong Preferences** — Usually present in winning candidates. Can be traded away for exceptional signals elsewhere.
- **Nice-to-Haves** — Bonus.
- **Dealbreakers** — Any of these present, candidate is a hard no. Keep this short and ruthless.

### Search Parameters

These map directly to structured LinkedIn search input. Concrete, specific, mappable.

- **Titles to Target** — exact strings, multiple variants.
- **Target Companies / Company Types** — named companies or clear archetypes ("Series B-D SaaS companies with $50-200M ARR").
- **Seniority Range** — map to LinkedIn seniority IDs where possible (Director=220, VP=300, CXO=310, etc.).
- **Locations** — explicit cities or regions. Note remote tolerance.
- **Industries** — specific verticals, not generic categories.
- **Years of Experience** — a range, not a floor.

### Archetypes That Resonate

Two or three named archetypes with a paragraph each. Name them plainly:

- **Archetype 1: "The ProCore Graduate"** — Enterprise AE at Procore, Autodesk, or PlanGrid for 3-6 years, 150%+ quota…
- **Archetype 2: "Capital Markets Crossover"** — Ex-investment banking or CRE debt broker who moved into a proptech sales role…

Archetypes do most of the work in sourcing — Apify searches are keyed on them.

### Anti-Patterns (What to Avoid)

Patterns from candidates that will get rejected. Be honest. "VP-level candidate looking for a leadership role — this is an IC seat." "Mortgage / retail banking backgrounds — wrong capital markets corner."

### Change Log

Dated entries, newest on top, describing what changed and why.

```
## Change Log
- 2026-04-15: Initial search criteria from kickoff (Logan Nash, source: HM sync 2026-04-14 + founder email thread).
```

Every downstream brain update appends to this. Never rewrite history.

## Length target

A good kickoff SEARCH.md lands between 400 and 900 words. Shorter than 400 usually means it's too generic; longer than 900 usually means it's padded. The goal is a document a human skimmer can absorb in 90 seconds.
