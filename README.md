# Github engineering

Shared reusable GitHub Actions content for application repositories.

See `CLEAN-CODE-STANDARDS.md` for repository conventions and `SCAFFOLDING.md` for the recommended growth structure.

Folder layout under `.github/actions`:

- `ci/`: main CI actions such as build, deploy, artifact upload, release versioning, approval, and capability-based stack folders
- `ci/install/python/`: Python-specific install actions such as `pip-install`
- `ci/quality/python/`: Python-specific quality actions
- `ci/test/python/`: Python-specific test actions
- `secops/`: security and DevSecOps actions such as dependency scan, secret scan, SAST, SBOM, image scan, and IaC scan
- `utils/`: notification and operational utility actions such as notify, rollback notify, and incident notify

Recommended repository structure for multiple tech stacks:

- Keep cross-technology actions generic and reusable: deploy, notify, approval, artifact upload, SBOM, image scan, IaC scan.
- Keep technology-specific actions separate by technology: Python, Java, and Node.js build/test/package actions should live in their own action folders or reusable workflows.
- Prefer capability-first stack folders under `ci/`, for example `ci/install/python/`, `ci/quality/python/`, `ci/test/python/`, and the equivalent Java and Node paths later.
- Treat environments as inputs, not folder names. Deployment logic should use one deploy action with environment parameters rather than one folder per environment.

Practical rule:

- Technology-specific concerns belong to technology-specific actions or workflows.
- Environment-specific concerns belong to workflow inputs and deployment parameters.

Current shared FastAPI pipeline use cases:

- `fast-api-ci.yml`: one reusable workflow with core CI always on and optional jobs for artifact upload, dependency scan, SBOM, secret scan, SAST, image scan, IaC scan, release, approval, deploy, and notifications

Caller-controlled feature flags:

- Consumers decide repository triggers in their own workflow file.
- Consumers enable only the optional jobs they want by setting boolean inputs on the single reusable workflow call.
- Keep install, quality, build, and test as the always-on path. Turn the rest on only when the team is ready.

Example caller overrides:

- Disable only SBOM: `run-sbom: false`
- Keep security checks but disable deployment: `run-deploy: false`
- Enable deploy only on `develop`: `run-deploy: ${{ github.event_name == 'push' && github.ref == 'refs/heads/develop' }}`

Notification utility actions with dummy targets:

- `notify-slack`: channel `#engineering-alerts`, dummy webhook URL
- `notify-pagerduty`: service `PDEV123`, dummy integration key
- `notify-email`: recipient `engineering-alerts@example.com`, SMTP host `smtp.example.com`
- `notify-generic`: endpoint `https://example.internal/hooks/notify`

---

## Composite Actions

### `ci/registry-login`

Log in to any container registry supported by `docker/login-action`.
Useful when you want to compose your own job instead of using the full `container-publish.yml` reusable workflow.

```yaml
steps:
  - uses: VibhuviOiO/github-engineering/.github/actions/ci/registry-login@main
    with:
      registry: ghcr.io
      username: ${{ github.actor }}
      password: ${{ secrets.GITHUB_TOKEN }}
```

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `registry` | no | — | Registry hostname (`ghcr.io`, `quay.io`, leave empty for Docker Hub) |
| `username` | yes | — | Registry username |
| `password` | yes | — | Password or access token |
| `logout` | no | `true` | Log out after the step |

---

## Reusable Workflows

### `container-publish.yml`

Generic, multi-registry Docker image build and publish pipeline.

**Capabilities:**
- Build once, push to GHCR, Docker Hub, and/or Quay.io
- Multi-platform builds via QEMU and Buildx
- Configurable tags via docker/metadata-action rules
- GitHub Actions cache for Docker layers
- Optional provenance and SBOM attestations

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `image-name` | yes | — | Image name without registry prefix |
| `dockerfile` | no | `Dockerfile` | Path to Dockerfile |
| `build-context` | no | `.` | Docker build context |
| `platforms` | no | `linux/amd64` | Comma-separated target platforms |
| `push` | no | `true` | Push after building |
| `tags` | no | `latest`, `sha` | docker/metadata-action tag rules |
| `ghcr-enabled` | no | `true` | Publish to GHCR |
| `dockerhub-enabled` | no | `false` | Publish to Docker Hub |
| `dockerhub-namespace` | no | repo owner | Docker Hub namespace |
| `quay-enabled` | no | `false` | Publish to Quay.io |
| `quay-namespace` | no | repo owner | Quay.io namespace |
| `use-cache` | no | `true` | Enable GHA layer cache |
| `provenance` | no | `false` | Generate provenance attestation |
| `sbom` | no | `false` | Generate SBOM attestation |
| `build-args` | no | — | Multi-line Docker build arguments (`KEY=value`) |

**Secrets:**

