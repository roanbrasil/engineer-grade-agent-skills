---
name: local-dev-environment
description: Set up, configure, or troubleshoot local development environments using Docker Compose, Dev Containers, Tilt, Skaffold, task runners, and environment variable management.
---

# Local Development Environment

A consistent, reproducible local dev environment eliminates "works on my machine" problems, reduces onboarding time, and mirrors production closely enough to catch integration issues early.

```
+----------------------------------------------------------+
|                   Developer Machine                      |
|                                                          |
|  +----------------+    +---------------------------+    |
|  | Task Runner    |    | Dev Container / Codespace |    |
|  | make dev       |--->| postCreateCommand         |    |
|  | task migrate   |    | VS Code Extensions        |    |
|  +----------------+    +---------------------------+    |
|          |                          |                    |
|          v                          v                    |
|  +-------------------------------------------+          |
|  |          Docker Compose                   |          |
|  |  +----------+  +-------+  +-----------+  |          |
|  |  |PostgreSQL|  | Redis |  |   Kafka   |  |          |
|  |  | :5432    |  | :6379 |  |   :9092   |  |          |
|  |  | (volume) |  |(vol.) |  | (vol.)    |  |          |
|  |  +----------+  +-------+  +-----------+  |          |
|  |           App :8080                       |          |
|  +-------------------------------------------+          |
|          |                                               |
|  direnv: auto-load .envrc on cd                        |
|  .env:   local overrides (never committed)              |
+----------------------------------------------------------+
```

---

## Docker Compose for Local Dev

Docker Compose is the standard way to model local infrastructure dependencies. The goal is one command (`docker compose up`) to start everything the app needs.

### Full `docker-compose.yml` Example

PostgreSQL + Redis + Kafka + Zookeeper + app with health checks, volumes, and profiles.

```yaml
# docker-compose.yml
version: "3.9"

services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${DB_USER:-appuser}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-secret}
      POSTGRES_DB: ${DB_NAME:-appdb}
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/init:/docker-entrypoint-initdb.d   # seed scripts run once
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-appuser} -d ${DB_NAME:-appdb}"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s
    profiles: ["infra", "all"]

  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    profiles: ["infra", "all"]

  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"
    volumes:
      - zk_data:/var/lib/zookeeper/data
      - zk_logs:/var/lib/zookeeper/log
    healthcheck:
      test: ["CMD", "bash", "-c", "echo ruok | nc localhost 2181 | grep imok"]
      interval: 10s
      timeout: 5s
      retries: 5
    profiles: ["infra", "all"]

  kafka:
    image: confluentinc/cp-kafka:7.6.0
    depends_on:
      zookeeper:
        condition: service_healthy
    ports:
      - "9092:9092"
      - "29092:29092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    volumes:
      - kafka_data:/var/lib/kafka/data
    healthcheck:
      test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:29092"]
      interval: 10s
      timeout: 10s
      retries: 10
      start_period: 30s
    profiles: ["infra", "all"]

  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
      target: dev
    ports:
      - "8080:8080"
      - "5005:5005"   # JVM remote debug
    volumes:
      - .:/workspace:cached        # source mount for hot reload
      - app_gradle:/root/.gradle   # cache build artifacts
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/${DB_NAME:-appdb}
      SPRING_DATASOURCE_USERNAME: ${DB_USER:-appuser}
      SPRING_DATASOURCE_PASSWORD: ${DB_PASSWORD:-secret}
      SPRING_REDIS_HOST: redis
      SPRING_KAFKA_BOOTSTRAP_SERVERS: kafka:29092
      JAVA_TOOL_OPTIONS: "-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005"
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
      kafka:
        condition: service_healthy
    profiles: ["all"]

volumes:
  postgres_data:
  redis_data:
  zk_data:
  zk_logs:
  kafka_data:
  app_gradle:
```

### Key Patterns

**Health checks before app starts**
```yaml
depends_on:
  postgres:
    condition: service_healthy   # waits for healthcheck to pass, not just container start
```

**Persist data between restarts**
```yaml
volumes:
  - postgres_data:/var/lib/postgresql/data   # named volume; survives compose down
  - .:/workspace:cached                       # bind mount with perf hint on macOS
```

