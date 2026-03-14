# test-explain-error-plugin

A Docker Compose–based Jenkins environment used to test the
[explain-error Jenkins plugin](https://plugins.jenkins.io/explain-error/).

## Quick Start

```bash
docker compose up --build
```

Jenkins will be available at <http://localhost:8080>.

| Credential | Value |
|---|---|
| Username | `admin` |
| Password | `admin` (override with `JENKINS_ADMIN_PASSWORD` env var) |

## What Is Pre-Configured

All configuration follows the **Infrastructure as Code** strategy using
[Jenkins Configuration as Code (JCasC)](https://www.jenkins.io/projects/jcasc/)
and the [Job DSL plugin](https://plugins.jenkins.io/job-dsl/).

### Folder: Team A

| Job | Role |
|---|---|
| **Build Pipeline** | Upstream – compiles and runs unit tests, then triggers **Deploy Pipeline** on success |
| **Deploy Pipeline** | Downstream – deploys to staging, runs smoke tests, deploys to production |

### Folder: Team B

| Job | Role |
|---|---|
| **Build Pipeline** | Intentionally fails during compilation to exercise error explanation |
| **Integration Test Pipeline** | Downstream – runs integration tests (also contains a failing step for testing) |

## Directory Layout

```
.
├── docker-compose.yml
└── jenkins/
    ├── Dockerfile      # Custom Jenkins LTS image with all plugins pre-installed
    ├── plugins.txt     # Plugin list consumed by jenkins-plugin-cli
    └── casc.yaml       # JCasC + Job DSL configuration (folders, pipelines, triggers)
```

## Customisation

* **Admin password** – set the `JENKINS_ADMIN_PASSWORD` environment variable before starting:

  ```bash
  JENKINS_ADMIN_PASSWORD=mysecret docker compose up --build
  ```

* **Add / modify jobs** – edit `jenkins/casc.yaml` and rebuild the image:

  ```bash
  docker compose up --build
  ```

* **Persist Jenkins home** – a named Docker volume (`jenkins_home`) is used, so job
  history and configuration survive container restarts.  To start fresh, remove the volume:

  ```bash
  docker compose down -v
  ```
