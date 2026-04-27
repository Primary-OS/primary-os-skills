---
name: deploy
description: >
  Trigger and manage Lovelace deployments to staging and production.
  Use when the user asks to: deploy to staging, deploy to production,
  trigger a deploy, promote to production, manage CI variables,
  add a secret, update an environment variable, create a pipeline schedule,
  "deploy the latest", "push to prod", "add a CI variable",
  "what variables are set", or any task involving triggering deployments
  or managing CI/CD configuration for the Lovelace monorepo.
---

# Deploy & CI/CD Configuration

Trigger deployments, manage CI/CD variables, and configure pipeline schedules for the Lovelace monorepo.

## Required context

- **Always load**: `${CLAUDE_PLUGIN_ROOT}/skills/pipeline/references/api-access.md` — GitLab API URL, token, project ID
- **For deploy troubleshooting**: `${CLAUDE_PLUGIN_ROOT}/skills/deploy/references/deploy-patterns.md` — service-specific deploy details and known gotchas

## Triggering deployments

### Staging

Staging deploys automatically on push to main. Only services with changed files deploy (path-filtered).

To force a staging deploy of everything, trigger a new pipeline:
```bash
curl -s --request POST --header "PRIVATE-TOKEN: $TOKEN" \
  "$GITLAB_URL/api/v4/projects/1/pipeline?ref=main"
```

**Note**: Path-filtered jobs only run if matching files changed in the triggering commit. A manually triggered pipeline runs all jobs since `rules:changes` evaluates as true for manual triggers.

### Production

Production deploys are restricted to Logan via protected environments. To deploy:

1. Find the `migrations-production` job in any main pipeline
2. Click the play button (or use the API below)
3. The rest cascades automatically: migrations → API + slackbot → MCP + web

```bash
# Find the migrations-production job ID
curl -s --header "PRIVATE-TOKEN: $TOKEN" \
  "$GITLAB_URL/api/v4/projects/1/pipelines/<PIPELINE_ID>/jobs" | python3 -c "
import json, sys
for j in json.loads(sys.stdin.read()):
    if j['name'] == 'migrations-production':
        print(f'Job ID: {j[\"id\"]} — status: {j[\"status\"]}')
"

# Trigger it
curl -s --request POST --header "PRIVATE-TOKEN: $TOKEN" \
  "$GITLAB_URL/api/v4/projects/1/jobs/<JOB_ID>/play"
```

## Managing CI/CD variables

Variables live at two levels:
- **Group-level** (Primary-OS, group ID 6): shared deploy tokens (Railway, Vercel, Cloudflare) — inherited by all projects in the group
- **Project-level**: project-specific config (Supabase, Slack, SMTP, etc.)

### List variables

```bash
# Group-level (shared deploy tokens)
curl -s --header "PRIVATE-TOKEN: $TOKEN" \
  "$GITLAB_URL/api/v4/groups/6/variables" | python3 -c "
import json, sys
for v in json.loads(sys.stdin.read()):
    print(f'{v[\"key\"]:<35} masked={v[\"masked\"]} protected={v[\"protected\"]}')
"

# Project-level (Lovelace-specific)
curl -s --header "PRIVATE-TOKEN: $TOKEN" \
  "$GITLAB_URL/api/v4/projects/1/variables" | python3 -c "
import json, sys
for v in json.loads(sys.stdin.read()):
    print(f'{v[\"key\"]:<35} masked={v[\"masked\"]} protected={v[\"protected\"]}')
"
```

### Add a variable

```bash
# To a project
curl -s --request POST --header "PRIVATE-TOKEN: $TOKEN" \
  --form "key=VAR_NAME" \
  --form "value=VAR_VALUE" \
  --form "masked=true" \
  --form "protected=false" \
  "$GITLAB_URL/api/v4/projects/<PROJECT_ID>/variables"

# To the group (shared across all projects)
curl -s --request POST --header "PRIVATE-TOKEN: $TOKEN" \
  --form "key=VAR_NAME" \
  --form "value=VAR_VALUE" \
  --form "masked=true" \
  --form "protected=false" \
  "$GITLAB_URL/api/v4/groups/6/variables"
```

### Update a variable

```bash
# Project-level
curl -s --request PUT --header "PRIVATE-TOKEN: $TOKEN" \
  --form "value=NEW_VALUE" \
  "$GITLAB_URL/api/v4/projects/<PROJECT_ID>/variables/VAR_NAME"

# Group-level
curl -s --request PUT --header "PRIVATE-TOKEN: $TOKEN" \
  --form "value=NEW_VALUE" \
  "$GITLAB_URL/api/v4/groups/6/variables/VAR_NAME"
```

### Delete a variable

```bash
# Project-level
curl -s --request DELETE --header "PRIVATE-TOKEN: $TOKEN" \
  "$GITLAB_URL/api/v4/projects/<PROJECT_ID>/variables/VAR_NAME"

# Group-level
curl -s --request DELETE --header "PRIVATE-TOKEN: $TOKEN" \
  "$GITLAB_URL/api/v4/groups/6/variables/VAR_NAME"
```

### Variable rules

- **Masked variables** cannot contain spaces, newlines, or be less than 8 characters
- **Protected variables** are only available to jobs running on protected branches (main is protected)
- If a variable is both masked and protected, it's only in production-path jobs on main
- **Project-level overrides group-level** — if the same key exists at both levels, the project value wins

## Pipeline schedules

### List schedules

```bash
curl -s --header "PRIVATE-TOKEN: $TOKEN" \
  "$GITLAB_URL/api/v4/projects/1/pipeline_schedules" | python3 -c "
import json, sys
for s in json.loads(sys.stdin.read()):
    print(f'ID={s[\"id\"]} | {s[\"description\"]} | cron={s[\"cron\"]} | active={s[\"active\"]}')
"
```

### Create a schedule

```bash
curl -s --request POST --header "PRIVATE-TOKEN: $TOKEN" \
  --form "description=Schedule Name" \
  --form "ref=main" \
  --form "cron=0 11 * * *" \
  --form "cron_timezone=UTC" \
  "$GITLAB_URL/api/v4/projects/1/pipeline_schedules"
```

To add a variable to the schedule (e.g., SCHEDULE_JOB):
```bash
curl -s --request POST --header "PRIVATE-TOKEN: $TOKEN" \
  --form "key=SCHEDULE_JOB" --form "value=compute-index" \
  "$GITLAB_URL/api/v4/projects/1/pipeline_schedules/<SCHEDULE_ID>/variables"
```

## Guardrails

- **Only Logan can trigger production deploys** via protected environments
- **Don't delete CI variables** without confirming — some are shared across multiple jobs
- **Masked variables can't have spaces** — remove spaces before adding (relevant for app passwords)
- **Test .gitlab-ci.yml changes via MR** — broken YAML creates failed pipelines with zero jobs
