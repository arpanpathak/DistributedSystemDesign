# Chapter 33: Custom Resource Definitions (CRDs)

> *"CRDs are the extension mechanism that turned Kubernetes from a container
> orchestrator into a universal control plane. With a single YAML file, you
> can teach the Kubernetes API server about an entirely new resource type —
> complete with validation, versioning, and kubectl support — without writing
> a single line of API server code."*

In Chapter 32 we learned how controllers reconcile desired state against
actual state. But we only worked with built-in resource types — Pods,
ConfigMaps, Namespaces. What if you want to manage something Kubernetes
does not know about? A `Website`, a `Database`, a `Certificate`, a
`KafkaTopic`?

That is what Custom Resource Definitions are for. A CRD tells the API server:
"I want a new resource type called `Website` in the `webapp.example.com` API
group." The API server dynamically creates REST endpoints for your new type.
You can immediately `kubectl get websites`, `kubectl create -f website.yaml`,
and `kubectl delete website my-site`. No code needed — just a CRD YAML.

But CRDs are only half the story. A CRD without a controller is just data in
etcd. It is the *combination* of a CRD (defining the API) and a controller
(implementing the behavior) that creates an **operator** — which is the
subject of Chapter 34. This chapter focuses on the CRD side: how to define
custom resource types, validate them, version them, and work with them in Go.

---

## 33.1 What Are CRDs?

### 33.1.1 Extending the Kubernetes API

Kubernetes has a fixed set of built-in resource types: Pods, Services,
Deployments, ConfigMaps, etc. These are compiled into the API server. You
cannot add new built-in types without recompiling the API server — which is
not practical.

CRDs provide a dynamic extension mechanism. When you create a CRD, the API
server:

1. **Registers new REST endpoints** for your resource type. For example, if
   you create a CRD for `websites.webapp.example.com`, the API server creates:
   - `GET /apis/webapp.example.com/v1/websites` — list all websites
   - `GET /apis/webapp.example.com/v1/namespaces/{ns}/websites/{name}` — get one
   - `POST /apis/webapp.example.com/v1/namespaces/{ns}/websites` — create
   - `PUT /apis/webapp.example.com/v1/namespaces/{ns}/websites/{name}` — update
   - `DELETE /apis/webapp.example.com/v1/namespaces/{ns}/websites/{name}` — delete
   - `WATCH /apis/webapp.example.com/v1/websites?watch=true` — watch for changes

2. **Validates objects** against the schema you define in the CRD.

3. **Stores objects in etcd**, just like built-in resources.

4. **Integrates with kubectl.** `kubectl get websites`, `kubectl describe
   website my-site`, and `kubectl apply -f website.yaml` all work
   automatically.

5. **Integrates with RBAC.** You can create roles that grant access to your
   custom resources, just like built-in resources.

6. **Integrates with the watch API.** Controllers can use informers to watch
   your custom resources, exactly as they watch Pods or Deployments.

### 33.1.2 CRD vs Aggregated API Server

CRDs are not the only way to extend the Kubernetes API. You can also run an
**aggregated API server** — a separate API server process that the main API
server proxies to. Aggregated API servers give you more control (you can
implement any storage backend, any validation logic, any subresource) but
are significantly more complex to build and operate.

For the vast majority of use cases, CRDs are sufficient. Use aggregated API
servers only when you need:
- A storage backend other than etcd (e.g., a metrics API backed by Prometheus).
- Custom authentication or authorization.
- Subresources beyond status and scale.
- Protocol-level customization (e.g., WebSocket support).

### 33.1.3 The CRD Lifecycle

```text
┌───────────────────────────────────────────────────────────┐
│                    CRD LIFECYCLE                           │
│                                                           │
│  1. User creates CRD YAML                                 │
│     ┌──────────────────────┐                              │
│     │ apiVersion: ...      │                              │
│     │ kind: CRD            │                              │
│     │ spec:                │                              │
│     │   group: webapp...   │                              │
│     │   names:             │                              │
│     │     kind: Website    │                              │
│     │   ...                │                              │
│     └──────────┬───────────┘                              │
│                │                                          │
│  2. kubectl apply -f crd.yaml                             │
│                │                                          │
│                ▼                                          │
│  3. API server validates CRD and creates REST endpoints   │
│                │                                          │
│                ▼                                          │
│  4. Users can now:                                        │
│     kubectl get websites                                  │
│     kubectl create -f my-website.yaml                     │
│     kubectl delete website my-site                        │
│                │                                          │
│                ▼                                          │
│  5. A controller watches Websites and takes action        │
│     (This is what makes it an "operator" — Ch 34)         │
└───────────────────────────────────────────────────────────┘
```

---

## 33.2 CRD Spec Deep Dive

### 33.2.1 The Anatomy of a CRD

Let us create a CRD for a `Website` custom resource. A Website represents a
web application that should be deployed and served in the cluster.

```yaml
# website-crd.yaml
#
# This CRD defines a "Website" custom resource in the
# webapp.example.com API group. Once applied, users can
# create Website objects and a controller can reconcile them
# into actual deployments + services.
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  # The name MUST be "{plural}.{group}".
  # This is enforced by the API server.
  name: websites.webapp.example.com
spec:
  # group is the API group for the resource. By convention,
  # it is a reversed domain name that you own or control.
  # This prevents naming collisions between different CRDs.
  group: webapp.example.com

  # names defines how the resource appears in the API.
  names:
    # kind is the CamelCase singular name used in YAML manifests.
    # This is what goes in the "kind:" field of resource YAML.
    kind: Website

    # listKind is the CamelCase name for a list of these resources.
    # Usually "{kind}List". Defaults to "{kind}List" if omitted.
    listKind: WebsiteList

    # plural is the lowercase plural name used in URLs.
    # /apis/webapp.example.com/v1/websites
    plural: websites

    # singular is the lowercase singular name used in kubectl output
    # and as an alias for the resource type.
    singular: website

    # shortNames are aliases for the resource. Users can type
    # "kubectl get ws" instead of "kubectl get websites".
    shortNames:
    - ws

    # categories group this resource with others for "kubectl get all".
    # With categories: [all], "kubectl get all" includes Websites.
    categories:
    - all
    - webapp

  # scope determines whether the resource is namespaced or cluster-scoped.
  # Namespaced: each object lives in a specific namespace.
  # Cluster: objects are cluster-wide (like Nodes or ClusterRoles).
  scope: Namespaced

  # versions defines one or more API versions for the resource.
  # Each version can have its own schema.
  versions:
  - name: v1
    # served: if true, this version is available via the API.
    # You can stop serving old versions while still storing them.
    served: true
    # storage: exactly ONE version must be the storage version.
    # This is the version stored in etcd. When objects are read,
    # they are converted to the requested version.
    storage: true
    # schema defines the structure of the resource using OpenAPI v3.
    schema:
      openAPIV3Schema:
        type: object
        description: >-
          Website represents a web application to be deployed and
          served in the Kubernetes cluster.
        properties:
          # apiVersion and kind are implicitly added by the API server.
          # metadata is also implicit. We only define spec and status.
          spec:
            type: object
            description: >-
              WebsiteSpec defines the desired state of the Website.
            required:
            - image
            - hostname
            properties:
              image:
                type: string
                description: >-
                  Container image to deploy (e.g., nginx:1.25).
                minLength: 1
              hostname:
                type: string
                description: >-
                  Hostname for the website (e.g., www.example.com).
                  Must be a valid DNS name.
                pattern: '^[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*$'
              replicas:
                type: integer
                description: >-
                  Number of replicas to run. Defaults to 1.
                minimum: 0
                maximum: 100
                default: 1
              port:
                type: integer
                description: >-
                  Container port the application listens on.
                minimum: 1
                maximum: 65535
                default: 80
              tls:
                type: object
                description: >-
                  TLS configuration for HTTPS.
                properties:
                  enabled:
                    type: boolean
                    default: false
                  secretName:
                    type: string
                    description: >-
                      Name of the Secret containing TLS certificate
                      and key.

          status:
            type: object
            description: >-
              WebsiteStatus reports the observed state of the Website.
            properties:
              phase:
                type: string
                description: >-
                  Current phase of the Website.
                enum:
                - Pending
                - Deploying
                - Running
                - Failed
              readyReplicas:
                type: integer
                description: >-
                  Number of replicas that are ready to serve traffic.
              url:
                type: string
                description: >-
                  The URL where the website is accessible.
              conditions:
                type: array
                items:
                  type: object
                  properties:
                    type:
                      type: string
                    status:
                      type: string
                      enum: ["True", "False", "Unknown"]
                    lastTransitionTime:
                      type: string
                      format: date-time
                    reason:
                      type: string
                    message:
                      type: string

    # subresources enables special API endpoints for the resource.
    subresources:
      # status enables the /status subresource, which allows
      # updating status independently from spec.
      status: {}
      # scale enables the /scale subresource, which allows
      # "kubectl scale website my-site --replicas=3".
      scale:
        specReplicasPath: .spec.replicas
        statusReplicasPath: .status.readyReplicas

    # additionalPrinterColumns defines custom columns for "kubectl get".
    additionalPrinterColumns:
    - name: Hostname
      type: string
      jsonPath: .spec.hostname
    - name: Replicas
      type: integer
      jsonPath: .spec.replicas
    - name: Ready
      type: integer
      jsonPath: .status.readyReplicas
    - name: Phase
      type: string
      jsonPath: .status.phase
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
```

