# PR Auto-Update Test Repository
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
