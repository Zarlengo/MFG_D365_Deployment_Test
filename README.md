# D365 Deployment Testing Repository

This is a test repository for validating D365 deployment workflows.

## Workflows

See [.github/workflows/README.md](.github/workflows/README.md) for complete documentation.

## Quick Test

1. Create a test PR
2. Comment \/deploy adt\ to trigger deployment
3. Check the Actions tab for results

## Full test suite

### 1. Normal flow

1. Create a change in Metadata folder on a branch
2. Create a PR with that branch
   - Expect PR blocked from merging
3. Add action deploy:adt
   - Expect label `deploy:adt` and `tested:adt` to be added
   - Expect PR blocked from merging
4. Add action deploy:qat
   - Expect label `deploy:qat` and `tested:qat` to be added
   - Expect PR blocked from merging
5. Add action deploy:crptrn
   - Expect label `deploy:crptrn` and `tested:crptrn` to be added
   - Expect PR to allow merging
6. Merge PR
7. Expect main branch > pipeline triggered

### 2. Label restrictions

These tests should show label's being automatically removed and a comment added about the restriction

1. Create a change in Metadata folder on a branch
2. Create a PR with that branch
3. Add a manual label to the PR of any text
4. Add a label of any of the following:
   - `deployed:adt`
   - `deployed:qat`
   - `deployed:crptrn`
   - `tested:adt`
   - `tested:qat`
   - `tested:crptrn`

### 3. Documentation flow

1. Create a change not in Metadata folder
2. Create a PR with that branch
   - Expect PR to allow merging

## Files

- **Metadata/**: Test folder for deployment requirement detection
