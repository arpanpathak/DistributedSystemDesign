# Chapter 38: Helm, Kustomize & GitOps

## Introduction

In the previous chapters, we built observable, autoscaling applications running on
Kubernetes. But how do we manage the dozens of YAML manifests — Deployments,
Services, ConfigMaps, HPAs, ServiceMonitors, Ingresses — that define our
applications? How do we deploy the same application to development, staging, and
production with different configurations? How do we ensure that what's running in
the cluster matches what's in Git?

This chapter covers three complementary tools:

| Tool      | Purpose                                         |
|-----------|-------------------------------------------------|
| Helm      | Package management — template and distribute K8s manifests |
| Kustomize | Configuration management — patch and overlay manifests     |
| ArgoCD/Flux | Deployment management — sync Git state to cluster state  |

Together, they form the **GitOps** workflow: developers push changes to Git,
a GitOps controller detects the change, and the cluster converges to the desired
state automatically.

---

## 38.1 Helm

### 38.1.1 What Helm Does

Helm is the **package manager for Kubernetes**. Just as `apt` manages packages on
Debian or `brew` on macOS, Helm manages Kubernetes applications. A Helm **chart**
is a versioned, distributable bundle of Kubernetes manifests with templating support.

**Key concepts:**

| Concept    | Description                                              |
|------------|----------------------------------------------------------|
| Chart      | A package of templated Kubernetes manifests              |
| Release    | An installed instance of a chart on a cluster            |
| Repository | A collection of charts (like a package registry)         |
| Values     | Configuration parameters that customize a chart          |

**Why Helm?**
- **Templating:** One chart, many environments (dev/staging/prod)
- **Packaging:** Distribute complex applications as a single unit
- **Versioning:** Track changes to your application packaging
- **Dependencies:** Compose applications from reusable sub-charts
- **Lifecycle hooks:** Run setup/teardown tasks during install/upgrade
- **Rollback:** `helm rollback` undoes a bad deployment instantly

### 38.1.2 Chart Structure

```text
myapp/
├── Chart.yaml           # Chart metadata (name, version, description)
├── values.yaml          # Default configuration values
├── charts/              # Sub-chart dependencies
├── templates/           # Kubernetes manifest templates
│   ├── NOTES.txt        # Post-install usage instructions
│   ├── _helpers.tpl     # Template helper functions
│   ├── deployment.yaml  # Deployment template
│   ├── service.yaml     # Service template
│   ├── ingress.yaml     # Ingress template
│   ├── hpa.yaml         # HPA template
│   ├── serviceaccount.yaml
│   ├── configmap.yaml
│   └── tests/           # Helm test pods
│       └── test-connection.yaml
├── .helmignore          # Files to exclude from the chart package
└── README.md            # Chart documentation
```

### 38.1.3 Chart.yaml

```yaml
# Chart.yaml
# Metadata for the chart. This is the "package.json" of Helm.
apiVersion: v2              # v2 for Helm 3 (v1 for Helm 2, deprecated)
name: myapp                 # Chart name (lowercase, hyphenated)
description: A Go microservice for order processing
type: application           # "application" or "library"
version: 1.2.0              # Chart version (SemVer, incremented on chart changes)
appVersion: "v3.1.0"        # Version of the application being deployed
home: https://github.com/myorg/myapp
maintainers:
  - name: Platform Team
    email: platform@example.com
keywords:
  - go
  - microservice
  - orders
# Dependencies — sub-charts that this chart depends on.
# These are fetched with `helm dependency update`.
dependencies:
  - name: postgresql
    version: "12.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: postgresql.enabled    # Only include if postgresql.enabled=true
  - name: redis
    version: "17.x.x"
    repository: "https://charts.bitnami.com/bitnami"
    condition: redis.enabled
```

### 38.1.4 values.yaml

```yaml
# values.yaml
# Default configuration values for the myapp chart.
# Users can override any of these values with:
#   helm install myapp ./myapp -f custom-values.yaml
#   helm install myapp ./myapp --set replicaCount=5

# --- Application Configuration ---
replicaCount: 2

image:
  repository: myregistry/myapp
  tag: ""                    # Defaults to Chart.appVersion if empty
  pullPolicy: IfNotPresent

imagePullSecrets: []

# --- Service Configuration ---
service:
  type: ClusterIP
  port: 80
  targetPort: 8080
  annotations: {}

# --- Ingress Configuration ---
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

# --- Resource Configuration ---
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi

# --- Autoscaling ---
autoscaling:
  enabled: false
  minReplicas: 2
  maxReplicas: 20
  targetCPUUtilizationPercentage: 60
  targetMemoryUtilizationPercentage: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300

# --- Pod Configuration ---
podAnnotations: {}
podLabels: {}
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 1000
  fsGroup: 1000

securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL

# --- Health Checks ---
livenessProbe:
  httpGet:
    path: /healthz
    port: http
  initialDelaySeconds: 5
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /readyz
    port: http
  initialDelaySeconds: 5
  periodSeconds: 5
  failureThreshold: 3

# --- Monitoring ---
monitoring:
  enabled: true
  serviceMonitor:
    enabled: true
    interval: 15s
    path: /metrics
    labels:
      release: kube-prometheus-stack

# --- Application-Specific Configuration ---
config:
  logLevel: info
  logFormat: json
  databaseURL: ""            # Override in environment-specific values
  redisURL: ""
  otelCollectorEndpoint: "otel-collector:4317"
  featureFlags:
    enableNewCheckout: false
    enableBetaAPI: false

# --- Dependencies ---
postgresql:
  enabled: true
  auth:
    database: myapp
    username: myapp
    existingSecret: myapp-postgresql

redis:
  enabled: false

# --- Node Affinity and Tolerations ---
nodeSelector: {}
tolerations: []
affinity: {}

# --- Service Account ---
serviceAccount:
  create: true
  name: ""
  annotations: {}

# --- Pod Disruption Budget ---
podDisruptionBudget:
  enabled: true
  minAvailable: 1
```

### 38.1.5 Template Language

Helm uses Go's `text/template` package with Sprig library extensions.

#### _helpers.tpl — Reusable Template Functions

