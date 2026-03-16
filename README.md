# test-explain-error-plugin

A Docker Compose–based Jenkins environment used to test the
[explain-error Jenkins plugin](https://plugins.jenkins.io/explain-error/).

## Quick Start

```bash
export AI_API_KEY=your-openai-api-key
docker compose up --build
```

Jenkins will be available at <http://localhost:8080>.

| Credential | Value |
|---|---|
| Username | `admin` |
| Password | `admin` (override with `JENKINS_ADMIN_PASSWORD` env var) |

> **`AI_API_KEY` is required.** All test pipelines call `explainError()`, which
> needs an AI provider API key to generate explanations. Without it the pipelines
> will still fail as expected, but no AI explanation will appear in the sidebar.

## What Is Pre-Configured

All configuration follows the **Infrastructure as Code** strategy using
[Jenkins Configuration as Code (JCasC)](https://www.jenkins.io/projects/jcasc/)
and the [Job DSL plugin](https://plugins.jenkins.io/job-dsl/).

There are two folders, each with a **Run All Cases** master pipeline that triggers
all case jobs in parallel:

### Folder: Team A – Folder-level AI Provider

Verifies every `explainError()` parameter combination. The folder has its own
AI provider configured at folder level (overrides the global config).

| Job | `explainError()` call | Simulated error |
|---|---|---|
| **Case 01 - Default** | `explainError()` | command not found |
| **Case 02 - maxLines** | `explainError(maxLines: 30)` | 50-line OOM log (last 30 sent to AI) |
| **Case 03 - logPattern** | `explainError(logPattern: '(?i)(error\|exception\|failed)')` | mixed INFO + ERROR test output |
| **Case 04 - language** | `explainError(language: '中文')` | DB connection refused |
| **Case 05 - customContext** | `explainError(customContext: '...')` | K8s OOMKilled deploy failure |
| **Case 06 - All Parameters** | all four params combined | Java compile error |
| **Case 07 - Return Value** | `def e = explainError(); echo e` | NullPointerException stack trace |
| **Run All Cases** | triggers Case 01–07 in parallel | — |

### Folder: Team B – Global-level AI Provider

Verifies that the global AI provider config (set in Manage Jenkins) works.
No folder-level AI override is applied.

| Job | Purpose |
|---|---|
| **Case 01 - Default** | Same scenario as Team A Case 01; uses global AI provider |
| **Case 07 - Return Value** | Same scenario as Team A Case 07; uses global AI provider |
| **Run All Cases** | Triggers both cases in parallel |

## Validating a New Plugin Release

1. Set your API key and start Jenkins:
   ```bash
   export AI_API_KEY=your-openai-api-key
   docker compose down -v && docker compose up --build
   ```
2. Open <http://localhost:8080> and run **Team A / Run All Cases**.
3. Once all jobs finish, open each case job and check the sidebar for an AI explanation.
4. Run **Team B / Run All Cases** to confirm the global-level provider works too.
5. If all sidebars show explanations → the plugin release is good to publish.

## Directory Layout

```
.
├── docker-compose.yml
└── jenkins/
    ├── Dockerfile      # Custom Jenkins LTS image with all plugins pre-installed
    ├── plugins.txt     # Plugin list consumed by jenkins-plugin-cli
    └── casc.yaml       # JCasC + Job DSL — single source of truth for all config and jobs
```

## Customisation

* **Admin password** – set `JENKINS_ADMIN_PASSWORD` before starting:
  ```bash
  JENKINS_ADMIN_PASSWORD=mysecret docker compose up --build
  ```

* **Add a new test case** – add a `- script: |` block to `jenkins/casc.yaml` and
  add the job name to the `cases` list in the appropriate `Run All Cases` pipeline,
  then rebuild:
  ```bash
  docker compose up --build
  ```

* **Fresh start** – destroy all job history and config:
  ```bash
  docker compose down -v
  ```
