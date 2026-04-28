# GrowthBook Enterprise Patch

The self-hosted GrowthBook instance runs on the enterprise plan without a license key. This is achieved by patching a single file inside the container.

## How it works

GrowthBook's backend checks `licenseUtil.js` to determine the current plan. The function `getEffectiveAccountPlan` has a fallback for self-hosted instances without a license key — by default it returns `"oss"` (the free open-source tier). The patch changes this to `"enterprise"`.

The patched file lives at `/opt/growthbook/licenseUtil.js` on the host and is bind-mounted read-only into the container via docker-compose.yml:

```yaml
volumes:
  - ./licenseUtil.js:/usr/local/src/app/packages/back-end/dist/enterprise/licenseUtil.js:ro
```

The `:ro` mount shadows the image's original file on every container start.

## Why it must be re-applied after image updates

When `docker compose pull` fetches a new GrowthBook image, the image's `licenseUtil.js` may have changed (refactored, new logic, different variable names). The host's patched file is based on the **previous** image's version. If the file structure changed, the patched file might not be compatible — GrowthBook could crash or silently revert to OSS behavior.

The safe procedure: extract the fresh file from the new image, re-apply the patch, then restart.

## Full upgrade procedure

```bash
ssh root@162.55.44.50
cd /opt/growthbook

# 1. Pull the latest image
docker compose pull growthbook

# 2. Start the new container (picks up new image, still has old patched mount)
docker compose up -d growthbook

# 3. Extract the fresh licenseUtil.js from the new container
docker cp growthbook-growthbook-1:/usr/local/src/app/packages/back-end/dist/enterprise/licenseUtil.js /opt/growthbook/licenseUtil.js

# 4. Apply the enterprise patch
sed -i 's/basicPlan = "oss"/basicPlan = "enterprise"/' /opt/growthbook/licenseUtil.js

# 5. Verify the patch landed
grep 'basicPlan' /opt/growthbook/licenseUtil.js
# Expected: basicPlan = "enterprise"  (not "oss")

# 6. Restart so the container picks up the re-patched file
docker compose up -d growthbook

# 7. Check logs for clean startup
docker logs growthbook-growthbook-1 --tail 20
```

## Verifying the patch

After restart, confirm enterprise features are active:

1. **From the CLI**:
   ```bash
   ssh root@162.55.44.50 "grep 'basicPlan' /opt/growthbook/licenseUtil.js"
   ```
   Should show `basicPlan = "enterprise"`.

2. **From the dashboard**: Visit https://gb.primary-os.com and check that gated features (e.g., "Restrict Access to Specific Environments") are accessible without a pro/enterprise upgrade prompt.

## If the patch fails

If `grep 'basicPlan'` doesn't show `"enterprise"` after the sed command, it means GrowthBook refactored the file in the new version and the sed pattern didn't match.

**Recovery steps:**

1. Look at the new file's structure:
   ```bash
   grep -n 'getEffectiveAccountPlan\|basicPlan\|oss\|enterprise' /opt/growthbook/licenseUtil.js
   ```

2. Find the line that sets the self-hosted fallback plan (the `else` branch of the `IS_CLOUD` check).

3. Patch that line to return `"enterprise"` instead of whatever the free tier is called.

4. Restart and verify:
   ```bash
   docker compose up -d growthbook
   docker logs growthbook-growthbook-1 --tail 20
   ```

## Enterprise features enabled by the patch

- Environment-level access control (restrict who can modify production flags)
- Advanced targeting rules (multi-attribute, percentage rollouts with sticky bucketing)
- Audit log
- SCIM provisioning
- Custom roles and permissions
- Visual editor for A/B tests
- Multi-org support
