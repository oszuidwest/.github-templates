# GitHub Actions Templates

Shared GitHub Actions for repositories at oszuidwest. Two delivery models:

- **Copy-paste templates** (`workflow-templates/`, `config-templates/`) — drop-in files. Use for project-shaped pieces (Dockerfile linting, Trivy scan, shellcheck) where the consumer barely customizes.
- **Reusable workflows** (`.github/workflows/`) — called via `uses: oszuidwest/.github-templates/.github/workflows/<name>.yml@v1`. Use for the larger Go CI/release/Docker pipeline where parameterization replaces copy-paste drift.

## Copy-paste templates

```bash
# Workflows
cp workflow-templates/docker-quality.yml   .github/workflows/
cp workflow-templates/docker-security.yml  .github/workflows/
cp workflow-templates/shell-quality.yml    .github/workflows/

# Config
cp config-templates/.hadolint.yaml         ./
cp config-templates/dependabot.yml         .github/
```

| Workflow | Purpose | Fails on |
|----------|---------|----------|
| docker-quality | Dockerfile, YAML and Compose linting | Lint errors |
| docker-security | Vulnerability scanning (Trivy) | CRITICAL CVEs |
| shell-quality | ShellCheck (only on .sh changes) | Lint errors |

### Customization

**Dockerfile not in root?**
```yaml
env:
  DOCKERFILE_PATH: "docker/Dockerfile"
```

**Adjust Hadolint rules?** Edit `.hadolint.yaml`, not the workflow.

## Reusable workflows

Call from the consumer repo's own `.github/workflows/` files. Pinning to `@v1` follows template fixes automatically; pinning to `@v1.0.0` freezes.

### `go-ci.yml` — Go CI

```yaml
name: CI
on:
  push: { branches: [main], paths: ['**.go', 'go.mod', 'go.sum'] }
  pull_request: { branches: [main], paths: ['**.go', 'go.mod', 'go.sum'] }
permissions: { contents: read }
jobs:
  ci:
    uses: oszuidwest/.github-templates/.github/workflows/go-ci.yml@v1
    with:
      golangci-lint-version: v2.12.1
```

Inputs:

| Input | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `golangci-lint-version` | string | no | `v2.12.1` | Passed to `golangci/golangci-lint-action`. |
| `go-version-file` | string | no | `go.mod` | Passed to `actions/setup-go`. |
| `enable-frontend` | boolean | no | `false` | Adds the frontend lint job. |
| `frontend-tool` | string | no | `bun` | Only `bun` is supported in v1. |

The Go job runs `go test -race -shuffle=on -v ./...`, `go vet`, `go fmt` with diff check, golangci-lint, `deadcode`, `govulncheck`, and `staticcheck`. Consumers must keep `deadcode`, `govulncheck`, and `staticcheck` available through Go tool directives in `go.mod`.

### `go-release.yml` — Go release

```yaml
name: Release
on:
  push: { tags: ['v*'] }
  workflow_dispatch:
    inputs:
      version: { description: 'Version to build', required: true, default: 'edge' }
permissions:
  contents: read
jobs:
  release:
    uses: oszuidwest/.github-templates/.github/workflows/go-release.yml@v1
    with:
      project-name: zwfm-metadata
      ldflags-target: zwfm-metadata/utils
      build-matrix: '[{"os":"linux","arch":"amd64"},{"os":"linux","arch":"arm64"},{"os":"darwin","arch":"arm64"}]'
      version: ${{ inputs.version }}
      enable-docker: true
      image-labels: |
        org.opencontainers.image.title=ZuidWest FM Metadata
        org.opencontainers.image.licenses=MIT
    permissions:
      contents: write
      packages: write  # Required even with enable-docker: false (see below).
```

Inputs:

| Input | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `project-name` | string | yes | n/a | Binary basename and edge artifact prefix. |
| `ldflags-target` | string | yes | n/a | Package path containing `Version`, `Commit`, and `BuildTime`. |
| `build-matrix` | string | yes | n/a | JSON array of `{os, arch, arm?}` build targets. |
| `main-package` | string | no | `.` | Set to `./cmd/foo` for non-root commands. |
| `version` | string | no | `''` | Forward the caller's `workflow_dispatch` input; leave empty for tag pushes. |
| `enable-docker` | boolean | no | `false` | Calls `docker-publish.yml` for releases only, never edge. |
| `image-labels` | string | no | `''` | Multiline OCI labels forwarded to Docker publish. |
| `dockerfile-path` | string | no | `Dockerfile` | Forwarded to Docker publish when enabled. |
| `platforms` | string | no | `linux/amd64,linux/arm64` | Comma-separated Docker platforms. |
| `release-body` | string | no | `''` | Optional markdown prepended to generated release notes. |

**Permissions caveat:** keep the caller workflow default at `contents: read`, then grant `contents: write` and `packages: write` only on the reusable release job. The caller job must always declare `packages: write` even when `enable-docker: false`. GitHub validates nested reusable-workflow permissions at workflow-parse time, so the conditional `docker` job's permission requirement applies regardless of whether `if:` evaluates true. Symptom if missed: `Invalid workflow file ... The nested job 'docker' is requesting 'packages: write', but is only allowed 'packages: none'`.

**LDFLAGS contract:** the workflow injects PascalCase symbols `Version`, `Commit`, `BuildTime` at the package path you pass in `ldflags-target`. The Go package must declare those exact identifiers (e.g. `var Version, Commit, BuildTime string`). Lowercase names (`version`, `buildTime`) are not patched.

