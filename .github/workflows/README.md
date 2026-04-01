# GitHub Actions Workflows

## Overview

This repository contains four workflows for managing environment deployments and testing:

1. **protect-deployment-labels.yml** - Automatically removes manually-added status labels
2. **pr-deployment-triggers.yml** - Handles PR comments to trigger deployments
3. **deploy-test-environments.yml** - Deploys and tests code in ADT, QAT, or CRPTRN environments
4. **check-deployment-status.yml** - Enforces deployment testing requirements before merge

## Workflows

### 1. Protect Deployment Labels

**Trigger:** Automatically runs when any label is added to a PR

**Purpose:** Ensures deployment and testing status labels can only be added by automation, preventing manual tampering.

**Protected Label Patterns:**
- `deployed:*` - Status labels (added after successful deployment)
- `tested:*` - Status labels (added after successful testing)

**Behavior:**
- If a user manually adds a status label, it will be automatically removed
- A comment is added to the PR explaining why the label was removed
- Only `github-actions[bot]` can add these labels

### 2. PR Deployment Triggers

**Trigger:** PR comments only

**Purpose:** Provides a simple way to trigger deployments directly from a PR via comments.

**Available Commands:**
- `/deploy adt` - Deploy and test in ADT
- `/deploy qat` - Deploy and test in QAT
- `/deploy crptrn` - Deploy and test in CRPTRN
- `/test adt` - Re-run tests in ADT (skip deployment)
- `/test qat` - Re-run tests in QAT (skip deployment)
- `/test crptrn` - Re-run tests in CRPTRN (skip deployment)

**Behavior:**
- Adds 🚀 reaction to comment when command is recognized
- Posts a confirmation comment with workflow link
- Workflow is triggered on the default branch with PR number passed as input
- Tracks who triggered the deployment for audit purposes

### 3. Deploy & Test Environments

**Trigger:** Called by PR deployment triggers workflow or manual via `workflow_dispatch`

**Inputs:**
- `pr_number` (optional) - PR number to apply success labels to
- `environment` - Which environment to deploy/test (adt, qat, or crptrn)
- `mode` - Action to perform:
  - `deploy` - Deploy and then test
  - `test` - Test only (skip deployment)
- `triggered_by` (optional) - Username for audit trail

**Flow (Deploy Mode):**

```
┌─────────────┐
│   Deploy    │ (30 seconds, 50% random failure)
└──────┬──────┘
       │ ✅ Success
       ▼
┌─────────────┐
│    Test     │ (30 seconds, 50% random failure)
└──────┬──────┘
       │ ✅ Success
       ▼
┌─────────────┐     ┌──────────────────┐
│  Add Labels │────→│ deployed:{env}   │
│             │     │ tested:{env}     │
└─────────────┘     └──────────────────┘
```

**Flow (Test Mode):**

```
┌─────────────┐
│    Test     │ (30 seconds, 50% random failure)
└──────┬──────┘
       │ ✅ Success
       ▼
┌─────────────┐     ┌──────────────────┐
│  Add Labels │────→│ tested:{env}     │
└─────────────┘     └──────────────────┘
```

**Failure Behavior:**
- ❌ If deployment fails → testing is skipped
- ❌ If testing fails → no labels are applied
- 📊 A summary is always generated showing success/failure/skipped status

**Labels Applied:**
- **Deploy mode:** Both `deployed:{env}` and `tested:{env}` labels (when both succeed)
- **Test mode:** Only `tested:{env}` label (when tests succeed)
- Labels trigger a PR comment with success notification and workflow run link

### 4. Check Deployment Status

**Trigger:** Automatically runs on PR open, sync, label changes

**Purpose:** Enforces deployment testing requirements before allowing merge.

**Rules:**
1. **If PR modifies `/Metadata` folder:**
   - ✅ Requires `tested:qat` label
   - ✅ Requires `tested:crptrn` label
   - ❌ Blocks merge if labels are missing

2. **If PR does NOT modify `/Metadata` folder:**
   - ✅ Bypass deployment requirements
   - ✅ Can merge immediately

