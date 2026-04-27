# GitLab API Access

## Connection details

| Field | Value |
|---|---|
| GitLab URL | `https://git.primary-os.com` |
| API base | `https://git.primary-os.com/api/v4` |
| Project ID | `1` (primary-os/lovelace) |
| API token | `glpat-tr0oMjHfwtdxXKLUglCey286MQp1OjEH.01.0w11dy8o7` |

## Authentication

All API calls use the header:
```
PRIVATE-TOKEN: glpat-tr0oMjHfwtdxXKLUglCey286MQp1OjEH.01.0w11dy8o7
```

## Quick reference

```bash
# Set these for convenience
GITLAB_URL="https://git.primary-os.com"
TOKEN="glpat-tr0oMjHfwtdxXKLUglCey286MQp1OjEH.01.0w11dy8o7"
PROJECT_ID=1
```

## Web UI links

- Pipelines: `https://git.primary-os.com/primary-os/lovelace/-/pipelines`
- Jobs: `https://git.primary-os.com/primary-os/lovelace/-/jobs`
- CI/CD variables: `https://git.primary-os.com/primary-os/lovelace/-/settings/ci_cd`
- Pipeline schedules: `https://git.primary-os.com/primary-os/lovelace/-/pipeline_schedules`
- Protected environments: `https://git.primary-os.com/primary-os/lovelace/-/settings/ci_cd#js-protected-environments`
