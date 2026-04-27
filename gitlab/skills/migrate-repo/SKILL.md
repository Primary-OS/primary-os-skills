---
name: migrate-repo
description: >
  Migrate a repository from GitHub to the self-hosted GitLab instance at
  git.primary-os.com. Use when a teammate asks to "move a repo to GitLab",
  "migrate from GitHub", "convert a repo", "mirror a repo to GitLab",
  "import from GitHub", or any task involving transferring a repository
  from GitHub to the Primary GitLab instance.
---

# GitHub to GitLab Repository Migration

Migrate a repository from GitHub to Primary's self-hosted GitLab EE instance at `git.primary-os.com`, preserving all branches, tags, and commit history.

## Prerequisites

- The user must have push access to the GitHub repo being migrated
- The user must have a GitLab account on `git.primary-os.com` with a valid Personal Access Token
- The `gh` CLI must be authenticated with GitHub (`gh auth status`)
- The destination GitLab group/namespace must already exist

## Procedure

### Step 1: Identify the source repo

Ask the user for the GitHub repository to migrate. Get the full name (e.g., `org/repo-name`).

Verify access:
```bash
gh repo view <ORG>/<REPO> --json name,defaultBranchRef,isPrivate
```

### Step 2: Create the project on GitLab

Create an empty project on GitLab under the target namespace. Use the GitLab API:

```bash
curl -s --request POST \
  --header "PRIVATE-TOKEN: <GITLAB_TOKEN>" \
  --header "Content-Type: application/json" \
  --data '{"name": "<REPO_NAME>", "path": "<repo-name-lowercase>", "namespace_id": <GROUP_ID>, "visibility": "private"}' \
  "https://git.primary-os.com/api/v4/projects"
```

To find the `namespace_id` for a group:
```bash
curl -s --header "PRIVATE-TOKEN: <GITLAB_TOKEN>" \
  "https://git.primary-os.com/api/v4/groups?search=<group-name>" | python3 -c "import json,sys; [print(g['id'], g['name']) for g in json.loads(sys.stdin.read())]"
```

**Note**: If the API returns 401 when creating a project under a group, the user may not have sufficient permissions on that group. An admin (or someone with Owner role on the group) may need to create the project or grant the user Developer+ access.

### Step 3: Clone a bare mirror from GitHub

```bash
cd /tmp
git clone --mirror https://github.com/<ORG>/<REPO>.git
```

This creates a bare repository with all branches, tags, and refs — no working directory.

### Step 4: Push the mirror to GitLab

```bash
cd /tmp/<REPO>.git
git push --mirror git@git.primary-os.com:<gitlab-namespace>/<repo-name>.git
```

This pushes everything: all branches, all tags, full history.

**Note**: This will also push GitHub-specific refs like `refs/pull/*/head` (pull request refs). These are harmless on GitLab and can be ignored.

### Step 5: Clean up

Remove the temporary bare clone:
```bash
rm -rf /tmp/<REPO>.git
```

### Step 6: Verify

Check the repo in the browser:
```
https://git.primary-os.com/<gitlab-namespace>/<repo-name>
```

Verify from the CLI:
```bash
git clone git@git.primary-os.com:<gitlab-namespace>/<repo-name>.git /tmp/<repo-name>-verify
cd /tmp/<repo-name>-verify
git log --oneline -10
git branch -a
rm -rf /tmp/<repo-name>-verify
```

### Step 7: Update local clones (if applicable)

If the user already has a local clone of the GitHub repo, add GitLab as a new remote:
```bash
cd <local-repo-directory>
git remote add gitlab git@git.primary-os.com:<gitlab-namespace>/<repo-name>.git
```

Or to replace the origin entirely:
```bash
git remote set-url origin git@git.primary-os.com:<gitlab-namespace>/<repo-name>.git
```

To keep both remotes (push to GitLab, still pull from GitHub during transition):
```bash
git remote set-url origin git@git.primary-os.com:<gitlab-namespace>/<repo-name>.git
git remote add github https://github.com/<ORG>/<REPO>.git
```

## What gets migrated

| Migrated | Not migrated |
|---|---|
| All branches | GitHub Issues |
| All tags | Pull Requests (as MRs) |
| Full commit history | GitHub Actions / Workflows |
| All authors and timestamps | GitHub Wiki (separate step needed) |
| LFS objects (if using git-lfs) | GitHub Releases |
| | Repository settings / webhooks |
| | GitHub Projects / Discussions |

## Troubleshooting

### Push fails with permission denied
Make sure the SSH key is added to the GitLab account that owns or has write access to the target project.

### Push fails with "remote: error: hook declined"
The branch protection on the GitLab project may prevent force pushes. Temporarily disable branch protection in **Settings > Repository > Protected branches**, push, then re-enable.

### Very large repos time out
For repos over 1 GB, consider using `git clone --mirror` over SSH instead of HTTPS for the GitHub clone:
```bash
git clone --mirror git@github.com:<ORG>/<REPO>.git
```

### GitHub-specific refs clutter GitLab
The `refs/pull/*/head` refs from GitHub PRs are pushed but don't affect GitLab. To clean them up after migration:
```bash
cd /tmp/<REPO>.git
git for-each-ref --format='%(refname)' refs/pull/ | xargs -I{} git push git@git.primary-os.com:<namespace>/<repo>.git --delete {}
```
