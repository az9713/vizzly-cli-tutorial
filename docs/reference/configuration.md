# Configuration Reference

Complete reference for `vizzly.config.js` and all environment variables.

## Config File

Vizzly uses [cosmiconfig](https://github.com/cosmiconfig/cosmiconfig) to find your configuration. It searches (in order):
- `vizzly.config.js`
- `vizzly.config.mjs`
- `vizzly.config.cjs`
- `vizzly.config.json`
- `"vizzly"` key in `package.json`

The config file must use the default export:

```javascript
export default {
  // ... options
};
```

Generate a config file:

```bash
vizzly init
```

## Full Config with Defaults

```javascript
export default {
  server: {
    port: 47392,
    timeout: 30000
  },
  build: {
    name: 'Build {timestamp}',
    environment: 'test'
  },
  upload: {
    screenshotsDir: './screenshots',
    batchSize: 10,
    timeout: 30000
  },
  comparison: {
    threshold: 2.0,
    minClusterSize: 10
  },
  tdd: {
    openReport: false
  }
};
```

## `server` options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `server.port` | `number` | `47392` | Port the local TDD server listens on. Change if port is in use. |
| `server.timeout` | `number` | `30000` | Milliseconds to wait after the last screenshot before timing out. Increase for slow test suites. |

## `build` options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `build.name` | `string` | `'Build {timestamp}'` | Display name for builds. `{timestamp}` is replaced with ISO timestamp. |
| `build.environment` | `string` | `'test'` | Environment label. Used to namespace baselines. Common values: `test`, `staging`, `production`. |

## `upload` options

Used by `vizzly upload <dir>` command.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `upload.screenshotsDir` | `string` | `'./screenshots'` | Default directory for `vizzly upload` when no directory is specified. |
| `upload.batchSize` | `number` | `10` | Number of screenshots to upload in parallel. |
| `upload.timeout` | `number` | `30000` | Per-request timeout in milliseconds for uploads. |

## `comparison` options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `comparison.threshold` | `number` | `2.0` | Maximum CIEDE2000 Delta E before a pixel is marked "changed." Range: 0–100. See [Comparison Engine](../concepts/comparison-engine.md). |
| `comparison.minClusterSize` | `number` | `10` | Minimum number of contiguous changed pixels required to count as a difference. Smaller clusters are ignored. |

## `tdd` options

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `tdd.openReport` | `boolean` | `false` | Whether to automatically open the HTML report in your browser after a one-shot run. |

## Environment Variables

Environment variables override config file values. Vizzly reads `.env` files automatically via dotenv.

| Variable | Description |
|----------|-------------|
| `VIZZLY_TOKEN` | API token for cloud mode. Required for `vizzly run`, `vizzly login`, and downloading cloud baselines. |
| `VIZZLY_SERVER_URL` | Override the TDD server URL. Useful when the server is on a different host or port than auto-discovery would find. Example: `http://localhost:9000`. |
| `VIZZLY_BUILD_ID` | Override the build ID for the current run. Useful for grouping parallel test runs. |
| `VIZZLY_TDD_MODE` | Set to `1` or `true` to force TDD mode behavior. |
| `VIZZLY_ENABLED` | Set to `0` or `false` to disable the Vizzly client SDK. All `vizzlyScreenshot()` calls return `null` silently. |
| `VIZZLY_CLIENT_LOG_LEVEL` | Client SDK log verbosity: `debug`, `info`, `warn`, `error`. Default: `error`. |
| `VIZZLY_HOME` | Override the `.vizzly/` directory location. Default: `.vizzly/` relative to project root. |

## CLI Overrides

Most config values can be overridden on the CLI. CLI flags take precedence over config file, which takes precedence over defaults.

| Config key | CLI flag |
|-----------|---------|
| `server.port` | `--port <n>` on `vizzly tdd start` |
| `comparison.threshold` | `--threshold <n>` |
| `comparison.minClusterSize` | `--min-cluster-size <n>` |
| `server.timeout` | `--timeout <ms>` |
| `build.environment` | `--environment <env>` |
| (global) | `--token <tok>` |
| (global) | `--config <path>` |
| (global) | `--verbose` |
| (global) | `--json` |
| (global) | `--no-color` |

## Config File Location

By default, Vizzly looks for the config file starting from the current working directory and walking up. To specify an explicit config file:

```bash
vizzly tdd run "npm test" --config ./path/to/vizzly.config.js
```

## TypeScript Types

If you want TypeScript type checking for your config file:

```typescript
// vizzly.config.ts
import type { VizzlyConfig } from '@vizzly-testing/cli/config';

const config: VizzlyConfig = {
  comparison: {
    threshold: 1.5
  }
};

export default config;
```

See also: [CLI Commands](cli-commands.md) | [Tuning Thresholds](../guides/tuning-thresholds.md) | [TDD Server](../concepts/tdd-server.md)
