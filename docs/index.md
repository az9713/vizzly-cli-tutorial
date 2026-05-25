# Vizzly CLI Documentation

Vizzly is a visual regression testing CLI that runs a local HTTP server during your tests, compares screenshots against approved baselines using perceptual color difference, and gives you an instant dashboard to review and accept changes — no cloud account required for local development.

This is a tutorial fork of [vizzly-testing/cli](https://github.com/vizzly-testing/cli) with comprehensive documentation written from scratch to demystify how the tool works internally.

## Navigation

| Section | Description |
|---------|-------------|
| **Overview** | |
| [What is Vizzly?](overview/what-is-vizzly.md) | Mental model, architecture, how it all fits together |
| [Key Concepts](overview/key-concepts.md) | Glossary of every important term |
| **Getting Started** | |
| [Prerequisites](getting-started/prerequisites.md) | Exact requirements with verify commands |
| [Quickstart](getting-started/quickstart.md) | Working in under 15 minutes |
| [Onboarding](getting-started/onboarding.md) | Zero-to-hero narrative walkthrough |
| **Concepts** | |
| [TDD Server](concepts/tdd-server.md) | How the local HTTP server works |
| [Baseline System](concepts/baseline-system.md) | Signature hashing, storage, pass/fail/new logic |
| [Comparison Engine](concepts/comparison-engine.md) | Honeydiff, CIEDE2000, threshold tuning |
| [Client SDK](concepts/client-sdk.md) | Auto-discovery, vizzlyScreenshot, vizzlyFlush |
| [Context Bundles](concepts/context-bundles.md) | Structured JSON evidence for LLM agents |
| **Guides** | |
| [Playwright Integration](guides/playwright-integration.md) | Step-by-step Playwright setup |
| [Jest Integration](guides/jest-integration.md) | Step-by-step Jest setup |
| [CI/CD Integration](guides/ci-cd-integration.md) | GitHub Actions workflow |
| [Setting Baselines](guides/setting-baselines.md) | Bulk update, per-screenshot, environments |
| [Tuning Thresholds](guides/tuning-thresholds.md) | Choosing the right Delta E threshold |
| **Reference** | |
| [Configuration](reference/configuration.md) | Every vizzly.config.js key and env var |
| [CLI Commands](reference/cli-commands.md) | Every command and option |
| [Client API](reference/client-api.md) | Every exported SDK function |
| **Architecture** | |
| [How It Works](architecture/how-it-works.md) | System design and data flow |
| **Troubleshooting** | |
| [Common Issues](troubleshooting/common-issues.md) | Symptom → cause → fix for top problems |
| **Original Repo Docs** | |
| [Browser Flags](browser-flags.md) | (from original repo) Browser flag reference |
| [JSON Output](json-output.md) | (from original repo) JSON output format |
