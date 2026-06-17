---
name: ci-cd-pipelines
description: Design and implement CI/CD pipelines with GitHub Actions or GitLab CI, including caching, secrets, deployment strategies, and rollback patterns
---

# CI/CD Pipeline Design & Implementation

## Core Principles

### Continuous Integration (CI)

```
RULE 1: Every commit to any branch triggers the pipeline.
RULE 2: Pipeline must provide feedback in < 10 minutes.
RULE 3: Fail fast — cheapest checks run first.
RULE 4: If the pipeline is red, fixing it is the team's top priority.
RULE 5: Never merge a failing pipeline (no "merge and see" exceptions).
```

### Continuous Delivery vs Continuous Deployment

```
Continuous Delivery:    Every passing build IS a candidate for deployment.
                        Deployment is a business decision, not a technical one.
                        Manual gate before production is acceptable.

Continuous Deployment:  Every passing build IS deployed to production automatically.
                        Requires extremely high test confidence and feature flags.

KEY INSIGHT: Separate DEPLOY (putting code in production) from RELEASE
             (making the feature visible to users). Feature flags enable this.
```

---

## Pipeline Stages and Ordering

```
COMMIT                              PASSING BUILD = DEPLOYABLE CANDIDATE
  |
  v
+-------------------+   FAST (~2 min)
|  1. Static        |   Lint, format check, type check
|     Analysis      |   Fail here before wasting time on slower stages
+-------------------+
  |
  v
+-------------------+   FAST (~3 min)
|  2. Unit Tests    |   Pure logic, no I/O, no external dependencies
|                   |   Should cover critical paths and edge cases
+-------------------+
  |
  v
+-------------------+   MEDIUM (~5 min)
|  3. Build /       |   Compile, generate artifacts
|     Compile       |   Fail here = dependency or compile error
+-------------------+
  |
  v
+-------------------+   MEDIUM (~5-10 min)
|  4. Integration   |   Real DB, real queues via TestContainers
|     Tests         |   Test actual service boundaries
+-------------------+
  |
  v
+-------------------+   MEDIUM (~5 min, parallelizable)
|  5. Security      |   SAST (static analysis for vulnerabilities)
|     Scan          |   Dependency audit (CVE check)
+-------------------+
  |
  v
+-------------------+   MEDIUM (~5-10 min)
|  6. Docker Build  |   Build optimized image
|     + Push        |   Push to registry with git SHA tag
+-------------------+
  |
  v
+-------------------+   SLOW (~10-15 min)
|  7. Deploy to     |   Automated deployment to staging environment
|     Staging       |
+-------------------+
  |
  v
+-------------------+   (~5 min)
|  8. Smoke Tests   |   Basic "is the deployment alive?" checks
|  + Contract Tests |   Pact or OpenAPI contract verification
+-------------------+
  |
  v
+-------------------+   Gated (human approval) OR automatic
|  9. Deploy to     |   Blue-green, canary, or rolling strategy
|     Production    |
+-------------------+

Total target time: < 45 minutes for full pipeline
                   < 10 minutes to unit test feedback
```

---

## GitHub Actions

### Workflow Anatomy

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]
  workflow_dispatch:  # manual trigger

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true  # cancel redundant runs on new push

jobs:
  lint:
    name: Static Analysis
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - name: Cache pip
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}
          restore-keys: |
            ${{ runner.os }}-pip-
      - run: pip install ruff mypy
      - run: ruff check .
      - run: mypy src/

  test:
    name: Unit Tests
    needs: lint          # only runs if lint passes
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: '3.12'
      - uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}
      - run: pip install -r requirements-dev.txt
      - run: pytest tests/unit/ --cov=src --cov-report=xml -q
      - uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage.xml
```

### Caching Strategies

```yaml
# Maven
- uses: actions/cache@v4
  with:
    path: ~/.m2/repository
    key: ${{ runner.os }}-maven-${{ hashFiles('**/pom.xml') }}
    restore-keys: ${{ runner.os }}-maven-

# Gradle
- uses: actions/cache@v4
  with:
    path: |
      ~/.gradle/caches
      ~/.gradle/wrapper
    key: ${{ runner.os }}-gradle-${{ hashFiles('**/*.gradle*', '**/gradle-wrapper.properties') }}

# pip
- uses: actions/cache@v4
  with:
    path: ~/.cache/pip
    key: ${{ runner.os }}-pip-${{ hashFiles('**/requirements*.txt') }}
    restore-keys: ${{ runner.os }}-pip-

# cargo (Rust)
- uses: actions/cache@v4
  with:
    path: |
      ~/.cargo/bin/
      ~/.cargo/registry/index/
      ~/.cargo/registry/cache/
      ~/.cargo/git/db/
      target/
    key: ${{ runner.os }}-cargo-${{ hashFiles('**/Cargo.lock') }}

