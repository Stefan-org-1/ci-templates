# ci-templates

Reusable workflows for Stefan-org-1. Call them by major tag:

```yaml
jobs:
  build:
    uses: Stefan-org-1/ci-templates/.github/workflows/node-build.yml@v1
    with:
      node-version: "22"
```

`v1` moves forward with every non-breaking release. Breaking changes ship as a new major.
