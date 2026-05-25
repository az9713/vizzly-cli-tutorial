# Client API Reference

Complete reference for all exports from `@vizzly-testing/cli/client`.

## Import

```javascript
import {
  vizzlyScreenshot,
  vizzlyFlush,
  isVizzlyReady,
  configure,
  setEnabled,
  getVizzlyInfo,
  autoDiscoverTddServer,
  shouldLogClient,
  LOG_LEVELS
} from '@vizzly-testing/cli/client';
```

---

## `vizzlyScreenshot(name, image, options?)`

Take a screenshot and send it to the Vizzly TDD server for visual comparison.

### Parameters

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `name` | `string` | Yes | Unique name for this screenshot. Used as the comparison key and in the baseline filename. Must be consistent across runs for the same UI element. |
| `image` | `Buffer \| string` | Yes | The screenshot data. Pass a Node.js `Buffer` containing PNG data, or a `string` file path to a PNG file on disk. |
| `options` | `object` | No | Configuration options (see below) |

### Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `options.properties` | `Record<string, any>` | `{}` | Key-value metadata attached to this screenshot. Properties listed in `signatureProperties` (default: `browser`, `viewport`) determine which baseline is used. |
| `options.threshold` | `number` | Global config value | Per-screenshot CIEDE2000 Delta E threshold override. Takes precedence over global `comparison.threshold`. |
| `options.minClusterSize` | `number` | Global config value | Per-screenshot minimum cluster size override. Takes precedence over global `comparison.minClusterSize`. |
| `options.fullPage` | `boolean` | `false` | Metadata flag indicating this is a full-page screenshot. Does not affect comparison logic. |

### Return Value

`Promise<ScreenshotResult | null>`

| Value | Meaning |
|-------|---------|
| `{ success: true, status: 'passed', name, diffPercentage }` | Comparison passed — diff below threshold |
| `{ success: true, status: 'failed', name, diffPercentage }` | Comparison failed — diff exceeds threshold (returned as `success: true` so tests continue) |
| `{ success: true, status: 'new', name }` | No baseline exists for this signature |
| `null` | Vizzly is disabled or server is not running — not an error |

### Examples

```javascript
// Basic usage
let screenshot = await page.screenshot();
let result = await vizzlyScreenshot('homepage', screenshot);

// With browser and viewport properties
let result = await vizzlyScreenshot('checkout', screenshot, {
  properties: {
    browser: 'chromium',
    viewport: '1440x900'
  }
});

// With per-screenshot threshold
let result = await vizzlyScreenshot('data-chart', screenshot, {
  threshold: 8.0,
  minClusterSize: 100
});

// With file path (more efficient for large screenshots)
await page.screenshot({ path: '/tmp/screenshot.png' });
let result = await vizzlyScreenshot('homepage', '/tmp/screenshot.png');

// Full-page with all options
let result = await vizzlyScreenshot('product-page', screenshot, {
  properties: {
    browser: 'chromium',
    viewport: '1920x1080',
    theme: 'dark',
    locale: 'en-US'
  },
  threshold: 2.5,
  minClusterSize: 15,
  fullPage: true
});

// Check the result
if (result?.status === 'failed') {
  console.log(`Visual diff: ${result.diffPercentage}%`);
}
```

---

## `vizzlyFlush()`

Signal to the TDD server that all screenshots for the current test run have been submitted. Triggers finalization: summary printing, report generation, state reset.

### Parameters

None.

### Return Value

`Promise<FlushResult | null>`

| Value | Meaning |
|-------|---------|
| `{ comparisons, summary }` | Server acknowledged flush, returns final summary |
| `null` | No server connected, or flush failed silently |

### When to Call

Call once per test suite run, in global teardown:

```javascript
// Playwright (playwright.config.js)
export default defineConfig({
  globalTeardown: './vizzly-teardown.js'
});

// vizzly-teardown.js
import { vizzlyFlush } from '@vizzly-testing/cli/client';
export default async () => { await vizzlyFlush(); };
```

```javascript
// Jest / Vitest
afterAll(async () => {
  await vizzlyFlush();
});
```

