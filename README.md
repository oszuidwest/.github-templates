# GitHub Actions Templates

Standard workflows for Docker-based repositories at oszuidwest.

## Quick Start

```bash
# Workflows
cp workflow-templates/docker-quality.yml   .github/workflows/
cp workflow-templates/docker-security.yml  .github/workflows/
cp workflow-templates/shell-quality.yml    .github/workflows/
cp workflow-templates/cleanup-ghcr.yml     .github/workflows/

# Config
cp config-templates/.hadolint.yaml         ./
cp config-templates/dependabot.yml         .github/
```

## Workflows

| Workflow | Purpose | Fails on |
|----------|---------|----------|
| docker-quality | Dockerfile, YAML and Compose linting | Lint errors |
| docker-security | Vulnerability scanning (Trivy) | CRITICAL CVEs |
| shell-quality | ShellCheck (only on .sh changes) | Lint errors |
| cleanup-ghcr | Remove untagged images (weekly) | Never |

## Customization

**Dockerfile not in root?**
```yaml
env:
  DOCKERFILE_PATH: "docker/Dockerfile"
```

**Multiple container images?** See matrix variant in `cleanup-ghcr.yml`.

**Adjust Hadolint rules?** Edit `.hadolint.yaml`, not the workflow.

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Hadolint can't find Dockerfile | Check `DOCKERFILE_PATH` in workflow |
| Security scan fails | Dockerfile must be buildable |
| Compose warning about env vars | Expected behavior, no action needed |
| Shell workflow doesn't run | Only triggers on `.sh` changes |
