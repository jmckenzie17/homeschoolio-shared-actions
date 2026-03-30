# Data Model: Terraform Destroy Workflow

**Date**: 2026-03-30 | **Feature**: 002-terraform-destroy-workflow

This feature is a GitHub Actions workflow file. There is no application data model. The entities below describe the workflow's interface contract — inputs, outputs, and state objects that flow through it.

## Workflow Interface: `destroy.yml`

### Inputs (`workflow_call`)

| Name | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `target-environment` | `string` | yes | — | Environment name to destroy (`dev`, `staging`, `production`). Maps to `environments/<name>/` directory. |
| `confirm-destroy` | `boolean` | yes | — | Must be `true`. Guard against accidental invocation. |
| `opentofu-version` | `string` | no | `"1.6.2"` | OpenTofu CLI version to install. |
| `terragrunt-version` | `string` | no | `"0.56.3"` | Terragrunt CLI version to install. |

### Secrets (`workflow_call`)

| Name | Required | Description |
|------|----------|-------------|
| `azure-client-id` | yes | Azure service principal client ID (same as `apply.yml`). |
| `azure-tenant-id` | yes | Azure tenant ID (same as `apply.yml`). |
| `azure-subscription-id` | yes | Azure subscription ID (same as `apply.yml`). |
| `github-token` | yes | GitHub token for workflow identity. |
| `pg_admin_password` | no | PostgreSQL admin password, passed as `TF_VAR_pg_admin_password` if provided. |

### Outputs

| Name | Description |
|------|-------------|
| `destroyed-sha` | Commit SHA that was in effect when destroy ran. |

## State Transitions

```text
Trigger (workflow_call)
  └─> Guard check: confirm-destroy == "true"
        ├─ false → FAIL (step exits 1, job fails immediately)
        └─ true  → Azure Login
                    └─> Init (terragrunt run-all init)
                          └─> Destroy (terragrunt run-all destroy)
                                ├─ success → emit job summary → set destroyed-sha output → SUCCESS
                                └─ failure → emit partial job summary → FAIL (non-zero exit)
```

## Filesystem Artifacts

No persistent artifacts are written. The workflow reads from:
- `environments/<target-environment>/` — Terragrunt root modules
- Remote state backend (Azure Blob or equivalent, configured in `terragrunt.hcl`)
