---
name: platform-engineering
description: Platform engineering and Internal Developer Platform (IDP) design — invoke when designing developer platforms, golden paths, self-service infrastructure, Backstage catalogs, paved roads, policy as code, or measuring developer experience.
---

# Platform Engineering

## What Platform Engineering Is

Platform engineering is the discipline of building internal products that enable application teams to deliver software faster and more safely without needing to understand underlying infrastructure details.

```
WITHOUT PLATFORM ENGINEERING           WITH PLATFORM ENGINEERING
------------------------------         ---------------------------------
Developer needs Kubernetes             Developer runs: platform new-service
→ Opens Jira ticket                    → Answers 3 prompts (name, team, lang)
→ Waits 2 days for infra team          → Gets: repo, CI/CD, observability,
→ Gets partial setup, misses             K8s manifests, secret management
  observability and secrets            → Coding in < 5 minutes
→ Asks again, waits again
→ 2 weeks to first deploy
```

### Platform Team Mission

- Build the platform as an internal product, not a ticket-processing service
- Reduce developer cognitive load: developers think about their service, not the platform
- Standardize without mandating: golden paths reduce friction; deviation is allowed with justification
- Enable self-service: developers provision what they need without filing tickets

### Platform as a Product

```
Traditional IT ops model:
  Developer → ticket → ops team → (wait) → resource created

Platform product model:
  Developer → self-service portal / CLI → resource created immediately
  Platform team → roadmap, backlog, releases, NPS, user research
```

The platform team has internal customers (developers). Treat them as such:
- Interview developers regularly about pain points
- Measure internal NPS (Net Promoter Score)
- Maintain a public roadmap visible to all engineering teams
- Write release notes for platform changes
- Define and publish an internal SLA (uptime, support response time)

---

## Golden Paths

A golden path is an opinionated, well-maintained way to do the most common things. It is not mandatory — teams can deviate — but deviating means leaving the happy path and losing free support.

```
GOLDEN PATH                    OFF-PATH
---------------------------    ---------------------------------
Supported by platform team     Team owns it entirely
Documentation maintained       Team writes their own docs
Updated when deps change       Team tracks updates manually
Observability pre-wired        Team adds observability manually
Security baseline included     Team responsible for security
CI/CD ready out of the box     Team configures CI/CD from scratch
```

### What Makes a Good Golden Path

1. Solves the 80% use case — do not try to handle every edge case
2. Opinionated — fewer choices reduce cognitive load
3. Maintained — kept up-to-date; deprecation path for old versions
4. Self-service — no ticket required to start using it
5. Documented — clear docs on what is included and how to customize
6. Observable — metrics, logs, traces wired up by default

### Template Technologies

| Tool         | Type               | Best For                              |
|--------------|--------------------|---------------------------------------|
| cookiecutter | Python CLI         | Simple file/directory templates       |
| copier       | Python CLI         | Templates with updates/migrations     |
| Backstage    | Web platform       | Full IDP with catalog + scaffolder    |
| Helm charts  | Kubernetes         | K8s deployment templates              |
| Terraform modules | IaC          | Reusable infra patterns               |

### Example Golden Path: New Microservice

```bash
$ platform new-service
? Service name: order-processor
? Team: payments-team
? Language: Kotlin
? Framework: Spring Boot

Creating service...
[✓] GitHub repository created: github.com/acme/order-processor
[✓] Branch protection rules configured
[✓] CI/CD pipeline (GitHub Actions) configured
[✓] Service registered in Backstage catalog
[✓] Kubernetes namespace created: order-processor-prod
[✓] Default resource limits applied (CPU: 500m/1000m, Mem: 512Mi/1Gi)
[✓] Prometheus metrics endpoint: /actuator/prometheus
[✓] Grafana dashboard created
[✓] Alert rules configured (error rate, latency p99)
[✓] Vault secret path provisioned: secret/order-processor
[✓] PagerDuty service created and linked
[✓] SonarQube project configured
[✓] README generated with team conventions

Done! Developer can start coding in order-processor/src/main/kotlin/
```

---

## Backstage

Backstage is an open-source Internal Developer Platform framework built by Spotify. It provides a unified frontend for all platform capabilities.

### Core Components

