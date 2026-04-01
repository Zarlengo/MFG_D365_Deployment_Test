# Reducing GitHub Actions Email Notifications

The deployment check workflow (`check-deployment-status.yml`) intentionally fails when PR labels are missing to block merging. This causes GitHub to send email notifications for each failure, which can be spammy during the deployment process.

## How to Disable These Emails

### Option 1: Disable Action Failure Emails Globally (Recommended)
1. Go to: https://github.com/settings/notifications
2. Under **"Actions"** section, uncheck:
   - ☐ **"Failed workflow runs"**
3. Click **"Save preferences"**

This will stop all Action failure emails across all repositories.

### Option 2: Disable Only for This Repository
1. Go to the repository: https://github.com/Zarlengo/MFG_D365_Deployment_Test
2. Click **"Watch"** dropdown in the top right
3. Select **"Custom"**
4. Under **"Actions"**, uncheck:
   - ☐ **Failing workflows only**
5. Click **"Apply"**

This will stop Action failure emails only for this repository.

### Option 3: Filter Emails with Email Rules
Create an email filter to automatically delete or archive emails from:
- **From**: `notifications@github.com`
- **Subject contains**: `[Zarlengo/MFG_D365_Deployment_Test]` AND `"Check Deployment Status"`

## Why the Check Must Fail

The deployment check **must** return exit code 1 (fail) when labels are missing because:
- GitHub branch protection rules only respect pass/fail status
- A passing check cannot block merging
- There is no "neutral" or "pending" state that blocks merges without failing

## Alternative: Use PR Comments Only

If you prefer to avoid the emails entirely, you can:
1. Remove the deployment check from branch protection rules
2. Rely solely on the automated PR comment to track deployment status
3. Manually verify all labels are present before merging

The comment will always show current status and update automatically when labels change.
