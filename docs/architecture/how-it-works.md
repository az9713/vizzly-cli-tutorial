# How Vizzly Works: System Design

A technical walkthrough of Vizzly's architecture — component relationships, data flow, and the reasoning behind key design decisions.

## Component Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│  Your Test Suite                                                     │
│                                                                     │
│  vizzlyScreenshot('name', buffer, options)                          │
│       │                                                             │
│       ▼                                                             │
│  Client SDK (@vizzly-testing/cli/client)                           │
│       │                                                             │
│       │  1. Auto-discover: walk dirs → find .vizzly/server.json   │
│       │  2. HTTP POST /screenshot (base64 or file path)            │
│       │  3. HTTP POST /flush (after all tests)                     │
└───────┼─────────────────────────────────────────────────────────────┘
        │
        │ HTTP (localhost)
        │
┌───────▼─────────────────────────────────────────────────────────────┐
│  TDD Server (Node.js HTTP on :47392)                                │
│                                                                     │
│  POST /screenshot                                                   │
│       │                                                             │
│       ├── Generate signature (name + properties → SHA hash)        │
│       ├── Save to .vizzly/current/<name>-<hash>.png               │
│       ├── Look up .vizzly/baselines/<name>-<hash>.png             │
│       │     ├── Not found → status: 'new'                          │
│       │     └── Found → run honeydiff                              │
│       │           ├── ΔE < threshold → status: 'passed'           │
│       │           └── ΔE ≥ threshold → status: 'failed'           │
│       │                  ├── Save diff to .vizzly/diff/            │
│       │                  └── Return HTTP 422 with comparison data  │
│       │                                                             │
│  POST /flush                                                        │
│       ├── Print summary table                                       │
│       ├── Generate .vizzly/report/index.html                       │
│       └── (one-shot mode) Exit                                      │
│                                                                     │
│  GET / (dashboard)                                                  │
│       └── SSE push comparison results as they arrive              │
└─────────────────────────────────────────────────────────────────────┘
        │
        ├── .vizzly/server.json      (discovery beacon)
        ├── .vizzly/baselines/       (approved truth)
        ├── .vizzly/current/         (this run)
        ├── .vizzly/diff/            (failed comparisons)
        └── .vizzly/report/          (HTML report)
```

## Data Flow: A Single Screenshot

Let's trace exactly what happens when your test calls `vizzlyScreenshot('checkout', screenshotBuffer, { properties: { browser: 'chromium', viewport: '1280x720' } })`.

**Step 1: Client SDK — auto-discovery**

The SDK checks if it already has a server URL. If not, it walks up the directory tree from `process.cwd()` looking for `.vizzly/server.json`. This file contains `{ "port": 47392 }`. The SDK constructs `http://localhost:47392`.

**Step 2: Client SDK — HTTP POST**

The SDK serializes the Buffer to base64 (or passes the file path directly) and POSTs to `http://localhost:47392/screenshot`:

```json
{
  "buildId": null,
  "name": "checkout",
  "image": "iVBORw0KGgoAAAANSUhEUgAA...",
  "type": "base64",
  "properties": {
    "browser": "chromium",
    "viewport": "1280x720"
  }
}
```

**Step 3: Server — signature generation**

The server computes the screenshot signature:

```
signatureProperties = ['browser', 'viewport']  (configurable)
input = "checkout" + "chromium" + "1280x720"
signature = SHA256(input) → "a3f8c2d1..."
filename = "checkout-a3f8c2d1.png"
```

**Step 4: Server — baseline lookup**

The server looks for `.vizzly/baselines/checkout-a3f8c2d1.png`.

- **Not found**: saves current screenshot to `.vizzly/current/checkout-a3f8c2d1.png`, returns `{ status: 'new' }`.

- **Found**: proceeds to comparison.

**Step 5: Server — honeydiff comparison**

`@vizzly-testing/honeydiff` compares the two PNGs:

1. Decode both images to pixel arrays
2. For each pixel pair, compute CIEDE2000 Delta E in CIE L*a*b* color space
3. Mark pixels where ΔE > threshold (2.0 by default) as "changed"
4. Group changed pixels into contiguous clusters
5. Discard clusters with fewer than `minClusterSize` pixels (10 by default)
6. Compute `diffPercentage` = changed pixels in valid clusters / total pixels

**Step 6: Server — result**

- **No valid clusters** → `{ status: 'passed', diffPercentage: 0.0 }`
- **Valid clusters found** → writes diff PNG to `.vizzly/diff/checkout-a3f8c2d1.png`, returns HTTP 422 with `{ tddMode: true, comparison: { status: 'failed', diffPercentage: 3.2, ... } }`

**Step 7: Client SDK — pass-through**