```
Backstage
├── Software Catalog        — registry of all services, libraries, APIs, teams
├── Software Templates      — self-service project creation (Scaffolder)
├── TechDocs                — docs-as-code rendered from Markdown
└── Plugins                 — extensible: CI, K8s, cost, security, etc.
```

### catalog-info.yaml

Every service registers itself in the catalog with this file at the repo root:

```yaml
apiVersion: backstage.io/v1alpha1
kind: Component
metadata:
  name: order-processor
  description: Processes customer orders and initiates fulfillment
  annotations:
    github.com/project-slug: acme/order-processor
    backstage.io/techdocs-ref: dir:.
    pagerduty.com/service-id: PXXXXXX
    sonarqube.org/project-key: order-processor
    prometheus.io/alert-manager-url: https://alertmanager.internal
  tags:
    - payments
    - kotlin
    - spring-boot
spec:
  type: service
  lifecycle: production
  owner: group:payments-team
  system: order-management
  dependsOn:
    - component:payment-gateway
    - component:inventory-service
    - resource:orders-database
  providesApis:
    - order-processor-api
```

### Software Templates (Scaffolder)

```yaml
# template.yaml
apiVersion: scaffolder.backstage.io/v1beta3
kind: Template
metadata:
  name: kotlin-microservice
  title: Kotlin Microservice
  description: Create a new Kotlin Spring Boot microservice
spec:
  owner: platform-team
  type: service
  parameters:
    - title: Service Details
      required: [name, team]
      properties:
        name:
          title: Service Name
          type: string
          pattern: '^[a-z][a-z0-9-]*$'
        team:
          title: Owning Team
          type: string
          ui:field: OwnerPicker
        description:
          title: Description
          type: string
  steps:
    - id: fetch-template
      name: Fetch Template
      action: fetch:template
      input:
        url: ./skeleton
        values:
          name: ${{ parameters.name }}
          team: ${{ parameters.team }}
    - id: publish
      name: Create GitHub Repo
      action: publish:github
      input:
        repoUrl: github.com?owner=acme&repo=${{ parameters.name }}
        defaultBranch: main
    - id: register
      name: Register in Catalog
      action: catalog:register
      input:
        repoContentsUrl: ${{ steps.publish.output.repoContentsUrl }}
        catalogInfoPath: /catalog-info.yaml
```

### Useful Plugins

| Plugin          | Provides                                              |
|-----------------|-------------------------------------------------------|
| GitHub Actions  | CI/CD run status in catalog entity page               |
| Kubernetes      | Pod/deployment status for the service                 |
| PagerDuty       | On-call schedule and active incidents                 |
| SonarQube       | Code quality metrics and vulnerability count          |
| Cost Insights   | AWS/GCP/Azure spend per team/service                  |
| Vault           | Secret management integration                         |
| ArgoCD          | GitOps deployment status                              |
| Grafana         | Embedded dashboards in the catalog entity page        |

---

## Self-Service Infrastructure

### Crossplane (Kubernetes-Native IaC)

Crossplane lets developers provision cloud resources (databases, queues, caches) by creating Kubernetes custom resources — no Terraform, no tickets.

```yaml
# Developer creates this in their namespace
apiVersion: database.acme.io/v1alpha1
kind: PostgreSQLInstance
metadata:
  name: orders-db
  namespace: order-processor
spec:
  parameters:
    storageGB: 20
    version: "15"
    tier: standard
  writeConnectionSecretToRef:
    name: orders-db-credentials  # auto-populated secret
```

Platform team defines the Composite Resource Definition (XRD) once; developers self-serve from that point.

```
Developer creates PostgreSQLInstance CR
        |
        v
Crossplane Composite Resource controller
        |
        +-- Creates RDS instance (AWS) or Cloud SQL (GCP)
        +-- Creates security groups
        +-- Stores connection string in Kubernetes Secret
        |
        v
Developer reads Secret in their pod spec — no access to cloud console needed
```

### Terraform + Atlantis (PR-Based Self-Service)

```
Developer forks platform-infra repo
        |
        v
Makes changes to their service's infra module
        |
        v
Opens PR → Atlantis posts plan as PR comment
        |
Developer and team review the plan
        |
        v
Merge PR → Atlantis runs apply automatically
        |
        v
Infrastructure created; no ops team involvement
```

