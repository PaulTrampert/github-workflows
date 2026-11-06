# GitHub Workflows

A collection of reusable GitHub Actions workflows for automating common CI/CD tasks.

## Overview

This repository contains reusable workflows that can be referenced from other repositories to standardize build, test, and deployment processes. These workflows are designed to be modular and configurable, allowing teams to maintain consistent automation across multiple projects.

## Available Workflows

### check-pr-title.yml

Validates that pull request titles follow a specific format including a change level indicator.

**Purpose:** Enforces PR title formatting to include semantic versioning change levels (MAJOR, MINOR, PATCH), ensuring consistent release management and changelog generation.

**Usage:**

```yaml
name: PR Title Check
on:
  pull_request:
    types: [opened, edited, synchronize, reopened]

jobs:
  check-title:
    uses: PaulTrampert/github-workflows/.github/workflows/check-pr-title.yml@main
    permissions:
      pull-requests: write
      contents: read
```

**Features:**
- Validates PR titles match the format: `(<change-level>) description`
- Supported change levels: `MAJOR`, `MINOR`, `PATCH`
- Adds sticky comments to PRs with formatting guidance
- Automatically updates comments when titles are corrected
- Fails the check if title format is invalid

**Example Valid Titles:**
- `(MAJOR) Introduce breaking change`
- `(MINOR) Add new feature`
- `(PATCH) Fix bug`

### dotnet-library.yml

A comprehensive workflow for building, testing, versioning, and releasing .NET libraries for NuGet.

**Purpose:** Automates the lifecycle of a .NET library from build to a NuGet-ready release, including smart change detection and version management.

**Usage:**

The shared workflow builds, tests, versions and creates the GitHub release, but does **not** push to
NuGet.org. NuGet.org [trusted publishing](https://learn.microsoft.com/en-us/nuget/nuget-org/trusted-publishing)
issues short-lived API keys to the calling repository's workflow, so the push happens in a job of your own:

```yaml
name: Main Build Pipeline

permissions:
  contents: write

on:
  pull_request:
    branches: ["main"]
  push:
    branches: ["main"]

jobs:
  dotnet-library:
    uses: PaulTrampert/github-workflows/.github/workflows/dotnet-library.yml@main
    with:
      project_name: "YourProjectName"
      main_branch: "refs/heads/main"  # Optional, defaults to refs/heads/main
      global_json: "./global.json"    # Optional, defaults to ./global.json

  publish-nuget:
    needs: dotnet-library
    if: ${{ needs.dotnet-library.outputs.released == 'true' }}
    runs-on: ubuntu-latest
    permissions:
      id-token: write  # Lets NuGet/login exchange a GitHub OIDC token for a temporary NuGet API key
    steps:
    - uses: actions/download-artifact@v8
      with:
        name: ${{ needs.dotnet-library.outputs.package_artifact }}
        path: package

    - name: NuGet login (OIDC → temporary API key)
      uses: NuGet/login@v1
      id: login
      with:
        user: ${{ secrets.NUGET_USER }}  # nuget.org profile name, not email

    # Pushing a .nupkg also pushes the matching .snupkg from the same folder.
    - name: Push to NuGet.org
      run: dotnet nuget push "package/*.nupkg" --api-key "${{ steps.login.outputs.NUGET_API_KEY }}" --source https://api.nuget.org/v3/index.json --skip-duplicate
```

On nuget.org, add a trusted publishing policy with the repository owner, repository name, and the file name
of **your** workflow (e.g. `dotnet-library.yml`, without the `.github/workflows/` path).

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `project_name` | Yes | - | The name of the .NET project to build and test. |
| `main_branch` | No | `refs/heads/main` | The main branch name |
| `global_json` | No | `./global.json` | Path to the global.json file for .NET SDK version |

**Outputs:**

| Output | Description |
|--------|-------------|
| `version` | The version that was built. |
| `released` | `'true'` when this run created a GitHub release, meaning the packages should be pushed to NuGet.org. Empty otherwise. |
| `package_artifact` | Name of the workflow run artifact containing the built `.nupkg` and `.snupkg` files. |

**Secrets:**

| Secret | Required | Description |
|--------|----------|-------------|
| `nuget_api_key` | No | **Deprecated.** When passed, the shared workflow pushes to NuGet.org with this key, as it used to. Omit it once your workflow pushes the package itself. |

**Expected Project Structure:**

This workflow expects your repository to follow this structure:
- `<project_name>.sln` - Solution file at the root of the repository
- `<project_name>/<project_name>.csproj` - The library project to be published

For example, if `project_name` is "MyLibrary":
```
repository-root/
├── MyLibrary.sln
├── MyLibrary/
│   └── MyLibrary.csproj
└── global.json
```

**Release Behavior:**

A release is only tagged (and `released` set to `'true'`) when:
1. The workflow runs on the main branch (as specified by `main_branch` input)
2. Changes are detected in the `<project_name>/` directory since the last git tag

This means updates to documentation, CI configuration, or other files outside the project directory will not trigger a new release.

**Features:**
- Change detection: Only publishes if changes in the target library are detected since the last tag
- Automatic versioning based on PR titles and git tags
- Builds and tests the project
- Exposes the built packages as a run artifact so the caller can push them to NuGet.org (only on main branch with changes)
- Creates a GitHub release whose notes list the commit messages since the previous tag, with `#123` references linked to the matching PR or issue
- Uses global.json for consistent .NET SDK versioning

**Workflow Jobs:**
1. **check-changes:** Detects if changes exist in `<project_name>/` since the last git tag
2. **version:** Calculates the next version number
3. **build-test:** Builds and tests the project
4. **publish:** Creates the GitHub release (conditional)

### dotnet-publish-docs.yml

Builds and publishes DocFX documentation to GitHub Pages.

**Purpose:** Automates the generation and deployment of API documentation using DocFX, making documentation accessible via GitHub Pages.

**Usage:**

```yaml
name: Publish Documentation
on:
  push:
    branches:
      - main

jobs:
  publish-docs:
    uses: PaulTrampert/github-workflows/.github/workflows/dotnet-publish-docs.yml@main
    with:
      global_json: "./global.json"  # Optional, defaults to ./global.json
    permissions:
      actions: read
      pages: write
      id-token: write
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `global_json` | No | `./global.json` | Path to the global.json file for .NET SDK version |

**Prerequisites:**
- A `docfx.json` configuration file in the root of your repository
- DocFX tool configured in your .NET tool manifest
- GitHub Pages enabled in repository settings

**Features:**
- Uses .NET SDK version from global.json
- Restores .NET tools and runs DocFX
- Uploads generated documentation to GitHub Pages
- Deploys with proper concurrency controls to prevent race conditions

**Workflow Steps:**
1. Checkout code
2. Setup .NET environment
3. Restore .NET tools
4. Run DocFX to generate documentation
5. Upload to GitHub Pages
6. Deploy to GitHub Pages

## Getting Started

To use these workflows in your repository:

1. Reference the workflow using the `uses` keyword
2. Specify the repository and path: `PaulTrampert/github-workflows/.github/workflows/<workflow-name>.yml@main`
3. Provide required inputs and secrets
4. Ensure proper permissions are granted

## Requirements

- GitHub Actions enabled in your repository
- For .NET workflows: A valid `global.json` file (or provide custom path)
- For publishing workflows: Appropriate repository secrets configured

## Contributing

When making changes to these workflows, ensure:
- Workflows remain backward compatible
- Documentation is updated accordingly
- Changes are tested in a real-world scenario before merging
