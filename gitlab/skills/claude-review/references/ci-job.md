# Claude Code Review — CI Job Configuration

## Job YAML (add to existing pipeline)

Add this job to your `.gitlab-ci.yml` in the `check` stage:

```yaml
claude-code-review:
  stage: check
  image: node:24-alpine3.21
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  variables:
    GIT_STRATEGY: fetch
    GIT_DEPTH: 0
  before_script:
    - apk update
    - apk add --no-cache git curl bash jq
    - curl -fsSL https://claude.ai/install.sh | bash
    - export PATH="$HOME/.local/bin:$PATH"
  script:
    - claude --version
    - |
      claude -p "Review this GitLab merge request.

      Project: ${CI_PROJECT_NAME}
      MR: !${CI_MERGE_REQUEST_IID}
      Author: ${GITLAB_USER_NAME}
      Branch: ${CI_MERGE_REQUEST_SOURCE_BRANCH_NAME} → ${CI_MERGE_REQUEST_TARGET_BRANCH_NAME}

      Focus on code quality, bugs, security, and performance.
      Be concise — only flag meaningful issues, not style nits.
      Format as markdown for a GitLab MR comment.

      The MR branch is checked out. Read any file for full context." \
        --allowedTools "Read Bash(git diff:*) Bash(git log:*) Bash(git show:*)" \
        --permission-mode acceptEdits \
        --output-format text > /tmp/review.md 2>/tmp/review-err.log || true
      echo "--- claude stderr ---" && cat /tmp/review-err.log || true
      echo "--- claude stdout ---" && cat /tmp/review.md || true

      if [ -s /tmp/review.md ]; then
        jq -n --rawfile body /tmp/review.md '{body: ("## Claude Code Review\n\n" + $body)}' > /tmp/note.json
        curl --fail-with-body --request POST \
          --header "PRIVATE-TOKEN: ${GITLAB_REVIEW_TOKEN}" \
          --header "Content-Type: application/json" \
          --data @/tmp/note.json \
          "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${CI_MERGE_REQUEST_IID}/notes"
      else
        echo "Review generation produced no output — check stderr above"
      fi
  allow_failure: true
  timeout: 10 minutes
```

## Minimal standalone .gitlab-ci.yml (for projects with no CI)

For projects that don't have a `.gitlab-ci.yml` yet:

```yaml
stages:
  - check

default:
  tags:
    - docker

claude-code-review:
  stage: check
  image: node:24-alpine3.21
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  variables:
    GIT_STRATEGY: fetch
    GIT_DEPTH: 0
  before_script:
    - apk update
    - apk add --no-cache git curl bash jq
    - curl -fsSL https://claude.ai/install.sh | bash
    - export PATH="$HOME/.local/bin:$PATH"
  script:
    - claude --version
    - |
      claude -p "Review this GitLab merge request.

      Project: ${CI_PROJECT_NAME}
      MR: !${CI_MERGE_REQUEST_IID}
      Author: ${GITLAB_USER_NAME}
      Branch: ${CI_MERGE_REQUEST_SOURCE_BRANCH_NAME} → ${CI_MERGE_REQUEST_TARGET_BRANCH_NAME}

      Focus on code quality, bugs, security, and performance.
      Be concise — only flag meaningful issues, not style nits.
      Format as markdown for a GitLab MR comment.

      The MR branch is checked out. Read any file for full context." \
        --allowedTools "Read Bash(git diff:*) Bash(git log:*) Bash(git show:*)" \
        --permission-mode acceptEdits \
        --output-format text > /tmp/review.md 2>/tmp/review-err.log || true
      echo "--- claude stderr ---" && cat /tmp/review-err.log || true
      echo "--- claude stdout ---" && cat /tmp/review.md || true

      if [ -s /tmp/review.md ]; then
        jq -n --rawfile body /tmp/review.md '{body: ("## Claude Code Review\n\n" + $body)}' > /tmp/note.json
        curl --fail-with-body --request POST \
          --header "PRIVATE-TOKEN: ${GITLAB_REVIEW_TOKEN}" \
          --header "Content-Type: application/json" \
          --data @/tmp/note.json \
          "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/merge_requests/${CI_MERGE_REQUEST_IID}/notes"
      else
        echo "Review generation produced no output — check stderr above"
      fi
  allow_failure: true
  timeout: 10 minutes
```

## Key configuration details

### Authentication

| Variable | Auth type | Used for |
|---|---|---|
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude subscription OAuth | Authenticating the Claude CLI (#5 in auth precedence) |
| `GITLAB_REVIEW_TOKEN` | GitLab PAT with `api` scope | Posting MR notes via GitLab API |

Both are masked group-level variables on Primary-OS (group ID 6), inherited by all projects.

### Claude CLI in Alpine

The official install script (`claude.ai/install.sh`) puts the binary at `~/.local/bin/claude`. Alpine doesn't include this in PATH by default, so the `before_script` must export it.

### Why not CI_JOB_TOKEN?

`CI_JOB_TOKEN` doesn't have permission to create MR notes via the GitLab API. A group/project access token with `api` scope is required.

### Allowed tools

The job restricts Claude to read-only operations:
- `Read` — read files in the repo
- `Bash(git diff:*)` — view diffs
- `Bash(git log:*)` — view commit history
- `Bash(git show:*)` — view specific commits

This prevents Claude from modifying the repo during review.
