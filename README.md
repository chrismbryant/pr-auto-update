# PR Auto-Update Mechanism

Automatically updates PR branches when auto-merge is enabled and the base branch moves forward, eliminating manual "Update branch" clicking.

## Problem

When using GitHub's auto-merge with required linear history, PRs become "out of date" each time another PR merges. You must manually click "Update branch" repeatedly to keep the merge queue moving.

## Solution

A GitHub Action that automatically:
1. Detects PRs with auto-merge enabled
2. Updates them when the base branch changes
3. Triggers CI on the updated branches
4. Allows auto-merge to complete the process

## Architecture

### Components

**GitHub App (Organization-level)**
- Created once in your organization settings
- Owns the authentication credentials
- Provides tokens that can trigger workflows
- Not tied to any individual user account

**Workflow File (Repository-level)**
- Added to `.github/workflows/auto-update-prs.yml` in **each repository** where you want auto-updates
- Uses credentials from the GitHub App
- Runs when the base branch receives new commits
- Updates PR branches that have auto-merge enabled

### Relationship

```
GitHub Organization
└── GitHub App (PR Auto-Update Bot)
    ├── Installed on → Repository A
    │   └── .github/workflows/auto-update-prs.yml (uses app credentials)
    ├── Installed on → Repository B
    │   └── .github/workflows/auto-update-prs.yml (uses app credentials)
    └── Installed on → Repository C
        └── .github/workflows/auto-update-prs.yml (uses app credentials)
```

The **GitHub App lives at the organization level** and provides authentication. The **workflow file lives in each repository** and uses the app's credentials.

## Setup

### Step 1: Create GitHub App (Organization Admin - Once)

See [GITHUB_APP_SETUP.md](./GITHUB_APP_SETUP.md) for detailed instructions.

Quick summary:
1. Create app in organization settings
2. Set permissions: `contents: write`, `pull-requests: read`
3. Generate private key
4. Install app on desired repositories

### Step 2: Add Workflow to Each Repository (Per Repository)

#### 2.1 Add Credentials

In **each repository** where you want auto-updates:

1. Go to Settings → Secrets and variables → Actions
2. Add **variable**: `APP_ID` = your app ID
3. Add **secret**: `APP_PRIVATE_KEY` = contents of .pem file

#### 2.2 Add Workflow File

Create `.github/workflows/auto-update-prs.yml`:

```yaml
name: Auto-update PRs with auto-merge enabled

on:
  push:
    branches:
      - main  # Change this to match your base branch

permissions:
  contents: write
  pull-requests: read

jobs:
  update-prs:
    runs-on: ubuntu-latest
    steps:
      - name: Generate GitHub App token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ vars.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}

      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ steps.app-token.outputs.token }}

      - name: Configure git
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"

      - name: Update PRs with auto-merge enabled
        env:
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
        run: |
          # See full script in .github/workflows/auto-update-prs.yml
```

See [`.github/workflows/auto-update-prs.yml`](./.github/workflows/auto-update-prs.yml) for the complete workflow.

## Configuration

### Specifying Base Branches

Edit the `on.push.branches` section to match your workflow:

**Single base branch:**
```yaml
on:
  push:
    branches:
      - main
```

**Multiple base branches:**
```yaml
on:
  push:
    branches:
      - main
      - develop
      - staging
```

**Pattern matching:**
```yaml
on:
  push:
    branches:
      - main
      - 'release/**'
```

The workflow will check for PRs targeting whichever branch received the push. To specify a specific target branch, modify line 53 in the workflow:
```bash
pr_numbers=$(gh pr list --base main --state open --json number --jq '.[].number')
```

Change `--base main` to `--base develop` or the appropriate branch name.

## Usage

### Enable Auto-merge on PRs

For PRs you want to auto-update and auto-merge:

```bash
gh pr merge PR_NUMBER --auto --squash
```

Or via GitHub UI: Click "Enable auto-merge" on the PR page.

### The Cascade

1. PR with auto-merge enabled passes CI
2. GitHub auto-merges the PR
3. Workflow detects the base branch update
4. Workflow updates remaining PRs with auto-merge enabled
5. CI runs on updated PRs
6. Process repeats until all PRs merge

## Testing

This repository contains validation evidence:
- [EVIDENCE.md](./EVIDENCE.md) - Complete test results
- Validated with 13 PRs across multiple test scenarios
- Average merge time: ~31 seconds per PR

## Requirements

- GitHub Actions enabled
- Branch protection with:
  - Required status checks
  - Require branches to be up to date before merging
  - Required linear history (optional, but recommended)
- GitHub App with proper permissions
- Repository secrets configured

## Troubleshooting

See [GITHUB_APP_SETUP.md](./GITHUB_APP_SETUP.md) for:
- Common issues and solutions
- Security best practices
- Maintenance procedures

## Why GitHub App Instead of PAT?

| Feature | GitHub App | Personal Access Token |
|---------|------------|----------------------|
| Organization-owned | ✅ Yes | ❌ No (tied to user) |
| Survives employee turnover | ✅ Yes | ❌ No |
| Fine-grained permissions | ✅ Yes | ⚠️ Limited |
| Token expiration | ✅ 1 hour | ❌ Manual rotation |
| Audit trail | ✅ Clear | ⚠️ Shows as user |

For organization repositories, GitHub Apps are the recommended approach.
