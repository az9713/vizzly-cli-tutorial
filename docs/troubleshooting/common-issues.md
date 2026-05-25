# Common Issues

Symptom → cause → fix for the most frequent Vizzly problems.

---

## ECONNREFUSED: Server not running

**Symptom:**

```
[vizzly] Server not accessible at http://localhost:47392/screenshot
```

Or tests run silently with no visual results.

**Cause:** The Vizzly TDD server is not running when your tests execute. The client SDK auto-disables itself after the first `ECONNREFUSED` error to avoid spamming.

**Fix:**

Use `vizzly tdd run "npm test"` instead of running the test command directly. This starts the server first:

```bash
# Wrong: server not started
npm test

# Correct: starts server, then runs tests
vizzly tdd run "npm test"
```

Or start the server separately in another terminal:

```bash
# Terminal 1
vizzly tdd start

# Terminal 2
npm test
```

Verify the server is running:

```bash
# Check if .vizzly/server.json exists
cat .vizzly/server.json
# Should show: { "port": 47392, "pid": ..., "startedAt": "..." }
```

---

## All Screenshots Are "New" Every Run

**Symptom:** Every run shows all screenshots as "new" instead of "passed." Exit code is 0 but no baselines are matched.

**Cause 1: Properties vary between runs.**

If `viewport`, `browser`, or other signature properties are set dynamically, they may produce different values each run — and therefore different signatures — so no baseline matches.

**Fix:** Use consistent, hardcoded property values:

```javascript
// Wrong: viewport depends on window size which may vary
let viewport = await page.viewportSize();
await vizzlyScreenshot('page', screenshot, {
  properties: { viewport: `${viewport.width}x${viewport.height}` }
});

// Correct: hardcoded consistent value
await vizzlyScreenshot('page', screenshot, {
  properties: { browser: 'chromium', viewport: '1280x720' }
});
```

**Cause 2: No baselines exist yet.**

On the first run, or after deleting baselines, all screenshots are "new" by definition.

**Fix:** Run with `--set-baseline`:

```bash
vizzly tdd run "npm test" --set-baseline
```

**Cause 3: Wrong working directory.**

Auto-discovery walks up from `process.cwd()`. If tests run in a subdirectory and `.vizzly/baselines/` is in the project root, the server may be saving to a different location.

**Fix:** Check where baselines are being saved:

```bash
find . -name "*.png" -path "*/.vizzly/*"
```

---

## Port Already in Use

**Symptom:**

```
Error: listen EADDRINUSE: address already in use :::47392
```

**Cause:** Another process is already using port 47392 — possibly a leftover Vizzly server from a previous run that didn't exit cleanly.

**Fix:**

Find and kill the process:

```bash
# On macOS/Linux
lsof -ti:47392 | xargs kill -9

# On Windows (PowerShell)
$pid = (Get-NetTCPConnection -LocalPort 47392 -State Listen).OwningProcess
Stop-Process -Id $pid -Force
```

Or use a different port:

```bash
vizzly tdd start --port 9000
```

And update your config:

```javascript
// vizzly.config.js
export default {
  server: { port: 9000 }
};
```

---

## Diff Too Sensitive (False Positives)

**Symptom:** Screenshots fail with small `diffPercentage` (< 0.5%) and ΔE values between 0.5–2.0. Differences look identical visually. Failures appear on CI but not locally, or vice versa.

**Cause:** Font antialiasing or subpixel rendering differences between environments (macOS vs Linux, different GPU, different font hinting configuration).

**Fix:**

Raise the threshold slightly:

```javascript
// vizzly.config.js
export default {
  comparison: {
    threshold: 3.0,    // Was 2.0
    minClusterSize: 20  // Was 10 — ignore more noise
  }
};
```

Or per-screenshot for affected tests:

```javascript
await vizzlyScreenshot('text-heavy-page', screenshot, {
  threshold: 3.0,
  minClusterSize: 20
});
```

For cross-platform issues, set baselines on the same OS as CI:

```bash
# On Linux (matching CI)
docker run -v $(pwd):/app -w /app node:22 sh -c \
  "npm ci && vizzly tdd run 'npm test' --set-baseline"
```

---

## Diff Too Lenient (Missing Regressions)

**Symptom:** A visual change occurred (button color, layout shift) but tests still passed.

**Cause:** Threshold too high for the type of change. A color change of ΔE 3.5 passes if threshold is 5.0.

