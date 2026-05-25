# Integrating Vizzly with Jest

Add visual regression testing to your Jest (or Vitest) test suite. This guide covers component-level and integration-level screenshot testing.

## Goal

By the end of this guide, your Jest tests will capture screenshots and send them to Vizzly for visual comparison, with results shown in the dashboard and summary.

## Prerequisites

- Node.js 22+
- Vizzly CLI installed: `npm install -g @vizzly-testing/cli`
- Jest 29+ or Vitest 1+ installed in your project

## Step 1: Install dependencies

```bash
npm install --save-dev @vizzly-testing/cli
```

For component testing with jsdom, you'll also need a way to capture screenshots. Options:
- **Puppeteer** — headless Chrome, works well with component test servers
- **Playwright** — can be used as a library without the test runner
- **`html-to-image`** — DOM-to-PNG in jsdom (limited fidelity)

This guide uses Puppeteer for browser-based component screenshots.

```bash
npm install --save-dev puppeteer
```

## Step 2: Initialize Vizzly

```bash
vizzly init
```

## Step 3: Create a global setup file

Jest runs `globalSetup` and `globalTeardown` outside the test environment, so `vizzlyFlush()` must be called from `globalTeardown`.

**`jest.config.js`:**

```javascript
export default {
  testEnvironment: 'node',
  globalSetup: './jest-global-setup.js',
  globalTeardown: './jest-global-teardown.js',
  transform: {}  // use native ESM
};
```

**`jest-global-setup.js`:**

```javascript
// Start a local server for your app if needed
export default async function globalSetup() {
  // e.g., start your dev server here
}
```

**`jest-global-teardown.js`:**

```javascript
import { vizzlyFlush } from '@vizzly-testing/cli/client';

export default async function globalTeardown() {
  await vizzlyFlush();
}
```

## Step 4: Write tests with screenshots

### Pattern 1: Full-page screenshots with Puppeteer

```javascript
// tests/pages.test.js
import puppeteer from 'puppeteer';
import { vizzlyScreenshot } from '@vizzly-testing/cli/client';

let browser;
let page;

beforeAll(async () => {
  browser = await puppeteer.launch();
  page = await browser.newPage();
  await page.setViewport({ width: 1280, height: 720 });
});

afterAll(async () => {
  await browser.close();
});

test('homepage looks correct', async () => {
  await page.goto('http://localhost:3000/');
  await page.waitForNetworkIdle();

  let screenshot = await page.screenshot({ type: 'png' });

  await vizzlyScreenshot('homepage', screenshot, {
    properties: {
      browser: 'chromium',
      viewport: '1280x720'
    }
  });
});

test('pricing page looks correct', async () => {
  await page.goto('http://localhost:3000/pricing');
  await page.waitForNetworkIdle();

  let screenshot = await page.screenshot({ fullPage: true, type: 'png' });

  await vizzlyScreenshot('pricing-full', screenshot, {
    properties: { browser: 'chromium', viewport: '1280x720' },
    fullPage: true
  });
});
```

### Pattern 2: Component-level screenshots

```javascript
// tests/components/button.test.js
import puppeteer from 'puppeteer';
import { vizzlyScreenshot } from '@vizzly-testing/cli/client';

let browser;

beforeAll(async () => {
  browser = await puppeteer.launch();
});

afterAll(async () => {
  await browser.close();
});

async function renderComponent(html) {
  let page = await browser.newPage();
  await page.setViewport({ width: 400, height: 200 });
  await page.setContent(`
    <!DOCTYPE html>
    <html>
    <head>
      <link rel="stylesheet" href="http://localhost:3000/styles.css">
    </head>
    <body style="padding: 20px; background: white;">
      ${html}
    </body>
    </html>
  `);
  await page.waitForNetworkIdle();
  return page;
}

test('primary button', async () => {
  let page = await renderComponent('<button class="btn btn-primary">Click me</button>');
  let screenshot = await page.screenshot({ type: 'png' });
  await vizzlyScreenshot('button-primary', screenshot);
  await page.close();
});

test('primary button - hover state', async () => {
  let page = await renderComponent('<button class="btn btn-primary">Click me</button>');
  await page.hover('.btn-primary');
  let screenshot = await page.screenshot({ type: 'png' });
  await vizzlyScreenshot('button-primary-hover', screenshot);
  await page.close();
});

test('button variants', async () => {
  let page = await renderComponent(`
    <div style="display: flex; gap: 10px;">
      <button class="btn btn-primary">Primary</button>
      <button class="btn btn-secondary">Secondary</button>
      <button class="btn btn-danger">Danger</button>
      <button class="btn" disabled>Disabled</button>
    </div>
  `);
  let screenshot = await page.screenshot({ type: 'png' });
  await vizzlyScreenshot('button-all-variants', screenshot);
  await page.close();
});
```

### Pattern 3: Vitest (identical to Jest)

Vitest uses the same Jest-compatible API. The only difference is configuration:

**`vitest.config.js`:**

```javascript
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globalSetup: './jest-global-setup.js',
    globalTeardown: './jest-global-teardown.js',
  }
});
```

Your test files are identical — use the same `import { vizzlyScreenshot }` and `beforeAll`/`afterAll` patterns.

## Step 5: Run tests

**Development:**

```bash
# Terminal 1
vizzly tdd start --open

# Terminal 2
npx jest
# or: npx vitest run
```

**One-shot:**

```bash
vizzly tdd run "npx jest"
# or: vizzly tdd run "npx vitest run"
```

**First run — set baselines:**

```bash
vizzly tdd run "npx jest" --set-baseline
```

**Watch mode:**

```bash
vizzly tdd start  # Keep server running

# In another terminal, watch mode
npx jest --watch
```

## Verification

```bash
# Verify baselines were created
ls .vizzly/baselines/

# Run and verify everything passes
vizzly tdd run "npx jest"
# Expected: 0 new, N passed, 0 failed
```

## Troubleshooting

**`vizzlyFlush` isn't being called**

If using `afterAll` inside individual test files, `afterAll` runs per-file. If tests run in parallel across workers, the flush from one file may not signal "all tests done." Use `globalTeardown` in Jest config for a single flush at the end of the entire suite.

**Screenshots look different between runs**

Puppeteer rendering can vary between runs if animated elements are mid-animation. Disable animations in your test environment:

```javascript
await page.addStyleTag({
  content: `
    *, *::before, *::after {
      animation-duration: 0s !important;
      transition-duration: 0s !important;
    }
  `
});
```

**Node.js version warnings**

Ensure you're running Node.js 22+:

```bash
node --version  # must be >= 22
```

See also: [Client SDK](../concepts/client-sdk.md) | [Setting Baselines](setting-baselines.md) | [Tuning Thresholds](tuning-thresholds.md)
