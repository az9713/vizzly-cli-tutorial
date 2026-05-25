# TDD Server

The TDD server is the heart of Vizzly's local workflow. It's a Node.js HTTP server that runs on your machine, accepts screenshots from your test suite, performs comparisons, and serves the live dashboard.

## How It Starts

Two commands start the TDD server:

**Daemon mode** — `vizzly tdd start`

The server starts and runs persistently. You run your tests separately. The server accumulates results across multiple runs until you stop it.

```bash
vizzly tdd start
# Server is now running in the foreground on :47392
# Run tests in another terminal
```

**One-shot mode** — `vizzly tdd run "command"`

The server starts, runs your test command as a subprocess, waits for all results (via the `/flush` endpoint), generates a static HTML report, and exits.

```bash
vizzly tdd run "npx playwright test"
# Server starts → tests run → report generated → server stops
```

## Server Descriptor File

When the server starts, it writes `.vizzly/server.json`:

```json
{
  "port": 47392,
  "pid": 12345,
  "startedAt": "2024-01-15T10:30:00.000Z"
}
```

This file is the **discovery beacon**. The client SDK walks up the directory tree looking for this file. When found, it reads the port and connects. No environment variables or configuration needed.

The file is deleted when the server stops.

## HTTP Endpoints

### `POST /screenshot`

The primary endpoint. Accepts a screenshot for comparison.

**Request body:**

```json
{
  "buildId": "optional-build-id",
  "name": "homepage",
  "image": "<base64-encoded-png or file path>",
  "type": "base64 | file-path",
  "properties": {
    "browser": "chromium",
    "viewport": "1280x720",
    "threshold": 2.0,
    "minClusterSize": 10,
    "fullPage": false
  }
}
```

**Response — new screenshot (no baseline):**

```json
{
  "status": "new",
  "name": "homepage",
  "signature": "a3f8c2d1...",
  "message": "No baseline found. Screenshot saved as current."
}
```

**Response — passed:**

```json
{
  "status": "passed",
  "name": "homepage",
  "diffPercentage": 0.12,
  "signature": "a3f8c2d1..."
}
```

**Response — failed (HTTP 422):**

```json
{
  "tddMode": true,
  "comparison": {
    "name": "homepage",
    "status": "failed",
    "diffPercentage": 8.43,
    "threshold": 2.0,
    "signature": "a3f8c2d1...",
    "baselinePath": ".vizzly/baselines/homepage-a3f8c2d1.png",
    "currentPath": ".vizzly/current/homepage-a3f8c2d1.png",
    "diffPath": ".vizzly/diff/homepage-a3f8c2d1.png"
  }
}
```

The client SDK returns the comparison result as `{ success: true, status: 'failed', name, diffPercentage }` for 422 responses with `tddMode: true` — this allows your tests to continue capturing remaining screenshots rather than aborting on the first failure.

### `POST /flush`

Signals that all screenshots for the current test run have been submitted. Call this in your test teardown via `vizzlyFlush()`.

**Request body:** `{}` (empty)

**Response:**

```json
{
  "comparisons": [...],
  "summary": {
    "total": 5,
    "passed": 3,
    "failed": 1,
    "new": 1
  }
}
```

After flush, the server:
1. Finalizes all pending comparisons
2. Prints the summary table to stdout
3. In one-shot mode: generates the HTML report and exits
4. In daemon mode: resets state for the next run

### Dashboard (SSE)

The dashboard at `http://localhost:47392` is served as a static web app. It receives real-time updates via Server-Sent Events (SSE). As each screenshot comparison completes on the server, the result is pushed to all connected dashboard clients immediately — you see diffs appear as tests run.

## Port Configuration

Default port is `47392`. Change it in `vizzly.config.js`:

```javascript
export default {
  server: {
    port: 9000
  }
};
```

Or via CLI flag:

```bash
vizzly tdd start --port 9000
```

If the port is already in use, the server fails to start with an `EADDRINUSE` error. See [Common Issues](../troubleshooting/common-issues.md#port-conflict) for how to handle this.

## Timeout Configuration

The server waits up to `server.timeout` milliseconds after the last screenshot before it considers a run "stuck" (default: 30000ms = 30 seconds). This prevents the server from hanging indefinitely if `vizzlyFlush()` is never called.

```javascript
export default {
  server: {
    timeout: 60000  // 60 seconds for slow test suites
  }
};
```

## Lifecycle in Daemon Mode

```
vizzly tdd start
    │
    ▼
Server starts, writes .vizzly/server.json
    │
    ▼
[waiting for connections]
    │
    ├──── Test run 1: POST /screenshot × N, POST /flush
    │         → summary printed, state reset
    │
    ├──── Test run 2: POST /screenshot × M, POST /flush
    │         → summary printed, state reset
    │
    └──── Ctrl-C → graceful shutdown, .vizzly/server.json deleted
```

## Lifecycle in One-Shot Mode

```
vizzly tdd run "npm test"
    │
    ▼
Server starts, writes .vizzly/server.json
    │
    ▼
Test command spawned as subprocess
    │
    ▼
Screenshots arrive via POST /screenshot
    │
    ▼
POST /flush received
    │
    ▼
Summary printed, HTML report generated
    │
    ▼
Server exits, .vizzly/server.json deleted
```

See also: [Client SDK](client-sdk.md) | [Baseline System](baseline-system.md) | [Common Issues](../troubleshooting/common-issues.md)