```hcl
# modules/microservice-infra/main.tf
# Platform team maintains this module
module "order-processor" {
  source  = "git::https://github.com/acme/platform-modules//microservice"

  service_name = "order-processor"
  team         = "payments-team"
  environment  = "production"

  database = {
    enabled     = true
    size_gb     = 20
    engine      = "postgres"
    version     = "15"
  }

  cache = {
    enabled     = true
    type        = "redis"
    size        = "cache.t3.micro"
  }
}
```

---

## Developer Experience Metrics

### DORA Metrics

The four key metrics from the DevOps Research and Assessment program. These measure software delivery performance.

```
METRIC                  ELITE       HIGH        MEDIUM      LOW
----------------------  ----------  ----------  ----------  ----------
Deployment Frequency    On-demand   Daily-week  Week-month  Monthly+
Lead Time for Changes   < 1 hour    Day-week    Week-month  Month-6mo
Change Failure Rate     0-5%        5-10%       10-15%      15-45%
MTTR (restore time)     < 1 hour    < 1 day     < 1 week    > 6 months
```

Measuring:
```python
# Deployment Frequency: count deploys per team per period
SELECT team, COUNT(*) as deploys, DATE_TRUNC('week', deployed_at) as week
FROM deployments
WHERE deployed_at > NOW() - INTERVAL '90 days'
GROUP BY team, week
ORDER BY week;

# Lead Time: first commit to production deploy
SELECT
  pr.team,
  AVG(EXTRACT(EPOCH FROM (d.deployed_at - c.committed_at))/3600) as avg_lead_time_hours
FROM deployments d
JOIN pull_requests pr ON d.pr_id = pr.id
JOIN commits c ON c.pr_id = pr.id AND c.is_first = true
WHERE d.deployed_at > NOW() - INTERVAL '30 days'
GROUP BY pr.team;
```

### SPACE Framework

Satisfaction and well-being, Performance, Activity, Communication and collaboration, Efficiency and flow.

| Dimension        | Example Metrics                                              |
|------------------|--------------------------------------------------------------|
| Satisfaction     | Developer NPS, retention, survey scores                      |
| Performance      | Reliability, code quality, fewer incidents                   |
| Activity         | PRs merged, deployments, code review throughput              |
| Communication    | PR review time, unblocked wait time                          |
| Efficiency       | Build time, deployment time, environment setup time          |

### Internal NPS Survey (Quarterly)

```
How likely are you to recommend the developer platform to a new colleague? (0-10)

What is the single most painful thing about your development workflow today?

In the last sprint, how much time did you spend on non-feature work (infra, config, debugging tooling)?
  □ < 1 hour   □ 1-4 hours   □ 4-8 hours   □ > 8 hours

What one thing would most improve your daily experience?
```

Track NPS trend, not just the number. A drop of 10+ points quarter-over-quarter is a signal to investigate immediately.

---

## Paved Roads and Guardrails

```
PAVED ROAD (golden path)           GUARDRAIL (enforced limit)
--------------------------------   --------------------------------
Easy to use, best practices        Cannot be bypassed
Maintained by platform team        Enforced automatically
Optional but highly recommended    Mandatory for compliance/security
Reduces friction for common case   Prevents dangerous configurations
Can be deviated from (with cost)   Deviation requires exception process
```

### Policy as Code

#### OPA Gatekeeper (Kubernetes)

```yaml
# ConstraintTemplate: define the policy schema
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: requireresourcelimits
spec:
  crd:
    spec:
      names:
        kind: RequireResourceLimits
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package requireresourcelimits
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.cpu
          msg := sprintf("Container %v must set CPU limits", [container.name])
        }
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.resources.limits.memory
          msg := sprintf("Container %v must set memory limits", [container.name])
        }

---
# Constraint: enforce the policy
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: RequireResourceLimits
metadata:
  name: must-have-resource-limits
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces: ["production", "staging"]
```

```yaml
# Approved image registry policy
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: AllowedImageRegistry
metadata:
  name: approved-registries-only
spec:
  parameters:
    allowedRegistries:
      - "docker.acme.internal"
      - "ghcr.io/acme"
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
```

