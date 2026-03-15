# Copilot Instructions

## What This Repo Does

This is a Docker Compose–based Jenkins environment used to test the [explain-error Jenkins plugin](https://plugins.jenkins.io/explain-error/). There is no application source code — the entire "project" is Jenkins infrastructure configuration.

## Starting and Stopping

```bash
# Start Jenkins (builds image on first run or after changes)
docker compose up --build

# Stop and remove containers (preserves jenkins_home volume)
docker compose down

# Start fresh (destroys all job history and config)
docker compose down -v
```

Jenkins will be available at http://localhost:8080 with `admin` / `admin`.

Override the password:
```bash
JENKINS_ADMIN_PASSWORD=mysecret docker compose up --build
```

## Architecture

All configuration is **Infrastructure as Code** — no manual Jenkins setup required:

- **`jenkins/Dockerfile`** — Builds a custom `jenkins/jenkins:lts-jdk17` image. Plugins are installed at image build time via `jenkins-plugin-cli`. JCasC config is copied to `/casc/casc.yaml` (outside `jenkins_home`) so it is always applied even when the named volume is mounted.
- **`jenkins/plugins.txt`** — Declares all plugins. Add new plugins here and rebuild.
- **`jenkins/casc.yaml`** — Single source of truth for Jenkins system config AND all jobs (via Job DSL scripts embedded inline). Modifying jobs means editing this file, not the Jenkins UI.

## Pre-Configured Jobs

| Folder | Job | Behavior |
|--------|-----|----------|
| Team A | Build Pipeline | Succeeds; triggers Deploy Pipeline downstream |
| Team A | Deploy Pipeline | Deploys to staging → runs smoke tests → deploys to prod |
| Team B | Build Pipeline | **Intentionally fails** (simulates a Java compile error) to exercise the explain-error plugin |
| Team B | Integration Test Pipeline | **Intentionally fails** (simulates a DB connection error) |

## Key Conventions

- **All jobs are defined in `casc.yaml`** using inline Job DSL `script` blocks — there are no `Jenkinsfile`s in external SCM repos.
- **Pipeline scripts use `sandbox(true)`** — scripts run in the Groovy sandbox; no script approval needed.
- **Failing jobs use `exit 1`** after printing realistic error messages to stdout — the explain-error plugin reads this output to generate explanations.
- **The `jenkins_home` named volume** persists job history across `docker compose down`. Always use `docker compose down -v` to reset state entirely.
- **`CASC_JENKINS_CONFIG` env var** points to `/casc/casc.yaml` inside the container — changing `casc.yaml` requires a rebuild and restart.

## Adding or Modifying Jobs

Edit `jenkins/casc.yaml`, then rebuild:

```bash
docker compose up --build
```

Each job is a `- script: |` block under the `jobs:` key, written in Job DSL Groovy syntax.
