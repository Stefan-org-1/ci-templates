# ci-templates

Reusable workflows for Stefan-org-1. Call them by major tag:

```yaml
jobs:
  build:
    uses: Stefan-org-1/ci-templates/.github/workflows/node-build.yml@v2
    permissions:
      contents: read
      id-token: write
      attestations: write
    with:
      node-version: "22"
```

`v1` and `v2` move forward with every non-breaking release. Breaking changes ship as a new major.

## v2

`node-build.yml@v2` runs the mandatory scan in its own job and signs the result as an attestation. The calling job must grant:

```yaml
permissions:
  contents: read
  id-token: write
  attestations: write
```

Custom build steps go in `.github/actions/pre-build/action.yml` and `.github/actions/post-build/action.yml` in your repo. They run in the `build` job only.
