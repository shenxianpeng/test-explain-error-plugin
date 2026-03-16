# Copilot Instructions

## What This Repo Does

This is a Docker Compose–based Jenkins environment used to test the [explain-error Jenkins plugin](https://plugins.jenkins.io/explain-error/). There is no application source code — the entire "project" is Jenkins infrastructure configuration.

## Starting and Stopping

```bash
# Set your AI API key (required — passed into Jenkins as AI_API_KEY)
export AI_API_KEY=your-openai-api-key

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

There are two folders. Each folder has a **Run All Cases** master pipeline.

### Team A – Folder-level AI Provider

Tests every `explainError()` parameter combination. The folder carries its own AI provider
configured at folder level via a Job DSL `configure {}` block (class:
`io.jenkins.plugins.explain_error.ExplainErrorFolderProperty`, provider class:
`io.jenkins.plugins.explain_error.provider.OpenAIProvider`).

| Job | `explainError()` call |
|-----|----------------------|
| Case 01 - Default | `explainError()` — all defaults |
| Case 02 - maxLines | `explainError(maxLines: 30)` |
| Case 03 - logPattern | `explainError(logPattern: '(?i)(error\|exception\|failed)')` |
| Case 04 - language | `explainError(language: '中文')` |
| Case 05 - customContext | `explainError(customContext: '...')` — overrides folder/global context |
| Case 06 - All Parameters | all four params: `maxLines`, `logPattern`, `language`, `customContext` |
| Case 07 - Return Value | captures return string: `def e = explainError(); echo e` |
| Run All Cases | triggers Case 01–07 in parallel with `propagate: false` |

### Team B – Global-level AI Provider

Verifies the global `unclassified.explainError` config works. No folder-level override.
Intentionally minimal (2 cases) to reduce maintenance — Team A already covers all parameters.

| Job | Purpose |
|-----|---------|
| Case 01 - Default | same scenario as Team A Case 01, uses global AI provider |
| Case 07 - Return Value | same scenario as Team A Case 07, uses global AI provider |
| Run All Cases | triggers both cases in parallel |

## Key Conventions

- **All jobs are defined in `casc.yaml`** using inline Job DSL `script` blocks — there are no `Jenkinsfile`s in external SCM repos.
- **Pipeline scripts use `sandbox(true)`** — scripts run in the Groovy sandbox; no script approval needed.
- **All case jobs intentionally fail** (`exit 1` after printing a realistic error) — the explain-error plugin reads the console output to generate its AI explanation.
- **`AI_API_KEY` env var** is passed from the host into the container via `docker-compose.yml` and consumed by `casc.yaml` for both the global provider (`${AI_API_KEY}`) and the Team A folder-level provider (`System.getenv('AI_API_KEY')`).
- **The `jenkins_home` named volume** persists job history across `docker compose down`. Always use `docker compose down -v` to reset state entirely.
- **`CASC_JENKINS_CONFIG` env var** points to `/casc/casc.yaml` inside the container — changing `casc.yaml` requires a rebuild and restart.
- **Groovy string quoting in casc.yaml**: pipeline scripts are wrapped in `'''...'''` (single triple-quotes). Use `"""..."""` for nested multi-line strings inside a pipeline script. Never nest `'''...'''` inside another `'''...'''` — it terminates the outer string and causes a `MultipleCompilationErrorsException` at boot.

## Adding a New Test Case

1. Add a `- script: |` block to `jenkins/casc.yaml` under the appropriate folder.
2. Add the job name to the `cases` list in that folder's `Run All Cases` pipeline script.
3. Rebuild:
   ```bash
   docker compose up --build
   ```

## Validating a Plugin Release

```bash
export AI_API_KEY=your-openai-api-key
docker compose down -v && docker compose up --build
```

Then run **Team A / Run All Cases** and **Team B / Run All Cases**.
If all job sidebars show AI explanations, the plugin is safe to release.
