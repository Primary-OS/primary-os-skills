# GitLab Operations Reference

## gitlab-ctl Commands

```bash
# Service management
gitlab-ctl status                    # Check all services
gitlab-ctl restart                   # Restart all services
gitlab-ctl restart puma              # Restart specific service (puma, sidekiq, nginx, etc.)
gitlab-ctl stop                      # Stop all services
gitlab-ctl start                     # Start all services
gitlab-ctl reconfigure               # Apply changes from gitlab.rb (MUST run after any config edit)

# Logs
gitlab-ctl tail                      # Tail all logs
gitlab-ctl tail puma                 # Tail specific service logs
gitlab-ctl tail nginx/gitlab_access  # Tail nginx access log

# Health
gitlab-rake gitlab:check             # Full system health check
gitlab-rake gitlab:env:info          # Environment info
gitlab-rake gitlab:doctor:secrets    # Check secrets health
```

## Configuration Changes

All config lives in `/etc/gitlab/gitlab.rb`. The workflow is always:

1. `ssh root@87.99.146.126`
2. Edit `/etc/gitlab/gitlab.rb`
3. Run `gitlab-ctl reconfigure`
4. Verify with `gitlab-ctl status`

Common settings:

```ruby
# Domain
external_url 'https://git.primary-os.com'

# SMTP (example for Postmark)
gitlab_rails['smtp_enable'] = true
gitlab_rails['smtp_address'] = "smtp.postmarkapp.com"
gitlab_rails['smtp_port'] = 587
gitlab_rails['smtp_user_name'] = "API_TOKEN"
gitlab_rails['smtp_password'] = "API_TOKEN"
gitlab_rails['smtp_authentication'] = "plain"
gitlab_rails['smtp_enable_starttls_auto'] = true
gitlab_rails['gitlab_email_from'] = 'gitlab@primary-os.com'

# Container registry
registry_external_url 'https://registry.primary-os.com'

# GitLab Pages
pages_external_url 'https://pages.primary-os.com'
```

## User Management

### Via API (preferred)

The admin API token is in the instance reference. Use it for user CRUD and group membership.

```bash
TOKEN="<admin-token-from-instance-reference>"
BASE="https://git.primary-os.com/api/v4"

# List all users
curl -s --header "PRIVATE-TOKEN: $TOKEN" "$BASE/users" | python3 -m json.tool

# Create user
curl -s --request POST --header "PRIVATE-TOKEN: $TOKEN" \
  --data-urlencode "email=jdoe@primary.vc" \
  --data-urlencode "username=jdoe" \
  --data-urlencode "name=Jane Doe" \
  --data "password=TempPass-2026!&skip_confirmation=true" \
  "$BASE/users"

# Block/unblock user
curl -s --request POST --header "PRIVATE-TOKEN: $TOKEN" "$BASE/users/<USER_ID>/block"
curl -s --request POST --header "PRIVATE-TOKEN: $TOKEN" "$BASE/users/<USER_ID>/unblock"

# List group members
curl -s --header "PRIVATE-TOKEN: $TOKEN" "$BASE/groups/6/members"

# Add user to Primary-OS group (access_level: Guest=10, Reporter=20, Developer=30, Maintainer=40, Owner=50)
curl -s --request POST --header "PRIVATE-TOKEN: $TOKEN" \
  --data "user_id=<USER_ID>&access_level=40" \
  "$BASE/groups/6/members"

# Change member access level
curl -s --request PUT --header "PRIVATE-TOKEN: $TOKEN" \
  --data "access_level=40" \
  "$BASE/groups/6/members/<USER_ID>"

# Remove member from group
curl -s --request DELETE --header "PRIVATE-TOKEN: $TOKEN" "$BASE/groups/6/members/<USER_ID>"
```

### Via Rails console (SSH required — use for operations the API doesn't support)

```bash
ssh root@87.99.146.126 "gitlab-rails runner '<ruby code>'"
```

```ruby
# Send password reset email (no API equivalent)
User.find(<USER_ID>).send_reset_password_instructions

# Make admin (API can also do this, but Rails is simpler)
user = User.find_by(username: 'jdoe')
user.admin = true
user.save!

# List all users with details
User.all.each { |u| puts "#{u.id} - #{u.username} - #{u.email} - admin:#{u.admin} - #{u.state}" }
```

**GitLab 18.x note**: `User.new(...).save!` no longer works — it fails with "Namespace can't be blank". Use the API for user creation instead.

## Updating GitLab

```bash
# 1. Create a backup first
gitlab-backup create

# 2. Update packages
apt-get update
apt-get install gitlab-ee

# 3. Reconfigure (runs automatically during install, but just in case)
gitlab-ctl reconfigure

# 4. Check version
gitlab-rake gitlab:env:info | grep "GitLab information"
```

**Important**: GitLab only supports upgrading one minor version at a time. Check the upgrade path at https://docs.gitlab.com/ee/update/#upgrade-paths before jumping versions.

## Troubleshooting

### 502 errors
Usually means Puma hasn't started yet. Check:
```bash
gitlab-ctl status puma
gitlab-ctl tail puma
# If stuck, restart:
gitlab-ctl restart puma
```

### High memory usage
```bash
free -h
gitlab-ctl status
# Check Sidekiq concurrency in gitlab.rb:
# sidekiq['concurrency'] = 10  (lower from default 25 if needed)
```

### Disk space
```bash
df -h /var/opt/gitlab
du -sh /var/opt/gitlab/git-data/repositories
du -sh /var/opt/gitlab/backups
# Expand Hetzner volume if needed (from local machine):
# hcloud volume resize gitlab-data --size 200
# Then on server: resize2fs /dev/disk/by-id/scsi-0HC_Volume_105492436
```

### Let's Encrypt renewal issues
```bash
gitlab-ctl renew-le-certs
```

### Full reset of a stuck reconfigure
```bash
gitlab-ctl cleanse    # WARNING: removes all data
# Only use as last resort — prefer targeted service restarts
```
