# Auto-Update PRs with Auto-Merge

A GitHub Actions workflow that automatically keeps pull requests up-to-date when auto-merge is enabled and the base branch changes.

## The Problem

When your repository uses:
- **Required linear history** (no merge commits in main/develop)
- **Required status checks before merge** (CI must pass)
- **Auto-merge on pull requests** (to automatically merge when ready)

You encounter this workflow issue:

1. PR #1 merges to `main` → Makes PR #2, #3, #4 out-of-date
2. You must manually click "Update branch" on PR #2 → Wait for CI
3. PR #2 merges → Makes PR #3, #4 out-of-date again
4. Repeat for every PR in the chain

This is tedious and time-consuming for teams working with stacked PRs or release trains.

## The Solution

This workflow automates the entire process:

1. Detects when commits are pushed to your base branch (e.g., `main`, `develop`)
2. Finds all open PRs targeting that branch with auto-merge enabled
3. Automatically merges the base branch into those PR branches
4. Triggers CI on the updated branches
5. Auto-merge completes once CI passes

**Result**: A chain of 5 PRs merges automatically in ~2-3 minutes with zero manual intervention.

## How It Works

```
Developer enables auto-merge on PR #1-5
    ↓
Merge PR #1 to main (manual)
    ↓
Workflow triggers → Updates PR #2-5 branches → CI runs
    ↓
PR #2 CI passes → Auto-merges → Workflow triggers again
    ↓
Workflow updates PR #3-5 → CI runs
    ↓
PR #3 merges... cascade continues
    ↓
All PRs merged automatically
```

## Key Features

- ✅ **Fully automated**: No manual "Update branch" clicks needed
- ✅ **Smart detection**: Only updates PRs with auto-merge enabled
- ✅ **Conflict handling**: Gracefully skips PRs with merge conflicts
- ✅ **Multi-branch support**: Configure for `main`, `develop`, `staging`, etc.
- ✅ **Dynamic adaptation**: Single source of truth for base branch configuration
- ✅ **Proper CI triggering**: Uses GitHub App tokens to ensure CI runs on updated branches

## Comparison with GitHub's Merge Queue

### GitHub's Merge Queue

GitHub offers a built-in [Merge Queue](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/configuring-pull-request-merges/managing-a-merge-queue) feature that provides a more robust solution for managing PR merges.

**How it works:**
- PRs are added to a queue when approved and checks pass
- GitHub creates temporary "merge group" branches that combine queued changes
- Tests run on these merge groups to catch integration issues before merging
- If tests pass, changes are merged to the base branch atomically
- If tests fail, the problematic PR is removed and the queue continues

**Benefits:**
- Catches integration bugs that only appear when multiple PRs are combined
- Prevents "broken main" scenarios where individually passing PRs break when merged together
- Managed by GitHub with automatic retry logic
- Built-in merge conflict handling

**Availability:** ⚠️ **Requires GitHub Enterprise Cloud** - not available on Free, Team, or GitHub Enterprise Server plans

### This Workflow

This workflow provides automatic PR updates for repositories without Enterprise Cloud access:

**How it works:**
- Watches for pushes to your base branch
- Automatically merges base branch changes into PR branches with auto-merge enabled
- Triggers CI on updated branches
- PRs merge individually when CI passes

**When to use:**
- You don't have access to GitHub Enterprise Cloud
- You want individual PR merges with preserved history
- You need customizable behavior (specific branches, merge strategies, etc.)
- Your CI catches integration issues reliably

**Trade-offs:**
- Does not test PRs together before merging (integration issues may slip through)
- Each PR merges sequentially rather than being validated as a batch
- Requires manual workflow setup and maintenance

---

## Setup Instructions

### Prerequisites

- **Organization admin access** (to create GitHub App) *or* Personal Access Token
- **Repository admin access** (to add secrets/variables)

### Option 1: GitHub App (Recommended for Organizations)

GitHub Apps are recommended for organization repositories because they:
- Are organization-owned (not tied to individual users)
- Survive employee turnover
- Have fine-grained permissions
- Use auto-expiring tokens (1 hour)

#### Step 1: Create GitHub App

**For an organization:**
1. Go to: `https://github.com/organizations/YOUR-ORG/settings/apps`
2. Click **"New GitHub App"**

**For a personal account (testing):**
1. Go to: `https://github.com/settings/apps`
2. Click **"New GitHub App"**

#### Step 2: Configure App Settings

**Basic settings:**
- **Name**: `PR Auto-Update Bot` (must be unique, adjust if taken)
- **Homepage URL**: Your organization's GitHub URL
- **Webhook**: **Uncheck "Active"** (not needed)

**Repository permissions:**
- **Contents**: `Read and write`
- **Pull requests**: `Read-only`

**Installation:**
- **Where can this GitHub App be installed?**: `Only on this account`

Click **"Create GitHub App"**

#### Step 3: Save Credentials

After creating the app:

1. **Note the App ID** at the top of the page:
   ```
   App ID: 1234567
   ```

2. **Generate a private key**:
   - Scroll to "Private keys" section
   - Click "Generate a private key"
   - A `.pem` file will download
   - **Save this file securely** (can only download once)

#### Step 4: Install App on Repository

1. In app settings, click **"Install App"** (left sidebar)
2. Click **"Install"** next to your organization/account
3. Select repositories (all or specific ones)
4. Click **"Install"**

#### Step 5: Add Credentials to Repository

Go to: `https://github.com/YOUR-ORG/YOUR-REPO/settings/secrets/actions`

**Add variable:**
1. Click "Variables" tab → "New repository variable"
2. Name: `APP_ID`
3. Value: Your App ID from Step 3 (e.g., `1234567`)

