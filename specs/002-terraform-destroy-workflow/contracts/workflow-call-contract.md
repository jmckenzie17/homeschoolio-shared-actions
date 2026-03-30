# Workflow Call Contract: `destroy.yml`

**Type**: GitHub Actions Reusable Workflow (`workflow_call`)
**File**: `.github/workflows/destroy.yml`
**Stability**: v0 (pre-release, on feature branch)

## Consumer Usage

```yaml
jobs:
  destroy-dev:
    uses: <org>/homeschoolio-shared-actions/.github/workflows/destroy.yml@v1
    with:
      target-environment: dev
      confirm-destroy: true
      # opentofu-version: "1.6.2"   # optional, defaults shown
      # terragrunt-version: "0.56.3" # optional, defaults shown
    secrets:
      azure-client-id: ${{ secrets.AZURE_CLIENT_ID }}
      azure-tenant-id: ${{ secrets.AZURE_TENANT_ID }}
      azure-subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
      github-token: ${{ secrets.GITHUB_TOKEN }}
      pg_admin_password: ${{ secrets.PG_ADMIN_PASSWORD }}  # optional
```

## Input Contract

| Input | Type | Constraint | Breaking if removed/renamed |
|-------|------|------------|----------------------------|
| `target-environment` | string | Non-empty; must match a directory under `environments/` | Yes |
| `confirm-destroy` | boolean | Must be `true` at call site or job fails immediately | Yes |
| `opentofu-version` | string | Semver string; defaults to `"1.6.2"` | No (has default) |
| `terragrunt-version` | string | Semver string; defaults to `"0.56.3"` | No (has default) |

## Output Contract

| Output | Type | Guarantee |
|--------|------|-----------|
| `destroyed-sha` | string | Set on success; empty on failure |

## Secret Contract

| Secret | Required | Purpose |
|--------|----------|---------|
| `azure-client-id` | yes | Azure OIDC / service principal auth |
| `azure-tenant-id` | yes | Azure OIDC / service principal auth |
| `azure-subscription-id` | yes | Azure subscription scope |
| `github-token` | yes | Workflow identity |
| `pg_admin_password` | no | Passed as `TF_VAR_pg_admin_password` if present |

## Failure Modes

| Condition | Behavior |
|-----------|----------|
| `confirm-destroy` is not `true` | Job fails at guard step before any infrastructure is touched |
| `target-environment` directory missing | `terragrunt run-all init` fails; job exits non-zero |
| Azure auth failure | `azure/login` step fails; job exits non-zero |
| Partial destroy failure | `run-all destroy` exits non-zero; job summary shows partial results |
| All resources already destroyed | `run-all destroy` exits 0 (no-op); job succeeds |

## Breaking Change Policy

Per Constitution Principle II, any change to required inputs, removal of outputs, or change in failure semantics requires a MAJOR version bump and advance notice to consumers.