```yaml
{{/*
templates/_helpers.tpl

Reusable template definitions for the myapp chart.
These are called with the `include` function throughout other templates.
Naming convention: <chartname>.<helper-name>
*/}}

{{/*
myapp.name — Expand the name of the chart.
Truncated to 63 characters because Kubernetes names are limited
to 63 characters by the DNS naming spec (RFC 1123).
*/}}
{{- define "myapp.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
myapp.fullname — Create a fully qualified app name.
If a fullnameOverride is provided, use that. Otherwise, construct
the name from the release name and chart name. Truncated to 63 chars.
*/}}
{{- define "myapp.fullname" -}}
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

{{/*
myapp.chart — Chart name and version for the chart label.
*/}}
{{- define "myapp.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
myapp.labels — Common labels applied to all resources.
These follow the Kubernetes recommended label conventions.
*/}}
{{- define "myapp.labels" -}}
helm.sh/chart: {{ include "myapp.chart" . }}
{{ include "myapp.selectorLabels" . }}
{{- if .Chart.AppVersion }}
app.kubernetes.io/version: {{ .Chart.AppVersion | quote }}
{{- end }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
{{- end }}

{{/*
myapp.selectorLabels — Selector labels used by Deployments and Services.
These must not change between upgrades; otherwise, the Deployment
will create new ReplicaSets and orphan old pods.
*/}}
{{- define "myapp.selectorLabels" -}}
app.kubernetes.io/name: {{ include "myapp.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
myapp.serviceAccountName — Name of the ServiceAccount to use.
If serviceAccount.create is true, generates a name. Otherwise,
uses the default ServiceAccount.
*/}}
{{- define "myapp.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "myapp.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}

{{/*
myapp.image — Full container image reference including tag.
Uses Chart.appVersion as the default tag if image.tag is not set.
*/}}
{{- define "myapp.image" -}}
{{- $tag := default .Chart.AppVersion .Values.image.tag -}}
{{- printf "%s:%s" .Values.image.repository $tag -}}
{{- end }}
```

#### deployment.yaml — Full Template Example

```yaml
{{/*
templates/deployment.yaml

Deployment template for the myapp application.
This template demonstrates:
  - Conditional rendering (if/else)
  - Range loops (iterating over lists)
  - Include calls to helper templates
  - Sprig functions (toYaml, nindent, quote)
  - Values interpolation
*/}}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  {{- with .Values.deploymentAnnotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  {{/* Only set replicas if HPA is disabled — HPA manages replicas otherwise */}}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      {{- with .Values.podAnnotations }}
      annotations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      labels:
        {{- include "myapp.labels" . | nindent 8 }}
        {{- with .Values.podLabels }}
        {{- toYaml . | nindent 8 }}
        {{- end }}
    spec:
      {{- with .Values.imagePullSecrets }}
      imagePullSecrets:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      serviceAccountName: {{ include "myapp.serviceAccountName" . }}
      {{- with .Values.podSecurityContext }}
      securityContext:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: {{ .Chart.Name }}
          image: {{ include "myapp.image" . }}
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          {{- with .Values.securityContext }}
          securityContext:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          ports:
            - name: http
              containerPort: {{ .Values.service.targetPort }}
              protocol: TCP
          {{- with .Values.livenessProbe }}
          livenessProbe:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.readinessProbe }}
          readinessProbe:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          {{- with .Values.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          env:
            - name: LOG_LEVEL
              value: {{ .Values.config.logLevel | quote }}
            - name: LOG_FORMAT
              value: {{ .Values.config.logFormat | quote }}
            - name: OTEL_EXPORTER_OTLP_ENDPOINT
              value: {{ .Values.config.otelCollectorEndpoint | quote }}
            {{- if .Values.config.databaseURL }}
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "myapp.fullname" . }}-secrets
                  key: database-url
            {{- end }}
            {{- /* Render feature flags as environment variables */ -}}
            {{- range $key, $value := .Values.config.featureFlags }}
            - name: FEATURE_{{ $key | upper | replace "-" "_" }}
              value: {{ $value | quote }}
            {{- end }}
          volumeMounts:
            - name: tmp
              mountPath: /tmp
      volumes:
        {{/* /tmp volume for readOnlyRootFilesystem */}}
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
```

#### service.yaml

```yaml
{{/*
templates/service.yaml

Service template exposing the application within the cluster.
Annotations are conditionally rendered to support cloud load balancers,
service mesh integration, etc.
*/}}
apiVersion: v1
kind: Service
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  {{- with .Values.service.annotations }}
  annotations:
    {{- toYaml . | nindent 4 }}
  {{- end }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "myapp.selectorLabels" . | nindent 4 }}
```

#### ingress.yaml

```yaml
{{/*
templates/ingress.yaml

Ingress template with conditional TLS and multi-host support.
Only created if ingress.enabled is true.
*/}}
{{- if .Values.ingress.enabled -}}
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
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
    {{- range .Values.ingress.tls }}
    - hosts:
        {{- range .hosts }}
        - {{ . | quote }}
        {{- end }}
      secretName: {{ .secretName }}
    {{- end }}
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
                name: {{ include "myapp.fullname" $ }}
                port:
                  number: {{ $.Values.service.port }}
          {{- end }}
    {{- end }}
{{- end }}
```

#### hpa.yaml

```yaml
{{/*
templates/hpa.yaml

HorizontalPodAutoscaler template. Only created if autoscaling is enabled.
Includes configurable scaling behavior to prevent thrashing.
*/}}
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "myapp.fullname" . }}
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
  {{- with .Values.autoscaling.behavior }}
  behavior:
    {{- toYaml . | nindent 4 }}
  {{- end }}
{{- end }}
```

#### servicemonitor.yaml

```yaml
{{/*
templates/servicemonitor.yaml

ServiceMonitor for Prometheus Operator integration.
Only created if monitoring.serviceMonitor.enabled is true.
*/}}
{{- if and .Values.monitoring.enabled .Values.monitoring.serviceMonitor.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
    {{- with .Values.monitoring.serviceMonitor.labels }}
    {{- toYaml . | nindent 4 }}
    {{- end }}
spec:
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  endpoints:
    - port: http
      path: {{ .Values.monitoring.serviceMonitor.path }}
      interval: {{ .Values.monitoring.serviceMonitor.interval }}
{{- end }}
```

### 38.1.6 Helm Hooks

Hooks let you run operations at specific points in the release lifecycle:

