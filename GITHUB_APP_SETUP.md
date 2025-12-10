# GitHub App Setup Guide for PR Auto-Update

This guide walks you through creating and configuring a GitHub App to enable the PR auto-update mechanism in your organization.

## Why GitHub App (Not Personal Access Token)?

For organization repositories, GitHub Apps are the recommended solution:

- ✅ **Organization-owned** - Not tied to any individual user account
- ✅ **Survives turnover** - Works even if the creator leaves the organization
- ✅ **Fine-grained permissions** - Only grant exactly what's needed
- ✅ **Auto-expiring tokens** - Tokens expire after 1 hour (more secure)
- ✅ **Better audit trail** - Clear attribution in logs and actions
- ✅ **No user license required** - Doesn't consume a seat

## Prerequisites

- Organization admin access (to create GitHub Apps)
- Repository admin access (to add secrets and variables)

## Step 1: Create the GitHub App

### 1.1 Navigate to GitHub Apps Settings

For an **organization**:
- Go to: `https://github.com/organizations/YOUR-ORG-NAME/settings/apps`
- Click **"New GitHub App"**

For a **personal account** (testing):
- Go to: `https://github.com/settings/apps`
- Click **"New GitHub App"**

### 1.2 Configure Basic Settings

Fill in the following fields:

**GitHub App name**: `PR Auto-Update Bot`
- Must be unique across all of GitHub
- Tip: Add your org name if "PR Auto-Update Bot" is taken (e.g., "Arena PR Auto-Update Bot")

**Homepage URL**: Your organization's website or GitHub URL
- Example: `https://github.com/arena-technologies`

**Webhook**:
- **Uncheck "Active"** - We don't need webhooks for this use case

