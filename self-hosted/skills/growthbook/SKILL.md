---
name: growthbook
description: >
  Manage the self-hosted GrowthBook instance at gb.primary-os.com.
  Use when the user asks to: manage GrowthBook, upgrade GrowthBook,
  check GrowthBook status, restart GrowthBook, back up GrowthBook,
  manage feature flags via the admin API, check flag state, enable/disable
  a flag, roll out a feature, target the internal team, manage GrowthBook
  SDK keys, manage GrowthBook environments, check GrowthBook logs,
  troubleshoot GrowthBook errors, manage the GrowthBook server on Hetzner,
  apply the enterprise patch, "is GrowthBook up", "gb is down",
  "update growthbook", "growthbook backup", resize the GrowthBook server,
  add a firewall, manage DNS for gb.primary-os.com, or any task involving
  the self-hosted GrowthBook feature flagging platform.
---

# GrowthBook Instance Management

Manage Primary's self-hosted GrowthBook instance running on Hetzner Cloud. This skill covers server administration, upgrades (with enterprise patch), backups, feature flag management via the API, and troubleshooting.

## Required context

Before performing any operation, load these references as needed:

- **Always load**: `${CLAUDE_PLUGIN_ROOT}/skills/growthbook/references/instance.md` — server IP, DNS, Docker Compose config, architecture
- **For upgrades**: `${CLAUDE_PLUGIN_ROOT}/skills/growthbook/references/enterprise-patch.md` — the enterprise license patch that must be re-applied after every image pull
- **For flag management / API**: `${CLAUDE_PLUGIN_ROOT}/skills/growthbook/references/api-reference.md` — REST API endpoints, SDK keys, team targeting, environment config
- **For hcloud / Hetzner operations**: `${CLAUDE_PLUGIN_ROOT}/../gitlab/references/hcloud.md` — hcloud CLI reference (shared with GitLab skill)

## Procedure

### Step 1: Identify the operation category

Map the user's request to one of these categories:

- **A. Health check / status** — is GrowthBook up, container status, resource usage
- **B. Feature flag management** — enable/disable flags, team targeting, GA rollout, check flag state
- **C. Upgrade** — pull new image, re-apply enterprise patch, restart
- **D. Backup & restore** — back up MongoDB data, restore from backup
- **E. Server management** — resize, firewall, snapshots, storage via hcloud
- **F. Configuration** — Docker Compose changes, environment variables, SSL
- **G. Troubleshooting** — dashboard unreachable, API errors, proxy issues, high resource usage
- **H. DNS management** — update Cloudflare DNS records for gb/gb-api/gb-proxy subdomains

### Step 2A: Health check / status

```bash
ssh root@162.55.44.50 "docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}' && echo '---' && free -h && echo '---' && df -h /"
```

Verify all three endpoints externally:
```bash
curl -sI https://gb.primary-os.com/ | head -5
curl -sI https://gb-api.primary-os.com/ | head -5
curl -sI https://gb-proxy.primary-os.com/ | head -5
```

Report: container status, uptime, memory/disk usage, HTTPS reachability for all three subdomains.

### Step 2B: Feature flag management

Load API reference: `${CLAUDE_PLUGIN_ROOT}/skills/growthbook/references/api-reference.md`

The API reference has complete procedures for:
- Checking a flag's current state
- Enabling a flag for the internal team only (most common operation)
- Rolling out a flag to all users (GA)
- Disabling a flag everywhere
- Custom user targeting

**Key details:**
- Admin API key is stored in 1Password ("GrowthBook" item in "Primary OS" vault)
- API auto-publishes — changes are live immediately, no separate publish step
- Boolean flag values are strings (`"true"` / `"false"`) in the API
- Three environments: `dev`, `staging`, `production`

### Step 2C: Upgrade

Load enterprise patch reference: `${CLAUDE_PLUGIN_ROOT}/skills/growthbook/references/enterprise-patch.md`

**Critical**: The enterprise patch must be re-applied after every image pull. If skipped, the instance reverts to the free OSS tier and enterprise features (environment access control, etc.) disappear.