**Fix:** Check the ΔE of the regression:

```bash
# Run the failing test and check diff details
vizzly tdd run "npm test" --json | jq '.comparisons[] | {name, diffPercentage}'
```

Lower the threshold:

```javascript
export default {
  comparison: { threshold: 1.5 }  // Was 2.0 or higher
};
```

If this causes false positives in noisy screenshots, apply a lower threshold only to the screenshots that need it:

```javascript
// Strict for brand-critical elements
await vizzlyScreenshot('logo', screenshot, { threshold: 0.5 });

// Relaxed for noisy elements
await vizzlyScreenshot('chart', screenshot, { threshold: 5.0 });
```

---

## Node.js Version Error

**Symptom:**

```
SyntaxError: Cannot use import statement in a module
```

Or:

```
Error [ERR_REQUIRE_ESM]: require() of ES Module not supported
```

**Cause:** Running Vizzly with Node.js < 22. The CLI uses modern ESM syntax and native Node.js features that require version 22+.

**Fix:**

```bash
# Check current version
node --version

# Install Node.js 22 via nvm
nvm install 22
nvm use 22

# Verify
node --version  # Should show v22.x.x
```

---

## Baselines Directory Is Empty

**Symptom:**

```bash
ls .vizzly/baselines/
# (empty)
```

Screenshots show as "new" every run. `--set-baseline` doesn't seem to work.

**Cause 1: `--set-baseline` was not used.**

After the first run, you must explicitly accept baselines.

**Fix:**

```bash
vizzly tdd run "npm test" --set-baseline
```

**Cause 2: `vizzlyFlush()` was not called.**

If `vizzlyFlush()` is never called, the server never finalizes the run and doesn't write baselines.

**Fix:** Add flush to your test teardown:

```javascript
// Playwright — in playwright.config.js
export default defineConfig({
  globalTeardown: './vizzly-teardown.js'
});

// vizzly-teardown.js
import { vizzlyFlush } from '@vizzly-testing/cli/client';
export default async () => { await vizzlyFlush(); };
```

```javascript
// Jest/Vitest
afterAll(async () => {
  await vizzlyFlush();
});
```

**Cause 3: Wrong `.vizzly/` directory.**

If `VIZZLY_HOME` is set, baselines are stored there instead of `.vizzly/` in the project root.

**Fix:**

```bash
echo $VIZZLY_HOME  # Check if set
ls $VIZZLY_HOME/baselines/  # Look there
```

---

## Dashboard Not Updating

**Symptom:** The dashboard at `http://localhost:47392` shows old results or doesn't update when tests run.

**Cause:** SSE connection may have dropped. Browser may have cached the old state.

**Fix:**

1. Refresh the browser tab
2. Check the server is still running: `cat .vizzly/server.json`
3. If server crashed, restart: `vizzly tdd start --open`

---

## Tests Hang After Completion

**Symptom:** Tests complete but the process doesn't exit. Hangs indefinitely.

**Cause:** The client SDK keeps the process alive because it has an open socket (HTTP with keep-alive). Vizzly uses `Connection: close` headers to prevent this, but some environments don't honor it.

**Fix:** Explicitly call `vizzlyFlush()` in teardown — this closes the connection cleanly. If you're using `vizzly tdd run`, the server controls the process lifecycle. If running the server manually and the process hangs, force-kill:

```bash
# Kill any remaining Vizzly server
lsof -ti:47392 | xargs kill -9
```

---

## Screenshots Are All Black or Blank

**Symptom:** Screenshot files exist but are all black, all white, or blank.

**Cause:** This is a screenshot capture issue in your test framework, not a Vizzly issue. The image data sent to Vizzly is already blank.

**Fix:** Debug the screenshot capture directly:

```javascript
// Write screenshot to disk and inspect it
import { writeFileSync } from 'fs';
let screenshot = await page.screenshot();
writeFileSync('/tmp/debug-screenshot.png', screenshot);
// Now open /tmp/debug-screenshot.png to see what was captured
```

Common causes: page not loaded, element not visible, headless browser starting before page renders. Add a `waitForLoadState('networkidle')` before capturing.

See also: [Tuning Thresholds](../guides/tuning-thresholds.md) | [TDD Server](../concepts/tdd-server.md) | [Client SDK](../concepts/client-sdk.md)
