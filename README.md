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
```

## 2. Understanding the workflow

### Workflow configuration

| Configuration | Description |
|---|---|
| `name` | Defines the workflow name displayed on GitHub. |
| `on` | Defines the events that trigger the workflow. |
| `branches` | Specifies the target branches for Pull Requests. |
| `permissions` | Restricts repository content access to read-only. |
| `jobs` | Defines the jobs that will be executed. |
| `runs-on` | Specifies the operating system used by the runner. |
| `steps` | Defines the sequence of tasks executed by the job. |
| `uses` | Executes an existing GitHub Action. |
| `run` | Executes a shell command. |
| `id` | Identifies a step so its outputs can be accessed by subsequent steps. |
| `if` | Executes a step only when a condition is met. |

### Workflow steps

1. **Checkout repository:** Downloads the repository source code into the runner.
2. **Detect package manager:** Determines whether the project uses npm or pnpm based on its lockfile.
3. **Setup pnpm:** Installs pnpm only when the project uses it.
4. **Setup Node.js:** Configures Node.js 24 and enables dependency caching for the detected package manager.
5. **Install dependencies:** Installs dependencies using the appropriate package manager.
6. **Build project:** Executes the build command and verifies that the application builds successfully.

### Automatic package manager detection

The workflow detects the package manager using the following rules:

| Condition | Result |
|---|---|
| `package-lock.json` exists | Use npm. |
| `pnpm-lock.yaml` exists | Use pnpm. |
| Both lockfiles exist | Fail the workflow. |
| No supported lockfile exists | Fail the workflow. |

The detected package manager is stored as a step output using `$GITHUB_OUTPUT`.

Subsequent steps access this output through:

```yaml
${{ steps.package-manager.outputs.manager }}
```

This allows the workflow to dynamically configure caching, install dependencies, and execute the build command.

### Dependency installation

Depending on the detected package manager, the workflow executes one of the following commands:

**npm:**

```bash
npm ci
```

**pnpm:**

```bash
pnpm install --frozen-lockfile
```

Both commands install dependencies using the committed lockfile without updating it.

This ensures consistent and reproducible dependency installations across development and CI environments.

### Build execution

For npm projects:

```bash
npm run build
```

For pnpm projects:

```bash
pnpm run build
```

The workflow fails if the build command returns a non-zero exit code.

### Dependency caching

GitHub Actions caches package manager data to reduce dependency download times during subsequent executions.

The cache is automatically configured for npm or pnpm based on the detection step.

Caching does not replace dependency installation.

## 3. Push the workflow to GitHub

Commit and push the workflow configuration:

```bash
git add .github/workflows/build-check.yml
git commit -m "ci: add React build check workflow"
git push
```

## 4. Create a Pull Request

Create a new feature branch:

```bash
git checkout -b feature/example
```

Make changes to your application, commit them, and push the branch:

```bash
git add .
git commit -m "feat: update application"
git push -u origin feature/example
```

Go to your GitHub repository and create a Pull Request targeting either `main` or `dev`.

GitHub Actions will automatically execute the workflow.

The workflow runs whenever a Pull Request is:

- Opened.
- Updated with new commits.
- Reopened.

## 5. Check the workflow results

You can view the execution results in the **Actions** tab of your GitHub repository or directly in the Pull Request.

| Status | Description |
|---|---|
| 🟡 Pending | The workflow is waiting or running. |
| 🟢 Success | The application builds successfully. |
| 🔴 Failure | An error occurred during workflow execution. |

If the workflow fails, open the execution logs to identify the error, fix it locally, and push a new commit.

GitHub Actions will automatically run the workflow again.

## 6. Configure branch protection

By default, a failed workflow does not prevent a Pull Request from being merged.

To enforce successful builds before merging:

1. Go to your repository on GitHub.
2. Navigate to **Settings → Branches**.
3. Create a branch protection rule.
4. Select the branch you want to protect.
5. Enable **Require status checks to pass before merging**.
6. Select the `build` status check.
7. Save the configuration.

Repeat this process for both `main` and `dev`.

Run the workflow at least once before configuring the required status check. Branch protection availability may depend on your GitHub plan and repository visibility.

## 7. Common errors

| Error | Possible solution |
|---|---|
| No supported lockfile found | Run `npm install` or `pnpm install` locally and commit the generated lockfile. |
| Multiple lockfiles detected | Keep only the lockfile associated with your package manager. |
| Missing pnpm version | Add the `packageManager` field to your `package.json`. |
| Missing build script | Verify the `scripts` section in your `package.json`. |
| Dependency installation failure | Make sure your `package.json` and lockfile are synchronized. |
| Build failure | Review the execution logs and fix the application errors. |
| Node.js compatibility issues | Verify that the configured Node.js version is compatible with your project. |
| Missing environment variables | Configure the environment variables required during the build. |

## Best practices

- Use a single package manager per project.
- Always commit your lockfile.
- Specify the pnpm version in your `package.json` when using pnpm.
- Use the same Node.js version locally and in CI.
- Never commit sensitive environment variables or credentials.
- Use the minimum permissions required by the workflow.
- Review failed workflow logs before merging changes.
- Keep GitHub Actions dependencies up to date.

## Conclusion

This workflow provides a simple introduction to Continuous Integration using GitHub Actions.

By automatically detecting the package manager and validating the application build on every Pull Request, developers can detect errors early and reduce the risk of introducing broken code into protected branches.

The same workflow can be used across multiple React projects without manually changing the package manager configuration.

**This workflow does not deploy the application. It only validates the build.**

## References

- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Checkout Action](https://github.com/actions/checkout)
- [GitHub Setup Node Action](https://github.com/actions/setup-node)
- [pnpm Setup Action](https://github.com/pnpm/action-setup)
- [npm ci Documentation](https://docs.npmjs.com/cli/commands/npm-ci)
- [pnpm Documentation](https://pnpm.io/)
- [Node.js](https://nodejs.org/)
