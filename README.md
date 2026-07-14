# GitHub Actions Templates

Shared GitHub Actions for repositories at oszuidwest. Two delivery models:

- **Copy-paste templates** (`workflow-templates/`, `config-templates/`) - drop-in files. Use for project-shaped pieces (Dockerfile linting, Trivy scan, shellcheck) where the consumer barely customizes.
- **Reusable workflows** (`.github/workflows/`) - called via `uses: oszuidwest/.github-templates/.github/workflows/<name>.yml@v2`. Use for the larger Go CI/release/Docker pipeline where parameterization replaces copy-paste drift.

## Copy-paste templates

```bash
# Workflows
cp workflow-templates/docker-quality.yml   .github/workflows/
cp workflow-templates/docker-security.yml  .github/workflows/
cp workflow-templates/shell-quality.yml    .github/workflows/
cp workflow-templates/workflow-quality.yml .github/workflows/
cp workflow-templates/wp-lint.yml           .github/workflows/
cp workflow-templates/wp-js-lint.yml        .github/workflows/
cp workflow-templates/wp-tests.yml          .github/workflows/
cp workflow-templates/wp-release.yml        .github/workflows/

# Config
cp config-templates/.hadolint.yaml         ./
cp config-templates/dependabot.yml         .github/
cp config-templates/phpcs.xml.dist         ./   # WordPress plugin baseline
```

| Workflow | Purpose | Fails on |
|----------|---------|----------|
| docker-quality | Dockerfile, YAML and Compose linting | Lint errors |
| docker-security | Thin caller for centralized Trivy scanning | Never; reports only |
| shell-quality | ShellCheck (only on .sh changes) | Lint errors |
| workflow-quality | GitHub Actions syntax validation | Actionlint errors |
| wp-lint | PHP, static analysis, Plugin Check and translations | Quality errors |
| wp-js-lint | npm audit plus the consumer's lint/build command | Quality errors |
| wp-tests | PHPUnit matrix with optional coverage | Test or coverage errors |
| wp-release | Thin caller for centralized WordPress releases | Release errors |

### Customization

**Dockerfile not in root?** Call `docker-security-build.yml` directly and pass
`dockerfile-path`; see the reusable workflow section below.

**Adjust Hadolint rules?** Edit `.hadolint.yaml`, not the workflow.

### `phpcs.xml.dist` - WordPress plugin baseline

Shared phpcs ruleset for oszuidwest WordPress plugins. Drop into the plugin root as `phpcs.xml` (or `phpcs.xml.dist`) and prepend the per-plugin specifics. Required additions:

```xml
<file>my-plugin.php</file>
<file>includes</file>

<config name="text_domain" value="my-plugin"/>

<rule ref="WordPress.NamingConventions.PrefixAllGlobals">
    <properties>
        <property name="prefixes" type="array">
            <element value="my_plugin"/>
            <element value="MY_PLUGIN"/>
        </properties>
    </properties>
</rule>
```

The baseline locks down `WordPress-Extra` + `WordPress-Docs` + `PHPCompatibilityWP` against `8.3-8.4` / WP 6.8+. Override this file's `<config name="testVersion"/>` only when a plugin needs a wider PHP range.

## Reusable workflows

Call from the consumer repo's own `.github/workflows/` files. Most reusable
workflows should pin to the moving major tag (`@v2`) so template fixes follow
automatically, or to a patch tag (`@v2.0.0`) to freeze. The Docker security
callers currently pin to `@main` because the whole point is to absorb scanning
policy updates centrally without rolling every repository.

### `docker-security-build.yml` - reusable Docker image scan

Builds the caller repository's Docker image locally and scans it with Trivy.
SARIF is uploaded to GitHub Security only for public non-PR runs; pull requests
and private repositories still get a non-blocking high/critical table summary.
The image is always built without layer cache: cached RUN layers freeze
package installs, so a cached scan keeps reporting stale packages and misses
security updates that a fresh build would pull in.

```yaml
name: Docker Security
"on":
  push: { branches: [main] }
  pull_request:
  schedule:
    - cron: "0 4 * * 0"
  workflow_dispatch:
permissions:
  contents: read
  security-events: write
jobs:
  security:
    uses: oszuidwest/.github-templates/.github/workflows/docker-security-build.yml@main
    with:
      event-name: ${{ github.event_name }}
      repository-private: ${{ github.event.repository.private }}
```