# Node / npm
- uses: actions/cache@v4
  with:
    path: ~/.npm
    key: ${{ runner.os }}-node-${{ hashFiles('**/package-lock.json') }}
```

**Cache key strategy:**
```
Good key:   ${{ runner.os }}-<tool>-${{ hashFiles('**/lockfile') }}
Why:        OS changes invalidate; lockfile changes invalidate (correct behavior)

Restore keys (fallback):
  ${{ runner.os }}-<tool>-    ← Partial restore if exact match fails
```

### Matrix Builds

```yaml
jobs:
  test:
    strategy:
      fail-fast: false   # don't cancel siblings on one failure
      matrix:
        python-version: ['3.10', '3.11', '3.12']
        os: [ubuntu-latest, windows-latest]
        exclude:
          - os: windows-latest
            python-version: '3.10'  # skip unsupported combo
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
```

### Reusable Workflows (DRY Pipelines)

```yaml
# .github/workflows/reusable-test.yml — called by other repos
name: Reusable Test Workflow
on:
  workflow_call:
    inputs:
      python-version:
        required: false
        type: string
        default: '3.12'
    secrets:
      CODECOV_TOKEN:
        required: false

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version }}
      - run: pip install -r requirements-dev.txt && pytest

---
# Caller in another repo:
jobs:
  test:
    uses: myorg/shared-workflows/.github/workflows/reusable-test.yml@main
    with:
      python-version: '3.11'
    secrets: inherit
```

### OIDC for Cloud Authentication (No Long-Lived Secrets)

```yaml
# Instead of storing AWS_ACCESS_KEY_ID in secrets:
jobs:
  deploy:
    permissions:
      id-token: write   # required for OIDC
      contents: read
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789:role/github-actions-deploy
          aws-region: us-east-1
          # GitHub gets a short-lived token via OIDC — no stored credentials
```

### Environments and Protection Rules

```yaml
jobs:
  deploy-production:
    environment:
      name: production
      url: https://app.example.com
    # GitHub waits for required reviewers before running this job
    # Configured in: Settings → Environments → production → Protection rules
    steps:
      - run: ./deploy.sh production
```

---

## GitLab CI

### Pipeline Structure

```yaml
# .gitlab-ci.yml
stages:
  - lint
  - test
  - build
  - security
  - deploy-staging
  - smoke-test
  - deploy-production

variables:
  DOCKER_DRIVER: overlay2
  IMAGE_TAG: $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

# Reuse with extends
.base-python:
  image: python:3.12-slim
  before_script:
    - pip install -r requirements.txt

lint:
  extends: .base-python
  stage: lint
  script:
    - ruff check .
    - mypy src/

unit-tests:
  extends: .base-python
  stage: test
  script:
    - pytest tests/unit/ --cov=src --cov-report=term-missing
  coverage: '/TOTAL.+ ([0-9]{1,3}%)/'
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml

docker-build:
  stage: build
  image: docker:24
  services:
    - docker:24-dind
  script:
    - docker build -t $IMAGE_TAG .
    - docker push $IMAGE_TAG
  only:
    - main

deploy-staging:
  stage: deploy-staging
  environment:
    name: staging
    url: https://staging.example.com
  script:
    - ./scripts/deploy.sh staging $IMAGE_TAG
  only:
    - main

deploy-production:
  stage: deploy-production
  environment:
    name: production
    url: https://app.example.com
  script:
    - ./scripts/deploy.sh production $IMAGE_TAG
  when: manual          # requires human approval
  only:
    - main
```

### Includes for Reuse

```yaml
# .gitlab-ci.yml — compose from shared templates
include:
  - project: 'devops/ci-templates'
    ref: main
    file: '/templates/docker-build.yml'
  - local: '.gitlab/ci/test.yml'
  - template: 'Security/SAST.gitlab-ci.yml'

# Override a job from included template
docker-build:
  extends: .docker-build-template
  variables:
    DOCKERFILE: Dockerfile.production
```

### MR vs Branch Pipelines

```yaml
# Run expensive tests only on MR; quick tests on every push
unit-tests:
  script: pytest tests/unit/
  # runs on all pipelines

integration-tests:
  script: pytest tests/integration/
  rules:
    - if: $CI_PIPELINE_SOURCE == "merge_request_event"
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
  # only runs on MRs and main branch — saves CI minutes on feature branches
```

---

## Docker Image Optimization

### Multi-Stage Build

```dockerfile
# BAD — single stage, includes build tools in final image
FROM python:3.12
WORKDIR /app
COPY requirements*.txt ./
RUN pip install -r requirements-dev.txt  # includes test dependencies!
COPY . .
RUN python -m pytest                     # tests run at build time (wrong)
CMD ["python", "-m", "uvicorn", "main:app"]
# Result: ~1.2GB image with compilers, test libs, source code