### 33.2.2 Field-by-Field Breakdown

**`group`**: The API group. Convention is a reversed domain you own. Examples:
`cert-manager.io`, `monitoring.coreos.com`, `webapp.example.com`. Avoid
generic groups like `apps` or `extensions` — those are reserved for built-in
types.

**`names.kind`**: The CamelCase type name. This appears in YAML manifests as
`kind: Website`. Must be unique within the group.

**`names.plural`**: The lowercase plural used in API URLs. `kubectl get
websites` uses this. By convention, the English plural of the kind.

**`names.singular`**: The lowercase singular. Used by kubectl for display and
as an alias. Usually the lowercase of kind.

**`names.shortNames`**: Convenience aliases. `ws` is much easier to type than
`websites`. Many built-in types have short names: `po` (pods), `svc`
(services), `deploy` (deployments), `ns` (namespaces).

**`scope`**: `Namespaced` or `Cluster`. Most custom resources are namespaced.
Use cluster scope for resources that are truly global (like a cluster-wide
policy or a node-level resource).

**`versions`**: One or more API versions. Each version has its own schema,
served flag, and storage flag. We cover versioning in detail in section 33.6.

### 33.2.3 Applying the CRD

```bash
# Apply the CRD.
$ kubectl apply -f website-crd.yaml
customresourcedefinition.apiextensions.k8s.io/websites.webapp.example.com created

# Verify the CRD is registered.
$ kubectl get crd websites.webapp.example.com
NAME                          CREATED AT
websites.webapp.example.com   2024-01-15T10:30:00Z

# The API server now knows about Websites. We can list them
# (there are none yet).
$ kubectl get websites
No resources found in default namespace.

# Using short name:
$ kubectl get ws
No resources found in default namespace.
```

### 33.2.4 Creating a Custom Resource Instance

```yaml
# my-website.yaml
#
# An instance of the Website custom resource. This tells the
# (hypothetical) Website controller to deploy an nginx web
# server with the hostname blog.example.com.
apiVersion: webapp.example.com/v1
kind: Website
metadata:
  name: my-blog
  namespace: default
spec:
  image: nginx:1.25
  hostname: blog.example.com
  replicas: 3
  port: 80
  tls:
    enabled: true
    secretName: blog-tls
```

```bash
$ kubectl apply -f my-website.yaml
website.webapp.example.com/my-blog created

$ kubectl get websites
NAME      HOSTNAME           REPLICAS   READY   PHASE   AGE
my-blog   blog.example.com   3                          5s

# The custom printer columns we defined in the CRD appear automatically!
```

---

## 33.3 Schema Validation

### 33.3.1 Structural Schemas

Starting with Kubernetes 1.15, all CRDs must use **structural schemas**.
A structural schema is an OpenAPI v3 schema that follows specific rules:

1. **Every field must have a type.** No untyped fields (except at the root).
2. **No `additionalProperties` at the root level.** The schema must fully
   describe the structure.
3. **Specific types only.** `object`, `array`, `string`, `integer`, `number`,
   `boolean`. No `oneOf`, `anyOf`, `allOf`, or `not` (with some exceptions).
4. **`properties` and `items` must be at the same level as `type`.** No
   inline schemas.

These rules exist because the API server uses the schema for:
- Validation (rejecting invalid objects)
- Defaulting (setting default values)
- Pruning (removing unknown fields — "pruning" or "trimming")
- OpenAPI publishing (for client generation and documentation)

### 33.3.2 Validation Keywords

The OpenAPI v3 schema supports many validation keywords:

```yaml
# String validations
name:
  type: string
  minLength: 1        # At least 1 character
  maxLength: 253      # At most 253 characters (DNS name limit)
  pattern: '^[a-z]'   # Must start with a lowercase letter
  enum:               # Must be one of these values
  - small
  - medium
  - large

# Integer validations
replicas:
  type: integer
  minimum: 0           # At least 0
  maximum: 100         # At most 100
  default: 1           # Default value if not specified

# Array validations
ports:
  type: array
  minItems: 1          # At least 1 item
  maxItems: 10         # At most 10 items
  items:
    type: integer
    minimum: 1
    maximum: 65535

# Object validations
config:
  type: object
  required:            # These fields are mandatory
  - name
  - value
  properties:
    name:
      type: string
    value:
      type: string

# Nested objects with map-like behavior
annotations:
  type: object
  additionalProperties:
    type: string       # Values must be strings (keys are always strings)

# Nullable fields
deletionTimestamp:
  type: string
  format: date-time
  nullable: true       # Field can be null
```

### 33.3.3 Hands-On: CRD with Comprehensive Validation

