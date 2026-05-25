# Setting Baselines

Baselines are the approved "ground truth" screenshots. This guide covers every workflow for creating, updating, and managing them.

## First-Time Setup: Setting All Baselines

On a fresh project, there are no baselines. All screenshots show as "new" on the first run. Accept them all at once:

```bash
vizzly tdd run "npm test" --set-baseline
```

This copies every screenshot from `.vizzly/current/` to `.vizzly/baselines/`. After this, subsequent runs compare against these approved screenshots.

**Verify it worked:**

```bash
ls .vizzly/baselines/
# Should show PNG files like: homepage-a3f8c2d1.png

# Run again without --set-baseline to confirm passes
vizzly tdd run "npm test"
# Summary: 0 new, N passed, 0 failed ✓
```

## Accepting All Changes After a Design Update

Your design team has shipped a new look. Many screenshots will fail. Accept them all:

```bash
# 1. See what changed
vizzly tdd run "npm test"
# Review .vizzly/report/index.html

# 2. If all changes look intentional, accept everything
vizzly tdd run "npm test" --set-baseline

# 3. Commit the new baselines
git add .vizzly/baselines/
git commit -m "🔖 update visual baselines for new design system"
```

## Accepting Individual Screenshots (Dashboard)

When you have mixed results — some intentional changes, some regressions — use the live dashboard to accept specific screenshots:

```bash
# Start persistent server with dashboard
vizzly tdd start --open

# In another terminal, run tests
npm test
```

In the dashboard:
1. Failed screenshots appear with a diff image
2. Review each one
3. Click "Accept" to set as new baseline for that specific screenshot only
4. Regressions: don't accept; fix the code and re-run

## Accepting a Single Screenshot via CLI

There's no direct CLI command to accept a single screenshot. The workflow:

```bash
# 1. Find which baseline to replace
ls .vizzly/baselines/ | grep homepage

# 2. Delete the old baseline
rm .vizzly/baselines/homepage-a3f8c2d1.png

# 3. Run tests — it will show as "new"
vizzly tdd run "npm test"

# 4. Accept it specifically
vizzly tdd run "npm test" --set-baseline
# Note: this accepts ALL new/current screenshots, not just homepage
```

If you only want to accept one specific screenshot, delete its baseline, run with `--set-baseline`, then `git add` only the new baseline file.

## Resetting All Baselines

After a major dependency upgrade (Playwright, a CSS framework, Node.js) that changes rendering across all screenshots:

```bash
# 1. Delete all existing baselines
rm -rf .vizzly/baselines/

# 2. Run and set new baselines
vizzly tdd run "npm test" --set-baseline

# 3. Commit
git add .vizzly/baselines/
git commit -m "🔖 reset baselines after playwright upgrade"
```

## Environment-Specific Baselines

Use different baselines for different environments. This is useful when staging has real data that makes it look different from your test environment.

**In `vizzly.config.js`:**

```javascript
export default {
  build: {
    environment: 'test'  // default
  }
};
```

**Override for staging:**

```bash
vizzly tdd run "npm test" --environment staging --set-baseline
```

Baselines for `staging` are stored separately from `test` baselines. The same screenshot name on different environments uses different baseline files (the environment is included in the signature computation).

**Verify:**

```bash
ls .vizzly/baselines/
# Will show files for both environments, differentiated by hash
```

## Committing Baselines to Git

Committing baselines enables:
- Team members to run tests without needing to set baselines themselves
- CI to compare against approved baselines (local mode)
- Historical tracking of approved UI state

Recommended `.gitignore` additions:

```gitignore
# Vizzly — commit baselines, ignore ephemeral files
.vizzly/current/
.vizzly/diff/
.vizzly/report/
.vizzly/server.json
# .vizzly/baselines/ is intentionally NOT ignored
```

```bash
git add .vizzly/baselines/
git commit -m "🔖 add visual baselines for homepage, checkout, marketing"
```

For large teams: consider storing baselines in Git LFS to avoid bloating the repository with binary files.

```bash
git lfs track "*.png"
git add .gitattributes
git commit -m "🔧 track PNG files with Git LFS"
```

## Downloading Baselines from a Cloud Build

If your team uses cloud mode and has approved baselines on vizzly.dev, download them for local TDD use:

```bash
vizzly tdd start --baseline-build abc123def456
```

This downloads baselines from the specified cloud build and uses them for local comparisons. Requires `VIZZLY_TOKEN`.

## Baseline Strategy for Teams

| Strategy | When to use | Trade-offs |
|----------|-------------|-----------|
| Commit baselines to Git | Small teams, stable rendering environment | Simple; baselines reviewed in PRs |
| Commit baselines to Git LFS | Large projects with many/large screenshots | Keeps repo size manageable |
| Cloud baselines only | Large teams, multi-env CI | Requires token everywhere; no offline mode |
| Hybrid: local dev, cloud CI | Most common for growing teams | Best of both; local TDD + cloud review in PRs |

See also: [Baseline System](../concepts/baseline-system.md) | [CI/CD Integration](ci-cd-integration.md) | [Configuration Reference](../reference/configuration.md)
