# GitLab Instance Details

## Server

| Field | Value |
|---|---|
| Provider | Hetzner Cloud |
| hcloud context | `plane` |
| Server name | `gitlab` |
| Server type | CCX33 (8 dedicated vCPU, 32 GB RAM, 240 GB disk) |
| Location | Ashburn, VA (ash) |
| OS | Ubuntu 24.04 LTS |
| IP | `87.99.146.126` |
| SSH | `ssh root@87.99.146.126` |

## Domain & SSL

| Field | Value |
|---|---|
| URL | `https://git.primary-os.com` |
| DNS | Cloudflare, A record, DNS only (no proxy), auto TTL |
| SSL | Let's Encrypt via Omnibus, auto-renew every 4 days at 3 AM |

## Storage

| Field | Value |
|---|---|
| Root disk | 240 GB (OS + GitLab binaries) |
| Data volume | 100 GB Hetzner Volume (`gitlab-data`) |
| Volume device | `/dev/disk/by-id/scsi-0HC_Volume_105492436` |
| Mount point | `/var/opt/gitlab` (all git repos, uploads, registry, DB) |
| Filesystem | ext4 |

## GitLab

| Field | Value |
|---|---|
| Edition | Enterprise Edition (EE) free tier |
| Version | 18.11.1-ee (as of 2026-04-24) |
| Install method | Omnibus (Linux package) |
| Config file | `/etc/gitlab/gitlab.rb` |
| Secrets file | `/etc/gitlab/gitlab-secrets.json` |

## Firewall

Hetzner Cloud Firewall `gitlab-fw`:

| Port | Protocol | Purpose |
|---|---|---|
| 22 | TCP | SSH / Git over SSH |
| 80 | TCP | HTTP (Let's Encrypt challenges, redirects to 443) |
| 443 | TCP | HTTPS (web UI, Git over HTTPS, API) |
| ICMP | - | Ping |

All other inbound traffic is blocked.

## Backups

| Layer | Schedule | Retention | What |
|---|---|---|---|
| Hetzner server snapshots | Nightly (automatic) | ~7 rolling | Full server image |
| GitLab app backup | 2:00 AM daily (cron) | 30 days | Repos, issues, CI data, uploads |
| Config backup | 2:15 AM daily (cron) | 30 days | gitlab.rb + gitlab-secrets.json |

Backup location: `/var/opt/gitlab/backups/`
Cron config: `/etc/cron.d/gitlab-backup`

## SMTP

| Field | Value |
|---|---|
| Provider | Google Workspace (Gmail SMTP) |
| SMTP address | smtp.gmail.com |
| SMTP port | 587 |
| Auth user | logan@primary.vc |
| From address | logan@primary.vc (Gmail overrides the from to match the authenticated account) |
| Auth method | App Password (plain auth + STARTTLS) |

## GitLab Runner

| Field | Value |
|---|---|
| Location | Same server as GitLab (87.99.146.126) |
| Executor | Docker |
| Concurrent jobs | 2 |
| Privileged | Yes (for Docker-in-Docker) |
| Default image | alpine:latest |
| Scope | Group-level (Primary-OS group) |
| Config | `/etc/gitlab-runner/config.toml` |

## AI / Duo

| Setting | Value |
|---|---|
| `duo_features_enabled` | `true` |
| `duo_availability` | `default_on` |
| Feature flags enabled | `ai_duo_chat_switch`, `ai_global_switch`, `global_ai_catalog`, `ai_catalog_project_level_enablement`, `duo_chat` |
| License / activation | None — foundational agents require activation from customers.gitlab.com |
| AI Gateway | Not configured (requires activation) |

## CI/CD Group Variables (Primary-OS, group 6)

| Variable | Purpose | Masked |
|---|---|---|
| `CLOUDFLARE_API_TOKEN` | Cloudflare API | Yes |
| `RAILWAY_TOKEN` | Railway staging deploys | Yes |
| `RAILWAY_TOKEN_PRODUCTION` | Railway production deploys | Yes |
| `VERCEL_ORG_ID` | Vercel org | No |
| `VERCEL_TOKEN` | Vercel deploys | Yes |
| `CLAUDE_CODE_OAUTH_TOKEN` | Claude Code subscription auth for CI review jobs | Yes |
| `GITLAB_REVIEW_TOKEN` | GitLab API — post MR review comments | Yes |

## Security Settings

- Public sign-up: **disabled** (invite-only)
- Web IDE single origin fallback: **disabled**
