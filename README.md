# ci-actions

Shared CI building blocks for the image fleet.

## `smoke-test`

Starts a built image as a container, polls its HTTP health endpoint until it
responds, and fails the step if the container never becomes healthy. Container
logs are always dumped into a collapsed log group, and the container is removed
on exit.

The image must already exist in the local Docker daemon, so build with
`load: true` (and `push: false`) before calling this action.

### Inputs

| Name | Description | Required | Default |
|------|-------------|----------|---------|
| `image` | Image reference to run. Must be present locally. | Yes | — |
| `container-port` | Port the application listens on inside the container. | Yes | — |
| `health-path` | HTTP path polled until it returns 2xx. | No | `/health` |
| `timeout` | Seconds to wait before failing. | No | `60` |
| `interval` | Seconds between attempts. | No | `3` |

### Outputs

| Name | Description |
|------|-------------|
| `reason` | Human-readable result, set on both success and failure. |

### Usage

```yaml
- uses: docker/setup-buildx-action@v3

- uses: docker/build-push-action@v6
  with:
    context: .
    push: false
    load: true
    tags: my-app:pr-${{ github.event.pull_request.number }}

- name: Smoke test image
  id: smoke
  uses: eliuma/ci-actions/smoke-test@master
  with:
    image: my-app:pr-${{ github.event.pull_request.number }}
    container-port: "3000"
    health-path: "/health"

- name: Publish result
  if: always()
  run: |
    echo "Smoke test: ${{ steps.smoke.outputs.reason }}" >> "$GITHUB_STEP_SUMMARY"
```

### Fleet usage

| Repo | Workflow | Port | Health path |
|------|----------|------|-------------|
| `py-dev-1` | `pr-image-smoke-test.yml` | 8001 | `/healthz` |
| `py-dev-1` | `pr-test-build.yml` (candidate base) | 8001 | `/healthz` |
| `node-dev-2` | `pr-image-smoke-test.yml` | 3000 | `/health` |
| `node-dev-2` | `base-build-check.yml` (candidate base) | 3000 | `/health` |
| `Docker-image` | `pr-validation-py.yml` | 8000 | `/health` |
| `Docker-image` | `pr-validation-node.yml` | 80 | `/` |

## The PR chain

A PR on `Docker-image` validates the candidate base against every consuming app
before it can merge:

```
PR on Docker-image
  └─ build candidate base, smoke test it, push as :pr-N
     └─ repository_dispatch ─> py-dev-1, node-dev-2
          ├─ build the app on the candidate base   (BASE_TAG=pr-N)
          ├─ smoke test the resulting container
          ├─ Trivy scan it, grouped by layer
          └─ a red run here means the candidate base breaks that app
```

Build and smoke run with `continue-on-error` so the scan and the run summary
still execute; the job then re-fails. The run summary states which stage broke
and why, so the failed run is the report.
