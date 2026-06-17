---
name: helm-charts
description: Design, write, test, or review Helm charts for Kubernetes — including values design, template best practices, subcharts, OCI registry, and CI testing.
---

# Helm Chart Design

Helm is the Kubernetes package manager. A chart is a reusable, parameterized set of Kubernetes manifests. Good chart design makes the application easy to deploy across environments without YAML copy-paste.

```
+--------------------------------------------------------------+
|  Helm Architecture                                          |
|                                                              |
|  Developer                                                  |
|    |                                                          |
|    |  helm install my-app ./chart -f prod-values.yaml       |
|    v                                                          |
|  +------------------+                                        |
|  | Helm Client      |                                        |
|  | - Render templates with values                           |
|  | - Validate rendered YAML                                  |
|  | - Send to Kubernetes API                                  |
|  +------------------+                                        |
|         |                                                    |
|         v                                                    |
|  +------------------+     +-----------------------------+   |
|  | Kubernetes API   |     | Helm Release Secret         |   |
|  | (applies YAML)   |     | stores rendered manifest    |   |
|  +------------------+     | enables rollback            |   |
|                            +-----------------------------+   |
|                                                              |
|  OCI Registry (ghcr.io)                                    |
|    helm push mychart-1.0.0.tgz oci://ghcr.io/org/charts   |
|    helm install my-app oci://ghcr.io/org/charts/mychart    |
+--------------------------------------------------------------+
```

---

## Helm Fundamentals

| Concept | Description |
|---------|-------------|
| Chart | Package of Kubernetes manifests with Go templating |
| Release | Deployed instance of a chart; `helm install <release-name> <chart>` |
| Values | Configuration inputs; `values.yaml` = defaults; override with `-f` or `--set` |
| Repository | Collection of charts; indexed by `index.yaml` |
| Subchart | Dependency chart bundled in `charts/` directory |
| Named template | Reusable snippet defined in `_helpers.tpl`; called with `include` |

```bash
# Core commands
helm install my-app ./chart                          # deploy
helm install my-app ./chart -f prod.yaml             # deploy with overrides
helm install my-app ./chart --set image.tag=v1.2.3  # deploy with single override
helm upgrade my-app ./chart -f prod.yaml             # upgrade existing release
helm upgrade --install my-app ./chart                # install or upgrade (idempotent)
helm uninstall my-app                                # delete release
helm rollback my-app 2                               # roll back to revision 2
helm history my-app                                  # list revisions
helm status my-app                                   # show release info
helm get values my-app                               # show values applied to release
helm get manifest my-app                             # show rendered manifests
```

---

## Chart Structure

```
my-chart/
├── Chart.yaml              # metadata: name, version, appVersion, dependencies
├── values.yaml             # default configuration values
├── templates/
│   ├── _helpers.tpl        # named templates (partials); not rendered directly
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   ├── configmap.yaml
│   ├── secret.yaml
│   ├── hpa.yaml
│   ├── serviceaccount.yaml
│   ├── pdb.yaml            # PodDisruptionBudget
│   └── NOTES.txt           # post-install instructions printed to user
└── charts/                 # downloaded subcharts (via helm dependency update)
```

### `Chart.yaml`

```yaml
apiVersion: v2
name: my-app
description: My Spring Boot microservice
type: application
version: 0.5.2          # chart version; bump when chart changes (semver)
appVersion: "2.1.0"     # application version; informational; used in labels

keywords:
  - java
  - spring-boot
  - microservice

maintainers:
  - name: Platform Team
    email: platform@myorg.com

dependencies:
  - name: postgresql
    version: "13.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled       # only include if values.postgresql.enabled=true
    alias: postgresql

  - name: redis
    version: "18.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

```bash
helm dependency update ./my-chart    # download subcharts to charts/
helm dependency list ./my-chart      # show dependency status
```

---

## `values.yaml` Design

Organize values into logical groups. Document every non-obvious field.

```yaml
# values.yaml

# ---------------------------------------------------------------------------
# Image
# ---------------------------------------------------------------------------
image:
  repository: myorg/my-app
  tag: ""           # overridden by CI; defaults to appVersion if empty
  pullPolicy: IfNotPresent
  pullSecrets: []   # list of image pull secret names

# ---------------------------------------------------------------------------
# Deployment
# ---------------------------------------------------------------------------
replicaCount: 2

strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 0
    maxSurge: 1

# ---------------------------------------------------------------------------
# Container
# ---------------------------------------------------------------------------
containerPort: 8080
managementPort: 8081   # Spring Boot actuator