**Behavior:**
- Posts a status comment on PRs requiring deployment testing
- Updates comment when labels are applied
- Provides instructions on how to trigger deployments
- Used as a required status check for branch protection

## Usage Examples

### Deploy to QAT

1. Go to your PR
2. Add a comment: `/deploy qat`
3. Comment gets a 🚀 reaction
4. Workflow runs and deploys/tests in QAT
5. On success, labels `deployed:qat` and `tested:qat` are applied

### Re-run Tests in CRPTRN

1. Go to your PR
2. Add a comment: `/test crptrn`
3. Tests run without re-deploying
4. On success, label `tested:crptrn` is applied (or updated)

### Deploy to ADT

1. Go to your PR
2. Add a comment: `/deploy adt`
3. Workflow runs and deploys/tests in ADT
4. On success, labels `deployed:adt` and `tested:adt` are applied

### Check if PR Can Be Merged

1. Go to your PR
2. Check the **Checks** tab
3. Look for **Check Deployment Status**
4. If it fails, follow the instructions to deploy to required environments

### Full Deployment Workflow

For a PR that modifies `/Metadata`:

```bash
# In PR comments:
/deploy adt      # Optional: Test in ADT first
/deploy qat      # Required: Deploy and test in QAT
/deploy crptrn   # Required: Deploy and test in CRPTRN

# Once both tested:qat and tested:crptrn labels are present:
# ✅ PR can be merged
```

## Testing Failure Scenarios

The workflows include 50% random failure rates to test failure pathways:

**Scenario 1: Deployment Fails (Deploy Mode)**
```
Comment: /deploy qat
❌ QAT Deploy (FAILED - random)
⏭️ QAT Test (SKIPPED - deployment failed)
⏭️ Labels (SKIPPED - deployment failed)
Result: No labels applied, try again
```

**Scenario 2: Testing Fails (Deploy Mode)**
```
Comment: /deploy adt
✅ ADT Deploy (SUCCESS)
❌ ADT Test (FAILED - random)
⏭️ Labels (SKIPPED - testing failed)
Result: No labels applied, use /test adt to retry tests
```

**Scenario 3: Test-Only Fails**
```
Comment: /test crptrn
❌ CRPTRN Test (FAILED - random)
⏭️ Labels (SKIPPED)
Result: No labels applied, try /test crptrn again
```

**Scenario 4: Full Success (Deploy Mode)**
```
Comment: /deploy qat
✅ QAT Deploy (SUCCESS)
✅ QAT Test (SUCCESS)
✅ Labels Applied: deployed:qat, tested:qat
```

**Scenario 5: Full Success (Test Mode)**
```
Comment: /test adt
✅ ADT Test (SUCCESS)
✅ Label Applied: tested:adt
Note: deployed:adt not added (test-only mode)
```

## Label Protection Testing

Try to manually add a deployment label to a PR:

1. Go to a PR
2. Add label `deployed:adt` manually
3. Within seconds, the label will be removed
4. A comment will appear explaining why

## Branch Protection Setup

To enforce deployment requirements before merge:

1. Go to **Settings** → **Branches** → **Branch protection rules**
2. Add rule for your main branch (e.g., `main`)
3. Enable **Require status checks to pass before merging**
4. Search for and select: **Verify Deployment Status**
5. Save changes

Now PRs that modify `/Metadata` cannot be merged without `tested:qat` and `tested:crptrn` labels.

## Notes

- **Timing:** Each stage (deploy/test) takes 30 seconds
- **Random Failures:** Each stage has a 50% chance of failure for testing purposes
- **One Environment at a Time:** Each deployment/test runs independently
- **Test-Only Mode:** Use `/test {env}` to re-run tests without redeploying
- **Metadata Bypass:** PRs without `/Metadata` changes don't need deployment testing
- **No ADO Integration:** This is a skeleton workflow for testing label-driven deployments
- **Audit Trail:** `triggered_by` parameter tracks who started each deployment
