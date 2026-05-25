# Integrating Vizzly with Playwright

Add visual regression testing to your Playwright test suite. This guide covers the complete setup from installation through CI.

## Goal

By the end of this guide, your Playwright tests will:
- Capture screenshots and send them to Vizzly
- Show visual diffs in the live dashboard during development
- Generate a static HTML report after each test run
- Fail CI if unexpected visual regressions are detected

## Prerequisites

- Node.js 22+
- Vizzly CLI installed: `npm install -g @vizzly-testing/cli`
- Playwright installed in your project

## Step 1: Install the Vizzly client in your project

```bash
npm install --save-dev @vizzly-testing/cli
```

Note: The CLI is installed globally for the `vizzly` command. The `@vizzly-testing/cli` package is also installed locally for the client SDK import.

## Step 2: Initialize Vizzly

```bash
vizzly init
```

This creates `vizzly.config.js`. Edit it to match your project:

```javascript
// vizzly.config.js
export default {
  server: {
    port: 47392,      // default — change if port is in use
    timeout: 60000    // increase for slow test suites
  },
  comparison: {
    threshold: 2.0,   // CIEDE2000 Delta E — lower = stricter
    minClusterSize: 10
  },
  tdd: {
    openReport: false // set to true to auto-open report after run
  }
};
```

## Step 3: Add Vizzly to your Playwright config

Edit `playwright.config.js` to add global teardown:

```javascript
// playwright.config.js
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './tests',
  use: {
    baseURL: 'http://localhost:3000',
    // Capture screenshots on failure (optional — separate from Vizzly)
    screenshot: 'only-on-failure',
  },
  // Global teardown ensures vizzlyFlush() is always called
  globalTeardown: './vizzly-teardown.js'
});
```

Create `vizzly-teardown.js`:

```javascript
// vizzly-teardown.js
import { vizzlyFlush } from '@vizzly-testing/cli/client';

export default async function globalTeardown() {
  await vizzlyFlush();
}
```

Using a global teardown ensures the flush runs even if individual tests fail.

## Step 4: Write tests with screenshots

```javascript
// tests/homepage.spec.js
import { test, expect } from '@playwright/test';
import { vizzlyScreenshot } from '@vizzly-testing/cli/client';

test('homepage', async ({ page, browserName }) => {
  await page.goto('/');
  await page.waitForLoadState('networkidle');

  let viewport = page.viewportSize();
  let viewportStr = `${viewport.width}x${viewport.height}`;

  // Full-page screenshot
  let screenshot = await page.screenshot({ fullPage: true });
  await vizzlyScreenshot('homepage-full', screenshot, {
    properties: {
      browser: browserName,
      viewport: viewportStr
    },
    fullPage: true
  });

  // Element-level screenshot
  let heroShot = await page.locator('.hero-section').screenshot();
  await vizzlyScreenshot('homepage-hero', heroShot, {
    properties: {
      browser: browserName,
      viewport: viewportStr
    }
  });
});

test('checkout flow', async ({ page, browserName }) => {
  await page.goto('/checkout');
  await page.waitForLoadState('networkidle');

  let viewport = page.viewportSize();

  // Step 1: Cart
  await vizzlyScreenshot('checkout-cart', await page.screenshot(), {
    properties: { browser: browserName, viewport: `${viewport.width}x${viewport.height}` }
  });

  // Fill form
  await page.fill('[data-testid="email"]', 'test@example.com');
  await page.fill('[data-testid="card-number"]', '4242424242424242');
  await page.click('[data-testid="next-step"]');
  await page.waitForLoadState('networkidle');

  // Step 2: Payment
  await vizzlyScreenshot('checkout-payment', await page.screenshot(), {
    properties: { browser: browserName, viewport: `${viewport.width}x${viewport.height}` }
  });

  await expect(page.locator('[data-testid="submit-order"]')).toBeVisible();
});
```

## Step 5: Run tests with Vizzly

**Development (live dashboard):**

```bash
# Terminal 1
vizzly tdd start --open

# Terminal 2
npx playwright test
```

**One-shot (report only):**

```bash
vizzly tdd run "npx playwright test"
```

**First run — accept baselines:**

```bash
vizzly tdd run "npx playwright test" --set-baseline
```

## Step 6: Multi-browser setup

Playwright can run the same test in multiple browsers. Each browser gets a separate baseline via the `browser` property:

```javascript
// playwright.config.js
export default defineConfig({
  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
    {
      name: 'firefox',
      use: { ...devices['Desktop Firefox'] },
    },
    {
      name: 'webkit',
      use: { ...devices['Desktop Safari'] },
    },
  ],
});
```

The `browserName` fixture in tests gives you the project name (`chromium`, `firefox`, `webkit`). Passing it as a property ensures each browser has its own baseline:

```javascript
test('page', async ({ page, browserName }) => {
  let screenshot = await page.screenshot();
  await vizzlyScreenshot('page', screenshot, {
    properties: { browser: browserName }
  });
});
```

## Verification

After running with `--set-baseline`, verify baselines were created:

```bash
ls .vizzly/baselines/
# homepage-full-a3f8c2d1.png
# homepage-full-b7e4f912.png     ← different browser
# homepage-hero-c1d5e8f2.png
# checkout-cart-d4e7f011.png
# checkout-payment-e5f8a122.png
```

Then run without `--set-baseline` and verify everything passes:

```bash
vizzly tdd run "npx playwright test"
# Summary: 0 new, 5 passed, 0 failed ✓
```

## Troubleshooting

**All screenshots are "new" every run**

This usually means properties are inconsistent between runs. For example, if viewport size varies, the signature changes. Lock it down:

```javascript
// Before taking screenshots, lock the viewport
await page.setViewportSize({ width: 1280, height: 720 });
let screenshot = await page.screenshot();
await vizzlyScreenshot('page', screenshot, {
  properties: { browser: browserName, viewport: '1280x720' }  // hardcoded, not dynamic
});
```

**ECONNREFUSED errors in logs**

The server isn't running. Make sure to use `vizzly tdd run "..."` (which starts the server) rather than running Playwright directly. Or start the server separately with `vizzly tdd start`.

**Tests are slow after adding Vizzly**

Each `vizzlyScreenshot()` call makes an HTTP request. For large images, base64 encoding adds overhead. Use file paths instead:

```javascript
let path = `./tmp/screenshot-${Date.now()}.png`;
await page.screenshot({ path });
await vizzlyScreenshot('page', path);  // more efficient for large screenshots
```

**Dashboard shows stale results**

The dashboard uses SSE and stays connected. If you see old results, refresh the browser page.

See also: [Client SDK](../concepts/client-sdk.md) | [Tuning Thresholds](tuning-thresholds.md) | [CI/CD Integration](ci-cd-integration.md)
