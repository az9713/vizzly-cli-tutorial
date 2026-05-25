# Baseline System

A baseline is the approved "ground truth" version of a screenshot. Every new screenshot is compared against its baseline. Understanding how baselines are named, stored, and managed is essential for getting reliable results.

## Directory Structure

All Vizzly state lives in `.vizzly/` in your project root:

```
.vizzly/
├── server.json              ← server discovery beacon (deleted on stop)
├── baselines/               ← approved screenshots (commit these)
│   ├── homepage-a3f8c2d1.png
│   ├── homepage-b7e4f912.png   ← different baseline for different properties
│   └── checkout-c9d2e3f4.png
├── current/                 ← screenshots from the current run (temp)
│   ├── homepage-a3f8c2d1.png
│   └── checkout-c9d2e3f4.png
├── diff/                    ← diff images for failed comparisons (temp)
│   └── homepage-a3f8c2d1.png
└── report/                  ← static HTML report (temp)
    └── index.html
```

**What to commit:**
- `.vizzly/baselines/` — commit these to share approved truth with your team

**What to ignore:**
- `.vizzly/current/` — changes every run
- `.vizzly/diff/` — changes every run
- `.vizzly/report/` — generated each run

## Signature: How Baselines Are Named

Every baseline has a unique filename built from a **signature**: `<name>-<hash>.png`.

The hash is computed from:
1. The screenshot `name` you pass to `vizzlyScreenshot()`
2. The `properties` you pass (specifically, those in `signatureProperties`)

```javascript
// These two calls produce DIFFERENT baselines
await vizzlyScreenshot('homepage', screenshot1, {
  properties: { browser: 'chromium', viewport: '1920x1080' }
});

await vizzlyScreenshot('homepage', screenshot2, {
  properties: { browser: 'firefox', viewport: '1280x800' }
});
```

Even though both screenshots are named `homepage`, they get different hashes and different baseline files. This is intentional: the same page at different viewports or in different browsers legitimately looks different and deserves a separate baseline.

**What's in `signatureProperties`?** By default, `browser` and `viewport` are signature properties. Other properties you attach are stored as metadata but don't affect the signature (and therefore don't select a different baseline).

Example filenames:
```
homepage-a3f8c2d1.png          ← chromium, 1920x1080
homepage-b7e4f912.png          ← firefox, 1280x800
homepage-c1d5e8f2.png          ← chromium, 375x812 (mobile)
```

## Comparison Results: New / Passed / Failed

Each comparison has one of three outcomes:

| Status | Meaning | What to do |
|--------|---------|------------|
| **New** | No baseline exists for this signature | Accept it as a baseline if it looks correct |
| **Passed** | Delta E difference < threshold | Nothing — looks good |
| **Failed** | Delta E difference ≥ threshold | Review the diff; accept if intentional, fix if a regression |

## Setting Baselines

### Method 1: `--set-baseline` flag (bulk)

Accept all current screenshots as the new baseline in one command:

```bash
vizzly tdd run "npm test" --set-baseline
```

This copies every screenshot from `.vizzly/current/` to `.vizzly/baselines/`. Use this to:
- Set baselines on first run
- Accept all changes after a deliberate design update
- Reset after a framework upgrade that changes rendering

### Method 2: Dashboard (individual)

When running `vizzly tdd start`, the live dashboard shows each comparison. Click "Accept" on a specific screenshot to set just that one as the new baseline. Useful when you have mixed results: some intentional changes, some regressions.

### Method 3: `vizzly tdd run` with no flag (review first)

Run without `--set-baseline`, check the report, then decide:

```bash
vizzly tdd run "npm test"
# Review .vizzly/report/index.html
# If all looks good:
vizzly tdd run "npm test" --set-baseline
```

## Updating a Single Baseline

There's no CLI command to accept a single specific baseline (yet). The workflow is:

1. Run tests — note which screenshot you want to update
2. Delete its baseline file from `.vizzly/baselines/`
3. Run tests again — the screenshot will show as "new"
4. Run `--set-baseline` or accept from dashboard

## Environment-Specific Baselines

Use the `environment` setting to maintain separate baselines for different environments:

```javascript
// vizzly.config.js
export default {
  build: {
    environment: 'staging'  // 'test' by default
  }
};
```

Or via CLI:

```bash
vizzly tdd run "npm test" --environment staging
```

Baselines are namespaced by environment, so `staging` and `test` maintain independent approved screenshots. This is useful when your staging environment has real data and legitimately looks different from your test environment.

## When Baselines Go Stale

After a framework upgrade (e.g., Playwright or a component library), rendering may change subtly across all screenshots due to font rendering, subpixel antialiasing, or layout engine differences. You'll see many "failed" comparisons with small Delta E values.

The fix: reset all baselines.

```bash
# Delete all baselines
rm -rf .vizzly/baselines/

# Re-run with --set-baseline to accept new truth
vizzly tdd run "npm test" --set-baseline
```

This is expected behavior after dependency upgrades. Increase your threshold temporarily if the differences are consistently small and non-meaningful, or tune `minClusterSize` to ignore small pixel clusters. See [Tuning Thresholds](../guides/tuning-thresholds.md).

See also: [Comparison Engine](comparison-engine.md) | [Setting Baselines Guide](../guides/setting-baselines.md) | [TDD Server](tdd-server.md)
