# Chapter 27: RBAC — Role-Based Access Control

> *"In a multi-tenant cluster, the question is never whether someone will try to do
> something they shouldn't — it's whether the system will stop them."*

Kubernetes ships with a powerful, flexible authorization framework called **RBAC** —
Role-Based Access Control. Every `kubectl` command, every API call from a Pod's
ServiceAccount, every controller reconciling resources — they all pass through the
RBAC authorizer before the API server will execute the request.

This chapter takes you from zero to production-grade RBAC. We'll understand the
model, build roles by hand, debug permission failures, write Go programs that
manage RBAC programmatically, and avoid the mistakes that lead to cluster-wide
compromise.

---

## 27.1 What Is RBAC?

### The Authorization Question

Every request to the Kubernetes API server carries three pieces of information:

1. **Who** is making the request (the *subject*)
2. **What** they want to do (the *verb*)
3. **Which resource** they want to do it on (the *resource*)

RBAC answers a single question: **Is this subject allowed to perform this verb on
this resource?**

```text
Subject  ──→  Verb  ──→  Resource
  │              │            │
  │              │            └── pods, deployments, secrets, nodes, ...
  │              └── get, list, create, update, delete, watch, ...
  └── user:jane, group:developers, serviceaccount:default:my-sa
```

### Enabling RBAC

RBAC is enabled via the `--authorization-mode` flag on the API server. Most
production clusters (and all managed Kubernetes services) enable it by default:

```bash
# Typical API server configuration
kube-apiserver \
  --authorization-mode=Node,RBAC \
  ...
```

The `Node` authorizer handles kubelet-specific authorization. `RBAC` handles
everything else. When multiple authorizers are configured, they are evaluated in
order — if one allows the request, it proceeds; if one denies, the next authorizer
is consulted; if none allow, the request is denied.

### The RBAC API Group

All RBAC resources live under a single API group:

```
rbac.authorization.k8s.io/v1
```

This group contains exactly four resource types:

| Resource             | Scope     | Purpose                               |
|----------------------|-----------|---------------------------------------|
| `Role`               | Namespace | Permissions within a namespace        |
| `ClusterRole`        | Cluster   | Permissions across the cluster        |
| `RoleBinding`        | Namespace | Binds a Role/ClusterRole to subjects  |
| `ClusterRoleBinding` | Cluster   | Binds a ClusterRole cluster-wide      |

That's it. Four resources. The entire Kubernetes authorization model is built on
these four primitives plus a clean set of rules connecting them.

---

## 27.2 Core Concepts

### The RBAC Triangle

RBAC has three core concepts that form a triangle:

```
        ┌──────────┐
        │  Subject  │
        │           │
        │ User      │
        │ Group     │
        │ SA        │
        └─────┬─────┘
              │
              │  Binding
              │  (RoleBinding / ClusterRoleBinding)
              │
              ▼
        ┌──────────┐         ┌─────────────────────┐
        │   Role   │────────→│   Permissions        │
        │          │  rules  │                       │
        │ Role /   │         │ apiGroups: [""]       │
        │ Cluster  │         │ resources: ["pods"]   │
        │ Role     │         │ verbs: ["get","list"] │
        └──────────┘         └─────────────────────┘
```

1. **Subject** — *who*: a User, Group, or ServiceAccount
2. **Role** — *what permissions*: a set of rules defining allowed operations
3. **Binding** — *the glue*: connects a subject to a role

### Subjects

Kubernetes recognizes three kinds of subjects:

#### Users

Users are **not** Kubernetes objects. There is no `User` resource. Instead, users
are represented by strings extracted from client certificates, bearer tokens, or
external identity providers:

```yaml
# In a RoleBinding, a user looks like this:
subjects:
- kind: User
  name: "jane@example.com"
  apiGroup: rbac.authorization.k8s.io
```

The user identity comes from the authentication layer (client cert CN, OIDC token
`sub` claim, etc.). RBAC only cares about the string — it doesn't verify that the
user "exists."

#### Groups

Groups are also strings, not Kubernetes objects. They come from the authentication
layer (client cert O field, OIDC `groups` claim, etc.):

```yaml
subjects:
- kind: Group
  name: "developers"
  apiGroup: rbac.authorization.k8s.io
```

Kubernetes has several built-in groups:

| Group                            | Members                          |
|----------------------------------|----------------------------------|
| `system:authenticated`           | All authenticated users          |
| `system:unauthenticated`         | All unauthenticated requests     |
| `system:masters`                 | Superuser group — bypasses RBAC  |
| `system:serviceaccounts`         | All ServiceAccounts              |
| `system:serviceaccounts:<ns>`    | All SAs in namespace `<ns>`      |

> **Warning**: Never bind roles to `system:unauthenticated`. This gives
> permissions to anonymous requests.

#### ServiceAccounts

ServiceAccounts **are** Kubernetes objects. They live in namespaces and are the
identity mechanism for Pods:

```yaml
subjects:
- kind: ServiceAccount
  name: "deploy-bot"
  namespace: "ci-cd"
```

The API server translates a ServiceAccount into:
- User: `system:serviceaccount:<namespace>:<name>`
- Groups: `system:serviceaccounts`, `system:serviceaccounts:<namespace>`

### Verbs

RBAC verbs map directly to API operations:

| Verb               | HTTP Method | Description                            |
|--------------------|-------------|----------------------------------------|
| `get`              | GET         | Read a single resource                 |
| `list`             | GET         | List resources (collection)            |
| `watch`            | GET         | Watch for changes (streaming)          |
| `create`           | POST        | Create a new resource                  |
| `update`           | PUT         | Replace a resource entirely            |
| `patch`            | PATCH       | Partially update a resource            |
| `delete`           | DELETE      | Delete a single resource               |
| `deletecollection` | DELETE      | Delete all resources of a type         |

There are also special non-resource verbs:
- `bind` — binding roles (used by the escalation prevention system)
- `escalate` — allowing role escalation
- `impersonate` — impersonating other users
- `use` — using PodSecurityPolicies (deprecated)

### Rules

A rule is a combination of API groups, resources, verbs, and optionally resource
names:

```yaml
rules:
- apiGroups: [""]           # "" = core API group
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  resourceNames: ["my-app"] # Optional: restrict to specific resource names
```

Key points about rules:
- `apiGroups: [""]` means the core API group (pods, services, configmaps, etc.)
- `apiGroups: ["apps"]` means the apps group (deployments, statefulsets, etc.)
- `resources` are always lowercase plural
- An empty `resourceNames` list means "all resources of this type"
- Rules are **additive** — there are no deny rules in RBAC

> **Critical**: RBAC has no deny rules. You cannot say "allow everything except
> secrets." You can only say "allow these specific things." This is a deliberate
> design choice that makes the system easier to reason about.

---

## 27.3 Role and ClusterRole

### Role — Namespaced Permissions

A `Role` defines permissions within a single namespace:

```yaml
# role-pod-reader.yaml
# Allows reading pods in the "development" namespace.
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: pod-reader
rules:
  # Rule 1: Allow reading Pods (get individual, list all, watch for changes)
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
  # Rule 2: Allow reading Pod logs (subresource)
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
```

Apply it:

```bash
kubectl apply -f role-pod-reader.yaml
kubectl get role pod-reader -n development -o yaml
```

### ClusterRole — Cluster-Scoped Permissions

A `ClusterRole` can define permissions for:
1. **Cluster-scoped resources** (nodes, namespaces, persistentvolumes)
2. **Namespaced resources across all namespaces** (when used with a ClusterRoleBinding)
3. **Non-resource URLs** (/healthz, /metrics, /api, etc.)

```yaml
# clusterrole-node-reader.yaml
# Allows reading nodes and persistent volumes cluster-wide.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: node-reader
rules:
  # Nodes are cluster-scoped — they don't belong to any namespace
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "watch"]
  # PersistentVolumes are also cluster-scoped
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list", "watch"]
  # Non-resource URLs — for health checks and metrics endpoints
- nonResourceURLs: ["/healthz", "/healthz/*"]
  verbs: ["get"]
```

### Subresources

Some resources have subresources that need separate permissions:

```yaml
rules:
  # Main resource — pods
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
  # Subresource — pod logs
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get"]
  # Subresource — pod exec (very dangerous!)
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create"]
  # Subresource — pod port-forward
- apiGroups: [""]
  resources: ["pods/portforward"]
  verbs: ["create"]
  # Subresource — deployment scale
- apiGroups: ["apps"]
  resources: ["deployments/scale"]
  verbs: ["get", "update", "patch"]
```

