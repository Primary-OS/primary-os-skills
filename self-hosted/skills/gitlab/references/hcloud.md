# Hetzner Cloud CLI (hcloud) Reference

## Setup

The `hcloud` CLI is installed locally (not on the server). All hcloud commands run from the local machine.

- Binary: `/opt/homebrew/bin/hcloud`
- Active context: `plane`
- Context contains the API token — no additional auth needed

## Resource Names

| Resource | Name | ID |
|---|---|---|
| Server | `gitlab` | 127829571 |
| Volume | `gitlab-data` | 105492436 |
| Firewall | `gitlab-fw` | 10881107 |
| SSH Key | `logan` | 108228380 |
| Location | `ash` (Ashburn, VA) | — |

All resources are labeled `service=gitlab`.

## Server Management

```bash
# Info
hcloud server describe gitlab
hcloud server list
hcloud server ip gitlab              # Quick IP lookup

# Power
hcloud server shutdown gitlab        # Graceful shutdown (ACPI)
hcloud server poweroff gitlab        # Hard power off (last resort)
hcloud server poweron gitlab
hcloud server reboot gitlab          # Graceful reboot

# Resize (requires server to be off)
hcloud server shutdown gitlab
hcloud server change-type gitlab --server-type <type>   # e.g. ccx43, ccx53
hcloud server poweron gitlab
# Available types: ccx13 (2c/8GB), ccx23 (4c/16GB), ccx33 (8c/32GB),
#                  ccx43 (16c/64GB), ccx53 (32c/128GB), ccx63 (48c/192GB)

# Rebuild (DESTRUCTIVE — reinstalls OS, erases root disk)
# Volume data at /var/opt/gitlab survives since it's a separate block device
hcloud server rebuild gitlab --image ubuntu-24.04

# Delete (DESTRUCTIVE)
hcloud server delete gitlab
```

## Volume Management

```bash
# Info
hcloud volume describe gitlab-data
hcloud volume list

# Resize (online — no downtime, no detach needed)
hcloud volume resize gitlab-data --size 200   # Size in GB, can only grow
# Then SSH in and expand filesystem:
# ssh root@87.99.146.126 "resize2fs /dev/disk/by-id/scsi-0HC_Volume_105492436"

# Detach / attach (for migration)
hcloud volume detach gitlab-data
hcloud volume attach gitlab-data --server gitlab

# Delete (DESTRUCTIVE — all git repos, database, uploads gone)
hcloud volume delete gitlab-data
```

## Firewall Management

```bash
# Info
hcloud firewall describe gitlab-fw
hcloud firewall list

# Add a rule (e.g., allow port 2222 for alternate SSH)
hcloud firewall add-rule gitlab-fw \
  --direction in --protocol tcp --port 2222 \
  --source-ips 0.0.0.0/0 --source-ips ::/0 \
  --description "Alternate SSH"

# Remove a rule (must specify all fields exactly as created)
hcloud firewall delete-rule gitlab-fw \
  --direction in --protocol tcp --port 2222 \
  --source-ips 0.0.0.0/0 --source-ips ::/0

# Apply firewall to server (already applied)
hcloud firewall apply-to-resource gitlab-fw --type server --server gitlab

# Remove firewall from server
hcloud firewall remove-from-resource gitlab-fw --type server --server gitlab
```

## Snapshots & Backups

```bash
# Hetzner automated backups (already enabled)
# These are managed by Hetzner, no CLI commands to trigger them

# Manual snapshot (point-in-time image, good before risky changes)
hcloud server create-image gitlab --type snapshot --description "pre-upgrade-$(date +%Y%m%d)"

# List snapshots
hcloud image list --type snapshot

# Restore from snapshot (DESTRUCTIVE — replaces root disk)
hcloud server rebuild gitlab --image <snapshot-id>

# Delete old snapshot
hcloud image delete <snapshot-id>

# Disable/enable automated backups
hcloud server disable-backup gitlab
hcloud server enable-backup gitlab
```

## SSH Keys

```bash
hcloud ssh-key list
hcloud ssh-key create --name <name> --public-key-from-file ~/.ssh/id_ed25519.pub

# SSH into the server (key "logan" is already authorized)
ssh root@87.99.146.126
```

## Networking

```bash
# Current IPs
hcloud server describe gitlab -o format='{{.PublicNet.IPv4.IP}}'
# IPv4: 87.99.146.126
# IPv6: 2a01:4ff:f0:efaa::1

# Reverse DNS
hcloud server set-rdns gitlab --ip 87.99.146.126 --hostname git.primary-os.com

# Floating IPs (for IP migration between servers)
hcloud floating-ip create --type ipv4 --home-location ash --description "gitlab"
hcloud floating-ip assign <id> gitlab
```

## Monitoring & Metrics

```bash
# Hetzner metrics (CPU, disk, network) — available in Hetzner Cloud Console
# No CLI equivalent, but you can check from the server directly:
ssh root@87.99.146.126 "top -bn1 | head -5 && echo '---' && free -h && echo '---' && df -h /"
```

## Cost Management

```bash
# No direct pricing CLI, but for reference:
# CCX33 (current): ~€51.50/month
# Volume 100GB: ~€4.80/month
# Backups: ~20% of server cost (~€10.30/month)
# Snapshots: €0.0108/GB/month (billed by actual image size)
# Total estimate: ~€67/month
```

## Common Workflows

### Pre-upgrade safety snapshot
```bash
hcloud server create-image gitlab --type snapshot --description "pre-upgrade-$(date +%Y%m%d)"
# Note the image ID from output, then proceed with upgrade
```

### Migrate to bigger server
```bash
# 1. Snapshot current state
hcloud server create-image gitlab --type snapshot --description "pre-resize"
# 2. Shutdown, resize, start
hcloud server shutdown gitlab
hcloud server change-type gitlab --server-type ccx43
hcloud server poweron gitlab
# 3. Volume auto-reattaches, IP stays the same, no DNS change needed
```

### Full disaster recovery (server destroyed)
```bash
# 1. Create new server
hcloud server create --name gitlab --type ccx33 --image ubuntu-24.04 --location ash --ssh-key logan --firewall gitlab-fw
# 2. Attach the surviving volume
hcloud volume attach gitlab-data --server gitlab
# 3. SSH in, mount volume, install GitLab, restore from backup
# See backup-restore.md for detailed restore procedure
# 4. Update DNS if IP changed
```
