# Quickstart: Terraform Destroy Workflow

**Feature**: 002-terraform-destroy-workflow | **Date**: 2026-03-30

## What this delivers

Two new workflow files:

| File | Purpose |
|------|---------|
| `.github/workflows/destroy.yml` | Reusable workflow — tears down a target environment |
| `.github/workflows/test-destroy.yml` | CI test — lints and SHA-pin-checks `destroy.yml` on every PR |

## Files to create

```text
.github/workflows/destroy.yml        ← primary deliverable
.github/workflows/test-destroy.yml   ← CI gate
```

No existing files are modified.

## How `destroy.yml` works

1. **Guard step**: Fails immediately if `confirm-destroy` input is not `true`.
2. **Setup**: Checks out repo, installs OpenTofu + Terragrunt, restores provider cache.
3. **Azure Login**: Authenticates using the same secrets as `apply.yml`.
4. **Init**: Runs `terragrunt run-all init` in `environments/<target-environment>/`.
5. **Destroy**: Runs `terragrunt run-all destroy --terragrunt-non-interactive -auto-approve -no-color`.
6. **Summary**: Writes resource counts to `$GITHUB_STEP_SUMMARY`; emits `destroyed-sha` output.

## Consumer example

```yaml
# In your consumer repo's workflow:
jobs:
  teardown:
    uses: <org>/homeschoolio-shared-actions/.github/workflows/destroy.yml@v1
    with:
      target-environment: dev
      confirm-destroy: true
    secrets:
      azure-client-id: ${{ secrets.AZURE_CLIENT_ID }}
      azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      azure-subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      github-token: ${{ secrets.GITHUB_TOKEN }}
```

## Protecting production

Add GitHub Environment protection rules to your `production` environment (required reviewers). When the consumer workflow targets `production`, GitHub will pause and require approval before the destroy job starts — no workflow changes needed.

## How `test-destroy.yml` works

Runs on every PR to `main`:
- **lint job**: Runs `actionlint` against `destroy.yml`
- **sha-pinning job**: Greps `destroy.yml` for mutable tag references (e.g., `@v1`, `@main`) and fails if found

## Release notes for consumers

After this PR merges and a MINOR release is cut, consumers can reference:
```
uses: <org>/homeschoolio-shared-actions/.github/workflows/destroy.yml@v1
```

## Future enhancements (out of scope for this feature)

- E2E CI test against a real ephemeral `dev` environment
- Destroy-plan preview step (analogous to `plan.yml`) before executing
- Selective module destroy via `--terragrunt-include-dir` filter input