# GOOD — multi-stage
FROM python:3.12-slim AS builder
WORKDIR /build
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

FROM python:3.12-slim AS production
WORKDIR /app
# Copy only the installed packages from builder
COPY --from=builder /root/.local /root/.local
COPY src/ ./src/
# Non-root user for security
RUN useradd -r -u 1001 appuser && chown -R appuser /app
USER appuser
CMD ["python", "-m", "uvicorn", "src.main:app", "--host", "0.0.0.0"]
# Result: ~180MB image, no build tools, no test deps
```

### .dockerignore

```
# .dockerignore — always include this
.git
.github
**/__pycache__
**/*.pyc
**/.pytest_cache
**/.mypy_cache
.env
.env.*
tests/
docs/
*.md
.dockerignore
Dockerfile*
```

### Layer Caching Strategy

```dockerfile
# Order layers from least-to-most-frequently-changed
FROM python:3.12-slim

# 1. System deps (rarely change) — cached longest
RUN apt-get update && apt-get install -y --no-install-recommends \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

# 2. Python deps (change when requirements.txt changes)
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# 3. Application code (changes most often) — last layer
COPY src/ ./src/
```

---

## Deployment Strategies

```
+-------------------+--------------------------------------------------+------------------+
| Strategy          | How it works                                     | Best for         |
+-------------------+--------------------------------------------------+------------------+
| Rolling Update    | Replace instances one-by-one; always some old,  | Most services;   |
|                   | some new running simultaneously                  | zero-downtime    |
+-------------------+--------------------------------------------------+------------------+
| Blue-Green        | Run full duplicate environment; switch traffic   | High-risk        |
|                   | instantly; old env stays for instant rollback    | deploys; needs   |
|                   |                                                  | 2x resources     |
+-------------------+--------------------------------------------------+------------------+
| Canary            | Route small % of traffic to new version;         | High-traffic     |
|                   | gradually increase; monitor; promote or rollback | services with    |
|                   |                                                  | measurable SLIs  |
+-------------------+--------------------------------------------------+------------------+
```

### Rolling Update (Kubernetes)

```yaml
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # allow 1 extra pod during update
      maxUnavailable: 0  # never reduce below desired count
  minReadySeconds: 10    # wait 10s before marking pod Ready
```

### Blue-Green with Feature Flags

```python
# Feature flag = decouple deploy from release
def get_checkout_flow(user_id: str) -> CheckoutFlow:
    if feature_flags.is_enabled("new-checkout-v2", user_id):
        return NewCheckoutFlow()   # new code deployed, only shown to % of users
    return LegacyCheckoutFlow()    # safe fallback always available
```

### Canary Analysis Gate

```yaml
# Argo Rollouts canary example
spec:
  strategy:
    canary:
      steps:
        - setWeight: 5       # 5% of traffic
        - pause: {duration: 10m}
        - analysis:
            templates:
              - templateName: error-rate-check
        - setWeight: 50
        - pause: {duration: 5m}
        - setWeight: 100
      analysis:
        templates:
          - templateName: error-rate-check
```

---

## Rollback Strategy

```
RULE: Every deployment MUST have an automated rollback trigger.
      If you can't roll back in < 5 minutes, your deployment strategy is wrong.

ROLLBACK TRIGGERS (automated):
  - Error rate > threshold for N minutes post-deploy
  - p99 latency > threshold post-deploy
  - Health check failures exceed threshold
  - Canary analysis fails
```

```bash
# Kubernetes — instant rollback
kubectl rollout undo deployment/api -n production

# Verify rollback is working
kubectl rollout status deployment/api -n production

# Rollback to specific revision
kubectl rollout undo deployment/api --to-revision=3 -n production
```

```yaml
# GitHub Actions — automated rollback job
deploy:
  runs-on: ubuntu-latest
  steps:
    - name: Deploy
      id: deploy
      run: ./deploy.sh $IMAGE_TAG

    - name: Wait and check error rate
      run: |
        sleep 120
        ERROR_RATE=$(./scripts/get-error-rate.sh)
        if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
          echo "Error rate $ERROR_RATE exceeds threshold — triggering rollback"
          exit 1
        fi

    - name: Rollback on failure
      if: failure() && steps.deploy.outcome == 'success'
      run: kubectl rollout undo deployment/api -n production
```

---

## Secret Management in CI

```
RULES:
  1. Never print secrets in logs (mask all secret variables)
  2. Never store secrets in the repository (not even encrypted, if avoidable)
  3. Use OIDC / workload identity instead of long-lived credentials
  4. Rotate secrets regularly (automate rotation)
  5. Use short-lived credentials wherever possible
  6. Audit secret access in CI logs
