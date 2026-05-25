# CI/CD Integration

Run visual regression tests in CI using Vizzly cloud mode (`vizzly run`). This guide covers GitHub Actions setup with `VIZZLY_TOKEN`.

## Overview

Local TDD mode runs everything on your machine with no token. For CI, you use **cloud mode** (`vizzly run` instead of `vizzly tdd run`). Cloud mode:
- Uploads screenshots to vizzly.dev
- Compares against cloud-stored baselines approved by your team
- Requires `VIZZLY_TOKEN`
- Posts results to the vizzly.dev dashboard for team review

If you want CI to use local baselines (committed to your repo) rather than cloud baselines, see [Using Local Baselines in CI](#using-local-baselines-in-ci) below.

## Prerequisites

1. A vizzly.dev account and project token
2. Your test suite is already working locally with `vizzly tdd run`

## Step 1: Add your token to GitHub Secrets

In your GitHub repository: **Settings → Secrets and variables → Actions → New repository secret**

- Name: `VIZZLY_TOKEN`
- Value: your project token from vizzly.dev

## Step 2: Add the GitHub Actions workflow

Create `.github/workflows/visual-tests.yml`:

```yaml
name: Visual Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  visual-tests:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium

      - name: Install Vizzly CLI
        run: npm install -g @vizzly-testing/cli

      - name: Run visual tests
        env:
          VIZZLY_TOKEN: ${{ secrets.VIZZLY_TOKEN }}
        run: vizzly run "npx playwright test" --wait

      - name: Upload visual report (on failure)
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: vizzly-report
          path: .vizzly/report/
          retention-days: 7
```

## Step 3: Configure branch tracking (optional)

Pass branch and commit information for better tracking:

```yaml
      - name: Run visual tests
        env:
          VIZZLY_TOKEN: ${{ secrets.VIZZLY_TOKEN }}
        run: |
          vizzly run "npx playwright test" \
            --wait \
            --branch "${{ github.head_ref || github.ref_name }}" \
            --commit "${{ github.sha }}"
```

## Step 4: Handle exit codes

`vizzly run --wait` exits with code `1` if visual differences are found (and not approved on vizzly.dev). The GitHub Actions step will fail automatically.

To capture the result and post a comment:

```yaml
      - name: Run visual tests
        id: vizzly
        env:
          VIZZLY_TOKEN: ${{ secrets.VIZZLY_TOKEN }}
        run: |
          vizzly run "npx playwright test" --wait --json > vizzly-result.json
          cat vizzly-result.json
        continue-on-error: true

      - name: Parse visual test results
        run: |
          STATUS=$(jq -r '.status' vizzly-result.json)
          PASSED=$(jq -r '.summary.passed' vizzly-result.json)
          FAILED=$(jq -r '.summary.failed' vizzly-result.json)
          NEW=$(jq -r '.summary.new' vizzly-result.json)
          echo "Visual tests: $STATUS ($PASSED passed, $FAILED failed, $NEW new)"

      - name: Fail if visual tests failed
        run: |
          EXIT_CODE=$(jq -r '.exitCode' vizzly-result.json)
          exit $EXIT_CODE
```

## Using Local Baselines in CI

If you prefer to commit baselines to your repo and compare against them in CI (without cloud):

```yaml
      - name: Run visual tests (local baselines)
        run: |
          vizzly tdd run "npx playwright test" --fail-on-diff --json
```

With `--fail-on-diff`, the command exits with code 1 if any comparison fails. No `VIZZLY_TOKEN` needed.

This approach:
- Uses baselines from `.vizzly/baselines/` (must be committed to repo)
- No upload to cloud
- No team review — failures must be investigated locally
- Generates static HTML report in `.vizzly/report/`

```yaml
      - name: Upload visual report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: vizzly-visual-report
          path: .vizzly/report/
          retention-days: 30
```

## Parallel Test Runs

If you run Playwright tests in parallel across multiple CI shards, use `--parallel-id` to group them into one build:

```yaml
jobs:
  visual-tests:
    strategy:
      matrix:
        shard: [1, 2, 3]

    steps:
      # ... setup steps ...

      - name: Run visual tests (shard ${{ matrix.shard }})
        env:
          VIZZLY_TOKEN: ${{ secrets.VIZZLY_TOKEN }}
        run: |
          vizzly run "npx playwright test --shard=${{ matrix.shard }}/3" \
            --parallel-id "pr-${{ github.event.number }}"
```

Each shard uploads under the same `parallel-id`, and vizzly.dev aggregates them into one build.

## JSON Output Reference

`vizzly run --json` outputs:

```json
{
  "status": "completed",
  "exitCode": 0,
  "comparisons": [
    {
      "name": "homepage",
      "status": "passed",
      "signature": "a3f8c2d1...",
      "diffPercentage": 0.0,
      "threshold": 2.0,
      "paths": {
        "baseline": null,
        "current": ".vizzly/current/homepage-a3f8c2d1.png",
        "diff": null
      },
      "viewport": { "width": 1280, "height": 720 },
      "browser": "chromium"
    }
  ],
  "summary": {
    "total": 5,
    "passed": 4,
    "failed": 1,
    "new": 0
  },
  "reportPath": ".vizzly/report/index.html",
  "contextCommand": "vizzly context build current --source local --agent"
}
```

`exitCode` is `0` for success, `1` for any failures or errors.

## Example: Full PR Workflow

```yaml
name: PR Checks

on:
  pull_request:
    branches: [main]

jobs:
  visual:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - run: npm ci
      - run: npx playwright install --with-deps chromium
      - run: npm install -g @vizzly-testing/cli

      - name: Start app
        run: npm run build && npm run serve &
        # Wait for app to be ready
      - run: npx wait-on http://localhost:3000

      - name: Visual regression tests
        env:
          VIZZLY_TOKEN: ${{ secrets.VIZZLY_TOKEN }}
        run: |
          vizzly run "npx playwright test" \
            --wait \
            --branch "${{ github.head_ref }}" \
            --commit "${{ github.sha }}" \
            --json > vizzly-result.json

      - name: Show summary
        if: always()
        run: |
          echo "## Visual Test Results" >> $GITHUB_STEP_SUMMARY
          echo "" >> $GITHUB_STEP_SUMMARY
          jq -r '"Status: \(.status) | Passed: \(.summary.passed) | Failed: \(.summary.failed) | New: \(.summary.new)"' vizzly-result.json >> $GITHUB_STEP_SUMMARY
```

See also: [CLI Commands Reference](../reference/cli-commands.md) | [Setting Baselines](setting-baselines.md) | [Configuration Reference](../reference/configuration.md)
