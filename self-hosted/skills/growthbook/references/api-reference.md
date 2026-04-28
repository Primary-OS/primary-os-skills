# GrowthBook API Reference

## Endpoints

| Endpoint | URL | Purpose |
|----------|-----|---------|
| **Dashboard** | `https://gb.primary-os.com` | Web UI for managing flags, experiments, environments |
| **Admin API** | `https://gb-api.primary-os.com` | REST API for programmatic flag management |
| **SDK Proxy** | `https://gb-proxy.primary-os.com` | Public endpoint consumed by frontend GrowthBook SDK |

## Authentication

Two types of keys — never mix them up:

| Key type | Pattern | Purpose | Storage |
|----------|---------|---------|---------|
| **Admin API key** | `secret_admin_...` | REST API for managing flags | 1Password: "GrowthBook" in "Primary OS" vault |
| **SDK client keys** | `sdk-...` | Frontend GrowthBook SDK (public, safe to commit) | Build env vars |

### SDK client keys (per environment)

| Environment | SDK Key | Where set |
|-------------|---------|-----------|
| `dev` (local) | `sdk-UyN6muhM6HHpYW7f` | `env/.env.local` as `VITE_GROWTHBOOK_CLIENT_KEY` |
| `staging` | `sdk-oVPebIeSRcmu4` | CI/CD build env (GitLab CI variable) |
| `production` | `sdk-tJewUEqNSGIOHC4M` | CI/CD build env (GitLab CI variable) |

## Internal team emails

For team-only flag targeting (early access before GA):

| Name | Email |
|------|-------|
| Logan Nash | logan@primary.vc |
| Tanmaye Bhatia | tanmaye@primary.vc |
| Theo Bhatia | theo@primary.vc |
| Jordan Fox | jordan.fox@primary.vc |

For **local dev** environments, also include `dev@lovelace.test`.

## Common operations

### Check a flag's current state

```bash
GB_KEY="<admin API key from 1Password>"
curl -s "https://gb-api.primary-os.com/api/v1/features/FLAG_KEY" \
  -H "Authorization: Bearer $GB_KEY" | python3 -m json.tool
```

### List all features

```bash
curl -s "https://gb-api.primary-os.com/api/v1/features?limit=100" \
  -H "Authorization: Bearer $GB_KEY" | python3 -c "
import json, sys
data = json.loads(sys.stdin.read())
for f in data['features']:
    archived = ' [ARCHIVED]' if f.get('archived') else ''
    print(f'{f[\"id\"]:<35} type={f[\"valueType\"]:<10} default={f.get(\"defaultValue\", \"\")}' + archived)
"
```

### Enable a flag for the internal team only

The most common operation — ship a feature to the team for testing before GA. The default stays `false` so external users don't see it.

```bash
GB_KEY="<admin API key>"
PAYLOAD=$(cat <<'JSONEOF'
{
  "environments": {
    "dev": {
      "enabled": true,
      "rules": [
        {
          "type": "force",
          "description": "Enable for team",
          "value": "true",
          "enabled": true,
          "condition": "{\"email\": {\"$in\": [\"logan@primary.vc\", \"tanmaye@primary.vc\", \"theo@primary.vc\", \"jordan.fox@primary.vc\", \"dev@lovelace.test\"]}}"
        }
      ]
    },
    "staging": {
      "enabled": true,
      "rules": [
        {
          "type": "force",
          "description": "Enable for team",
          "value": "true",
          "enabled": true,
          "condition": "{\"email\": {\"$in\": [\"logan@primary.vc\", \"tanmaye@primary.vc\", \"theo@primary.vc\", \"jordan.fox@primary.vc\"]}}"
        }
      ]
    },
    "production": {
      "enabled": true,
      "rules": [
        {
          "type": "force",
          "description": "Enable for team",
          "value": "true",
          "enabled": true,
          "condition": "{\"email\": {\"$in\": [\"logan@primary.vc\", \"tanmaye@primary.vc\", \"theo@primary.vc\", \"jordan.fox@primary.vc\"]}}"
        }
      ]
    }
  }
}
JSONEOF
)
curl -s -X POST "https://gb-api.primary-os.com/api/v1/features/FLAG_KEY" \
  -H "Authorization: Bearer $GB_KEY" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD" | python3 -m json.tool
```

