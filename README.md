# Qualys GitHub Actions

> ⚠️ **Unofficial project.** This is a personal project and is not affiliated with, endorsed by, or supported by Qualys, Inc.

GitHub Actions for security scanning using Qualys Container Security. Scan container images and code repositories for vulnerabilities directly in your CI/CD pipelines.

## Features

- **Container Scanning**: Scan Docker/OCI container images for vulnerabilities
- **Code Scanning (SCA)**: Scan source code for vulnerable dependencies
- **Secrets Detection**: Optional scanning for exposed secrets
- **SBOM Generation**: Generate Software Bill of Materials (SPDX/CycloneDX)
- **GitHub Security Integration**: Automatic upload of results to GitHub Security tab
- **GitHub Issue Creation**: Automatically create issues for discovered vulnerabilities
- **Flexible Thresholds**: Set vulnerability limits or use Qualys cloud policies
- **Organization-wide Deployment**: Use org secrets and reusable workflows for seamless multi-repo deployment

## Quick Start

### 1. Configure Organization Secrets (Recommended)

For organization-wide usage, configure secrets at the organization level:

1. Go to your GitHub Organization → **Settings** → **Secrets and variables** → **Actions**
2. Create **Organization secret**: `QUALYS_ACCESS_TOKEN` (your Qualys API token)
3. Create **Organization variable**: `QUALYS_POD` (e.g., `US1`, `US2`, `US3`, `EU1`)

This allows all repositories in your organization to use Qualys scanning without individual configuration.

### 2. Add to Your Workflow

#### Code Scan

```yaml
name: Security Scan

on:
  push:
    branches: [main]
  pull_request:

jobs:
  scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
      issues: write

    steps:
      - uses: actions/checkout@v4

      - name: Qualys Code Scan
        uses: qualys/qualys-github/code-scan@v1
        with:
          qualys_access_token: ${{ secrets.QUALYS_ACCESS_TOKEN }}
          qualys_pod: ${{ vars.QUALYS_POD }}
          max_critical: 0
          max_high: 5
          create_issues: true
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

#### Container Scan

```yaml
name: Container Security

on:
  push:
    branches: [main]

jobs:
  build-and-scan:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write

    steps:
      - uses: actions/checkout@v4

      - name: Build Image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Qualys Container Scan
        uses: qualys/qualys-github/container-scan@v1
        with:
          qualys_access_token: ${{ secrets.QUALYS_ACCESS_TOKEN }}
          qualys_pod: ${{ vars.QUALYS_POD }}
          image_id: myapp:${{ github.sha }}
          max_critical: 0
          max_high: 3
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

## Organization-Wide Deployment

### Option 1: Reusable Workflows

Create a centralized workflow in your `.github` repository that all repos can call:

**In any repository:**
```yaml
name: Security Scan

on: [push, pull_request]

jobs:
  code-scan:
    uses: your-org/.github/.github/workflows/qualys-code-scan.yml@main
    secrets:
      QUALYS_ACCESS_TOKEN: ${{ secrets.QUALYS_ACCESS_TOKEN }}
```

### Option 2: Required Workflows (GitHub Enterprise)

For GitHub Enterprise, configure required workflows at the organization level to automatically run scans on all repositories.

### Option 3: Starter Workflows

Add workflow templates to your organization's `.github` repository under `workflow-templates/` so they appear when creating new workflows.

## Configuration

### Container Scan Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `qualys_access_token` | Yes | - | Qualys API access token |
| `qualys_pod` | Yes | `US3` | Qualys platform POD |
| `image_id` | Yes | - | Container image to scan |
| `storage_driver` | No | `none` | Storage driver for local images |
| `platform` | No | - | Multi-arch platform (e.g., `linux/amd64`) |
| `use_policy_evaluation` | No | `false` | Use Qualys cloud policy |
| `policy_tags` | No | - | Comma-separated policy tags |
| `max_critical` | No | `0` | Max critical vulnerabilities (-1 = unlimited) |
| `max_high` | No | `0` | Max high vulnerabilities |
| `max_medium` | No | `-1` | Max medium vulnerabilities |
| `max_low` | No | `-1` | Max low vulnerabilities |
| `scan_secrets` | No | `false` | Enable secrets detection |
| `scan_timeout` | No | `300` | Scan timeout in seconds |
| `upload_sarif` | No | `true` | Upload to GitHub Security tab |
| `continue_on_error` | No | `false` | Continue on threshold violation |
| `create_issues` | No | `false` | Create GitHub Issues for vulnerabilities |
| `issue_min_severity` | No | `4` | Min severity for issues (5=crit, 4=high, 3=med, 2=low) |
| `issue_labels` | No | - | Additional labels for issues |
| `issue_assignees` | No | - | GitHub usernames to assign |