| Secret | Required when | Description |
|--------|---------------|-------------|
| `dockerhub-username` | `dockerhub-enabled: true` | Docker Hub username |
| `dockerhub-token` | `dockerhub-enabled: true` | Docker Hub access token |
| `quay-username` | `quay-enabled: true` | Quay.io username |
| `quay-token` | `quay-enabled: true` | Quay.io password or robot token |

GHCR uses the built-in `GITHUB_TOKEN`; no extra secret required.

**Example caller (GHCR + Docker Hub, version from code):**

```yaml
name: Publish

on:
  workflow_dispatch:
  push:
    tags:
      - 'v*'

permissions:
  contents: read
  packages: write
  id-token: write

jobs:
  version:
    runs-on: ubuntu-latest
    outputs:
      version: ${{ steps.version.outputs.version }}
    steps:
      - uses: actions/checkout@v4
      - id: version
        run: |
          VERSION=$(cat VERSION | tr -d '[:space:]')
          echo "version=${VERSION}" >> "$GITHUB_OUTPUT"

  publish:
    needs: version
    uses: VibhuviOiO/github-engineering/.github/workflows/container-publish.yml@main
    with:
      image-name: my-app
      dockerfile: Dockerfile.prod
      platforms: linux/amd64,linux/arm64
      ghcr-enabled: true
      dockerhub-enabled: true
      dockerhub-namespace: myorg
      build-args: |
        APP_VERSION=${{ needs.version.outputs.version }}
      tags: |
        type=raw,value=latest
        type=raw,value=${{ needs.version.outputs.version }}
        type=semver,pattern={{version}}
        type=sha,prefix=
    secrets: inherit
```

**Example caller (all three registries):**

```yaml
jobs:
  publish:
    uses: VibhuviOiO/github-engineering/.github/workflows/container-publish.yml@main
    with:
      image-name: my-app
      ghcr-enabled: true
      dockerhub-enabled: true
      dockerhub-namespace: myorg
      quay-enabled: true
      quay-namespace: myorg
    secrets: inherit
```

### `validate-container-secrets.yml`

Test registry authentication without building or pushing. Run this after adding secrets to a repository.

**Inputs:**

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `ghcr-enabled` | no | `true` | Validate GHCR login |
| `dockerhub-enabled` | no | `false` | Validate Docker Hub login |
| `quay-enabled` | no | `false` | Validate Quay.io login |

**Secrets:** same as `container-publish.yml`.

**Example caller:**

```yaml
name: Validate Secrets
on: workflow_dispatch

permissions:
  contents: read
  packages: read

jobs:
  validate:
    uses: VibhuviOiO/github-engineering/.github/workflows/validate-container-secrets.yml@main
    with:
      ghcr-enabled: true
      dockerhub-enabled: true
      quay-enabled: true
    secrets: inherit
```

---

## Registry Setup Guide

### GitHub Container Registry (GHCR)

No extra credentials required. The workflow uses the built-in `GITHUB_TOKEN`.

1. Go to **Settings → Actions → General** in the repository.
2. Under **Workflow permissions**, ensure **Read and write permissions** is selected.
3. The first push will create the package automatically under `https://github.com/users/VibhuviOiO/packages`.

### Docker Hub

1. Log in to [hub.docker.com](https://hub.docker.com).
2. Create an access token:
   - **Account Settings → Security → New Access Token**
   - Name it `github-actions-ldap` or similar
   - Copy the token (you will only see it once)
3. Add repository/org **Secrets**:
   - `DOCKERHUB_USERNAME` = your Docker Hub username
   - `DOCKERHUB_TOKEN` = the access token
4. Repositories are created automatically on first push (e.g., `vibhuvioio/ldap-manager`, `vibhuvioio/openldap`).

### Quay.io

1. Sign up or log in to [quay.io](https://quay.io).
2. Create an organization (e.g., `vibhuvioio`) or use your personal namespace.
3. Create a robot account or use your password:
   - **Organization → Robot Accounts → Create Robot Account**
   - Copy the username and token
4. Add repository/org **Secrets**:
   - `QUAY_USERNAME` = robot account username or your username
   - `QUAY_TOKEN` = robot token or encrypted password
5. Repositories can be created automatically on first push or manually in the UI.

### Recommended secrets per repository

| Repository | GHCR | Docker Hub | Quay |
|---|---|---|---|
| `ldap-manager` | `GITHUB_TOKEN` (auto) | `DOCKERHUB_USERNAME`<br>`DOCKERHUB_TOKEN` | `QUAY_USERNAME`<br>`QUAY_TOKEN` |
| `openldap-docker` | `GITHUB_TOKEN` (auto) | `DOCKERHUB_USERNAME`<br>`DOCKERHUB_TOKEN` | `QUAY_USERNAME`<br>`QUAY_TOKEN` |