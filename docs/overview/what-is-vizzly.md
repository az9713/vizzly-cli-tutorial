# What is Vizzly?

Vizzly is a visual regression testing tool that captures screenshots from your existing test suite, compares them against approved baselines using perceptual color difference, and shows you exactly what changed — locally on your machine, with no external dependencies.

## Mental Model

Think of Vizzly like a spell-checker for your UI. Just as a spell-checker compares your words against a known-correct dictionary, Vizzly compares your UI screenshots against a dictionary of approved "how it should look" images (baselines). When something drifts — even a subtle color shift or a 2px layout change — it flags it.

The key insight: Vizzly doesn't care how you take screenshots. It's a comparison and review system, not a screenshot-taking system. You bring the screenshots; Vizzly tells you whether they match what was approved.

## What Actually Happens When You Run `vizzly tdd start`

Most tools are black boxes. Here's exactly what Vizzly does:

**1. An HTTP server starts on port 47392.**

Vizzly launches a Node.js HTTP server on your local machine. This server accepts incoming screenshot data from your test suite.

**2. A server descriptor file is written to disk.**

Vizzly writes `.vizzly/server.json` containing the server's port number. This file acts as a discovery beacon for the client SDK.

**3. The dashboard becomes available.**

A local web dashboard opens (or is available at `http://localhost:47392`). It shows incoming screenshots and diffs in real time via Server-Sent Events (SSE).

**4. Your tests run as normal — but screenshots go to Vizzly.**

In your test code, you call `vizzlyScreenshot('my-page', screenshotBuffer)`. The client SDK walks up your directory tree looking for `.vizzly/server.json`. When it finds it, it POSTs the screenshot to the server.

**5. The server compares against the baseline.**

For each incoming screenshot, the server looks up the corresponding baseline file in `.vizzly/baselines/`. If no baseline exists, the screenshot is marked "new". If a baseline exists, honeydiff computes the CIEDE2000 Delta E difference. If the difference exceeds the threshold (default 2.0), it's "failed"; otherwise "passed".

**6. Results appear in the dashboard immediately.**

The SSE connection pushes each comparison result to the dashboard as it arrives. You see diffs live, not after all tests finish.

## Architecture Overview

```
Your test suite
    │
    │ vizzlyScreenshot('name', buffer)
    ▼
Client SDK (auto-discovered)
    │
    │ HTTP POST /screenshot
    ▼
TDD Server (:47392)
    │
    ├─── .vizzly/server.json      ← discovery beacon
    ├─── .vizzly/baselines/       ← approved screenshots
    ├─── .vizzly/current/         ← this run's screenshots
    ├─── .vizzly/diff/            ← diff images
    └─── .vizzly/report/          ← static HTML report
    │
    │ SSE push
    ▼
Dashboard (http://localhost:47392)
```

**The client SDK is intentionally thin.** It is a tiny HTTP client that POSTs to the server. It has no comparison logic — that all lives in the server. This means you can use Vizzly with any test framework that can execute JavaScript.

**The server is stateful per-run.** Each time you start a run, the server tracks which screenshots arrived, compares them, and accumulates results. When tests finish, your code calls `vizzlyFlush()` which POSTs to `/flush`, signaling the server to finalize the run and print the summary.

## Local TDD vs Cloud Mode

Vizzly has two modes:

| Feature | Local TDD | Cloud |
|---------|-----------|-------|
| Command | `vizzly tdd start` / `vizzly tdd run "..."` | `vizzly run "..."` |
| Token required | No | Yes (`VIZZLY_TOKEN`) |
| Baselines stored | `.vizzly/baselines/` on disk | vizzly.dev cloud |
| Review UI | Local dashboard | vizzly.dev web app |
| Team sharing | No | Yes |
| Use case | Local development, fast iteration | CI/CD, team review |

**Local TDD mode** is designed for a single developer iterating quickly. No account, no token, no upload. Everything lives in `.vizzly/` in your project directory.

**Cloud mode** (`vizzly run`) uploads screenshots to vizzly.dev, where your team can review and approve changes through a collaborative web interface. Baselines are managed in the cloud and downloaded on demand.

You can combine both: develop locally with TDD mode, then run CI with cloud mode using the same test code.

## How It All Fits Together

A typical development workflow:

1. `vizzly tdd start` — server starts, dashboard is live
2. `npm test -- --watch` — your tests run in watch mode
3. For each test run: screenshots POST to the server, comparisons appear in the dashboard
4. New screenshots appear as "new" (yellow) — you accept them as baselines from the dashboard
5. You change some CSS — the affected screenshot appears as "failed" (red) with a diff
6. Intentional? Accept it as the new baseline. Regression? Fix your code.

The dashboard is purely a convenience. Every result is also written to disk under `.vizzly/`, so you can inspect images, run `vizzly context build current --source local`, or integrate with scripts.

See also: [TDD Server](../concepts/tdd-server.md) | [Baseline System](../concepts/baseline-system.md) | [Key Concepts](key-concepts.md)