### Code Scan Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `qualys_access_token` | Yes | - | Qualys API access token |
| `qualys_pod` | Yes | `US3` | Qualys platform POD |
| `scan_path` | No | Repo root | Directory to scan |
| `exclude_dirs` | No | - | Directories to exclude |
| `exclude_files` | No | - | File patterns to exclude |
| `use_policy_evaluation` | No | `false` | Use Qualys cloud policy |
| `policy_tags` | No | - | Comma-separated policy tags |
| `max_critical` | No | `0` | Max critical vulnerabilities |
| `max_high` | No | `0` | Max high vulnerabilities |
| `max_medium` | No | `-1` | Max medium vulnerabilities |
| `max_low` | No | `-1` | Max low vulnerabilities |
| `scan_secrets` | No | `false` | Enable secrets detection |
| `offline_scan` | No | `false` | Offline scan (no cloud upload) |
| `generate_sbom` | No | `false` | Generate SBOM |
| `sbom_format` | No | `spdx` | SBOM format (spdx, cyclonedx, both) |
| `scan_timeout` | No | `300` | Scan timeout in seconds |
| `upload_sarif` | No | `true` | Upload to GitHub Security tab |
| `continue_on_error` | No | `false` | Continue on threshold violation |
| `create_issues` | No | `false` | Create GitHub Issues |
| `issue_min_severity` | No | `4` | Min severity for issues |
| `issue_labels` | No | - | Additional labels |
| `issue_assignees` | No | - | Users to assign |

### Outputs

| Output | Description |
|--------|-------------|
| `vulnerability_count` | Total vulnerabilities found |
| `critical_count` | Critical vulnerabilities |
| `high_count` | High vulnerabilities |
| `medium_count` | Medium vulnerabilities |
| `low_count` | Low vulnerabilities |
| `policy_result` | Policy result: ALLOW, DENY, AUDIT, NONE |
| `scan_passed` | Whether scan passed (true/false) |
| `sarif_path` | Path to SARIF report |
| `json_path` | Path to JSON report |
| `sbom_path` | Path to SBOM (code scan only) |
| `issues_created` | Number of issues created |

## Threshold vs Policy Evaluation

### Manual Thresholds

Set maximum allowed vulnerabilities per severity:

```yaml
- uses: qualys/qualys-github/code-scan@v1
  with:
    qualys_access_token: ${{ secrets.QUALYS_ACCESS_TOKEN }}
    qualys_pod: ${{ vars.QUALYS_POD }}
    max_critical: 0      # Fail on any critical
    max_high: 3          # Allow up to 3 high
    max_medium: -1       # Unlimited medium
    max_low: -1          # Unlimited low
```

### Qualys Cloud Policies

Use centralized policies defined in Qualys:

```yaml
- uses: qualys/qualys-github/container-scan@v1
  with:
    qualys_access_token: ${{ secrets.QUALYS_ACCESS_TOKEN }}
    qualys_pod: ${{ vars.QUALYS_POD }}
    image_id: myapp:latest
    use_policy_evaluation: true
    policy_tags: production,pci-dss
```

## GitHub Integration

### Security Tab

Results automatically appear in **Security → Code scanning alerts** when `upload_sarif: true`.

### Issue Creation

Enable `create_issues: true` to automatically create GitHub Issues for vulnerabilities. Issues include:
- Severity label
- CVE links
- Package and version info
- Remediation guidance
- Automatic deduplication (won't create duplicates)

## Qualys POD Regions

| POD | Region |
|-----|--------|
| US1 | United States 1 |
| US2 | United States 2 |
| US3 | United States 3 |
| US4 | United States 4 |
| EU1 | Europe 1 |
| EU2 | Europe 2 |
| CA1 | Canada |
| IN1 | India |
| AU1 | Australia |
| UK1 | United Kingdom |
| AE1 | UAE |
| KSA1 | Saudi Arabia |

## Documentation

See the [docs/architecture.md](docs/architecture.md) for detailed architecture diagrams and design documentation.

## Requirements

- **Runner**: Linux x86_64 (ubuntu-latest)
- **Qualys Account**: Container Security module with API access
- **GitHub**: Repository with Actions enabled

## License

MIT
