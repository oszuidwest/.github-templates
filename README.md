# Docker Quality Workflow Templates

Standaard GitHub Actions workflows voor Docker-gebaseerde repositories van oszuidwest.

## Quick Start

Kopieer de benodigde bestanden naar je repository:

```bash
# Workflows
cp workflow-templates/docker-quality.yml    YOUR_REPO/.github/workflows/
cp workflow-templates/docker-security.yml   YOUR_REPO/.github/workflows/
cp workflow-templates/shell-quality.yml     YOUR_REPO/.github/workflows/
cp workflow-templates/cleanup-ghcr.yml      YOUR_REPO/.github/workflows/

# Configuration
cp config-templates/.hadolint.yaml          YOUR_REPO/
cp config-templates/dependabot.yml          YOUR_REPO/.github/
```

## Workflows

### docker-quality.yml
Linting en validatie voor Docker projecten.

| Check | Tool | Beschrijving |
|-------|------|--------------|
| Dockerfile | Hadolint | Best practices en security checks |
| YAML | yamllint | Syntax en formatting validatie |
| Docker Compose | docker compose config | Compose file validatie |

**Triggers:** push naar main, pull requests, handmatig

### docker-security.yml
Security scanning voor container images.

| Check | Tool | Beschrijving |
|-------|------|--------------|
| Vulnerabilities | Trivy | CVE scanning (CRITICAL/HIGH) |
| Secrets | Trivy | Hardcoded secrets detection |
| SARIF Upload | CodeQL | GitHub Security tab integratie |

**Triggers:** push naar main, pull requests, wekelijks (zondag 4:00 UTC), handmatig

**Belangrijk:** Workflow faalt alleen bij CRITICAL vulnerabilities. HIGH vulnerabilities worden gerapporteerd maar blokkeren niet.

### shell-quality.yml
Shell script linting.

| Check | Tool | Beschrijving |
|-------|------|--------------|
| Shell Scripts | ShellCheck | Syntax en best practices |

**Triggers:** Alleen wanneer `.sh` bestanden wijzigen (path filter)

### cleanup-ghcr.yml
Automatische cleanup van untagged container images.

**Triggers:** Wekelijks (zondag 3:00 UTC), handmatig

## Configuratie

### .hadolint.yaml
Hadolint configuratie voor Dockerfile linting.

**Standaard genegeerde regels:**
- `DL3008` - apt-get version pinning (praktisch voor security updates)
- `DL3009` - apt-get list cleanup (afgehandeld in multi-stage builds)

**Belangrijk:** Zet ALLE ignore regels in `.hadolint.yaml`, NIET in de workflow file.

### dependabot.yml
Automatische dependency updates.

**Standaard ecosystems:**
- GitHub Actions (altijd)
- Docker (altijd)

**Optionele ecosystems:** Go, Node.js, Python, Rust (uncomment naar behoefte)

## Customization

### Dockerfile in subdirectory
Als je Dockerfile niet in de root staat:

```yaml
# In docker-quality.yml en docker-security.yml
env:
  DOCKERFILE_PATH: "docker/Dockerfile"
```

### Multiple container images
Voor repos met meerdere images, gebruik de matrix variant in `cleanup-ghcr.yml`:

```yaml
strategy:
  matrix:
    package:
      - my-app
      - my-app-full
      - my-app-minimal
```

## Best Practices

### Action Versions
Gebruik consistente versies in alle workflows:

| Action | Versie |
|--------|--------|
| actions/checkout | @v6 |
| docker/setup-buildx-action | @v3 |
| docker/build-push-action | @v6 |
| aquasecurity/trivy-action | @0.33.0 |
| hadolint/hadolint-action | @v3.3.0 |
| github/codeql-action/upload-sarif | @v4 |

### Permissions
Gebruik altijd expliciete permissions:

```yaml
permissions:
  contents: read
  packages: write        # Alleen voor push naar registry
  security-events: write # Alleen voor SARIF upload
```

### Caching
Gebruik GitHub Actions cache voor Docker builds:

```yaml
cache-from: type=gha
cache-to: type=gha,mode=max
```

## Repository-Specific Notes

| Repository | Customization |
|------------|---------------|
| zwfm-liquidsoap | Houd bestaande liquidsoap-quality.yml |
| zwfm-odrbuilds | DOCKERFILE_PATH: "docker/Dockerfile" |
| zwfm-aerontoolbox | Test Dockerfile niet in linting |

## Troubleshooting

### Hadolint vindt Dockerfile niet
Check of `DOCKERFILE_PATH` correct is ingesteld. Default is `"Dockerfile"` in de root.

### Security scan faalt altijd
Check of de Dockerfile buildable is. De security scan moet eerst een image bouwen.

### Docker Compose validatie warning
Dit is normaal als je repo environment variables nodig heeft.

### Shell quality workflow draait niet
Dit is correct gedrag - de workflow triggert alleen op `.sh` bestandswijzigingen.
