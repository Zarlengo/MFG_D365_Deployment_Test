# CRITICAL FIX: Deploy/Test Correct Branch

## Issue Discovered
The workflows were deploying and testing code from the `main` branch instead of the actual PR branch, giving false confidence that PR changes were validated.

## What Was Wrong

### Before Fix (BROKEN):
```
PR #5 on branch "feature/new-changes"
  ↓
User comments: /deploy qat
  ↓
Workflow triggered with --ref main
  ↓
No checkout step (or would checkout main)
  ↓
Deploys/tests main branch code ❌
  ↓
Labels added to PR #5 ✅ (FALSE!)
  ↓
PR merged thinking it's tested ❌
```

**Result**: PRs could break production even with all labels present!

### After Fix (CORRECT):
```
PR #5 on branch "feature/new-changes"
  ↓
User comments: /deploy qat
  ↓
pr-comment-triggers gets PR branch name: "feature/new-changes"
  ↓
Passes pr_branch to workflow
  ↓
Workflow checks out "feature/new-changes" ✅
  ↓
Deploys/tests actual PR code ✅
  ↓
Labels added only if PR code succeeds ✅
```

**Result**: Labels actually represent that THIS PR's code is deployed/tested!

## Changes Made

### 1. pr-comment-triggers.yml
**Both `trigger-deploy` and `trigger-test` jobs now:**
```bash
# Get PR head branch name
PR_BRANCH=$(gh pr view "$PR_NUMBER" --json headRefName --jq '.headRefName')

# Pass to workflow
gh workflow run ... -f pr_branch="$PR_BRANCH"
```

### 2. ado-pipeline-deploy.yml
**Added:**
- Input parameter: `pr_branch` (required)
- Checkout step: `uses: actions/checkout@v4` with `ref: ${{ github.event.inputs.pr_branch }}`

### 3. ado-pipeline-test.yml
**Added:**
- Input parameter: `pr_branch` (required)
- Checkout step: `uses: actions/checkout@v4` with `ref: ${{ github.event.inputs.pr_branch }}`

## Impact

### Security Impact: HIGH
- **Before**: Any PR could get labels without actually testing PR code
- **After**: Labels only added when actual PR code is deployed/tested

### Trust Impact: CRITICAL
- **Before**: False sense of security - "tests passed" meant nothing
- **After**: Labels accurately represent that THIS PR works in those environments

## How to Verify It's Working

### Check workflow logs:
1. Trigger `/deploy qat` on a PR
2. Go to Actions → Deploy to Environment
3. Look for: "Checkout PR branch" step
4. Verify it checks out the PR branch name (not main)

### Check what code is deployed:
If your deploy simulation showed file changes:
```bash
# In the workflow, after checkout:
git branch --show-current  # Should show PR branch
git log -1                  # Should show PR's latest commit
```

## Why This Happened

The original implementation:
1. Used `--ref main` to trigger workflows (correct - workflow files from main)
2. But forgot to checkout the PR branch in the triggered workflows
3. Workflows ran with no code checked out (just simulated)
4. Even if they had code, it would've been main

## Lesson Learned

When triggering workflows via `workflow_dispatch`:
- The `--ref` parameter controls which **workflow file version** runs
- It does NOT control what code the workflow checks out
- Always explicitly checkout the branch you want to deploy/test

## Testing Recommendations

After this fix:
1. Test a PR with actual code changes
2. Verify deploy/test workflows see those changes
3. Break something in PR and verify tests fail
4. Confirm labels only added when PR code succeeds
