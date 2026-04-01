# Deployment Status Check - Timeout Handling

## Timeout Configuration

### Wait Job Timeouts
Both `wait-for-deploy` and `wait-for-test` jobs have a **7-hour timeout** (420 minutes):
- ADO pipeline timeout: 6 hours
- GH Action timeout: 7 hours
- **Buffer**: 1 hour to see ADO timeout failures

This ensures GitHub Actions won't timeout before ADO, allowing proper failure detection and status check creation.

## Manual Status Check Trigger

If workflows timeout or you need to re-check status, use the `/check` command:

### Via PR Comment
Comment on the PR with:
```
/check
```

This will immediately run the deployment status check and create/update the commit status.

## When to Use /check

Use the `/check` command when:
- ✅ Workflow times out before creating status check
- ✅ Need to re-verify status after manual label changes
- ✅ Want to check status without re-running deploy/test
- ✅ Debugging deployment requirements
- ✅ Verifying all labels are present before requesting review

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
- ✅ **Solution**: Comment `/check` on PR
- Check will see current label state

### Scenario 3: Network issues or API failures
- `gh run watch` might lose connection
- GH Action may fail to detect completion
- ✅ **Solution**: Comment `/check` on PR
- Creates status based on current state

## Best Practices

1. **Monitor long-running workflows** - Check Actions tab for timeout warnings
2. **Use /check proactively** - If you see workflows taking >5 hours
3. **Re-check after manual fixes** - After manually adding labels, use `/check`
4. **Branch protection** - Require "Verify Deployment Status" check before merge

## Troubleshooting

**Q: Status check not appearing on PR?**
- Check if workflow completed (Actions tab)
- Comment `/check` on the PR
- Verify all required labels are present

**Q: Workflow timed out, what now?**
- Comment `/check` on the PR to create status
- If labels are present, status will be success
- If labels missing, deploy/test manually then use `/check`

**Q: Can I increase timeout beyond 7 hours?**
- Yes, edit `timeout-minutes` in workflow files
- Max GitHub Actions timeout: 6 hours (free) or 72 hours (paid)
- ADO timeout is separate (configured in ADO pipeline)

## Available Commands

- `/deploy <env>` - Deploy and test in environment (adt, qat, crptrn)
- `/test <env>` - Test in environment (adt, qat, crptrn)
- `/check` - Check deployment status and create/update commit status
