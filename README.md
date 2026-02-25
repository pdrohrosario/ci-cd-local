# 🚀 Local CI/CD Pipeline

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [File Structure](#2-file-structure)
3. [Detailed Components](#3-detailed-components)
4. [Full Pipeline Flow](#4-full-pipeline-flow)
5. [Getting Started](#5-getting-started)
6. [Access & Ports](#6-access--ports)

---

## 1. Architecture Overview

```
                          ┌──────────────────┐
                          │   GitHub Repo    │
                          │  (source code)   │
                          └────────┬─────────┘
                                   │
                          push / webhook / polling
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────┐
│                        ci-cd_ci-cd-network (bridge)             │
│                                                                 │
│   ┌─────────────────────────────────────────────────────┐       │
│   │                JENKINS (:8080)                       │       │
│   │                                                      │       │
│   │   1. Checkout ──▶ Clones code from GitHub            │       │
│   │   2. Build    ──▶ Compiles with Maven (container)    │       │
│   │   3. Test     ──▶ Runs unit tests                    │       │
│   │   4. Image    ──▶ Builds Docker image                │       │
│   │   5. Push     ──▶ Pushes image to local Registry     │       │
│   │                                                      │       │
│   │   📊 Exposes metrics at /prometheus/                 │       │
│   └──────────┬──────────────────────┬────────────────────┘       │
│              │                      │                            │
│         docker build           docker push                       │
│         docker run             docker tag                        │
│              │                      │                            │
│              ▼                      ▼                            │
│   ┌──────────────────┐   ┌─────────────────────┐                │
│   │   Docker Host    │   │  REGISTRY (:5001)    │                │
│   │  (via socket)    │   │                      │                │
│   │                  │   │  Stores Docker       │                │
│   │  Executes builds │   │  images locally      │                │
│   │  and containers  │   │                      │                │
│   └──────────────────┘   └─────────────────────┘                │
│                                                                 │
│   ┌─────────────────────────────────────────────────────┐       │
│   │              OBSERVABILITY                           │       │
│   │                                                      │       │
│   │   ┌──────────────────┐     ┌──────────────────┐      │       │
│   │   │ PROMETHEUS (:9090)│────▶│  GRAFANA (:3000) │      │       │
│   │   │                  │     │                  │      │       │
│   │   │ Collects metrics │     │ Visual           │      │       │
│   │   │ from all         │     │ dashboards for   │      │       │
│   │   │ services every   │     │ real-time        │      │       │
│   │   │ 15 seconds       │     │ monitoring       │      │       │
│   │   └──────────────────┘     └──────────────────┘      │       │
│   └─────────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. File Structure

```
ci-cd/
├── compose.yaml                          # Orchestrates all services
├── Dockerfile.jenkins                    # Custom Jenkins image
├── plugins.txt                           # Pre-installed Jenkins plugins
├── prometheus.yml                        # Metrics collection configuration
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── datasource.yml            # Auto-configures Prometheus in Grafana
└── README.md                             # This document
```

---

## 3. Detailed Components

### 3.1 Jenkins — The CI/CD Orchestrator

**Files:** `compose.yaml` (service `jenkins`) + `Dockerfile.jenkins`

Jenkins is the heart of the pipeline. It is responsible for:
- Listening for GitHub events (push, pull request)
- Executing the `Jenkinsfile` defined in each repository
- Orchestrating the build, test, package, and deploy stages

#### Custom Dockerfile

The official Jenkins image (`jenkins/jenkins:lts`) does **not** include the Docker CLI. Since
our pipeline needs to execute Docker commands (build, push, pull), we build a custom image
that adds:

| Package | Purpose |
|---------|---------|
| `docker-ce-cli` | Enables `docker build`, `docker push`, etc. |
| `docker-compose-plugin` | Enables `docker compose` inside the pipeline |
| `jq` | JSON manipulation in pipeline scripts |
| `wget` | Downloads and healthchecks |
| `curl`, `gnupg`, `ca-certificates` | Required to install the Docker repository |

#### Communication with Docker

```yaml
volumes:
  - /var/run/docker.sock:/var/run/docker.sock
```

Jenkins does **not** run its own Docker daemon. It shares the host machine's Docker daemon
through the **Docker Socket** (`/var/run/docker.sock`). This means:

- When Jenkins runs `docker build`, it uses **your computer's** Docker
- Built images are available on the host Docker daemon
- No Docker-in-Docker (DinD) overhead

#### Environment Variables

```yaml
environment:
  - JAVA_OPTS=-Djenkins.install.runSetupWizard=false    # Skips the initial setup wizard
  - DOCKER_HOST=unix:///var/run/docker.sock              # Points to the host socket
```

#### Healthcheck

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:8080/login"]
  interval: 30s        # Checks every 30 seconds
  timeout: 10s         # Waits up to 10s for a response
  retries: 5           # Tries 5 times before marking as unhealthy
  start_period: 60s    # Waits 60s for Jenkins to initialize (it's heavy)
```

Docker monitors Jenkins' health. If it fails 5 consecutive times, the container is
automatically restarted (`restart: always`).

#### Ports

| Port | Usage |
|------|-------|
| `8080` | Jenkins web interface |
| `50000` | Jenkins agent communication (JNLP). Used if you add remote agents |

---

### 3.2 Jenkins Plugins

**File:** `plugins.txt`

Plugins are automatically installed during the image build via `jenkins-plugin-cli`.
This avoids manual installation through the browser.

#### Pipeline & Workflow
| Plugin | Purpose |
|--------|---------|
| `workflow-aggregator` | **Essential.** Enables declarative and scripted pipelines (Jenkinsfile) |
| `pipeline-stage-view` | Stage visualization in the classic interface |
| `pipeline-graph-view` | Graph-based stage visualization (more modern) |

#### Git & GitHub
| Plugin | Purpose |
|--------|---------|
| `git` | Git operations support (clone, checkout, fetch) |
| `github` | GitHub integration (webhooks, status checks) |
| `github-branch-source` | Automatic branch and PR scanning from GitHub |

#### Docker
| Plugin | Purpose |
|--------|---------|
| `docker-workflow` | Enables `docker.image()` and `docker.build()` in Jenkinsfile |
| `docker-plugin` | Docker management as a build agent |

#### Credentials
| Plugin | Purpose |
|--------|---------|
| `credentials` | Centralized credentials management |
| `credentials-binding` | Injects credentials as environment variables in the pipeline |
| `ssh-credentials` | SSH key support (deploy keys, server access) |

#### UI & Quality of Life
| Plugin | Purpose |
|--------|---------|
| `blueocean` | Modern, visual interface for pipelines |
| `locale` | Localization support |
| `dark-theme` | Dark theme for the interface |

#### Monitoring
| Plugin | Purpose |
|--------|---------|
| `prometheus` | Exposes Jenkins metrics in Prometheus format (`/prometheus/`) |
| `metrics` | Internal Jenkins metrics (JVM, builds, etc.) |

#### Notifications
| Plugin | Purpose |
|--------|---------|
| `mailer` | Email notifications at the end of builds (success/failure) |

---

### 3.3 Local Registry — Docker Image Storage

**File:** `compose.yaml` (service `registry`)

The Registry is a Docker image server running locally. This is where Jenkins sends
(`docker push`) the images built during the pipeline.

#### Why a local Registry?

- **Speed:** Push and pull images without depending on internet
- **Cost:** No pull limits like Docker Hub
- **Control:** Your images stay on your machine, with no external exposure
- **Independence:** The pipeline works even offline

#### How Jenkins pushes images

In the Jenkinsfile, the flow is:

```groovy
// 1. Build the image
docker.build("localhost:5001/my-app:${BUILD_NUMBER}")

// 2. Push to the local registry
docker.image("localhost:5001/my-app:${BUILD_NUMBER}").push()
```

#### Important configuration

```yaml
environment:
  REGISTRY_STORAGE_DELETE_ENABLED: "true"   # Allows deleting old images (cleanup)
```

---

### 3.4 Prometheus — Metrics Collection

**File:** `compose.yaml` (service `prometheus`) + `prometheus.yml`

Prometheus operates on a **pull** model: it goes to each service and "scrapes" metrics
at regular intervals.

#### What is monitored

| Job | Target | Endpoint | What it collects |
|-----|--------|----------|------------------|
| `prometheus` | `localhost:9090` | `/metrics` | Self-monitoring |
| `jenkins` | `jenkins:8080` | `/prometheus/` | Builds, queues, JVM, jobs |
| `registry` | `local-registry:5000` | `/metrics` | Pulls, pushes, storage |
| `grafana` | `grafana:3000` | `/metrics` | Grafana health |

> **Note:** Targets use **container names** (e.g., `jenkins:8080`) because they are
> all on the same Docker network `ci-cd-network`. Docker's internal DNS resolves these names.

#### Storage settings

```yaml
command:
  - '--storage.tsdb.path=/prometheus'          # Where data is stored
  - '--storage.tsdb.retention.time=15d'        # Retains 15 days of metrics
  - '--web.enable-lifecycle'                    # Allows config reload via API
```

---

### 3.5 Grafana — Dashboards & Visualization

**File:** `compose.yaml` (service `grafana`) + `grafana/provisioning/`

Grafana is the visualization layer. It reads data from Prometheus and displays it in
interactive dashboards.

#### Auto-provisioning

```yaml
volumes:
  - ./grafana/provisioning:/etc/grafana/provisioning:ro
```

The `datasource.yml` file is automatically read by Grafana at startup, configuring
Prometheus as the data source **without manual intervention**.

#### Startup order

```yaml
depends_on:
  prometheus:
    condition: service_healthy
```

Grafana **only starts** after Prometheus is healthy (responding to healthchecks).
This ensures the datasource is available when Grafana comes up.

---

### 3.6 Network — ci-cd-network

```yaml
networks:
  ci-cd-network:
    driver: bridge
```

All services share a Bridge network called `ci-cd-network`. This enables:

- **DNS resolution by name:** `jenkins`, `prometheus`, `grafana`, `local-registry`
- **Isolation:** Services outside this network cannot access the containers
- **Direct communication:** No need to expose ports between services

```
jenkins ◀──────▶ local-registry     (image push)
jenkins ◀──────▶ prometheus          (metrics via /prometheus/)
prometheus ◀───▶ grafana             (metrics queries)
prometheus ◀───▶ local-registry      (registry metrics)
```

---

### 3.7 Volumes — Data Persistence

```yaml
volumes:
  jenkins_home:       # Configs, jobs, credentials, build history
  registry_data:      # Stored Docker images
  prometheus_data:    # Time series (last 15 days of metrics)
  grafana_data:       # Dashboards, users, custom alerts
```

Without these volumes, **all data would be lost** when restarting the containers.
Docker Named Volumes persist on the host at `/var/lib/docker/volumes/`.

---

## 4. Full Pipeline Flow

When you `git push` to GitHub, the pipeline follows this flow:

```
 DEVELOPER                GITHUB                  JENKINS                    DOCKER                  REGISTRY
    │                       │                        │                         │                       │
    │── git push ──────────▶│                        │                         │                       │
    │                       │── webhook/polling ────▶│                         │                       │
    │                       │                        │                         │                       │
    │                       │                   ┌────┴────┐                    │                       │
    │                       │                   │STAGE 1  │                    │                       │
    │                       │                   │Checkout │                    │                       │
    │                       │                   │git clone│                    │                       │
    │                       │                   └────┬────┘                    │                       │
    │                       │                        │                         │                       │
    │                       │                   ┌────┴─────────┐               │                       │
    │                       │                   │STAGE 2       │               │                       │
    │                       │                   │Build & Test  │               │                       │
    │                       │                   │(parallel per │               │                       │
    │                       │                   │ service)     │──docker run──▶│                       │
    │                       │                   │              │  maven:3.9    │                       │
    │                       │                   │ mvn clean    │◀──results────│                       │
    │                       │                   │ mvn test     │               │                       │
    │                       │                   └────┬─────────┘               │                       │
    │                       │                        │                         │                       │
    │                       │                   ┌────┴──────────┐              │                       │
    │                       │                   │STAGE 3        │              │                       │
    │                       │                   │Build Docker   │──docker ────▶│                       │
    │                       │                   │Images         │  build       │                       │
    │                       │                   └────┬──────────┘              │                       │
    │                       │                        │                         │                       │
    │                       │                   ┌────┴──────────┐              │    ┌──────────────┐   │
    │                       │                   │STAGE 4        │──docker ────▶│───▶│              │   │
    │                       │                   │Push Images    │  push        │    │ localhost:   │   │
    │                       │                   │               │              │    │ 5001         │   │
    │                       │                   └────┬──────────┘              │    └──────────────┘   │
    │                       │                        │                         │                       │
    │                       │                   ┌────┴────┐                    │                       │
    │                       │                   │ POST    │                    │                       │
    │                       │                   │ Actions │                    │                       │
    │                       │                   │ cleanup │                    │                       │
    │                       │                   └─────────┘                    │                       │
```

### Stage Breakdown

| # | Stage | What it does | Tool |
|---|-------|--------------|------|
| 1 | **Checkout** | Clones the GitHub repository into the Jenkins workspace | `git` |
| 2 | **Build & Test** | Runs `mvn clean package` and `mvn test` inside a Maven container. Executes in parallel (one per microservice) | `docker` + `maven:3.9.7` |
| 3 | **Build Docker Images** | Creates Docker images with the compiled JARs, using the project's Dockerfile | `docker build` |
| 4 | **Push Images** | Pushes images to the local registry (`localhost:5001`) with version tags | `docker push` |
| 5 | **Post Actions** | Cleans workspace, notifies success/failure | `cleanWs()` |

---

## 5. Getting Started

```bash
# 1. Start all services (first time — with build)
docker compose up -d --build

# 2. Check if all services are healthy
docker compose ps

# 3. Follow logs in real time
docker compose logs -f

# 4. Stop the environment
docker compose down

# 5. Stop and DELETE all data (volumes)
docker compose down -v
```

---

## 6. Access & Ports

| Service | URL | Credentials |
|---------|-----|-------------|
| **Jenkins** | [http://localhost:8080](http://localhost:8080) | Set up on first access |
| **Registry** | [http://localhost:5001/v2/_catalog](http://localhost:5001/v2/_catalog) | No authentication |
| **Prometheus** | [http://localhost:9090](http://localhost:9090) | No authentication |
| **Grafana** | [http://localhost:3000](http://localhost:3000) | `admin` / `admin` |
