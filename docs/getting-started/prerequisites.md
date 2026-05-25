# Prerequisites

## Required

### Node.js 22 or later

Vizzly uses Node.js native test runner features and modern ESM syntax that require Node.js 22+.

Verify your version:

```bash
node --version
# Expected: v22.0.0 or higher
```

If you need to install or upgrade Node.js, use [nvm](https://github.com/nvm-sh/nvm) (macOS/Linux) or [nvm-windows](https://github.com/coreybutler/nvm-windows):

```bash
nvm install 22
nvm use 22
```

### npm 10 or later

npm 10+ ships with Node.js 22. Verify:

```bash
npm --version
# Expected: 10.x.x or higher
```

### Vizzly CLI

Install globally:

```bash
npm install -g @vizzly-testing/cli
```

Verify:

```bash
vizzly --version
# Expected: 0.32.0 or higher
```

## Optional

### VIZZLY_TOKEN (cloud mode only)

A project API token is **only required** when using cloud features:
- `vizzly run "..."` (uploads to vizzly.dev)
- `vizzly login`
- Downloading baselines from cloud builds (`--baseline-build`, `--baseline-comparison`)

For local TDD mode — `vizzly tdd start` and `vizzly tdd run "..."` — no token is needed. Everything runs on your machine.

If you do need a token, get it from your vizzly.dev project settings and set it:

```bash
export VIZZLY_TOKEN=your-project-token
```

Or add it to your `.env` file (which Vizzly reads automatically via dotenv):

```
VIZZLY_TOKEN=your-project-token
```

### A test framework

Vizzly works with any test runner that can execute JavaScript. Common choices:
- [Playwright](https://playwright.dev) — recommended for browser tests
- [Jest](https://jestjs.io) — recommended for component tests
- [Vitest](https://vitest.dev) — Jest-compatible, works identically
- [Cypress](https://cypress.io)
- Node.js built-in test runner

See [Playwright Integration](../guides/playwright-integration.md) and [Jest Integration](../guides/jest-integration.md) for step-by-step setup.

## Verify Everything

Run the built-in setup check:

```bash
vizzly doctor
```

This verifies your Node.js version, checks for a config file, and validates any token you have configured.