```yaml
# validated-crd.yaml
#
# A CRD with extensive validation rules demonstrating most
# OpenAPI v3 validation keywords available in Kubernetes.
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.data.example.com
spec:
  group: data.example.com
  names:
    kind: Database
    listKind: DatabaseList
    plural: databases
    singular: database
    shortNames:
    - db
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        description: >-
          Database represents a managed database instance.
        required:
        - spec
        properties:
          spec:
            type: object
            description: >-
              DatabaseSpec defines the desired state of the database.
            required:
            - engine
            - version
            - storage
            properties:
              engine:
                type: string
                description: >-
                  Database engine type.
                enum:
                - postgres
                - mysql
                - redis
                - mongodb
              version:
                type: string
                description: >-
                  Database engine version. Must be semver format.
                pattern: '^\d+\.\d+(\.\d+)?$'
              storage:
                type: object
                description: >-
                  Storage configuration for the database.
                required:
                - size
                properties:
                  size:
                    type: string
                    description: >-
                      Storage size (e.g., "10Gi", "100Gi").
                    pattern: '^\d+(Mi|Gi|Ti)$'
                  storageClass:
                    type: string
                    description: >-
                      Kubernetes StorageClass to use.
                    minLength: 1
                    maxLength: 63
              replicas:
                type: integer
                description: >-
                  Number of replicas for high availability.
                minimum: 1
                maximum: 7
                default: 1
              backup:
                type: object
                description: >-
                  Backup configuration.
                properties:
                  enabled:
                    type: boolean
                    default: true
                  schedule:
                    type: string
                    description: >-
                      Cron schedule for backups (e.g., "0 2 * * *").
                    pattern: '^(\S+\s+){4}\S+$'
                  retentionDays:
                    type: integer
                    description: >-
                      Number of days to retain backups.
                    minimum: 1
                    maximum: 365
                    default: 30
              parameters:
                type: object
                description: >-
                  Engine-specific configuration parameters.
                  Keys are parameter names, values are strings.
                additionalProperties:
                  type: string
                x-kubernetes-map-type: granular

          status:
            type: object
            description: >-
              DatabaseStatus reports the observed state.
            properties:
              phase:
                type: string
                enum:
                - Provisioning
                - Running
                - Failed
                - Deleting
              endpoint:
                type: string
                description: >-
                  Connection endpoint for the database.
              readyReplicas:
                type: integer
              conditions:
                type: array
                items:
                  type: object
                  required:
                  - type
                  - status
                  properties:
                    type:
                      type: string
                    status:
                      type: string
                      enum: ["True", "False", "Unknown"]
                    lastTransitionTime:
                      type: string
                      format: date-time
                    reason:
                      type: string
                      maxLength: 256
                    message:
                      type: string
                      maxLength: 32768

    subresources:
      status: {}
      scale:
        specReplicasPath: .spec.replicas
        statusReplicasPath: .status.readyReplicas

    additionalPrinterColumns:
    - name: Engine
      type: string
      jsonPath: .spec.engine
    - name: Version
      type: string
      jsonPath: .spec.version
    - name: Replicas
      type: integer
      jsonPath: .spec.replicas
    - name: Phase
      type: string
      jsonPath: .status.phase
    - name: Endpoint
      type: string
      jsonPath: .status.endpoint
      priority: 1
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
```

### 33.3.4 CEL Validation Rules

Starting with Kubernetes 1.25, CRDs support **CEL (Common Expression
Language)** validation rules. CEL allows you to write custom validation
logic that goes beyond what OpenAPI v3 can express — cross-field validation,
conditional requirements, and complex constraints.

CEL rules are evaluated by the API server at admission time. They run in a
sandboxed environment with resource limits, so they cannot hang or consume
excessive resources.

```yaml
# cel-validation-crd.yaml
#
# Demonstrates CEL validation rules for cross-field validation
# and complex constraints that OpenAPI v3 cannot express.
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: deploymentpolicies.policy.example.com
spec:
  group: policy.example.com
  names:
    kind: DeploymentPolicy
    plural: deploymentpolicies
    singular: deploymentpolicy
    shortNames:
    - dp
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required:
            - minReplicas
            - maxReplicas
            properties:
              minReplicas:
                type: integer
                minimum: 0
              maxReplicas:
                type: integer
                minimum: 1
              targetCPU:
                type: integer
                description: >-
                  Target CPU utilization percentage for autoscaling.
                minimum: 1
                maximum: 100
              schedule:
                type: object
                description: >-
                  Maintenance window schedule.
                properties:
                  startHour:
                    type: integer
                    minimum: 0
                    maximum: 23
                  endHour:
                    type: integer
                    minimum: 0
                    maximum: 23
                  timezone:
                    type: string
                # CEL rule: endHour must be after startHour.
                x-kubernetes-validations:
                - rule: "self.endHour > self.startHour"
                  message: "endHour must be greater than startHour"

            # CEL rules at the spec level — these can reference
            # multiple fields for cross-field validation.
            x-kubernetes-validations:
            # Rule 1: maxReplicas must be >= minReplicas.
            - rule: "self.maxReplicas >= self.minReplicas"
              message: "maxReplicas must be greater than or equal to minReplicas"
              fieldPath: ".maxReplicas"

            # Rule 2: if minReplicas > 1, targetCPU must be set
            # (autoscaling requires a CPU target).
            - rule: "self.minReplicas <= 1 || has(self.targetCPU)"
              message: "targetCPU is required when minReplicas > 1"

            # Rule 3: maxReplicas must not be more than 10x minReplicas
            # (prevent runaway scaling).
            - rule: "self.minReplicas == 0 || self.maxReplicas <= self.minReplicas * 10"
              message: "maxReplicas must not exceed 10x minReplicas"
```

### 33.3.5 CEL Functions and Variables

CEL provides a rich set of built-in functions:

| Function / Operator          | Example                                  |
|------------------------------|------------------------------------------|
| `has(field)`                 | `has(self.spec.tls)` — field exists?      |
| `size(list)`                 | `size(self.spec.ports) > 0`              |
| `self`                       | Current object being validated            |
| `oldSelf`                    | Previous version (for transition rules)   |
| String functions             | `self.name.startsWith("prod-")`          |
| List functions               | `self.items.all(x, x.size > 0)`         |
| `url()`, `ip()`, `cidr()`   | `ip(self.address).family() == 4`         |
| Regex                        | `self.name.matches('^[a-z]+$')`          |

Transition rules (using `oldSelf`) are particularly powerful — they let you
enforce immutability:

```yaml
x-kubernetes-validations:
# Immutable field — cannot be changed after creation.
- rule: "oldSelf == self"
  message: "engine is immutable once set"
```

---

## 33.4 Subresources

### 33.4.1 The Status Subresource

The status subresource is one of the most important CRD features. It creates
a separate API endpoint (`/status`) for updating the `.status` field of a
resource, independent of the `.spec`.

**Why this matters:**

1. **Separation of concerns.** Users control `.spec`. Controllers control
   `.status`. The status subresource enforces this boundary — a `status
   update` call can only modify `.status`, not `.spec` or `.metadata`.

2. **Avoiding infinite loops.** If a controller updates both spec and status
   in one call, the spec change triggers another reconciliation, which updates
   status, which triggers another reconciliation... The status subresource
   breaks this cycle.

3. **Conflict resolution.** Spec and status have separate resource versions.
   A user updating spec does not conflict with a controller updating status.

```yaml
# Enable the status subresource in your CRD:
subresources:
  status: {}
```

With the status subresource enabled:

```bash
# Regular update — can only modify spec and metadata.
# Any changes to .status are silently ignored.
kubectl apply -f my-website.yaml

# Status update — can only modify status.
# Any changes to .spec are silently ignored.
# (Typically done by the controller, not users.)
kubectl proxy &
curl -X PUT http://localhost:8001/apis/webapp.example.com/v1/namespaces/default/websites/my-blog/status \
  -H "Content-Type: application/json" \
  -d '{"status":{"phase":"Running","readyReplicas":3}}'
```

