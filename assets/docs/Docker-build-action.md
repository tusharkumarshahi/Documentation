# Composite Action — Docker Build and Push to ACR

## Scope

Reusable GitHub Actions composite action that handles Docker image build, tagging, and push to Azure Container Registry (ACR). Designed to be called from any repository workflow within the `medica-dev-platform` organization via `github-common-actions`.

Covers:
- Docker image build using BuildKit
- Image tagging with a caller-supplied version tag
- Push to Azure Container Registry
- Build layer caching via GitHub Actions cache
- Optional push — can be used for build-only validation without registry access

Does not cover Maven build, version determination, or deployment — those are handled by separate steps or actions in the calling workflow.

---

## Logic

The action executes four steps in sequence:

**1. Set up Docker Buildx**
Initializes BuildKit via `docker/setup-buildx-action@v4` — the modern Docker build engine. Required for GitHub Actions cache layer support and efficient multi-layer builds.

**2. Log in to ACR**
Authenticates against the provided ACR instance using `docker/login-action@v4`. This step is conditional — it only runs when `push` input is `true`. For build-only runs this step is skipped entirely, meaning no credentials are required.

**3. Build and push Docker image**
Builds the image using `docker/build-push-action@v7` with BuildKit. Layer cache is read from and written to GitHub Actions cache (`type=gha`) to speed up subsequent runs. The image is pushed to ACR only when `push` is `true`. The image tag is supplied by the calling workflow — typically the output of a preceding version determination step.

**4. Set image reference output**
Constructs and exposes the full image reference as a step output for use in downstream steps such as deployment or notifications. When `push` is `false`, the reference is local only (no registry prefix).

---

## Why This Approach?

**Official Docker actions over raw CLI**  
Uses maintained Docker actions for proper credential handling, BuildKit setup, and caching — avoiding errors common with raw CLI.

**BuildKit via `setup-buildx-action`**  
Enables faster builds, better caching, and GitHub Actions cache support (`type=gha`).

**`push` input toggle**  
Supports build-only mode (`push: false`) for validation without registry access — useful for PRs and early pipeline stages.

**Image tag as input (not generated)**  
Tagging is handled by the calling workflow, keeping this action decoupled and reusable across projects.

**Image digest as output**  
Exposes the image SHA256 digest so deployments can use an immutable reference instead of a mutable tag.

---

## Secret / Environment Handling

ACR credentials are only required when `push: true`. For build-only runs no secrets are needed.

| Input | Required | Description |
|-------|----------|-------------|
| `image-name` | Yes | Image name without registry prefix e.g. `my-app` |
| `image-tag` | Yes | Image tag — pass output from previous version step |
| `dockerfile-path` | No (default: `./Dockerfile`) | Path to Dockerfile |
| `build-context` | No (default: `.`) | Docker build context directory |
| `build-args` | No | Additional `--build-arg` flags, one per line |
| `push` | No (default: `false`) | Whether to push to ACR |
| `acr-login-server` | When `push: true` | ACR login server e.g. `myregistry.azurecr.io` |
| `acr-username` | When `push: true` | ACR username (service principal client ID or admin) |
| `acr-password` | When `push: true` | ACR password (service principal secret or admin) |

**GitHub Secrets required when pushing:**

| GitHub Secret | Description |
|--------------|-------------|
| `ACR_LOGIN_SERVER` | ACR login server URL |
| `ACR_USERNAME` | Service principal client ID or admin username |
| `ACR_PASSWORD` | Service principal client secret or admin password |

ACR credentials are found in Azure Portal → ACR resource → Access keys. Admin user must be enabled, or a service principal with `AcrPush` role assigned should be used for production.

---

## Code / PR Link

| Item | Link |
|------|------|
| Composite action file | `github-common-actions/.github/actions/docker-build/action.yml` |
| Branch | `feature/docker-build-action` |

---

## Walkthrough / Runbook

### Action Location

```
github-common-actions
└── .github/
    └── actions/
        └── docker-build/
            └── action.yml
```

### How to Call This Action

**Build only — no registry needed:**
```yaml
- name: Docker Build
  id: docker
  uses: medica-dev-platform/github-common-actions/.github/actions/docker-build@main
  with:
    image-name: 'my-app'
    image-tag: ${{ steps.version.outputs.tag }}
    push: 'false'
```

**Build and push to ACR:**
```yaml
- name: Docker Build and Push
  id: docker
  uses: medica-dev-platform/github-common-actions/.github/actions/docker-build@main
  with:
    image-name: 'my-app'
    image-tag: ${{ steps.version.outputs.tag }}
    push: 'true'
    acr-login-server: ${{ secrets.ACR_LOGIN_SERVER }}
    acr-username: ${{ secrets.ACR_USERNAME }}
    acr-password: ${{ secrets.ACR_PASSWORD }}
```

**Using outputs in a downstream step:**
```yaml
- name: Deploy
  run: |
    echo "Image: ${{ steps.docker.outputs.image-full-ref }}"
    echo "Digest: ${{ steps.docker.outputs.image-digest }}"
```

### Dockerfile Requirements

The repository calling this action must have a `Dockerfile` at the root (or at the path specified via `dockerfile-path`). For a Java Maven application the recommended minimal Dockerfile is:

```dockerfile
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY target/*.jar app.jar
ENTRYPOINT ["java", "-jar", "app.jar"]
```

The Maven build step must run before this action so that `target/*.jar` exists in the workspace.

### Runner Requirements

| Requirement | Status |
|-------------|--------|
| Docker | Pre-installed on `ubuntu-latest` GitHub-hosted runners |
| Docker | Must be installed on self-hosted runner for client environment |
| ACR network access | Required when `push: true` |

### Testing Status

| Test | Environment | Runner | Result |
|------|-------------|--------|--------|
| Build only (`push: false`) | Personal GitHub| `ubuntu-latest` | ✅ Passed |
| Build and push to ACR (`push: true`) | Personal GitHub (`test-maven-app`) | `ubuntu-latest` | ✅ Passed — image visible in ACR with version tag |
| Full flow with version step | Personal GitHub (`test-maven-app`) | `ubuntu-latest` | ✅ Passed — tagged `1.0.0-d435e82` |
| Client environment | Client GHE | `dlx110-ratp002-1` | ✅ Passed |

### Outputs Reference

| Output | Description | Example |
|--------|-------------|---------|
| `image-full-ref` | Full image reference pushed to ACR | `myregistry.azurecr.io/my-app:1.0.0-a1b2c3d` |
| `image-digest` | SHA256 digest of pushed image | `sha256:abc123...` |