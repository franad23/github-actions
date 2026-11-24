# React Build Check with GitHub Actions

## Overview

This workflow implements a basic Continuous Integration (CI) pipeline using GitHub Actions.

Its purpose is to automatically verify that a React application builds successfully whenever a Pull Request is opened or updated.

The workflow supports both **npm and pnpm**, automatically detecting which package manager the project uses.

## Prerequisites

Before configuring the workflow, make sure that:

- Your React project is hosted on GitHub.
- Your project uses either npm or pnpm.
- Your lockfile is committed to the repository.
- Your `package.json` includes a valid `build` script.
- GitHub Actions is enabled in your repository.

### Package manager configuration

If you are using npm, your project must contain:

```text
package-lock.json
```

If you are using pnpm, your project must contain:

```text
pnpm-lock.yaml
```

For pnpm projects, you must also specify your pnpm version in the `package.json` file.

For example:

```json
{
  "packageManager": "pnpm@10.9.8"
}
```

Replace the example version with the version installed in your development environment. You can check it by running:

```bash
pnpm --version
```

Do not commit both lockfiles. Each project should use a single package manager.

## 1. Create the workflow

Inside the root directory of your project, create the following directory and file:

```text
.github/
└── workflows/
    └── build-check.yml
```

Add the following configuration to `build-check.yml`:

```yaml
name: React Build Check

on:
  pull_request:
    branches: [main, dev]

permissions:
  contents: read

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v7

      - name: Detect package manager
        id: package-manager
        shell: bash
        run: |
          if [ -f pnpm-lock.yaml ] && [ -f package-lock.json ]; then
            echo "::error::Multiple package manager lockfiles detected."
            exit 1
          elif [ -f pnpm-lock.yaml ]; then
            echo "manager=pnpm" >> "$GITHUB_OUTPUT"
          elif [ -f package-lock.json ]; then
            echo "manager=npm" >> "$GITHUB_OUTPUT"
          else
            echo "::error::No supported lockfile found."
            exit 1
          fi

      - name: Setup pnpm
        if: steps.package-manager.outputs.manager == 'pnpm'
        uses: pnpm/action-setup@v6

      - name: Setup Node.js
        uses: actions/setup-node@v7
        with:
          node-version: 24
          cache: ${{ steps.package-manager.outputs.manager }}

      - name: Install dependencies
        run: |
          if [ "${{ steps.package-manager.outputs.manager }}" = "pnpm" ]; then
            pnpm install --frozen-lockfile
          else
            npm ci
          fi

      - name: Build project
        run: ${{ steps.package-manager.outputs.manager }} run build