### `docker-publish.yml` — reusable Docker build/push

Called transitively by `go-release.yml` when `enable-docker: true`. Direct call only when a repo has its own release flow. Trigger from a tag push so `github.ref_name` resolves to the tag — empty values and `edge` are rejected by the workflow:

```yaml
name: Publish Docker
on:
  push: { tags: ['v*'] }
permissions: { contents: read }
jobs:
  publish:
    uses: oszuidwest/.github-templates/.github/workflows/docker-publish.yml@v1
    with:
      version: ${{ github.ref_name }}
    permissions:
      contents: read
      packages: write
```

Inputs:

| Input | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `version` | string | yes | n/a | Release version. Empty and `edge` are rejected. |
| `image-name` | string | no | `''` | Empty resolves to `${{ github.repository }}`. |
| `dockerfile-path` | string | no | `Dockerfile` | Dockerfile path relative to repo root. |
| `platforms` | string | no | `linux/amd64,linux/arm64` | Comma-separated Docker buildx platforms. |
| `image-labels` | string | no | `''` | Multiline OCI labels passed to metadata/build. |

### Composing custom pre-release gates

`go-release.yml` is build-and-ship only — it does not run tests, lint, or fmt. The assumption is that PR CI already gated the merge into main, so a tag on main is releasable by construction. This is the standard "trust main" model: don't repeat verification work in the release path.

If a consumer needs an extra pre-release check that PR CI doesn't cover (integration tests against a live service, smoke tests, etc.), gate the release on it via `needs:` instead of asking the template to grow another input:

```yaml
name: Release
on:
  push: { tags: ['v*'] }
  workflow_dispatch:
    inputs:
      version: { description: 'Version to build', required: true, default: 'edge' }
permissions:
  contents: read
jobs:
  pre-release:
    uses: ./.github/workflows/integration-test.yml

  release:
    needs: pre-release
    uses: oszuidwest/.github-templates/.github/workflows/go-release.yml@v1
    with:
      project-name: my-service
      ldflags-target: github.com/org/my-service/version
      build-matrix: '[{"os":"linux","arch":"amd64"}]'
      version: ${{ inputs.version }}
    permissions:
      contents: write
      packages: write
```

The custom job lives in the consumer repo, the template stays small, and `needs:` short-circuits the release if the gate fails.

## Maintenance contract

- **Versioning:** semver. Patch for compatible bug fixes and documentation clarifications, minor for additive optional inputs, major for breaking changes (renamed/removed inputs, permission model changes, or default changes that break consumers). Both an immutable `v1.x.y` and a moving `v1` tag exist; consumers pin to `@v1` to follow fixes, `@v1.x.y` to freeze.
- **Release tags:** after a validated merge to `main`, tag the exact merge commit and move the major tag:
  ```bash
  VERSION=v1.1.3
  git fetch origin main --tags
  git switch main
  git pull --ff-only
  git tag "$VERSION"
  git tag -f v1 "$VERSION"
  git push origin "$VERSION"
  git push --force origin refs/tags/v1
  ```
- **Consumer bumps:** `@v1` consumers receive fixes when the moving tag is updated; monitor their next CI/release runs after each template release. Patch-pinned consumers (`@v1.x.y`) need a PR that changes the workflow ref and records the smoke test performed.
- **Dependabot:** keep the `github-actions` ecosystem enabled in this repo and in consumers:
  ```yaml
  - package-ecosystem: "github-actions"
    directory: "/"
    schedule: { interval: "weekly", day: "monday" }
  ```
  Language ecosystems (`gomod`, `npm`, `pip`, `cargo`) remain consumer-specific. The template file keeps them commented so each repo opts into the ecosystems it actually uses.
- **Phase 0 standards** (locked in): action major-pinning, `golangci-lint` pinned, race detection required (`-race -shuffle=on`), strict deadcode findings, `:edge` Docker tag NOT published (artifacts only), `BUILD_TIME` from `date -u`, `provenance: false` and `sbom: false` (opt-in only).
- **Action-pinning exceptions:** major-pinning is the default. Some upstream actions don't publish a moving major tag — for those, pin to the most specific tag they publish:
  - `ludeeus/action-shellcheck` — patch tag (e.g. `@2.0.0`); upstream has no `@v2`/`@2`.
  - `aquasecurity/trivy-action` — release tag (e.g. `@0.36.0`); upstream uses `0.x.0` schema, no major tag.
- **Drift before templating:** new copy-paste templates only after audiologger-style hand-alignment proves the shape on at least two consumers. Don't template a shape that hasn't stabilized.
- **Known v2 candidate:** `go-release.yml` currently keeps `enable-docker` for v1 compatibility, but its nested Docker job forces `packages: write` on all callers at parse time. A future v2 may remove that shortcut and require Docker consumers to call `docker-publish.yml` directly.

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Hadolint can't find Dockerfile | Check `DOCKERFILE_PATH` in workflow |
| Security scan fails | Dockerfile must be buildable |
| Compose warning about env vars | Expected behavior, no action needed |
| Shell workflow doesn't run | Only triggers on `.sh` changes |
| Reusable workflow can't see secrets | `workflow_call` only forwards `GITHUB_TOKEN`; declare other secrets explicitly |
| `go-release.yml` rejects with "nested job 'docker' is requesting 'packages: write'" | Caller must declare `packages: write` even when `enable-docker: false` (parse-time validation) |