```yaml
{{/*
templates/hooks/db-migrate.yaml

A pre-upgrade hook that runs database migrations before the new
version of the application is deployed. This ensures the database
schema is compatible with the new code.
*/}}
apiVersion: batch/v1
kind: Job
metadata:
  name: {{ include "myapp.fullname" . }}-db-migrate
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  annotations:
    # This annotation makes it a Helm hook.
    "helm.sh/hook": pre-upgrade,pre-install
    # Run before other pre-upgrade hooks (lower weight = earlier).
    "helm.sh/hook-weight": "-5"
    # Delete the Job after it succeeds (clean up).
    "helm.sh/hook-delete-policy": hook-succeeded
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: migrate
          image: {{ include "myapp.image" . }}
          command: ["./myapp", "migrate", "--up"]
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: {{ include "myapp.fullname" . }}-secrets
                  key: database-url
  backoffLimit: 3
```

**Hook types:**

| Hook            | When it runs                          |
|-----------------|---------------------------------------|
| pre-install     | Before any resources are created      |
| post-install    | After all resources are created       |
| pre-delete      | Before any resources are deleted      |
| post-delete     | After all resources are deleted       |
| pre-upgrade     | Before upgrade resources are created  |
| post-upgrade    | After upgrade resources are created   |
| pre-rollback    | Before rollback                       |
| post-rollback   | After rollback                        |
| test            | When `helm test` is run               |

### 38.1.7 Hands-On: Creating a Helm Chart from Scratch

```bash
# Step 1: Create the chart scaffold
helm create myapp

# This generates the full chart structure with sensible defaults.
# Modify the generated files to match your application.

# Step 2: Customize the chart (edit templates as shown above)

# Step 3: Lint the chart (catches common errors)
helm lint myapp/

# Step 4: Template the chart (render manifests without installing)
helm template myapp myapp/ -f values-dev.yaml

# Step 5: Dry-run install (validates against the cluster)
helm install myapp myapp/ --dry-run --debug

# Step 6: Install the chart
helm install myapp myapp/ \
  --namespace default \
  --set image.tag=v3.1.0 \
  -f values-production.yaml

# Step 7: Check the release
helm list
helm status myapp
helm get values myapp

# Step 8: Upgrade the release
helm upgrade myapp myapp/ \
  --set image.tag=v3.2.0 \
  -f values-production.yaml

# Step 9: Rollback if something goes wrong
helm rollback myapp 1    # Rollback to revision 1

# Step 10: Uninstall
helm uninstall myapp
```

### 38.1.8 Values Override Hierarchy

Values are merged in this order (later overrides earlier):

```
1. Parent chart's values.yaml (lowest priority)
2. Sub-chart's values.yaml
3. Current chart's values.yaml
4. -f / --values files (left to right)
5. --set flags
6. --set-string flags
7. --set-file flags (highest priority)
```

Example:

```bash
# values.yaml has replicaCount: 2
# values-prod.yaml has replicaCount: 5
# --set has replicaCount: 10

helm install myapp ./myapp \
  -f values-prod.yaml \
  --set replicaCount=10

# Result: replicaCount = 10 (--set wins)
```

### 38.1.9 Hands-On: Template Functions, Conditionals, Loops

```yaml
{{/*
templates/configmap.yaml

Demonstrates advanced Helm template patterns:
  - Conditionals (if/else)
  - Loops (range)
  - Sprig functions (upper, replace, toJson, b64enc)
  - Default values
  - Multi-line strings
  - Dictionary creation with dict
*/}}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "myapp.fullname" . }}-config
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
data:
  # Simple value interpolation with default
  log-level: {{ .Values.config.logLevel | default "info" | quote }}

  # Conditional value
  {{- if eq .Values.config.logFormat "json" }}
  log-format: "json"
  {{- else }}
  log-format: "text"
  {{- end }}

  # Feature flags as a JSON object
  feature-flags.json: |
    {{- $flags := dict }}
    {{- range $key, $value := .Values.config.featureFlags }}
    {{- $_ := set $flags $key $value }}
    {{- end }}
    {{ $flags | toJson }}

  # Loop over a list to generate configuration
  {{- if .Values.config.allowedOrigins }}
  allowed-origins: {{ .Values.config.allowedOrigins | join "," | quote }}
  {{- end }}

  # Application configuration file
  app.yaml: |
    server:
      port: {{ .Values.service.targetPort }}
      readTimeout: 10s
      writeTimeout: 30s
    database:
      maxOpenConns: {{ .Values.config.dbMaxConns | default 25 }}
      maxIdleConns: {{ .Values.config.dbMaxIdleConns | default 5 }}
    telemetry:
      otlpEndpoint: {{ .Values.config.otelCollectorEndpoint | quote }}
      serviceName: {{ include "myapp.fullname" . | quote }}
      serviceVersion: {{ .Chart.AppVersion | quote }}
```

### 38.1.10 Hands-On: Chart Testing with helm test

```yaml
{{/*
templates/tests/test-connection.yaml

Helm test that verifies the application is accessible after deployment.
Run with: helm test myapp
*/}}
apiVersion: v1
kind: Pod
metadata:
  name: "{{ include "myapp.fullname" . }}-test-connection"
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
  annotations:
    "helm.sh/hook": test
    "helm.sh/hook-delete-policy": before-hook-creation,hook-succeeded
spec:
  restartPolicy: Never
  containers:
    - name: test-health
      image: busybox:1.36
      command: ['sh', '-c']
      args:
        - |
          echo "Testing health endpoint..."
          wget -qO- --timeout=5 http://{{ include "myapp.fullname" . }}:{{ .Values.service.port }}/healthz
          echo ""
          echo "Testing metrics endpoint..."
          wget -qO- --timeout=5 http://{{ include "myapp.fullname" . }}:{{ .Values.service.port }}/metrics | head -20
          echo ""
          echo "All tests passed!"
```

```bash
# Run the tests
helm test myapp

# Expected output:
# NAME: myapp
# LAST DEPLOYED: ...
# STATUS: deployed
# TEST SUITE:     myapp-test-connection
# Last Started:   ...
# Last Completed: ...
# Phase:          Succeeded
```

### 38.1.11 Library Charts

Library charts contain reusable template definitions but no actual Kubernetes
resources. They are used as dependencies by application charts.

```yaml
# library-chart/Chart.yaml
apiVersion: v2
name: myorg-common
type: library    # This chart cannot be installed directly
version: 1.0.0
description: Common Helm template helpers for myorg charts
```