### 33.4.2 The Scale Subresource

The scale subresource enables `kubectl scale` for your custom resource:

```yaml
subresources:
  scale:
    # Where to read the desired replica count from the spec.
    specReplicasPath: .spec.replicas
    # Where to read the current replica count from the status.
    statusReplicasPath: .status.readyReplicas
    # Optional: label selector for the pods (for HPA integration).
    # labelSelectorPath: .status.selector
```

With this enabled:

```bash
$ kubectl scale website my-blog --replicas=5
website.webapp.example.com/my-blog scaled

# This updates .spec.replicas to 5, triggering a reconciliation.
```

The scale subresource also enables the **Horizontal Pod Autoscaler (HPA)**
to work with your custom resource. If you provide `labelSelectorPath`, the
HPA can query metrics for the pods managed by your resource and automatically
adjust the replica count.

### 33.4.3 Hands-On: Working with Subresources in Go

```go
// updateStatus demonstrates updating the status subresource of a
// custom resource using controller-runtime. The Status().Update()
// method sends a request to the /status endpoint, which only modifies
// .status — not .spec or .metadata.
//
// This is the standard pattern used by all Kubernetes controllers
// to report observed state back to the API server.
func (r *WebsiteReconciler) updateStatus(
	ctx context.Context,
	website *webappv1.Website,
	phase string,
	readyReplicas int32,
) error {
	// Modify the status fields on the in-memory object.
	website.Status.Phase = phase
	website.Status.ReadyReplicas = readyReplicas
	website.Status.URL = fmt.Sprintf("https://%s", website.Spec.Hostname)

	// Set a condition to provide detailed status information.
	meta.SetStatusCondition(&website.Status.Conditions, metav1.Condition{
			Type:               "Ready",
			Status:             metav1.ConditionTrue,
			Reason:             "AllReplicasReady",
			Message:            fmt.Sprintf("%d/%d replicas ready", readyReplicas, website.Spec.Replicas),
			LastTransitionTime: metav1.Now(),
		})

	// Status().Update() calls the /status subresource endpoint.
	// Only .status changes are persisted — any .spec changes are ignored.
	return r.Status().Update(ctx, website)
}
```

---

## 33.5 Additional Printer Columns

### 33.5.1 Custom kubectl Output

By default, `kubectl get` only shows NAME and AGE for custom resources. That
is not very useful. Additional printer columns let you add custom columns:

```yaml
additionalPrinterColumns:
# Standard columns (always visible with kubectl get).
- name: Engine
  type: string
  description: Database engine type.
  jsonPath: .spec.engine
- name: Version
  type: string
  jsonPath: .spec.version
- name: Phase
  type: string
  jsonPath: .status.phase
- name: Age
  type: date
  jsonPath: .metadata.creationTimestamp

# Priority 1 columns — only visible with kubectl get -o wide.
- name: Endpoint
  type: string
  jsonPath: .status.endpoint
  priority: 1
- name: Storage
  type: string
  jsonPath: .spec.storage.size
  priority: 1
```

```bash
$ kubectl get databases
NAME        ENGINE     VERSION   PHASE     AGE
mydb        postgres   16.1      Running   5d
cache       redis      7.2       Running   3d

$ kubectl get databases -o wide
NAME   ENGINE    VERSION  PHASE    AGE  ENDPOINT                    STORAGE
mydb   postgres  16.1     Running  5d   mydb.default.svc:5432       100Gi
cache  redis     7.2      Running  3d   cache.default.svc:6379      10Gi
```

### 33.5.2 Column Types

| Type      | Description                                | Example              |
|-----------|--------------------------------------------|----------------------|
| `string`  | Plain text                                 | `Running`            |
| `integer` | Integer number                             | `3`                  |
| `number`  | Floating-point number                      | `0.95`               |
| `boolean` | True/false                                 | `true`               |
| `date`    | Timestamp (displayed as age: "5d", "10m")  | `2024-01-15T10:30Z`  |

---

## 33.6 CRD Versioning

### 33.6.1 Why Version?

APIs evolve. You will inevitably need to add fields, change types, or
restructure your custom resource. Kubernetes CRD versioning lets you:

1. **Introduce new versions** without breaking existing clients.
2. **Deprecate old versions** gradually.
3. **Convert between versions** transparently.

### 33.6.2 Version Lifecycle

The typical version lifecycle for a CRD:

```
v1alpha1 → v1beta1 → v1

v1alpha1: Experimental. May change at any time. Not for production.
v1beta1:  Feature-complete. Backward-compatible changes only.
v1:       Stable. No breaking changes within the same major version.
```

### 33.6.3 Multiple Versions

A CRD can serve multiple versions simultaneously:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: websites.webapp.example.com
spec:
  group: webapp.example.com
  names:
    kind: Website
    plural: websites
  scope: Namespaced
  versions:
  # v1 — the current stable version. This is the storage version.
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required:
            - image
            - hostname
            properties:
              image:
                type: string
              hostname:
                type: string
              replicas:
                type: integer
                default: 1
              port:
                type: integer
                default: 80
              # New field added in v1 (not present in v1beta1).
              resources:
                type: object
                properties:
                  cpu:
                    type: string
                  memory:
                    type: string
          status:
            type: object
            properties:
              phase:
                type: string
              readyReplicas:
                type: integer
    subresources:
      status: {}

  # v1beta1 — the old version. Still served for backward compatibility,
  # but not the storage version.
  - name: v1beta1
    served: true
    storage: false
    # deprecated: true marks this version as deprecated in API discovery.
    deprecated: true
    deprecationWarning: "webapp.example.com/v1beta1 is deprecated; use v1"
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required:
            - image
            - hostname
            properties:
              image:
                type: string
              hostname:
                type: string
              replicas:
                type: integer
                default: 1
          status:
            type: object
            properties:
              phase:
                type: string
    subresources:
      status: {}
```

### 33.6.4 Storage Version

Exactly one version must be marked `storage: true`. This is the version that
objects are stored in etcd as. When a client requests a different version, the
API server converts on the fly.

For simple conversions (adding/removing fields with defaults), the API server
handles this automatically. For complex conversions, you need a **conversion
webhook**.

### 33.6.5 Conversion Webhooks

A conversion webhook is an HTTPS endpoint that the API server calls to convert
objects between versions. It is needed when the structural changes between
versions cannot be handled by simple field addition/removal.

```yaml
# In the CRD spec, configure the conversion strategy:
spec:
  conversion:
    strategy: Webhook
    webhook:
      conversionReviewVersions: ["v1"]
      clientConfig:
        # Option 1: Service reference (if webhook runs in-cluster).
        service:
          name: website-webhook
          namespace: webapp-system
          path: /convert
          port: 443
        # Option 2: URL (if webhook runs outside the cluster).
        # url: https://webhook.example.com/convert
        #
        # The CA bundle validates the webhook's TLS certificate.
        caBundle: LS0tLS1C...  # base64-encoded CA cert
