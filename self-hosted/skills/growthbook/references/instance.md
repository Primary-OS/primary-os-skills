# GrowthBook Instance Details

## Server

| Property | Value |
|----------|-------|
| **Provider** | Hetzner Cloud |
| **Server name** | `growthbook` |
| **Server type** | CAX21 (ARM, 4 shared cores, 8 GB RAM, 80 GB local disk) |
| **OS** | Ubuntu 24.04 (ARM) |
| **IPv4** | `162.55.44.50` |
| **IPv6** | `2a01:4f8:c014:850::/64` |
| **Location** | Falkenstein DC Park 1 (fsn1), Germany |
| **Created** | 2026-03-26 |
| **SSH** | `ssh root@162.55.44.50` (key: `logan`) |

### Security posture

- **Firewall**: None applied (all ports open — should add `growthbook-fw`)
- **Volumes**: None attached (data on local 80 GB disk)
- **Backups**: Disabled (rely on MongoDB dumps)
- **Cloudflare proxy**: Disabled on all three subdomains (direct A records)

## DNS records (Cloudflare — primary-os.com zone)

| Subdomain | Type | Value | Proxied | Purpose |
|-----------|------|-------|---------|---------|
| `gb.primary-os.com` | A | `162.55.44.50` | No | Dashboard |
| `gb-api.primary-os.com` | A | `162.55.44.50` | No | Admin REST API |
| `gb-proxy.primary-os.com` | A | `162.55.44.50` | No | SDK proxy (frontend clients) |

All three resolve to the same server — GrowthBook routes internally based on hostname.

## Docker Compose architecture

GrowthBook runs via Docker Compose at `/opt/growthbook/` on the server.

### Containers

| Container | Image | Purpose | Ports |
|-----------|-------|---------|-------|
| `growthbook-growthbook-1` | `growthbook/growthbook:latest` | App server (dashboard + API + proxy) | 443 (HTTPS) |
| `growthbook-mongo-1` | `mongo:7.0` | MongoDB database | 27017 (internal only) |

### Volume mounts

```yaml
volumes:
  # MongoDB data
  - mongo-data:/data/db

  # GrowthBook uploads
  - uploads:/usr/local/src/app/packages/back-end/uploads

  # Enterprise patch — shadows the image's licenseUtil.js with patched version
  - ./licenseUtil.js:/usr/local/src/app/packages/back-end/dist/enterprise/licenseUtil.js:ro
```

The `licenseUtil.js` mount is what enables enterprise features without a license key. See `enterprise-patch.md` for details.

### Key environment variables

| Variable | Purpose |
|----------|---------|
| `APP_ORIGIN` | Dashboard URL: `https://gb.primary-os.com` |
| `API_HOST` | API URL: `https://gb-api.primary-os.com` |
| `MONGODB_URI` | MongoDB connection (points to co-located container) |
| `JWT_SECRET` | Session signing |
| `ENCRYPTION_KEY` | Data encryption at rest |
| `PROXY_ENABLED` | Enables the SDK proxy at `gb-proxy.primary-os.com` |

### TLS

TLS is handled on the server (not Cloudflare). GrowthBook's built-in Caddy server handles Let's Encrypt certificate issuance and renewal for all three subdomains. This is why the Cloudflare DNS records are NOT proxied — proxying would cause double-TLS issues.

## File layout on server

```
/opt/growthbook/
├── docker-compose.yml      # Main compose config
├── licenseUtil.js          # Patched enterprise license file (host copy)
└── backups/                # MongoDB dump archives (manual)
```

## Related infrastructure

| Service | Server | IP |
|---------|--------|----|
| GitLab | `gitlab` (Hetzner, Ashburn) | `87.99.146.126` |
| Affinity MCP | `affinity-mcp` (Hetzner, Ashburn) | `5.161.67.124` |
| **GrowthBook** | **`growthbook` (Hetzner, Falkenstein)** | **`162.55.44.50`** |

Note: GrowthBook is in Falkenstein (Germany) while GitLab and Affinity MCP are in Ashburn (US). This is a geographic separation — latency between them is ~100ms but they don't directly communicate.