Common subresources:

| Subresource              | Description                              |
|--------------------------|------------------------------------------|
| `pods/log`               | Container logs                           |
| `pods/exec`              | Execute commands in containers           |
| `pods/portforward`       | Port forwarding to pods                  |
| `pods/attach`            | Attach to a running container            |
| `pods/status`            | Pod status (usually read-only)           |
| `deployments/scale`      | Scale a deployment                       |
| `services/proxy`         | Proxy requests to a service              |
| `nodes/proxy`            | Proxy requests to a node                 |
| `secrets/token`          | Generate a token for a service account   |

> **Security note**: `pods/exec` is nearly as dangerous as full node access.
> Anyone who can exec into a Pod can read its secrets, access its network,
> and potentially escape to the host.

### Aggregated ClusterRoles

Aggregated ClusterRoles allow you to compose multiple ClusterRoles into one.
This is how Kubernetes builds the default `admin`, `edit`, and `view` roles:

```yaml
# The "monitoring-view" role aggregates into "view"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-view
  labels:
    # This label makes this role aggregate into the "view" ClusterRole
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
- apiGroups: ["monitoring.coreos.com"]
  resources: ["prometheusrules", "servicemonitors"]
  verbs: ["get", "list", "watch"]
```

The built-in aggregation labels are:
- `rbac.authorization.k8s.io/aggregate-to-admin: "true"`
- `rbac.authorization.k8s.io/aggregate-to-edit: "true"`
- `rbac.authorization.k8s.io/aggregate-to-view: "true"`

When you install a CRD, you should create aggregated ClusterRoles so the default
roles automatically include permissions for your custom resources.

The aggregating ClusterRole uses `aggregationRule`:

```yaml
# This is how the built-in "view" ClusterRole works
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: view
aggregationRule:
  clusterRoleSelectors:
  - matchLabels:
      rbac.authorization.k8s.io/aggregate-to-view: "true"
rules: [] # Rules are automatically filled by the controller
```

### Hands-on: Creating Roles for Common Scenarios

```bash
# Scenario 1: Read-only access to pods and services in "staging"
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: staging
  name: read-only
rules:
- apiGroups: [""]
  resources: ["pods", "services", "endpoints", "configmaps"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch"]
EOF

# Scenario 2: Developer role — manage apps but not secrets
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: staging
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["pods/log", "pods/portforward"]
  verbs: ["get", "create"]
# Note: no access to secrets!
EOF

# Scenario 3: Secret reader — only specific secrets
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: staging
  name: specific-secret-reader
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  resourceNames: ["app-config", "tls-cert"]
# Can only read these two specific secrets, not list all secrets
EOF

# Verify the roles
kubectl get roles -n staging
kubectl describe role developer -n staging
```

---

## 27.4 RoleBinding and ClusterRoleBinding

### RoleBinding — Namespace-Scoped Grant

A `RoleBinding` grants the permissions defined in a Role to one or more subjects
within a specific namespace:

```yaml
# rolebinding-dev-pods.yaml
# Grants "pod-reader" role to user "jane" in the "development" namespace.
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: development
subjects:
  # Subject 1: a specific user
- kind: User
  name: "jane@example.com"
  apiGroup: rbac.authorization.k8s.io
  # Subject 2: a group (all members get the role)
- kind: Group
  name: "backend-team"
  apiGroup: rbac.authorization.k8s.io
  # Subject 3: a ServiceAccount in the same namespace
- kind: ServiceAccount
  name: "monitoring-agent"
  namespace: development
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```

Key rules for `roleRef`:
- `roleRef` is **immutable** — you cannot change it after creation (delete and recreate)
- `roleRef.kind` must be `Role` or `ClusterRole`
- `roleRef.name` is the name of the Role or ClusterRole

### ClusterRoleBinding — Cluster-Wide Grant

A `ClusterRoleBinding` grants permissions across the entire cluster:

```yaml
# clusterrolebinding-node-readers.yaml
# Grants "node-reader" ClusterRole to the "infrastructure" group cluster-wide.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: read-nodes
subjects:
- kind: Group
  name: "infrastructure"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: node-reader
  apiGroup: rbac.authorization.k8s.io
```

### The Power Move: ClusterRole + RoleBinding

This is one of the most important patterns in RBAC. You can define a ClusterRole
once and then grant it in specific namespaces using RoleBindings:

```yaml
# Step 1: Define permissions ONCE as a ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: app-developer
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
# Step 2: Grant it in "team-a" namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-developers
  namespace: team-a
subjects:
- kind: Group
  name: "team-a-devs"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole       # <-- Note: ClusterRole, not Role
  name: app-developer
  apiGroup: rbac.authorization.k8s.io
---
# Step 3: Grant the SAME ClusterRole in "team-b" namespace
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-b-developers
  namespace: team-b
subjects:
- kind: Group
  name: "team-b-devs"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: app-developer
  apiGroup: rbac.authorization.k8s.io
```

This pattern gives you:
- **DRY** — define the role once, bind it many times
- **Namespace isolation** — team-a-devs can't touch team-b resources
- **Easy updates** — change the ClusterRole, all bindings get updated

```
                   ClusterRole: app-developer
                   ┌───────────────────────┐
                   │ pods: CRUD            │
                   │ deployments: CRUD     │
                   │ services: CRUD        │
                   └───────┬───────────────┘
                           │
            ┌──────────────┼──────────────┐
            │              │              │
    RoleBinding       RoleBinding     RoleBinding
    ns: team-a        ns: team-b      ns: team-c
    ┌──────────┐     ┌──────────┐    ┌──────────┐
    │ team-a   │     │ team-b   │    │ team-c   │
    │ devs     │     │ devs     │    │ devs     │
    └──────────┘     └──────────┘    └──────────┘
```

### Hands-on: Complete Binding Examples

```bash
# Bind a user to the built-in "view" ClusterRole in a namespace
kubectl create rolebinding jane-view \
  --clusterrole=view \
  --user=jane@example.com \
  --namespace=production

# Bind a group to the built-in "edit" ClusterRole in a namespace
kubectl create rolebinding devs-edit \
  --clusterrole=edit \
  --group=developers \
  --namespace=development

# Bind a ServiceAccount to a Role
kubectl create rolebinding sa-pod-reader \
  --role=pod-reader \
  --serviceaccount=development:monitoring-agent \
  --namespace=development

# Bind cluster-wide
kubectl create clusterrolebinding global-readers \
  --clusterrole=view \
  --group=all-employees

# List all bindings in a namespace
kubectl get rolebindings -n development -o wide

# List all cluster bindings
kubectl get clusterrolebindings -o wide | head -20

# Describe a binding to see its details
kubectl describe rolebinding jane-view -n production
```

---

## 27.5 ServiceAccounts

### Default ServiceAccount

Every namespace automatically gets a `default` ServiceAccount:

```bash
kubectl get serviceaccounts -n default
# NAME      SECRETS   AGE
# default   0         30d
```

Every Pod that doesn't specify a ServiceAccount uses the `default` one. The
`default` ServiceAccount typically has **no** RBAC permissions — it can't do
anything useful through the API.

### Creating ServiceAccounts

```bash
# Create a ServiceAccount
kubectl create serviceaccount deploy-bot -n ci-cd

# Or with YAML
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: deploy-bot
  namespace: ci-cd
  labels:
    app: ci-cd-pipeline
  annotations:
    description: "Used by CI/CD pipeline to deploy applications"
EOF
```

### Token Projection

Kubernetes automatically projects a ServiceAccount token into Pods. Since
Kubernetes 1.22, this uses **bound service account tokens** — short-lived,
audience-scoped JWTs:

```yaml
# Pod with explicit ServiceAccount
apiVersion: v1
kind: Pod
metadata:
  name: deploy-job
  namespace: ci-cd
spec:
  serviceAccountName: deploy-bot    # Use this ServiceAccount
  containers:
  - name: deployer
    image: bitnami/kubectl:latest
    command: ["kubectl", "get", "pods", "-n", "target-namespace"]
```

The token is automatically mounted at `/var/run/secrets/kubernetes.io/serviceaccount/token`.

### Disabling Token Mounting

For Pods that don't need API access, disable auto-mounting to reduce the attack
surface:

```yaml
# On the ServiceAccount (affects all Pods using it)
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-api-access
  namespace: default
automountServiceAccountToken: false
---
# Or on the Pod (overrides ServiceAccount setting)
apiVersion: v1
kind: Pod
metadata:
  name: web-app
spec:
  serviceAccountName: no-api-access
  automountServiceAccountToken: false   # Belt AND suspenders
  containers:
  - name: app
    image: nginx:1.27
```