Inputs:

| Input | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `context` | string | no | `.` | Docker build context. |
| `dockerfile-path` | string | no | `Dockerfile` | Dockerfile path relative to repo root. |
| `image-ref` | string | no | `<owner>/<repo>:security-scan` | Local image ref used by Trivy. |
| `build-args` | string | no | `''` | Multiline Docker build args. |
| `category` | string | no | `''` | SARIF upload category. Set a unique value per image when one repo scans multiple images; empty uses the codeql-action default. |
| `event-name` | string | yes | n/a | Pass `${{ github.event_name }}` from the caller. |
| `repository-private` | boolean | yes | n/a | Pass `${{ github.event.repository.private }}`. |
| `sarif-severity` | string | no | `CRITICAL,HIGH,MEDIUM` | Severities uploaded to GitHub Security. |
| `summary-severity` | string | no | `CRITICAL,HIGH` | Severities shown in logs. |
| `scanners` | string | no | `vuln,secret` | Trivy scanners. |
| `timeout-minutes` | number | no | `30` | Job timeout. |

Example for a non-root Dockerfile with build args:

```yaml
jobs:
  security:
    uses: oszuidwest/.github-templates/.github/workflows/docker-security-build.yml@main
    with:
      dockerfile-path: docker/Dockerfile
      build-args: |
        TOOL=odr-padenc
        VERSION=${{ needs.env.outputs.odr-padenc-version }}
      event-name: ${{ github.event_name }}
      repository-private: ${{ github.event.repository.private }}
```

When one Dockerfile produces multiple images (different build args), call the
workflow with a matrix and give every image its own `category`. Without a
unique category the SARIF uploads share one analysis and overwrite each other,
so only the last-uploaded image is visible in the Security tab:

```yaml
jobs:
  security:
    name: ${{ matrix.image }}
    strategy:
      fail-fast: false
      matrix:
        include:
          - { image: odr-padenc, tool: odr-padenc, variant: "" }
          - { image: odr-audioenc-full, tool: odr-audioenc, variant: full }
    uses: oszuidwest/.github-templates/.github/workflows/docker-security-build.yml@main
    with:
      dockerfile-path: docker/Dockerfile
      build-args: |
        TOOL=${{ matrix.tool }}
        VARIANT=${{ matrix.variant }}
      category: security-scan-${{ matrix.image }}
      event-name: ${{ github.event_name }}
      repository-private: ${{ github.event.repository.private }}
```

### `docker-security-images.yml` - reusable existing image scan

Scans one or more existing image references. Use this for deployment repos that
pin upstream images in Compose instead of building a local Dockerfile.

```yaml
jobs:
  images:
    runs-on: ubuntu-latest
    outputs:
      images-json: ${{ steps.images.outputs.images-json }}
    steps:
      - uses: actions/checkout@v6
      - id: images
        run: |
          IMAGE="$(awk '$2 ~ /^example:/ { print $2; exit }' docker-compose.yml)"
          echo "images-json=$(jq -cn --arg image "$IMAGE" \
            '[{name:"app", image:$image, category:"container-app"}]')" \
            >> "$GITHUB_OUTPUT"

  security:
    needs: images
    uses: oszuidwest/.github-templates/.github/workflows/docker-security-images.yml@main
    with:
      images-json: ${{ needs.images.outputs.images-json }}
      event-name: ${{ github.event_name }}
      repository-private: ${{ github.event.repository.private }}
```

`images-json` is a JSON array. Each entry supports:

| Field | Required | Notes |
|-------|----------|-------|
| `name` | yes | Matrix display name. |
| `image` | yes | Image reference for Trivy. |
| `category` | yes | SARIF category, e.g. `container-caddy`. |
| `pkg_types` | no | Trivy package types. Defaults to `os,library`; use `os` for OS-only scans. |

### `go-ci.yml` - Go CI

```yaml
name: CI
"on":
  push: { branches: [main], paths: ['**.go', 'go.mod', 'go.sum'] }
  pull_request: { branches: [main], paths: ['**.go', 'go.mod', 'go.sum'] }
permissions: { contents: read }
jobs:
  ci:
    uses: oszuidwest/.github-templates/.github/workflows/go-ci.yml@v2
    with:
      golangci-lint-version: v2.12.1
```

