---
name: setup-local
description: >
  Set up a teammate's local machine to work with Primary's self-hosted GitLab
  instance at git.primary-os.com. Use when a new team member needs to get
  connected to GitLab, when someone asks "how do I set up GitLab",
  "connect to git.primary-os.com", "set up glab", "add my SSH key to GitLab",
  "configure git for GitLab", or any onboarding task involving local GitLab
  tooling and authentication.
---

# GitLab Local Setup

Set up a teammate's machine to work with Primary's self-hosted GitLab EE instance at `git.primary-os.com`. This covers installing the CLI, authenticating, adding SSH keys, and configuring git.

## Prerequisites

- macOS with Homebrew installed
- A GitLab account on `git.primary-os.com` — an admin must create this since public sign-up is disabled. The teammate will receive a password setup email from GitLab (sent from `logan@primary.vc`). They must set their password and log in before proceeding.

## Procedure

### Step 1: Gather info

Ask the user for:
- Their GitLab username (if they don't have an account yet, ask an admin to create one via Admin Area > Overview > Users > New user)
- Whether they already have an SSH key (`ls ~/.ssh/id_ed25519.pub` or `ls ~/.ssh/id_rsa.pub`)

### Step 2: Install the GitLab CLI (`glab`)

```bash
brew install glab
```

Verify:
```bash
glab --version
```

### Step 3: Create a Personal Access Token

The teammate creates their own PAT through the GitLab web UI:

1. Log in to `https://git.primary-os.com`
2. Go to **User Settings > Access Tokens**: `https://git.primary-os.com/-/user_settings/personal_access_tokens`
3. Click **Add new token**
4. Name: `glab-cli` (or any descriptive name)
5. Expiration: 1 year
6. Select scopes: `api`, `read_user`, `read_repository`, `write_repository`
7. Click **Create personal access token**
8. **Copy the token immediately** — it is only shown once

### Step 4: Authenticate `glab`

Use `glab config set` to store the token. Do NOT use `glab auth login --stdin` — it fails on self-hosted instances with a "Set client_id first" error.

```bash
glab config set token "<TOKEN>" --host git.primary-os.com
glab config set git_protocol ssh --host git.primary-os.com
glab config set api_protocol https --host git.primary-os.com
```

Verify:
```bash
glab auth status --hostname git.primary-os.com
```

Expected output should show: `Logged in to git.primary-os.com as <username>`.

**Important**: Most `glab` subcommands don't support `--hostname`. Set `GITLAB_HOST` to target the Primary instance:

```bash
GITLAB_HOST=git.primary-os.com glab repo list
```

To make this automatic, add to the shell profile:
```bash
echo 'export GITLAB_HOST=git.primary-os.com' >> ~/.zshrc
source ~/.zshrc
```

### Step 5: Set up SSH key

**If the user does NOT have an SSH key**, generate one first:
```bash
ssh-keygen -t ed25519 -C "<user>@primary.vc"
```
Accept defaults (press Enter for file location and optionally set a passphrase).

**Add the SSH key to GitLab** using the PAT from Step 3:
```bash
curl -s --request POST \
  --header "PRIVATE-TOKEN: <TOKEN>" \
  --data-urlencode "title=<DESCRIPTIVE-NAME>" \
  --data-urlencode "key=$(cat ~/.ssh/id_ed25519.pub)" \
  "https://git.primary-os.com/api/v4/user/keys"
```

The response should include `"title":"<DESCRIPTIVE-NAME>"` confirming the key was added.

**Note**: Each SSH key can only be on one GitLab account. If you get `fingerprint_sha256 has already been taken`, the key is already registered to another account.

### Step 6: Add GitLab to known hosts

```bash
ssh-keyscan git.primary-os.com >> ~/.ssh/known_hosts 2>/dev/null
```

Without this step, the first SSH connection will fail with "Host key verification failed."

### Step 7: Verify everything works

Test SSH:
```bash
ssh -T git@git.primary-os.com
```
Expected: `Welcome to GitLab, @<username>!`

Test CLI:
```bash
glab auth status --hostname git.primary-os.com
```

Test cloning (if any repos exist):
```bash
git clone git@git.primary-os.com:<namespace>/<repo>.git
```

### Step 8: Configure git identity

Set the user's name and email for commits:
```bash
git config --global user.name "Their Name"
git config --global user.email "their-email@primary.vc"
```

## Troubleshooting

### `glab auth login` fails with "Set 'client_id' first"
Known issue with self-hosted instances. Use `glab config set token` instead (Step 4).

### `--hostname` flag not recognized on glab subcommands
Most glab subcommands don't support `--hostname`. Use `GITLAB_HOST=git.primary-os.com` env var instead.

### SSH says "Host key verification failed"
Run `ssh-keyscan git.primary-os.com >> ~/.ssh/known_hosts` (Step 6).

### SSH key "fingerprint has already been taken"
The same SSH key can't be on two GitLab accounts. Either remove it from the other account or generate a new key.

### 422 "The change you requested was rejected" on web login
CSRF token issue from stale session cookies, not a password problem. Clear cookies for `git.primary-os.com` or use an incognito window.

### API returns 401 Unauthorized
The Personal Access Token may have expired (they're set to 1 year). Create a new one at `https://git.primary-os.com/-/user_settings/personal_access_tokens`.
