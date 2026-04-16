---
name: start-sourcing-project
description: Scaffolds a Cowork Project for a Primary sourcing effort of any type — recruiting for a portfolio company, GTM / sales sourcing, investment sourcing, LP/fund outreach, advisor sourcing, or custom. Use when the user says things like "set up sourcing for {company}", "start a sourcing project", "initialize this project for recruiting", "new sourcing project for {theme}", "prep this project for a new investment thesis", "set up an LP outreach project", or any phrasing that implies setting up a whole Project for repeated sourcing (as distinct from kicking off one specific search, which uses the kickoff-role skill). Captures the use case up front so every downstream skill adapts its prompts, context gathering, and scoring rubric.
---

# Scaffold a Primary sourcing Project

Prepare a fresh Cowork Project as a sourcing workspace. The Project is the team-shared container for one anchor — typically a portfolio company (for recruiting or GTM work), an investment thesis (for deal sourcing), a fundraising effort (for LP sourcing), or any other sourcing theme the team cares about. Individual searches happen *inside* this Project using the `kickoff-role` skill.

## When to use this skill

- The teammate just created a new Cowork Project and wants it prepared for ongoing sourcing.
- An existing empty Project needs to be converted into a sourcing workspace.
- The team is setting up a fresh anchor (company, thesis, fundraise, theme) and wants files and context in place before kicking off individual searches.

Do **not** use this skill to start a new search inside an already-scaffolded Project — that is `kickoff-role`.

## What this skill produces

After this skill completes, the Project workspace contains:

- `CLAUDE.md` at the root — project instructions that teach every Cowork conversation in this Project how to behave. Includes use-case-specific behavioral guidance.
- `project-context/project-brief.md` — synthesized context about the Project subject pulled from connected tools.
- `roles/` folder with a `.keep` file — populated as teammates kick off searches.
- `templates/SEARCH-TEMPLATE.md` — the template `kickoff-role` uses for each search's SEARCH.md.
- `.primary-sourcing-project` marker file — JSON capturing version, use case, subject, scaffold timestamp. Downstream skills read this to know how to behave.

## Procedure

Read `references/context-intake.md` fully before running this skill. Intake quality is the single biggest predictor of plugin success. Every principle there applies. All user-facing questions go through AskUserQuestion, never plain text.

### Step 1 — Detect the workspace

Confirm the current working directory is a Cowork Project workspace (use `pwd` via Bash or equivalent). If unclear, ask the user to confirm before writing any files.

### Step 2 — Identify the use case (critical)

This is the most important step. The use case selected here branches every downstream skill's behavior. See `references/context-intake.md § Picking the use case` for the exact options and phrasing.

Ask via AskUserQuestion:

> What is this Project for?
> - Recruiting for a portfolio company
> - Sales / GTM sourcing for a portfolio company
> - Investment sourcing (founders or companies)
> - Fund / LP sourcing
> - Advisors, board members, or subject matter experts
> - Other (I'll describe)

Persist the chosen `use_case` as one of: `recruiting-portco`, `gtm-sourcing-portco`, `investment-sourcing`, `fund-lp-sourcing`, `advisor-board-sourcing`, `other`.

See `${CLAUDE_PLUGIN_ROOT}/skills/kickoff-role/references/use-cases.md` for each use case's definition and expectations.

### Step 3 — Identify the Project subject

The subject depends on use case. See `references/context-intake.md § Naming the Project subject` for the exact phrasing per use case.

Examples:

- Recruiting / GTM: which portfolio company.
- Investment sourcing: thesis or theme (e.g. "Vertical SaaS ops tools — Series Seed-A").
- Fund/LP: fundraising effort or ecosystem category.
- Advisor/board: PortCo (if PortCo-specific) or theme.
- Other: user's own description, captured verbatim.

Capture:

- `subject_name` — human-readable.
- `subject_slug` — lowercased, hyphenated. Derive automatically; confirm if non-trivial.

If the use case is PortCo-centric and Affinity is connected, auto-validate the subject against Affinity. Surface the match confidence to the user — don't silently assume.

### Step 4 — Check MCP prerequisites

Check which MCPs are connected. Report status via a simple formatted list (see `references/mcp-prereqs.md` for formatting and which MCPs matter for which use cases).

Do not block scaffolding on missing MCPs, but call out anything required for the chosen use case so the user can address it before kicking off a search.

### Step 5 — Gather Project-level context (run in parallel)

Based on the use case, run relevant MCP queries **in parallel** (all in a single turn). Context sources vary per use case — see `${CLAUDE_PLUGIN_ROOT}/skills/kickoff-role/references/use-cases.md` for which tools matter most for each.

Across all use cases, the usual sources are: Granola, Slack, Notion, Google Drive, Gmail, Affinity. Use them when connected; skip when not.

Synthesize findings into a concise `project-context/project-brief.md` (~500 words max). Cite sources inline. If nothing is found or relevant MCPs aren't connected, write a minimal brief with subject name + a note that context is pending.

### Step 6 — Synthesize back and confirm before writing

Present to the user:

> Before I scaffold this Project, here's what I captured:
> - **Use case**: {use_case}
> - **Project subject**: {subject_name} (slug: {subject_slug})
> - **Initial context**: {one-line summary or "pending — no context found yet"}
>
> Looks right? [Yes / Adjust]

If they say "Adjust", re-ask the specific thing.

### Step 7 — Scaffold the files

Only proceed after confirmation. Write:

| Write to                              | Source                                                                                             | Notes                                                                                                     |
| ------------------------------------- | -------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- |
| `./CLAUDE.md`                         | `${CLAUDE_PLUGIN_ROOT}/templates/company-project/CLAUDE.md`                                        | Replace `{{USE_CASE}}`, `{{SUBJECT_NAME}}`, `{{SUBJECT_SLUG}}` placeholders.                              |
| `./project-context/project-brief.md`  | Synthesized in step 5                                                                              | Create folder if needed.                                                                                  |
| `./roles/.keep`                       | Empty file                                                                                         | Creates the folder.                                                                                       |
| `./templates/SEARCH-TEMPLATE.md`      | `${CLAUDE_PLUGIN_ROOT}/templates/role/SEARCH-TEMPLATE.md`                                          | Verbatim copy.                                                                                            |
| `./.primary-sourcing-project`         | JSON (see below)                                                                                   | Marker file.                                                                                              |

Marker file content:

```json
{
  "plugin_version": "0.2.0",
  "use_case": "{use_case}",
  "subject_name": "{subject_name}",
  "subject_slug": "{subject_slug}",
  "scaffolded_at": "{ISO-8601 UTC}",
  "scaffolded_by": "{current user email}"
}
```

### Step 8 — Handle re-scaffolding

If `.primary-sourcing-project` already exists:

- Same subject + same use case: inform the user and offer `kickoff-role` instead. Do not overwrite.
- Different subject OR different use case: warn the user — they may be in the wrong Project. Ask explicitly before overwriting anything.

### Step 9 — Confirm and explain the next step

Send a brief summary:

- Project scaffolded as a **{use_case}** workspace for **{subject_name}**.
- Files written (brief bullet list).
- MCP status (call out anything missing required for the use case).
- Next step: a natural-language hint like "Say 'kick off a new search for this Project' (or 'kick off a role', 'kick off a new deal', whatever fits your use case) to start your first search."

## Guardrails

- Never create Slack channels, Lovelace sourcing projects, or scheduled tasks at scaffold time. Those are per-search concerns for `kickoff-role`.
- Never write outside the current Cowork Project workspace.
- The use case chosen here is sticky — changing it later is disruptive. If there's any ambiguity in step 2, ask again rather than guess.
- The `.primary-sourcing-project` marker is the source of truth for downstream skills. Keep it well-formed JSON and never overwrite silently.