```yaml
# library-chart/templates/_labels.tpl
{{/*
Common label generator used by all myorg charts.
Usage:
  {{ include "myorg-common.labels" (dict "name" "myapp" "version" "1.0.0") }}
*/}}
{{- define "myorg-common.labels" -}}
app.kubernetes.io/name: {{ .name }}
app.kubernetes.io/version: {{ .version }}
app.kubernetes.io/managed-by: helm
company: myorg
{{- end }}
```

---

## 38.2 Kustomize

### 38.2.1 What Kustomize Does

Kustomize is a **template-free** configuration management tool built into kubectl.
Unlike Helm, which uses Go templates to generate YAML, Kustomize uses **overlays**
and **patches** to modify plain YAML files.

**Key differences from Helm:**

| Feature         | Helm                    | Kustomize                    |
|-----------------|-------------------------|------------------------------|
| Templating      | Go templates            | None (plain YAML)            |
| Configuration   | values.yaml + --set     | Patches and overlays         |
| Packaging       | Charts (tgz archives)   | Directories (no packaging)   |
| Distribution    | Chart repositories      | Git repositories             |
| Built into kubectl | No                  | Yes (`kubectl apply -k`)     |
| Complexity      | Higher (templates)      | Lower (just YAML)            |

**When to use which:**
- **Helm**: Distributing charts to external users, complex templating needs
- **Kustomize**: Internal deployments, environment-specific overlays, simpler setup

### 38.2.2 kustomization.yaml

The `kustomization.yaml` file is the entry point. It declares what resources to
manage and what transformations to apply.

```yaml
# base/kustomization.yaml
# Declares the base resources and common transformations.
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

# Metadata applied to all resources
namespace: default
namePrefix: myapp-
nameSuffix: ""

# Common labels added to all resources
commonLabels:
  app: myapp
  team: platform

# Common annotations added to all resources
commonAnnotations:
  managed-by: kustomize

# Resources to include (plain YAML files)
resources:
  - deployment.yaml
  - service.yaml
  - serviceaccount.yaml
  - hpa.yaml

# ConfigMap generator — creates ConfigMaps from files or literals
configMapGenerator:
  - name: myapp-config
    literals:
      - LOG_LEVEL=info
      - LOG_FORMAT=json
    files:
      - configs/app.yaml

# Secret generator — creates Secrets from files or literals
secretGenerator:
  - name: myapp-secrets
    literals:
      - DATABASE_PASSWORD=changeme
    type: Opaque

# Images — override image tags without patching
images:
  - name: myregistry/myapp
    newTag: v3.1.0
```

### 38.2.3 Base Resources

```yaml
# base/deployment.yaml
# Plain Kubernetes YAML — no templating, no special syntax.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: server    # Kustomize will prefix this with "myapp-"
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      serviceAccountName: server
      containers:
        - name: server
          image: myregistry/myapp    # Tag set by kustomization.yaml images field
          ports:
            - name: http
              containerPort: 8080
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          envFrom:
            - configMapRef:
                name: myapp-config
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
          readinessProbe:
            httpGet:
              path: /readyz
              port: http
---
# base/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: server
spec:
  selector:
    app: myapp
  ports:
    - name: http
      port: 80
      targetPort: http
---
# base/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: server
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: server
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
```

### 38.2.4 Overlays

Overlays modify the base for specific environments:

```
myapp/
├── base/
│   ├── kustomization.yaml
│   ├── deployment.yaml
│   ├── service.yaml
│   └── hpa.yaml
└── overlays/
    ├── development/
    │   ├── kustomization.yaml
    │   └── patches/
    │       └── deployment-patch.yaml
    ├── staging/
    │   ├── kustomization.yaml
    │   └── patches/
    │       ├── deployment-patch.yaml
    │       └── hpa-patch.yaml
    └── production/
        ├── kustomization.yaml
        ├── patches/
        │   ├── deployment-patch.yaml
        │   └── hpa-patch.yaml
        └── ingress.yaml
```

```yaml
# overlays/development/kustomization.yaml
# Development overlay — single replica, debug logging, no HPA.
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: development

resources:
  - ../../base

# Override the image tag for development
images:
  - name: myregistry/myapp
    newTag: dev-latest

# Override ConfigMap values
configMapGenerator:
  - name: myapp-config
    behavior: merge          # Merge with base ConfigMap
    literals:
      - LOG_LEVEL=debug
      - LOG_FORMAT=text      # Human-readable logs in dev

# Patches to modify base resources
patches:
  # Reduce to 1 replica in development
  - target:
      kind: Deployment
      name: server
    patch: |
      - op: replace
        path: /spec/replicas
        value: 1
  # Disable HPA in development
  - target:
      kind: HorizontalPodAutoscaler
      name: server
    patch: |
      $patch: delete
```

```yaml
# overlays/production/kustomization.yaml
# Production overlay — more replicas, stricter resources, ingress.
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: production

resources:
  - ../../base
  - ingress.yaml             # Additional resource only in production

images:
  - name: myregistry/myapp
    newTag: v3.1.0

configMapGenerator:
  - name: myapp-config
    behavior: merge
    literals:
      - LOG_LEVEL=info
      - LOG_FORMAT=json

# Strategic merge patches (YAML patch files)
patches:
  - path: patches/deployment-patch.yaml
  - path: patches/hpa-patch.yaml
```

```yaml
# overlays/production/patches/deployment-patch.yaml
# Strategic merge patch for the Deployment.
# Only the fields specified here are modified; all other fields
# remain unchanged from the base.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: server
spec:
  replicas: 5
  template:
    spec:
      containers:
        - name: server
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: "2"
              memory: 1Gi
          # Add production-specific environment variables
          env:
            - name: GOMAXPROCS
              value: "4"
      # Add anti-affinity in production to spread pods across nodes
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchLabels:
                    app: myapp
                topologyKey: kubernetes.io/hostname
```

```yaml
# overlays/production/patches/hpa-patch.yaml
# Increase HPA limits for production traffic.
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: server
spec:
  minReplicas: 5
  maxReplicas: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 2
          periodSeconds: 120
```

```yaml
# overlays/production/ingress.yaml
# Ingress resource only deployed in production.
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: server
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp-server
                port:
                  number: 80
```

### 38.2.5 Strategic Merge Patches vs JSON Patches

**Strategic merge patch:** Kubernetes-aware. Knows how to merge lists by key.

```yaml
# Strategic merge — adds a container, doesn't replace the list
apiVersion: apps/v1
kind: Deployment
metadata:
  name: server
spec:
  template:
    spec:
      containers:
        - name: sidecar
          image: envoyproxy/envoy:v1.28
          ports:
            - containerPort: 9901
```

