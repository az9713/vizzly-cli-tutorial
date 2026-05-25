# Client SDK

The client SDK (`@vizzly-testing/cli/client`) is the interface your test code uses to send screenshots to the TDD server. It's intentionally thin — it's an HTTP client, not a comparison engine. The comparison happens on the server.

## Import

```javascript
import {
  vizzlyScreenshot,
  vizzlyFlush,
  isVizzlyReady,
  configure,
  getVizzlyInfo,
  setEnabled
} from '@vizzly-testing/cli/client';
```

## Auto-Discovery

When `vizzlyScreenshot()` is first called, the SDK needs to find the TDD server. If no server URL has been configured explicitly, it auto-discovers it:

1. Start at `process.cwd()`
2. Look for `.vizzly/server.json`
3. If not found, go up one directory
4. Repeat until the filesystem root
5. If found, read the `port` field and construct `http://localhost:<port>`
6. If not found anywhere, return `null` — Vizzly silently disables itself

This means you can call `vizzlyScreenshot()` in your tests without any configuration. As long as the TDD server is running and `.vizzly/server.json` exists somewhere in the directory tree above your test files, the SDK will find it.

```javascript
// No configuration needed — auto-discovery handles it
await vizzlyScreenshot('homepage', screenshot);
```

## `vizzlyScreenshot(name, image, options?)`

Takes a screenshot and sends it to the TDD server for comparison.

**Parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `name` | `string` | Unique name for this screenshot. Used as the comparison key and in the baseline filename. |
| `image` | `Buffer \| string` | PNG image data as a Node.js Buffer, OR an absolute file path to a PNG file. |
| `options` | `object` | Optional configuration (see below). |

**Options:**

| Option | Type | Description |
|--------|------|-------------|
| `properties` | `Record<string, any>` | Metadata to attach to the screenshot. Properties in `signatureProperties` (e.g., `browser`, `viewport`) affect which baseline is used. |
| `threshold` | `number` | Per-screenshot CIEDE2000 Delta E threshold override. |
| `minClusterSize` | `number` | Per-screenshot minimum cluster size override. |
| `fullPage` | `boolean` | Whether this is a full-page screenshot (metadata only, doesn't affect comparison). |

**Returns:** `Promise<object | null>`

| Return value | Meaning |
|-------------|---------|
| `{ success: true, status: 'passed', name, diffPercentage }` | Comparison passed |
| `{ success: true, status: 'failed', name, diffPercentage }` | Comparison failed (returns success: true so your test can continue) |
| `{ success: true, status: 'new', name, ... }` | No baseline exists |
| `null` | Server not available, or SDK is disabled |

**Important:** A return value of `null` means Vizzly is not running — it does NOT mean the screenshot failed. The SDK is designed to be transparent when Vizzly isn't present.

**Examples:**

```javascript
// Basic usage with Playwright Buffer
let screenshot = await page.screenshot();
await vizzlyScreenshot('homepage', screenshot);

// With properties (browser and viewport affect which baseline is used)
await vizzlyScreenshot('checkout', screenshot, {
  properties: {
    browser: 'chromium',
    viewport: '1440x900'
  }
});

// With a file path (for tools that write screenshots to disk)
await page.screenshot({ path: './tmp/homepage.png' });
await vizzlyScreenshot('homepage', './tmp/homepage.png');

// With threshold override
await vizzlyScreenshot('data-chart', screenshot, {
  threshold: 5.0,
  minClusterSize: 50
});

// With all options
await vizzlyScreenshot('product-hero', screenshot, {
  properties: {
    browser: 'chromium',
    viewport: '1920x1080',
    theme: 'dark'
  },
  threshold: 1.5,
  minClusterSize: 20,
  fullPage: false
});
```

## `vizzlyFlush()`

Signals to the TDD server that all screenshots for the current test run have been submitted. Call this in your test framework's global teardown.

**Returns:** `Promise<object | null>` — the flush summary, or `null` if no server is connected.

Without a flush, the server doesn't know when tests are done and won't finalize the run (print summary, generate report, reset state for next run).

**Playwright:**

```javascript
test.afterAll(async () => {
  await vizzlyFlush();
});
```

**Jest/Vitest:**

```javascript
afterAll(async () => {
  await vizzlyFlush();
});
```

**Node test runner:**

```javascript
after(async () => {
  await vizzlyFlush();
});
```

## `isVizzlyReady()`

Returns `true` if the SDK is connected to a TDD server and ready to capture screenshots.

```javascript
if (isVizzlyReady()) {
  let screenshot = await page.screenshot();
  await vizzlyScreenshot('homepage', screenshot);
}
```

Use this when you want to conditionally add screenshot capture without failing when Vizzly isn't running.

## `configure(config)`

Explicitly configure the SDK. Useful when auto-discovery won't work (e.g., tests run in a subprocess with a different working directory).

```javascript
configure({
  serverUrl: 'http://localhost:47392',  // Override auto-discovery
  enabled: true  // Explicitly enable
});

// Disable for this test file
configure({ enabled: false });
```

## `getVizzlyInfo()`

Returns the current state of the SDK. Useful for debugging.

```javascript
let info = getVizzlyInfo();
console.log(info);
// {
//   enabled: true,
//   serverUrl: 'http://localhost:47392',
//   ready: true,
//   buildId: null,
//   tddMode: true,
//   disabled: false
// }
```

## `setEnabled(enabled)`

Enable or disable screenshot capture.

```javascript
setEnabled(false);  // Stop capturing screenshots
setEnabled(true);   // Resume
```

## Auto-Disable Behavior

The SDK automatically disables itself after the **first failure** — meaning an HTTP error (not a visual diff failure). This prevents error spam when the server is not running.

Scenarios that trigger auto-disable:
- `ECONNREFUSED` — server not running at the expected port
- Request timeout (30 seconds by default)
- Unexpected HTTP errors

What does NOT trigger auto-disable:
- Visual diffs (HTTP 422 with `tddMode: true`) — these are normal comparison results
- HTTP 422 from a legitimately failed comparison

After auto-disable, all subsequent `vizzlyScreenshot()` calls return `null` immediately without network requests.

## Log Level Control

The SDK respects `VIZZLY_CLIENT_LOG_LEVEL` (default: `'error'`). Set to `'debug'` to see all HTTP timing:

```bash
VIZZLY_CLIENT_LOG_LEVEL=debug npx playwright test
# [vizzly-client] homepage HTTP completed in 12ms
# [vizzly-client] checkout HTTP completed in 8ms
```

| Level | What you see |
|-------|-------------|
| `debug` | HTTP timings, request details |
| `info` | Connection events |
| `warn` | Non-fatal issues |
| `error` (default) | Connection failures only |

## Buffer vs File Path

The SDK accepts two image formats:

**Buffer** (in-memory PNG data):
```javascript
let screenshot = await page.screenshot(); // returns Buffer
await vizzlyScreenshot('page', screenshot);
```

**File path** (absolute or relative path to PNG):
```javascript
await page.screenshot({ path: '/tmp/page.png' });
await vizzlyScreenshot('page', '/tmp/page.png');
```

When a file path is provided, it's sent to the server as-is. The server reads the file. This is more efficient for large screenshots since the data isn't base64-encoded for the HTTP request.

See also: [TDD Server](tdd-server.md) | [Client API Reference](../reference/client-api.md) | [Playwright Integration](../guides/playwright-integration.md)
