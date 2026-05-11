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
sonar.projectKey=devsecops-pipeline
sonar.projectName=devsecops-pipeline
sonar.sources=app,scripts,terraform,ansible,gitops
sonar.sourceEncoding=UTF-8
sonar.exclusions=**/node_modules/**,**/.terraform/**,**/*.tfstate,**/*.tfstate.*,**/dist/**,**/build/**,**/__pycache__/**,**/*.pyc,juice-shop/**
sonar.python.version=3.11
```

Notes:

- `juice-shop/**` is excluded because the local copy only contains a
  `Dockerfile`; the upstream OWASP Juice Shop source is not vendored here.
- `**/.terraform/**` and `*.tfstate*` are excluded to avoid scanning
  provider plugins and state files.

## Required GitHub secrets

| Secret | Purpose |
|---|---|
| `SONAR_TOKEN` | User token from the SonarQube server with **Execute Analysis** permission on the `devsecops-pipeline` project |
| `SONAR_HOST_URL` | Base URL of the SonarQube server (e.g. `https://sonarqube.example.com`) |

Both are referenced as `${{ secrets.* }}` in the workflow's `Run SonarQube
Scan` step. `SONAR_HOST_URL` is also surfaced in the job summary.

## Detected languages / sensors

From the most recent scan, 97 files were indexed across 3 languages:

- Python (`app/main.py`)
- Terraform (11 files under `terraform/`)
- YAML / Kubernetes IaC (54 files under `gitops/`)
- Plus the global Text & Secrets sensor (94 files)

## Known issues

### 1. SonarQube server rejects the upload (open)

**Status:** Blocking — the scanner runs to completion locally but the
report upload is denied by the SonarQube server.

**Error (verbatim, runs `25642495289` and `25643191225`):**

```
ERROR You're not authorized to analyze this project or the project
doesn't exist on SonarQube and you're not authorized to create it.
Please contact an administrator.
INFO  EXECUTION FAILURE
Action failed: sonar-scanner failed with exit code 3
```

**Root cause:** server-side, not workflow-side. One of:

- The project `devsecops-pipeline` does not exist on the SonarQube server.
- The project exists but the user that owns `SONAR_TOKEN` lacks
  **Execute Analysis** permission on it.
- Auto-provisioning is disabled and the token's user lacks the global
  **Create Projects** permission.

**Resolution (server-side; pick one):**

1. **Create the project manually**
   SonarQube UI → *Projects* → *Create Project* → *Manually* →
   set **Project key** = `devsecops-pipeline`.
2. **Grant Execute Analysis** on the existing project
   *Project Settings* → *Permissions* → add the token's user/group with
   **Execute Analysis**.
3. **Enable auto-provisioning**
   *Administration* → *Security* → *Global Permissions* → grant
   **Create Projects** to the token's user.
4. **Use an existing project key**
   If a project already exists under a different key, update
   `sonar.projectKey` in `sonar-project.properties` to match it.

After resolving, re-run the latest workflow:

```bash
gh run rerun <run-id> --repo mdixon47/devsecops-pipeline --failed
```

## Change history

| Date | Change | Commit |
|---|---|---|
| 2026-05-10 | Initial workflow + `sonar-project.properties` added (replaced extensionless `github-sonarqube-workflow` that GitHub Actions silently ignored); action pinned to `@v5` | `b43f8db` |
| 2026-05-10 | Bumped `sonarqube-scan-action` `@v5` → `@v6` (v5 flagged by GitHub as deprecated / containing a security vulnerability) | `2feef23` |
| 2026-05-10 | Added `paths-ignore` (`docs/**`, `**/*.md`) on `push` and `pull_request` triggers so docs-only changes do not consume scans | `9a9c6a9` |