**JSON patch (RFC 6902):** Explicit operations (add, remove, replace, move, copy).

```yaml
# JSON patch — precise operations
patches:
  - target:
      kind: Deployment
      name: server
    patch: |
      - op: replace
        path: /spec/replicas
        value: 10
      - op: add
        path: /spec/template/spec/containers/0/env/-
        value:
          name: NEW_VAR
          value: "hello"
      - op: remove
        path: /spec/template/spec/containers/0/resources/limits/cpu
```

### 38.2.6 Hands-On: Building and Applying

```bash
# Preview what Kustomize will generate (development overlay)
kubectl kustomize overlays/development/

# Apply development overlay
kubectl apply -k overlays/development/

# Preview production overlay
kubectl kustomize overlays/production/

# Apply production overlay
kubectl apply -k overlays/production/

# Diff against what's currently in the cluster
kubectl diff -k overlays/production/
```

---

## 38.3 ArgoCD

### 38.3.1 GitOps Principles

GitOps is an operational framework where:

1. **Git is the single source of truth** for the desired state of infrastructure
   and applications.
2. **Changes are made via Git** (pull requests, code review, merge).
3. **A controller** continuously compares Git state vs. cluster state and reconciles
   differences.
4. **Drift detection**: If someone makes a manual change to the cluster, the
   controller reverts it to match Git.

**Benefits:**
- Full audit trail (Git history)
- Rollback is `git revert`
- Pull request-based change management
- Self-healing infrastructure
- Environment consistency

### 38.3.2 ArgoCD Architecture

```
┌─────────────────────────────────────────────────────────┐
│                      ArgoCD                              │
│                                                          │
│  ┌────────────────┐  ┌──────────────┐  ┌─────────────┐ │
│  │  API Server    │  │  Repo Server  │  │ Application │ │
│  │                │  │              │  │ Controller   │ │
│  │ - REST/gRPC    │  │ - Clones Git │  │             │ │
│  │ - Web UI       │  │   repos      │  │ - Watches   │ │
│  │ - Auth (SSO)   │  │ - Renders    │  │   Application│ │
│  │ - RBAC         │  │   manifests  │  │   CRDs      │ │
│  │                │  │   (Helm,     │  │ - Compares   │ │
│  │                │  │    Kustomize,│  │   desired vs │ │
│  │                │  │    plain)    │  │   live state │ │
│  │                │  │              │  │ - Syncs      │ │
│  └────────────────┘  └──────────────┘  └─────────────┘ │
│                                                          │
│  ┌──────────────────────────────────────────────────┐   │
│  │              Redis (caching)                      │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Git Repo │ │ Git Repo │ │ Git Repo │
        │ (app1)   │ │ (app2)   │ │ (infra)  │
        └──────────┘ └──────────┘ └──────────┘
```

**API Server:** Exposes the ArgoCD API (REST + gRPC), serves the Web UI, handles
authentication (OIDC/SSO), and enforces RBAC policies.

**Repo Server:** Clones Git repositories and renders manifests. It understands
Helm charts, Kustomize overlays, and plain YAML directories.

**Application Controller:** The core reconciliation loop. It watches ArgoCD
Application CRDs, compares desired state (from Git) with live state (from the
cluster), and optionally syncs differences.

### 38.3.3 Installation

```bash
# Install ArgoCD in its own namespace
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for all pods to be ready
kubectl -n argocd wait --for=condition=ready pod --all --timeout=300s

# Get the initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Access the UI (port-forward)
kubectl -n argocd port-forward svc/argocd-server 8443:443 &

# Install the ArgoCD CLI
# (macOS)  brew install argocd
# (Linux)  curl -sSL -o argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64 && chmod +x argocd && sudo mv argocd /usr/local/bin/

# Login via CLI
argocd login localhost:8443 --insecure --username admin --password <password>

# Change the admin password
argocd account update-password
```

### 38.3.4 Application CRD

The `Application` CRD is ArgoCD's core resource. It defines the mapping from a
Git source to a Kubernetes destination.

```yaml
# application.yaml
# Deploys the myapp application from Git to the cluster.
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-production
  namespace: argocd            # Applications must be in the argocd namespace
  # Finalizer ensures ArgoCD cleans up cluster resources when the
  # Application is deleted.
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  # Project restricts what this application can do (RBAC).
  project: default

  # Source — where to get the manifests
  source:
    repoURL: https://github.com/myorg/myapp-deploy.git
    targetRevision: main       # Branch, tag, or commit SHA
    path: overlays/production  # Path within the repo

    # If using Kustomize, you can pass options:
    kustomize:
      images:
        - myregistry/myapp:v3.1.0

  # Destination — where to deploy
  destination:
    server: https://kubernetes.default.svc  # In-cluster
    namespace: production

  # Sync policy — how ArgoCD manages the application
  syncPolicy:
    # Automated sync — ArgoCD applies changes automatically when
    # it detects a difference between Git and the cluster.
    automated:
      # Prune: delete resources that exist in the cluster but not in Git.
      prune: true
      # Self-heal: revert manual changes made directly to the cluster.
      selfHeal: true
      # Allow empty: allow syncing when the app has no resources (useful
      # during initial setup).
      allowEmpty: false

    # Sync options
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true           # Prune after all other resources are synced

    # Retry policy for failed syncs
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m

  # Ignore differences in certain fields (e.g., fields mutated by controllers)
  ignoreDifferences:
    - group: apps
      kind: Deployment
      jsonPointers:
        - /spec/replicas          # Ignore because HPA manages replicas
    - group: autoscaling
      kind: HorizontalPodAutoscaler
      jqPathExpressions:
        - .status                 # Ignore status subresource
```

#### Application with Helm Source

