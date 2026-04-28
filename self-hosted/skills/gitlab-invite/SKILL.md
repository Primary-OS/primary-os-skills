---
name: gitlab-invite
description: >
  Invite new users to Primary's self-hosted GitLab instance.
  Use when the user asks to: invite someone to GitLab, add a new GitLab user,
  onboard a teammate to GitLab, create a GitLab account, "add <name> to GitLab",
  "invite <name> to Primary-OS", "give <name> GitLab access",
  set up a new developer, onboard someone, or any task involving
  creating a GitLab account and granting group access.
---

# GitLab User Invite

Create a GitLab account, add the user to the Primary-OS group, and send a password reset email so they can set their own password.

## Required context

Load the instance reference for the API token and server details:
- `${CLAUDE_PLUGIN_ROOT}/skills/gitlab/references/instance.md`

## Procedure

### Step 1: Gather user details

Collect from the user's message or ask with `AskUserQuestion`:
- **Email** (required) — must be a `@primary.vc` address
- **Full name** (required) — infer from the email if not provided (e.g., `jordan.fox@primary.vc` -> `Jordan Fox`)
- **Username** (optional) — default to the email prefix (e.g., `jordan.fox`)
- **Access level** (optional) — default to `Maintainer` (40). Options: Guest (10), Reporter (20), Developer (30), Maintainer (40), Owner (50)

If the user provides multiple people, process them all in a single batch.

### Step 2: Create user accounts via API

Use the GitLab REST API — no SSH required.

```bash
TOKEN="<admin-token-from-instance-reference>"
BASE="https://git.primary-os.com/api/v4"

# Check if user already exists
curl -s --header "PRIVATE-TOKEN: $TOKEN" "$BASE/users?search=<EMAIL>"

# Create user (skip_confirmation so no email verification needed)
curl -s --request POST --header "PRIVATE-TOKEN: $TOKEN" \
  --data-urlencode "email=<EMAIL>" \
  --data-urlencode "username=<USERNAME>" \
  --data-urlencode "name=<FULL_NAME>" \
  --data "password=TempPass-2026!&skip_confirmation=true" \
  "$BASE/users"
```

The response includes the user's `id` — save it for the next steps.

If the user already exists, skip creation and use the existing `id`.

### Step 3: Add to Primary-OS group

```bash
# Primary-OS group ID is 6
curl -s --request POST --header "PRIVATE-TOKEN: $TOKEN" \
  --data "user_id=<USER_ID>&access_level=<ACCESS_LEVEL>" \
  "$BASE/groups/6/members"
```

If the user is already a member, the API returns `"Member already exists"` — this is fine, not an error.

### Step 4: Send password reset email

This requires SSH — the GitLab API does not have a password reset endpoint.

```bash
ssh root@87.99.146.126 "gitlab-rails runner 'User.find(<USER_ID>).send_reset_password_instructions'"
```

For multiple users, batch them in a single SSH call:

```bash
ssh root@87.99.146.126 "gitlab-rails runner '
[<ID1>, <ID2>, <ID3>].each do |uid|
  u = User.find(uid)
  u.send_reset_password_instructions
  puts \"#{u.name} (#{u.email}) — password reset email sent\"
end
'"
```

### Step 5: Verify and report

Confirm the group membership by listing members:

```bash
curl -s --header "PRIVATE-TOKEN: $TOKEN" "$BASE/groups/6/members" | \
  python3 -c "
import sys, json
levels = {10:'Guest',20:'Reporter',30:'Developer',40:'Maintainer',50:'Owner'}
for m in json.load(sys.stdin):
    print(f\"{m['username']:20s} {m['name']:20s} {levels.get(m['access_level'], m['access_level'])}\")"
```

Report to the user:
- Which accounts were created (or already existed)
- Group membership and access level
- That password reset emails were sent
- Remind them the emails come from `logan@primary.vc` (Gmail SMTP) — check spam if not received
- Login URL: `https://git.primary-os.com`

## Notes

- Public sign-up is disabled — all accounts must be created by an admin
- The password reset email lets users set their own password (no need to share a temp password)
- Emails come from `logan@primary.vc` due to Gmail SMTP configuration
- Each SSH key can only be on one GitLab account — if users need SSH access for Git, they should add their key via Profile > SSH Keys after logging in
