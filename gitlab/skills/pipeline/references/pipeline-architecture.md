# Pipeline Architecture

## Overview

The Lovelace monorepo uses a single `.gitlab-ci.yml` with two deployment tracks in the same pipeline:

- **Staging**: Auto-deploys on push to main (path-filtered per service)
- **Production**: Manual one-click promote from any main pipeline (Logan only)

No production branch — everything deploys from main.

## Stages (in order)

```
check → migrate → deploy-tier-1 → deploy-tier-2 → e2e
  → production-migrate → production-tier-1 → production-tier-2 → scheduled
```

## Deployment order

```
Staging:
  supabase migrations (optional) → API + Slackbot (parallel) → MCP + Web (parallel) → E2E

Production:
  migrations-production (manual gate) → API + Slackbot (parallel) → MCP + Web (parallel)
```

## Jobs

### Check stage (MR pipelines only)

| Job | What it does | Triggers on |
|-----|-------------|-------------|
| api-docker-smoke-test | Builds API Docker image, health checks it | MR + API/packages changes |
| web-typecheck | Runs `bun run build` on web app | MR + web changes |
| supabase-tests | Runs migration checks and DB tests | MR or push to main |

### Staging (auto on push to main)

| Job | Deploy target | Triggers on |
|-----|--------------|-------------|
| supabase-deploy-staging | Supabase (migrations) | migration file changes |
| api-deploy-staging | Railway (staging) | API/packages changes |
| slackbot-deploy-staging | Railway (staging) | slackbot/packages changes |
| mcp-deploy-staging | Cloudflare Workers (staging) | MCP changes |
| web-deploy-staging | Vercel (staging.ada.primary-os.com) | web changes |
| e2e-staging | Playwright tests | any app/package changes |

### Production (manual trigger, full stack)

| Job | Deploy target | Depends on |
|-----|--------------|------------|
| migrations-production | Supabase (production) | nothing — manual gate |
| api-deploy-production | Railway (production) | migrations |
| slackbot-deploy-production | Railway (production) | migrations |
| mcp-deploy-production | Cloudflare Workers (production) | API + slackbot |
| web-deploy-production | Vercel (ada.primary-os.com) | API + slackbot |

### Scheduled

| Job | What it does | Schedule |
|-----|-------------|----------|
| compute-index-fetch | Fetches financial data | Daily 11:00 UTC (SCHEDULE_JOB=compute-index) |

## Service URLs

| Service | Staging | Production |
|---------|---------|------------|
| API | lovelace-api-staging.up.railway.app | lovelace.primary-os.com |
| Web | staging.ada.primary-os.com | ada.primary-os.com |
| MCP | (Cloudflare Workers staging) | (Cloudflare Workers production) |