**Profiles — start only infra**
```bash
# Start only infrastructure (no app)
docker compose --profile infra up -d

# Start everything
docker compose --profile all up

# Tear down but keep volumes
docker compose --profile all down

# Tear down and delete all data
docker compose --profile all down -v
```

**`.env` file for local overrides**
```bash
# .env (never commit — add to .gitignore)
DB_USER=myuser
DB_PASSWORD=localpassword123
DB_NAME=myapp_dev

# .env.example (commit this as a template)
DB_USER=appuser
DB_PASSWORD=changeme
DB_NAME=appdb
```

**.gitignore additions**
```
.env
.env.local
*.env
!.env.example
```

**`docker compose watch` (Compose 2.22+)**

Sync file changes into container without rebuild — sub-second feedback for interpreted languages.

```yaml
# In the service definition
develop:
  watch:
    - action: sync
      path: ./src
      target: /workspace/src
      ignore:
        - node_modules/
    - action: rebuild
      path: package.json   # rebuild when dependencies change
```

```bash
docker compose watch   # watches and syncs; Ctrl-C to stop
```

### Anti-Patterns

- **No health checks** — app starts before DB is ready; random connection errors on first boot
- **Root user in postgres** — use a dedicated app user; matches production
- **No volume for DB data** — data lost on every `compose down`; migrations re-run constantly
- **Hard-coded credentials in compose file** — use environment variable substitution with `.env`
- **`latest` image tag** — breaks reproducibility; pin to a specific version like `postgres:16-alpine`

---

## Dev Containers

A Dev Container runs the entire development environment inside a Docker container. Every team member gets an identical environment; no "install this first" setup docs needed.

```
+---------------------------------------------+
| Host Machine                                |
|  +---------------------------------------+  |
|  | VS Code / JetBrains / GitHub Codespace|  |
|  |  +----------------------------------+ |  |
|  |  | Dev Container                    | |  |
|  |  |  Java 21, Maven, Node 20         | |  |
|  |  |  VS Code Extensions installed    | |  |
|  |  |  postCreateCommand: mvn install  | |  |
|  |  |  /workspace -> source code       | |  |
|  |  +----------------------------------+ |  |
|  |         |                             |  |
|  |  Port Forwards: 8080, 5005, 9092     |  |
|  +---------------------------------------+  |
+---------------------------------------------+
```

### Full `devcontainer.json` for Spring Boot + Kafka

```json
// .devcontainer/devcontainer.json
{
  "name": "Spring Boot + Kafka Dev",
  "dockerComposeFile": ["../docker-compose.yml"],
  "service": "app",
  "workspaceFolder": "/workspace",

  "features": {
    "ghcr.io/devcontainers/features/java:1": {
      "version": "21",
      "jdkDistro": "ms"
    },
    "ghcr.io/devcontainers/features/node:1": {
      "version": "20"
    },
    "ghcr.io/devcontainers/features/docker-in-docker:2": {
      "version": "latest"
    },
    "ghcr.io/devcontainers/features/kubectl-helm-minikube:1": {
      "version": "latest",
      "helm": "latest",
      "minikube": "none"
    }
  },

  "customizations": {
    "vscode": {
      "extensions": [
        "vscjava.vscode-java-pack",
        "vmware.vscode-spring-boot",
        "redhat.java",
        "redhat.vscode-yaml",
        "ms-azuretools.vscode-docker",
        "eamodio.gitlens",
        "usernamehw.errorlens",
        "sonarsource.sonarlint-vscode"
      ],
      "settings": {
        "java.jdt.ls.java.home": "/usr/local/sdkman/candidates/java/current",
        "editor.formatOnSave": true,
        "editor.defaultFormatter": "redhat.java"
      }
    }
  },

  "forwardPorts": [8080, 5005, 9092, 5432, 6379],
  "portsAttributes": {
    "8080": { "label": "App", "onAutoForward": "notify" },
    "5005": { "label": "JVM Debug", "onAutoForward": "silent" },
    "9092": { "label": "Kafka", "onAutoForward": "silent" }
  },

  "postCreateCommand": "mvn dependency:go-offline -q && echo 'Dependencies cached'",
  "postStartCommand": "docker compose --profile infra up -d",

  "remoteUser": "vscode",
  "runServices": ["postgres", "redis", "zookeeper", "kafka"]
}
```

