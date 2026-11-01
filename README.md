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

## Customizing your pipeline

Your team owns `ci.yml`. Without asking platform you can:

- Add your own jobs: lint, e2e, nightly runs.
- Configure templates through their inputs.
- Add build steps through the `pre-build` and `post-build` hooks.
- Choose what starts a deploy: a push to `main`, a manual run, or a tag.

### Why that cannot skip a mandatory check

| If a pipeline... | Then |
|---|---|
| Drops the template's scan job | No signed scan result exists. The gate rejects the merge and the deploy, where the repo's rule sets include `scanned` |
| Adds its own job that signs "scan passed" | The signature names your `ci.yml`, not this repo's template. Rejected |
| Tampers with the scan from a hook | Hooks run in the `build` job. The scan runs in its own job, on a fresh runner, from a fresh checkout |
| Deploys without `deploy.yml` | Rejected (`central-deploy`) |
| Deploys from a job with no environment | It gets no prod credentials: those belong in the `prod` environment, released only after the gate approves |
| Removes the gate from its workflow | The gate is not in the workflow. A repo ruleset and the environment's protection rule require it, and teams cannot edit either |
| Uses an action not on the allow-list, or not pinned to a commit SHA | The org Actions policy blocks the run |

Any change under `.github/workflows/` also needs a sentinel's approval (`workflow-sentinel`).
