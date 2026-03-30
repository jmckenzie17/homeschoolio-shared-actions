# Feature Specification: Terraform Destroy Workflow

**Feature Branch**: `002-terraform-destroy-workflow`
**Created**: 2026-03-30
**Status**: Draft
**Input**: User description: "create github workflows to tear down terraform infrastructure that was created using the apply yaml"

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Safe Targeted Destroy (Priority: P1)

An operator needs to tear down infrastructure for a specific environment (e.g., `dev`) that was previously provisioned by the Apply workflow. They trigger the destroy workflow manually, specifying the target environment, and the workflow tears down all resources cleanly while requiring explicit confirmation to prevent accidental destruction.

**Why this priority**: Destroying infrastructure is high-risk and irreversible. The primary value is a reliable, audited destroy path that mirrors the apply workflow's interface so operators don't have to resort to ad-hoc CLI commands.

**Independent Test**: Can be fully tested by triggering the destroy workflow against a `dev` environment with known resources and verifying all resources are removed without manual intervention after the confirmation step.

**Acceptance Scenarios**:

1. **Given** an environment with provisioned infrastructure, **When** an operator triggers the destroy workflow with `target-environment: dev`, **Then** all resources in that environment are destroyed and the run exits successfully.
2. **Given** the destroy workflow is triggered, **When** the workflow runs, **Then** it requires explicit manual approval before executing the destroy (e.g., via GitHub environment protection rules or a required acknowledgment input).
3. **Given** the destroy workflow completes, **When** reviewing the run, **Then** the job summary lists the count of resources destroyed.

---

### User Story 2 - Destroy Mirrors Apply Configuration (Priority: P2)

An operator expects the destroy workflow to accept the same version and authentication inputs as the Apply workflow, so consumer repositories can reuse the same secrets and tool versions without additional configuration.

**Why this priority**: Consistency with the Apply workflow reduces integration friction for consumers and ensures the destroy operation uses the same tool versions that created the infrastructure.

**Independent Test**: Can be fully tested by calling the destroy workflow with the same inputs/secrets used for apply and confirming it initializes and authenticates without errors.

**Acceptance Scenarios**:

1. **Given** a consumer workflow that already passes `opentofu-version`, `terragrunt-version`, and Azure secrets to the Apply workflow, **When** they wire the same values to the destroy workflow, **Then** no additional secrets or inputs are required.
2. **Given** the `opentofu-version` or `terragrunt-version` inputs are omitted, **When** the workflow runs, **Then** it uses the same default versions as the Apply workflow.

---

### User Story 3 - Destroy Failure Surfaces Clearly (Priority: P3)

An operator triggers a destroy that partially fails (e.g., a resource has a deletion lock or a permission issue). They need to know exactly which resources failed to be destroyed so they can intervene manually.

**Why this priority**: Silent partial destroys leave orphaned resources and unexpected costs. Operators must be able to identify what remains.

**Independent Test**: Can be tested by simulating a destroy against a root where one resource has a deletion guard, then confirming the workflow fails with a clear error and summary of unremoved resources.

**Acceptance Scenarios**:

1. **Given** a destroy operation where one or more resources cannot be deleted, **When** the workflow completes, **Then** it exits with a non-zero status and the job summary identifies the failed resources.
2. **Given** a destroy that fails mid-run, **When** the operator views the job summary, **Then** they see which roots succeeded and which failed.

---

### Edge Cases

- What happens when the specified environment directory does not exist?
- How does the workflow handle an environment that has already been fully destroyed (empty or missing state)?
- What happens when Azure authentication fails mid-destroy?
- How does the workflow behave if `pg_admin_password` is required for state cleanup but not provided?

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The destroy workflow MUST accept a `target-environment` input identifying which environment to destroy (e.g., `dev`, `staging`, `production`).
- **FR-002**: The destroy workflow MUST accept the same `opentofu-version`, `terragrunt-version`, Azure credential secrets, and optional `pg_admin_password` secret as the Apply workflow.
- **FR-003**: The destroy workflow MUST require explicit acknowledgment or manual approval before executing the destroy operation to prevent accidental runs.
- **FR-004**: The destroy workflow MUST initialize the Terragrunt/OpenTofu state backend before running destroy, using the same init approach as the Apply workflow.
- **FR-005**: The destroy workflow MUST execute a non-interactive destroy against all modules in the target environment directory.
- **FR-006**: The destroy workflow MUST produce a job summary listing the count of resources destroyed per environment root.
- **FR-007**: The destroy workflow MUST exit with a failure status if any resource fails to be destroyed.
- **FR-008**: The destroy workflow MUST be a reusable workflow (`workflow_call`) so consumer repositories can call it from their own workflows.
- **FR-009**: The destroy workflow MUST use SHA-pinned external actions, consistent with the project's security policy for all other shared workflows.

### Key Entities

- **Target Environment**: The named environment (e.g., `dev`, `staging`, `production`) whose infrastructure will be destroyed; maps to a directory under `environments/`.
- **Destroy Run**: A single execution of the workflow against one environment; produces a job summary and an exit status.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: Operators can trigger a complete environment teardown with a single workflow dispatch, with no manual CLI commands required.
- **SC-002**: 100% of resources in the target environment are destroyed in a successful run, leaving no orphaned resources.
- **SC-003**: A destroy cannot proceed without an explicit human approval or acknowledgment step, preventing accidental teardown.
- **SC-004**: Any partial destroy failure is surfaced in the job summary within the same run, with no need to inspect raw logs.
- **SC-005**: Consumer repositories can integrate the destroy workflow by reusing existing Apply workflow inputs/secrets with zero additional configuration.

## Assumptions

- The destroy workflow targets the same `environments/<name>/` directory structure used by the Apply workflow.
- Azure service principal / OIDC authentication (same secrets as Apply) is sufficient to authorize resource deletion.
- OpenTofu/Terragrunt state is stored remotely and accessible during the destroy run; no local state files are required.
- GitHub environment protection rules (required reviewers on the `production` environment) are the primary safeguard for production; the workflow enforces a confirmation input for all environments.
- `pg_admin_password` is optional on destroy (same as apply) since not all resources require it at teardown.
- Dependency-aware destroy ordering is delegated to Terragrunt's built-in `run-all destroy` behavior.
- The workflow will be added as a new file in `.github/workflows/` alongside `apply.yml`, not modifying existing workflows.