### Key Fields

| Field | Purpose |
|-------|---------|
| `features` | Add tools without custom Dockerfile; versioned and composable |
| `postCreateCommand` | Runs once after container created; install deps, compile |
| `postStartCommand` | Runs every time container starts; start infra services |
| `forwardPorts` | Ports available on host; click to open in browser |
| `runServices` | Which compose services start with the dev container |

### Using Features Without Custom Dockerfile

```json
"features": {
  "ghcr.io/devcontainers/features/python:1": { "version": "3.12" },
  "ghcr.io/devcontainers/features/terraform:1": { "version": "latest" },
  "ghcr.io/devcontainers/features/aws-cli:1": {}
}
```

### GitHub Codespaces

Same `devcontainer.json` works in Codespaces. Open any PR in a pre-configured environment:

```bash
# Open current branch in Codespace
gh codespace create --repo myorg/myapp --branch feature/my-feature

# List codespaces
gh codespace list

# SSH into codespace
gh codespace ssh
```

---

## Tilt for Kubernetes Local Dev

Tilt watches your source code and keeps your Kubernetes local cluster in sync automatically.

```
+--------------------------------------------------+
|  Tilt Dev Loop                                   |
|                                                  |
|  Source Change                                   |
|      |                                           |
|      v                                           |
|  docker_build() --> push to local registry       |
|      |                                           |
|      v                                           |
|  Live Update: copy files into pod (sub-second)  |
|   OR                                            |
|  k8s_yaml(): apply manifest changes             |
|      |                                           |
|      v                                           |
|  Tilt UI (localhost:10350)                       |
|  - Build status per service                      |
|  - Logs streamed in real time                    |
|  - Port forwards clickable                       |
+--------------------------------------------------+
```

### `Tiltfile`

```python
# Tiltfile

# Load extensions
load('ext://helm_resource', 'helm_resource', 'helm_repo')
load('ext://namespace', 'namespace_create')

# Create namespace if missing
namespace_create('dev')

# Build the app image
docker_build(
    'myorg/myapp',
    context='.',
    dockerfile='Dockerfile',
    only=['src/', 'pom.xml'],
    live_update=[
        # Sync compiled classes without rebuild
        sync('./target/classes', '/app/classes'),
        # Run command in pod after sync
        run('touch /app/restart.txt', trigger=['./target/classes']),
    ],
    ignore=['target/', '.git/', '**/*.md']
)

# Deploy Kubernetes manifests
k8s_yaml(kustomize('./k8s/overlays/local'))

# Configure resources
k8s_resource(
    'myapp',
    port_forwards=['8080:8080', '5005:5005'],
    resource_deps=['postgres', 'redis'],
    labels=['app']
)

# Infrastructure via Helm
helm_repo('bitnami', 'https://charts.bitnami.com/bitnami')

helm_resource(
    'postgres',
    'bitnami/postgresql',
    namespace='dev',
    flags=['--set', 'auth.password=secret', '--set', 'auth.database=appdb'],
    port_forwards=['5432:5432'],
    labels=['infra']
)

helm_resource(
    'redis',
    'bitnami/redis',
    namespace='dev',
    flags=['--set', 'auth.enabled=false'],
    port_forwards=['6379:6379'],
    labels=['infra']
)
```

### Key Commands

```bash
tilt up           # start Tilt; opens UI at localhost:10350
tilt down         # stop all resources
tilt ci           # run in CI mode; exit with code on failure
tilt args -- --env dev   # pass args to Tiltfile
```

---

## Skaffold

Skaffold is Google's alternative to Tilt; integrates tightly with Helm and Kustomize.

```yaml
# skaffold.yaml
apiVersion: skaffold/v4beta6
kind: Config
metadata:
  name: myapp

build:
  artifacts:
    - image: myorg/myapp
      jib:
        project: com.myorg:myapp
        args:
          - -Plocal
  local:
    push: false
    useBuildkit: true

deploy:
  helm:
    releases:
      - name: myapp
        chartPath: ./helm/myapp
        valuesFiles:
          - ./helm/myapp/values-local.yaml
        setValues:
          image.repository: myorg/myapp
          image.tag: ""  # skaffold injects this

portForward:
  - resourceType: service
    resourceName: myapp
    namespace: dev
    port: 8080
    localPort: 8080
  - resourceType: service
    resourceName: myapp
    namespace: dev
    port: 5005
    localPort: 5005

profiles:
  - name: local
    activation:
      - kubeContext: docker-desktop
    patches:
      - op: replace
        path: /build/local/push
        value: false
  - name: ci
    patches:
      - op: replace
        path: /build/local/push
        value: true
```

