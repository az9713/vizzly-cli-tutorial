# Quickstart

Get Vizzly capturing and comparing screenshots in under 15 minutes. This guide uses Playwright, but the pattern is identical for any test framework.

## Step 1: Install

```bash
npm install -g @vizzly-testing/cli
npm install --save-dev @playwright/test
npx playwright install chromium
```

## Step 2: Initialize Vizzly in your project

Run this in your project root:

```bash
vizzly init
```

This creates `vizzly.config.js`:

```javascript
export default {
  server: { port: 47392, timeout: 30000 },
  build: { name: 'Build {timestamp}', environment: 'test' },
  comparison: { threshold: 2.0 },
  tdd: { openReport: false }
};
```

The `.vizzly/` directory (where baselines and reports are stored) is created automatically on first run. Add it to `.gitignore` if you don't want to commit baselines:

```bash
echo ".vizzly/" >> .gitignore
```

Or commit `.vizzly/baselines/` to share approved baselines with your team (recommended):

```bash
echo ".vizzly/current/" >> .gitignore
echo ".vizzly/diff/" >> .gitignore
echo ".vizzly/report/" >> .gitignore
# .vizzly/baselines/ is intentionally NOT ignored
```

## Step 3: Write a test with a screenshot

Create `tests/homepage.spec.js`:

```javascript
import { test, expect } from '@playwright/test';
import { vizzlyScreenshot, vizzlyFlush } from '@vizzly-testing/cli/client';

test.afterAll(async () => {
  await vizzlyFlush();
});

test('homepage looks correct', async ({ page }) => {
  await page.goto('https://example.com');
  await page.waitForLoadState('networkidle');

  let screenshot = await page.screenshot();

  await vizzlyScreenshot('homepage', screenshot, {
    properties: {
      browser: 'chromium',
      viewport: '1280x720'
    }
  });

  // Your normal assertions still work alongside Vizzly
  await expect(page).toHaveTitle(/Example Domain/);
});
```

## Step 4: Run your first visual test

```bash
vizzly tdd run "npx playwright test"
```

Expected output:

```
  Initializing TDD server...

  Running: npx playwright test

  Running tests 1

  Visual Results
  ──────────────────────────────────────
  ● homepage [new] — no baseline yet

  Summary: 1 new, 0 passed, 0 failed

  Report: .vizzly/report/index.html
  Context  vizzly context build current --source local --agent
```

The first run marks all screenshots as "new" because there are no baselines yet. This is expected.

## Step 5: Accept baselines

Accept all current screenshots as the approved baselines:

```bash
vizzly tdd run "npx playwright test" --set-baseline
```

Expected output:

```
  Visual Results
  ──────────────────────────────────────
  ✓ homepage [baseline set]

  Summary: 1 baseline(s) set
```

Now `.vizzly/baselines/homepage-<hash>.png` exists on disk.

## Step 6: Run again to verify

```bash
vizzly tdd run "npx playwright test"
```

Expected output:

```
  Visual Results
  ──────────────────────────────────────
  ✓ homepage [passed] ΔE 0.00

  Summary: 0 new, 1 passed, 0 failed
```

The screenshot matches the baseline exactly. Exit code is 0.

## Step 7: See a failure

Change something in your test — for example, navigate to a different URL — and run again:

```bash
vizzly tdd run "npx playwright test"
```

If the page looks different from the baseline:

```
  Visual Results
  ──────────────────────────────────────
  ✗ homepage [failed] ΔE 18.42 (threshold: 2.0)

  Summary: 0 new, 0 passed, 1 failed
```

The exit code is 1. The diff image is at `.vizzly/diff/homepage-<hash>.png`. The static HTML report at `.vizzly/report/index.html` shows side-by-side baseline vs. current with highlighted diff regions.

## Step 8: Use the live dashboard (optional)

Instead of one-shot runs, you can keep a server running and watch diffs appear in real time:

```bash
# Terminal 1: start the persistent server
vizzly tdd start --open

# Terminal 2: run tests in watch mode
npx playwright test --watch
```

The `--open` flag opens the dashboard in your browser automatically. Every test run sends results to the dashboard as they complete.

## What's next

- [Onboarding](onboarding.md) — full mental model and realistic workflow walkthrough
- [Playwright Integration](../guides/playwright-integration.md) — advanced Playwright setup
- [Tuning Thresholds](../guides/tuning-thresholds.md) — reduce false positives
- [Configuration Reference](../reference/configuration.md) — all config options