#### conftest (Terraform / Kubernetes YAML Validation)

```rego
# policy/terraform/deny_public_s3.rego
package main

deny[msg] {
  resource := input.resource.aws_s3_bucket[name]
  resource.acl == "public-read"
  msg := sprintf("S3 bucket '%s' must not be public", [name])
}

deny[msg] {
  resource := input.resource.aws_security_group[name]
  ingress := resource.ingress[_]
  ingress.cidr_blocks[_] == "0.0.0.0/0"
  ingress.from_port == 22
  msg := sprintf("Security group '%s' exposes SSH to the internet", [name])
}
```

```bash
# Run in CI before terraform plan
conftest test --policy policy/terraform/ plan.json
```

### Common Guardrails

| Domain          | Guardrail                                          | Enforcement         |
|-----------------|----------------------------------------------------|---------------------|
| Kubernetes      | All containers must have resource limits           | OPA Gatekeeper      |
| Kubernetes      | Images only from approved registries               | OPA Gatekeeper      |
| Kubernetes      | No containers running as root                      | OPA Gatekeeper      |
| Terraform       | No public S3 buckets                               | conftest in CI      |
| Terraform       | No security groups with 0.0.0.0/0 on port 22/3389 | conftest in CI      |
| CI/CD           | All PRs must pass security scan (Trivy, Snyk)      | Required status check|
| Secrets         | No secrets in source code                          | git-secrets in hook |
| Dependencies    | No critical CVEs in direct dependencies            | Dependabot + block  |

---

## Platform Team Patterns

### Platform Roadmap Structure

```
NOW (this quarter)
  - Golden path v2: add observability pre-wired
  - Crossplane S3 composition
  - Backstage GitHub Actions plugin

NEXT (next quarter)
  - Self-service Redis provisioning
  - Cost insights dashboard by team
  - Developer portal mobile-responsive UI

LATER (backlog, prioritized)
  - Auto-scaling policy templates
  - Service mesh (Istio) golden path
  - Internal chaos engineering tooling
```

### Inner Source

Platform code lives in an internal repository. Application teams can:
- Read all platform code
- Open issues for bugs and feature requests
- Submit PRs (reviewed by platform team within 2 business days)
- Fork for experimental deviations (with notification to platform team)

Benefits:
- Platform gets contributions for edge cases it does not prioritize
- Teams understand the platform better by reading the code
- Trust increases when the platform is not a black box

### Toil Reduction Targets

Toil: repetitive, manual work that does not produce lasting value.

```
TOIL TO ELIMINATE                    AUTOMATION
-----------------------------------  -----------------------------------------
"Can you create a namespace?"        Crossplane / self-service portal
"Can you add me to the secrets?"     Vault + team-based auto-provisioning
"Why is my build failing?"           AI-powered build log analysis
"My staging env is broken"           Ephemeral environments per PR (auto-created)
"Can you rotate the credentials?"    Automated credential rotation via Vault leases
"I need a Grafana dashboard"         Auto-generated from service annotations
"Can you bump the image version?"    Renovate / Dependabot auto-PRs
```

### Platform SLA Example

```
Service Catalog (Backstage):        99.9% uptime; < 1h to resolve P1 incidents
CI/CD infrastructure:               99.5% uptime; < 30min to resolve P1
Secret management (Vault):          99.99% uptime; < 15min to restore on failure
Self-service CLI (platform tool):   Bug fixes within 2 business days
Template requests:                  New golden path template: 2 sprints
Support response:                   P1 < 1 hour; P2 < 4 hours; P3 < 2 business days
```

---

## Example: Complete New Service Golden Path Flow

