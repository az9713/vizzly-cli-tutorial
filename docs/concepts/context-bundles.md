# Context Bundles

Context bundles are structured JSON documents that package visual evidence from a Vizzly build or comparison into a single, queryable artifact. They're designed to be fed to LLM agents as prompt context, but they're also useful for debugging, CI reporting, and integration with other tools.

## The Problem They Solve

After running visual tests, you have evidence scattered across multiple files: baseline PNGs, current PNGs, diff PNGs, comparison metadata. If you want to give an LLM agent a summary of what changed visually in the UI, you'd need to assemble this manually.

Context bundles do the assembly for you: one command, one JSON payload with everything an agent needs to understand the visual state of the build.

## Commands

### Local context (from `.vizzly/` directory)

```bash
# Summary bundle for the current local build
vizzly context build current --source local

# Compact agent handoff (good for prompt assembly)
vizzly context build current --source local --agent

# Full bundle including all comparison details
vizzly context build current --source local --full

# Machine-readable JSON
vizzly context build current --source local --json

# Combined: compact agent format as JSON
vizzly context build current --source local --agent --json
```

### Cloud context (from vizzly.dev)

```bash
# Fetch context for a specific cloud build
vizzly context build abc123def456

# Fetch a specific comparison
vizzly context comparison def456ghi789 --json

# Screenshot-level context
vizzly context screenshot build-detail-screenshots --source local --json
```

## Output Formats

### Default (human-readable)

```
Build: current (local)
Environment: test
Timestamp: 2024-01-15T10:30:00.000Z

Comparisons:
  ✓ homepage           passed  ΔE 0.00
  ✓ checkout-cart      passed  ΔE 0.12
  ✗ checkout-payment   failed  ΔE 4.32  (threshold: 2.0)
  ● marketing-hero     new

Summary: 2 passed, 1 failed, 1 new (4 total)

To inspect further: vizzly context build current --source local --full
```

### `--agent` format (compact, prompt-ready)

The `--agent` flag produces a compact handoff summary optimized for LLM prompts. It's smaller than `--full` but contains enough information for an agent to understand what changed.

```json
{
  "source": "local",
  "buildId": "current",
  "environment": "test",
  "summary": {
    "total": 4,
    "passed": 2,
    "failed": 1,
    "new": 1
  },
  "failures": [
    {
      "name": "checkout-payment",
      "diffPercentage": 4.32,
      "threshold": 2.0,
      "diffPath": ".vizzly/diff/checkout-payment-c9d2e3f4.png"
    }
  ],
  "new": [
    { "name": "marketing-hero" }
  ],
  "contextCommand": "vizzly context build current --source local --agent"
}
```

### `--full` format

Includes complete metadata for every comparison: paths to all three images (baseline, current, diff), viewport, browser, properties, signature hash. Use when an agent needs to access or display specific images.

```json
{
  "source": "local",
  "buildId": "current",
  "environment": "test",
  "timestamp": "2024-01-15T10:30:00.000Z",
  "comparisons": [
    {
      "name": "homepage",
      "status": "passed",
      "signature": "a3f8c2d1...",
      "diffPercentage": 0.0,
      "threshold": 2.0,
      "paths": {
        "baseline": ".vizzly/baselines/homepage-a3f8c2d1.png",
        "current": ".vizzly/current/homepage-a3f8c2d1.png",
        "diff": null
      },
      "viewport": { "width": 1280, "height": 720 },
      "browser": "chromium"
    },
    {
      "name": "checkout-payment",
      "status": "failed",
      "signature": "c9d2e3f4...",
      "diffPercentage": 4.32,
      "threshold": 2.0,
      "paths": {
        "baseline": ".vizzly/baselines/checkout-payment-c9d2e3f4.png",
        "current": ".vizzly/current/checkout-payment-c9d2e3f4.png",
        "diff": ".vizzly/diff/checkout-payment-c9d2e3f4.png"
      },
      "viewport": { "width": 1440, "height": 900 },
      "browser": "chromium"
    }
  ],
  "summary": { "total": 4, "passed": 2, "failed": 1, "new": 1 }
}
```

## Using Context Bundles with LLM Agents

The primary use case is feeding the context bundle to an LLM agent that needs to understand visual state before making UI changes.

**Example prompt pattern:**

```
Current visual test state:
<paste output of: vizzly context build current --source local --agent --json>

The checkout-payment screenshot is failing with ΔE 4.32.
Please investigate the CSS change that caused this regression
and suggest a fix.
```

**In an automated agent workflow:**

```bash
# Run tests
vizzly tdd run "npm test" --json > /tmp/tdd-result.json

# Extract context for the agent
vizzly context build current --source local --agent --json > /tmp/visual-context.json

# Feed both to your agent
my-agent --visual-context /tmp/visual-context.json --tdd-result /tmp/tdd-result.json
```

## `--include` Flag

When you don't want the full bundle, use `--include` to select specific fields:

```bash
vizzly context build current --source local --include screenshots,diffs --json
```

Valid fields: `screenshots`, `diffs`, `comments`, `metadata`.

## `--source` Flag

| Source | Description |
|--------|-------------|
| `local` | Reads from `.vizzly/` directory on disk |
| `cloud` | Downloads from vizzly.dev (requires `VIZZLY_TOKEN`) |
| `auto` | Tries local first, falls back to cloud |

## When to Use Context Bundles

- **CI reporting**: Parse `--json` output to generate custom reports or post summaries to Slack/GitHub
- **Agent debugging**: Give an AI agent the exact list of what changed visually before asking it to fix CSS
- **Diff review automation**: Feed context to a model to automatically categorize failures (intentional vs. regression)
- **Handoff to reviewers**: Share the compact `--agent` bundle with teammates for async review

See also: [CLI Commands Reference](../reference/cli-commands.md) | [JSON Output](../json-output.md) | [What is Vizzly?](../overview/what-is-vizzly.md)
