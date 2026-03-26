# homeschoolio-shared-actions

Shared GitHub Actions reusable workflows for the homeschoolio ecosystem.

## Available Workflows

### [semver-release](docs/semver-release/README.md)

Automates semantic versioning using [Conventional Commits](https://www.conventionalcommits.org/).
Parses commit messages on merge to `main`, creates the appropriate SemVer tag and
GitHub Release, and maintains a moving major version pointer (e.g. `v1`).

**Quick adoption** — add this to your repo's `.github/workflows/release.yml`:

```yaml
name: Release
on:
  push:
    branches: [main]
jobs:
  release:
    uses: homeschoolio/homeschoolio-shared-actions/.github/workflows/semver-release.yml@v1
    secrets: inherit
```

See [docs/semver-release/README.md](docs/semver-release/README.md) for full
documentation including inputs, outputs, and examples.

---

## Versioning & SHA-Pinning Policy

This repository follows the same semver conventions it implements:

- **Consumers**: Pin to `@v1` for automatic patch/minor updates, or `@v1.2.3` for
  a locked exact version.
- **Contributors**: All external actions used inside workflows MUST be pinned to a
  full commit SHA (never a mutable tag like `@v4` or `@main`). The CI workflow
  enforces this on every PR.

### Why SHA pinning?

Mutable tags (e.g. `@v4`) can be silently updated by the upstream author — including
maliciously. A commit SHA is immutable: once pinned, the exact code that ran is
reproducible forever. This is a hard requirement per the homeschoolio Actions
Constitution (Principle III).