```
+-------------------+
| Developer runs:   |
| platform new-svc  |
+--------+----------+
         |
         v
+--------+----------+
| CLI prompts:      |
| name, team, lang  |
+--------+----------+
         |
         v
+--------+------------------+
| GitHub repo created       |
| from template skeleton    |
| Branch protection: ON     |
| Required reviewers: 1     |
+--------+------------------+
         |
         v
+--------+------------------+
| CI/CD configured:         |
| - Build + test on PR      |
| - Security scan (Trivy)   |
| - conftest policy check   |
| - Auto-deploy to staging  |
| - Manual gate to prod     |
+--------+------------------+
         |
         v
+--------+------------------+
| Backstage catalog entry   |
| catalog-info.yaml created |
| TechDocs scaffold present |
+--------+------------------+
         |
         v
+--------+------------------+
| K8s resources:            |
| - Namespace created       |
| - ResourceQuota applied   |
| - NetworkPolicy: default  |
| - ServiceAccount created  |
| - HPA template included   |
+--------+------------------+
         |
         v
+--------+------------------+
| Observability:            |
| - Prometheus scrape config|
| - Grafana dashboard       |
| - Alert rules (error rate,|
|   latency p99, saturation)|
| - PagerDuty service linked|
+--------+------------------+
         |
         v
+--------+------------------+
| Secret management:        |
| - Vault path provisioned  |
| - External Secrets Operator|
|   configured for namespace|
+--------+------------------+
         |
         v
+--------+------------------+
| Developer starts coding   |
| All boilerplate done      |
| Time to first commit: 5m  |
+----------------------------+
```

---

## Anti-Patterns

| Anti-Pattern                                   | Problem                                  | Fix                                          |
|------------------------------------------------|------------------------------------------|----------------------------------------------|
| Building features no one asked for             | Wasted effort; poor adoption             | User research first; validate with 3+ teams  |
| Mandating the platform (no deviation allowed)  | Teams work around it; toxic relationship | Paved road model; deviation allowed with cost|
| No SLA for platform                            | Developers cannot rely on it             | Define and publish internal SLA              |
| Ticket-driven provisioning                     | Bottleneck; cognitive load on developers | Self-service for everything in the 80% case  |
| Platform team = police                         | Adversarial relationship                 | Platform team = product team for developers  |
| One-size-fits-all golden path                  | Doesn't fit edge cases; abandoned        | Opinionated for 80%; escape hatch documented |
| No feedback loop from developers               | Building in the dark                     | Quarterly NPS + weekly user interviews       |
| Guardrails enforced too late (in prod)         | Costly failures; frustrated developers   | Shift left: enforce in CI, not runtime       |
| Platform undocumented                          | Developers don't adopt it                | Docs-as-code in Backstage TechDocs           |
| DORA metrics used to blame teams               | Gaming the metric; trust destroyed       | Use metrics for system improvement, not perf |

## Platform Engineering Checklist

**Foundation**
- [ ] Platform team has a published product roadmap visible to all developers
- [ ] Internal SLA defined and published for each platform service
- [ ] Quarterly developer NPS survey in place; trend being tracked
- [ ] Platform team holds regular user interviews with application teams
- [ ] Inner source model: platform code readable and PR-able by all engineers

**Golden Paths**
- [ ] At least one golden path for "start a new service" in each primary language/framework
- [ ] Golden path includes: repo, CI/CD, K8s manifests, observability, secrets, catalog registration
- [ ] Templates use a versioned template tool (copier/Backstage scaffolder)
- [ ] Deviation from golden path is documented (what you give up, how to get support)
- [ ] Golden paths have automated tests (scaffold a service, verify it builds and deploys)

**Self-Service**
- [ ] Developers can provision databases without filing a ticket
- [ ] Developers can create namespaces or environments without filing a ticket
- [ ] Secrets self-service: developers can create/rotate secrets for their service
- [ ] All self-service actions are logged and auditable

**Policy and Security**
- [ ] OPA Gatekeeper policies enforce resource limits on all production pods
- [ ] Image registry policy: only approved registries allowed in production
- [ ] No-root policy enforced in admission control
- [ ] Terraform policies in CI: no public S3, no open SSH ingress
- [ ] Secret scanning in pre-commit hooks and CI

**Observability**
- [ ] DORA metrics dashboard visible to all teams and leadership
- [ ] Build time and deploy time tracked as platform performance KPIs
- [ ] Golden path services get Prometheus + Grafana + alerting out of the box
- [ ] Platform team runs regular toil-reduction sprints

**Backstage**
- [ ] All services have catalog-info.yaml registered in Backstage
- [ ] Service ownership defined for every catalog entry
- [ ] TechDocs present for all platform-maintained tools
- [ ] Backstage plugins installed for: CI status, K8s status, cost insights