Upgrade procedure summary:
1. SSH into `162.55.44.50`
2. `cd /opt/growthbook`
3. `docker compose pull growthbook`
4. `docker compose up -d growthbook` (start with new image)
5. Extract fresh `licenseUtil.js` from container
6. Apply enterprise patch via `sed`
7. Verify patch with `grep`
8. `docker compose up -d growthbook` (restart with patched file)
9. Check logs and verify dashboard

Full step-by-step in the enterprise patch reference.

### Step 2D: Backup & restore

GrowthBook uses MongoDB for all persistent data. The MongoDB data lives in a Docker volume.

**Create a backup:**
```bash
ssh root@162.55.44.50 "cd /opt/growthbook && docker compose exec mongo mongodump --archive=/tmp/gb-backup-\$(date +%Y%m%d).archive --gzip && docker cp growthbook-mongo-1:/tmp/gb-backup-\$(date +%Y%m%d).archive /opt/growthbook/backups/"
```

**List backups:**
```bash
ssh root@162.55.44.50 "ls -lh /opt/growthbook/backups/ 2>/dev/null || echo 'No backups directory'"
```

**Restore from backup** (destructive — confirm with user first):
```bash
ssh root@162.55.44.50 "cd /opt/growthbook && docker cp /opt/growthbook/backups/BACKUP_FILE growthbook-mongo-1:/tmp/restore.archive && docker compose exec mongo mongorestore --archive=/tmp/restore.archive --gzip --drop"
```

**Download a backup locally:**
```bash
scp root@162.55.44.50:/opt/growthbook/backups/BACKUP_FILE ./
```

### Step 2E: Server management

The GrowthBook server is managed via `hcloud` (runs locally, not on the server).

**Server details:**
- Name: `growthbook`
- Type: CAX21 (ARM, 4 cores, 8 GB RAM, 80 GB local disk)
- IP: `162.55.44.50`
- Location: Falkenstein DC Park 1 (fsn1), Germany
- No volumes attached, no firewall, backups disabled

**Check server status:**
```bash
hcloud server describe growthbook
```

**Create a snapshot** (before risky operations):
```bash
hcloud server create-image growthbook --type snapshot --description "pre-upgrade $(date +%Y%m%d)"
```
Note: Server is briefly paused during snapshot creation.

**Resize server** (causes downtime):
```bash
# Power off first
hcloud server poweroff growthbook
# Resize (e.g., to cax31 for 8 cores / 16 GB)
hcloud server change-type growthbook cax31
# Power back on
hcloud server poweron growthbook
```

**Add a firewall** (recommended — currently none):
```bash
# Create firewall
hcloud firewall create --name growthbook-fw

# Add rules: SSH, HTTP (Let's Encrypt), HTTPS
hcloud firewall add-rule growthbook-fw --direction in --protocol tcp --port 22 --source-ips 0.0.0.0/0 --source-ips ::/0 --description "SSH"
hcloud firewall add-rule growthbook-fw --direction in --protocol tcp --port 80 --source-ips 0.0.0.0/0 --source-ips ::/0 --description "HTTP (Let's Encrypt)"
hcloud firewall add-rule growthbook-fw --direction in --protocol tcp --port 443 --source-ips 0.0.0.0/0 --source-ips ::/0 --description "HTTPS"
hcloud firewall add-rule growthbook-fw --direction in --protocol icmp --source-ips 0.0.0.0/0 --source-ips ::/0 --description "Ping"

# Apply to server
hcloud firewall apply-to-resource growthbook-fw --type server --server growthbook
```

### Step 2F: Configuration

**Docker Compose** lives at `/opt/growthbook/docker-compose.yml` on the server.

Key environment variables in the compose file:
- `APP_ORIGIN` — dashboard URL (`https://gb.primary-os.com`)
- `API_HOST` — API URL (`https://gb-api.primary-os.com`)
- `MONGODB_URI` — connection string to the co-located MongoDB container
- `JWT_SECRET` — session signing secret
- `ENCRYPTION_KEY` — data encryption key

**View current config:**
```bash
ssh root@162.55.44.50 "cat /opt/growthbook/docker-compose.yml"
```

**Edit config:**
```bash
ssh root@162.55.44.50 "cp /opt/growthbook/docker-compose.yml /opt/growthbook/docker-compose.yml.bak.\$(date +%Y%m%d%H%M) && nano /opt/growthbook/docker-compose.yml"
```

