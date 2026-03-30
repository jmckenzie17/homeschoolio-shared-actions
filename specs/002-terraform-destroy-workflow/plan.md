# Implementation Plan: Terraform Destroy Workflow

**Branch**: `002-terraform-destroy-workflow` | **Date**: 2026-03-30 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/002-terraform-destroy-workflow/spec.md`

## Summary

Add a reusable `destroy.yml` GitHub Actions workflow (`workflow_call`) that tears down Terragrunt-managed infrastructure for a specified environment. The workflow mirrors the `apply.yml` interface (same inputs, same secrets, same tool-setup steps) and adds a mandatory confirmation gate to prevent accidental execution. A companion `test-destroy.yml` CI workflow validates the new file on every PR to main.

## Technical Context

**Language/Version**: YAML (GitHub Actions workflow syntax) — no application runtime
**Primary Dependencies**: `opentofu/setup-opentofu` (SHA-pinned), `azure/login` (SHA-pinned), `actions/checkout` (SHA-pinned), `actions/cache` (SHA-pinned), Terragrunt CLI (binary download)
**Storage**: N/A — remote Terraform state (Azure Blob / equivalent); no local state
**Testing**: `actionlint` lint + `rhysd/actionlint` GitHub Action; SHA-pin verification script (mirrors pattern in `test-semver-release.yml`)
**Target Platform**: GitHub Actions runner (`ubuntu-latest`)
**Project Type**: Reusable GitHub Actions workflow (library)
**Performance Goals**: N/A — destroy run time is dominated by provider API calls, not workflow overhead
**Constraints**: All external action references MUST be SHA-pinned (Constitution III); workflow MUST be `workflow_call` only (Constitution I); confirmation gate MUST exist before destroy executes
**Scale/Scope**: One environment per workflow invocation; matches existing per-environment apply pattern

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Status | Notes |
|-----------|--------|-------|
| I. Reusable-Action-First | ✅ PASS | `destroy.yml` is a `workflow_call` reusable workflow, same as `apply.yml` |
| II. Semantic Versioning | ✅ PASS | New workflow is a backward-compatible addition → MINOR bump after merge |
| III. Trusted & Auditable Third-Party Actions | ✅ PASS | All external actions will be SHA-pinned; SHA-pinning CI check included in test workflow |
| IV. Minimal Surface Area | ✅ PASS | Inputs match `apply.yml` exactly; no speculative parameters added |
| V. Tested Before Release | ✅ PASS | `test-destroy.yml` CI workflow exercises lint + SHA-pin check on every PR |
| Security: Least-Privilege Permissions | ✅ PASS | Permissions declared at job level: `id-token: write`, `contents: read` only |
| Security: No Secret Logging | ✅ PASS | Azure secrets passed as env vars, not echoed; same pattern as `apply.yml` |

**Post-Phase-1 re-check**: No design changes introduced violations.

## Project Structure

### Documentation (this feature)

```text
specs/002-terraform-destroy-workflow/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
└── tasks.md             # Phase 2 output (/speckit.tasks — not created here)
```

### Source Code (repository root)

```text
.github/
└── workflows/
    ├── destroy.yml              # NEW — reusable destroy workflow (primary deliverable)
    └── test-destroy.yml         # NEW — CI test workflow (PR gate for destroy.yml)

docs/
└── semver-release/              # existing; no changes
```

**Structure Decision**: Single-file addition to `.github/workflows/` — no new directories required. Follows the identical convention established by `apply.yml` and tested by `test-semver-release.yml`.

## Complexity Tracking

No Constitution violations to justify.
