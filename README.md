# Centralized CI & Renovate Config (`bryanhitc/ci`)

Single source of truth for GitHub Actions reusable workflows and Renovate dependency-update presets across `bryanhitc/*` repositories.

## Contents

### Reusable Workflows (`.github/workflows/`)
- `rust.yml`: Runs `cargo fmt --check`, `cargo clippy`, `cargo test` (sharing one warm cache/build directory), and `cargo deny check advisories` (using the repo's `deny.toml` if present, or a sensible fallback that fails on real advisories and direct unmaintained crates without failing on transitive unmaintained crates).
- `ci.yml`: Self-validates the JSON and Renovate presets in this repository.

### Renovate Presets
- `base.json`: Shared policy **without** auto-merge (ideal for infra/GitOps repos like `homelab`):
  - 7-day cooldown (`minimumReleaseAge`, synced to Cargo/package managers via `minimumReleaseAgePackageManagerSync`) on routine updates, scheduled Mondays before 5 AM.
  - 12-hour soak (`minimumReleaseAge`) on security fixes (`vulnerabilityAlerts` + `osvVulnerabilityAlerts`), scheduled at any time and grouped together into one PR so multi-advisory repos don't deadlock against `cargo deny check advisories`.
  - Dependency Dashboard enabled.
  - Assigns `bryanhitc` with `assignAutomerge: false`, so PRs requiring manual review (e.g. `major` updates or `base.json` repos) notify immediately while auto-merge PRs stay silent unless CI/tests fail.
- `default.json`: Extends `base.json` and adds PR-based auto-merge (`automergeType: "pr"`, `ignoreTests: false`) for code repositories with automated tests:
  - Groups `minor`/`patch`/`pin`/`digest` updates into a single weekly PR and auto-merges when CI passes.
  - Auto-merges the grouped security PR once CI passes.
  - Runs monthly `lockFileMaintenance` with auto-merge to keep transitive dependencies (e.g. `Cargo.lock`) patched.
  - Leaves `major` updates open for manual review.

## Onboarding a Repository

Add two small files to any target repository (and delete any legacy daily `audit.yml` cron workflows):

### 1. `.github/workflows/ci.yml`
```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  rust:
    uses: bryanhitc/ci/.github/workflows/rust.yml@main
```

Optional `with:` inputs for `rust.yml`:
- `rust-version` (default: `stable`)
- `working-directory` (default: `.`)
- `cargo-args` (default: `--all-targets`)
- `clippy-args` (default: `-D warnings`)
- `run-fmt`, `run-clippy`, `run-test`, `run-audit` (all default: `true`)

### 2. `renovate.json`

For code repos (with PR auto-merge):
```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>bryanhitc/ci"
  ]
}
```

For infra/GitOps repos (cooldowns & dashboard only, no auto-merge):
```json
{
  "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": [
    "github>bryanhitc/ci:base"
  ]
}
```
