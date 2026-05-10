# Repository Guidelines

## Project Structure & Module Organization

- `README.md` documents the public contract, usage examples, inputs, outputs, and release process.
- `.github/workflows/` contains reusable workflows called with `uses: oszuidwest/.github-templates/.github/workflows/<name>.yml@v2`. Go/Docker reusables (`go-ci.yml`, `go-release.yml`, `docker-publish.yml`) and WordPress reusables (`wp-ci.yml`, `wp-js-ci.yml`, `wp-release.yml`) live side by side.
- `workflow-templates/` contains copy-paste workflow files for consumer repositories.
- `config-templates/` contains supporting config files such as Dependabot and Hadolint templates.

There is no application source tree or test suite. Workflow YAML and documentation examples are the primary deliverables.

## Build, Test, and Development Commands

No build step is required. Validate changes before opening a PR:

- `git diff --check` checks for whitespace errors.
- `yamllint .github/workflows workflow-templates config-templates` validates YAML style if `yamllint` is installed.
- `actionlint .github/workflows/*.yml workflow-templates/*.yml` validates GitHub Actions syntax if `actionlint` is installed.

For behavior changes, copy the edited workflow into a representative consumer repository or call it from a test branch before release.

## Coding Style & Naming Conventions

Use two-space indentation for YAML. Start YAML files with `---`. Keep workflow and job names clear and stable. Prefer kebab-case filenames, for example `go-release.yml` or `docker-security.yml`.

Keep permissions minimal and explicit. Default to `permissions: { contents: read }` unless a step requires more. Pin third-party actions to explicit versions, and preserve existing input names unless making a documented major release.

## Testing Guidelines

Testing is validation-focused. Run YAML and Actions linting for every workflow change. For reusable workflows, verify `workflow_call` inputs, outputs, permissions, and failure behavior against `README.md`.

When changing copy-paste templates, confirm they still work when copied into a consumer repository without repository-specific assumptions beyond those documented in the file.

## Commit & Pull Request Guidelines

Use Conventional Commits as shown in the history: `feat(release): ...`, `fix: ...`, `docs: ...`, or `chore: ...`. Mark breaking changes with `!`, for example `feat(release)!: ...`.

Pull requests should describe affected templates or reusable workflows, explain consumer impact, and link related issues. Include validation evidence such as linter output, a dry run, or a consumer smoke test. Update `README.md` whenever inputs, outputs, permissions, examples, or release behavior change.

## Release & Compatibility Notes

Follow semver. Patch releases are for compatible fixes and docs clarifications, minor releases add optional behavior, and major releases cover breaking workflow contracts. After merging to `main`, tag the exact merge commit and move the matching major tag, as documented in `README.md`.