Inputs:

| Input | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `golangci-lint-version` | string | no | `v2.12.1` | Passed to `golangci/golangci-lint-action`. |
| `go-version-file` | string | no | `go.mod` | Passed to `actions/setup-go`. |
| `enable-frontend` | boolean | no | `false` | Adds the frontend lint job. |
| `frontend-tool` | string | no | `bun` | Only `bun` is supported. |

The Go job runs `go test -race -shuffle=on -v ./...`, `go vet`, `go fmt` with diff check, golangci-lint, `deadcode`, `govulncheck`, and `staticcheck`. Consumers must keep `deadcode`, `govulncheck`, and `staticcheck` available through Go tool directives in `go.mod`.

### `go-release.yml` - Go release

```yaml
name: Release
"on":
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
| `build-matrix` | string | yes | n/a | JSON array of `{os, arch, arm?, ldflags?}` build targets; optional `ldflags` is appended after the injected symbols, e.g. `{"os":"windows","arch":"amd64","ldflags":"-H windowsgui"}`. |
| `main-package` | string | no | `.` | Set to `./cmd/foo` for non-root commands. |
| `version` | string | no | `''` | Forward the caller's `workflow_dispatch` input; leave empty for tag pushes. |
| `allow-tag-rebuild` | boolean | no | `false` | Permit a manual rebuild of a pre-existing tag whose commit differs from the dispatch ref. |
| `release-body` | string | no | `''` | Optional markdown prepended to generated release notes. |

Outputs:

| Output | Notes |
|--------|-------|
| `version` | Resolved release version. `vX.Y.Z` for tagged runs, `edge-<short-sha>` for edge. |
| `is_release` | `'true'` for tagged releases (including prereleases), `'false'` for edge. Gate downstream Docker publish on this. |
| `commit` | Resolved build commit SHA. Usually matches `github.sha`; differs only for explicit tag rebuilds. |

**LDFLAGS contract:** the workflow injects PascalCase symbols `Version`, `Commit`, `BuildTime` at the package path you pass in `ldflags-target`. The Go package must declare those exact identifiers (e.g. `var Version, Commit, BuildTime string`). Lowercase names (`version`, `buildTime`) are not patched.

For manual releases, `version: vX.Y.Z` creates the tag on the dispatch commit
when it does not already exist. If the tag already exists, the workflow only
reuses it by default when it points at the same commit as the dispatch ref. This
keeps caller-side `needs:` gates aligned with the binaries that are published.
Set `allow-tag-rebuild: true` only when intentionally rebuilding an existing
tag from its original commit.

Supported triggers are tag pushes (`refs/tags/*`) and `workflow_dispatch`. Any
other event — including branch pushes, `schedule`, and `release` — fails fast
instead of being treated as a release.

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
    uses: oszuidwest/.github-templates/.github/workflows/docker-publish.yml@v2
    with:
      version: ${{ needs.release.outputs.version }}
      image-labels: |
        org.opencontainers.image.title=ZuidWest FM Metadata
        org.opencontainers.image.licenses=MIT
    permissions:
      contents: read
      packages: write
```

The `if:` skips Docker for edge runs; `needs.release.outputs.version` forwards the resolved tag (with `v` prefix) so `docker-publish.yml`'s semver patterns produce the expected `:1.2.3`, `:1.2`, `:1`, `:latest` tags. Each job declares only the permissions it needs - non-Docker callers stay at `contents: write` only.

### `docker-publish.yml` - reusable Docker build/push

Called from a release flow alongside `go-release.yml` (see "Releasing with a Docker image" above) or directly from a tag push when no Go binary is involved. `github.ref_name` resolves to the tag - empty values and `edge` are rejected by the workflow:

```yaml
name: Publish Docker
"on":
  push: { tags: ['v*'] }
permissions: { contents: read }
jobs:
  publish:
    uses: oszuidwest/.github-templates/.github/workflows/docker-publish.yml@v2
    with:
      version: ${{ github.ref_name }}
    permissions:
      contents: read
      packages: write
```

The workflow sets up QEMU before Buildx so the default
`linux/amd64,linux/arm64` platforms build on GitHub-hosted runners.
Docker layer cache is imported from GitHub Actions cache and exported
best-effort after the build.

Inputs:

| Input | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `version` | string | yes | n/a | Release version. Empty and `edge` are rejected. |
| `image-name` | string | no | `''` | Empty resolves to `${{ github.repository }}`. |
| `dockerfile-path` | string | no | `Dockerfile` | Dockerfile path relative to repo root. |
| `platforms` | string | no | `linux/amd64,linux/arm64` | Comma-separated Docker buildx platforms. |
| `image-labels` | string | no | `''` | Multiline OCI labels passed to metadata/build. |

### `wp-ci.yml` - WordPress plugin lint

PHP matrix `["8.3", "8.4"]` with recursive syntax validation, Composer,
PHP_CodeSniffer, PHPStan, Plugin Check, and an optional translations job that
auto-runs when `languages/*.pot` exists. Existing v2 behavior remains the
default; stricter validation and dependency auditing are opt-in.

```yaml
name: Lint
"on":
  push: { branches: [main], paths: ['**/*.php', 'composer.json', 'composer.lock', 'phpcs.xml', 'phpstan.neon', 'languages/**'] }
  pull_request: { paths: ['**/*.php', 'composer.json', 'composer.lock', 'phpcs.xml', 'phpstan.neon', 'languages/**'] }
  workflow_dispatch:
permissions: { contents: read }
jobs:
  lint:
    uses: oszuidwest/.github-templates/.github/workflows/wp-ci.yml@v2
    with:
      composer-audit: true
      exact-pot-validation: true
      strict-composer-validation: true
```

The `php` job runs recursive `php --syntax-check`, `composer validate`,
`composer install`, `phpcs` (checkstyle/cs2pr), `phpstan`
(checkstyle/cs2pr), and `wordpress/plugin-check-action@v1` once on PHP 8.4.
Plugin Check inputs allow consumers to provide their slug, build directory,
and project-specific file or directory exclusions while retaining the shared
default check exclusions.

The `translations` job auto-detects `languages/*.pot`, runs `wp i18n make-pot` against a fresh copy, and fails on missing strings. Slug and domain are derived from the POT filename (e.g. `languages/foo.pot` -> slug+domain `foo`).

Consumer requirements: `composer.json` declares `phpcs` and `phpstan` as dev dependencies; `phpcs.xml` (or `phpcs.xml.dist`) and `phpstan.neon` exist at repo root.

Inputs:

| Input | Type | Default | Notes |
|-------|------|---------|-------|
| `composer-audit` | boolean | `false` | Runs `composer audit --locked` once on PHP 8.4. |
| `strict-composer-validation` | boolean | `false` | Adds `--strict` to `composer validate`. |
| `exact-pot-validation` | boolean | `false` | Diffs the complete POT after removing volatile headers. |
| `pot-include` | string | `''` | Comma-separated paths included by `wp i18n make-pot`. |
| `pot-exclude` | string | `''` | Comma-separated paths excluded by `wp i18n make-pot`. |
| `plugin-check-build-dir` | string | `./` | Plugin build directory passed to Plugin Check. |
| `plugin-check-slug` | string | `''` | Plugin slug override passed to Plugin Check. |
| `plugin-check-exclude-files` | string | `''` | Newline-separated excluded files. |
| `plugin-check-exclude-directories` | string | `''` | Newline-separated excluded directories. |
| `plugin-check-exclude-checks` | string | shared list | Overrides the shared excluded checks. |
| `php-timeout-minutes` | number | `15` | Timeout for each PHP matrix job. |
| `translations-timeout-minutes` | number | `10` | Timeout for the translations job. |

### `wp-js-ci.yml` - WordPress plugin JS lint

Runs `npm ci` followed by an optional high-severity audit and the consumer's
lint/build command. Node 24 remains the default; consumers can instead use a
version file such as `.nvmrc`.

```yaml
name: JS Lint
"on":
  push: { branches: [main], paths: ['**/*.js', '**/*.ts', '**/*.css', 'package.json', 'package-lock.json', 'biome.json'] }
  pull_request: { paths: ['**/*.js', '**/*.ts', '**/*.css', 'package.json', 'package-lock.json', 'biome.json'] }
  workflow_dispatch:
permissions: { contents: read }
jobs:
  lint:
    uses: oszuidwest/.github-templates/.github/workflows/wp-js-ci.yml@v2
    with:
      npm-audit: true
```

Inputs:

| Input | Type | Default | Notes |
|-------|------|---------|-------|
| `node-version` | string | `24` | Used when `node-version-file` is empty. |
| `node-version-file` | string | `''` | Optional file such as `.nvmrc`. |
| `npm-audit` | boolean | `false` | Fails on high or critical npm advisories. |
| `lint-command` | string | `npm run lint` | Consumer lint/build command. |
| `timeout-minutes` | number | `15` | Job timeout. |

### `wp-tests.yml` - WordPress plugin PHPUnit

Runs a configurable PHP matrix. Coverage is disabled by default; set
`coverage-php` to run the consumer's coverage command on one PHP version and
upload the resulting report.

```yaml
name: Tests
"on":
  push: { branches: [main], paths: ['**/*.php', 'composer.*', 'tests/**'] }
  pull_request: { paths: ['**/*.php', 'composer.*', 'tests/**'] }
  workflow_dispatch:
permissions: { contents: read }
jobs:
  phpunit:
    uses: oszuidwest/.github-templates/.github/workflows/wp-tests.yml@v2
    with:
      coverage-php: '8.4'
