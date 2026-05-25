# CLI Commands Reference

Complete reference for every `vizzly` command and option.

## Global Options

These options apply to all commands.

| Option | Description |
|--------|-------------|
| `--config <path>` | Path to config file (overrides auto-discovery) |
| `--token <token>` | API token (overrides `VIZZLY_TOKEN` env var) |
| `--verbose` | Enable verbose output |
| `--log-level <level>` | Log level: `debug`, `info`, `warn`, `error` |
| `--json [fields]` | Output results as JSON. Optionally specify comma-separated fields to include. |
| `--color` | Force color output |
| `--no-color` | Disable color output |
| `--strict` | Strict mode — treat warnings as errors |
| `-V, --version` | Show version number |
| `-h, --help` | Show help |

---

## `vizzly init`

Generate a `vizzly.config.js` file in the current directory.

```bash
vizzly init
```

Creates:

```javascript
export default {
  server: { port: 47392, timeout: 30000 },
  build: { name: 'Build {timestamp}', environment: 'test' },
  upload: { screenshotsDir: './screenshots', batchSize: 10, timeout: 30000 },
  comparison: { threshold: 2.0 },
  tdd: { openReport: false }
};
```

---

## `vizzly tdd start`

Start the local TDD server as a persistent daemon. The server runs until you stop it with Ctrl-C.

```bash
vizzly tdd start [options]
```

### Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--port <n>` | number | `47392` | Port to listen on |
| `--open` | flag | false | Open the dashboard in your default browser on start |
| `--baseline-build <id>` | string | — | Download baselines from this cloud build ID. Requires token. |
| `--baseline-comparison <id>` | string | — | Download baselines from this cloud comparison ID. Requires token. |
| `--environment <env>` | string | `test` | Environment label for this run |
| `--threshold <n>` | number | `2.0` | CIEDE2000 Delta E threshold |
| `--min-cluster-size <n>` | number | `10` | Minimum cluster size |
| `--timeout <ms>` | number | `30000` | Server timeout in milliseconds |
| `--fail-on-diff` | flag | false | Exit with code 1 if any visual differences are found |
| `--token <token>` | string | — | API token (for baseline download) |

### Examples

```bash
# Basic start
vizzly tdd start

# Start with dashboard open
vizzly tdd start --open

# Custom port
vizzly tdd start --port 9000

# Strict mode: exit 1 on any diff
vizzly tdd start --fail-on-diff

# Download baselines from cloud build
vizzly tdd start --baseline-build abc123def456
```

---

## `vizzly tdd run <command>`

Start the TDD server, run a test command, wait for completion, generate a report, and exit.

```bash
vizzly tdd run <command> [options]
```

### Arguments

| Argument | Description |
|----------|-------------|
| `<command>` | Test command to execute (quoted if it contains spaces or flags) |

### Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--set-baseline` | flag | false | Accept all current screenshots as new baselines |
| `--fail-on-diff` | flag | false | Exit with code 1 if any comparisons fail |
| `--no-open` | flag | false | Don't open the report after completion |
| `--port <n>` | number | `47392` | Port to listen on |
| `--environment <env>` | string | `test` | Environment label |
| `--threshold <n>` | number | `2.0` | CIEDE2000 threshold |
| `--min-cluster-size <n>` | number | `10` | Minimum cluster size |
| `--timeout <ms>` | number | `30000` | Server timeout |
| `--branch <branch>` | string | auto-detected | Git branch name |
| `--commit <sha>` | string | auto-detected | Git commit SHA |

### Examples

```bash
# Basic run
vizzly tdd run "npm test"

# Set baselines (first run)
vizzly tdd run "npm test" --set-baseline

# Don't open report after run
vizzly tdd run "npm test" --no-open

# JSON output
vizzly tdd run "npm test" --json

# Custom threshold for this run
vizzly tdd run "npm test" --threshold 3.0

# Playwright with specific browser
vizzly tdd run "npx playwright test --project=chromium"

# Fail CI if any diffs
vizzly tdd run "npm test" --fail-on-diff
```

