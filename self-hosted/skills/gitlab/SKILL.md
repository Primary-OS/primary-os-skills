---
name: gitlab
description: >
  Manage Primary's self-hosted GitLab instance at git.primary-os.com.
  Use when the user asks to: manage GitLab, add/remove/list GitLab users,
  check GitLab status, restart GitLab, update GitLab, manage GitLab backups,
  restore GitLab, check disk space, expand storage, view GitLab logs,
  troubleshoot GitLab errors (502, slow, OOM), configure GitLab settings
  (SMTP, registry, pages, runners), manage SSL certs, renew Let's Encrypt,
  change GitLab configuration, "check on git.primary-os.com",
  "is GitLab up", "GitLab is down", resize server, expand volume,
  manage snapshots, change firewall rules, manage Hetzner resources,
  "hcloud", understand GitLab architecture, database schema, data stores,
  PostgreSQL tables, Redis usage, Gitaly, CI database decomposition,
  or any task involving the self-hosted GitLab server on Hetzner Cloud.
---

# GitLab Instance Management

Manage Primary's self-hosted GitLab EE instance running on Hetzner Cloud. This skill covers server administration, user management, backups, upgrades, monitoring, and troubleshooting.

## Required context

Before performing any operation, load these references as needed:

- **Always load**: `${CLAUDE_PLUGIN_ROOT}/skills/gitlab/references/instance.md` — server IP, SSH credentials, volume, domain, current config
- **For hcloud / Hetzner operations**: `${CLAUDE_PLUGIN_ROOT}/skills/gitlab/references/hcloud.md` — full hcloud CLI reference, resource IDs, resize/snapshot/firewall/migration workflows
- **For backup/restore**: `${CLAUDE_PLUGIN_ROOT}/skills/gitlab/references/backup-restore.md` — backup architecture, restore procedure, disaster recovery
- **For GitLab admin**: `${CLAUDE_PLUGIN_ROOT}/skills/gitlab/references/operations.md` — gitlab-ctl commands, config patterns, user management, troubleshooting
- **For architecture / schema / internals**: `${CLAUDE_PLUGIN_ROOT}/skills/gitlab/references/architecture.md` — data stores (PostgreSQL, Redis, Gitaly, Object Storage), database schema (~573 tables), table relationships, CI database decomposition, schema design patterns

## Procedure

### Step 1: Identify the operation category

Map the user's request to one of these categories, then follow the relevant procedure:

- **A. Health check / status** — is it up, service status, resource usage
- **B. User management** — add, remove, list, block, promote users
- **C. Configuration change** — edit gitlab.rb settings (SMTP, registry, pages, runners, etc.)
- **D. Backup & restore** — manual backup, verify backups, restore from backup
- **E. Update / upgrade** — update GitLab to a newer version
- **F. Storage management** — check disk, expand volume, clean up
- **G. Troubleshooting** — 502 errors, high memory, slow performance, cert issues
- **H. Hetzner infrastructure** — server resize, firewall changes, snapshots
- **I. Architecture & internals** — data stores, database schema, table relationships, design patterns

### Step 2A: Health check / status

```bash
ssh root@87.99.146.126 "gitlab-ctl status && echo '---' && free -h && echo '---' && df -h /var/opt/gitlab"
```

Also verify HTTPS externally:
```bash
curl -sI https://git.primary-os.com/ | head -5
```

Report: all services up/down, memory usage, disk usage, HTTPS status.

### Step 2B: User management

Load operations reference: `${CLAUDE_PLUGIN_ROOT}/skills/gitlab/references/operations.md`

**Preferred method**: Use the GitLab REST API — it doesn't require SSH access and handles user creation, listing, blocking, and group membership. The admin API token is in the instance reference.

**Fallback to SSH/Rails console** for operations the API doesn't support: sending password reset emails, promoting to admin, and direct database queries.

**Adding users**: Use `POST /api/v4/users` with `skip_confirmation=true`. After creating the user, trigger a password reset email so they can set their own password (requires Rails console: `User.find(id).send_reset_password_instructions`).

**Group membership**: Use `POST /api/v4/groups/:id/members` with `user_id` and `access_level` (Guest=10, Reporter=20, Developer=30, Maintainer=40, Owner=50). Primary-OS group ID is 6.

**Listing users/members**: Use `GET /api/v4/users` or `GET /api/v4/groups/:id/members`.

**Never**: Delete users without explicit confirmation. Prefer blocking over deletion to preserve contribution history.

### Step 2C: Configuration change

1. SSH in and read current config: `grep -n 'relevant_setting' /etc/gitlab/gitlab.rb`
2. Show the user what you plan to change
3. Edit `/etc/gitlab/gitlab.rb`
4. Run `gitlab-ctl reconfigure`
5. Verify the change took effect

