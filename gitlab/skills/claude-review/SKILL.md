---
name: claude-review
description: >
  Set up automated Claude Code reviews on GitLab merge requests.
  Use when the user asks to: add Claude reviews to a GitLab project,
  set up AI code review on MRs, configure Claude Code for GitLab CI,
  enable automated MR reviews, "add claude review to this repo",
  "set up code review on gitlab", "enable AI reviews",
  troubleshoot the Claude review job, fix the review not posting,
  or any task involving Claude Code reviewing merge requests on GitLab.
---

# Claude Code MR Reviews

Set up automated Claude Code reviews on merge requests for any project on git.primary-os.com.

## Required context

- **Always load**: `${CLAUDE_PLUGIN_ROOT}/skills/claude-review/references/ci-job.md` — the CI job YAML and configuration details
- **For API access**: `${CLAUDE_PLUGIN_ROOT}/skills/pipeline/references/api-access.md` — GitLab API URL, token, project ID

## How it works

A CI job in `.gitlab-ci.yml` runs on every `merge_request_event`:

1. Installs Claude Code CLI via the official install script
2. Authenticates with `CLAUDE_CODE_OAUTH_TOKEN` (Claude subscription, group variable)
3. Runs `claude -p` in print mode to generate a review of the MR diff
4. Posts the review as a MR note via the GitLab API using `GITLAB_REVIEW_TOKEN`

## Prerequisites

Two group-level CI variables must exist on Primary-OS (already configured):

| Variable | Purpose | How to generate |
|---|---|---|
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude subscription auth | Run `claude setup-token` locally, copy the token |
| `GITLAB_REVIEW_TOKEN` | GitLab API — post MR comments | Group access token with `api` scope |

If these are missing or expired, see the **Regenerating tokens** section below.

## Adding reviews to a new project

### Step 1: Check if CI variables are inherited

The variables are set at the Primary-OS group level. Any project under the group inherits them automatically. Verify:

```bash
curl -s --header "PRIVATE-TOKEN: $TOKEN" \
  "$GITLAB_URL/api/v4/groups/6/variables" | python3 -c "
import json, sys
for v in json.loads(sys.stdin.read()):
    if v['key'] in ('CLAUDE_CODE_OAUTH_TOKEN', 'GITLAB_REVIEW_TOKEN'):
        print(f'{v[\"key\"]}: present (masked={v[\"masked\"]})')
"
```

### Step 2: Add the review job to .gitlab-ci.yml

Load the CI job reference for the exact YAML to add:

```
${CLAUDE_PLUGIN_ROOT}/skills/claude-review/references/ci-job.md
```

Insert the `claude-code-review` job in the `check` stage (or create one if the project doesn't have stages yet). The reference file contains two variants:

- **For projects with an existing pipeline** — add just the job definition
- **For projects with no CI config** — a minimal standalone `.gitlab-ci.yml`

### Step 3: Commit and push

Commit the `.gitlab-ci.yml` change to main (or to a branch and MR). The review job triggers automatically on the next MR.

### Step 4: Verify

Create a test MR with a small change. Check:

1. The pipeline has a `claude-code-review` job
2. The job installs Claude Code and generates a review
3. A "Claude Code Review" comment appears on the MR

If the job fails, check the log for:
- **PATH issues**: `claude` installs to `~/.local/bin` — the job must export this to PATH
- **401 on note POST**: Token expired or revoked — regenerate `GITLAB_REVIEW_TOKEN`
- **Empty review output**: Check `CLAUDE_CODE_OAUTH_TOKEN` is valid — regenerate with `claude setup-token`

## Regenerating tokens

### CLAUDE_CODE_OAUTH_TOKEN (expires after 1 year)

Run locally (requires a Claude Pro/Max/Team/Enterprise subscription):

```bash
claude setup-token
```

This opens a browser for OAuth. Copy the printed token and update the group variable:

```bash
curl -s --request PUT --header "PRIVATE-TOKEN: $TOKEN" \
  --form "value=NEW_TOKEN_VALUE" \
  "$GITLAB_URL/api/v4/groups/6/variables/CLAUDE_CODE_OAUTH_TOKEN"
```

### GITLAB_REVIEW_TOKEN (expires after 1 year)

Create a new group access token in GitLab Admin or via the API, then update:

```bash
curl -s --request PUT --header "PRIVATE-TOKEN: $TOKEN" \
  --form "value=NEW_TOKEN_VALUE" \
  "$GITLAB_URL/api/v4/groups/6/variables/GITLAB_REVIEW_TOKEN"
```

## Customizing the review prompt

Edit the `claude -p` prompt in the CI job to change what Claude focuses on. Common customizations:

- Add project-specific review criteria
- Reference the project's CLAUDE.md for conventions
- Change the model (add `--model claude-opus-4-7` for deeper reviews)
- Adjust allowed tools (currently read-only git operations)

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| Job succeeds but no comment | Review output was empty | Check `CLAUDE_CODE_OAUTH_TOKEN` validity |
| `401 Unauthorized` on curl POST | `GITLAB_REVIEW_TOKEN` expired/revoked | Regenerate the token |
| `claude: not found` | PATH not set after install | Add `export PATH="$HOME/.local/bin:$PATH"` to `before_script` |
| Job times out (10 min) | Large diff or slow model | Increase `timeout`, add `--max-turns 3` to claude args |
| Review posts error text | stderr mixed into review output | Separate stderr: `> /tmp/review.md 2>/tmp/err.log` |
