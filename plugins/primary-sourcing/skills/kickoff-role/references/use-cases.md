# Use Cases Reference

The plugin's primary use case is recruiting for Primary portfolio companies, but the underlying workflow (scaffold a Project, run targeted LinkedIn searches with dedup, collect feedback, learn over time) generalizes to any domain where Primary teammates need to source people repeatedly and thoughtfully.

This reference lists every supported use case, what the anchor "Project subject" looks like for each, and how the kickoff flow and scoring adapt. All skills in this plugin read `use_case` from `.primary-sourcing-project` and branch accordingly.

## Supported use cases

### 1. `recruiting-portco` — Executive recruiting for a portfolio company

**Headline use case.** A Primary Talent partner (Annabelle today, the whole team tomorrow) sources executives or ICs for a portfolio company's open role.

- **Project subject**: portfolio company name (e.g. "LightTable", "Chief", "Acme Robotics").
- **Search anchor**: the role (e.g. "Founding AE", "Director of GTM Engineering").
- **Lead definition**: individual candidates.
- **Context sources that matter most**: Granola (HM syncs), Slack (#pf-{company} channels), Google Drive (job briefs), Affinity (founder relationship strength).
- **Follow-up questions**: archetype, candidate paths, industry depth, location, pipeline gap, avoid-list, sourcing cadence, weekly digest.
- **Scoring rubric**: `scoring-rubric.md` § recruiting.

### 2. `gtm-sourcing-portco` — GTM/sales sourcing for a portfolio company

A Primary GTM team member (or the PortCo CEO via us) sources prospects or buyer personas for a PortCo's sales motion.

- **Project subject**: portfolio company name (same as recruiting).
- **Search anchor**: the buyer persona or deal profile (e.g. "Head of FP&A at mid-market SaaS", "VP Procurement at construction firms 500-2000 employees").
- **Lead definition**: individual prospects at target accounts.
- **Context sources that matter most**: PortCo's ICP doc, Slack threads with the PortCo team, outreach templates in Drive.
- **Follow-up questions**: target account size, persona, geography, account tier, existing customer overlap (avoid double-touch), outreach cadence, weekly report opt-in.
- **Scoring rubric**: `scoring-rubric.md` § gtm-prospect.

### 3. `investment-sourcing` — Founder/company sourcing for the fund

A Primary investor sources potential founders or companies to invest in, around a thesis or sector.

- **Project subject**: the investment theme/thesis (e.g. "Vertical SaaS ops tools, Series Seed-A", "Climate infrastructure at industrial scale").
- **Search anchor**: the target company or founder archetype (e.g. "Repeat founder in infra-SaaS", "Early-stage proptech company with $1-5M ARR").
- **Lead definition**: founders or companies.
- **Context sources that matter most**: Affinity (deal pipeline), Notion (thesis docs), Granola (founder calls), Slack (#deal-flow channels).
- **Follow-up questions**: stage, geography, traction signals, check size, avoid list (already seen / passed), sourcing cadence, weekly report.
- **Scoring rubric**: `scoring-rubric.md` § investment.

### 4. `fund-lp-sourcing` — Funds or LPs for fundraising and ecosystem mapping

A Primary partner sources potential LPs, other fund GPs for co-invest relationships, or ecosystem partners.

- **Project subject**: the fundraising effort or ecosystem category (e.g. "Fund V LPs — family offices", "Emerging fund GPs we want to co-invest with").
- **Search anchor**: LP type, fund archetype, or role at a fund.
- **Lead definition**: individuals at LP orgs or GPs at other funds.
- **Context sources that matter most**: Affinity (existing relationships), Granola (LP meetings), Slack (#fundraising), Gmail.
- **Follow-up questions**: LP type/size, geography, mandate fit, relationship status filter, avoid list (already committed), outreach cadence.
- **Scoring rubric**: `scoring-rubric.md` § fund-lp.

### 5. `advisor-board-sourcing` — Advisors, board members, or subject-matter experts

A Primary teammate or PortCo sources advisors, board members, or SMEs around a specific expertise.

- **Project subject**: the portfolio company (if sourcing for a PortCo board) or the expertise theme (if sourcing for Primary's own advisor bench).
- **Search anchor**: the expertise or title (e.g. "Former CFO at public fintech", "Cybersecurity CISO with healthcare exposure").
- **Lead definition**: individuals.
- **Context sources**: Granola, Slack, Affinity, LinkedIn.
- **Follow-up questions**: expertise area, seniority, commitment level expected (board seat vs. informal advisor), geography, avoid list.
- **Scoring rubric**: `scoring-rubric.md` § advisor.

### 6. `other` — Custom / catchall

Anything that doesn't fit the above. The user describes the sourcing goal in their own words. The skill adapts as best it can — asks open-ended questions, uses a generic scoring rubric. Flag to the user that this use case isn't optimized yet and offer to submit feedback.

## How skills adapt per use case

### `start-sourcing-project`

- Step 2 — Ask the user which use case applies. Offer the six above as AskUserQuestion options. If they pick recruiting, also ask for the portfolio company name. If they pick investment sourcing, ask for the thesis/theme. Etc.
- Step 3 — Context gathering parallelism stays the same, but query construction changes (e.g. Affinity takes priority for investment sourcing; Drive takes priority for recruiting).
- Step 5 — The CLAUDE.md template loaded into the Project root is the same file, but it contains use-case-aware instructions.
- Step 5 — The `.primary-sourcing-project` marker file includes `use_case`. Downstream skills read it.

### `kickoff-role`

- Step 4 — The two AskUserQuestion batches adapt per use case. Each use case has its own question set (see above). Keep to 4 questions per batch.
- Step 5 — SEARCH.md generation uses the generic template but the model is prompted with use-case-specific guidance about how to fill it.
- Step 10 — Scheduled task prompts are identical regardless of use case (the skills themselves hold the logic).

### `run-sourcing-batch`

- Step 4 — Scoring uses the rubric section matching the role's use case.
- Step 5 — Dedup rules are identical per use case (the contract is universal).
- Step 7 — Slack card format is identical regardless of use case.

### `run-weekly-summary`

- Step 4 — The analysis prompt mentions the use case so the model frames patterns appropriately ("Yes trends" → for investment sourcing, "Deals we liked" might be more natural phrasing).

## Terminology

Internally we use generic terminology so the code is consistent:

| Generic term     | Recruiting                    | GTM sourcing         | Investment sourcing    | Fund/LP            | Advisor/board        |
| ---------------- | ----------------------------- | -------------------- | ---------------------- | ------------------ | -------------------- |
| Project subject  | Portfolio company             | Portfolio company    | Investment thesis      | Fundraising theme  | PortCo or theme      |
| Search           | Role                          | Buyer persona        | Target archetype       | LP/GP archetype    | Advisor archetype    |
| Lead             | Candidate                     | Prospect             | Founder / company      | LP / fund GP       | Advisor              |
| SEARCH.md        | Role brief                    | ICP doc              | Thesis doc             | Ideal LP profile   | Ideal advisor profile|
| Slack channel    | `sourcing-{role_slug}`        | Same                 | Same                   | Same               | Same                 |

The user's vocabulary varies — "role", "search", "deal", "thesis" — but the plugin normalizes everything to "search" internally.

## When a use case is added

New use cases go through this playbook:

1. Add a section above.
2. Add a scoring rubric in `scoring-rubric.md`.
3. Add an AskUserQuestion batch in `kickoff-role/SKILL.md` branching.
4. Bump the plugin version (MINOR — new functionality).

Keep the plugin flexible but not sprawling — if a proposed use case overlaps heavily with an existing one, fold it in as a variation rather than a new top-level use case.