### Roll out to all users (GA)

Remove targeting rules and set the default to `true`:

```bash
PAYLOAD=$(cat <<'JSONEOF'
{
  "defaultValue": "true",
  "environments": {
    "dev": { "enabled": true, "rules": [] },
    "staging": { "enabled": true, "rules": [] },
    "production": { "enabled": true, "rules": [] }
  }
}
JSONEOF
)
curl -s -X POST "https://gb-api.primary-os.com/api/v1/features/FLAG_KEY" \
  -H "Authorization: Bearer $GB_KEY" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD" | python3 -m json.tool
```

### Disable a flag everywhere

```bash
PAYLOAD=$(cat <<'JSONEOF'
{
  "defaultValue": "false",
  "environments": {
    "dev": { "enabled": false, "rules": [] },
    "staging": { "enabled": false, "rules": [] },
    "production": { "enabled": false, "rules": [] }
  }
}
JSONEOF
)
curl -s -X POST "https://gb-api.primary-os.com/api/v1/features/FLAG_KEY" \
  -H "Authorization: Bearer $GB_KEY" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD" | python3 -m json.tool
```

### Create a new feature flag

```bash
PAYLOAD=$(cat <<'JSONEOF'
{
  "id": "new-flag-key",
  "valueType": "boolean",
  "defaultValue": "false",
  "environments": {
    "dev": { "enabled": true, "rules": [] },
    "staging": { "enabled": true, "rules": [] },
    "production": { "enabled": true, "rules": [] }
  }
}
JSONEOF
)
curl -s -X POST "https://gb-api.primary-os.com/api/v1/features" \
  -H "Authorization: Bearer $GB_KEY" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD" | python3 -m json.tool
```

### Custom user targeting

Target specific users by email:
```json
{
  "condition": "{\"email\": {\"$in\": [\"user1@example.com\", \"user2@example.com\"]}}"
}
```

Target by attribute:
```json
{
  "condition": "{\"is_admin\": true}"
}
```

## GrowthBook environments

Three environments are configured, mapping to Lovelace's deployment stages:

| Environment | SDK Key | Used by |
|-------------|---------|---------|
| `dev` | `sdk-UyN6muhM6HHpYW7f` | Local development (`bun dev`) |
| `staging` | `sdk-oVPebIeSRcmu4` | `staging.ada.primary-os.com` |
| `production` | `sdk-tJewUEqNSGIOHC4M` | `ada.primary-os.com` |

## SDK integration in Lovelace

The Lovelace web app (`apps/web`) uses `@growthbook/growthbook-react`:

```typescript
// apps/web/src/lib/growthbook.ts
const gb = new GrowthBook({
  apiHost: "https://gb-proxy.primary-os.com",
  clientKey: import.meta.env.VITE_GROWTHBOOK_CLIENT_KEY || "",
  enableDevMode: import.meta.env.DEV,
  trackingCallback: (experiment, result) => { ... }
});
```

User attributes are synced from Supabase auth via `GrowthBookAttributeSync` component:
```typescript
gb.setAttributes({
  id: session.user.id,
  email: session.user.email ?? "",
});
```

## Type generation

Feature flag types are auto-generated from the API:

```bash
# In the Lovelace repo
make flags
# or directly:
./scripts/generate-gb-types.sh
```

This produces `apps/web/src/lib/app-features.ts` with the `AppFeatures` type used for type-safe flag checks.

## Important notes

- The API **auto-publishes** — changes are live immediately, no separate publish step
- Boolean flags use string `"true"` / `"false"` in the API (not bare booleans)
- The GrowthBook CLI (`growthbook features toggle`) only supports on/off — it can't add targeting rules, so use the API directly for team-only rollouts
- The CLI config lives at `~/.growthbook/config.toml`
