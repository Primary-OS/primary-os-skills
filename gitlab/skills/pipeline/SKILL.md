---
name: pipeline
description: >
  Check Lovelace pipeline status and debug failures on GitLab CI/CD.
  Use when the user asks to: check pipeline status, view job logs,
  see what's deploying, debug a failed job, retry a job, cancel a stuck job,
  "why did the deploy fail", "what's the pipeline doing", "is the deploy done",
  "check CI", "show me the logs", "retry the build", "cancel that job",
  or any task involving monitoring or debugging Lovelace CI/CD pipelines.
---

# Pipeline Status & Debugging

Check pipeline status, view job logs, retry failed jobs, and cancel stuck jobs on the Lovelace GitLab CI/CD pipeline.

## Required context

Load these references based on the task:

- **Always load**: `${CLAUDE_PLUGIN_ROOT}/skills/pipeline/references/api-access.md` — GitLab API URL, token, project ID
- **For understanding failures**: `${CLAUDE_PLUGIN_ROOT}/skills/pipeline/references/pipeline-architecture.md` — stages, job dependencies, what each job does

## Procedure

### Check recent pipeline status

```bash
curl -s --header "PRIVATE-TOKEN: $TOKEN" \
  "$GITLAB_URL/api/v4/projects/1/pipelines?per_page=5" | python3 -c "
import json, sys
for p in json.loads(sys.stdin.read()):
    print(f'Pipeline #{p[\"id\"]} | {p[\"status\"]:<12} | ref={p[\"ref\"]} | source={p[\"source\"]}')
"
```

### List jobs in a pipeline

```bash
curl -s --header "PRIVATE-TOKEN: $TOKEN" \
  "$GITLAB_URL/api/v4/projects/1/pipelines/<PIPELINE_ID>/jobs?per_page=30" | python3 -c "
import json, sys
jobs = json.loads(sys.stdin.read())
for j in sorted(jobs, key=lambda x: x['id']):
    dur = f'{j[\"duration\"]:.0f}s' if j.get('duration') else ''
    print(f'{j[\"id\"]:>4} {j[\"name\"]:<35} {j[\"status\"]:<12} {dur}')
"
```

### View job logs

```bash
curl -s --header "PRIVATE-TOKEN: $TOKEN" \
  "$GITLAB_URL/api/v4/projects/1/jobs/<JOB_ID>/trace" | tail -40
```

### Retry a failed job

```bash
curl -s --request POST --header "PRIVATE-TOKEN: $TOKEN" \
  "$GITLAB_URL/api/v4/projects/1/jobs/<JOB_ID>/retry"
```

### Cancel a stuck/running job

```bash
curl -s --request POST --header "PRIVATE-TOKEN: $TOKEN" \
  "$GITLAB_URL/api/v4/projects/1/jobs/<JOB_ID>/cancel"
```

### Trigger a new pipeline on main

```bash
curl -s --request POST --header "PRIVATE-TOKEN: $TOKEN" \
  "$GITLAB_URL/api/v4/projects/1/pipeline?ref=main"
```

## Interpreting results

### Pipeline sources
- `push` — triggered by a push to main (staging auto-deploy)
- `merge_request_event` — MR checks (typecheck, smoke test, supabase tests)
- `schedule` — scheduled job (compute-index)
- `web` — manually triggered from GitLab UI

### Job statuses
- `success` — completed OK
- `failed` — exited with non-zero code, check logs
- `running` — in progress
- `pending` — queued, waiting for runner
- `manual` — waiting for manual trigger (e.g., production deploy button)
- `canceled` — stopped by user
- `skipped` — not triggered (path filter didn't match, or dependency failed)

### Common failure patterns

1. **Job failed immediately with no log output** → YAML syntax error. Check `.gitlab-ci.yml` for invalid YAML.
2. **"Invalid project token for environment"** → Railway token mismatch. Staging token used for production. Check `export RAILWAY_TOKEN` in the job script.
3. **Vercel deploy hangs after upload** → The `--prod` flag on `vercel deploy` hangs. Must use the sed+deploy+alias pattern.
4. **"prebuilt environment mismatch"** → Build target (prod) doesn't match deploy target (preview). Need the sed patch on builds.json.
5. **DinD "HTTP request to HTTPS server"** → Missing `DOCKER_TLS_CERTDIR: ""` and `DOCKER_HOST` variables.

## Guardrails

- **Read job logs before retrying** — understand why it failed first
- **Don't cancel production deploy jobs** unless they're genuinely stuck (10+ minutes with no log output)
- **Only Logan can trigger production deploys** — the play button on `migrations-production` is restricted via protected environments
