# GitHub Actions Templates

Shared GitHub Actions for repositories at oszuidwest. Two delivery models:

- **Copy-paste templates** (`workflow-templates/`, `config-templates/`) — drop-in files. Use for project-shaped pieces (Dockerfile linting, Trivy scan, shellcheck) where the consumer barely customizes.
- **Reusable workflows** (`.github/workflows/`) — called via `uses: oszuidwest/.github-templates/.github/workflows/<name>.yml@vX` (per-workflow major: `go-ci.yml@v1`, `go-release.yml@v2`, `docker-publish.yml@v1`). Use for the larger Go CI/release/Docker pipeline where parameterization replaces copy-paste drift.

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

Call from the consumer repo's own `.github/workflows/` files. Pinning to the moving major tag (`@v1`, `@v2`) follows template fixes automatically; pinning to a patch tag (`@v1.0.0`, `@v2.0.0`) freezes. Each reusable workflow has its own major line — see the per-workflow examples below for current versions.

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
    uses: oszuidwest/.github-templates/.github/workflows/go-release.yml@v2
    with:
      project-name: zwfm-metadata
      ldflags-target: zwfm-metadata/utils
      build-matrix: '[{"os":"linux","arch":"amd64"},{"os":"linux","arch":"arm64"},{"os":"darwin","arch":"arm64"}]'
      version: ${{ inputs.version }}
    permissions:
      contents: write
```

Inputs:

| Input | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `project-name` | string | yes | n/a | Binary basename and edge artifact prefix. |
| `ldflags-target` | string | yes | n/a | Package path containing `Version`, `Commit`, and `BuildTime`. |
| `build-matrix` | string | yes | n/a | JSON array of `{os, arch, arm?}` build targets. |
| `main-package` | string | no | `.` | Set to `./cmd/foo` for non-root commands. |
| `version` | string | no | `''` | Forward the caller's `workflow_dispatch` input; leave empty for tag pushes. |
| `release-body` | string | no | `''` | Optional markdown prepended to generated release notes. |

Outputs:

| Output | Notes |
|--------|-------|
| `version` | Resolved release version. `vX.Y.Z` for tagged runs, `edge-<short-sha>` for edge. |
| `is_release` | `'true'` for tagged releases (including prereleases), `'false'` for edge. Gate downstream Docker publish on this. |

**LDFLAGS contract:** the workflow injects PascalCase symbols `Version`, `Commit`, `BuildTime` at the package path you pass in `ldflags-target`. The Go package must declare those exact identifiers (e.g. `var Version, Commit, BuildTime string`). Lowercase names (`version`, `buildTime`) are not patched.

#### Releasing with a Docker image

v2 dropped the nested docker job. Compose with `docker-publish.yml` from a second job in the caller workflow, gated on the release outputs:

```yaml
jobs:
  release:
    uses: oszuidwest/.github-templates/.github/workflows/go-release.yml@v2
    with:
      project-name: zwfm-metadata
      ldflags-target: zwfm-metadata/utils
      build-matrix: '[{"os":"linux","arch":"amd64"},{"os":"linux","arch":"arm64"}]'
      version: ${{ inputs.version }}
    permissions:
      contents: write

  docker:
    needs: release
    if: needs.release.outputs.is_release == 'true'
    uses: oszuidwest/.github-templates/.github/workflows/docker-publish.yml@v1
    with:
      version: ${{ needs.release.outputs.version }}
      image-labels: |
        org.opencontainers.image.title=ZuidWest FM Metadata
        org.opencontainers.image.licenses=MIT
    permissions:
      contents: read
      packages: write
```

The `if:` skips Docker for edge runs; `needs.release.outputs.version` forwards the resolved tag (with `v` prefix) so `docker-publish.yml`'s semver patterns produce the expected `:1.2.3`, `:1.2`, `:1`, `:latest` tags. Each job declares only the permissions it needs — non-Docker callers stay at `contents: write` only.

### `docker-publish.yml` — reusable Docker build/push

Called from a release flow alongside `go-release.yml` (see "Releasing with a Docker image" above) or directly from a tag push when no Go binary is involved. `github.ref_name` resolves to the tag — empty values and `edge` are rejected by the workflow:

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
    uses: oszuidwest/.github-templates/.github/workflows/go-release.yml@v2
    with:
      project-name: my-service
      ldflags-target: github.com/org/my-service/version
      build-matrix: '[{"os":"linux","arch":"amd64"}]'
      version: ${{ inputs.version }}
    permissions:
      contents: write
```

The custom job lives in the consumer repo, the template stays small, and `needs:` short-circuits the release if the gate fails.

## Maintenance contract

- **Versioning:** semver. Patch for compatible bug fixes and documentation clarifications, minor for additive optional inputs, major for breaking changes (renamed/removed inputs, permission model changes, or default changes that break consumers). Each major line keeps an immutable `vX.Y.Z` and a moving `vX` tag; consumers pin to `@vX` to follow fixes, `@vX.Y.Z` to freeze.
- **Active major lines:** `v1` (legacy — `go-release.yml` with nested `docker` job) and `v2` (current — composes with `docker-publish.yml` from a separate caller job). New consumers pin `@v2`; v1 stays available for already-pinned consumers until they migrate.
- **Release tags:** after a validated merge to `main`, tag the exact merge commit and move the major tag:
  ```bash
  VERSION=v2.0.1   # or v1.x.y for a v1-line patch
  MAJOR=v2         # match VERSION's major
  git fetch origin main --tags
  git switch main
  git pull --ff-only
  git tag "$VERSION"
  git tag -f "$MAJOR" "$VERSION"
  git push origin "$VERSION"
  git push --force origin "refs/tags/$MAJOR"
  ```
- **Consumer bumps:** `@vX` consumers receive fixes when the moving tag is updated; monitor their next CI/release runs after each template release. Patch-pinned consumers (`@vX.Y.Z`) need a PR that changes the workflow ref and records the smoke test performed.
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

### Migrating `go-release.yml` from v1 to v2

v2 dropped the nested `docker` job and the four inputs that fed it (`enable-docker`, `image-labels`, `dockerfile-path`, `platforms`). Consumers that publish a Docker image now add a second job that calls `docker-publish.yml` directly.

For a consumer currently calling `go-release.yml@v1` with `enable-docker: true`:

1. Bump `@v1` → `@v2` on the release job.
2. Drop `enable-docker`, `image-labels`, `dockerfile-path`, `platforms` from the release call.
3. Drop `packages: write` from the release job's permissions (`contents: write` is enough now).
4. Add a second job that calls `docker-publish.yml@v1` with `needs: release`, gated on `needs.release.outputs.is_release == 'true'`, and forwards `version` from the release outputs. See "Releasing with a Docker image" for the full snippet.

For a consumer currently calling `go-release.yml@v1` with `enable-docker: false`:

1. Bump `@v1` → `@v2`.
2. Drop `packages: write` from the release job's permissions — the parse-time requirement is gone.

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Hadolint can't find Dockerfile | Check `DOCKERFILE_PATH` in workflow |
| Security scan fails | Dockerfile must be buildable |
| Compose warning about env vars | Expected behavior, no action needed |
| Shell workflow doesn't run | Only triggers on `.sh` changes |
| Reusable workflow can't see secrets | `workflow_call` only forwards `GITHUB_TOKEN`; declare other secrets explicitly |
