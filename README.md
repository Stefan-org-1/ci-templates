# ci-templates

Reusable workflows for Stefan-org-1.

| Template | What it does | Required |
|---|---|---|
| `scan.yml` | Mandatory scans: secrets, SonarQube (mock). Signs the result | Yes. The deploy gate requires its signed result |
| `nodejs-build.yml` | Builds and tests a Node.js repo, with team hooks | No. A team may build in its own job |
| `deploy.yml` | Deploys to an environment | Yes. The gate rejects deploys that do not use it |
| `nodejs-ci.yml` | `scan.yml` and `nodejs-build.yml` in one call | Convenience over those two |
| `release.yml` | Validates, then tags the branch head as release `X.Y.Z` | The paved road for releases |
| `node-build.yml` | Deprecated: build and scan in one template. Use `scan.yml` and `nodejs-build.yml` | |

```yaml
jobs:
  scan:
    uses: Stefan-org-1/ci-templates/.github/workflows/scan.yml@scan-v1
    permissions:
      contents: read
      id-token: write
      attestations: write

  build:
    uses: Stefan-org-1/ci-templates/.github/workflows/nodejs-build.yml@nodejs-build-v1
    with:
      node-version: "22"

  deploy:
    needs: [scan, build]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    uses: Stefan-org-1/ci-templates/.github/workflows/deploy.yml@deploy-v1
    with:
      environment: prod
```

Or scan and build in one call:

```yaml
jobs:
  ci:
    uses: Stefan-org-1/ci-templates/.github/workflows/nodejs-ci.yml@nodejs-ci-v1
    permissions:
      contents: read
      id-token: write
      attestations: write

  lint:
    runs-on: ubuntu-latest
    steps: [...]

  deploy:
    needs: [ci, lint]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    uses: Stefan-org-1/ci-templates/.github/workflows/deploy.yml@deploy-v1
    with:
      environment: prod
```

`nodejs-ci.yml` calls the other templates at their moving tags, so their fixes arrive without a release of it.

No template includes the deploy. A job inside a called template cannot wait for jobs in your file, so a bundled deploy could run before your own checks pass. Call `deploy.yml` yourself and list your jobs under `needs:`.

## Versions

Each template is versioned on its own, with tags named after it: `scan-v1`, `nodejs-build-v1`, `deploy-v1`, `nodejs-ci-v1`, `release-v1`.

- `<template>-v1` moves forward with every non-breaking release, so fixes reach every repo with no PRs.
- Breaking changes ship as the next major, such as `scan-v2`. Teams move to it with a PR, within a deprecation window.
- `<template>-v1.2.3` tags are fixed, for pinning.

`node-build.yml` still uses the older repo-wide tags `v1` and `v2`.

## scan.yml

Runs in its own job, on a fresh runner, from a fresh checkout of the exact commit. Signs `{"secret_scan": "pass", "sonarqube": "pass", "sonarqube_source": "mock"}` as an attestation over `<repo>@<sha>`. The calling job must grant `id-token: write` and `attestations: write`.

## nodejs-build.yml

Runs `npm ci`, `npm run build` and `npm test` as defined in your `package.json`. Custom steps go in `.github/actions/pre-build/action.yml` and `.github/actions/post-build/action.yml` in your repo.

## release.yml

Call it from a workflow your team runs by hand, from `main` or a `release/X.Y.z` branch:

```yaml
on:
  workflow_dispatch:
    inputs:
      version:
        description: Exact X.Y.Z. Empty picks the next one.
        default: ""
      bump:
        type: choice
        options: [patch, minor, major]
        default: patch

jobs:
  release:
    uses: Stefan-org-1/ci-templates/.github/workflows/release.yml@release-v1
    permissions:
      contents: write
      checks: read
      pull-requests: read
    with:
      version: ${{ inputs.version }}
      bump: ${{ inputs.bump }}
```

Before tagging, it checks:

- The branch is `main` or `release/X.Y.z`.
- The commit reached the branch through a PR that passed the `policy` check.
- The commit is not already released.
- The version is new and newer than the latest on its line. `release/2.3.z` only releases `2.3.N` patches. `minor` and `major` bumps come from `main`.

Release tags `X.Y.Z` cannot be moved or deleted, so a bad tag would use up its version for good. Pushing a tag by hand still works, but skips these checks. The deploy gate checks the release again either way.

## Customizing your pipeline

Your team owns `ci.yml`. Without asking platform you can:

- Add your own jobs: lint, e2e, nightly runs.
- Configure templates through their inputs.
- Add build steps through the `pre-build` and `post-build` hooks, or replace `nodejs-build.yml` with your own build.
- Choose what starts a deploy: a push to `main`, a manual run, or a tag.

### Why that cannot skip a mandatory check

| If a pipeline... | Then |
|---|---|
| Does not call `scan.yml` | No signed scan result exists. The gate rejects the merge and the deploy, where the repo's rule sets include `scanned` |
| Adds its own job that signs "scan passed" | The signature names your `ci.yml`, not `scan.yml`. Rejected |
| Tampers with the scan from a hook or its own build | Team code runs in other jobs. The scan runs in its own job, on a fresh runner, from a fresh checkout |
| Deploys without `deploy.yml` | Rejected (`central-deploy`) |
| Deploys from a job with no environment | It gets no prod credentials: those belong in the `prod` environment, released only after the gate approves |
| Removes the gate from its workflow | The gate is not in the workflow. A repo ruleset and the environment's protection rule require it, and teams cannot edit either |
| Uses an action not on the allow-list, or not pinned to a commit SHA | The org Actions policy blocks the run |

Any change under `.github/workflows/` also needs a sentinel's approval (`workflow-sentinel`).

A team's own build is not yet checked. Once deploys ship real images, the gate will need proof of where the image was built.