```javascript
// Node test runner
after(async () => {
  await vizzlyFlush();
});
```

---

## `isVizzlyReady()`

Check whether the client is connected to a TDD server and ready to capture screenshots.

### Parameters

None.

### Return Value

`boolean` — `true` if connected and enabled, `false` otherwise.

### Example

```javascript
if (isVizzlyReady()) {
  let screenshot = await page.screenshot();
  await vizzlyScreenshot('page', screenshot);
} else {
  console.log('Vizzly not running — skipping visual test');
}
```

---

## `configure(config)`

Explicitly configure the client. Use when auto-discovery won't work, or to disable Vizzly for specific test files.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `config` | `object` | Configuration to apply |
| `config.serverUrl` | `string \| null` | Override the TDD server URL. Pass `null` to clear. |
| `config.enabled` | `boolean` | Enable or disable screenshot capture. |

### Examples

```javascript
// Set explicit server URL (bypasses auto-discovery)
configure({ serverUrl: 'http://localhost:9000' });

// Disable for this test file
configure({ enabled: false });

// Re-enable
configure({ enabled: true });
```

---

## `setEnabled(enabled)`

Shorthand for `configure({ enabled })`.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `enabled` | `boolean` | Whether to enable screenshot capture |

### Example

```javascript
setEnabled(false);  // Disable
setEnabled(true);   // Re-enable
```

---

## `getVizzlyInfo()`

Get the current state of the Vizzly client. Useful for debugging.

### Parameters

None.

### Return Value

```typescript
{
  enabled: boolean;      // Whether Vizzly is enabled
  serverUrl: string | null;  // Configured or auto-discovered server URL
  ready: boolean;        // Whether client is connected and ready
  buildId: string | null;    // Current build ID from VIZZLY_BUILD_ID env var
  tddMode: boolean;      // Whether VIZZLY_TDD_MODE is set
  disabled: boolean;     // Whether auto-disabled due to connection failure
}
```

### Example

```javascript
let info = getVizzlyInfo();
console.log(info);
// {
//   enabled: true,
//   serverUrl: 'http://localhost:47392',
//   ready: true,
//   buildId: null,
//   tddMode: false,
//   disabled: false
// }
```

---

## `autoDiscoverTddServer(startDir?, deps?)`

Manually trigger TDD server auto-discovery. Useful for debugging or when you need to know the discovered URL without taking a screenshot.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `startDir` | `string` | Directory to start searching from (default: `process.cwd()`) |
| `deps` | `object` | Injectable dependencies for testing |

### Return Value

`string | null` — The discovered server URL (e.g., `'http://localhost:47392'`), or `null` if not found.

### Example

```javascript
let serverUrl = autoDiscoverTddServer();
console.log(serverUrl);
// 'http://localhost:47392' or null
```

---

## `shouldLogClient(level, configuredLevel?)`

Check if the client should log at the given level, based on `VIZZLY_CLIENT_LOG_LEVEL` env var.

### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `level` | `'debug' \| 'info' \| 'warn' \| 'error'` | Log level to check |
| `configuredLevel` | `string` | Override (defaults to env var) |

### Return Value

`boolean`

---

## `LOG_LEVELS`

Object mapping log level names to numeric values.

```javascript
{
  debug: 0,
  info: 1,
  warn: 2,
  error: 3
}
```

---

## Auto-Disable Behavior

The client disables itself automatically after the first HTTP-level failure (not a visual diff failure). This prevents error spam when the server isn't running.

After auto-disable:
- `vizzlyScreenshot()` returns `null` immediately (no network request)
- `isVizzlyReady()` returns `false`
- `getVizzlyInfo().disabled` is `true`

To re-enable after auto-disable:

```javascript
configure({ enabled: true });
```

---

## Request Timeout

Default request timeout is 30 seconds. If a screenshot request times out, the client auto-disables.

There is no API to configure this timeout per-request. If you need a longer timeout (e.g., server is under load), it must be configured on the server side via `server.timeout` in `vizzly.config.js`.

See also: [Client SDK Concepts](../concepts/client-sdk.md) | [Configuration Reference](configuration.md)