```

### 33.6.6 Hands-On: Conversion Webhook in Go

```go
// Package webhook implements a CRD conversion webhook that converts
// Website resources between v1beta1 and v1 versions.
//
// Conversion flow:
//   Client requests v1 → API server has v1beta1 in etcd → API server
//   calls this webhook to convert v1beta1 → v1 → returns v1 to client.
//
// The webhook must handle ALL version pairs:
//   v1beta1 → v1   (reading old objects as new version)
//   v1 → v1beta1   (reading new objects as old version)
//
// Conversion must be lossless for fields that exist in both versions.
// Fields that only exist in one version may use defaults when converting.
package webhook

import (
	"encoding/json"
	"fmt"
	"io"
	"net/http"

	apiextv1 "k8s.io/apiextensions-apiserver/pkg/apis/apiextensions/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/klog/v2"
)

// ConvertHandler handles conversion webhook requests from the API server.
// Each request contains one or more objects to convert from one version
// to another.
func ConvertHandler(w http.ResponseWriter, r *http.Request) {
	// Read the request body.
	body, err := io.ReadAll(r.Body)
	if err != nil {
		http.Error(w, "reading body: "+err.Error(), http.StatusBadRequest)
		return
	}

	// Deserialize the ConversionReview.
	var review apiextv1.ConversionReview
	if err := json.Unmarshal(body, &review); err != nil {
		http.Error(w, "unmarshaling: "+err.Error(), http.StatusBadRequest)
		return
	}

	// Process each object in the request.
	convertedObjects := make([]apiextv1.RawExtension, len(review.Request.Objects))
	for i, raw := range review.Request.Objects {
		// Parse the object as unstructured (generic JSON map).
		var obj unstructured.Unstructured
		if err := json.Unmarshal(raw.Raw, &obj.Object); err != nil {
			sendError(w, &review, fmt.Sprintf("parsing object %d: %v", i, err))
			return
		}

		// Convert to the desired version.
		converted, err := convert(&obj, review.Request.DesiredAPIVersion)
		if err != nil {
			sendError(w, &review, fmt.Sprintf("converting object %d: %v", i, err))
			return
		}

		// Serialize the converted object.
		raw, err := json.Marshal(converted.Object)
		if err != nil {
			sendError(w, &review, fmt.Sprintf("marshaling object %d: %v", i, err))
			return
		}
		convertedObjects[i] = apiextv1.RawExtension{Raw: raw}
	}

	// Build the response.
	review.Response = &apiextv1.ConversionResponse{
		UID:              review.Request.UID,
		ConvertedObjects: convertedObjects,
		Result:           metav1.Status{Status: "Success"},
	}
	review.Request = nil

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(review)
}

// convert transforms a single object to the desired API version.
// It dispatches to version-specific conversion functions based on
// the source and target versions.
//
// The conversion logic must:
//   - Preserve ALL fields that exist in both versions.
//   - Set sensible defaults for fields that only exist in the target.
//   - Drop fields that do not exist in the target (but preserve them
	//     in annotations if round-trip fidelity is needed).
func convert(
	obj *unstructured.Unstructured,
	desiredVersion string,
) (*unstructured.Unstructured, error) {
	currentVersion := obj.GetAPIVersion()
	klog.V(2).Infof("Converting %s/%s from %s to %s",
		obj.GetNamespace(), obj.GetName(), currentVersion, desiredVersion)

	// If already the desired version, return as-is.
	if currentVersion == desiredVersion {
		return obj, nil
	}

	switch {
	case currentVersion == "webapp.example.com/v1beta1" &&
		desiredVersion == "webapp.example.com/v1":
		return convertV1beta1ToV1(obj)

	case currentVersion == "webapp.example.com/v1" &&
		desiredVersion == "webapp.example.com/v1beta1":
		return convertV1ToV1beta1(obj)

	default:
		return nil, fmt.Errorf(
			"unsupported conversion: %s → %s", currentVersion, desiredVersion)
	}
}

// convertV1beta1ToV1 converts a Website from v1beta1 to v1.
// v1 adds a "port" field (default 80) and a "resources" field.
func convertV1beta1ToV1(obj *unstructured.Unstructured) (*unstructured.Unstructured, error) {
	// Change the API version.
	obj.SetAPIVersion("webapp.example.com/v1")

	// Set default values for fields that exist in v1 but not v1beta1.
	spec, ok := obj.Object["spec"].(map[string]interface{})
	if ok {
		if _, exists := spec["port"]; !exists {
			spec["port"] = int64(80)
		}
		// resources is optional, so we do not set a default.
	}

	return obj, nil
}

// convertV1ToV1beta1 converts a Website from v1 to v1beta1.
// v1beta1 does not have "port" or "resources", so we drop them.
func convertV1ToV1beta1(obj *unstructured.Unstructured) (*unstructured.Unstructured, error) {
	obj.SetAPIVersion("webapp.example.com/v1beta1")

	// Drop fields that do not exist in v1beta1.
	spec, ok := obj.Object["spec"].(map[string]interface{})
	if ok {
		delete(spec, "port")
		delete(spec, "resources")
	}

	return obj, nil
}

// sendError sends an error ConversionResponse back to the API server.
func sendError(w http.ResponseWriter, review *apiextv1.ConversionReview, msg string) {
	klog.Error(msg)
	review.Response = &apiextv1.ConversionResponse{
		UID: review.Request.UID,
		Result: metav1.Status{
			Status:  "Failure",
			Message: msg,
		},
	}
	review.Request = nil
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(review)
}
```

---

## 33.7 Categories and Short Names

### 33.7.1 Categories

Categories let you group your custom resources with built-in resources:

```yaml
names:
  categories:
  - all      # "kubectl get all" includes this resource
  - webapp   # "kubectl get webapp" lists all resources in this category
```

This is useful for discoverability. Users can type `kubectl get all` and see
your custom resources alongside Pods, Deployments, and Services.

You can also create custom categories. `kubectl get webapp` would list all
CRDs that have `webapp` in their categories.

### 33.7.2 Short Names

Short names save typing:

```yaml
names:
  shortNames:
  - ws    # kubectl get ws
  - web   # kubectl get web