env: {}                # extra env vars: {MY_VAR: "value"}
envFrom: []            # extra envFrom sources (ConfigMaps, Secrets)

# ---------------------------------------------------------------------------
# Resources
# ---------------------------------------------------------------------------
resources:
  requests:
    cpu: 250m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi

# ---------------------------------------------------------------------------
# Probes
# ---------------------------------------------------------------------------
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: management
  initialDelaySeconds: 60
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: management
  initialDelaySeconds: 20
  periodSeconds: 5
  failureThreshold: 3

startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: management
  failureThreshold: 30
  periodSeconds: 10

# ---------------------------------------------------------------------------
# Service
# ---------------------------------------------------------------------------
service:
  type: ClusterIP
  port: 80
  targetPort: http   # named port from containerPort

# ---------------------------------------------------------------------------
# Ingress
# ---------------------------------------------------------------------------
ingress:
  enabled: false
  className: nginx
  annotations: {}
  hosts:
    - host: myapp.example.com
      paths:
        - path: /
          pathType: Prefix
  tls: []

# ---------------------------------------------------------------------------
# Autoscaling
# ---------------------------------------------------------------------------
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

# ---------------------------------------------------------------------------
# Pod Disruption Budget
# ---------------------------------------------------------------------------
podDisruptionBudget:
  enabled: true
  minAvailable: 1   # or maxUnavailable

# ---------------------------------------------------------------------------
# Service Account
# ---------------------------------------------------------------------------
serviceAccount:
  create: true
  name: ""          # auto-generated if empty
  annotations: {}   # for IRSA: eks.amazonaws.com/role-arn: arn:aws:iam::...

# ---------------------------------------------------------------------------
# Pod Settings
# ---------------------------------------------------------------------------
podAnnotations: {}
podLabels: {}
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000

containerSecurityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: [ALL]

# ---------------------------------------------------------------------------
# Scheduling
# ---------------------------------------------------------------------------
nodeSelector: {}
tolerations: []
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app.kubernetes.io/name: my-app
          topologyKey: kubernetes.io/hostname

# ---------------------------------------------------------------------------
# Persistence
# ---------------------------------------------------------------------------
persistence:
  enabled: false
  storageClass: ""
  size: 1Gi
  accessMode: ReadWriteOnce

# ---------------------------------------------------------------------------
# ConfigMap / Secrets
# ---------------------------------------------------------------------------
config:
  APPLICATION_PROFILE: production
  LOG_LEVEL: INFO

existingSecret: ""   # use pre-created secret instead of creating one

# ---------------------------------------------------------------------------
# Subcharts
# ---------------------------------------------------------------------------
postgresql:
  enabled: false   # enable for local/test environments only

redis:
  enabled: false
```

---

## `_helpers.tpl` — Reusable Templates

Every chart should have a `_helpers.tpl` with standard helpers. This eliminates duplication across all template files.

```yaml
{{/*
# templates/_helpers.tpl
*/}}

