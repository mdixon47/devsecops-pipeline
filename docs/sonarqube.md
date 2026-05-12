# SonarQube Integration

This document describes the SonarQube static-analysis integration for the
`devsecops-pipeline` repository and tracks known issues with the current
deployment.

## Overview

| Item | Value |
|---|---|
| Workflow file | `.github/workflows/sonarqube.yml` |
| Project config | `sonar-project.properties` (repo root) |
| Scan action | `SonarSource/sonarqube-scan-action@v6` |
| Scanner CLI | `sonar-scanner-cli` 7.2.0.5079 (provided by the action) |
| JDK | Temurin 17 (`actions/setup-java@v4`) |
| Triggered on | `push` to `main`, `pull_request` to `main` (`opened`, `synchronize`, `reopened`) |
| Path filters | `paths-ignore: ['docs/**', '**/*.md']` on both triggers (docs-only changes skip the scan) |
| Concurrency | `sonarqube-${{ github.ref }}`, `cancel-in-progress: true` |
| Quality gate | `sonar.qualitygate.wait=true` (job fails if gate fails) |

## Project configuration

`sonar-project.properties` at the repo root:

```properties
sonar.projectKey=devsecblueprint_devsecops-pipeline
sonar.projectName=devsecblueprint/devsecops-pipeline
sonar.organization=devsecblueprint
sonar.sources=app,scripts,terraform,ansible,gitops
sonar.sourceEncoding=UTF-8
sonar.exclusions=**/node_modules/**,**/.terraform/**,**/*.tfstate,**/*.tfstate.*,**/dist/**,**/build/**,**/__pycache__/**,**/*.pyc,juice-shop/**
sonar.python.version=3.11
```

Notes:

- `sonar.projectKey` follows the SonarCloud `{org}_{repo}` convention,
  scoped under the `devsecblueprint` organization. Each repository
  analyzed under this organization should have its own project key.
- `sonar.organization=devsecblueprint` is required by SonarCloud to
  resolve which organization owns the project.
- `juice-shop/**` is excluded because the local copy only contains a
  `Dockerfile`; the upstream OWASP Juice Shop source is not vendored here.
- `**/.terraform/**` and `*.tfstate*` are excluded to avoid scanning
  provider plugins and state files.

### Fallback: roll analysis under the existing curriculum project

If the dedicated `devsecblueprint_devsecops-pipeline` project cannot be
provisioned, results can be temporarily published under the existing
`devsecblueprint_devsecblueprint` project by setting:

```properties
sonar.projectKey=devsecblueprint_devsecblueprint
sonar.organization=devsecblueprint
```

This is **not recommended** for sustained use — it mixes metrics from
two distinct codebases under one project — but is documented here as a
last-resort fallback.

## Required GitHub secrets

| Secret | Purpose |
|---|---|
| `SONAR_TOKEN` | User token with **Execute Analysis** permission on the `devsecblueprint_devsecops-pipeline` project under the `devsecblueprint` organization |
| `SONAR_HOST_URL` | Base URL of the SonarQube / SonarCloud server (e.g. `https://sonarcloud.io`) |

Both are referenced as `${{ secrets.* }}` in the workflow's `Run SonarQube
Scan` step. `SONAR_HOST_URL` is also surfaced in the job summary.

## Detected languages / sensors

From the most recent scan, 97 files were indexed across 3 languages:

- Python (`app/main.py`)
- Terraform (11 files under `terraform/`)
- YAML / Kubernetes IaC (54 files under `gitops/`)
- Plus the global Text & Secrets sensor (94 files)

## Known issues

### 1. Upload rejected by SonarQube server (resolved)

**Status:** Resolved — root cause was an incorrect project key.

**Original error (runs `25642495289`, `25643191225`, `25644064551`):**

```
ERROR You're not authorized to analyze this project or the project
doesn't exist on SonarQube and you're not authorized to create it.
Please contact an administrator.
INFO  EXECUTION FAILURE
Action failed: sonar-scanner failed with exit code 3
```

**Root cause:** the analysis was being sent under `sonar.projectKey=devsecops-pipeline`,
which is not a project the configured `SONAR_TOKEN` has access to. The
token is scoped to the `devsecblueprint` organization, where projects
follow the `{org}_{repo}` key convention.

**Fix:** updated `sonar-project.properties` to use the correct
`devsecblueprint`-scoped key and organization:

```properties
sonar.projectKey=devsecblueprint_devsecops-pipeline
sonar.organization=devsecblueprint
```

After applying, re-run the latest workflow:

```bash
gh run rerun <run-id> --repo mdixon47/devsecops-pipeline --failed
```

## Change history

| Date | Change | Commit |
|---|---|---|
| 2026-05-10 | Initial workflow + `sonar-project.properties` added (replaced extensionless `github-sonarqube-workflow` that GitHub Actions silently ignored); action pinned to `@v5` | `b43f8db` |
| 2026-05-10 | Bumped `sonarqube-scan-action` `@v5` → `@v6` (v5 flagged by GitHub as deprecated / containing a security vulnerability) | `2feef23` |
| 2026-05-10 | Added `paths-ignore` (`docs/**`, `**/*.md`) on `push` and `pull_request` triggers so docs-only changes do not consume scans | `9a9c6a9` |
| 2026-05-11 | Corrected `sonar.projectKey` to `devsecblueprint_devsecops-pipeline` and added `sonar.organization=devsecblueprint` so analysis lands in the right SonarCloud organization (root cause of prior auth failures) | _pending_ |
