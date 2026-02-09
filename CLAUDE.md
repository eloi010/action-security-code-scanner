# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

A GitHub Actions monorepo providing automated security code scanning (CodeQL + Semgrep) for MetaMask repositories. It auto-detects languages, runs parallel scans, and uploads SARIF results to GitHub's Security tab.

## Common Commands

### Install & Lint

```bash
yarn install --immutable # Install dependencies (immutable lockfile)
yarn lint                # Prettier + depcheck across all packages
yarn lint:fix            # Auto-fix lint issues
yarn validate            # Validate action.yml files
```

### Testing

```bash
yarn test # Run tests across all workspaces (note: semgrep-action has no Jest tests)

# Language detector (from repo root)
yarn workspace @metamask/language-detector test
yarn workspace @metamask/language-detector test:unit
yarn workspace @metamask/language-detector test:integration
yarn workspace @metamask/language-detector test:watch

# CodeQL action (from repo root)
yarn workspace @metamask/codeql-action test
```

Tests require `NODE_OPTIONS=--experimental-vm-modules` (already set in package.json scripts).

### Semgrep Rules

Semgrep rules have their own test/validation tooling (requires Semgrep CLI installed):

```bash
packages/semgrep-action/bin/validate-rules   # Validate rule YAML syntax
packages/semgrep-action/bin/test             # Run Semgrep rule tests
packages/semgrep-action/bin/scan path/to/dir # Test rules against local code
```

## Architecture

### Monorepo Structure (Yarn workspaces)

Three packages under `packages/`, each a GitHub composite action:

1. **`language-detector`** - Detects repo languages via GitHub API, generates a job matrix for parallel CodeQL scanning. Core logic in `src/job-configurator.js`.

2. **`codeql-action`** - Runs CodeQL analysis per language. Loads repo-specific config (`repo-configs/<repo-name>.js`), merges with defaults, renders an EJS template (`config/codeql-template.yml`) into a final CodeQL config. Config priority: workflow input > repo config > `default.js`.

3. **`semgrep-action`** - Runs custom Semgrep rules. Rules live in `rules/src/<category>/<rule-name>.yml` with matching tests in `rules/test/<category>/<rule-name>.<ext>`.

### Workflow Orchestration

`.github/workflows/security-scan.yml` is the reusable workflow entry point:

- **Setup job**: language-detector creates the scan matrix
- **CodeQL jobs**: matrix of per-language codeql-action runs
- **Semgrep job**: single semgrep-action run across all languages
- **Finalize job**: aggregates results, notifications

### Key Conventions

- All packages use **ES Modules** (`"type": "module"`) — use `import`/`export`, not `require`
- Node.js >= 20.0.0
- Yarn 4.7.0 (with PnP or node-modules linker)
- Prettier for formatting (JSON, MD, YML, JS, TS)
- LavaMoat for supply chain security (`@lavamoat/allow-scripts`)

### Semgrep Rule Structure

Rule files follow a strict naming convention: the `id` field must match the filename (minus `.yml`). Each rule needs metadata fields (`tags`, `shortDescription`, `help`, `confidence`) for proper GitHub Security tab rendering. Test files mirror the `rules/src` structure under `rules/test` with appropriate language extensions.

### CodeQL Repo Configs

Per-repo configs are ES modules in `packages/codeql-action/repo-configs/`. They export `pathsIgnored`, `rulesExcluded`, `languages_config` (build mode, build command, JDK version), and `queries` (CodeQL query suites from `query-suites/`).