> **Best practice**: Set `automountServiceAccountToken: false` on the `default`
> ServiceAccount in every namespace. Then explicitly enable it only on Pods
> that need API access.

```bash
# Patch the default SA in a namespace
kubectl patch serviceaccount default -n production \
  -p '{"automountServiceAccountToken": false}'
```

### Hands-on: ServiceAccount with Specific Permissions

Let's create a complete ServiceAccount setup for a monitoring agent:

```bash
# 1. Create the namespace
kubectl create namespace monitoring

# 2. Create the ServiceAccount
kubectl create serviceaccount prometheus-scraper -n monitoring

# 3. Create a ClusterRole for reading metrics
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: metrics-reader
rules:
- apiGroups: [""]
  resources: ["pods", "services", "endpoints"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["nodes", "nodes/metrics", "nodes/proxy"]
  verbs: ["get", "list"]
- nonResourceURLs: ["/metrics", "/metrics/*"]
  verbs: ["get"]
EOF

# 4. Bind the ClusterRole to the ServiceAccount (cluster-wide)
kubectl create clusterrolebinding prometheus-scraper-metrics \
  --clusterrole=metrics-reader \
  --serviceaccount=monitoring:prometheus-scraper

# 5. Verify
kubectl auth can-i get pods --as=system:serviceaccount:monitoring:prometheus-scraper
# yes
kubectl auth can-i delete pods --as=system:serviceaccount:monitoring:prometheus-scraper
# no
kubectl auth can-i get nodes --as=system:serviceaccount:monitoring:prometheus-scraper
# yes

# 6. Create a Pod using this ServiceAccount
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: scraper
  namespace: monitoring
spec:
  serviceAccountName: prometheus-scraper
  containers:
  - name: scraper
    image: curlimages/curl:8.5.0
    command: ["sh", "-c", "while true; do curl -s https://kubernetes.default.svc/api/v1/pods --header \"Authorization: Bearer $(cat /var/run/secrets/kubernetes.io/serviceaccount/token)\" --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt | head -c 200; sleep 60; done"]
EOF
```

---

## 27.6 Practical RBAC Examples

### Example 1: Read-Only Access to a Namespace

The most common RBAC request — someone needs to see what's running but not
change anything:

```yaml
# rbac-readonly.yaml
# Complete read-only access to the "production" namespace.
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: production-viewers
  namespace: production
subjects:
- kind: Group
  name: "all-engineers"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: view              # Built-in ClusterRole — read-only access
  apiGroup: rbac.authorization.k8s.io
```

The built-in `view` ClusterRole allows get/list/watch on most resources but
excludes secrets, roles, and rolebindings.

### Example 2: Developer Access

Developers need to create and update applications but shouldn't manage secrets
or RBAC:

```yaml
# rbac-developer.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer
rules:
  # Core workload resources
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # Applications
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # Batch jobs
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # Networking
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # Debugging
- apiGroups: [""]
  resources: ["pods/log", "pods/portforward", "pods/exec"]
  verbs: ["get", "create"]
  # Events (read-only)
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "list", "watch"]
  # HPA
- apiGroups: ["autoscaling"]
  resources: ["horizontalpodautoscalers"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-binding
  namespace: development
subjects:
- kind: Group
  name: "backend-developers"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

### Example 3: CI/CD Pipeline Access

A CI/CD system needs to deploy to specific namespaces. It needs more power than
a developer but should be scoped tightly:

```yaml
# rbac-cicd.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: ci-deployer
rules:
  # Deploy workloads