**Apply config changes:**
```bash
ssh root@162.55.44.50 "cd /opt/growthbook && docker compose up -d"
```

### Step 2G: Troubleshooting

**Dashboard unreachable (gb.primary-os.com):**
1. Check DNS resolves: `dig +short gb.primary-os.com` → should be `162.55.44.50`
2. Check containers running: `ssh root@162.55.44.50 "docker ps"`
3. Check logs: `ssh root@162.55.44.50 "docker logs growthbook-growthbook-1 --tail 50"`
4. Check if port 443 is open: `nc -zv 162.55.44.50 443`
5. Check disk space: `ssh root@162.55.44.50 "df -h /"`

**API errors (gb-api.primary-os.com):**
1. Test API directly: `curl -s https://gb-api.primary-os.com/api/v1/features -H "Authorization: Bearer $GB_KEY" | head -c 200`
2. Check GrowthBook logs for errors: `ssh root@162.55.44.50 "docker logs growthbook-growthbook-1 --tail 50 2>&1 | grep -i error"`

**SDK proxy issues (gb-proxy.primary-os.com):**
- The proxy is served by the same GrowthBook container
- Check that the `PROXY_ENABLED=1` env var is set in docker-compose.yml
- Test: `curl -s https://gb-proxy.primary-os.com/api/features/sdk-UyN6muhM6HHpYW7f | head -c 200`

**Enterprise features missing:**
- The patch reverted — see upgrade procedure in enterprise patch reference
- Verify: `ssh root@162.55.44.50 "grep 'basicPlan' /opt/growthbook/licenseUtil.js"`
- Should show `basicPlan = "enterprise"`, not `"oss"`

**High memory / slow performance:**
```bash
ssh root@162.55.44.50 "free -h && echo '---' && docker stats --no-stream"
```
Server has 8 GB RAM — if consistently above 80%, consider resizing to CAX31 (16 GB).

**MongoDB issues:**
```bash
ssh root@162.55.44.50 "docker logs growthbook-mongo-1 --tail 30"
```

### Step 2H: DNS management

All three GrowthBook subdomains are A records in Cloudflare (NOT proxied):

| Subdomain | Record | Value | Proxied |
|-----------|--------|-------|---------|
| `gb.primary-os.com` | A | `162.55.44.50` | No |
| `gb-api.primary-os.com` | A | `162.55.44.50` | No |
| `gb-proxy.primary-os.com` | A | `162.55.44.50` | No |

To update DNS (e.g., after server migration), use the Cloudflare API with the `edit-dns-token` from 1Password ("Cloudflare — API Tokens" in "Primary OS" vault):

```bash
CF_TOKEN=$(op read "op://Primary OS/Cloudflare — API Tokens/edit-dns-token")
ZONE_ID="fc32e7119c524bf0ba44573a7b9c23b5"

# Update a record (get record ID first)
curl -s -H "Authorization: Bearer $CF_TOKEN" \
  "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records?name=gb.primary-os.com" | python3 -m json.tool

# Then update
curl -s -X PUT -H "Authorization: Bearer $CF_TOKEN" -H "Content-Type: application/json" \
  "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/RECORD_ID" \
  -d '{"type":"A","name":"gb.primary-os.com","content":"NEW_IP","ttl":1,"proxied":false}'
```

**Note**: These records are intentionally NOT proxied through Cloudflare because GrowthBook handles its own TLS via Caddy/Let's Encrypt on the server. Enabling Cloudflare proxy would require disabling server-side TLS to avoid double encryption.

## Guardrails

- **Never delete the server or MongoDB volume** without explicit user confirmation
- **Always back up MongoDB before**: upgrades, config changes, restore operations
- **Always re-apply the enterprise patch** after pulling a new GrowthBook image
- **Always confirm before**: server resize (causes downtime), restore operations, DNS changes
- **Never commit** the admin API key (`secret_admin_...`) or `GB_KEY` — these are in 1Password only
- **SDK client keys** (`sdk-...`) are public and safe to reference in code/CI
- **Snapshot before risky operations** — `hcloud server create-image growthbook --type snapshot`