{{/* Expand the name of the chart. */}}
{{- define "my-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
Truncated at 63 chars because some Kubernetes name fields are limited to 64 chars.
*/}}
{{- define "my-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/* Chart label: "my-app-0.5.2" */}}
{{- define "my-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Common labels (apply to all resources).
*/}}
{{- define "my-app.labels" -}}
helm.sh/chart: {{ include "my-app.chart" . }}
{{ include "my-app.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
Selector labels (used in spec.selector.matchLabels and pod template labels).
These must be immutable once set — do not add chart version here.
*/}}
{{- define "my-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "my-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
ServiceAccount name.
*/}}
{{- define "my-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "my-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
```

---

## Production-Grade Templates

### `templates/deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "my-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "my-app.selectorLabels" . | nindent 6 }}
  strategy:
    {{- toYaml .Values.strategy | nindent 4 }}
  template:
    metadata:
      annotations:
        # Force pod restart when configmap changes
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
        {{- with .Values.podAnnotations }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
      labels:
        {{- include "my-app.selectorLabels" . | nindent 8 }}
        {{- with .Values.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      {{- with .Values.image.pullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "my-app.serviceAccountName" . }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          securityContext:
            {{- toYaml .Values.containerSecurityContext | nindent 12 }}
          ports:
            - name: http
              containerPort: {{ .Values.containerPort }}
              protocol: TCP
            - name: management
              containerPort: {{ .Values.managementPort }}
              protocol: TCP
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
          envFrom:
            - configMapRef:
                name: {{ include "my-app.fullname" . }}
            - secretRef:
                {{- if .Values.existingSecret }}
                name: {{ .Values.existingSecret }}
                {{- else }}
                name: {{ include "my-app.fullname" . }}
                {{- end }}
            {{- with .Values.envFrom }}
            {{- toYaml . | nindent 12 }}
            {{- end }}
          livenessProbe:
            {{- toYaml .Values.livenessProbe | nindent 12 }}
          readinessProbe:
            {{- toYaml .Values.readinessProbe | nindent 12 }}
          startupProbe:
            {{- toYaml .Values.startupProbe | nindent 12 }}
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          volumeMounts:
            - name: tmp
              mountPath: /tmp     # needed for readOnlyRootFilesystem
      volumes:
        - name: tmp
          emptyDir: {}
      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      terminationGracePeriodSeconds: 60
```

### `templates/service.yaml`

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "my-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: {{ .Values.service.targetPort }}
      protocol: TCP
      name: http
  selector:
    {{- include "my-app.selectorLabels" . | nindent 4 }}
```

### `templates/ingress.yaml`

```yaml
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "my-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
  {{- with .Values.ingress.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if .Values.ingress.className }}
  ingressClassName: {{ .Values.ingress.className }}
  {{- end }}
  {{- if .Values.ingress.tls }}
  tls:
    {{- toYaml .Values.ingress.tls | nindent 4 }}
  {{- end }}
  rules:
    {{- range .Values.ingress.hosts }}
    - host: {{ .host | quote }}
      http:
        paths:
          {{- range .paths }}
          - path: {{ .path }}
            pathType: {{ .pathType }}
            backend:
              service:
                name: {{ include "my-app.fullname" $ }}
                port:
                  name: http
          {{- end }}
    {{- end }}
{{- end }}
```

### `templates/hpa.yaml`

```yaml
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "my-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "my-app.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    {{- if .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    {{- end }}
    {{- if .Values.autoscaling.targetMemoryUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
    {{- end }}
{{- end }}
```

### `templates/serviceaccount.yaml`

```yaml
{{- if .Values.serviceAccount.create -}}
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ include "my-app.serviceAccountName" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "my-app.labels" . | nindent 4 }}
  {{- with .Values.serviceAccount.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
automountServiceAccountToken: false
{{- end }}
```

### `templates/NOTES.txt`

```
Application {{ include "my-app.fullname" . }} has been deployed.

Version:  {{ .Chart.AppVersion }}
Revision: {{ .Release.Revision }}

{{- if .Values.ingress.enabled }}
Access the application at:
{{- range .Values.ingress.hosts }}
  http{{ if $.Values.ingress.tls }}s{{ end }}://{{ .host }}
{{- end }}
{{- else }}
Port-forward to access the application:
  kubectl port-forward svc/{{ include "my-app.fullname" . }} 8080:{{ .Values.service.port }} -n {{ .Release.Namespace }}
  Then open: http://localhost:8080
{{- end }}

Check status:
  kubectl get pods -n {{ .Release.Namespace }} -l "{{ include "my-app.selectorLabels" . | replace "\n" "," }}"
```

---

## Helm OCI Registry

```bash
# Package the chart
helm package ./my-chart   # produces my-chart-0.5.2.tgz

# Authenticate to GHCR
echo $GITHUB_TOKEN | helm registry login ghcr.io -u $GITHUB_USER --password-stdin

# Push to OCI registry
helm push my-chart-0.5.2.tgz oci://ghcr.io/myorg/charts

# Pull and install from OCI registry
helm install my-app oci://ghcr.io/myorg/charts/my-chart --version 0.5.2

# Show chart info from registry
helm show chart oci://ghcr.io/myorg/charts/my-chart --version 0.5.2
```

### GitHub Actions: Publish Chart on Tag

```yaml
# .github/workflows/helm-publish.yaml
name: Publish Helm Chart
on:
  push:
    tags: ['chart/v*']

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      packages: write
    steps:
      - uses: actions/checkout@v4
      - name: Install Helm
        uses: azure/setup-helm@v4
      - name: Login to GHCR
        run: |
          echo "${{ secrets.GITHUB_TOKEN }}" | \
          helm registry login ghcr.io -u ${{ github.actor }} --password-stdin
      - name: Package and push
        run: |
          helm package ./charts/my-chart
          helm push my-chart-*.tgz oci://ghcr.io/${{ github.repository_owner }}/charts
```

---

## Testing Helm Charts

### `helm template` and `helm lint`

```bash
# Render templates locally; catch syntax errors
helm template my-app ./my-chart -f values-test.yaml

# Render and validate against live cluster (dry run)
helm template my-app ./my-chart | kubectl apply --dry-run=server -f -

# Lint the chart
helm lint ./my-chart
helm lint ./my-chart -f values-production.yaml   # lint with production values

# Test with required values set
helm lint ./my-chart --set image.tag=v1.0.0
```

### `helm-unittest`

```bash
# Install plugin
helm plugin install https://github.com/helm-unittest/helm-unittest.git

# Test file: tests/deployment_test.yaml
suite: Deployment Tests
templates:
  - templates/deployment.yaml

tests:
  - it: should render with correct image
    set:
      image.repository: myorg/myapp
      image.tag: v1.2.3
    asserts:
      - equal:
          path: spec.template.spec.containers[0].image
          value: myorg/myapp:v1.2.3

  - it: should not render replicas when HPA enabled
    set:
      autoscaling.enabled: true
    asserts:
      - notExists:
          path: spec.replicas

  - it: should have required labels
    asserts:
      - isNotNull:
          path: metadata.labels["app.kubernetes.io/name"]
      - isNotNull:
          path: metadata.labels["app.kubernetes.io/instance"]

  - it: should render ingress when enabled
    set:
      ingress.enabled: true
      ingress.hosts[0].host: myapp.example.com
    template: templates/ingress.yaml
    asserts:
      - hasDocuments:
          count: 1
      - equal:
          path: spec.rules[0].host
          value: myapp.example.com
```

```bash
helm unittest ./my-chart          # run all tests
helm unittest ./my-chart -v       # verbose output
```

### `chart-testing` (ct) in CI

```yaml
# .github/workflows/helm-test.yaml
name: Helm Chart Tests
on: [pull_request]

jobs:
  lint-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0   # ct needs full history

      - uses: azure/setup-helm@v4

      - uses: helm/chart-testing-action@v2.6.1

      - name: Run chart-testing (lint)
        run: ct lint --target-branch ${{ github.event.repository.default_branch }}

      - name: Create kind cluster
        uses: helm/kind-action@v1.9.0
        if: steps.list-changed.outputs.changed == 'true'

      - name: Run chart-testing (install)
        run: ct install --target-branch ${{ github.event.repository.default_branch }}
```

---

## Versioning

```bash
# Bump chart version when chart templates change
# Bump appVersion when application changes

# Chart version is semantic: MAJOR.MINOR.PATCH
# - PATCH: fix a template bug
# - MINOR: add a new optional feature (new values key with default)
# - MAJOR: breaking change (rename a required value, restructure values)

# Chart and appVersion are independent
# Example: chart 1.3.0 ships appVersion "2.1.0"
# Example: chart 1.3.1 fixes a template bug; still ships appVersion "2.1.0"
```

---

## Anti-Patterns

- **Hard-coding namespace** — use `{{ .Release.Namespace }}`; never hard-code `default`
- **`image.tag: latest`** — breaks rollback; always use a specific tag in production
- **No `checksum/config` annotation** — pods don't restart when ConfigMap changes; add `checksum/config` annotation
- **No resource requests/limits** — scheduler can't make good decisions; always set both
- **Mixing selector labels and display labels** — `spec.selector.matchLabels` must be immutable; use `selectorLabels` helper (no `chart` version) and add chart version only in `metadata.labels`
- **`required` on optional values** — use `default` for optional; use `required` only for values that have no safe default
- **Subchart values flat in root** — prefix subchart values with the subchart alias to avoid collision: `postgresql.auth.password`

## Checklist

- [ ] `_helpers.tpl` has `fullname`, `labels`, `selectorLabels`, `serviceAccountName`
- [ ] All resources use `include "chart.labels"` and `include "chart.selectorLabels"`
- [ ] `image.tag` defaults to `""` (uses `appVersion` at deploy time)
- [ ] Resources requests and limits set in `values.yaml`
- [ ] Liveness, readiness, and startup probes configured
- [ ] `ingress.enabled: false` default; enabled per environment
- [ ] `autoscaling.enabled: false` default; HPA template guards with `{{- if .Values.autoscaling.enabled }}`
- [ ] `podSecurityContext` and `containerSecurityContext` set (non-root, readOnly root FS)
- [ ] `checksum/config` annotation on Deployment to trigger rolling restart on ConfigMap change
- [ ] `helm lint` passes with no warnings
- [ ] `helm unittest` tests cover at least: correct image, HPA toggle, ingress toggle, required labels
- [ ] Chart version bumped on every PR that changes templates or values schema