```

Choose short names carefully — they must not conflict with existing resources
or other CRDs. Common conventions:
- 2-3 letter abbreviations of the kind name.
- Avoid conflicts with built-in short names (`po`, `svc`, `deploy`, `ns`,
  `cm`, `sa`, `pv`, `pvc`, etc.).

---

## 33.8 Go Types for CRDs

### 33.8.1 Why Go Types?

When you write a controller in Go (which most people do), you want to work
with strongly-typed Go structs — not generic `map[string]interface{}`. Go
types give you:

1. **Compile-time type safety.** Catch errors at build time, not runtime.
2. **IDE support.** Autocompletion, refactoring, documentation.
3. **CRD generation.** Use controller-gen to generate the CRD YAML from your
   Go types, ensuring the schema always matches the code.
4. **DeepCopy.** Required by the Kubernetes API machinery for safe concurrent
   access.

### 33.8.2 Hands-On: Full Go Type Definitions

```go
// Package v1 contains the Go types for the webapp.example.com/v1 API
// group. These types define the structure of the Website custom
// resource and are used by:
//
//   1. Controllers — to read and write Website objects with type safety.
//   2. controller-gen — to generate the CRD YAML schema.
//   3. code-generator — to generate typed clients, listers, informers.
//
// The +kubebuilder markers on these types control CRD generation.
// controller-gen reads these markers and produces:
//   - CRD YAML (OpenAPI v3 schema, subresources, printer columns)
//   - DeepCopy methods (required by the API machinery)
//
// +kubebuilder:object:generate=true
// +groupName=webapp.example.com
package v1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// WebsiteSpec defines the desired state of a Website.
// The controller reads this spec and reconciles the cluster state
// to match it (creating Deployments, Services, Ingresses, etc.).
type WebsiteSpec struct {
	// Image is the container image to deploy.
	// Must be a valid container image reference.
	//
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:MinLength=1
	Image string `json:"image"`

	// Hostname is the DNS name where the website will be accessible.
	// Must be a valid DNS subdomain name.
	//
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:Pattern=`^[a-z0-9]([-a-z0-9]*[a-z0-9])?(\.[a-z0-9]([-a-z0-9]*[a-z0-9])?)*$`
	Hostname string `json:"hostname"`

	// Replicas is the desired number of pod replicas.
	// The controller ensures exactly this many pods are running.
	//
	// +kubebuilder:validation:Minimum=0
	// +kubebuilder:validation:Maximum=100
	// +kubebuilder:default=1
	// +optional
	Replicas *int32 `json:"replicas,omitempty"`

	// Port is the container port the application listens on.
	// The Service created by the controller will target this port.
	//
	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=65535
	// +kubebuilder:default=80
	// +optional
	Port *int32 `json:"port,omitempty"`

	// TLS configures HTTPS for the website.
	// If enabled, the controller creates an Ingress with TLS termination.
	//
	// +optional
	TLS *TLSConfig `json:"tls,omitempty"`

	// Resources defines CPU and memory limits for the web server
	// containers. If not specified, the controller uses sensible defaults.
	//
	// +optional
	Resources *ResourceRequirements `json:"resources,omitempty"`
}

// TLSConfig holds TLS/HTTPS configuration for a Website.
type TLSConfig struct {
	// Enabled controls whether HTTPS is configured.
	// When true, the controller creates a TLS Ingress.
	//
	// +kubebuilder:default=false
	Enabled bool `json:"enabled"`

	// SecretName is the name of the Kubernetes Secret containing the
	// TLS certificate and private key. The Secret must be of type
	// kubernetes.io/tls.
	//
	// +optional
	SecretName string `json:"secretName,omitempty"`
}

// ResourceRequirements defines compute resource requirements.
type ResourceRequirements struct {
	// CPU is the CPU limit (e.g., "500m", "1").
	//
	// +optional
	CPU string `json:"cpu,omitempty"`

	// Memory is the memory limit (e.g., "128Mi", "1Gi").
	//
	// +optional
	Memory string `json:"memory,omitempty"`
}

// WebsiteStatus reports the observed state of a Website.
// This is written by the controller, not by users.
type WebsiteStatus struct {
	// Phase is the high-level summary of the Website's state.
	//
	// +kubebuilder:validation:Enum=Pending;Deploying;Running;Failed
	// +optional
	Phase string `json:"phase,omitempty"`

	// ReadyReplicas is the number of replicas that are ready to
	// serve traffic. Used by the scale subresource and HPA.
	//
	// +optional
	ReadyReplicas int32 `json:"readyReplicas,omitempty"`

	// URL is the full URL where the website is accessible.
	//
	// +optional
	URL string `json:"url,omitempty"`

	// Conditions represent the latest available observations of the
	// Website's state. Standard Kubernetes condition pattern.
	//
	// +optional
	// +listType=map
	// +listMapKey=type
	Conditions []metav1.Condition `json:"conditions,omitempty"`
}

// Website is the Schema for the websites API. It represents a web
// application to be deployed and served in the Kubernetes cluster.
//
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:subresource:scale:specpath=.spec.replicas,statuspath=.status.readyReplicas
// +kubebuilder:printcolumn:name="Hostname",type=string,JSONPath=`.spec.hostname`
// +kubebuilder:printcolumn:name="Replicas",type=integer,JSONPath=`.spec.replicas`
// +kubebuilder:printcolumn:name="Ready",type=integer,JSONPath=`.status.readyReplicas`
// +kubebuilder:printcolumn:name="Phase",type=string,JSONPath=`.status.phase`
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`
// +kubebuilder:resource:shortName=ws,categories={all,webapp}
type Website struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   WebsiteSpec   `json:"spec,omitempty"`
	Status WebsiteStatus `json:"status,omitempty"`
}

// WebsiteList contains a list of Website objects.
// This type is required by the Kubernetes API machinery for list
// operations (kubectl get websites).
//
// +kubebuilder:object:root=true
type WebsiteList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []Website `json:"items"`
}
```

### 33.8.3 The Kubebuilder Markers Explained

| Marker | Purpose |
|--------|---------|
| `+kubebuilder:object:root=true` | This is a root API type (has TypeMeta+ObjectMeta) |
| `+kubebuilder:object:generate=true` | Generate DeepCopy methods for this package |
| `+kubebuilder:subresource:status` | Enable the /status subresource |
| `+kubebuilder:subresource:scale` | Enable the /scale subresource |
| `+kubebuilder:printcolumn` | Add custom kubectl column |
| `+kubebuilder:resource:shortName` | Set short name aliases |
| `+kubebuilder:resource:categories` | Set resource categories |
| `+kubebuilder:validation:Required` | Field is required |
| `+kubebuilder:validation:Minimum` | Minimum value for integers |
| `+kubebuilder:validation:Maximum` | Maximum value |
| `+kubebuilder:validation:MinLength` | Minimum string length |
| `+kubebuilder:validation:Pattern` | Regex pattern for strings |
| `+kubebuilder:validation:Enum` | Allowed values |
| `+kubebuilder:default` | Default value |
| `+optional` | Field is optional |
| `+listType=map` | List has map semantics (merge by key) |
| `+listMapKey=type` | Key field for map-type lists |

### 33.8.4 Generating CRD YAML from Go Types

The `controller-gen` tool reads the kubebuilder markers and generates:

1. **CRD YAML** — complete with OpenAPI v3 schema, subresources, printer
   columns, validation rules.
2. **DeepCopy methods** — `DeepCopyObject()`, `DeepCopyInto()`, `DeepCopy()`.
3. **RBAC manifests** — from `+kubebuilder:rbac` markers on reconcilers.
4. **Webhook manifests** — from `+kubebuilder:webhook` markers.

```bash
# Install controller-gen.
go install sigs.k8s.io/controller-tools/cmd/controller-gen@latest

# Generate CRD YAML from Go types.
controller-gen crd paths="./api/..." output:crd:artifacts:config=config/crd/bases

# Generate DeepCopy methods.
controller-gen object paths="./api/..."

# Generate both at once.
controller-gen crd object paths="./api/..." output:crd:artifacts:config=config/crd/bases
```

The generated CRD YAML will match your Go types exactly. If you add a field to
the Go type and run `controller-gen`, the CRD schema is automatically updated.
This eliminates the error-prone process of manually keeping Go types and CRD
YAML in sync.

### 33.8.5 The `zz_generated.deepcopy.go` File

controller-gen generates a file called `zz_generated.deepcopy.go` that
contains DeepCopy methods for all your types. These methods are required by
the Kubernetes API machinery for safe concurrent access to API objects.