**Webhook URL**: Leave blank (inactive webhooks don't need a URL)

### 1.3 Set Repository Permissions

Scroll to the **"Repository permissions"** section and configure:

| Permission | Access Level | Why Needed |
|------------|-------------|------------|
| **Contents** | `Read and write` | To push merged changes to PR branches |
| **Pull requests** | `Read-only` | To list and query PR status |

Leave all other permissions at **"No access"**.

### 1.4 Set Installation Scope

**Where can this GitHub App be installed?**
- Choose: **"Only on this account"**
- This restricts the app to your organization only

### 1.5 Create the App

Click **"Create GitHub App"** at the bottom of the page.

## Step 2: Generate and Save Credentials

After creation, you'll be on the app's settings page.

### 2.1 Note the App ID

At the top of the page, you'll see:
```
App ID: 1234567
```

**Copy this number** - you'll need it in Step 4.

### 2.2 Generate a Private Key

Scroll down to the **"Private keys"** section:

1. Click **"Generate a private key"**
2. A `.pem` file will download automatically
3. **Save this file securely** - you'll need it in Step 4
4. **Important**: This is the only time you can download this key. If lost, you'll need to generate a new one.

## Step 3: Install the App on Repositories

### 3.1 Navigate to Installation

In the left sidebar of the app settings page:
- Click **"Install App"**

### 3.2 Install on Your Organization

1. Click **"Install"** next to your organization name
2. Choose installation scope:
   - **"All repositories"** (easier, app works everywhere)
   - **"Only select repositories"** (more secure, choose specific repos like atlas-core)
3. Click **"Install"**

You'll be redirected to a confirmation page showing which repositories the app can access.

## Step 4: Add Credentials to Your Repository

For each repository where you want to use the auto-update mechanism:

### 4.1 Add App ID as a Variable

1. Go to repository **Settings → Secrets and variables → Actions**
2. Click the **"Variables"** tab
3. Click **"New repository variable"**
4. Name: `APP_ID`
5. Value: Your App ID from Step 2.1 (e.g., `1234567`)
6. Click **"Add variable"**

### 4.2 Add Private Key as a Secret

1. Stay in **Settings → Secrets and variables → Actions**
2. Click the **"Secrets"** tab
3. Click **"New repository secret"**
4. Name: `APP_PRIVATE_KEY`
5. Value: Open the `.pem` file from Step 2.2 in a text editor and copy the **entire contents**, including the `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----` lines
6. Click **"Add secret"**

**Example of what to copy:**
```
-----BEGIN RSA PRIVATE KEY-----
MIIEpAIBAAKCAQEA1234567890abcdef...
(many more lines)
...xyz789
-----END RSA PRIVATE KEY-----
```

## Step 5: Update Your Workflow

### 5.1 Add GitHub App Token Generation Step

In your workflow file (e.g., `.github/workflows/auto-update-prs.yml`), add this step **before** any git operations:

```yaml
jobs:
  update-prs:
    runs-on: ubuntu-latest
    steps:
      # Generate token from GitHub App
      - name: Generate token
        id: app-token
        uses: actions/create-github-app-token@v1
        with:
          app-id: ${{ vars.APP_ID }}
          private-key: ${{ secrets.APP_PRIVATE_KEY }}

      # Use the token for checkout
      - name: Checkout repository
        uses: actions/checkout@v4
        with:
          token: ${{ steps.app-token.outputs.token }}
          fetch-depth: 0

      # ... rest of your workflow
```

### 5.2 Use GitHub App Token Instead of GITHUB_TOKEN

Replace all instances of `${{ github.token }}` with `${{ steps.app-token.outputs.token }}`:

```yaml
      - name: Update PRs with auto-merge enabled
        env:
          # Use GitHub App token instead of GITHUB_TOKEN
          GH_TOKEN: ${{ steps.app-token.outputs.token }}
        run: |
          # Your PR update logic here
```

### 5.3 Complete Example

See [`.github/workflows/auto-update-prs-with-app.yml`](https://github.com/chrismbryant/pr-auto-update-test/blob/main/.github/workflows/auto-update-prs-with-app.yml) in this repository for a complete working example.

## Step 6: Test the Setup

### 6.1 Create Test PRs

1. Create 2-3 test PRs from different branches
2. Enable auto-merge on all of them:
   ```bash
   gh pr merge PR_NUMBER --auto --squash
   ```

### 6.2 Trigger the Cascade

Manually merge one PR to main, or push a commit to main. This will trigger the auto-update workflow.

### 6.3 Verify Success

Check that:
1. ✅ The auto-update workflow runs successfully
2. ✅ PR branches are updated with main
3. ✅ CI runs on the updated PR branches (this is the key validation!)
4. ✅ PRs auto-merge after CI passes

## Troubleshooting

### Issue: "Resource not accessible by integration"

**Cause**: App doesn't have the correct permissions.

**Solution**:
1. Go to app settings
2. Check that **Contents** has `Read and write` permission
3. Reinstall the app on your repository

### Issue: "Bad credentials"

**Cause**: Private key is incorrect or corrupted.

**Solution**:
1. Verify the entire private key was copied, including begin/end markers
2. Check for no extra spaces or line breaks
3. If issues persist, generate a new private key and update the secret

### Issue: CI still not triggering on updated PRs

**Cause**: Still using `github.token` instead of the app token.

**Solution**:
1. Verify all instances of `github.token` are replaced with `steps.app-token.outputs.token`
2. Check that the checkout action uses the app token: `token: ${{ steps.app-token.outputs.token }}`

### Issue: "App not found"

**Cause**: App ID is incorrect or app was deleted.

**Solution**:
1. Go to app settings and verify the App ID
2. Update the `APP_ID` variable with the correct number

## Security Best Practices

1. **Rotate private keys periodically** (every 6-12 months)
2. **Use separate apps** for different environments (dev/staging/prod)
3. **Review app permissions** regularly and remove unused permissions
4. **Monitor app activity** in organization audit logs
5. **Limit repository access** - only install on repos that need it

## Maintenance

### Rotating the Private Key

1. Go to app settings → Private keys
2. Click "Generate a private key" to create a new one
3. Update the `APP_PRIVATE_KEY` secret in all repositories
4. Revoke the old key after confirming the new one works

### Updating Permissions

If you need to add or change permissions:
1. Go to app settings → Permissions & events
2. Update the required permissions
3. Repositories will be notified of the permission change
4. Org admins must approve the new permissions

### Uninstalling

To remove the app from a repository:
1. Go to repository Settings → GitHub Apps
2. Find your app and click "Configure"
3. Scroll down and click "Uninstall"

## Additional Resources

- [GitHub Apps Documentation](https://docs.github.com/en/apps)
- [actions/create-github-app-token](https://github.com/actions/create-github-app-token)
- [About authentication with a GitHub App](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/about-authentication-with-a-github-app)

## Support

If you encounter issues not covered in this guide:
1. Check the [Actions logs](https://github.com/YOUR-ORG/YOUR-REPO/actions) for detailed error messages
2. Review the [test repository](https://github.com/chrismbryant/pr-auto-update-test) for a working example
3. Consult GitHub's [troubleshooting guide](https://docs.github.com/en/apps/creating-github-apps/guides/troubleshooting-github-app-installations)
