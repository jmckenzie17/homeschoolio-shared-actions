# homeschoolio-shared-actions Development Guidelines

Auto-generated from all feature plans. Last updated: 2026-03-30

## Active Technologies
- N/A (Git tags and GitHub Releases are the persistent artifacts) (001-semver-cicd-pipelines)
- YAML (GitHub Actions workflow syntax) — no application runtime + `opentofu/setup-opentofu` (SHA-pinned), `azure/login` (SHA-pinned), `actions/checkout` (SHA-pinned), `actions/cache` (SHA-pinned), Terragrunt CLI (binary download) (002-terraform-destroy-workflow)
- N/A — remote Terraform state (Azure Blob / equivalent); no local state (002-terraform-destroy-workflow)

- YAML (GitHub Actions workflow syntax) — no application runtime + `cycjimmy/semantic-release-action@b12c8f6015dc215fe37bc154d4ad456dd3833c90` (v6.0.0), `actions/checkout@34e114876b0b11c390a56381ad16ebd13914f8d5` (v4.3.1) (001-semver-cicd-pipelines)

## Project Structure

```text
.github/
└── workflows/
    ├── semver-release.yml          # Reusable workflow (workflow_call) — primary deliverable
    ├── release.yml                 # Self-release caller (this repo versions itself via local path)
    └── test-semver-release.yml     # CI test workflow (runs on every PR to main)

docs/
└── semver-release/
    ├── README.md                   # Consumer-facing docs
    └── examples/
        ├── consumer-workflow.yml
        └── consumer-workflow-with-deploy.yml
```

## Commands

# No build/test commands — YAML-only project. Use actionlint for local linting:
# brew install actionlint && actionlint .github/workflows/semver-release.yml

## Code Style

YAML (GitHub Actions workflow syntax) — no application runtime: Follow standard conventions

## Recent Changes
- 002-terraform-destroy-workflow: Added YAML (GitHub Actions workflow syntax) — no application runtime + `opentofu/setup-opentofu` (SHA-pinned), `azure/login` (SHA-pinned), `actions/checkout` (SHA-pinned), `actions/cache` (SHA-pinned), Terragrunt CLI (binary download)
- 001-semver-cicd-pipelines: Added YAML (GitHub Actions workflow syntax) — no application runtime
- 001-semver-cicd-pipelines: Added YAML (GitHub Actions workflow syntax) — no application runtime


<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