```

```yaml
# GitHub Actions — using secrets safely
- name: Deploy to AWS
  env:
    # Reference, never hardcode
    AWS_ACCOUNT_ID: ${{ secrets.AWS_ACCOUNT_ID }}
  run: |
    # WRONG: echo $AWS_ACCOUNT_ID  (prints to log)
    # RIGHT: use variable without echoing
    aws ecr get-login-password --region us-east-1 | \
      docker login --username AWS --password-stdin \
      ${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com
```

```yaml
# GitLab CI — masked variables
variables:
  DATABASE_PASSWORD:
    value: ""
    description: "Production DB password"
# Mark as Masked in GitLab UI: Settings → CI/CD → Variables → Masked
# Never appears in job logs even if accidentally echoed
```

---

## Pipeline Anti-Patterns

```
+--------------------------------------+--------------------------------------------------+
| Anti-Pattern                         | Correct Approach                                 |
+--------------------------------------+--------------------------------------------------+
| Manual gate before unit tests        | Automate everything before production gate       |
| Flaky tests in CI                    | Fix or quarantine; never "retry to see if green" |
| Slow feedback (> 20 min to PR)       | Parallelize jobs; cache aggressively             |
| Monolithic pipeline (can't run       | Make stages executable locally (Makefile/just)   |
| locally)                             |                                                  |
| Secrets in environment variables     | Use secrets manager (Vault, AWS Secrets Manager) |
| committed to repo                    | with CI integration                              |
| Same Docker image tag (latest)       | Tag with git SHA; immutable images               |
| No artifact versioning               | Every build produces versioned, traceable output |
| Testing in production only           | Staging environment mirrors production exactly   |
| No rollback plan                     | Every deploy script has a revert step            |
| Skipping security scan "to go fast" | Security scan is non-optional; parallelize it    |
+--------------------------------------+--------------------------------------------------+
```

---

## Complete GitHub Actions Example

```yaml
# .github/workflows/pipeline.yml
name: Full CI/CD Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  lint-and-type-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: pip-${{ hashFiles('requirements*.txt') }}
      - run: pip install ruff mypy && ruff check . && mypy src/

  unit-tests:
    needs: lint-and-type-check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: pip-${{ hashFiles('requirements*.txt') }}
      - run: pip install -r requirements-dev.txt
      - run: pytest tests/unit/ -q --tb=short

  integration-tests:
    needs: unit-tests
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: { python-version: '3.12' }
      - run: pip install -r requirements-dev.txt
      - run: pytest tests/integration/ -q
        env:
          DATABASE_URL: postgresql://postgres:test@localhost:5432/test

  security-scan:
    needs: unit-tests
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install pip-audit && pip-audit -r requirements.txt
      - uses: github/codeql-action/analyze@v3

  build-and-push:
    needs: [integration-tests, security-scan]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write
    outputs:
      image-digest: ${{ steps.push.outputs.digest }}
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        id: push
        with:
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-staging:
    needs: build-and-push
    environment: staging
    runs-on: ubuntu-latest
    steps:
      - run: |
          kubectl set image deployment/api \
            api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            -n staging

  deploy-production:
    needs: deploy-staging
    environment: production  # requires manual approval
    runs-on: ubuntu-latest
    steps:
      - run: |
          kubectl set image deployment/api \
            api=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }} \
            -n production
      - name: Verify deployment
        run: |
          kubectl rollout status deployment/api -n production --timeout=5m
```

---

## Quick Reference Checklist

### Pipeline Design

- [ ] Every commit triggers pipeline (no manual start)
- [ ] Lint and type check runs first (< 2 min)
- [ ] Unit tests run before build (fast feedback)
- [ ] Integration tests use real dependencies (TestContainers or CI services)
- [ ] Security scan runs in parallel with integration tests
- [ ] Docker image tagged with immutable git SHA (never `latest`)
- [ ] Staging deploy is fully automated; production gate is explicit
- [ ] Every pipeline has a rollback step

### Security

- [ ] No secrets hardcoded or committed
- [ ] OIDC used for cloud provider auth (no long-lived credentials)
- [ ] All secret variables masked in CI logs
- [ ] Dependency audit runs on every build
- [ ] SAST scan configured and breaking on HIGH findings

### Operations

- [ ] Pipeline runs in < 15 minutes (end-to-end including deploy)
- [ ] Failed pipeline notifications go to team channel
- [ ] Artifact versions are traceable back to git commit
- [ ] Flaky tests tracked and fixed within one sprint
- [ ] Pipeline steps can be run locally with equivalent commands