- apiGroups: ["apps"]
  resources: ["deployments", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
  # Manage services and configmaps
- apiGroups: [""]
  resources: ["services", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
  # Manage secrets (needed for TLS certs, image pull secrets)
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
  # Check rollout status
- apiGroups: ["apps"]
  resources: ["deployments/status"]
  verbs: ["get"]
  # Manage HPA
- apiGroups: ["autoscaling"]
  resources: ["horizontalpodautoscalers"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
  # Manage ingress
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch", "create", "update", "patch"]
---
# Grant to staging
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer-staging
  namespace: staging
subjects:
- kind: ServiceAccount
  name: github-actions
  namespace: ci-cd
roleRef:
  kind: ClusterRole
  name: ci-deployer
  apiGroup: rbac.authorization.k8s.io
---
# Grant to production (same role, different namespace)
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: ci-deployer-production
  namespace: production
subjects:
- kind: ServiceAccount
  name: github-actions
  namespace: ci-cd
roleRef:
  kind: ClusterRole
  name: ci-deployer
  apiGroup: rbac.authorization.k8s.io
```

### Example 4: Monitoring Access

Monitoring systems need read access across all namespaces:

```yaml
# rbac-monitoring.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: monitoring-reader
rules:
  # Read all workloads across all namespaces
- apiGroups: [""]
  resources: ["pods", "services", "endpoints", "nodes", "namespaces"]
  verbs: ["get", "list", "watch"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets", "daemonsets"]
  verbs: ["get", "list", "watch"]
  # Read events for alerting
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "list", "watch"]
  # Read metrics
- apiGroups: ["metrics.k8s.io"]
  resources: ["pods", "nodes"]
  verbs: ["get", "list"]
  # Access non-resource URLs for metrics endpoints
- nonResourceURLs: ["/metrics", "/healthz", "/readyz"]
  verbs: ["get"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: monitoring-reader-binding
subjects:
- kind: ServiceAccount
  name: monitoring-sa
  namespace: monitoring
roleRef:
  kind: ClusterRole
  name: monitoring-reader
  apiGroup: rbac.authorization.k8s.io
```

### Example 5: Namespace Admin

Someone who can manage everything within a namespace but nothing outside it:

```yaml
# rbac-namespace-admin.yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: team-a-admin
  namespace: team-a
subjects:
- kind: User
  name: "alice@example.com"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: admin             # Built-in ClusterRole — full access within namespace
  apiGroup: rbac.authorization.k8s.io
```

The built-in `admin` ClusterRole grants full access to most namespaced resources
including the ability to create Roles and RoleBindings within the namespace —
effectively delegating RBAC management.

### Example 6: Hands-on — Full Multi-Team Cluster Setup

```bash
#!/bin/bash
# full-rbac-setup.sh — Sets up RBAC for a multi-team cluster.

set -euo pipefail

TEAMS=("team-alpha" "team-beta" "team-gamma")

# Step 1: Create namespaces for each team
for team in "${TEAMS[@]}"; do
  kubectl create namespace "$team" --dry-run=client -o yaml | kubectl apply -f -
  echo "Created namespace: $team"
done

# Step 2: Create shared ClusterRoles
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: team-developer
rules:
- apiGroups: [""]
  resources: ["pods", "services", "configmaps", "persistentvolumeclaims"]
  verbs: ["*"]
- apiGroups: ["apps"]
  resources: ["deployments", "replicasets", "statefulsets"]
  verbs: ["*"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["*"]
- apiGroups: [""]
  resources: ["pods/log", "pods/portforward"]
  verbs: ["get", "create"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: team-viewer
rules:
- apiGroups: ["", "apps", "batch", "networking.k8s.io"]
  resources: ["*"]
  verbs: ["get", "list", "watch"]
EOF

# Step 3: Bind roles per team
for team in "${TEAMS[@]}"; do
  # Team developers get full development access in their namespace
  kubectl create rolebinding "${team}-developers" \
    --clusterrole=team-developer \
    --group="${team}-devs" \
    --namespace="$team" \
    --dry-run=client -o yaml | kubectl apply -f -

  # Team leads get admin access in their namespace
  kubectl create rolebinding "${team}-admins" \
    --clusterrole=admin \
    --group="${team}-leads" \
    --namespace="$team" \
    --dry-run=client -o yaml | kubectl apply -f -

  # All engineers get view access to all team namespaces
  kubectl create rolebinding "${team}-all-viewers" \
    --clusterrole=team-viewer \
    --group="all-engineers" \
    --namespace="$team" \
    --dry-run=client -o yaml | kubectl apply -f -

  echo "RBAC configured for: $team"
done

# Step 4: Patch default SA in all namespaces
for team in "${TEAMS[@]}"; do
  kubectl patch serviceaccount default -n "$team" \
    -p '{"automountServiceAccountToken": false}'
done

echo "Multi-team RBAC setup complete!"
```

---

## 27.7 Least Privilege Patterns

### The Golden Rule: Start with Nothing

```
Default permission level: ZERO

Add permissions only when someone can demonstrate:
1. What they need access to
2. Why they need it
3. For how long
```

### Pattern 1: No Wildcards

```yaml
# BAD — gives access to everything
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]

# GOOD — explicit about what's needed
rules:
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch", "update", "patch"]
```

### Pattern 2: Avoid cluster-admin Except for Emergencies

The `cluster-admin` ClusterRole has `*/*` permissions. It should be bound to:
- A break-glass emergency account (stored securely, audited)
- The initial cluster bootstrap user
- **Nobody else**

```bash
# Audit who has cluster-admin
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.roleRef.name=="cluster-admin") | {name: .metadata.name, subjects: .subjects}'
```

### Pattern 3: Prefer Namespace-Scoped Roles

```
ClusterRoleBinding → ClusterRole    # Most dangerous: cluster-wide power
RoleBinding → ClusterRole           # Moderate: reusable but namespace-scoped
RoleBinding → Role                  # Safest: explicitly namespace-scoped
```

### Pattern 4: Restrict Secret Access

Secrets deserve special attention:

```yaml
# Allow reading only specific, named secrets
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  resourceNames: ["app-config", "db-password"]
# Do NOT use:
#   verbs: ["list"]     — can list ALL secret names
#   verbs: ["watch"]    — can watch ALL secret changes
#   resourceNames: []   — means ALL secrets
```

### Pattern 5: Audit Existing Permissions

```bash
# List ALL ClusterRoleBindings and their subjects
kubectl get clusterrolebindings -o custom-columns=\
NAME:.metadata.name,\
ROLE:.roleRef.name,\
SUBJECTS:.subjects[*].name

# Find all bindings for a specific user
kubectl get rolebindings,clusterrolebindings --all-namespaces -o json | \
  jq '.items[] | select(.subjects[]? | .name == "jane@example.com") | {kind: .kind, name: .metadata.name, namespace: .metadata.namespace}'

# Find all bindings for a ServiceAccount
kubectl get rolebindings,clusterrolebindings --all-namespaces -o json | \
  jq '.items[] | select(.subjects[]? | .kind == "ServiceAccount" and .name == "deploy-bot") | {kind: .kind, name: .metadata.name, namespace: .metadata.namespace}'
```

---

## 27.8 Debugging RBAC

### kubectl auth can-i

The most important debugging tool. Test whether a subject can perform an action:

```bash
# Can I do this? (as current user)
kubectl auth can-i create deployments -n production
# yes

# Can I do this? (as a different user)
kubectl auth can-i create deployments -n production --as=jane@example.com
# no

# Can a ServiceAccount do this?
kubectl auth can-i get pods \
  --as=system:serviceaccount:ci-cd:github-actions \
  -n staging
# yes

# Can a ServiceAccount delete secrets?
kubectl auth can-i delete secrets \
  --as=system:serviceaccount:ci-cd:github-actions \
  -n production
# no

# List ALL permissions for current user in a namespace
kubectl auth can-i --list -n production
# Resources                     Non-Resource URLs   Resource Names   Verbs
# pods                          []                  []               [get list watch]
# deployments.apps              []                  []               [get list watch create update patch delete]
# ...

# List ALL permissions for a ServiceAccount
kubectl auth can-i --list \
  --as=system:serviceaccount:monitoring:prometheus-scraper \
  -n default
```

### kubectl auth whoami

Available since Kubernetes 1.27:

```bash
kubectl auth whoami
# ATTRIBUTE   VALUE
# Username    jane@example.com
# Groups      [developers system:authenticated]
```

### Debugging Permission Denied Errors

When you see `Error from server (Forbidden)`, follow this process:

```bash
# Step 1: Identify the subject
kubectl auth whoami  # or check the ServiceAccount of the Pod

# Step 2: Test the specific permission
kubectl auth can-i <verb> <resource> -n <namespace> --as=<subject>

# Step 3: Check what bindings exist for the subject
kubectl get rolebindings -n <namespace> -o json | \
  jq '.items[] | select(.subjects[]? | .name == "<subject-name>")'

kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.subjects[]? | .name == "<subject-name>")'

# Step 4: Check what rules the bound role has
kubectl describe role <role-name> -n <namespace>
kubectl describe clusterrole <clusterrole-name>

# Step 5: Enable audit logging to see the exact request being denied
# In the API server audit policy:
```

### Audit Logging

The API server can log all authorization decisions. Configure an audit policy:

```yaml
# audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all authorization failures at the RequestResponse level
- level: RequestResponse
  omitStages:
  - RequestReceived
  resources:
  - group: ""
    resources: ["pods", "services", "secrets"]
  - group: "apps"
    resources: ["deployments"]
  # Log RBAC changes (someone modifying roles/bindings)
- level: RequestResponse
  resources:
  - group: "rbac.authorization.k8s.io"
    resources: ["roles", "rolebindings", "clusterroles", "clusterrolebindings"]
  verbs: ["create", "update", "patch", "delete"]
```

### Hands-on: Debugging a Permission Denied Error

```bash
# Scenario: A Pod can't list services. Let's debug it.

# 1. The Pod is running as ServiceAccount "web-app" in namespace "frontend"
SA="system:serviceaccount:frontend:web-app"

# 2. Test the permission
kubectl auth can-i list services -n frontend --as=$SA
# no

# 3. Check bindings in the namespace
kubectl get rolebindings -n frontend
# NAME            ROLE               AGE
# web-app-role    Role/web-app-role  5d

# 4. Check the role
kubectl describe role web-app-role -n frontend
# PolicyRule:
#   Resources  Verbs
#   ---------  -----
#   pods       [get list watch]
# --> Missing "services" resource!

# 5. Fix it — add services to the role
kubectl edit role web-app-role -n frontend
# Add:
# - apiGroups: [""]
#   resources: ["services"]
#   verbs: ["get", "list", "watch"]

# 6. Verify
kubectl auth can-i list services -n frontend --as=$SA
# yes
```

---

## 27.9 RBAC for Custom Resources

When you create a CRD, Kubernetes automatically creates API endpoints for it.
You need to create RBAC rules that reference the CRD's API group.

### Example: CRD for a Custom Resource

```yaml
# crd-database.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: databases.mycompany.io
spec:
  group: mycompany.io
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
            properties:
              engine:
                type: string
              version:
                type: string
              replicas:
                type: integer
  scope: Namespaced
  names:
    plural: databases
    singular: database
    kind: Database
    shortNames:
    - db
```

### RBAC for the Custom Resource

```yaml
# rbac-database-operator.yaml
# The operator needs full access to manage Database resources
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: database-operator
rules:
  # Full access to the custom resource
- apiGroups: ["mycompany.io"]
  resources: ["databases"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # Access to the status subresource
- apiGroups: ["mycompany.io"]
  resources: ["databases/status"]
  verbs: ["get", "update", "patch"]
  # The operator also needs to manage the underlying resources
- apiGroups: ["apps"]
  resources: ["statefulsets"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["services", "secrets", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
# Users only need limited access
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: database-user
  labels:
    # Aggregate into the built-in "edit" role
    rbac.authorization.k8s.io/aggregate-to-edit: "true"
rules:
- apiGroups: ["mycompany.io"]
  resources: ["databases"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
# View-only access aggregates into "view"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: database-viewer
  labels:
    rbac.authorization.k8s.io/aggregate-to-view: "true"
rules:
- apiGroups: ["mycompany.io"]
  resources: ["databases"]
  verbs: ["get", "list", "watch"]
```

### Hands-on: Complete CRD RBAC

```bash
# Apply the CRD
kubectl apply -f crd-database.yaml

# Apply the RBAC
kubectl apply -f rbac-database-operator.yaml

# Verify — user with "edit" role should now be able to manage databases
kubectl auth can-i create databases.mycompany.io -n default \
  --as=jane@example.com
# (depends on jane having "edit" in this namespace)

# Create a Database resource
cat <<'EOF' | kubectl apply -f -
apiVersion: mycompany.io/v1
kind: Database
metadata:
  name: my-postgres
  namespace: default
spec:
  engine: postgres
  version: "16"
  replicas: 3
EOF

kubectl get databases
```

---

## 27.10 Go: RBAC Management with client-go

### Creating Roles and Bindings Programmatically

```go
// Package rbacmanager provides utilities for creating and managing RBAC
// resources programmatically using client-go. This is the foundation for
// any operator or controller that needs to set up permissions for the
// resources it manages.
//
// In production Kubernetes environments, RBAC is rarely static YAML — it
// is dynamically created by operators, CI/CD systems, and platform tools.
// This package demonstrates the patterns used in real-world RBAC automation.
package rbacmanager

import (
	"context"
	"fmt"
	"log"
	"path/filepath"
	"time"

	rbacv1 "k8s.io/api/rbac/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
)

// RBACManager encapsulates the Kubernetes client and provides high-level
// methods for managing RBAC resources. It handles the common patterns of
// creating roles, bindings, and verifying permissions — abstracting away
// the verbose client-go API.
type RBACManager struct {
	// clientset is the Kubernetes API client. All RBAC operations
	// go through the RbacV1() sub-client.
	clientset *kubernetes.Clientset
}

// NewRBACManager creates a new RBACManager by loading the kubeconfig from
// the user's home directory (~/.kube/config). In production, you would
// typically use in-cluster configuration via rest.InClusterConfig().
//
// Returns an error if the kubeconfig cannot be loaded or the Kubernetes
// client cannot be created.
func NewRBACManager() (*RBACManager, error) {
	// Build the kubeconfig path. In a real operator, you would use
	// rest.InClusterConfig() instead.
	kubeconfig := filepath.Join(homedir.HomeDir(), ".kube", "config")

	// Build a *rest.Config from the kubeconfig file.
	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		return nil, fmt.Errorf("building kubeconfig: %w", err)
	}

	// Create the typed Kubernetes clientset. This gives us access to
	// all Kubernetes API groups including RBAC.
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		return nil, fmt.Errorf("creating clientset: %w", err)
	}

	return &RBACManager{clientset: clientset}, nil
}

// CreateNamespacedRole creates a Role in the specified namespace with
// the given rules. A Role is namespace-scoped — it only grants permissions
// within the namespace where it is created.
//
// Parameters:
//   - ctx: Context for cancellation and timeout control.
//   - namespace: The namespace in which to create the Role.
//   - name: The name of the Role. Must be unique within the namespace.
//   - rules: The set of PolicyRules that define what this Role allows.
//
// Returns the created Role object or an error if creation fails (e.g.,
	// the Role already exists, or the caller lacks permission to create Roles).
func (m *RBACManager) CreateNamespacedRole(
	ctx context.Context,
	namespace, name string,
	rules []rbacv1.PolicyRule,
) (*rbacv1.Role, error) {
	// Construct the Role object. The ObjectMeta contains the name and
	// namespace. The Rules field contains the actual permissions.
	role := &rbacv1.Role{
		ObjectMeta: metav1.ObjectMeta{
			Name:      name,
			Namespace: namespace,
			Labels: map[string]string{
				// Label for easy identification of programmatically
				// created roles.
				"managed-by": "rbac-manager",
			},
		},
		Rules: rules,
	}

	// Call the Kubernetes API to create the Role. The RbacV1() sub-client
	// provides typed access to all RBAC resources.
	created, err := m.clientset.RbacV1().Roles(namespace).Create(
		ctx, role, metav1.CreateOptions{},
	)
	if err != nil {
		return nil, fmt.Errorf("creating role %s/%s: %w", namespace, name, err)
	}

	log.Printf("Created Role %s/%s with %d rules", namespace, name, len(rules))
	return created, nil
}

// CreateClusterRole creates a cluster-scoped ClusterRole with the given
// rules. ClusterRoles can define permissions for:
//   - Cluster-scoped resources (nodes, namespaces, PVs)
//   - Namespaced resources across all namespaces (when used with ClusterRoleBinding)
//   - Non-resource URLs (/healthz, /metrics)
//
// Parameters:
//   - ctx: Context for cancellation and timeout control.
//   - name: The name of the ClusterRole. Must be unique cluster-wide.
//   - rules: The set of PolicyRules that define what this ClusterRole allows.
//
// Returns the created ClusterRole or an error.
func (m *RBACManager) CreateClusterRole(
	ctx context.Context,
	name string,
	rules []rbacv1.PolicyRule,
) (*rbacv1.ClusterRole, error) {
	// Construct the ClusterRole. Note: no namespace in ObjectMeta
	// because ClusterRoles are cluster-scoped.
	clusterRole := &rbacv1.ClusterRole{
		ObjectMeta: metav1.ObjectMeta{
			Name: name,
			Labels: map[string]string{
				"managed-by": "rbac-manager",
			},
		},
		Rules: rules,
	}

	created, err := m.clientset.RbacV1().ClusterRoles().Create(
		ctx, clusterRole, metav1.CreateOptions{},
	)
	if err != nil {
		return nil, fmt.Errorf("creating clusterrole %s: %w", name, err)
	}

	log.Printf("Created ClusterRole %s with %d rules", name, len(rules))
	return created, nil
}

// CreateRoleBinding creates a RoleBinding that connects subjects to a Role
// or ClusterRole within a specific namespace. This is how you actually grant
// permissions — a Role by itself does nothing until it is bound to subjects.
//
// Parameters:
//   - ctx: Context for cancellation and timeout control.
//   - namespace: The namespace in which to create the binding.
//   - name: The name of the RoleBinding.
//   - roleRef: Reference to the Role or ClusterRole to bind.
//   - subjects: The list of subjects (Users, Groups, ServiceAccounts) to bind.
//
// Returns the created RoleBinding or an error.
func (m *RBACManager) CreateRoleBinding(
	ctx context.Context,
	namespace, name string,
	roleRef rbacv1.RoleRef,
	subjects []rbacv1.Subject,
) (*rbacv1.RoleBinding, error) {
	binding := &rbacv1.RoleBinding{
		ObjectMeta: metav1.ObjectMeta{
			Name:      name,
			Namespace: namespace,
			Labels: map[string]string{
				"managed-by": "rbac-manager",
			},
		},
		// RoleRef is immutable after creation. If you need to change it,
		// delete and recreate the RoleBinding.
		RoleRef:  roleRef,
		Subjects: subjects,
	}

	created, err := m.clientset.RbacV1().RoleBindings(namespace).Create(
		ctx, binding, metav1.CreateOptions{},
	)
	if err != nil {
		return nil, fmt.Errorf("creating rolebinding %s/%s: %w", namespace, name, err)
	}

	log.Printf("Created RoleBinding %s/%s → %s/%s for %d subjects",
		namespace, name, roleRef.Kind, roleRef.Name, len(subjects))
	return created, nil
}

// SetupApplicationRBAC creates a complete RBAC setup for an application.
// This is the pattern used by operators and platform tools: create a
// ServiceAccount, a Role with the minimum necessary permissions, and
// a RoleBinding connecting them.
//
// Parameters:
//   - ctx: Context for cancellation and timeout control.
//   - namespace: The namespace where the application lives.
//   - appName: The name of the application (used to derive resource names).
//
// This function creates:
//   1. A Role named "<appName>-role" with permissions to manage Pods,
//      ConfigMaps, and Services.
//   2. A RoleBinding named "<appName>-binding" connecting the
//      ServiceAccount "<appName>-sa" to the Role.
//
// The ServiceAccount itself should be created separately (usually by
	// the application's Deployment manifest).
func (m *RBACManager) SetupApplicationRBAC(
	ctx context.Context,
	namespace, appName string,
) error {
	// Define the minimum permissions this application needs.
	// In a real operator, these would be determined by the application's
	// requirements — e.g., a database operator needs StatefulSet access.
	rules := []rbacv1.PolicyRule{
		{
			// Core API group — pods, services, configmaps
			APIGroups: []string{""},
			Resources: []string{"pods", "services", "configmaps"},
			Verbs:     []string{"get", "list", "watch"},
		},
		{
			// Apps API group — deployments (read-only)
			APIGroups: []string{"apps"},
			Resources: []string{"deployments"},
			Verbs:     []string{"get", "list", "watch"},
		},
		{
			// Events — for the application to read its own events
			APIGroups: []string{""},
			Resources: []string{"events"},
			Verbs:     []string{"get", "list", "watch"},
		},
	}

	// Step 1: Create the Role.
	roleName := appName + "-role"
	_, err := m.CreateNamespacedRole(ctx, namespace, roleName, rules)
	if err != nil {
		return fmt.Errorf("creating role for %s: %w", appName, err)
	}

	// Step 2: Create the RoleBinding.
	// Connect the application's ServiceAccount to the Role.
	bindingName := appName + "-binding"
	roleRef := rbacv1.RoleRef{
		APIGroup: "rbac.authorization.k8s.io",
		Kind:     "Role",
		Name:     roleName,
	}
	subjects := []rbacv1.Subject{
		{
			Kind:      "ServiceAccount",
			Name:      appName + "-sa",
			Namespace: namespace,
		},
	}

	_, err = m.CreateRoleBinding(ctx, namespace, bindingName, roleRef, subjects)
	if err != nil {
		return fmt.Errorf("creating rolebinding for %s: %w", appName, err)
	}

	log.Printf("RBAC setup complete for application %s in namespace %s",
		appName, namespace)
	return nil
}
```

### Checking Permissions via SubjectAccessReview

```go
// Package rbacauditor provides tools for checking and auditing RBAC
// permissions in a Kubernetes cluster. The SubjectAccessReview API
// allows you to programmatically test whether a subject can perform
// a specific action — the equivalent of "kubectl auth can-i" but
// from Go code.
//
// This is essential for:
//   - Pre-flight checks before deploying applications
//   - Security audits of existing permissions
//   - Operators that need to verify their own permissions
//   - Admission webhooks that check caller permissions
package rbacauditor

import (
	"context"
	"fmt"
	"log"
	"path/filepath"
	"strings"

	authorizationv1 "k8s.io/api/authorization/v1"
	rbacv1 "k8s.io/api/rbac/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
)

// PermissionCheck represents a single permission test — "can this subject
// perform this verb on this resource in this namespace?"
type PermissionCheck struct {
	// Namespace is the namespace to check the permission in.
	// Empty string means cluster-scoped resources.
	Namespace string

	// Verb is the API verb to check (get, list, create, delete, etc.).
	Verb string

	// Resource is the resource type to check (pods, deployments, etc.).
	Resource string

	// APIGroup is the API group of the resource ("" for core, "apps",
		// "rbac.authorization.k8s.io", etc.).
	APIGroup string

	// ResourceName is an optional specific resource name. If empty,
	// checks permission on all resources of this type.
	ResourceName string
}

// PermissionResult holds the result of a single permission check,
// including whether it was allowed and any reason provided by the
// authorizer.
type PermissionResult struct {
	// Check is the original permission check that was tested.
	Check PermissionCheck

	// Allowed is true if the subject has permission to perform the action.
	Allowed bool

	// Reason is the human-readable reason from the authorizer explaining
	// why the request was allowed or denied.
	Reason string

	// EvaluationError is set if the authorization check itself failed
	// (as opposed to being denied).
	EvaluationError string
}

// RBACAuditor provides methods for checking RBAC permissions and auditing
// the cluster's authorization state. It wraps the SubjectAccessReview API
// and provides higher-level audit functions.
type RBACAuditor struct {
	// clientset provides access to the Kubernetes API, including the
	// AuthorizationV1 and RbacV1 sub-clients.
	clientset *kubernetes.Clientset
}

// NewRBACAuditor creates a new RBACAuditor from the default kubeconfig.
// The caller must have permission to create SubjectAccessReview resources
// (typically requires cluster-admin or a dedicated auditor role).
func NewRBACAuditor() (*RBACAuditor, error) {
	kubeconfig := filepath.Join(homedir.HomeDir(), ".kube", "config")

	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		return nil, fmt.Errorf("building kubeconfig: %w", err)
	}

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		return nil, fmt.Errorf("creating clientset: %w", err)
	}

	return &RBACAuditor{clientset: clientset}, nil
}

// CanI checks whether the specified subject (user or ServiceAccount) can
// perform the specified action. This is the programmatic equivalent of
// "kubectl auth can-i --as=<subject>".
//
// Parameters:
//   - ctx: Context for cancellation and timeout control.
//   - subject: The user or ServiceAccount to check. For ServiceAccounts,
//     use the format "system:serviceaccount:<namespace>:<name>".
//   - check: The permission to test.
//
// Returns a PermissionResult with the authorization decision. The Allowed
// field is true if the subject has permission. The Reason field explains
// why (e.g., which RBAC rule granted the permission).
//
// Note: This creates a SubjectAccessReview, which is a cluster-scoped
// resource. The caller must have permission to create SubjectAccessReviews.
func (a *RBACAuditor) CanI(
	ctx context.Context,
	subject string,
	check PermissionCheck,
) (*PermissionResult, error) {
	// Build the SubjectAccessReview request. This is the API object
	// that the authorization system evaluates.
	sar := &authorizationv1.SubjectAccessReview{
		Spec: authorizationv1.SubjectAccessReviewSpec{
			// The user or SA we're checking permissions for.
			User: subject,
			// The resource attributes describe what action is being checked.
			ResourceAttributes: &authorizationv1.ResourceAttributes{
				Namespace: check.Namespace,
				Verb:      check.Verb,
				Group:     check.APIGroup,
				Resource:  check.Resource,
				Name:      check.ResourceName,
			},
		},
	}

	// Submit the SubjectAccessReview to the API server. The authorizer
	// evaluates it and returns a decision in the Status field.
	result, err := a.clientset.AuthorizationV1().SubjectAccessReviews().Create(
		ctx, sar, metav1.CreateOptions{},
	)
	if err != nil {
		return nil, fmt.Errorf("creating SubjectAccessReview: %w", err)
	}

	return &PermissionResult{
		Check:           check,
		Allowed:         result.Status.Allowed,
		Reason:          result.Status.Reason,
		EvaluationError: result.Status.EvaluationError,
	}, nil
}

// AuditServiceAccount performs a comprehensive permission audit for a
// ServiceAccount. It tests a standard set of permissions across common
// resources and reports what the ServiceAccount can and cannot do.
//
// Parameters:
//   - ctx: Context for cancellation and timeout control.
//   - namespace: The namespace of the ServiceAccount.
//   - saName: The name of the ServiceAccount to audit.
//
// Returns a slice of PermissionResults — one for each check performed.
// The caller can filter for allowed/denied results to understand the
// ServiceAccount's effective permissions.
func (a *RBACAuditor) AuditServiceAccount(
	ctx context.Context,
	namespace, saName string,
) ([]PermissionResult, error) {
	// Build the fully qualified ServiceAccount user string.
	// Kubernetes represents ServiceAccounts as users with the format:
	// system:serviceaccount:<namespace>:<name>
	subject := fmt.Sprintf("system:serviceaccount:%s:%s", namespace, saName)

	// Define the standard checks — these cover the most common
	// resources and verbs that a ServiceAccount might need.
	checks := []PermissionCheck{
		// Core resources
		{Namespace: namespace, Verb: "get", Resource: "pods", APIGroup: ""},
		{Namespace: namespace, Verb: "list", Resource: "pods", APIGroup: ""},
		{Namespace: namespace, Verb: "create", Resource: "pods", APIGroup: ""},
		{Namespace: namespace, Verb: "delete", Resource: "pods", APIGroup: ""},
		{Namespace: namespace, Verb: "get", Resource: "secrets", APIGroup: ""},
		{Namespace: namespace, Verb: "list", Resource: "secrets", APIGroup: ""},
		{Namespace: namespace, Verb: "get", Resource: "configmaps", APIGroup: ""},
		{Namespace: namespace, Verb: "get", Resource: "services", APIGroup: ""},
		// Apps resources
		{Namespace: namespace, Verb: "get", Resource: "deployments", APIGroup: "apps"},
		{Namespace: namespace, Verb: "create", Resource: "deployments", APIGroup: "apps"},
		{Namespace: namespace, Verb: "update", Resource: "deployments", APIGroup: "apps"},
		{Namespace: namespace, Verb: "delete", Resource: "deployments", APIGroup: "apps"},
		// Dangerous permissions
		{Namespace: namespace, Verb: "create", Resource: "pods/exec", APIGroup: ""},
		{Namespace: namespace, Verb: "create", Resource: "pods/portforward", APIGroup: ""},
		// Cluster-scoped resources
		{Verb: "get", Resource: "nodes", APIGroup: ""},
		{Verb: "list", Resource: "namespaces", APIGroup: ""},
		// RBAC resources (should be tightly controlled)
		{Namespace: namespace, Verb: "create", Resource: "roles", APIGroup: "rbac.authorization.k8s.io"},
		{Namespace: namespace, Verb: "create", Resource: "rolebindings", APIGroup: "rbac.authorization.k8s.io"},
	}

	// Execute all permission checks.
	var results []PermissionResult
	for _, check := range checks {
		result, err := a.CanI(ctx, subject, check)
		if err != nil {
			return nil, fmt.Errorf("checking %s %s: %w", check.Verb, check.Resource, err)
		}
		results = append(results, *result)
	}

	return results, nil
}

// PrintAuditReport prints a formatted audit report to the log, showing
// which permissions are allowed and which are denied. Allowed permissions
// are marked with ✓ and denied with ✗. Dangerous permissions (secrets,
	// exec, RBAC) are flagged with warnings.
//
// Parameters:
//   - saName: The name of the ServiceAccount (for display purposes).
//   - results: The slice of PermissionResults from AuditServiceAccount.
func PrintAuditReport(saName string, results []PermissionResult) {
	log.Printf("=== RBAC Audit Report for ServiceAccount: %s ===", saName)
	log.Printf("%-10s %-8s %-30s %s", "STATUS", "VERB", "RESOURCE", "NOTES")
	log.Printf(strings.Repeat("-", 80))

	// dangerousResources is a set of resources that warrant extra
	// attention during security audits.
	dangerousResources := map[string]bool{
		"secrets":      true,
		"pods/exec":    true,
		"roles":        true,
		"rolebindings": true,
	}

	var allowedCount, deniedCount int
	for _, r := range results {
		status := "✗ DENIED"
		if r.Allowed {
			status = "✓ ALLOWED"
			allowedCount++
		} else {
			deniedCount++
		}

		// Flag dangerous permissions that are allowed.
		notes := ""
		if r.Allowed && dangerousResources[r.Check.Resource] {
			notes = "⚠ SECURITY: Review this permission"
		}

		resourceStr := r.Check.Resource
		if r.Check.Namespace != "" {
			resourceStr = r.Check.Namespace + "/" + r.Check.Resource
		}

		log.Printf("%-10s %-8s %-30s %s", status, r.Check.Verb, resourceStr, notes)
	}

	log.Printf(strings.Repeat("-", 80))
	log.Printf("Total: %d allowed, %d denied out of %d checks",
		allowedCount, deniedCount, len(results))
}

// FindOverprivilegedBindings scans all RoleBindings and ClusterRoleBindings
// in the cluster, looking for bindings that grant overly broad permissions.
// This includes:
//   - Bindings that use wildcard ("*") verbs or resources
//   - Bindings to cluster-admin
//   - Bindings to system:unauthenticated or system:anonymous
//
// Returns a slice of strings describing each finding.
func (a *RBACAuditor) FindOverprivilegedBindings(
	ctx context.Context,
) ([]string, error) {
	var findings []string

	// Check all ClusterRoleBindings for dangerous patterns.
	crbs, err := a.clientset.RbacV1().ClusterRoleBindings().List(
		ctx, metav1.ListOptions{},
	)
	if err != nil {
		return nil, fmt.Errorf("listing clusterrolebindings: %w", err)
	}

	for _, crb := range crbs.Items {
		// Flag cluster-admin bindings.
		if crb.RoleRef.Name == "cluster-admin" {
			for _, subject := range crb.Subjects {
				// Skip system components that legitimately need cluster-admin.
				if strings.HasPrefix(subject.Name, "system:") {
					continue
				}
				findings = append(findings,
					fmt.Sprintf("CRITICAL: ClusterRoleBinding %q grants cluster-admin to %s %q",
						crb.Name, subject.Kind, subject.Name))
			}
		}

		// Flag bindings to unauthenticated users.
		for _, subject := range crb.Subjects {
			if subject.Name == "system:unauthenticated" || subject.Name == "system:anonymous" {
				findings = append(findings,
					fmt.Sprintf("CRITICAL: ClusterRoleBinding %q grants %q to unauthenticated users",
						crb.Name, crb.RoleRef.Name))
			}
		}
	}

	// Check all ClusterRoles for wildcard permissions.
	crs, err := a.clientset.RbacV1().ClusterRoles().List(
		ctx, metav1.ListOptions{},
	)
	if err != nil {
		return nil, fmt.Errorf("listing clusterroles: %w", err)
	}

	for _, cr := range crs.Items {
		// Skip system ClusterRoles — they're expected to have broad permissions.
		if strings.HasPrefix(cr.Name, "system:") {
			continue
		}
		for _, rule := range cr.Rules {
			if containsWildcard(rule.Verbs) && containsWildcard(rule.Resources) {
				findings = append(findings,
					fmt.Sprintf("WARNING: ClusterRole %q has wildcard verbs AND resources: %v on %v",
						cr.Name, rule.Verbs, rule.APIGroups))
			}
		}
	}

	return findings, nil
}

// containsWildcard checks whether a string slice contains the wildcard
// entry "*". This is used to detect overly broad RBAC rules.
func containsWildcard(items []string) bool {
	for _, item := range items {
		if item == "*" {
			return true
		}
	}
	return false
}

// ListBindingsForSubject finds all RoleBindings and ClusterRoleBindings
// that grant permissions to the specified subject. This is useful for
// understanding the effective permissions of a user or ServiceAccount
// by finding every role they are bound to.
//
// Parameters:
//   - ctx: Context for cancellation and timeout control.
//   - subjectKind: "User", "Group", or "ServiceAccount".
//   - subjectName: The name of the subject.
//
// Returns a formatted list of binding descriptions.
func (a *RBACAuditor) ListBindingsForSubject(
	ctx context.Context,
	subjectKind, subjectName string,
) ([]string, error) {
	var bindings []string

	// Check namespace-scoped RoleBindings across all namespaces.
	rbs, err := a.clientset.RbacV1().RoleBindings("").List(
		ctx, metav1.ListOptions{},
	)
	if err != nil {
		return nil, fmt.Errorf("listing rolebindings: %w", err)
	}

	for _, rb := range rbs.Items {
		for _, subject := range rb.Subjects {
			if subject.Kind == subjectKind && subject.Name == subjectName {
				bindings = append(bindings,
					fmt.Sprintf("RoleBinding %s/%s → %s/%s",
						rb.Namespace, rb.Name, rb.RoleRef.Kind, rb.RoleRef.Name))
			}
		}
	}

	// Check ClusterRoleBindings.
	crbs, err := a.clientset.RbacV1().ClusterRoleBindings().List(
		ctx, metav1.ListOptions{},
	)
	if err != nil {
		return nil, fmt.Errorf("listing clusterrolebindings: %w", err)
	}

	for _, crb := range crbs.Items {
		for _, subject := range crb.Subjects {
			if subject.Kind == subjectKind && subject.Name == subjectName {
				bindings = append(bindings,
					fmt.Sprintf("ClusterRoleBinding %s → %s/%s",
						crb.Name, crb.RoleRef.Kind, crb.RoleRef.Name))
			}
		}
	}

	return bindings, nil
}
```

### Hands-on: Complete RBAC Setup Program

```go
// Command rbac-setup demonstrates a complete, production-ready RBAC setup
// workflow using client-go. It creates a namespace, ServiceAccount, Role,
// and RoleBinding for an application, then verifies the permissions are
// correct using SubjectAccessReview.
//
// This is the pattern used by real platform tools and operators — they
// don't just apply YAML, they verify that the RBAC setup actually works.
//
// Usage:
//
//	go run main.go
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"time"

	corev1 "k8s.io/api/core/v1"
	rbacv1 "k8s.io/api/rbac/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	// Load kubeconfig. Use KUBECONFIG env var if set, otherwise default path.
	kubeconfigPath := os.Getenv("KUBECONFIG")
	if kubeconfigPath == "" {
		home, _ := os.UserHomeDir()
		kubeconfigPath = home + "/.kube/config"
	}

	config, err := clientcmd.BuildConfigFromFlags("", kubeconfigPath)
	if err != nil {
		log.Fatalf("Failed to build kubeconfig: %v", err)
	}

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		log.Fatalf("Failed to create clientset: %v", err)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
	defer cancel()

	namespace := "demo-app"
	appName := "my-web-app"
	saName := appName + "-sa"

	// ── Step 1: Create the namespace ──────────────────────────────────
	log.Printf("Step 1: Creating namespace %q", namespace)
	ns := &corev1.Namespace{
		ObjectMeta: metav1.ObjectMeta{Name: namespace},
	}
	if _, err := clientset.CoreV1().Namespaces().Create(ctx, ns, metav1.CreateOptions{}); err != nil {
		log.Printf("Namespace may already exist: %v", err)
	}

	// ── Step 2: Create the ServiceAccount ─────────────────────────────
	log.Printf("Step 2: Creating ServiceAccount %q", saName)
	sa := &corev1.ServiceAccount{
		ObjectMeta: metav1.ObjectMeta{
			Name:      saName,
			Namespace: namespace,
		},
	}
	if _, err := clientset.CoreV1().ServiceAccounts(namespace).Create(ctx, sa, metav1.CreateOptions{}); err != nil {
		log.Printf("ServiceAccount may already exist: %v", err)
	}

	// ── Step 3: Create a Role with least-privilege permissions ────────
	log.Printf("Step 3: Creating Role for %q", appName)
	role := &rbacv1.Role{
		ObjectMeta: metav1.ObjectMeta{
			Name:      appName + "-role",
			Namespace: namespace,
		},
		Rules: []rbacv1.PolicyRule{
			{
				// The app needs to read its own ConfigMaps and Secrets.
				APIGroups: []string{""},
				Resources: []string{"configmaps"},
				Verbs:     []string{"get", "list", "watch"},
			},
			{
				// Read specific secrets only.
				APIGroups:     []string{""},
				Resources:     []string{"secrets"},
				Verbs:         []string{"get"},
				ResourceNames: []string{appName + "-tls", appName + "-config"},
			},
			{
				// The app creates events to report its status.
				APIGroups: []string{""},
				Resources: []string{"events"},
				Verbs:     []string{"create", "patch"},
			},
		},
	}
	if _, err := clientset.RbacV1().Roles(namespace).Create(ctx, role, metav1.CreateOptions{}); err != nil {
		log.Printf("Role may already exist: %v", err)
	}

	// ── Step 4: Create the RoleBinding ────────────────────────────────
	log.Printf("Step 4: Creating RoleBinding")
	binding := &rbacv1.RoleBinding{
		ObjectMeta: metav1.ObjectMeta{
			Name:      appName + "-binding",
			Namespace: namespace,
		},
		RoleRef: rbacv1.RoleRef{
			APIGroup: "rbac.authorization.k8s.io",
			Kind:     "Role",
			Name:     appName + "-role",
		},
		Subjects: []rbacv1.Subject{
			{
				Kind:      "ServiceAccount",
				Name:      saName,
				Namespace: namespace,
			},
		},
	}
	if _, err := clientset.RbacV1().RoleBindings(namespace).Create(ctx, binding, metav1.CreateOptions{}); err != nil {
		log.Printf("RoleBinding may already exist: %v", err)
	}

	// ── Step 5: Verify permissions ────────────────────────────────────
	log.Printf("Step 5: Verifying permissions via SubjectAccessReview")
	subject := fmt.Sprintf("system:serviceaccount:%s:%s", namespace, saName)

	// Define the expected permissions matrix.
	checks := []struct {
		verb     string
		resource string
		group    string
		expect   bool // true = should be allowed
	}{
		{"get", "configmaps", "", true},
		{"list", "configmaps", "", true},
		{"create", "configmaps", "", false},   // Should NOT be allowed
		{"get", "secrets", "", true},           // Allowed (specific names)
		{"list", "secrets", "", false},         // Should NOT be allowed
		{"create", "events", "", true},
		{"get", "pods", "", false},             // Not in our role
		{"create", "deployments", "apps", false}, // Not in our role
	}

	allPassed := true
	for _, c := range checks {
		result, err := clientset.AuthorizationV1().SubjectAccessReviews().Create(
			ctx,
			&authorizationv1.SubjectAccessReview{
				Spec: authorizationv1.SubjectAccessReviewSpec{
					User: subject,
					ResourceAttributes: &authorizationv1.ResourceAttributes{
						Namespace: namespace,
						Verb:      c.verb,
						Group:     c.group,
						Resource:  c.resource,
					},
				},
			},
			metav1.CreateOptions{},
		)
		if err != nil {
			log.Printf("  ERROR checking %s %s: %v", c.verb, c.resource, err)
			allPassed = false
			continue
		}

		status := "✓"
		if result.Status.Allowed != c.expect {
			status = "✗ MISMATCH"
			allPassed = false
		}
		log.Printf("  %s %s %s: allowed=%v (expected=%v)",
			status, c.verb, c.resource, result.Status.Allowed, c.expect)
	}

	if allPassed {
		log.Printf("All permission checks passed! RBAC setup is correct.")
	} else {
		log.Printf("Some permission checks failed. Review the RBAC configuration.")
		os.Exit(1)
	}
}
```

---

## 27.11 Common RBAC Mistakes

### Mistake 1: Wildcard Everything

```yaml
# THE WORST POSSIBLE ROLE — grants godmode
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"]
```

This is functionally identical to `cluster-admin`. Never do this. Be explicit
about every apiGroup, resource, and verb.

### Mistake 2: Using cluster-admin for CI/CD

```yaml
# DON'T DO THIS
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin  # Name tells you everything about the mistake
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: ci-cd
roleRef:
  kind: ClusterRole
  name: cluster-admin   # <-- NOPE
  apiGroup: rbac.authorization.k8s.io
```

Instead, create a specific ClusterRole with only the permissions your CI/CD
system needs (see Example 3 in section 27.6).

### Mistake 3: Not Restricting Secret Access

Secrets contain passwords, API keys, TLS certificates, and tokens. If your
role allows `list` on secrets, anyone with that role can enumerate ALL
secrets in the namespace:

```bash
# This lists ALL secret names (even without knowing them)
kubectl get secrets -n production

# Then read each one
kubectl get secret database-password -n production -o jsonpath='{.data.password}' | base64 -d
```

Use `resourceNames` to restrict to specific secrets:

```yaml
rules:
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]
  resourceNames: ["app-config"]  # Can ONLY read this specific secret
```

### Mistake 4: Privilege Escalation via Role Creation

If a subject can create Roles and RoleBindings, they can grant themselves
any permission:

```yaml
# Step 1: Create a new Role with secret access
# Step 2: Create a RoleBinding binding themselves to it
# Step 3: Read all the secrets
```

Kubernetes has built-in escalation prevention — you cannot create a Role
with permissions you don't already have, unless you have the `escalate`
verb on roles. Similarly, you cannot create a RoleBinding to a Role unless
you already have all the permissions in that Role, or you have the `bind`
verb on rolebindings.

```yaml
# This allows a user to bind any role (dangerous!)
rules:
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["rolebindings"]
  verbs: ["create"]
- apiGroups: ["rbac.authorization.k8s.io"]
  resources: ["roles"]
  verbs: ["bind"]     # <-- This bypasses escalation prevention!
```

> **Rule**: Never grant `bind` or `escalate` verbs unless you fully
> understand the implications. These verbs exist for platform tools,
> not end users.

### Mistake 5: Forgetting About Groups

Users often belong to groups they don't know about. A ClusterRoleBinding
to the group `system:authenticated` affects **every authenticated user**:

```yaml
# This gives EVERY authenticated user access to delete pods
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bad-idea
subjects:
- kind: Group
  name: "system:authenticated"
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: pod-manager
  apiGroup: rbac.authorization.k8s.io
```

Audit group-based bindings regularly:

```bash
kubectl get clusterrolebindings -o json | \
  jq '.items[] | select(.subjects[]? | .kind=="Group") | {name: .metadata.name, role: .roleRef.name, groups: [.subjects[] | select(.kind=="Group") | .name]}'
```

---

## 27.12 Summary

RBAC is the backbone of Kubernetes security. Here's what you should take away:

| Concept                      | Key Takeaway                                        |
|------------------------------|-----------------------------------------------------|
| **Subjects**                 | Users (strings), Groups (strings), ServiceAccounts  |
| **Role vs ClusterRole**      | Namespace-scoped vs cluster-scoped permissions      |
| **RoleBinding**              | Grants a role within a specific namespace            |
| **ClusterRoleBinding**       | Grants a ClusterRole across the entire cluster       |
| **ClusterRole + RoleBinding**| Define once, bind per-namespace — the best pattern   |
| **Least privilege**          | Start with zero, add explicitly, no wildcards        |
| **Debugging**                | `kubectl auth can-i` is your best friend             |
| **SubjectAccessReview**      | Programmatic permission checking via client-go       |
| **Aggregation**              | CRDs should aggregate into view/edit/admin           |
| **Escalation prevention**    | Can't grant what you don't have (unless bind/escalate)|

> **Interview tip**: When asked about RBAC, start with the triangle
> (Subject → Binding → Role), then explain the namespaced vs cluster-scoped
> distinction, then demonstrate with `kubectl auth can-i`. Mention least
> privilege, secret restriction, and escalation prevention to show depth.

---

*Next: Chapter 28 — Pod Security Standards & Admission, where we secure the
workloads themselves after controlling who can deploy them.*
