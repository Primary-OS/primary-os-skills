# Backup & Restore Procedures

## Backup Architecture

Three layers protect against data loss:

1. **Hetzner server snapshots** — full machine image, automatic nightly
2. **GitLab app backups** — repos, database, uploads, CI data
3. **Config backups** — gitlab.rb and gitlab-secrets.json (not included in app backups)

All app and config backups are stored on the attached Hetzner Volume at `/var/opt/gitlab/backups/`.

## Manual Backup

```bash
# Full application backup
gitlab-backup create

# Config backup (always do this alongside app backup)
tar czf /var/opt/gitlab/backups/gitlab-config-$(date +%Y%m%d).tar.gz \
  /etc/gitlab/gitlab.rb /etc/gitlab/gitlab-secrets.json
```

## Verify Backups

```bash
# List existing backups
ls -lh /var/opt/gitlab/backups/

# Check cron is running
cat /etc/cron.d/gitlab-backup
grep gitlab /var/log/syslog | tail -5
cat /var/log/gitlab-backup.log | tail -20
```

## Restore from Backup

**Critical**: You need BOTH the app backup tarball AND the config backup (gitlab.rb + gitlab-secrets.json). Without gitlab-secrets.json, encrypted data (2FA secrets, CI variables, etc.) is unrecoverable.

### Full restore procedure

```bash
# 1. Stop data-writing services
gitlab-ctl stop puma
gitlab-ctl stop sidekiq
gitlab-ctl status   # Verify puma and sidekiq are down

# 2. Restore the config files FIRST
tar xzf /var/opt/gitlab/backups/gitlab-config-YYYYMMDD.tar.gz -C /
gitlab-ctl reconfigure

# 3. Restore the application backup
# Use the timestamp prefix from the backup filename (e.g., 1776955652_2026_04_23_18.11.1-ee)
gitlab-backup restore BACKUP=1776955652_2026_04_23_18.11.1-ee

# 4. Reconfigure and restart
gitlab-ctl reconfigure
gitlab-ctl restart

# 5. Verify
gitlab-rake gitlab:check SANITIZE=true
```

### Restore to a NEW server

1. Provision a new server (same OS, same or newer GitLab version)
2. Install GitLab EE: `EXTERNAL_URL='https://git.primary-os.com' apt-get install gitlab-ee`
3. Copy backup files to the new server's `/var/opt/gitlab/backups/`
4. Restore config files (gitlab.rb, gitlab-secrets.json)
5. Run `gitlab-ctl reconfigure`
6. Follow restore steps 1-5 above
7. Update DNS to point to new server IP

## Disaster Recovery Checklist

- [ ] Can you SSH into the server?
- [ ] Is the latest backup file present in `/var/opt/gitlab/backups/`?
- [ ] Do you have gitlab-secrets.json? (check config backup tarballs)
- [ ] If server is dead: can you create a new one from Hetzner snapshot?
- [ ] If volume is dead: do you have offsite copies of backups?

## Setting Up Offsite Backups

To copy backups off the server (recommended):

```bash
# Option 1: Hetzner Storage Box via SFTP (add to cron)
scp /var/opt/gitlab/backups/*_gitlab_backup.tar \
  uXXXXXX@uXXXXXX.your-storagebox.de:backups/

# Option 2: S3-compatible storage
# Add to gitlab.rb:
gitlab_rails['backup_upload_connection'] = {
  'provider' => 'AWS',
  'region' => 'us-east-1',
  'aws_access_key_id' => 'KEY',
  'aws_secret_access_key' => 'SECRET',
  'endpoint' => 'https://s3.amazonaws.com'
}
gitlab_rails['backup_upload_remote_directory'] = 'primary-gitlab-backups'
```
