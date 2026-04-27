# Deploy Patterns & Gotchas

## Service deploy targets

| Service | Platform | Staging | Production |
|---------|----------|---------|------------|
| API | Railway | lovelace-api-staging.up.railway.app | lovelace.primary-os.com |
| Slackbot | Railway | — | — |
| MCP | Cloudflare Workers | staging env | production env |
| Web | Vercel | staging.ada.primary-os.com | ada.primary-os.com |
| Supabase | Supabase CLI | edrrcgieovgiktldcpel | lpzjonmdgblrdabzytfu |

## Known gotchas

### Railway

- **Uses debian:bookworm-slim**, not alpine — Railway CLI needs glibc
- **Tokens are environment-scoped** — staging and production need separate tokens (RAILWAY_TOKEN and RAILWAY_TOKEN_PRODUCTION)
- **Production jobs use `export RAILWAY_TOKEN="$RAILWAY_TOKEN_PRODUCTION"`** in the script, not the YAML `variables:` block — GitLab group/project-level CI variables override YAML job-level variables
- Service names: `lovelace-api` (API), `ada-slack-bot` (Slackbot)

### Vercel

- **All commands need `--scope=$VERCEL_ORG_ID`** — without it, Vercel defaults to personal scope
- **`vercel deploy --prebuilt --prod` hangs indefinitely** — use the sed+deploy+alias workaround instead:
  1. Build with `--prod` (production env vars)
  2. `sed -i s/production/preview/ .vercel/output/builds.json` (patch target)
  3. Deploy without `--prod` (fast, no hang)
  4. Alias to production domain
- **GitHub Git integration was disconnected** (2026-04-27) — deploys are CLI-only via GitLab CI
- Vercel project: `ada-web` (prj_jMTQZALHzO4s5GnVJB46FrkWWSSo)
- Team: team_fP5YPM3ex1cpHh3BROUXmL0N

### Docker-in-Docker (DinD)

- Must set `DOCKER_TLS_CERTDIR: ""` and `DOCKER_HOST: "tcp://docker:2375"`
- Health check URL: `http://docker:8000/health` (not localhost)

### YAML

- **sed with double quotes breaks YAML** — use `sed -i s/foo/bar/ file` (no quotes)
- **Masked variables can't contain spaces** — relevant for app passwords

### GitLab CI variable precedence

Variables override in this order (highest wins):
1. Trigger/manual/scheduled pipeline variables
2. **Project-level CI variables** (Settings → CI/CD → Variables)
3. **Group-level CI variables** (Group → Settings → CI/CD → Variables)
4. Instance-level variables
5. Inherited env vars
6. **YAML job-level `variables:`** ← this is LOWER priority than group or project

This means you cannot override a group-level or project-level variable with a job's `variables:` block. Use `export` in the script instead.

### Variable locations

Shared deploy platform tokens are at the **group level** (Primary-OS):
- `RAILWAY_TOKEN`, `RAILWAY_TOKEN_PRODUCTION`, `VERCEL_TOKEN`, `VERCEL_ORG_ID`, `CLOUDFLARE_API_TOKEN`

Project-specific variables stay at the **project level** (e.g., Lovelace):
- `SUPABASE_ACCESS_TOKEN`, `VERCEL_PROJECT_ID`, `SLACK_BOT_TOKEN`, etc.
