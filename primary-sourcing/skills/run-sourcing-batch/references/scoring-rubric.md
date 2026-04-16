# Candidate Scoring Rubric (per use case)

Every raw profile gets a 1-10 score against the search's SEARCH.md. The rubric varies per use case — a "7" for recruiting means something different from a "7" for investment sourcing. Pick the section that matches the Search's `use_case` field.

Across all use cases: be ruthless. Most profiles from a broad LinkedIn search should score 1-3. The recruiter/investor/outreach operator's time is the bottleneck — a loose filter is worse than no filter.

## General prompt shape

Use this prompt skeleton, substituting the use-case-specific rubric in the bracketed block:

```
You are a precise sourcing scorer for Primary Venture Partners. Your job
is to protect the reviewer's time — only candidates with a HIGH likelihood
of being relevant should score above 6.

<search_criteria>
{full SEARCH.md content}
</search_criteria>

<candidates>
{numbered list of profile summaries}
</candidates>

[USE-CASE-SPECIFIC RUBRIC — pick one of the sections below]

Return ONLY valid JSON, no markdown fence:
[
  {"index": 0, "score": 8, "rationale": "1-2 sentences citing concrete profile details."},
  ...
]
```

## Per-use-case rubrics

### `recruiting-portco`

```
AUTOMATIC DISQUALIFIERS (1-2):
- Missing every "Must-Have" from SEARCH.md
- Triggers any "Dealbreaker"
- Wrong seniority (e.g. VP applying for IC seat)
- Outside approved locations

LOW FIT (3-4):
- Tangential background; no clear bridge to role archetypes
- Right title but entirely wrong industry

MODERATE FIT (5-6):
- Adjacent industry + right title, but missing key criteria
  (deal size, domain depth, stage fit)

STRONG FIT (7-8):
- Matches one of the named archetypes in SEARCH.md
- Checks most Must-Haves and several Strong Preferences
- Right location and seniority

EXCEPTIONAL FIT (9-10):
- Checks every Must-Have, most Strong Preferences, fits an archetype cleanly
- You'd stake your reputation on this as a real contender for the role
```

### `gtm-sourcing-portco`

```
AUTOMATIC DISQUALIFIERS (1-2):
- Wrong company size (outside the ICP band)
- Wrong buyer persona (not a decision-maker for this kind of purchase)
- Triggers any "Dealbreaker" (e.g. at a listed competitor's customer)
- Already in the PortCo's customer CRM (avoid double-touch)

LOW FIT (3-4):
- Right persona but wrong industry/vertical
- Right company size but wrong function

MODERATE FIT (5-6):
- In-ICP industry + right function, but seniority off or no buying signal
- Tangential buyer — would need to route to the right contact

STRONG FIT (7-8):
- In-ICP account + correct persona + reachable seniority
- Some buying signal (recent hire into the function, company growth signal)

EXCEPTIONAL FIT (9-10):
- In-ICP account + exact buyer persona + active buying signal
  (e.g. just posted a related job, recent fundraise)
- Plausibly closable within a normal sales cycle
```

### `investment-sourcing`

```
AUTOMATIC DISQUALIFIERS (1-2):
- Outside the thesis (wrong sector, wrong stage, wrong business model)
- Already passed by Primary (in Affinity as declined)
- Founder profile doesn't match the archetype (e.g. no technical depth for a DeepTech thesis)

LOW FIT (3-4):
- Adjacent sector; would require a meaningful thesis stretch
- Right stage but weak founder signal

MODERATE FIT (5-6):
- In-thesis sector + founder archetype reasonable, but unclear traction
- Right founder shape but company too early or too late

STRONG FIT (7-8):
- In-thesis sector + strong founder archetype match + reasonable traction signal
- The kind of company Primary would take a meeting with

EXCEPTIONAL FIT (9-10):
- Clear thesis fit + exceptional founder (repeat, credentialed, or rare domain expertise)
- The kind of company a partner would sprint to meet
```

### `fund-lp-sourcing`

```
AUTOMATIC DISQUALIFIERS (1-2):
- LP type outside the mandate (e.g. corporate LP when fund targets family offices)
- Already committed to Fund V (or previous fund — avoid re-pitch)
- Explicitly declined Primary in the past
- Geography/regulatory mismatch

LOW FIT (3-4):
- Right LP type but wrong size/check-writing band
- Adjacent mandate but not a clean fit

MODERATE FIT (5-6):
- Right LP type + size, but unclear appetite for emerging managers / this vintage
- Relationship is cold; no warm intro path

STRONG FIT (7-8):
- Right LP type + size + mandate alignment + relationship is reachable (2nd-degree or closer)
- History of investing in similar funds

EXCEPTIONAL FIT (9-10):
- Exact mandate fit + strong relationship path + known to write quickly
- LP a GP would prioritize in a raise
```

### `advisor-board-sourcing`

```
AUTOMATIC DISQUALIFIERS (1-2):
- Missing the required expertise entirely
- Active competitor (would create a conflict of interest)
- Can't legally advise (active at a public competitor, for example)
- Outside reasonable geography if in-person is required

LOW FIT (3-4):
- Expertise tangential; would need a stretch to be useful
- Right function but wrong industry context

MODERATE FIT (5-6):
- Core expertise fit but unclear availability or past advisor experience
- Right industry but wrong functional depth

STRONG FIT (7-8):
- Exact expertise + past advisor/board track record + reachable
- Reputation would add signal alongside the advice

EXCEPTIONAL FIT (9-10):
- Best-in-class expertise + known advisor mentality + clean intro path
- The kind of person a founder would proudly list
```

### `other`

For custom use cases, use a generic rubric asking the model to score against SEARCH.md's Must-Haves, Strong Preferences, Nice-to-Haves, and Dealbreakers directly. No use-case-specific calibration — score 7+ means "clearly matches the SEARCH.md intent."

## Shared rules across all rubrics

- Be harsh. 60-75% of profiles should score 1-3.
- Rationales must cite **specific profile details** (company name, tenure, title transition). Never "seems like a good fit."
- After scoring: drop everything below 6, sort descending, cap at 25.
- Log the score distribution and top 3 names/scores for every batch — these land in the weekly summary.

## When to revisit the calibration

If a search is consistently producing all-1 or all-9 scores, the rubric may be miscalibrated for that use case. Surface the issue to the user (in the Slack channel note after the sourcing run) and suggest reviewing the SEARCH.md.