```yaml
# application-helm.yaml
# Deploys an application using a Helm chart from a Git repo.
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp-helm
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp-chart.git
    targetRevision: main
    path: charts/myapp
    # Helm-specific configuration
    helm:
      # Override values.yaml
      valueFiles:
        - values.yaml
        - values-production.yaml
      # Additional --set overrides
      parameters:
        - name: image.tag
          value: v3.1.0
        - name: autoscaling.enabled
          value: "true"
      # Release name (defaults to application name)
      releaseName: myapp
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### 38.3.5 App of Apps Pattern

The "App of Apps" pattern uses one ArgoCD Application to manage other Applications.
This is how you bootstrap an entire platform from a single Git repository.

```
platform-repo/
├── apps/                    # App of Apps root
│   └── kustomization.yaml   # Lists all Application manifests
├── applications/            # Individual Application manifests
│   ├── myapp.yaml
│   ├── monitoring.yaml
│   ├── ingress-nginx.yaml
│   ├── cert-manager.yaml
│   └── keda.yaml
└── README.md
```

```yaml
# applications/monitoring.yaml
# ArgoCD Application for the monitoring stack.
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: monitoring
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: infrastructure
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: 55.5.0
    helm:
      releaseName: kube-prometheus-stack
      valueFiles: []
      values: |
        prometheus:
          prometheusSpec:
            retention: 15d
            serviceMonitorSelectorNilUsesHelmValues: false
        grafana:
          adminPassword: changeme
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```yaml
# applications/myapp.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/myapp-deploy.git
    targetRevision: main
    path: overlays/production
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```yaml
# apps/kustomization.yaml
# The "App of Apps" — this is what the root Application points to.
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../applications/monitoring.yaml
  - ../applications/myapp.yaml
  - ../applications/ingress-nginx.yaml
  - ../applications/cert-manager.yaml
  - ../applications/keda.yaml
```

```yaml
# root-application.yaml
# The one Application to rule them all.
# This is the only thing you manually apply to bootstrap the platform.
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: platform
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/myorg/platform-repo.git
    targetRevision: main
    path: apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

```bash
# Bootstrap the entire platform with one command
kubectl apply -f root-application.yaml

# ArgoCD will:
# 1. Sync the root Application → discovers all child Applications
# 2. Sync each child Application → deploys monitoring, myapp, etc.
# 3. Continue watching Git for changes
```

### 38.3.6 ApplicationSets

ApplicationSets generate Applications from templates — useful for deploying the
same application to multiple clusters or environments.

```yaml
# applicationset-environments.yaml
# Generates one Application per environment (dev, staging, production).
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: myapp-environments
  namespace: argocd
spec:
  generators:
    # List generator — explicitly define each environment
    - list:
        elements:
          - env: development
            namespace: development
            branch: develop
            replicas: "1"
            cluster: https://kubernetes.default.svc
          - env: staging
            namespace: staging
            branch: main
            replicas: "3"
            cluster: https://kubernetes.default.svc
          - env: production
            namespace: production
            branch: main
            replicas: "5"
            cluster: https://production-cluster.example.com
  template:
    metadata:
      name: 'myapp-{{env}}'
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/myapp-deploy.git
        targetRevision: '{{branch}}'
        path: 'overlays/{{env}}'
      destination:
        server: '{{cluster}}'
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
        syncOptions:
          - CreateNamespace=true
```

```yaml
# applicationset-clusters.yaml
# Generates Applications for ALL clusters registered in ArgoCD.
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: monitoring-all-clusters
  namespace: argocd
spec:
  generators:
    # Cluster generator — creates an Application for each ArgoCD cluster
    - clusters:
        selector:
          matchLabels:
            monitoring: enabled
  template:
    metadata:
      name: 'monitoring-{{name}}'
    spec:
      project: infrastructure
      source:
        repoURL: https://github.com/myorg/platform-repo.git
        path: monitoring
        targetRevision: main
      destination:
        server: '{{server}}'
        namespace: monitoring
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

```yaml
# applicationset-git-directories.yaml
# Generates Applications from directories in a Git repo.
# Each directory under "apps/" becomes its own Application.
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: platform-apps
  namespace: argocd
spec:
  generators:
    - git:
        repoURL: https://github.com/myorg/platform-repo.git
        revision: main
        directories:
          - path: "apps/*"
          - path: "apps/experimental/*"
            exclude: true       # Exclude experimental apps
  template:
    metadata:
      name: '{{path.basename}}'
    spec:
      project: default
      source:
        repoURL: https://github.com/myorg/platform-repo.git
        targetRevision: main
        path: '{{path}}'
      destination:
        server: https://kubernetes.default.svc
        namespace: '{{path.basename}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
```

---

## 38.4 Flux

### 38.4.1 Flux Architecture

Flux is the other major GitOps controller in the CNCF ecosystem. It uses a
**modular controller architecture** where each controller handles a specific concern.

```
┌────────────────────────────────────────────────────┐
│                       Flux                          │
│                                                     │
│  ┌────────────────┐  ┌────────────────────────┐    │
│  │ Source          │  │ Kustomize              │    │
│  │ Controller      │  │ Controller              │    │
│  │                │  │                         │    │
│  │ - GitRepository│  │ - Kustomization CRD     │    │
│  │ - HelmRepo     │──│ - Applies kustomize     │    │
│  │ - Bucket (S3)  │  │   output to cluster     │    │
│  └────────────────┘  └────────────────────────┘    │
│                                                     │
│  ┌────────────────┐  ┌────────────────────────┐    │
│  │ Helm           │  │ Notification            │    │
│  │ Controller      │  │ Controller              │    │
│  │                │  │                         │    │
│  │ - HelmRelease  │  │ - Alerts → Slack/Teams  │    │
│  │ - Installs/    │  │ - Webhook receivers     │    │
│  │   upgrades     │  │                         │    │
│  │   Helm charts  │  │                         │    │
│  └────────────────┘  └────────────────────────┘    │
└────────────────────────────────────────────────────┘
```

### 38.4.2 Key CRDs

```yaml
# flux-gitrepository.yaml
# GitRepository tells Flux where to find your manifests.
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 1m           # How often to check for new commits
  url: https://github.com/myorg/myapp-deploy.git
  ref:
    branch: main
  secretRef:
    name: github-credentials    # SSH key or token for private repos
```

```yaml
# flux-kustomization.yaml
# Kustomization tells Flux what to apply from the GitRepository.
# (This is Flux's Kustomization CRD, not to be confused with
# the kustomize kustomization.yaml file.)
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 5m             # How often to reconcile
  sourceRef:
    kind: GitRepository
    name: myapp
  path: ./overlays/production    # Path within the repo
  prune: true                    # Delete resources removed from Git
  targetNamespace: production
  healthChecks:
    - apiVersion: apps/v1
      kind: Deployment
      name: myapp-server
      namespace: production
  timeout: 3m
```

```yaml
# flux-helmrelease.yaml
# HelmRelease installs a Helm chart via Flux.
apiVersion: helm.toolkit.fluxcd.io/v2beta2
kind: HelmRelease
metadata:
  name: kube-prometheus-stack
  namespace: monitoring