---

## `vizzly run <command>`

Run tests with cloud integration. Uploads screenshots to vizzly.dev. Requires `VIZZLY_TOKEN`.

```bash
vizzly run <command> [options]
```

### Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--wait` | flag | false | Wait for cloud comparison to complete, then exit with appropriate code |
| `--branch <branch>` | string | auto-detected | Git branch name for the build |
| `--commit <sha>` | string | auto-detected | Git commit SHA |
| `--parallel-id <id>` | string | — | Group parallel shards into one build |
| `--upload-all` | flag | false | Upload all screenshots, even those that passed |

### Examples

```bash
# Basic cloud run
export VIZZLY_TOKEN=your-token
vizzly run "npm test"

# Wait for results (CI mode)
vizzly run "npm test" --wait

# With branch/commit tracking
vizzly run "npm test" --wait --branch main --commit abc123

# Parallel shards
vizzly run "npx playwright test --shard=1/3" --parallel-id "pr-42"
vizzly run "npx playwright test --shard=2/3" --parallel-id "pr-42"
vizzly run "npx playwright test --shard=3/3" --parallel-id "pr-42"
```

---

## `vizzly context build <id>`

Fetch a visual context bundle for a build.

```bash
vizzly context build <id> [options]
```

### Arguments

| Argument | Description |
|----------|-------------|
| `<id>` | Build ID, or `current` for the most recent local build |

### Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--source <source>` | `auto\|cloud\|local` | `auto` | Where to read the build from |
| `--agent` | flag | false | Output a compact agent-friendly summary |
| `--full` | flag | false | Output the complete build payload including all comparison details |
| `--include <fields>` | string | — | Comma-separated fields to include: `screenshots`, `diffs`, `comments`, `metadata` |

### Examples

```bash
# Local build summary
vizzly context build current --source local

# Agent-friendly compact JSON
vizzly context build current --source local --agent --json

# Full payload
vizzly context build current --source local --full --json

# Cloud build
vizzly context build abc123def456

# Include only failures
vizzly context build current --source local --json | jq '.comparisons[] | select(.status == "failed")'
```

---

## `vizzly context comparison <id>`

Fetch context for a specific comparison.

```bash
vizzly context comparison <id> [options]
```

### Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--json` | flag | false | Output as JSON |
| `--full` | flag | false | Full payload |

---

## `vizzly context screenshot <name>`

Fetch context for a specific screenshot within a build.

```bash
vizzly context screenshot <name> [options]
```

### Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--source <source>` | `auto\|cloud\|local` | `auto` | Data source |
| `--json` | flag | false | Output as JSON |

---

## `vizzly upload <dir>`

Upload a directory of screenshots to vizzly.dev. Useful when you already have screenshots on disk.

```bash
vizzly upload <dir> [options]
```

### Arguments

| Argument | Description |
|----------|-------------|
| `<dir>` | Directory containing PNG files to upload |

### Examples

```bash
vizzly upload ./screenshots
vizzly upload ./test-output/screenshots --token your-token
```

---

## `vizzly login`

Authenticate with vizzly.dev via browser OAuth.

```bash
vizzly login
```

Stores the token in your local config. After login, `VIZZLY_TOKEN` does not need to be set explicitly.

---

## `vizzly logout`

Remove stored authentication credentials.

```bash
vizzly logout
```

---

## `vizzly doctor`

Validate your local Vizzly setup. Checks Node.js version, config file, token, and connectivity.

```bash
vizzly doctor
```

---

## Exit Codes

| Code | Meaning |
|------|---------|
| `0` | Success — all comparisons passed (or no failures detected) |
| `1` | Failure — visual regressions detected, or error occurred |

See also: [Configuration Reference](configuration.md) | [Client API Reference](client-api.md)
