# Comparison Engine

Vizzly uses **honeydiff** (`@vizzly-testing/honeydiff`) to compare screenshots. This document explains how honeydiff works, what CIEDE2000 Delta E means in practice, and how to tune the comparison for your use case.

## Why Not Just Count Different Pixels?

Naive pixel-counting has a fundamental problem: it treats all differences equally. A 1-pixel antialiasing difference on a text edge counts the same as a button changing color from blue to red. This leads to either too many false positives (constant failures from rendering noise) or too many false negatives (real changes that get lost in the noise).

Honeydiff solves this with two mechanisms:
1. **Perceptual color difference** — uses CIEDE2000 Delta E, which weights color differences the way humans perceive them
2. **Cluster filtering** — ignores isolated pixel changes below a minimum size, eliminating rendering noise

## CIEDE2000 Delta E

Delta E (ΔE) is a number representing how different two colors look to a human observer. It's defined in the CIEDE2000 standard and operates in the CIE L*a*b* color space, which is designed to be perceptually uniform — equal Delta E values correspond to equal perceived differences across all hues and lightness levels.

| ΔE value | Human perception |
|----------|-----------------|
| 0.0 | Identical — no difference |
| 0.5 | Imperceptible difference |
| 1.0 | Just-noticeable difference (trained observer, controlled conditions) |
| 2.0 | Visible but small difference (Vizzly's default threshold) |
| 5.0 | Obvious difference — different shade |
| 10.0 | Very obvious difference |
| 20.0+ | Completely different color (e.g., blue vs. green) |

Vizzly compares each pixel pair, computes ΔE, and marks pixels where ΔE exceeds the threshold as "changed." The `diffPercentage` in the result represents what fraction of pixels in the image exceeded the threshold and belong to a cluster large enough to count.

## Cluster Filtering

After marking individual changed pixels, honeydiff groups them into **clusters** — contiguous regions of changed pixels. Clusters smaller than `minClusterSize` pixels are ignored.

This eliminates:
- Single-pixel antialiasing differences on text edges
- Subpixel rendering differences between GPU/OS configurations
- 1-2px border rendering variations in CSS

**Default:** `minClusterSize: 10` — clusters must be at least 10 pixels to count.

**Tuning:**
- Increase to `50` or `100` if you're seeing many false positives from rendering noise
- Decrease to `1` or `3` if you need to catch very small changes (e.g., icon pixel-perfect testing)

## Threshold Tuning Guide

The `threshold` value is the maximum acceptable ΔE before a pixel is considered "changed."

### Threshold 2.0 (default)

Good for: most web applications. Catches color changes, layout shifts, content changes. Tolerates minor rendering variation between OS/GPU configurations.

```javascript
export default {
  comparison: { threshold: 2.0 }
};
```

### Threshold 1.0 (strict)

Good for: design systems, marketing pages, brand-sensitive UIs where subtle color shifts matter.

```javascript
export default {
  comparison: { threshold: 1.0 }
};
```

With threshold 1.0, you'll catch changes a human might not consciously notice but that violate brand standards.

### Threshold 0.5 (very strict / pixel-perfect)

Good for: icon libraries, exact design spec validation. Expect more noise from cross-platform rendering.

### Threshold 5.0 (tolerant)

Good for: pages with dynamic content, data-driven charts, or heavy use of gradient/shadow rendering that differs between browsers.

### Threshold 0.0 (exact match)

Pixel-perfect. Any pixel difference fails. Only practical for very controlled rendering environments.

### Per-Screenshot Threshold Override

Override the threshold for a specific screenshot without changing the global config:

```javascript
await vizzlyScreenshot('chart-with-gradients', screenshot, {
  threshold: 5.0  // More tolerant for this specific screenshot
});

await vizzlyScreenshot('brand-logo', screenshot, {
  threshold: 0.5  // More strict for logo
});
```

## minClusterSize Tuning Guide

| minClusterSize | Use case |
|---------------|---------|
| 1 | Catch any changed pixel — maximum sensitivity |
| 5 | Very sensitive, minimal noise tolerance |
| 10 (default) | Balanced — good for most applications |
| 50 | Tolerant of rendering variation |
| 100 | Very tolerant — only catches large region changes |

Per-screenshot override:

```javascript
await vizzlyScreenshot('data-chart', screenshot, {
  minClusterSize: 50  // Ignore small rendering variations in chart
});
```

## Diff Image Format

When a comparison fails, Vizzly generates a diff PNG at `.vizzly/diff/<name>-<hash>.png`. The diff image highlights changed regions:

- **Red pixels**: individual pixels that exceed the threshold
- **Dark overlay**: unchanged pixels (dimmed to make changed areas stand out)
- Changed clusters that meet `minClusterSize` are highlighted

The diff image has the same dimensions as the baseline and current screenshot.

## Understanding diffPercentage

`diffPercentage` is a float between 0 and 100 representing the percentage of pixels in the image that:
1. Exceeded the threshold AND
2. Belong to a cluster ≥ minClusterSize

A `diffPercentage` of `0.12` means 0.12% of pixels changed. For a 1280×720 screenshot (921,600 total pixels), that's about 1,106 pixels.

The comparison **fails** when any cluster exists that meets the minClusterSize requirement, regardless of diffPercentage. The threshold is binary: exceeded or not.

## Practical Examples

**Scenario: Button color change (blue → slightly lighter blue)**

- ΔE ≈ 3.5 for affected pixels
- With threshold 2.0: fails (ΔE > 2.0)
- With threshold 5.0: passes (ΔE < 5.0)
- Cluster size: large (entire button), so minClusterSize doesn't matter

**Scenario: Font antialiasing difference between macOS and Linux**

- ΔE ≈ 0.8 per affected pixel
- With threshold 2.0: passes (ΔE < 2.0) — correct behavior, no false positive
- With threshold 0.5: fails — too strict for cross-platform testing

**Scenario: 2-pixel border rendering variation**

- ΔE ≈ 4.0 but cluster size = 4 pixels (thin border segment)
- With minClusterSize 10: passes — cluster too small to count
- With minClusterSize 1: fails

See also: [Tuning Thresholds Guide](../guides/tuning-thresholds.md) | [Baseline System](baseline-system.md) | [Configuration Reference](../reference/configuration.md)