spec:
  interval: 10m
  chart:
    spec:
      chart: kube-prometheus-stack
      version: "55.x"
      sourceRef:
        kind: HelmRepository
        name: prometheus-community
        namespace: flux-system
  values:
    prometheus:
      prometheusSpec:
        retention: 15d
    grafana:
      adminPassword: changeme
  # Upgrade and rollback configuration
  upgrade:
    remediation:
      retries: 3
  rollback:
    cleanupOnFail: true
```

### 38.4.3 Hands-On: Flux Setup Basics

```bash
# Install the Flux CLI
# (macOS)  brew install fluxcd/tap/flux
# (Linux)  curl -s https://fluxcd.io/install.sh | sudo bash

# Check prerequisites
flux check --pre

# Bootstrap Flux (creates the flux-system namespace and controllers,
# commits the Flux manifests back to your Git repo)
flux bootstrap github \
  --owner=myorg \
  --repository=platform-repo \
  --path=clusters/production \
  --personal

# This creates:
# clusters/production/flux-system/
#   ├── gotk-components.yaml    # Flux controllers
#   ├── gotk-sync.yaml          # Self-referential GitRepository + Kustomization
#   └── kustomization.yaml

# Add a GitRepository source
flux create source git myapp \
  --url=https://github.com/myorg/myapp-deploy \
  --branch=main \
  --interval=1m

# Add a Kustomization to deploy the app
flux create kustomization myapp \
  --source=GitRepository/myapp \
  --path=./overlays/production \
  --prune=true \
  --interval=5m \
  --target-namespace=production

# Check the status
flux get kustomizations
flux get sources git
flux get helmreleases -A

# Force reconciliation (don't wait for interval)
flux reconcile kustomization myapp

# Suspend/resume (useful during maintenance)
flux suspend kustomization myapp
flux resume kustomization myapp
```

### 38.4.4 ArgoCD vs Flux

| Feature           | ArgoCD                    | Flux                        |
|-------------------|---------------------------|-----------------------------|
| Architecture      | Monolithic (3 components) | Modular (4+ controllers)    |
| UI                | Rich web UI               | CLI only (Weave GitOps UI)  |
| Multi-cluster     | Built-in                  | Via Kustomization CRD       |
| Helm support      | Template-only (no Tiller) | Full lifecycle (HelmRelease)|
| RBAC              | Built-in (projects)       | Kubernetes RBAC             |
| Notifications     | Built-in                  | Notification controller     |
| Learning curve    | Moderate                  | Lower                       |
| Ecosystem         | Large (CNCF graduated)    | Large (CNCF graduated)      |

**Choose ArgoCD when:** You need a web UI, multi-cluster management from a
single control plane, or RBAC beyond Kubernetes native RBAC.

**Choose Flux when:** You prefer a lightweight, modular approach, want native
Helm lifecycle management, or want everything as Kubernetes CRDs.

---

## 38.5 GitOps Best Practices

### 38.5.1 Repository Structure

**Monorepo** — All environments in one repository:

```
platform/
├── apps/
│   ├── myapp/
│   │   ├── base/
│   │   └── overlays/
│   │       ├── development/
│   │       ├── staging/
│   │       └── production/
│   ├── inventory-service/
│   │   ├── base/
│   │   └── overlays/
│   └── payment-service/
│       ├── base/
│       └── overlays/
├── infrastructure/
│   ├── monitoring/
│   ├── ingress/
│   └── cert-manager/
└── clusters/
    ├── development/
    ├── staging/
    └── production/
```

**Polyrepo** — Separate repos for apps and infrastructure:

```
# Repo 1: myorg/myapp-deploy (application manifests)
myapp-deploy/
├── base/
└── overlays/
    ├── development/
    ├── staging/
    └── production/

# Repo 2: myorg/platform-infra (infrastructure)
platform-infra/
├── monitoring/
├── ingress/
└── cert-manager/

# Repo 3: myorg/platform-config (ArgoCD applications)
platform-config/
├── applications/
└── applicationsets/
```

**Recommendation:** Start with a monorepo. Split into polyrepo when you have
distinct teams that need separate access control, review processes, or release
cadences.

### 38.5.2 Environment Promotion

Promote changes through environments systematically:

```
dev → staging → production

1. Developer merges PR to main branch
2. CI builds and pushes container image
3. CI updates image tag in overlays/development/
4. ArgoCD syncs development automatically
5. Automated tests run against development
6. (Manual or automated) PR created to update staging
7. ArgoCD syncs staging
8. Integration tests run against staging
9. (Manual) PR approved to update production
10. ArgoCD syncs production
```

**Automated image promotion with Flux:**

```yaml
# Image automation — Flux scans a container registry and updates Git
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageRepository
metadata:
  name: myapp
  namespace: flux-system
spec:
  image: myregistry/myapp
  interval: 1m
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImagePolicy
metadata:
  name: myapp
  namespace: flux-system
spec:
  imageRepositoryRef:
    name: myapp
  policy:
    semver:
      range: ">=3.0.0"     # Only promote semver tags >= 3.0.0
---
apiVersion: image.toolkit.fluxcd.io/v1beta2
kind: ImageUpdateAutomation
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 1m
  sourceRef:
    kind: GitRepository
    name: myapp
  git:
    checkout:
      ref:
        branch: main
    commit:
      author:
        email: flux@example.com
        name: Flux
      messageTemplate: "chore: update myapp to {{.NewTag}}"
    push:
      branch: main
  update:
    path: ./overlays/production
    strategy: Setters
```

### 38.5.3 Secret Management in GitOps

Secrets cannot be stored in Git as plain text. Here are the main approaches:

#### Sealed Secrets (Bitnami)

Encrypts secrets with a cluster-specific key. Only the cluster can decrypt them.

```bash
# Install the controller
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace kube-system

# Install the CLI
# brew install kubeseal

# Create a regular secret
kubectl create secret generic myapp-secrets \
  --from-literal=DATABASE_PASSWORD=supersecret \
  --dry-run=client -o yaml > secret.yaml

# Seal it (encrypts with the cluster's public key)
kubeseal --format yaml < secret.yaml > sealed-secret.yaml

# The sealed secret is safe to commit to Git
cat sealed-secret.yaml
```

```yaml
# sealed-secret.yaml — SAFE to commit to Git
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: myapp-secrets
  namespace: production
