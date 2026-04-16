# Context Intake — How to Talk to the User

The plugin is only as good as the context gathered at kickoff. Skill quality failures almost always trace back to bad intake: wrong use case picked, misread project subject, missed a dealbreaker the user mentioned in passing.

This reference is the playbook for how to extract context from a teammate at scaffold time and at role kickoff. Every question in this plugin's skills should honor these principles.

## Principles

### 1. Ask; don't guess

If the user's message is ambiguous, ask. Asking one more question is almost always cheaper than running a wrong flow that has to be undone.

Examples:

- User says "set up sourcing for Acme" — is that recruiting, sales sourcing for Acme's GTM team, or investment sourcing where Acme is the target? **Ask.**
- User says "kick off a role" but the Project is set up for investment sourcing — do they mean an investment search? Or did they switch contexts? **Ask.**
- User gives a role title like "founder" — for recruiting that's unusual (we don't hire founders); they might mean investment sourcing. **Ask.**

### 2. Always use AskUserQuestion, never plain text

AskUserQuestion surfaces a clean UI for the user to click through. Plain-text prompts feel like homework. In this plugin, **every** user-facing question that could be a multiple-choice should go through AskUserQuestion. The only exceptions are:

- Open-ended free-text that genuinely has no finite answer set (e.g. "what's the role's one-sentence north star?").
- Confirmations where the answer is yes/no and the question is already in context.

AskUserQuestion always offers a free-text "Other" fallback, so even tight option lists don't lock the user out.

### 3. Pre-populate options from context you already have

Don't ask blind questions. If Affinity is connected and the user said "Acme" — look up Acme in Affinity first, confirm we're talking about the same company, and populate the portfolio company field from the Affinity record. Blank AskUserQuestion prompts leak effort onto the user.

### 4. Ask in batches, not one by one

AskUserQuestion supports multiple questions in one call. Batch related questions together — users fill them in seconds when they see all four on screen, and burn time context-switching if you ask them sequentially.

Keep batches to 4 questions max. More than 4 feels like a form.

### 5. Adapt every subsequent question based on what you've already learned

If the user said "investment sourcing" in batch 1, don't ask "what's the seniority level" in batch 2. That's a recruiting question. Adapt.

This means:

- Use case selection is always the **first** question.
- Subsequent questions branch per use case.
- Within a use case, later questions should reference earlier answers ("You said you want Series A companies — should the search be limited to companies with $1-5M ARR, or broader?").

### 6. Synthesize back before committing

After gathering context and asking follow-ups, **always** play back what you understood before doing anything irreversible (creating a Slack channel, creating a scheduled task, writing SEARCH.md). The user gets a last chance to correct misinterpretations cheaply.

Format:

> Here's what I captured. Confirm or correct:
> - Use case: **Investment sourcing**
> - Project subject: **Vertical SaaS ops tools — Series Seed-A**
> - Search anchor: **Repeat founders in infra-SaaS with prior enterprise sales experience**
> - Schedule: **Weekly Monday 9am**
> - Avoiding: companies already in Affinity
>
> Proceed? [Yes / Adjust]

If the user says "adjust", re-ask the specific thing rather than re-running the whole flow.

### 7. Never flatten open-ended input prematurely

Users often drop context in free-text that doesn't fit a question you asked. Preserve that text verbatim in the Project's `project-context/` folder or the role's `KICKOFF.md`. Don't summarize it away until you've confirmed the summary is accurate.

### 8. When you don't know, say so

If a question is genuinely unanswerable from context and the user hasn't answered it yet, **say "I don't know yet — let's leave this empty and you can fill it in later"** rather than making up a plausible-sounding value. SEARCH.md tolerates empty sections.

## Specific intake patterns

### Picking the use case

In `start-sourcing-project`, the use case is the hinge. Every downstream decision depends on it. Ask via AskUserQuestion with these options:

```
What is this Project for?
  ( ) Recruiting for a portfolio company — hire execs or ICs for a PortCo
  ( ) Sales / GTM sourcing for a portfolio company — find buyer prospects for a PortCo's sales motion
  ( ) Investment sourcing — find founders or companies to invest in
  ( ) Fund / LP sourcing — find LPs, co-invest funds, or ecosystem partners
  ( ) Advisors, board members, or subject matter experts
  ( ) Other — I'll describe what I'm sourcing
```

If the user picks "Other", capture their description verbatim and note in the marker file that downstream skills should operate in a best-effort mode with generic prompts.

### Naming the Project subject

After use case, ask for the Project subject. The phrasing of the question varies by use case. Examples:

- **Recruiting**: "Which portfolio company is this Project for?"
- **GTM sourcing for PortCo**: "Which portfolio company's GTM motion are you sourcing for?"
- **Investment sourcing**: "What's the investment thesis or theme? (A sector, stage, or archetype — doesn't need to be precise yet.)"
- **Fund/LP sourcing**: "What's this fundraising or ecosystem effort? (E.g. 'Fund V LPs — family offices', 'Emerging fund GPs we want to co-invest with'.)"
- **Advisor/board**: "Who are you sourcing for? (A portfolio company's board, Primary's own advisor bench, or something else.)"

If Affinity is connected AND the answer is a portfolio company, auto-validate: look up the name in Affinity, confirm match, pull company metadata into `project-context/`.

### Gathering initial context

Run the parallel MCP search (Granola, Slack, Notion, Drive, Gmail, Affinity) **before** asking any follow-up questions. That way your questions can reference what you found ("I see an HM sync from March 28 where the founder mentioned wanting a specialist — is this search still around that shape?") rather than asking blind.

### When the user's answer doesn't fit the options

AskUserQuestion includes a free-text field. If the user types something unusual, don't force-fit it to one of the presets. Capture the raw text, store it in the context, and either:

1. Branch your follow-ups off the raw text (treating it as "Other"), or
2. Ask a clarifying follow-up: "I'm not sure which bucket that fits — could you say more?"

## Red flags

Watch for these during intake; they mean you should slow down and clarify:

- User's phrasing mixes use cases ("sourcing a CEO and also LPs for the same company"). Split into two Projects.
- User gives a Project name with no company or theme attached ("just call it 'Q2 search'"). Press for the anchor — "what are we searching for?".
- User picks "Other" but describes something that matches a known use case. Gently offer: "This sounds like investment sourcing — want to use that preset?"
- User's role title is a verb phrase ("need someone who can scale our team"). That's a mission, not a role. Extract the title underneath.

## Intake at role kickoff (vs. project scaffolding)

Role kickoff intake is shorter because the Project scaffold already captured use case + subject + context. The role kickoff focuses on: role title, role-specific context, archetype, dealbreakers, schedule. Don't re-ask Project-level questions.

Always confirm the use case and Project subject carried forward correctly before asking role-specific questions.
