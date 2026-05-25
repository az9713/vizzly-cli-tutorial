# Onboarding: Visual Testing from Zero to Productive

This guide assumes you've never used Vizzly before. It builds the mental model first, then walks through a complete realistic scenario.

## The Core Problem Vizzly Solves

When you change CSS, how do you know you haven't broken something you weren't looking at?

You could write assertions: `expect(element).toHaveCSS('color', 'rgb(0, 0, 255)')`. But that's brittle — it tests the implementation, not the appearance. The button could have the right color but the wrong size. The layout could be broken on a viewport you didn't test.

Screenshot testing solves this by capturing "here's what it actually looks like" and comparing it to "here's what it's supposed to look like." The diff tells you exactly what changed, where, and by how much.

The problem with most screenshot tools: they're all-or-nothing. One pixel of antialiasing difference fails your build. You end up either ignoring failures or spending hours tuning fuzzy matchers.

Vizzly's approach: use a **perceptual color difference** metric (CIEDE2000 Delta E) that matches how humans actually perceive differences, group changed pixels into **clusters** to ignore rendering noise, and give you a **live dashboard** so you can review and accept changes instantly.

## Mental Model: What Is Actually Running

Think of Vizzly as three separate things that work together:

**1. A local HTTP server (the hub)**

When you run `vizzly tdd start` or `vizzly tdd run "..."`, a small Node.js HTTP server starts on port 47392. It's the brains of the operation. It stores baselines, does comparisons, and serves the dashboard.

**2. A tiny client library (the reporter)**

You add `vizzlyScreenshot()` to your tests. This function is extremely simple: it takes a screenshot buffer and POSTs it to the server. It has no comparison logic — it's just an HTTP client. The comparison happens on the server.

**3. A file system (persistent state)**

All baselines, current screenshots, and diff images are stored in `.vizzly/` in your project. The baselines directory is the "approved truth." The server reads and writes to this directory.

The client finds the server automatically. When `vizzlyScreenshot()` is called, it looks for `.vizzly/server.json` starting from the current directory and walking up the tree. That file contains the port number. No configuration needed.

## A Complete Realistic Scenario

Let's follow Maya, a frontend developer, through a typical day.

### Morning: Starting fresh on a new component

Maya is adding a new "Subscribe" button to the newsletter section of a marketing page. She wants to catch any visual regressions during development.

**She starts the TDD server:**

```bash
vizzly tdd start --open
```

The terminal shows:
```
  ʕ□ᴥ□ʔ  vizzly tdd  local
  Server running at http://localhost:47392
  Dashboard open in browser
```

The dashboard is now live in her browser.

**Her existing test file (already has screenshots for header, hero, footer):**

```javascript
// tests/marketing.spec.js
import { test } from '@playwright/test';
import { vizzlyScreenshot, vizzlyFlush } from '@vizzly-testing/cli/client';

test.afterAll(() => vizzlyFlush());

test('marketing page sections', async ({ page }) => {
  await page.goto('http://localhost:3000/marketing');
  await page.waitForLoadState('networkidle');

  // Existing screenshots
  let headerShot = await page.locator('header').screenshot();
  await vizzlyScreenshot('marketing-header', headerShot);

  let heroShot = await page.locator('.hero').screenshot();
  await vizzlyScreenshot('marketing-hero', heroShot);

  let footerShot = await page.locator('footer').screenshot();
  await vizzlyScreenshot('marketing-footer', footerShot);
});
```

**She adds her new component screenshot:**

```javascript
  // New: capture the newsletter section
  let newsletterShot = await page.locator('.newsletter').screenshot();
  await vizzlyScreenshot('marketing-newsletter', newsletterShot, {
    properties: { viewport: '1280x800', browser: 'chromium' }
  });
```

**She runs the tests:**

```bash
npx playwright test tests/marketing.spec.js
```

The dashboard immediately shows:
- `marketing-header` ✓ passed (ΔE 0.0 — unchanged)
- `marketing-hero` ✓ passed (ΔE 0.0 — unchanged)
- `marketing-footer` ✓ passed (ΔE 0.0 — unchanged)
- `marketing-newsletter` ● new — no baseline yet (shown in yellow)

**She accepts the newsletter baseline** by clicking "Accept" in the dashboard, or by running:

```bash
vizzly tdd run "npx playwright test" --set-baseline
```

Now `.vizzly/baselines/marketing-newsletter-a3f8c2d1.png` exists.

### Afternoon: Accidentally breaking the header

Maya is tweaking the CSS and accidentally introduces a bug — the `header` background color changes from `#1a1a2e` to `#1a1a3e` (one digit off). She doesn't notice visually.

She runs the tests:

```bash
npx playwright test tests/marketing.spec.js
```

The dashboard shows:
- `marketing-header` ✗ **failed** (ΔE 4.2)
- `marketing-hero` ✓ passed
- `marketing-footer` ✓ passed
- `marketing-newsletter` ✓ passed

The diff image highlights the header area in red. Maya clicks "View Diff" in the dashboard, sees the change, realizes it was a typo, fixes it, re-runs. Now everything passes.

### Late Afternoon: Intentional design change

The design team has approved a new hero section layout. The hero screenshot now looks different — intentionally.

Maya runs the tests:

```bash
npx playwright test tests/marketing.spec.js
```

Dashboard shows `marketing-hero` ✗ failed (ΔE 22.7). This time it's correct — the design changed. Maya reviews the diff, confirms it matches what design approved, and accepts it as the new baseline from the dashboard. From now on, the new hero is the "truth."

### End of Day: One-shot report for the PR

Before pushing, Maya wants a clean report to attach to her PR:

```bash
vizzly tdd run "npx playwright test" --no-open
```

This starts a fresh server, runs all tests, generates `.vizzly/report/index.html`, and exits. She attaches the report link to her PR.

If she wanted a machine-readable summary:

```bash
vizzly tdd run "npx playwright test" --json
```

```json
{
  "status": "completed",
  "exitCode": 0,
  "comparisons": [
    { "name": "marketing-header", "status": "passed", "diffPercentage": 0 },
    { "name": "marketing-hero", "status": "passed", "diffPercentage": 0.02 },
    { "name": "marketing-footer", "status": "passed", "diffPercentage": 0 },
    { "name": "marketing-newsletter", "status": "passed", "diffPercentage": 0 }
  ],
  "summary": { "total": 4, "passed": 4, "failed": 0, "new": 0 },
  "contextCommand": "vizzly context build current --source local --agent"
}
```

## Key Things to Remember

**You don't need a cloud account for any of this.** Local TDD mode is completely self-contained. The baselines live in `.vizzly/baselines/` in your project.

**The first run always shows "new".** Every screenshot will be "new" until you set a baseline. Run with `--set-baseline` to approve them all, or accept individually from the dashboard.

**The client disables itself gracefully if the server isn't running.** If you run your tests without Vizzly running, `vizzlyScreenshot()` returns `null` silently. Your tests don't fail. This is intentional — Vizzly is additive, not intrusive.

**Different properties = different baselines.** A screenshot named `homepage` on Chrome 1920px is a completely different baseline from `homepage` on Firefox 1280px. This is by design. Use consistent properties in each run and you get consistent comparison behavior.

## Next Steps

- [Quickstart](quickstart.md) — if you haven't run your first test yet
- [TDD Server](../concepts/tdd-server.md) — understand the server in depth
- [Baseline System](../concepts/baseline-system.md) — understand signatures and storage
- [Playwright Integration](../guides/playwright-integration.md) — full Playwright setup