```

Inputs:

| Input | Type | Default | Notes |
|-------|------|---------|-------|
| `php-versions` | string | `["8.3", "8.4"]` | JSON PHP version array. |
| `test-command` | string | `composer test` | Regular test command. |
| `coverage-php` | string | `''` | Matrix version used for coverage; empty disables it. |
| `coverage-command` | string | `composer coverage` | Coverage command. |
| `coverage-artifact` | string | `coverage.xml` | Uploaded report path. |
| `timeout-minutes` | number | `15` | Timeout for each matrix job. |

### `wp-release.yml` - WordPress plugin release

Manual `workflow_dispatch`. Reads the `Version:` header from `<plugin-slug>.php`, compares against the latest git tag, and tags + zips + publishes when bumped (or when `force: true`). Translations are auto-compiled when `languages/*.po` exists.

```yaml
name: Release
"on":
  workflow_dispatch:
    inputs:
      force:
        description: 'Release even if the version was not bumped'
        type: boolean
        default: false
permissions: { contents: read }
jobs:
  release:
    uses: oszuidwest/.github-templates/.github/workflows/wp-release.yml@v2
    with:
      force: ${{ inputs.force }}
      # plugin-slug: zuidwest-cache-manager  # override only if slug != repo name
    permissions:
      contents: write
```

Inputs:

| Input | Type | Required | Default | Notes |
|-------|------|----------|---------|-------|
| `plugin-slug` | string | no | repo name | Used as zip rootdir, zip filename, and expected main file (`<slug>.php` at repo root). Override only when slug differs from repo name. |
| `force` | boolean | no | `false` | Release even when the version is not bumped. |

Outputs:

| Output | Notes |
|--------|-------|
| `version` | Version extracted from the plugin header. |
| `released` | `'true'` when a GitHub release was created, `'false'` when skipped. |

**Fixed release behavior:**

- Version regex: `^[0-9]+\.[0-9]+\.[0-9]+(-(alpha|beta|rc)\.[0-9]+)?$`. Pre-release versions (`X.Y.Z-beta.N`, etc.) are auto-marked `prerelease` on the GitHub release.
- Production build: PHP 8.3 + a non-interactive optimized Composer install.
- Translations: `msgfmt` runs over `languages/*.po` when any exist. `vendor/` and the compiled `.mo` files are bundled into the zip.
- rsync exclude list (kept identical across consumers): `release/`, `.git/`, `.github/`, `.claude/`, `node_modules/`, `composer.json`, `composer.lock`, `package.json`, `package-lock.json`, `biome.json`, `phpcs.xml`, `phpcs.xml.dist`, `phpstan.neon`, `phpstan-bootstrap.php`, `phpstan-bootstrap.stub`, `*.log`, `.gitignore`, `CLAUDE.md`, `AGENTS.md`, `.DS_Store`. `vendor/` and `README.md` are kept in the zip.

The release job tags the current commit, builds `<slug>-<version>.zip`, creates
a SHA-256 checksum, and publishes both files with generated release notes. If
the latest tag already matches the header version and `force` is `false`, the
workflow exits with a notice and the `released` output is `'false'`.

The `plugin-slug` input exists for plugins where the slug currently differs from the repo name (`zw-cacheman` -> `zuidwest-cache-manager`, `zw-liveblog` -> `zuidwest-liveblog`, `zw-gr26-wp` -> `zw-gr26`). Those plugins have open rename issues; once landed, they can drop the override.

### Composing custom pre-release gates

`go-release.yml` is build-and-ship only - it does not run tests, lint, or fmt. The assumption is that PR CI already gated the merge into main, so a tag on main is releasable by construction. This is the standard "trust main" model: don't repeat verification work in the release path.

If a consumer needs an extra pre-release check that PR CI doesn't cover (integration tests against a live service, smoke tests, etc.), gate the release on it via `needs:` instead of asking the template to grow another input:

```yaml
name: Release
"on":
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

The custom job lives in the consumer repo, the template stays small, and
`needs:` short-circuits the release if the gate fails. On manual releases of an
existing `vX.Y.Z` tag, the release workflow also verifies that the tag points at
the same commit as the dispatch ref. That prevents a gate on `main` from
accidentally approving binaries rebuilt from an older tag. If you must rebuild
an existing tag, set `allow-tag-rebuild: true` on the release job and treat that
run as an explicit bypass of caller-side gates for the tag commit.

## Maintenance contract

- **Versioning:** semver. Patch for compatible bug fixes and documentation clarifications, minor for additive optional inputs, major for breaking changes (renamed/removed inputs, permission model changes, or default changes that break consumers). Each major line keeps an immutable `vX.Y.Z` and a moving `vX` tag; consumers pin to `@vX` to follow fixes, `@vX.Y.Z` to freeze.
- **Active major lines:** `v2` only. The `v1` line was retired in 2026-05 (zero remaining consumers); its tags no longer resolve.
- **Release tags:** after a validated merge to `main`, tag the exact merge commit and move the major tag:
  ```bash
  VERSION=v2.0.1
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
- **Action-pinning exceptions:** major-pinning is the default. Some upstream actions don't publish a moving major tag - for those, pin to the most specific tag they publish:
  - `ludeeus/action-shellcheck` - patch tag (e.g. `@2.0.0`); upstream has no `@v2`/`@2`.
  - `aquasecurity/trivy-action` - release tag (e.g. `@0.36.0`); upstream uses `0.x.0` schema, no major tag.
- **Drift before templating:** new copy-paste templates only after audiologger-style hand-alignment proves the shape on at least two consumers. Don't template a shape that hasn't stabilized.

### Migrating `go-release.yml` from v1 to v2

> **Note:** the `v1` tag was retired in 2026-05 - `go-release.yml@v1` no longer resolves. The steps below are kept as historical reference for anyone whose repo still carries v1 syntax. Replace `@v1` with `@v2` and follow the migration before pushing.

v2 dropped the nested `docker` job and the four inputs that fed it (`enable-docker`, `image-labels`, `dockerfile-path`, `platforms`). Consumers that publish a Docker image now add a second job that calls `docker-publish.yml` directly.

For a consumer currently calling `go-release.yml@v1` with `enable-docker: true`:

1. Bump `@v1` → `@v2` on the release job.
2. Drop `enable-docker`, `image-labels`, `dockerfile-path`, `platforms` from the release call.
3. Drop `packages: write` from the release job's permissions (`contents: write` is enough now).
4. Add a second job that calls `docker-publish.yml@v2` with `needs: release`, gated on `needs.release.outputs.is_release == 'true'`, and forwards `version` from the release outputs. See "Releasing with a Docker image" for the full snippet.

For a consumer currently calling `go-release.yml@v1` with `enable-docker: false`:

1. Bump `@v1` → `@v2`.
2. Drop `packages: write` from the release job's permissions - the parse-time requirement is gone.

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Hadolint can't find Dockerfile | Check `DOCKERFILE_PATH` in workflow |
| Security scan fails | Dockerfile must be buildable |
| Compose warning about env vars | Expected behavior, no action needed |
| Shell workflow doesn't run | Only triggers on `.sh` changes |
| Reusable workflow can't see secrets | `workflow_call` only forwards `GITHUB_TOKEN`; declare other secrets explicitly |
