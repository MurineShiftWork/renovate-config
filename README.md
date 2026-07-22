# renovate-config

Shared CI + dependency policy for the **MurineShiftWork** fleet, managed in one
place. Destined for `MurineShiftWork/renovate-config`. Two things live here:

1. **`default.json`** : the shared Renovate policy (weekly-batched, MSW-ecosystem-only).
2. **`.github/workflows/*.yml`** : reusable CI, versioning, release, and Renovate-runner
   workflows that each repo calls with a thin caller.

Pin callers to a tag (`@v1`) so a change here rolls out deliberately, not instantly.

---

## Renovate policy

Each repo keeps a one-line `renovate.json`:

```json
{ "$schema": "https://docs.renovatebot.com/renovate-schema.json",
  "extends": ["github>MurineShiftWork/renovate-config"] }
```

The policy: opens PRs only in the Monday-before-6am window, batches non-major MSW
updates into one auto-merged PR (`rangeStrategy: bump` raises `>=` floors), waits a
3-day stability window, keeps majors manual, and ignores everything outside the MSW
ecosystem (third-party libs, GitHub Actions, the Python runtime). Auth is a
`RENOVATE_TOKEN` PAT whose events trigger CI + versioning (the built-in token does not).

Per-repo caller `.github/workflows/renovate.yml`:

```yaml
name: Renovate
on:
  schedule:
    - cron: "0 2 * * *"   # nightly; the Monday window in the policy gates PR creation
  workflow_dispatch:
jobs:
  renovate:
    uses: MurineShiftWork/renovate-config/.github/workflows/renovate.yml@v1
    secrets:
      RENOVATE_TOKEN: ${{ secrets.RENOVATE_TOKEN }}
```

---

## Reusable CI / versioning / release

**`ci.yml`** (caller runs on push + PR to main):

```yaml
name: CI
on:
  push: { branches: [main], tags-ignore: ["v*"] }
  pull_request: { branches: [main] }
jobs:
  ci:
    uses: MurineShiftWork/renovate-config/.github/workflows/ci.yml@v1
    with:
      python-versions: '["3.12", "3.13"]'
      sync-extras: dev          # e.g. "dev test" for multiple extras
      # test-command: uv run pytest
      # lint: true
```

**`versioning.yml`** (caller runs on push to main):

```yaml
name: Versioning
on: { push: { branches: [main] } }
jobs:
  version:
    uses: MurineShiftWork/renovate-config/.github/workflows/versioning.yml@v1
```

**`release.yml`** (caller runs on tag push + dispatch):

```yaml
name: Release
on:
  push: { tags: ['v*'] }
  workflow_dispatch:
    inputs:
      tag: { description: 'Tag to publish', required: true }
jobs:
  release:
    uses: MurineShiftWork/renovate-config/.github/workflows/release.yml@v1
    with:
      pypi: true                # false / omit for internal GitHub-release-only repos
      tag: ${{ inputs.tag }}
```

---

## Prerequisites per repo

- `RENOVATE_TOKEN` secret (org secret with ALL visibility covers the fleet): a
  fine-grained PAT with Contents + Pull requests + Workflows write.
- For PyPI repos: trusted publishing configured on PyPI (no secret needed; OIDC).
- Repos that push directly to `main` from versioning must NOT have a required-status-check
  branch protection, or the bot push is blocked.

## Notes

- These reusable workflows are the current fleet pattern (single-trigger CI, the
  version->release split, Node-24 pins) lifted into `workflow_call`. Reconcile with the
  `templatepy` generator so new repos are stamped with callers, not full copies.
- Tag this repo (`v1`, moving) once adopted; bump the tag to roll changes fleet-wide.
- The Renovate policy is intentionally MSW-ecosystem-only; a repo can add its own
  `packageRules` in its local `renovate.json` (they merge on top of the preset).
