# homeschoolio-shared-actions Development Guidelines

Auto-generated from all feature plans. Last updated: 2026-03-26

## Active Technologies
- N/A (Git tags and GitHub Releases are the persistent artifacts) (001-semver-cicd-pipelines)

- YAML (GitHub Actions workflow syntax) — no application runtime + `googleapis/release-please-action@16a9c90856f42705d54a6fda1823352bdc62cf38` (v4.4.0), `actions/checkout` (v4, SHA pinned at implementation) (001-semver-cicd-pipelines)

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
- 001-semver-cicd-pipelines: Added YAML (GitHub Actions workflow syntax) — no application runtime

- 001-semver-cicd-pipelines: Added YAML (GitHub Actions workflow syntax) — no application runtime + `googleapis/release-please-action@16a9c90856f42705d54a6fda1823352bdc62cf38` (v4.4.0), `actions/checkout` (v4, SHA pinned at implementation)

<!-- MANUAL ADDITIONS START -->
<!-- MANUAL ADDITIONS END -->