```bash
skaffold dev                  # watch loop; rebuild + redeploy on change
skaffold dev --profile=local  # use local profile
skaffold run                  # one-shot deploy (no watch)
skaffold delete               # remove deployed resources
```

---

## Task Runners

Task runners codify common development commands. `make` is universally available; `Task` (taskfile.dev) is more ergonomic for complex workflows.

### Makefile

```makefile
# Makefile
.PHONY: dev test lint migrate seed clean build docker-build help

COMPOSE=docker compose --profile all
INFRA=docker compose --profile infra

help:  ## Show this help
	@grep -E '^[a-zA-Z_-]+:.*?## .*$$' $(MAKEFILE_LIST) | awk 'BEGIN {FS = ":.*?## "}; {printf "\033[36m%-20s\033[0m %s\n", $$1, $$2}'

dev: ## Start full local environment (infra + app with hot reload)
	$(INFRA) up -d
	./gradlew bootRun --args='--spring.profiles.active=local'

infra: ## Start only infrastructure services
	$(INFRA) up -d

test: ## Run all tests
	./gradlew test

test-unit: ## Run unit tests only
	./gradlew test --tests '*Unit*'

test-integration: ## Run integration tests (requires infra running)
	$(INFRA) up -d
	./gradlew integrationTest

lint: ## Run linters and static analysis
	./gradlew spotlessCheck checkstyleMain

lint-fix: ## Auto-fix lint issues
	./gradlew spotlessApply

migrate: ## Run database migrations
	$(INFRA) up -d postgres
	./gradlew flywayMigrate

seed: ## Seed database with test data
	$(INFRA) up -d postgres
	./gradlew bootRun --args='--spring.profiles.active=seed' 2>&1 | grep -E 'Seeded|ERROR'

build: ## Build the application
	./gradlew build -x test

docker-build: build ## Build Docker image
	docker build -t myorg/myapp:local .

clean: ## Stop all services and remove containers
	$(COMPOSE) down -v
	./gradlew clean

logs: ## Tail logs from all compose services
	$(COMPOSE) logs -f

ps: ## Show status of compose services
	$(COMPOSE) ps
```

### Taskfile (YAML, cross-platform)

```yaml
# Taskfile.yml
version: '3'

vars:
  APP_NAME: myapp
  COMPOSE_CMD: docker compose --profile all
  INFRA_CMD: docker compose --profile infra

env:
  GRADLE_OPTS: "-Dorg.gradle.daemon=false"

tasks:
  default:
    desc: Show available tasks
    cmds:
      - task --list

  dev:
    desc: Start full local environment with hot reload
    cmds:
      - task: infra
      - ./gradlew bootRun --args='--spring.profiles.active=local'

  infra:
    desc: Start only infrastructure services (DB, cache, broker)
    cmds:
      - "{{.INFRA_CMD}} up -d"
    status:
      - docker compose ps postgres | grep healthy

  test:
    desc: Run all tests
    cmds:
      - ./gradlew test
    sources:
      - src/**/*.java
      - build.gradle*
    generates:
      - build/reports/tests/**

  test:unit:
    desc: Run unit tests only
    cmds:
      - ./gradlew test --tests '*Unit*'

  test:integration:
    desc: Run integration tests (starts infra automatically)
    deps: [infra]
    cmds:
      - ./gradlew integrationTest

  lint:
    desc: Run linters
    cmds:
      - ./gradlew spotlessCheck checkstyleMain

  lint:fix:
    desc: Auto-fix lint issues
    cmds:
      - ./gradlew spotlessApply

  migrate:
    desc: Run database migrations
    deps: [infra]
    cmds:
      - ./gradlew flywayMigrate

  seed:
    desc: Seed database with development data
    deps: [migrate]
    cmds:
      - ./gradlew bootRun --args='--spring.profiles.active=seed'

  build:
    desc: Build application artifact
    cmds:
      - ./gradlew build -x test

  docker:build:
    desc: Build Docker image
    deps: [build]
    cmds:
      - docker build -t myorg/{{.APP_NAME}}:local .

  clean:
    desc: Stop all services and clean build artifacts
    cmds:
      - "{{.COMPOSE_CMD}} down -v"
      - ./gradlew clean

  logs:
    desc: Tail logs from all services
    cmds:
      - "{{.COMPOSE_CMD}} logs -f"
```