**Add secret:**
1. Click "Secrets" tab → "New repository secret"
2. Name: `APP_PRIVATE_KEY`
3. Value: Entire contents of `.pem` file:
   ```
   -----BEGIN RSA PRIVATE KEY-----
   (all the key data)
   -----END RSA PRIVATE KEY-----
   ```

#### Step 6: Add Workflow File

Copy `.github/workflows/auto-update-prs.yml` from this repository to yours.

Edit the trigger to match your base branch:
```yaml
on:
  push:
    branches:
      - main  # Change to your base branch (develop, staging, etc.)
```

Commit and push. The workflow is now active!

---

### Option 2: Personal Access Token (For Personal Repos)

If you don't want to create a GitHub App (e.g., for personal repositories):

#### Step 1: Generate PAT

1. Go to: `https://github.com/settings/tokens?type=beta`
2. Click "Generate new token"
3. Give it a name: "PR Auto-Update"
4. Select repositories (all or specific)
5. Permissions:
   - **Contents**: Read and write
   - **Pull requests**: Read-only
6. Click "Generate token" and copy it

#### Step 2: Add PAT to Repository

1. Go to: `https://github.com/YOUR-USERNAME/YOUR-REPO/settings/secrets/actions`
2. Click "New repository secret"
3. Name: `PAT_TOKEN`
4. Value: Your PAT from Step 1

#### Step 3: Modify Workflow

In `.github/workflows/auto-update-prs.yml`, replace the "Generate GitHub App token" step with:

```yaml
- name: Checkout repository
  uses: actions/checkout@v4
  with:
    fetch-depth: 0
    token: ${{ secrets.PAT_TOKEN }}
```

And update the environment variable:
```yaml
- name: Update PRs with auto-merge enabled
  env:
    GH_TOKEN: ${{ secrets.PAT_TOKEN }}  # Changed from app token
    BASE_BRANCH: ${{ github.ref_name }}
```

---

## Usage

### Enable Auto-Merge on PRs

**Via GitHub CLI:**
```bash
gh pr merge <PR_NUMBER> --auto --squash
```

**Via GitHub UI:**
1. Open the pull request
2. Click "Enable auto-merge" button
3. Select your merge method (squash/merge/rebase)

### The Cascade Effect

When you have multiple PRs with auto-merge enabled:

```
PR #1: Merge to main (manual)
   ↓ (workflow triggers)
PR #2-5: Branches updated automatically
   ↓ (CI runs)
PR #2: CI passes → Auto-merges
   ↓ (workflow triggers)
PR #3-5: Branches updated automatically
   ↓ (CI runs)
PR #3: CI passes → Auto-merges
   ↓ (continues...)
```

**Performance**: Typically ~30-60 seconds per PR (depending on CI duration)

### Testing the Setup

1. **Create test PRs**:
   ```bash
   git checkout -b test-pr-1
   echo "test" > test1.txt
   git add . && git commit -m "Test 1"
   git push -u origin test-pr-1
   gh pr create --base main --title "Test PR 1"
   gh pr merge $(gh pr view --json number -q .number) --auto --squash
   ```

2. **Repeat for 2-3 more PRs**

3. **Trigger the cascade**:
   - Merge the first PR, or
   - Push a commit directly to `main`

4. **Verify**:
   - Check Actions tab for workflow runs
   - Watch PRs update and merge automatically
   - Confirm CI triggers on updated branches

---

## Configuration

### Multiple Base Branches

To trigger on multiple branches (e.g., `main`, `develop`, `staging`):

```yaml
on:
  push:
    branches:
      - main
      - develop
      - staging
```

The workflow automatically adapts to whichever branch triggered it - no other changes needed!

### Pattern Matching

Use glob patterns for release branches:

```yaml
on:
  push:
    branches:
      - main
      - 'release/**'
```

### Adjusting CI Workflow

The included `ci.yml` workflow is a simple example with a 10-second sleep. Replace it with your actual CI:

```yaml
name: CI
on:
  pull_request:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Run tests
        run: |
          npm install
          npm test
```

---

## Troubleshooting

### "Resource not accessible by integration"

**Cause**: GitHub App lacks required permissions.

**Fix**:
1. Go to app settings → Permissions & events
2. Verify **Contents** = "Read and write"
3. Reinstall app on repository

### "Bad credentials"

**Cause**: Private key or PAT is incorrect.

**Fix**:
1. Verify entire key copied (including `-----BEGIN/END-----` markers)
2. Check for extra spaces or line breaks
3. Regenerate key/token if needed

### CI not triggering on updated PRs

**Cause**: Using `GITHUB_TOKEN` instead of app token or PAT.

**Fix**: The workflow in this repo already uses the correct token. Verify:
- For GitHub App: Token generated via `actions/create-github-app-token@v1`
- For PAT: Token set in checkout action and GH_TOKEN env var

### Workflow not running at all

**Cause**: Missing credentials or workflow disabled.

**Fix**:
1. Verify secrets/variables exist in repository settings
2. Check workflow is enabled: `gh workflow list`
3. Verify push to base branch triggers the workflow

### PRs not auto-merging

**Cause**: Auto-merge not enabled on PR, or CI failing.

**Fix**:
1. Verify auto-merge is enabled: `gh pr view <NUMBER> --json autoMergeRequest`
2. Check CI status: `gh pr view <NUMBER> --json statusCheckRollup`
3. Ensure branch protection requires status checks

---

## Security Considerations

- **GitHub App tokens**: Auto-expire after 1 hour (more secure than long-lived PATs)
- **Minimal permissions**: Only Contents (write) and Pull requests (read)
- **Secrets encryption**: All credentials stored as encrypted GitHub secrets
- **Audit trail**: All actions logged and attributed to the app/bot
