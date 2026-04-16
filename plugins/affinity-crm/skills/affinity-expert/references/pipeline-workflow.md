# Pipeline Workflow Guide

Step-by-step workflows for common deal pipeline operations. Follow these sequences to avoid errors and handle edge cases correctly.

## Workflow 1: Log a new company to the pipeline

**Trigger:** User says "log X", "add X to the pipeline", "track X", "new deal with X".

### Happy path

```
1. log_company(name: "Acme Inc", status: "First Meeting")
2. Response: success=true → done
```

### With full context

```
1. get_current_user() → extract user_id
2. get_status_options() → confirm valid statuses (skip if the status value is already known)
3. log_company(
     name: "Acme Inc",
     domain: "acme.com",
     status: "First Meeting",
     round: "Series A",
     segment: "B2B",
     owner_id: {user_id from step 1},
     note: "Met at TechCrunch Disrupt, strong product-market fit."
   )
```

### When company isn't found

```
1. log_company(name: "Stealth AI Co") → not_found: true
2. Ask user: "Not found in Affinity's database. Create it?"
3. If yes: log_company(name: "Stealth AI Co", domain: "stealthai.com", create_if_not_found: true)
```

### When multiple companies match

```
1. log_company(name: "Acme") → disambiguation: 3 matches
2. Present matches: Acme Inc (acme.com), Acme Labs (acmelabs.io), Acme Corp (acmecorp.com)
3. User picks "Acme Inc" → company_id: 123
4. log_company(company_id: 123, status: "First Meeting")
```

### When company is already on the pipeline

```
1. log_company(name: "Acme Inc") → exists_on_pipeline, current_status: "First Meeting"
2. Tell user: "Acme Inc is already on the pipeline with status 'First Meeting'."
3. If user wants a change: update_company_status(company_id: 123, status: "Active Deal")
```

## Workflow 2: Update a deal's status

**Trigger:** "Move X to Active Deal", "Mark X as Passed", "Update status of X".

### Happy path

```
1. update_company_status(company: "Acme Inc", status: "Active Deal")
2. Response: success=true, previous_status, new_status → done
```

### With a note

```
1. update_company_status(
     company: "Acme Inc",
     status: "Active Deal",
     note: "Term sheet sent. Expecting response by Friday."
   )
```

### When company isn't on the pipeline

```
1. update_company_status(company: "NewCo") → "not on the Master Deal Pipeline"
2. Tell user: "NewCo isn't on the pipeline yet. Want me to log it?"
3. If yes: log_company(name: "NewCo", status: "Active Deal")
```

### When status is invalid

```
1. update_company_status(company: "Acme", status: "In Progress") → invalid, shows available options
2. Present the valid statuses to the user
3. User picks "Active Deal"
4. update_company_status(company: "Acme", status: "Active Deal")
```

## Workflow 3: View recent deals

**Trigger:** "Show me my deals", "What's in my pipeline", "Recent deals".

### Standard flow

```
1. get_current_user() → user_id: 12345
2. get_recent_deals(owner_id: 12345)
3. Present deals table + summary statistics
```

### With pagination

```
1. get_current_user() → user_id: 12345
2. get_recent_deals(owner_id: 12345, limit: 50)
3. If has_more: get_recent_deals(owner_id: 12345, limit: 50, offset: 50)
```

### For a specific team member

```
1. get_owners(term: "Sarah") → find Sarah's ID
2. get_recent_deals(owner_id: {sarah_id})
```

## Workflow 4: Find and assign deal owners

**Trigger:** "Assign this deal to X", "Who owns deals", "Set me as owner".

### Set self as owner on new deal

```
1. get_current_user() → user_id: 12345
2. log_company(name: "NewCo", owner_id: 12345)
```

### Find a team member to assign

```
1. get_owners(term: "Sarah")
   → owners: [{ id: 200, firstName: "Sarah", lastName: "Chen", email: "sarah@primary.vc" }]
2. log_company(name: "NewCo", owner_id: 200)
```

### List all potential owners

```
1. get_owners()  → returns all internal team members sorted alphabetically
```

## Common error patterns and how to handle them

### "Either 'company' (name/domain) or 'company_id' is required"
Neither `name`/`company` nor `company_id` was provided. Include one and retry.

### "owner_id is required. Call get_current_user..."
`get_recent_deals` needs an `owner_id`. Call `get_current_user` first, extract `user_id`, pass as `owner_id`.

### "Status 'X' not found. Available options: ..."
The status string doesn't match any dropdown option. It IS case-insensitive, but must otherwise match exactly. Present the available options to the user.

### "Company is not on the Master Deal Pipeline. Use log_company to add it first."
The company exists in Affinity but hasn't been added to the pipeline. Use `log_company` instead of `update_company_status`.

### Disambiguation response with matches
Multiple companies matched a name search. Present the options with IDs, names, and domains. Let the user pick. Retry with the chosen `company_id`.

### Round or segment warnings
Non-fatal. The company was logged successfully, but the round or segment value didn't match a dropdown option. Relay the warning to the user.