**Always back up gitlab.rb before making changes**:
```bash
cp /etc/gitlab/gitlab.rb /etc/gitlab/gitlab.rb.bak.$(date +%Y%m%d%H%M)
```

### Step 2D: Backup & restore

Load backup reference: `${CLAUDE_PLUGIN_ROOT}/skills/gitlab/references/backup-restore.md`

**For manual backup**: Run both the app backup AND config backup.

**For restore**: STOP. Confirm with the user before proceeding. Restore is destructive — it replaces the current database. Verify they have the right backup file AND the matching gitlab-secrets.json.

### Step 2E: Update / upgrade

1. Check current version: `gitlab-rake gitlab:env:info | grep Version`
2. Check available version: `apt-cache policy gitlab-ee | head -5`
3. **Before upgrading**: Create a full backup (app + config)
4. GitLab only supports upgrading one minor version at a time. Check the upgrade path.
5. Confirm the upgrade plan with the user before running `apt-get install gitlab-ee`

**Never skip the backup step before an upgrade.**

### Step 2F: Storage management

Check current usage:
```bash
ssh root@87.99.146.126 "df -h /var/opt/gitlab && echo '---' && du -sh /var/opt/gitlab/git-data/repositories /var/opt/gitlab/backups /var/opt/gitlab/gitlab-rails/uploads 2>/dev/null"
```

To expand the volume (from local machine, NOT the server):
```bash
hcloud volume resize gitlab-data --size <NEW_SIZE_GB>
```
Then on server:
```bash
resize2fs /dev/disk/by-id/scsi-0HC_Volume_105492436
```

**Cleanup options** (confirm with user first):
- Prune old backups: backups older than 30 days should auto-clean, but check
- Clean Docker registry (if enabled): `gitlab-ctl registry-garbage-collect`
- Clean CI artifacts: Admin > Settings > CI/CD > Artifact expiration

### Step 2G: Troubleshooting

Load operations reference: `${CLAUDE_PLUGIN_ROOT}/skills/gitlab/references/operations.md`

**502 errors**: Check `gitlab-ctl status puma` — Puma may need a restart or more time to boot after a reconfigure. Wait 30 seconds and retry before restarting.

**High memory**: Check `free -h` and `gitlab-ctl status`. Consider reducing Sidekiq concurrency or Puma workers in gitlab.rb if consistently above 80%.

**SSL/cert errors**: Run `gitlab-ctl renew-le-certs`. If that fails, check DNS is still pointed correctly with `dig +short git.primary-os.com`.

**Slow performance**: Check `top` or `htop` for CPU/memory hogs. Check `gitlab-ctl tail sidekiq` for stuck jobs. Consider upgrading the server type if consistently maxed.

### Step 2H: Hetzner infrastructure

Load hcloud reference: `${CLAUDE_PLUGIN_ROOT}/skills/gitlab/references/hcloud.md`

**Important**: `hcloud` runs locally (not on the server). The active context is `plane`.

This covers: server resize, snapshots, volume expansion, firewall rule changes, networking, disaster recovery, and migration workflows. The reference has exact resource names/IDs and complete command syntax.

**Always confirm with the user before**: resizing (causes downtime), deleting resources, rebuilding, or changing firewall rules.

### Step 2I: Architecture & internals

Load architecture reference: `${CLAUDE_PLUGIN_ROOT}/skills/gitlab/references/architecture.md`

Use this when the user asks about GitLab's internal architecture, database schema, data stores, how data is organized, which tables exist, CI database decomposition, or when you need context about GitLab internals to inform an operational decision (e.g., understanding which PostgreSQL tables are affected by a migration, what Redis stores, how Gitaly works).

The reference covers:
- All 6 data stores and what each one holds
- PostgreSQL schema overview (~573 tables across 5 logical databases)
- Key tables and relationships (core entities, CI/CD, issues, MRs, auth, security)
- Schema design patterns (primary keys, dual IDs, partitioning, foreign keys, indexing)
- Database decomposition (main vs ci database split)
- Traffic distribution across table categories

## Guardrails

- **Never delete the server, volume, or backups** without explicit user confirmation
- **Never run `gitlab-ctl cleanse`** — this destroys all data
- **Always backup before**: upgrades, major config changes, restore operations
- **Always confirm before**: user deletion, server resize, restore operations, firewall changes
- **Config changes**: Back up gitlab.rb before editing, and always run `gitlab-ctl reconfigure` after
- **Upgrades**: Follow the official upgrade path — never skip minor versions
- **Restore**: Requires both the app backup AND gitlab-secrets.json — warn if either is missing
