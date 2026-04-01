# Deployment Status Check - Timeout Handling

## Timeout Configuration

### Wait Job Timeouts
Both `wait-for-deploy` and `wait-for-test` jobs have a **7-hour timeout** (420 minutes):
- ADO pipeline timeout: 6 hours
- GH Action timeout: 7 hours
- **Buffer**: 1 hour to see ADO timeout failures

This ensures GitHub Actions won't timeout before ADO, allowing proper failure detection and status check creation.

## Manual Status Check Trigger

If workflows timeout or you need to re-check status, use the manual trigger:

### Via GitHub UI
1. Go to **Actions** tab
2. Select **"Manual Deployment Status Check"** workflow
3. Click **"Run workflow"**
4. Enter **PR number**
5. Click **"Run workflow"**

### Via GitHub CLI
```bash
gh workflow run manual-deployment-check.yml \
  --ref main \
  -f pr_number="5"
```

## When to Use Manual Trigger

Use the manual status check when:
- ✅ Workflow times out before creating status check
- ✅ Need to re-verify status after manual label changes
- ✅ Want to check status without re-running deploy/test
- ✅ Debugging deployment requirements

## What It Does

The manual check performs the same validation as automatic:
1. Checks for metadata changes (`git diff` on PR)
2. Validates all 4 labels present:
   - `deployed:qat`
   - `tested:qat`
   - `deployed:crptrn`
   - `tested:crptrn`
3. Creates/updates commit status on PR
4. Context: "Verify Deployment Status"
5. State: success (all present) or failure (missing labels)

## Timeout Scenarios

### Scenario 1: ADO times out at 6 hours
- GH Action continues running (7-hour timeout)
- ADO reports failure to GH workflow
- GH Action detects failure via `gh run watch`
- Status check created: **failure**
- ✅ Works correctly

### Scenario 2: GH Action times out at 7 hours
- ADO still running (rare, but possible)
- GH Action stops watching
- No status check created automatically
- ✅ **Solution**: Run manual status check
- Manual check will see current label state

### Scenario 3: Network issues or API failures
- `gh run watch` might lose connection
- GH Action may fail to detect completion
- ✅ **Solution**: Run manual status check
- Creates status based on current state

## Best Practices

1. **Monitor long-running workflows** - Check Actions tab for timeout warnings
2. **Use manual trigger proactively** - If you see workflows taking >5 hours
3. **Re-run after fixes** - After manually adding labels, trigger manual check
4. **Branch protection** - Require "Verify Deployment Status" check before merge

## Troubleshooting

**Q: Status check not appearing on PR?**
- Check if workflow completed (Actions tab)
- Run manual status check workflow
- Verify PR number is correct

**Q: Workflow timed out, what now?**
- Run manual status check with PR number
- If labels are present, status will be success
- If labels missing, deploy/test manually then re-check

**Q: Can I increase timeout beyond 7 hours?**
- Yes, edit `timeout-minutes` in workflow files
- Max GitHub Actions timeout: 6 hours (free) or 72 hours (paid)
- ADO timeout is separate (configured in ADO pipeline)