spec:
  encryptedData:
    DATABASE_PASSWORD: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq... # encrypted
```

#### SOPS (Mozilla)

Encrypts specific values in YAML files using KMS, PGP, or age keys.

```bash
# Encrypt a secret file with SOPS + AWS KMS
sops --encrypt --kms "arn:aws:kms:us-east-1:123456:key/abc-123" \
  secret.yaml > secret.enc.yaml

# The encrypted file looks like:
# apiVersion: v1
# kind: Secret
# metadata:
#   name: myapp-secrets
# data:
#   DATABASE_PASSWORD: ENC[AES256_GCM,data:...]  # encrypted value
# sops:
#   kms:
#     - arn: arn:aws:kms:...
```

Flux has built-in SOPS support:

```yaml
# Flux Kustomization with SOPS decryption
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
spec:
  sourceRef:
    kind: GitRepository
    name: myapp
  path: ./overlays/production
  prune: true
  # Enable SOPS decryption for this Kustomization
  decryption:
    provider: sops
    secretRef:
      name: sops-age    # Age private key stored as a K8s Secret
```

#### External Secrets Operator

Syncs secrets from external providers (AWS Secrets Manager, HashiCorp Vault,
Azure Key Vault, GCP Secret Manager) into Kubernetes Secrets.

```yaml
# external-secret.yaml
# Fetches secrets from AWS Secrets Manager and creates a K8s Secret.
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: myapp-secrets
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: aws-secrets-manager
  target:
    name: myapp-secrets
    creationPolicy: Owner
  data:
    - secretKey: DATABASE_PASSWORD
      remoteRef:
        key: production/myapp
        property: database_password
    - secretKey: API_KEY
      remoteRef:
        key: production/myapp
        property: api_key
```

### 38.5.4 Complete GitOps Workflow

Here is the complete workflow from code change to production deployment:

```
┌─────────────┐     ┌─────────────┐     ┌─────────────────────┐
│  Developer   │     │  CI/CD      │     │  GitOps Controller  │
│              │     │  Pipeline   │     │  (ArgoCD/Flux)      │
│  1. Write    │     │             │     │                     │
│     code     │     │  3. Build   │     │  5. Detect change   │
│  2. Push to  │────▶│     image   │     │     in Git          │
│     Git      │     │  4. Update  │────▶│  6. Render          │
│              │     │     image   │     │     manifests        │
│              │     │     tag in  │     │  7. Apply to        │
│              │     │     deploy  │     │     cluster          │
│              │     │     repo    │     │  8. Report status    │
└─────────────┘     └─────────────┘     └─────────────────────┘
```

```go
// ci_update.go
//
// A simple Go program used in CI/CD pipelines to update the image
// tag in a Kustomize overlay after building a new container image.
// This bridges the CI (build) and CD (deploy) pipelines by committing
// the new image tag to the deploy repository.
//
// Usage:
//
//	go run ci_update.go \
//	  --repo=myorg/myapp-deploy \
//	  --path=overlays/production \
//	  --image=myregistry/myapp \
//	  --tag=v3.2.0
//
// This creates a commit in the deploy repo with the updated image tag,
// which ArgoCD/Flux detects and syncs to the cluster.
package main

import (
	"flag"
	"fmt"
	"log"
	"os"
	"os/exec"
)

func main() {
	repo := flag.String("repo", "", "GitHub repository (owner/name)")
	path := flag.String("path", "", "Path to Kustomize overlay")
	image := flag.String("image", "", "Container image name")
	tag := flag.String("tag", "", "New image tag")
	flag.Parse()

	if *repo == "" || *path == "" || *image == "" || *tag == "" {
		flag.Usage()
		os.Exit(1)
	}

	// Clone the deploy repository.
	repoURL := fmt.Sprintf("https://github.com/%s.git", *repo)
	tmpDir := "deploy-repo-checkout"
	run("git", "clone", "--depth=1", repoURL, tmpDir)
	defer os.RemoveAll(tmpDir)

	// Update the image tag using kustomize edit.
	kustomizePath := fmt.Sprintf("%s/%s", tmpDir, *path)
	run("kustomize", "edit", "set", "image",
		fmt.Sprintf("%s=%s:%s", *image, *image, *tag),
		// Note: kustomize edit must be run from the overlay directory
	)
	_ = kustomizePath // Used via chdir in production version

	// Commit and push the change.
	run("git", "-C", tmpDir, "add", "-A")
	run("git", "-C", tmpDir, "commit", "-m",
		fmt.Sprintf("chore: update %s to %s", *image, *tag))
	run("git", "-C", tmpDir, "push")

	log.Printf("Successfully updated %s to %s in %s", *image, *tag, *repo)
}

// run executes a command and logs the output. Exits on failure.
func run(name string, args ...string) {
	cmd := exec.Command(name, args...)
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr
	if err := cmd.Run(); err != nil {
		log.Fatalf("command failed: %s %v: %v", name, args, err)
	}
}
```

---

## 38.6 Summary

| Tool      | Primary Use                  | Key Feature                       |
|-----------|------------------------------|-----------------------------------|
| Helm      | Package & template manifests | Go templating, chart distribution |
| Kustomize | Overlay & patch manifests    | Template-free, built into kubectl |
| ArgoCD    | GitOps deployment            | Web UI, multi-cluster, RBAC       |
| Flux      | GitOps deployment            | Modular, native Helm lifecycle    |

**Key takeaways:**

1. **Helm for packaging** — use it when distributing charts to users or when you
   need complex templating with conditionals and loops.
2. **Kustomize for overlays** — use it for environment-specific configuration
   (dev/staging/prod) without templates.
3. **Helm + Kustomize** — they complement each other. Use Helm for third-party
   charts and Kustomize for your internal overlays.
4. **GitOps is the deployment model** — ArgoCD or Flux reconcile Git state to
   cluster state continuously.
5. **App of Apps / ApplicationSets** — scale to dozens of services and clusters.
6. **Never store secrets in Git** — use Sealed Secrets, SOPS, or External Secrets.
7. **Automate image promotion** — CI updates Git, GitOps deploys from Git.

Together with the observability (Chapter 36) and autoscaling (Chapter 37) concepts,
you now have a complete production Kubernetes platform: applications are packaged
with Helm, deployed via GitOps, monitored with Prometheus/Grafana/OTel, and
autoscaled with HPA/KEDA — all defined declaratively in Git.
