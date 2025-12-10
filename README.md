# PR Auto-Update Test Repository

This repository tests an automated mechanism for updating PR branches when auto-merge is enabled and the base branch moves forward.

## Problem Statement

When using GitHub's auto-merge feature with required linear history and branch protection rules, PRs can get stuck in a queue. Each time a PR merges, all other PRs become "out of date" and require manual clicking of the "Update branch" button before they can merge.

For chains of PRs, this creates a manual babysitting process where you must monitor each PR and click "Update" repeatedly.

## Solution

This repo implements a GitHub Action that automatically updates PR branches when:
- The base branch (main) is updated
- The PR has auto-merge enabled
- The PR is behind the base branch
- The PR has no merge conflicts

## How It Works

1. When a PR merges to main, the `auto-update-prs.yml` workflow triggers
2. The workflow lists all open PRs targeting main
3. For each PR with auto-merge enabled:
   - Checks if the branch is behind main
   - Merges main into the PR branch (creating a merge commit in the feature branch)
   - Pushes the updated branch
4. The PR's CI runs again on the updated branch
5. Once CI passes, GitHub's auto-merge kicks in and merges the PR
6. This triggers the workflow again for remaining PRs

## Repository Configuration

- Branch protection on main:
  - Require status checks to pass (CI workflow)
  - Require branches to be up to date before merging
  - Require linear history
  - Only allow squash merging
- CI workflow: 10-second sleep to simulate real CI
- Auto-update workflow: Runs on every push to main

## Test Scenario

This repo contains test PRs to validate the mechanism:
- `pr1-auto` through `pr5-auto`: Five PRs with auto-merge enabled
- `pr6-no-automerge`: PR without auto-merge (should not be updated)
- `pr7-conflict`: PR with conflicts (should be skipped)

Expected behavior:
1. Manually merge pr1-auto
2. The workflow auto-updates pr2-auto through pr5-auto sequentially
3. Each PR merges after ~10 seconds (CI time)
4. Total cascade time: ~50-60 seconds
5. pr6 and pr7 are not updated

## Evidence

See [EVIDENCE.md](./EVIDENCE.md) for documented results of the test run.

## Validation

Run `scripts/validate_test.sh` to programmatically verify the expected behavior.

## Deployment to Production

Once validated here, this workflow can be adapted for use in production repositories like atlas-core.

### Important: GitHub App Required for Production

For production use in **organization repositories**, you must use a **GitHub App** instead of `GITHUB_TOKEN`. This is because GitHub does not trigger PR workflows when `github-actions[bot]` pushes to branches (security feature to prevent infinite loops).

**Why GitHub App (not PAT)?**
- ✅ Owned by the organization, not a single user
- ✅ Continues working even if users leave
- ✅ Fine-grained permissions (more secure)
- ✅ Tokens auto-expire after 1 hour
- ✅ Better audit trail

**Setup Steps for Organization Repos:**

1. **Create a GitHub App**:
   - Go to Organization Settings → Developer settings → GitHub Apps → New GitHub App
   - Name: `PR Auto-Update Bot` (or similar)
   - Homepage URL: Your organization's URL
   - Webhook: Uncheck "Active" (not needed)
   - Permissions:
     - Repository permissions → Contents: Read and write
     - Repository permissions → Pull requests: Read-only
   - Click "Create GitHub App"

2. **Generate and store credentials**:
   - In the app settings, click "Generate a private key" and download the file
   - Note the App ID (shown at the top)
   - Go to your repository Settings → Secrets and variables → Actions
   - Add variable `APP_ID` with your app ID
   - Add secret `APP_PRIVATE_KEY` with the contents of the downloaded .pem file

3. **Install the app**:
   - In the app settings, click "Install App"
   - Choose your organization
   - Select "All repositories" or specific repos

4. **Update the workflow**:
   Add this before your existing steps:
   ```yaml
   - uses: actions/create-github-app-token@v1
     id: app-token
     with:
       app-id: ${{ vars.APP_ID }}
       private-key: ${{ secrets.APP_PRIVATE_KEY }}
   ```
   Then change `GH_TOKEN: ${{ github.token }}` to `GH_TOKEN: ${{ steps.app-token.outputs.token }}`

**For Personal Repos (Alternative):**
You can use a Personal Access Token (PAT) instead. Generate a PAT with `repo` scope, add it as `AUTO_UPDATE_TOKEN` secret, and use it in the workflow.

Without this change, the mechanism will update PR branches but won't trigger CI, preventing auto-merge from completing.