The SDK receives the response. For HTTP 422 with `tddMode: true`, it returns `{ success: true, status: 'failed', name, diffPercentage }` — `success: true` so the test can continue and capture remaining screenshots. The test doesn't fail from this return value; the build's exit code is set to 1 based on the final summary from `/flush`.

**Step 8: Dashboard — SSE push**

After each comparison, the server pushes the result to all connected dashboard clients via SSE. The dashboard updates in real time.

## Design Decisions

### Why a local HTTP server instead of file-based IPC?

Alternatives considered:
- **Named pipes / Unix sockets** — platform-specific, complex to implement cross-OS
- **Writing to a shared JSON file** — polling required, race conditions in parallel test workers
- **In-process library** — would require Vizzly to be bundled with every test framework adapter

HTTP is:
- Universally supported across all platforms (Windows, macOS, Linux)
- Works transparently with parallel test workers (each worker makes independent HTTP requests)
- Allows the comparison server to be completely decoupled from the test framework
- Enables the dashboard to connect to the same server without any IPC bridge

The overhead of HTTP on localhost is minimal (typically 5–15ms per request), which is negligible compared to screenshot capture time (50–500ms).

### Why CIEDE2000 instead of pixel diff (XOR)?

**Raw pixel diff** counts pixels where any RGB channel differs. Problems:

1. **Perceptually non-uniform**: RGBA(0, 0, 255, 255) vs RGBA(0, 0, 254, 255) looks identical but counts as a "different" pixel. RGBA(255, 0, 0, 255) vs RGBA(0, 0, 255, 255) looks completely different but also counts as one "different" pixel. The count doesn't reflect visual impact.

2. **Subpixel noise**: Antialiasing produces 1–3px differences at edges that look identical visually but fail pixel diff.

3. **Cross-platform failure**: Font rendering differs between macOS, Linux, and Windows, causing pixel diff to fail on CI even for identical visual output.

**CIEDE2000** converts colors to CIE L*a*b* space (designed to be perceptually uniform) and computes a distance that matches human perception. It tolerates antialiasing noise (ΔE < 1) while catching actual color changes (ΔE 3–30+).

### Why signature-hashed filenames for baselines?

Alternative: store baselines by name only (e.g., `checkout.png`).

Problems with name-only approach:
- A screenshot named `checkout` captured at 1920×1080 and 375×812 would overwrite each other
- Browser-specific rendering differences (Chrome vs Firefox vs Safari) can't be tracked separately
- Multi-environment setups (test vs staging) can't maintain separate baselines

Signature approach:
- Each unique combination of name + signature properties gets its own baseline file
- Adding a new browser to your test matrix creates new baseline files without affecting existing ones
- Changing a property's value (e.g., viewport from `1280x720` to `1440x900`) creates a new baseline automatically

The tradeoff: you can't easily "see" which baseline corresponds to which test at a glance. The report and dashboard handle this mapping for you.

### Why not store baselines in a database?

Baselines are PNG files on disk. Alternatives considered: SQLite, embedded key-value store, cloud-only.

PNG files on disk are:
- **Diffable in PRs** (GitHub, GitLab show image diffs)
- **Committable to Git** without extra tooling (or Git LFS for large repos)
- **Viewable immediately** with any image viewer, no tool required
- **Trivially portable** — `cp -r .vizzly/baselines/` is a complete backup

The tradeoff: a flat directory of hashed filenames isn't human-readable. The dashboard and report provide the human-readable view.

## One-Shot Mode vs Daemon Mode

**One-shot** (`vizzly tdd run "npm test"`):
- Server starts fresh every run
- No state bleed between runs
- Clean lifecycle: start → run → flush → report → exit
- Better for CI-like local runs where you want a clean report

**Daemon** (`vizzly tdd start` + tests separately):
- Server stays running between runs
- State accumulates until flush
- Better for `--watch` mode development: one server, many test runs
- Dashboard stays connected across runs

The server's internal state (current screenshots, comparison results) is reset after each `/flush`. The file system (`.vizzly/current/`, `.vizzly/diff/`) is also cleaned at the start of each run.

## Security Model

The TDD server only listens on `localhost`. It does not bind to external interfaces by default. This means:

- No authentication is required (only local processes can connect)
- The server URL in `.vizzly/server.json` is always `localhost`
- It's safe to run on shared development machines — other users can't connect

For cloud mode (`vizzly run`), all communication is via HTTPS to vizzly.dev with `VIZZLY_TOKEN` as the bearer token.

See also: [TDD Server](../concepts/tdd-server.md) | [Baseline System](../concepts/baseline-system.md) | [Comparison Engine](../concepts/comparison-engine.md)