```bash
# Install Task
brew install go-task   # macOS
# or
sh -c "$(curl --location https://taskfile.dev/install.sh)" -- -d

task dev         # start dev environment
task test        # run tests
task lint:fix    # fix lint issues
task --list      # show all tasks
```

---

## Environment Variable Management

### direnv — Auto-Load on `cd`

`direnv` hooks into your shell and loads `.envrc` automatically when you enter a directory.

```bash
# Install
brew install direnv
# Add to ~/.zshrc or ~/.bashrc
eval "$(direnv hook zsh)"

# .envrc in project root
export DB_URL="postgresql://appuser:secret@localhost:5432/appdb"
export REDIS_URL="redis://localhost:6379"
export KAFKA_BROKERS="localhost:9092"
export LOG_LEVEL="DEBUG"
export AWS_PROFILE="myapp-dev"

# Python virtualenv layout
layout python3

# Allow the .envrc (first time, or after changes)
direnv allow .
```

### dotenv Libraries

```bash
# Python
pip install python-dotenv

# .env file
DB_URL=postgresql://appuser:secret@localhost:5432/appdb
DEBUG=true

# Python code
from dotenv import load_dotenv
import os
load_dotenv()  # loads .env from cwd
db_url = os.getenv("DB_URL")
```

```go
// Go — godotenv
import "github.com/joho/godotenv"
func init() {
    _ = godotenv.Load()  // loads .env; ignores if missing in production
}
```

```javascript
// Node.js
require('dotenv').config()  // or import 'dotenv/config'
const dbUrl = process.env.DB_URL
```

### Secret Management

```bash
# 1Password CLI — inject secrets at runtime
eval $(op signin)
export DB_PASSWORD=$(op read "op://dev-vault/postgres/password")

# AWS Secrets Manager
export DB_PASSWORD=$(aws secretsmanager get-secret-value \
  --secret-id myapp/dev/db-password \
  --query SecretString --output text)

# HashiCorp Vault
export DB_PASSWORD=$(vault kv get -field=password secret/myapp/dev/postgres)
```

### `.env.example` Template

```bash
# .env.example — commit this; explains every variable
# Copy to .env and fill in real values

# Database
DB_HOST=localhost
DB_PORT=5432
DB_NAME=appdb
DB_USER=appuser
DB_PASSWORD=changeme         # get from 1Password: dev-vault/postgres/password

# Cache
REDIS_URL=redis://localhost:6379

# Kafka
KAFKA_BROKERS=localhost:9092

# App
LOG_LEVEL=DEBUG
APP_PORT=8080

# AWS (use AWS SSO in dev)
AWS_PROFILE=myapp-dev
AWS_REGION=us-east-1
```

---

## Checklist

### New Project Setup
- [ ] `docker-compose.yml` with health checks for all dependencies
- [ ] Named volumes for all stateful services
- [ ] Compose profiles: `infra` for infrastructure only, `all` for everything
- [ ] `.env.example` committed; `.env` in `.gitignore`
- [ ] `Makefile` or `Taskfile.yml` with `dev`, `test`, `lint`, `migrate`, `seed`, `clean`
- [ ] `README.md` quick start: clone → `cp .env.example .env` → `make dev`

### Dev Container (Team Uniformity)
- [ ] `.devcontainer/devcontainer.json` in repo
- [ ] `features` for runtime (Java/Node/Python) instead of custom Dockerfile
- [ ] `postCreateCommand` caches dependencies
- [ ] All required VS Code extensions listed
- [ ] Port forwards for app + debug port

### Security
- [ ] No real credentials in any committed file
- [ ] `.env` and `*.env` in `.gitignore`
- [ ] Secrets retrieved from vault/1Password in `postCreateCommand`
- [ ] Dev DB uses non-root Postgres user
