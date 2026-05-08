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

## Reusable workflows (Phase 2)

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

Inputs: `golangci-lint-version` (default `v2.12.1`), `go-version-file` (default `go.mod`), `enable-frontend` (default `false`), `frontend-tool` (default `bun`).

### `go-release.yml` — Go release

```yaml
name: Release
on:
  push: { tags: ['v*'] }
  workflow_dispatch:
    inputs:
      version: { description: 'Version to build', required: true, default: 'edge' }
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

Inputs: `project-name`, `ldflags-target`, `build-matrix` (JSON), `main-package` (default `.` — set to `./cmd/foo` for repos with a non-root main), `version` (forward `inputs.version` from the consumer's `workflow_dispatch`; leave empty for tag pushes), `image-labels` (multiline), `enable-docker` (default `false`).

**Permissions caveat:** the caller must always declare `packages: write` even when `enable-docker: false`. GitHub validates nested reusable-workflow permissions at workflow-parse time, so the conditional `docker` job's permission requirement applies regardless of whether `if:` evaluates true. Symptom if missed: `Invalid workflow file ... The nested job 'docker' is requesting 'packages: write', but is only allowed 'packages: none'`.

**LDFLAGS contract:** the workflow injects PascalCase symbols `Version`, `Commit`, `BuildTime` at the package path you pass in `ldflags-target`. The Go package must declare those exact identifiers (e.g. `var Version, Commit, BuildTime string`). Lowercase names (`version`, `buildTime`) are not patched.

### `docker-publish.yml` — reusable Docker build/push

Called transitively by `go-release.yml` when `enable-docker: true`. Direct call only when a repo has its own release flow:

```yaml
jobs:
  publish:
    uses: oszuidwest/.github-templates/.github/workflows/docker-publish.yml@v1
    with:
      version: ${{ github.ref_name }}  # only call from a tag push or after tagging
    permissions:
      contents: read
      packages: write
```

Inputs: `version` (required, must be non-empty and not `edge` — the workflow rejects edge per Phase 0), `image-name` (default `${{ github.repository }}`), `dockerfile-path` (default `Dockerfile`), `platforms` (default `linux/amd64,linux/arm64`), `image-labels`.

## Maintenance contract

- **Versioning:** semver. Minor for additive optional inputs, major for breaking changes (renamed/removed inputs, default changes that break consumers). Both an immutable `v1.x.y` and a moving `v1` tag exist; consumers pin to `@v1` to follow fixes, `@v1.x.y` to freeze.
- **Bumps:** dependabot via `github-actions` ecosystem in this repo + each consumer.
- **Phase 0 standards** (locked in): action major-pinning, `golangci-lint` pinned, race detection required (`-race -shuffle=on`), `:edge` Docker tag NOT published (artifacts only), `BUILD_TIME` from `date -u`, `provenance: false` and `sbom: false` (opt-in only).
- **Action-pinning exceptions:** major-pinning is the default. Some upstream actions don't publish a moving major tag — for those, pin to the most specific tag they publish:
  - `ludeeus/action-shellcheck` — patch tag (e.g. `@2.0.0`); upstream has no `@v2`/`@2`.
  - `aquasecurity/trivy-action` — release tag (e.g. `@0.36.0`); upstream uses `0.x.0` schema, no major tag.
- **Drift before templating:** new copy-paste templates only after audiologger-style hand-alignment proves the shape on at least two consumers. Don't template a shape that hasn't stabilized.

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Hadolint can't find Dockerfile | Check `DOCKERFILE_PATH` in workflow |
| Security scan fails | Dockerfile must be buildable |
| Compose warning about env vars | Expected behavior, no action needed |
| Shell workflow doesn't run | Only triggers on `.sh` changes |
| Reusable workflow can't see secrets | `workflow_call` only forwards `GITHUB_TOKEN`; declare other secrets explicitly |
| `go-release.yml` rejects with "nested job 'docker' is requesting 'packages: write'" | Caller must declare `packages: write` even when `enable-docker: false` (parse-time validation) |