```go
// zz_generated.deepcopy.go — AUTO-GENERATED. DO NOT EDIT.
//
// DeepCopyObject returns a deep copy of the Website as a
// runtime.Object interface. This is called by the API machinery
// when it needs to copy an API object (e.g., for caching,
	// informer event delivery, etc.).
func (in *Website) DeepCopyObject() runtime.Object {
	if c := in.DeepCopy(); c != nil {
		return c
	}
	return nil
}

// DeepCopy returns a complete deep copy of the Website struct.
// All nested pointers, slices, and maps are copied (not shared).
func (in *Website) DeepCopy() *Website {
	if in == nil {
		return nil
	}
	out := new(Website)
	in.DeepCopyInto(out)
	return out
}

// DeepCopyInto copies all fields from the receiver into the target.
// This is the core implementation used by DeepCopy and DeepCopyObject.
func (in *Website) DeepCopyInto(out *Website) {
	*out = *in
	out.TypeMeta = in.TypeMeta
	in.ObjectMeta.DeepCopyInto(&out.ObjectMeta)
	in.Spec.DeepCopyInto(&out.Spec)
	in.Status.DeepCopyInto(&out.Status)
}
```

You never edit this file manually. It is regenerated every time you run
`controller-gen object`.

---

## 33.9 Working with Custom Resources in Go

### 33.9.1 Typed Client (controller-runtime)

If you defined Go types (section 33.8), controller-runtime's client works
natively with them:

```go
// Package main demonstrates typed CRUD operations on custom resources
// using controller-runtime's client. The client uses informer-backed
// caching for reads and direct API server calls for writes.
package main

import (
	"context"
	"fmt"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"

	// Import your generated types package.
	webappv1 "example.com/website-operator/api/v1"
)

func main() {
	// Create a controller-runtime client. This client knows about
	// both built-in types and your custom types (via the scheme).
	scheme := runtime.NewScheme()
	webappv1.AddToScheme(scheme)

	cl, err := client.New(ctrl.GetConfigOrDie(), client.Options{
			Scheme: scheme,
		})
	if err != nil {
		panic(err)
	}

	ctx := context.Background()

	// ---------- CREATE ----------
	// Create a new Website custom resource.
	website := &webappv1.Website{
		ObjectMeta: metav1.ObjectMeta{
			Name:      "my-blog",
			Namespace: "default",
		},
		Spec: webappv1.WebsiteSpec{
			Image:    "nginx:1.25",
			Hostname: "blog.example.com",
			Replicas: int32Ptr(3),
		},
	}
	if err := cl.Create(ctx, website); err != nil {
		panic(err)
	}
	fmt.Println("Created website:", website.Name)

	// ---------- READ ----------
	// Get a specific Website by namespace/name.
	var fetched webappv1.Website
	err = cl.Get(ctx, types.NamespacedName{
			Namespace: "default",
			Name:      "my-blog",
		}, &fetched)
	if err != nil {
		panic(err)
	}
	fmt.Printf("Fetched: %s, image=%s, replicas=%d\n",
		fetched.Name, fetched.Spec.Image, *fetched.Spec.Replicas)

	// ---------- LIST ----------
	// List all Websites in a namespace.
	var websiteList webappv1.WebsiteList
	err = cl.List(ctx, &websiteList, client.InNamespace("default"))
	if err != nil {
		panic(err)
	}
	fmt.Printf("Found %d websites\n", len(websiteList.Items))

	// List with label selector.
	err = cl.List(ctx, &websiteList,
		client.InNamespace("default"),
		client.MatchingLabels{"app": "blog"},
	)

	// ---------- UPDATE ----------
	// Update the Website's spec.
	fetched.Spec.Replicas = int32Ptr(5)
	if err := cl.Update(ctx, &fetched); err != nil {
		panic(err)
	}
	fmt.Println("Updated replicas to 5")

	// ---------- UPDATE STATUS ----------
	// Update the Website's status (via status subresource).
	fetched.Status.Phase = "Running"
	fetched.Status.ReadyReplicas = 5
	if err := cl.Status().Update(ctx, &fetched); err != nil {
		panic(err)
	}
	fmt.Println("Updated status to Running")

	// ---------- PATCH ----------
	// Patch is safer than Update when multiple actors modify the same
	// object. MergeFrom only sends the diff, avoiding conflicts.
	patch := client.MergeFrom(fetched.DeepCopy())
	fetched.Spec.Image = "nginx:1.26"
	if err := cl.Patch(ctx, &fetched, patch); err != nil {
		panic(err)
	}
	fmt.Println("Patched image to nginx:1.26")

	// ---------- DELETE ----------
	// Delete the Website.
	if err := cl.Delete(ctx, &fetched); err != nil {
		panic(err)
	}
	fmt.Println("Deleted website:", fetched.Name)
}

// int32Ptr is a helper to get a pointer to an int32 value.
// Go does not allow taking the address of a literal, so this
// helper is needed for optional pointer fields.
func int32Ptr(i int32) *int32 {
	return &i
}
```

### 33.9.2 Dynamic Client (Unstructured)

Sometimes you do not have Go types — for example, when building a generic tool
that works with any CRD. The dynamic client uses `unstructured.Unstructured`
(essentially `map[string]interface{}`):

```go
// Package main demonstrates the dynamic client for working with
// custom resources when Go types are not available. The dynamic
// client uses unstructured objects (JSON maps) instead of typed
// structs.
//
// Use cases:
//   - Generic tools that work with any CRD (e.g., backup tools).
//   - Scripting and automation without code generation.
//   - Working with CRDs defined by third parties.
package main

import (
	"context"
	"fmt"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	// Build a dynamic client. Unlike the typed clientset, the dynamic
	// client does not need to know about specific resource types at
	// compile time.
	config, err := clientcmd.BuildConfigFromFlags("",
		clientcmd.RecommendedHomeFile)
	if err != nil {
		panic(err)
	}

	dynClient, err := dynamic.NewForConfig(config)
	if err != nil {
		panic(err)
	}

	// Define the GVR (Group/Version/Resource) for the Website CRD.
	// This is the dynamic equivalent of importing the typed package.
	websiteGVR := schema.GroupVersionResource{
		Group:    "webapp.example.com",
		Version:  "v1",
		Resource: "websites",
	}

	ctx := context.Background()

	// ---------- CREATE ----------
	// Build an unstructured object (JSON map).
	website := &unstructured.Unstructured{
		Object: map[string]interface{}{
			"apiVersion": "webapp.example.com/v1",
			"kind":       "Website",
			"metadata": map[string]interface{}{
				"name":      "dynamic-site",
				"namespace": "default",
			},
			"spec": map[string]interface{}{
				"image":    "nginx:1.25",
				"hostname": "dynamic.example.com",
				"replicas": int64(2),
			},
		},
	}

	created, err := dynClient.Resource(websiteGVR).
	Namespace("default").
	Create(ctx, website, metav1.CreateOptions{})
	if err != nil {
		panic(err)
	}
	fmt.Println("Created:", created.GetName())

	// ---------- READ ----------
	// Get a specific resource.
	fetched, err := dynClient.Resource(websiteGVR).
	Namespace("default").
	Get(ctx, "dynamic-site", metav1.GetOptions{})
	if err != nil {
		panic(err)
	}

	// Access nested fields using unstructured helpers.
	// These return the value and a bool indicating whether the field exists.
	image, found, err := unstructured.NestedString(
		fetched.Object, "spec", "image")
	if err != nil || !found {
		panic("image not found")
	}
	fmt.Println("Image:", image)

	replicas, found, err := unstructured.NestedInt64(
		fetched.Object, "spec", "replicas")
	if err != nil || !found {
		panic("replicas not found")
	}
	fmt.Println("Replicas:", replicas)

	// ---------- LIST ----------
	// List all Websites in a namespace.
	list, err := dynClient.Resource(websiteGVR).
	Namespace("default").
	List(ctx, metav1.ListOptions{})
	if err != nil {
		panic(err)
	}
	fmt.Printf("Found %d websites\n", len(list.Items))

	for _, item := range list.Items {
		name := item.GetName()
		hostname, _, _ := unstructured.NestedString(
			item.Object, "spec", "hostname")
		fmt.Printf("  %s → %s\n", name, hostname)
	}

	// ---------- UPDATE ----------
	// Modify and update using the dynamic client.
	err = unstructured.SetNestedField(
		fetched.Object, int64(5), "spec", "replicas")
	if err != nil {
		panic(err)
	}

	updated, err := dynClient.Resource(websiteGVR).
	Namespace("default").
	Update(ctx, fetched, metav1.UpdateOptions{})
	if err != nil {
		panic(err)
	}
	fmt.Println("Updated:", updated.GetName())

	// ---------- DELETE ----------
	err = dynClient.Resource(websiteGVR).
	Namespace("default").
	Delete(ctx, "dynamic-site", metav1.DeleteOptions{})
	if err != nil {
		panic(err)
	}
	fmt.Println("Deleted: dynamic-site")
}
```

