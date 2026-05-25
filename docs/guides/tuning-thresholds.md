# Tuning Thresholds

The comparison threshold controls how much visual difference is allowed before a screenshot is marked "failed." Picking the right threshold eliminates false positives without letting real regressions through.

## The Two Controls

**`threshold`** — Maximum CIEDE2000 Delta E per pixel. Pixels with ΔE above this value are marked "changed."

**`minClusterSize`** — Minimum number of contiguous changed pixels to count as a difference. Smaller clusters are ignored.

Both are configurable globally and per-screenshot.

## Quick Reference: Threshold Values

| Threshold | Sensitivity | Best for |
|-----------|------------|---------|
| 0.0 | Exact match | Controlled environments only |
| 0.5 | Very strict | Brand-sensitive design systems |
| 1.0 | Strict | Marketing pages, exact design spec |
| **2.0** | **Balanced (default)** | **Most web applications** |
| 3.0 | Tolerant | Slight cross-platform variation |
| 5.0 | Very tolerant | Charts, gradients, dynamic content |
| 10.0 | Loose | Highly variable renders |

## Decision Guide

### Scenario: Too many false positives (passes you don't want to fail)

Symptoms: Tests fail on CI but look identical visually. Small ΔE values in results (0.1–2.0). Failures appear on different OS/GPU configurations.

Cause: Font rendering, subpixel antialiasing, or GPU-specific rendering produces slightly different pixels.

Fix: Raise threshold or increase `minClusterSize`.

```javascript
// Option 1: Raise threshold globally
export default {
  comparison: { threshold: 3.0 }
};

// Option 2: Increase minimum cluster size (ignore small noise)
export default {
  comparison: { threshold: 2.0, minClusterSize: 30 }
};

// Option 3: Per-screenshot for the noisy ones only
await vizzlyScreenshot('gradient-hero', screenshot, {
  threshold: 5.0,
  minClusterSize: 50
});
```

### Scenario: Missing real regressions (failures you want to catch)

Symptoms: A button changed color and tests still pass. Layout shifted and tests didn't catch it.

Cause: Threshold is too high — real changes are within the tolerance.

Fix: Lower threshold. Check what ΔE value the regression would have.

```javascript
export default {
  comparison: { threshold: 1.0 }
};
```

After lowering, run your tests. If you get many new failures, some of them are probably rendering noise you were previously ignoring. Address those individually with per-screenshot thresholds rather than raising the global threshold back up.

### Scenario: Font antialiasing noise between macOS and Linux CI

The most common false-positive scenario. macOS uses subpixel antialiasing for text; Linux CI uses grayscale. Even small text can have ΔE 0.5–1.5 per affected pixel.

Fix: Use threshold 2.0 (or higher) and set baselines on the same OS as CI:

```bash
# If CI runs on Linux, set baselines on Linux too
# Or use a Docker container for consistency
docker run -v $(pwd):/app node:22 sh -c "cd /app && vizzly tdd run 'npm test' --set-baseline"
```

Alternatively, disable system fonts and use web fonts with consistent rendering:

```css
* {
  -webkit-font-smoothing: antialiased;
  -moz-osx-font-smoothing: grayscale;
}
```

### Scenario: Charts with random or real-time data

Data-driven charts render differently each time. Screenshot comparisons will always show some change.

**Option 1: Mock data in tests**

The best solution. Provide consistent test data so the chart always renders identically.

**Option 2: High threshold for chart screenshots only**

```javascript
await vizzlyScreenshot('revenue-chart', screenshot, {
  threshold: 10.0,
  minClusterSize: 200  // Only care about large changes
});
```

**Option 3: Exclude charts from visual testing**

```javascript
// Only take screenshots of stable UI elements
let headerShot = await page.locator('header').screenshot();
await vizzlyScreenshot('page-header', headerShot);
// Skip the chart section entirely
```

### Scenario: After Playwright or CSS framework upgrade

Many screenshots fail with small ΔE (0.5–2.5) after upgrading because the rendering engine changed subtly. These are not regressions — they're legitimate rendering differences from the new version.

Fix: Reset baselines.

```bash
rm -rf .vizzly/baselines/
vizzly tdd run "npm test" --set-baseline
```

Don't raise the threshold to "fix" this — reset the baselines instead. Raising the threshold permanently hides future regressions.

## Per-Screenshot Threshold Override

Override the threshold for specific screenshots without changing the global config:

```javascript
// Global: 2.0 (default)

// This screenshot needs to be exact — it's the logo
await vizzlyScreenshot('brand-logo', logoShot, {
  threshold: 0.5
});

// This chart is noisy — be tolerant
await vizzlyScreenshot('analytics-chart', chartShot, {
  threshold: 8.0,
  minClusterSize: 100
});

// This page changes frequently — medium tolerance
await vizzlyScreenshot('dashboard', dashShot, {
  threshold: 3.0
});
```

## Debugging Noisy Diffs

When you're getting unexpected failures, inspect the diff details:

```bash
vizzly tdd run "npm test" --json | jq '.comparisons[] | select(.status == "failed") | {name, diffPercentage, threshold}'
```

Example output:

```json
{
  "name": "homepage",
  "diffPercentage": 0.02,
  "threshold": 2.0
}
```

A `diffPercentage` of 0.02% means very few pixels changed. If those pixels are in a cluster just barely over `minClusterSize`, raising `minClusterSize` will eliminate the false positive:

```javascript
await vizzlyScreenshot('homepage', screenshot, {
  minClusterSize: 30  // Was 10, now ignores tiny clusters
});
```

Also inspect the diff image directly:

```bash
# Open the diff image to see exactly what changed
open .vizzly/diff/homepage-a3f8c2d1.png
```

Red highlighted areas in the diff show where pixels exceeded the threshold. If they're scattered single pixels along text edges — that's antialiasing noise. If they're a solid block — that's a real change.

## Reference: What ΔE Values Mean Visually

| ΔE range | Typical cause | Recommendation |
|----------|--------------|---------------|
| 0–0.5 | Imperceptible; subpixel rendering | Use minClusterSize ≥ 10 to ignore |
| 0.5–1.5 | Font antialiasing, OS rendering differences | Threshold 2.0 handles this |
| 1.5–3.0 | Small color shift, slight layout difference | Threshold 2.0 catches this |
| 3.0–10 | Visible color change, clear layout shift | Any threshold ≤ 5.0 catches this |
| 10–30 | Major color change (e.g., theme swap) | Catches with any threshold |
| 30+ | Completely different content | Always caught |

See also: [Comparison Engine](../concepts/comparison-engine.md) | [Configuration Reference](../reference/configuration.md) | [Common Issues](../troubleshooting/common-issues.md)