### 33.9.3 Typed vs Dynamic: When to Use Each

| Aspect              | Typed Client            | Dynamic Client           |
|---------------------|-------------------------|--------------------------|
| Type safety         | Compile-time            | Runtime only             |
| IDE support         | Full autocomplete       | No autocomplete          |
| Code generation     | Required                | Not needed               |
| Generic tools       | Not suitable            | Ideal                    |
| Performance         | Slightly faster         | Slightly slower          |
| Maintenance         | Must regenerate on changes | No regeneration needed |
| Recommended for     | Your own CRDs           | Third-party CRDs, tooling |

---

## 33.10 CRD Best Practices

### 33.10.1 Always Use Structural Schemas

Every CRD should have a complete structural schema. This enables:
- Validation (the API server rejects invalid objects)
- Defaulting (the API server sets default values)
- Pruning (unknown fields are automatically removed)
- Documentation (the schema is published in OpenAPI)

Never use `x-kubernetes-preserve-unknown-fields: true` at the root level. It
disables pruning and makes your API untyped. Only use it for specific fields
that genuinely need to hold arbitrary JSON (e.g., a `config` field that
mirrors an external system's schema).

### 33.10.2 Always Use the Status Subresource

Enable the status subresource for every CRD that has a controller. This is
not optional — it is a fundamental architectural requirement:

- Users control spec. Controllers control status.
- Without the status subresource, spec and status updates conflict.
- The status subresource enables optimistic concurrency for status updates
  independent of spec updates.

```yaml
subresources:
  status: {}
```

### 33.10.3 Version Carefully

API compatibility is critical. Once you publish a version, existing clients
depend on it. Follow these rules:

1. **Adding optional fields is always safe.** Existing objects will not have
   the field, and the API server will use the default value.

2. **Removing fields is a breaking change.** Never remove a field from a
   served version. Deprecate it first, then remove it in a new version.

3. **Changing field types is a breaking change.** `integer` → `string` will
   break all existing clients.

4. **Changing validation is risky.** Making validation stricter may invalidate
   existing objects. Making it looser is usually safe.

5. **Use the standard version progression.** `v1alpha1` → `v1beta1` → `v1`.
   Alpha versions have no stability guarantees. Beta versions are feature-
   complete but may have minor changes. Stable versions must not have breaking
   changes.

### 33.10.4 Use Categories for Discoverability

Add your CRDs to the `all` category so they appear in `kubectl get all`:

```yaml
names:
  categories:
  - all
```

Create domain-specific categories for grouping related CRDs:

```yaml
# All CRDs in the database operator:
names:
  categories:
  - all
  - database   # kubectl get database → lists all database-related CRDs
```

### 33.10.5 Document Your CRD API

Your CRD is an API. Treat it like one:

1. **Add `description` to every field** in the OpenAPI schema. These
   descriptions appear in `kubectl explain website.spec`.

2. **Use `// comments` on Go types.** controller-gen converts these to
   descriptions in the generated CRD.

3. **Provide example manifests.** Show users how to create your custom
   resources.

4. **Document the controller behavior.** What happens when a field changes?
   What status fields are set? What events are recorded?

```bash
# Users can explore your CRD with kubectl explain:
$ kubectl explain website.spec
KIND:     Website
VERSION:  webapp.example.com/v1

RESOURCE: spec <Object>

DESCRIPTION:
     WebsiteSpec defines the desired state of the Website.

FIELDS:
   hostname     <string> -required-
     Hostname for the website (e.g., www.example.com).
     Must be a valid DNS name.

   image        <string> -required-
     Container image to deploy (e.g., nginx:1.25).

   replicas     <integer>
     Number of replicas to run. Defaults to 1.

   port         <integer>
     Container port the application listens on. Defaults to 80.

   tls          <Object>
     TLS configuration for HTTPS.
```

---

## 33.11 Summary

Custom Resource Definitions are the extension mechanism that makes Kubernetes
a platform, not just an orchestrator. With a CRD, you can:

1. **Define new resource types** without modifying the API server.
2. **Validate resources** with OpenAPI v3 schemas and CEL rules.
3. **Version your API** with multiple versions and conversion webhooks.
4. **Integrate with kubectl** — get, describe, explain, scale all work.
5. **Integrate with RBAC** — control access to your custom resources.
6. **Work with typed Go clients** using generated types and controller-gen.

But a CRD by itself is just data in etcd. It has no behavior. The behavior
comes from a **controller** that watches the CRD and reconciles desired state
against actual state — which is exactly what we built in Chapter 32.

The combination of a CRD (the API) and a controller (the behavior) is called
an **operator**. In Chapter 34, we will build a complete operator from scratch:
a CRD, a controller, webhooks, RBAC, deployment manifests, and tests — all
wired together into a production-ready system.

---

## 33.12 Further Reading

- [Kubernetes CRD Documentation](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
- [Kubebuilder Book](https://book.kubebuilder.io/) — the definitive guide
  to building operators with CRDs and controller-runtime
- [API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
  — the rules for Kubernetes API design
- [CEL in Kubernetes](https://kubernetes.io/docs/reference/using-api/cel/) —
  comprehensive guide to CEL validation rules
- [Structural Schemas](https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/#specifying-a-structural-schema)
  — rules and examples for structural schemas
- [API Versioning](https://kubernetes.io/docs/reference/using-api/#api-versioning)
  — the Kubernetes API versioning policy
- [Programming Kubernetes (O'Reilly)](https://www.oreilly.com/library/view/programming-kubernetes/9781492047094/)
  by Michael Hausenblas and Stefan Schimanski — deep dive into CRDs and
  controllers in Go
