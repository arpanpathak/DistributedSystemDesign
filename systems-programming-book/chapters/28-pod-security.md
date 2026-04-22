# Chapter 28: Pod Security Standards & Admission

> *"A container is only as secure as its configuration. Default Kubernetes Pods
> run as root, can escalate privileges, and have write access to the entire
> filesystem. Every one of those defaults is a vulnerability."*

Chapter 27 controlled **who** can do **what** in the cluster. This chapter
controls **how workloads run** — ensuring that even a legitimate, authorized
deployment cannot introduce security holes through overly permissive Pod
configurations.

We'll trace the evolution from the removed PodSecurityPolicy to today's Pod
Security Standards (PSS) and Pod Security Admission (PSA). Then we'll dive
deep into SecurityContext, explore policy engines like OPA/Gatekeeper and
Kyverno, and end with a complete hands-on hardening exercise that takes a
Pod from insecure defaults to production-grade security.

---

## 28.1 The Evolution of Pod Security

### PodSecurityPolicy (PSP) — The Fallen Guardian

PodSecurityPolicy was Kubernetes' first attempt at controlling Pod security.
It was a cluster-scoped resource that defined a set of conditions a Pod had
to meet to be admitted.

```yaml
# HISTORICAL — PodSecurityPolicy (removed in Kubernetes 1.25)
# Shown for context only. Do NOT use this in new clusters.
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  runAsUser:
    rule: MustRunAsNonRoot
  fsGroup:
    rule: RunAsAny
  volumes:
  - 'configMap'
  - 'secret'
  - 'emptyDir'
  - 'persistentVolumeClaim'
```

**Why PSP was removed:**

1. **Confusing activation model** — PSPs were activated via RBAC. If a user or
   ServiceAccount had `use` permission on a PSP, it applied. This meant you
   could accidentally grant permissive policies through RBAC bindings that
   seemed unrelated to security.

2. **Ordering problems** — When multiple PSPs applied to a Pod, Kubernetes used
   a complex priority algorithm to pick one. This was unpredictable and led to
   subtle security holes.

3. **Mutation side effects** — PSPs could modify Pods (adding seccomp profiles,
   setting runAsUser). This made it hard to understand what was actually running.

4. **All-or-nothing adoption** — You had to enable the PSP admission controller
   cluster-wide. If you didn't have policies for every workload, system Pods
   would be rejected.

5. **No dry-run/audit mode** — You couldn't test PSP changes without risking
   production workloads.

**Timeline:**
- Kubernetes 1.21: PSP deprecated
- Kubernetes 1.22: Pod Security Admission introduced (alpha)
- Kubernetes 1.23: PSA promoted to beta (enabled by default)
- Kubernetes 1.25: PSP **removed entirely**

### Pod Security Standards (PSS) — The Replacement

Pod Security Standards are a **specification** — they define three security levels
that Pods can be evaluated against. PSS is not an implementation; it's a contract
that admission controllers enforce.

### Pod Security Admission (PSA) — The Built-In Enforcer

Pod Security Admission is the **built-in** admission controller that enforces PSS.
It's compiled into the API server — no webhooks, no external controllers, no
installation required.

```
                                 ┌─────────────────────┐
  kubectl apply ─→ API Server ─→ │ Pod Security         │ ─→ etcd
                                 │ Admission Controller │
                                 │                      │
                                 │ Checks Pod spec      │
                                 │ against PSS level    │
                                 │ configured on the    │
                                 │ namespace.            │
                                 └─────────────────────┘
```

---

## 28.2 Pod Security Standards — Three Levels

PSS defines exactly three profiles. They are cumulative — Baseline includes
everything in Privileged, and Restricted includes everything in Baseline.

### Level 1: Privileged — No Restrictions

The **Privileged** profile imposes zero security restrictions. Pods can:
- Run as root
- Use host networking, host PID, host IPC
- Mount any host path
- Run privileged containers
- Use any Linux capabilities
- Disable seccomp

This level is for system-level workloads that genuinely need elevated access:
- `kube-system` Pods (kube-proxy, CNI plugins, CSI drivers)
- Logging agents that need host filesystem access
- Node-level monitoring

```yaml
# This Pod is only valid under the Privileged profile
apiVersion: v1
kind: Pod
metadata:
  name: privileged-system-pod
spec:
  hostNetwork: true        # Access host network stack
  hostPID: true            # See host processes
  containers:
  - name: node-agent
    image: monitoring/node-agent:v1
    securityContext:
      privileged: true     # Full host access
      runAsUser: 0         # Run as root
    volumeMounts:
    - name: host-root
      mountPath: /host
  volumes:
  - name: host-root
    hostPath:
      path: /
```

### Level 2: Baseline — Minimal Restrictions

The **Baseline** profile prevents known privilege escalation vectors while
remaining compatible with most applications. It blocks dangerous settings
but doesn't require specific secure settings.

**What Baseline restricts:**

| Field                           | Restriction                                        |
|---------------------------------|----------------------------------------------------|
| `spec.hostNetwork`              | Must be `false` or unset                           |
| `spec.hostPID`                  | Must be `false` or unset                           |
| `spec.hostIPC`                  | Must be `false` or unset                           |
| `spec.volumes[*].hostPath`      | Forbidden                                          |
| `spec.containers[*].securityContext.privileged` | Must be `false` or unset        |
| `spec.containers[*].ports[*].hostPort` | Must be 0 or undefined                     |
| `spec.containers[*].securityContext.capabilities.add` | Only allows: `AUDIT_WRITE, CHOWN, DAC_OVERRIDE, FCHMOD, FOWNER, FSETID, KILL, MKNOD, NET_BIND_SERVICE, SETFCAP, SETGID, SETPCAP, SETUID, SYS_CHROOT` |
| `spec.containers[*].securityContext.procMount` | Must be `Default` or unset        |
| `spec.containers[*].securityContext.seLinuxOptions.type` | Must be one of: `container_t, container_init_t, container_kvm_t` or unset |
| `spec.containers[*].securityContext.seccompProfile.type` | Must not be `Unconfined` (Kubernetes 1.25+) |
| `spec.containers[*].securityContext.windowsOptions.hostProcess` | Must be `false` or unset |
| `spec.containers[*].securityContext.appArmorProfile.type` | Must not be custom non-standard profiles |

A Baseline-compliant Pod looks like this:

```yaml
# This Pod passes Baseline validation
apiVersion: v1
kind: Pod
metadata:
  name: baseline-pod
spec:
  containers:
  - name: app
    image: nginx:1.27
    ports:
    - containerPort: 8080      # containerPort is fine; hostPort is not
    securityContext:
      privileged: false        # Explicitly not privileged
      capabilities:
        add: ["NET_BIND_SERVICE"]  # Only allowed capabilities
```

### Level 3: Restricted — Heavily Restricted

The **Restricted** profile is for security-critical workloads. It requires
explicit security settings and drops all unnecessary privileges.

**What Restricted requires (in addition to Baseline):**

| Field                           | Requirement                                        |
|---------------------------------|----------------------------------------------------|
| `spec.volumes[*]`              | Only allowed types: `configMap, csi, downwardAPI, emptyDir, ephemeral, persistentVolumeClaim, projected, secret` |
| `spec.containers[*].securityContext.runAsNonRoot` | Must be `true`                  |
| `spec.containers[*].securityContext.runAsUser`    | Must not be `0`                 |
| `spec.containers[*].securityContext.allowPrivilegeEscalation` | Must be `false`   |
| `spec.containers[*].securityContext.capabilities.drop` | Must include `ALL`         |
| `spec.containers[*].securityContext.capabilities.add` | Only `NET_BIND_SERVICE` allowed |
| `spec.containers[*].securityContext.seccompProfile.type` | Must be `RuntimeDefault` or `Localhost` |

A Restricted-compliant Pod:

```yaml
# This Pod passes Restricted validation — production-ready security
apiVersion: v1
kind: Pod
metadata:
  name: restricted-pod
spec:
  securityContext:
    runAsNonRoot: true              # Pod-level: no root containers
    seccompProfile:
      type: RuntimeDefault          # Pod-level seccomp profile
  containers:
  - name: app
    image: myapp:v1.2.3
    securityContext:
      runAsNonRoot: true            # Container-level: redundant but explicit
      runAsUser: 1000               # Run as non-root user
      runAsGroup: 1000              # Run as non-root group
      allowPrivilegeEscalation: false  # Cannot gain more privileges
      readOnlyRootFilesystem: true  # Immutable container filesystem
      capabilities:
        drop: ["ALL"]               # Drop ALL Linux capabilities
        add: ["NET_BIND_SERVICE"]   # Add back only what's needed
      seccompProfile:
        type: RuntimeDefault        # Container-level seccomp
    volumeMounts:
    - name: data
      mountPath: /data
    - name: tmp
      mountPath: /tmp               # Writable tmp via emptyDir
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: app-data
  - name: tmp
    emptyDir: {}                    # Allowed volume type
```

### Quick Reference: Level Comparison

```
┌────────────────────────────────────────────────────────────────┐
│                    Privileged                                   │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                  Baseline                                 │  │
│  │  ┌────────────────────────────────────────────────────┐  │  │
│  │  │                Restricted                           │  │  │
│  │  │                                                     │  │  │
│  │  │  ✓ Non-root                                         │  │  │
│  │  │  ✓ Drop ALL capabilities                            │  │  │
│  │  │  ✓ No privilege escalation                          │  │  │
│  │  │  ✓ Seccomp RuntimeDefault                           │  │  │
│  │  │  ✓ Restricted volume types                          │  │  │
│  │  └────────────────────────────────────────────────────┘  │  │
│  │                                                           │  │
│  │  ✓ No hostNetwork/hostPID/hostIPC                        │  │
│  │  ✓ No privileged containers                              │  │
│  │  ✓ No hostPath volumes                                   │  │
│  │  ✓ No dangerous capabilities                             │  │
│  │  ✓ No hostPorts                                          │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                │
│  No restrictions at all                                        │
└────────────────────────────────────────────────────────────────┘
```

---

## 28.3 Pod Security Admission (PSA)

### How PSA Works

PSA is an admission controller built into the API server. It evaluates Pods
against PSS levels based on **labels on the namespace**. No CRDs, no webhooks,
no external dependencies.

### Namespace Labels

PSA is configured entirely through namespace labels:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce: reject Pods that violate the "restricted" profile
    pod-security.kubernetes.io/enforce: restricted

    # Audit: log violations of the "restricted" profile
    pod-security.kubernetes.io/audit: restricted

    # Warn: show warnings for violations of the "restricted" profile
    pod-security.kubernetes.io/warn: restricted
```

### The Three Modes

| Mode      | Behavior                                          | Use Case                          |
|-----------|---------------------------------------------------|-----------------------------------|
| `enforce` | **Rejects** Pods that violate the policy          | Production enforcement            |
| `audit`   | **Logs** violations to the audit log              | Monitoring and compliance         |
| `warn`    | **Shows warnings** to the user (kubectl output)   | Migration and awareness           |

You can mix modes and levels:

```yaml
labels:
  # Enforce baseline (block dangerous Pods)
  pod-security.kubernetes.io/enforce: baseline
  # Warn about restricted violations (prepare for upgrade)
  pod-security.kubernetes.io/warn: restricted
  # Audit restricted violations (for compliance reports)
  pod-security.kubernetes.io/audit: restricted
```

This is the recommended migration pattern:
1. Start with `warn: restricted` to see what would break
2. Add `audit: restricted` to log violations
3. After fixing all workloads, switch to `enforce: restricted`

### Version Pinning

You can pin PSA to a specific Kubernetes version's definition of the standard:

```yaml
labels:
  pod-security.kubernetes.io/enforce: restricted
  pod-security.kubernetes.io/enforce-version: v1.28
```

This ensures that when you upgrade Kubernetes, new restrictions added to the
"restricted" profile don't suddenly break your workloads. You upgrade the
version label explicitly after verifying compatibility.

If omitted, PSA uses the latest version — which changes with each Kubernetes
release.

### Hands-on: Configuring PSA for a Namespace

```bash
# Create a namespace with PSA labels
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: secure-apps
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
EOF

# Verify the labels
kubectl get namespace secure-apps --show-labels

# Or apply labels to an existing namespace
kubectl label namespace existing-ns \
  pod-security.kubernetes.io/enforce=baseline \
  pod-security.kubernetes.io/warn=restricted \
  --overwrite
```

### Hands-on: Testing PSA Violations

```bash
# Test 1: Try to create a privileged Pod in the restricted namespace
cat <<'EOF' | kubectl apply -f - 2>&1
apiVersion: v1
kind: Pod
metadata:
  name: bad-pod
  namespace: secure-apps
spec:
  containers:
  - name: bad
    image: nginx:1.27
    securityContext:
      privileged: true
EOF
# Error from server (Forbidden): error when creating "STDIN":
# pods "bad-pod" is forbidden: violates PodSecurity "restricted:v1.28":
# privileged (container "bad" must not set securityContext.privileged=true),
# allowPrivilegeEscalation != false, ...

# Test 2: Try a Pod that runs as root
cat <<'EOF' | kubectl apply -f - 2>&1
apiVersion: v1
kind: Pod
metadata:
  name: root-pod
  namespace: secure-apps
spec:
  containers:
  - name: root
    image: nginx:1.27
    securityContext:
      runAsUser: 0       # Root!
EOF
# Error: violates PodSecurity "restricted:v1.28":
# runAsNonRoot must be true, runAsUser must not be 0

# Test 3: A properly configured Pod that passes
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: good-pod
  namespace: secure-apps
spec:
  securityContext:
    runAsNonRoot: true
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: app
    image: nginx:1.27
    securityContext:
      runAsUser: 1000
      runAsGroup: 1000
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/nginx
    - name: run
      mountPath: /var/run
  volumes:
  - name: tmp
    emptyDir: {}
  - name: cache
    emptyDir: {}
  - name: run
    emptyDir: {}
EOF
# pod/good-pod created ✓

# Test 4: Check what happens with warn mode
kubectl label namespace secure-apps \
  pod-security.kubernetes.io/enforce=baseline \
  --overwrite

# Now the privileged Pod is blocked, but a root Pod gets a warning:
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: root-pod-2
  namespace: secure-apps
spec:
  containers:
  - name: root
    image: nginx:1.27
EOF
# Warning: would violate PodSecurity "restricted:v1.28": ...
# pod/root-pod-2 created  <-- Still created! Only a warning.
```

### Exemptions

PSA supports exemptions for specific users, namespaces, and runtime classes:

```yaml
# Configured via the API server admission configuration file
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: PodSecurity
  configuration:
    apiVersion: pod-security.admission.config.k8s.io/v1
    kind: PodSecurityConfiguration
    defaults:
      enforce: baseline
      enforce-version: latest
      audit: restricted
      audit-version: latest
      warn: restricted
      warn-version: latest
    exemptions:
      # These users bypass PSA checks
      usernames:
      - "system:serviceaccount:kube-system:replicaset-controller"
      # These namespaces are exempt
      namespaces:
      - kube-system
      - kube-public
      # Pods using these RuntimeClasses are exempt
      runtimeClasses:
      - kata-containers
```

---

## 28.4 SecurityContext Deep Dive

SecurityContext is the Kubernetes API field where you configure container and
Pod-level security settings. It exists at two levels: the Pod level and the
container level.

### Container-Level SecurityContext

Every container has its own SecurityContext:

```yaml
containers:
- name: app
  image: myapp:v1
  securityContext:
    # ── Identity ──────────────────────────────────────────
    # Run the container process as user ID 1000 (not root).
    runAsUser: 1000

    # Run the container process with group ID 1000.
    runAsGroup: 1000

    # Kubernetes will reject the Pod if the container image
    # is configured to run as root (UID 0).
    runAsNonRoot: true

    # ── Filesystem ────────────────────────────────────────
    # Mount the container's root filesystem as read-only.
    # The container cannot write to /app, /etc, /usr, etc.
    # Use emptyDir volumes for writable directories.
    readOnlyRootFilesystem: true

    # ── Privilege Escalation ──────────────────────────────
    # Prevent the container from gaining additional privileges
    # via setuid binaries or capabilities. This sets the
    # no_new_privs Linux flag.
    allowPrivilegeEscalation: false

    # ── Capabilities ──────────────────────────────────────
    # Linux capabilities provide fine-grained privilege control.
    # Drop ALL and add back only what's needed.
    capabilities:
      drop: ["ALL"]
      # NET_BIND_SERVICE allows binding to ports < 1024.
      # Only add this if the application needs it.
      add: ["NET_BIND_SERVICE"]

    # ── Seccomp ───────────────────────────────────────────
    # Seccomp (Secure Computing Mode) restricts which system
    # calls the container can make. RuntimeDefault provides
    # a sensible default that blocks dangerous syscalls.
    seccompProfile:
      type: RuntimeDefault
      # Alternatives:
      # type: Unconfined          # No restrictions (dangerous)
      # type: Localhost            # Custom profile
      # localhostProfile: "profiles/my-app.json"

    # ── SELinux ───────────────────────────────────────────
    # SELinux options (only on SELinux-enabled hosts like RHEL).
    seLinuxOptions:
      level: "s0:c123,c456"
      # type: "container_t"       # Default container type
```

### Pod-Level SecurityContext

Pod-level settings apply to all containers and also control volume permissions:

```yaml
spec:
  securityContext:
    # ── Identity (applies to all containers) ──────────────
    runAsUser: 1000
    runAsGroup: 1000
    runAsNonRoot: true

    # ── Volume Permissions ────────────────────────────────
    # fsGroup sets the group ownership of ALL volumes mounted
    # into the Pod. The kubelet will:
    # 1. Change the group of all files in all volumes to this GID
    # 2. Set the setgid bit so new files inherit this group
    # This ensures the container can read/write volume data.
    fsGroup: 2000

    # fsGroupChangePolicy controls how fsGroup is applied:
    # "Always" (default): change ownership on every mount
    # "OnRootMismatch": only change if current ownership doesn't match
    fsGroupChangePolicy: "OnRootMismatch"

    # supplementalGroups adds the container user to these groups.
    # Useful for accessing files owned by specific groups.
    supplementalGroups: [3000, 4000]

    # ── Sysctls ───────────────────────────────────────────
    # Kernel parameters that can be set per-Pod (namespace-scoped).
    # Only "safe" sysctls are allowed by default. Unsafe sysctls
    # must be explicitly enabled on the kubelet.
    sysctls:
    - name: "net.ipv4.ip_unprivileged_port_start"
      value: "0"    # Allow non-root to bind to any port

    # ── Seccomp (Pod-level) ───────────────────────────────
    seccompProfile:
      type: RuntimeDefault
```

### Priority: Container vs Pod SecurityContext

Container-level settings override Pod-level settings:

```yaml
spec:
  securityContext:
    runAsUser: 1000              # Default for all containers
    runAsNonRoot: true
  containers:
  - name: app
    image: myapp:v1
    # Inherits runAsUser: 1000 from Pod
  - name: sidecar
    image: envoy:v1
    securityContext:
      runAsUser: 2000            # Overrides Pod-level setting
```

### Hands-on: Complete SecurityContext for Production

```yaml
# production-pod.yaml
# A fully hardened Pod suitable for production deployment.
# Every security field is explicitly set — no reliance on defaults.
apiVersion: v1
kind: Pod
metadata:
  name: production-app
  namespace: production
  labels:
    app: my-app
    version: v1.2.3
spec:
  # ── ServiceAccount ────────────────────────────────────────
  serviceAccountName: my-app-sa
  automountServiceAccountToken: false   # Don't mount SA token

  # ── Pod-Level Security ────────────────────────────────────
  securityContext:
    runAsNonRoot: true
    runAsUser: 10000
    runAsGroup: 10000
    fsGroup: 10000
    fsGroupChangePolicy: "OnRootMismatch"
    seccompProfile:
      type: RuntimeDefault

  # ── DNS Policy ────────────────────────────────────────────
  # Prevent DNS rebinding attacks by restricting DNS
  dnsPolicy: ClusterFirst

  # ── Host Settings ─────────────────────────────────────────
  hostNetwork: false
  hostPID: false
  hostIPC: false

  # ── Containers ────────────────────────────────────────────
  containers:
  - name: app
    image: registry.example.com/my-app:v1.2.3@sha256:abc123...
    # Always use image digest for immutability ↑

    ports:
    - containerPort: 8080
      protocol: TCP

    securityContext:
      runAsNonRoot: true
      runAsUser: 10000
      runAsGroup: 10000
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
        # No capabilities added — most apps don't need any
      seccompProfile:
        type: RuntimeDefault

    # ── Resource Limits ─────────────────────────────────────
    resources:
      requests:
        memory: "128Mi"
        cpu: "100m"
      limits:
        memory: "256Mi"
        cpu: "500m"

    # ── Health Checks ───────────────────────────────────────
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 30
    readinessProbe:
      httpGet:
        path: /readyz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 10

    # ── Volume Mounts ───────────────────────────────────────
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: data
      mountPath: /data
      readOnly: true              # Mount as read-only where possible

  # ── Volumes ───────────────────────────────────────────────
  volumes:
  - name: tmp
    emptyDir:
      sizeLimit: "100Mi"          # Limit tmp size to prevent DoS
  - name: data
    configMap:
      name: my-app-config
```

---

## 28.5 OPA/Gatekeeper — Policy Engine

### What Is OPA?

**Open Policy Agent (OPA)** is a general-purpose policy engine. You write
policies in a language called **Rego**, and OPA evaluates inputs against those
policies. OPA is used for:
- Kubernetes admission control
- API authorization
- Infrastructure-as-code validation
- Data filtering

### What Is Gatekeeper?

**Gatekeeper** is the Kubernetes-native integration of OPA. It runs as an
admission webhook and provides:
- CRD-based policy management (ConstraintTemplates and Constraints)
- Audit functionality (scan existing resources for violations)
- Status reporting (which resources violate which policies)
- Mutation support (modify resources to comply with policies)

```
                                    ┌─────────────────┐
kubectl apply ─→ API Server ─→ ─→  │ Gatekeeper       │
                                    │ Webhook          │
                                    │                  │
                                    │ Evaluates Rego   │
                                    │ policies against │
                                    │ the resource     │
                                    └──────┬──────────┘
                                           │
                                    Allow / Deny
```

### Architecture: ConstraintTemplate → Constraint

Gatekeeper uses a two-layer model:

1. **ConstraintTemplate** — defines the policy logic in Rego
2. **Constraint** — applies the policy to specific resources with parameters

```
ConstraintTemplate                 Constraint
(defines the Rego policy)         (applies the policy)
┌──────────────────────┐          ┌────────────────────────────┐
│ "Pods must have       │    ←──── │ Match: all Pods             │
│  these required       │          │ Parameters:                 │
│  labels"              │          │   requiredLabels:           │
│                       │          │   - "app"                   │
│ (reusable template)  │          │   - "team"                  │
└──────────────────────┘          └────────────────────────────┘
```

### Hands-on: Installing Gatekeeper

```bash
# Install Gatekeeper using the official Helm chart
helm repo add gatekeeper https://open-policy-agent.github.io/gatekeeper/charts
helm repo update

helm install gatekeeper gatekeeper/gatekeeper \
  --namespace gatekeeper-system \
  --create-namespace \
  --set replicas=3 \
  --set audit.replicas=1

# Verify installation
kubectl get pods -n gatekeeper-system
# NAME                                      READY   STATUS    RESTARTS   AGE
# gatekeeper-audit-xyz                      1/1     Running   0          30s
# gatekeeper-controller-manager-abc         1/1     Running   0          30s
# gatekeeper-controller-manager-def         1/1     Running   0          30s
# gatekeeper-controller-manager-ghi         1/1     Running   0          30s

# Check CRDs
kubectl get crd | grep gatekeeper
# configs.config.gatekeeper.sh
# constraintpodstatuses.status.gatekeeper.sh
# constrainttemplatepodstatuses.status.gatekeeper.sh
# constrainttemplates.templates.gatekeeper.sh
# ...
```

### Hands-on: ConstraintTemplate — Required Labels

```yaml
# template-required-labels.yaml
# This ConstraintTemplate defines a policy that requires specific
# labels on resources. The template is reusable — you create
# Constraints to apply it with different parameters.
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            # Parameters that the Constraint can pass to this template.
            labels:
              type: array
              description: "List of labels that must be present on the resource."
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      # Package name must match the ConstraintTemplate name.
      package k8srequiredlabels

      # violation is the entry point. Gatekeeper calls this for each
      # resource that matches the Constraint's match criteria.
      # It receives the resource in input.review.object and the
      # parameters in input.parameters.
      violation[{"msg": msg}] {
        # Get the set of labels that exist on the resource.
        provided := {label | input.review.object.metadata.labels[label]}

        # Get the set of required labels from the constraint parameters.
        required := {label | label := input.parameters.labels[_]}

        # Find any missing labels (required but not provided).
        missing := required - provided

        # If there are missing labels, this is a violation.
        count(missing) > 0

        # Build the error message.
        msg := sprintf("Resource %v is missing required labels: %v", [
          input.review.object.metadata.name,
          missing
        ])
      }
---
# constraint-require-app-label.yaml
# Apply the template: ALL Pods must have "app" and "team" labels.
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: pods-must-have-labels
spec:
  enforcementAction: deny    # Options: deny, dryrun, warn
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    # Exclude system namespaces
    excludedNamespaces:
    - kube-system
    - kube-public
    - gatekeeper-system
  parameters:
    labels:
    - "app"
    - "team"
```

```bash
# Apply the template and constraint
kubectl apply -f template-required-labels.yaml
kubectl apply -f constraint-require-app-label.yaml

# Test: Pod without labels (should be rejected)
cat <<'EOF' | kubectl apply -f - 2>&1
apiVersion: v1
kind: Pod
metadata:
  name: unlabeled-pod
spec:
  containers:
  - name: app
    image: nginx:1.27
EOF
# Error: admission webhook "validation.gatekeeper.sh" denied the request:
# [pods-must-have-labels] Resource unlabeled-pod is missing required labels: {"app", "team"}

# Test: Pod with labels (should pass)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: labeled-pod
  labels:
    app: my-app
    team: backend
spec:
  containers:
  - name: app
    image: nginx:1.27
EOF
# pod/labeled-pod created ✓

# Check audit results (violations of existing resources)
kubectl get k8srequiredlabels pods-must-have-labels -o yaml
```

### Hands-on: ConstraintTemplate — Block Privileged Containers

```yaml
# template-block-privileged.yaml
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sblockprivileged
spec:
  crd:
    spec:
      names:
        kind: K8sBlockPrivileged
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8sblockprivileged

      # Check all containers (including init containers and
      # ephemeral containers) for privileged mode.
      violation[{"msg": msg}] {
        # Iterate over all container types.
        container := input.review.object.spec.containers[_]
        container.securityContext.privileged == true
        msg := sprintf("Container %v in Pod %v is privileged. Privileged containers are not allowed.", [
          container.name,
          input.review.object.metadata.name
        ])
      }

      violation[{"msg": msg}] {
        container := input.review.object.spec.initContainers[_]
        container.securityContext.privileged == true
        msg := sprintf("Init container %v in Pod %v is privileged. Privileged containers are not allowed.", [
          container.name,
          input.review.object.metadata.name
        ])
      }
---
# Apply to all Pods cluster-wide (except system namespaces)
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sBlockPrivileged
metadata:
  name: block-privileged-containers
spec:
  enforcementAction: deny
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    excludedNamespaces:
    - kube-system
    - kube-public
    - gatekeeper-system
```

### Hands-on: ConstraintTemplate — Image Registry Whitelist

```yaml
# template-allowed-registries.yaml
# Only allow container images from approved registries.
# This prevents pulling images from public registries or
# unknown sources in production.
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sallowedregistries
spec:
  crd:
    spec:
      names:
        kind: K8sAllowedRegistries
      validation:
        openAPIV3Schema:
          type: object
          properties:
            registries:
              type: array
              description: "List of allowed image registry prefixes."
              items:
                type: string
  targets:
  - target: admission.k8s.gatekeeper.sh
    rego: |
      package k8sallowedregistries

      violation[{"msg": msg}] {
        container := input.review.object.spec.containers[_]
        not image_allowed(container.image)
        msg := sprintf("Container %v uses image %v which is not from an allowed registry. Allowed registries: %v", [
          container.name,
          container.image,
          input.parameters.registries
        ])
      }

      violation[{"msg": msg}] {
        container := input.review.object.spec.initContainers[_]
        not image_allowed(container.image)
        msg := sprintf("Init container %v uses image %v which is not from an allowed registry. Allowed registries: %v", [
          container.name,
          container.image,
          input.parameters.registries
        ])
      }

      # image_allowed checks if the image starts with any of the
      # approved registry prefixes.
      image_allowed(image) {
        registry := input.parameters.registries[_]
        startswith(image, registry)
      }
---
# Only allow images from our private registry and gcr.io
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sAllowedRegistries
metadata:
  name: production-registries
spec:
  enforcementAction: deny
  match:
    kinds:
    - apiGroups: [""]
      kinds: ["Pod"]
    namespaces:
    - production
    - staging
  parameters:
    registries:
    - "registry.example.com/"
    - "gcr.io/my-project/"
    - "us-docker.pkg.dev/my-project/"
```

```bash
# Test: image from unauthorized registry
cat <<'EOF' | kubectl apply -f - -n production 2>&1
apiVersion: v1
kind: Pod
metadata:
  name: bad-image
  labels:
    app: test
    team: test
spec:
  containers:
  - name: app
    image: nginx:1.27    # From Docker Hub — not allowed!
EOF
# Error: Container app uses image nginx:1.27 which is not from an allowed registry.

# Test: image from approved registry
cat <<'EOF' | kubectl apply -f - -n production
apiVersion: v1
kind: Pod
metadata:
  name: good-image
  labels:
    app: test
    team: test
spec:
  containers:
  - name: app
    image: registry.example.com/nginx:1.27   # Allowed!
EOF
# pod/good-image created ✓
```

---

## 28.6 Kyverno — Alternative Policy Engine

### Why Kyverno?

Kyverno is a Kubernetes-native policy engine that uses **YAML** instead of Rego.
For teams that don't want to learn a new language, Kyverno is more accessible.

Key differences from Gatekeeper:

| Feature           | Gatekeeper (OPA)              | Kyverno                      |
|-------------------|-------------------------------|------------------------------|
| Policy language   | Rego                          | YAML (declarative)           |
| Policy types      | Validate                      | Validate, Mutate, Generate   |
| Learning curve    | Steep (Rego)                  | Low (YAML)                   |
| Flexibility       | Very high (Turing-complete)   | High for common patterns     |
| Community         | CNCF graduated                | CNCF incubating              |

### Installation

```bash
# Install Kyverno via Helm
helm repo add kyverno https://kyverno.github.io/kyverno/
helm repo update

helm install kyverno kyverno/kyverno \
  --namespace kyverno \
  --create-namespace

# Verify
kubectl get pods -n kyverno
```

### Validate Policy — Require Non-Root

```yaml
# kyverno-require-nonroot.yaml
# Kyverno policy that requires all containers to run as non-root.
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-run-as-nonroot
  annotations:
    policies.kyverno.io/title: Require Non-Root
    policies.kyverno.io/category: Pod Security
    policies.kyverno.io/severity: high
    policies.kyverno.io/description: >-
      Containers must set runAsNonRoot to true and must not
      run as UID 0 (root). Running containers as root is a
      significant security risk.
spec:
  validationFailureAction: Enforce   # or Audit
  background: true                    # Scan existing resources
  rules:
  - name: run-as-non-root
    match:
      any:
      - resources:
          kinds:
          - Pod
    exclude:
      any:
      - resources:
          namespaces:
          - kube-system
    validate:
      message: "Containers must not run as root. Set runAsNonRoot: true and runAsUser to a non-zero value."
      pattern:
        spec:
          containers:
          - securityContext:
              runAsNonRoot: true
              runAsUser: ">0"
```

### Mutate Policy — Add Default Security Settings

```yaml
# kyverno-mutate-security-defaults.yaml
# Kyverno mutation policy that automatically adds security defaults
# to Pods that don't have them. This is the "shift-left" approach —
# instead of rejecting Pods, fix them automatically.
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: add-security-defaults
  annotations:
    policies.kyverno.io/title: Add Security Defaults
    policies.kyverno.io/category: Pod Security
    policies.kyverno.io/description: >-
      Automatically adds security defaults to Pods that don't
      specify them. Adds readOnlyRootFilesystem, drops all
      capabilities, and sets runAsNonRoot.
spec:
  rules:
  - name: add-readonly-rootfs
    match:
      any:
      - resources:
          kinds:
          - Pod
    exclude:
      any:
      - resources:
          namespaces:
          - kube-system
    mutate:
      patchStrategicMerge:
        spec:
          containers:
          - (name): "*"
            securityContext:
              readOnlyRootFilesystem: true
              allowPrivilegeEscalation: false
              capabilities:
                drop:
                - ALL
```

### Generate Policy — Auto-Create NetworkPolicy

```yaml
# kyverno-generate-netpol.yaml
# Kyverno generate policy that automatically creates a default-deny
# NetworkPolicy in every new namespace. This ensures network
# isolation by default.
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: generate-default-deny-netpol
  annotations:
    policies.kyverno.io/title: Generate Default Deny NetworkPolicy
    policies.kyverno.io/description: >-
      Automatically creates a default-deny NetworkPolicy in every
      new namespace to ensure network isolation by default.
spec:
  rules:
  - name: default-deny
    match:
      any:
      - resources:
          kinds:
          - Namespace
    exclude:
      any:
      - resources:
          names:
          - kube-system
          - kube-public
          - kube-node-lease
          - kyverno
    generate:
      apiVersion: networking.k8s.io/v1
      kind: NetworkPolicy
      name: default-deny
      namespace: "{{request.object.metadata.name}}"
      synchronize: true     # Keep in sync if policy changes
      data:
        spec:
          podSelector: {}   # Applies to all Pods in the namespace
          policyTypes:
          - Ingress
          - Egress
```

### Kyverno Policy for Common Security Requirements

```yaml
# kyverno-security-bundle.yaml
# A bundle of common security policies for production clusters.

# Policy 1: Require resource limits
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-resource-limits
spec:
  validationFailureAction: Enforce
  rules:
  - name: check-limits
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "All containers must have CPU and memory limits set."
      pattern:
        spec:
          containers:
          - resources:
              limits:
                memory: "?*"
                cpu: "?*"
---
# Policy 2: Disallow latest tag
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
spec:
  validationFailureAction: Enforce
  rules:
  - name: no-latest
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "The ':latest' tag is not allowed. Use a specific version tag."
      pattern:
        spec:
          containers:
          - image: "!*:latest"
---
# Policy 3: Require probes
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: require-probes
spec:
  validationFailureAction: Audit   # Audit first, enforce later
  rules:
  - name: check-probes
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: "Containers must have liveness and readiness probes."
      pattern:
        spec:
          containers:
          - livenessProbe:
              httpGet:
                path: "?*"
            readinessProbe:
              httpGet:
                path: "?*"
```

---

## 28.7 Runtime Security

### Falco — Runtime Threat Detection

**Falco** is a CNCF runtime security tool that detects unexpected behavior
in running containers. While PSA and Gatekeeper work at admission time
(before the Pod runs), Falco works at **runtime** (while the Pod is running).

```
Admission Time                          Runtime
┌──────────────┐                       ┌──────────────────────┐
│ PSA          │                       │ Falco                │
│ Gatekeeper   │  Pod runs ────────→   │                      │
│ Kyverno      │                       │ Monitors syscalls    │
│              │                       │ Detects anomalies    │
│ "Should this │                       │ "Is this container   │
│  Pod be      │                       │  doing something     │
│  allowed?"   │                       │  unexpected?"        │
└──────────────┘                       └──────────────────────┘
```

Falco monitors Linux system calls using eBPF or a kernel module and
applies rules to detect suspicious activity:
- Shell spawned in a container
- Sensitive file read (/etc/shadow, /etc/passwd)
- Unexpected network connections
- Binary execution that wasn't in the original image
- Privilege escalation attempts

### Installing Falco

```bash
# Install Falco via Helm
helm repo add falcosecurity https://falcosecurity.github.io/charts
helm repo update

helm install falco falcosecurity/falco \
  --namespace falco \
  --create-namespace \
  --set driver.kind=ebpf \
  --set falcosidekick.enabled=true \
  --set falcosidekick.config.slack.webhookurl="https://hooks.slack.com/..."
```

### Hands-on: Falco Rules for Container Security

```yaml
# falco-custom-rules.yaml
# Custom Falco rules for detecting container security violations.

# Rule 1: Detect shell spawned in a container
# A shell being opened in a container is almost always either:
# - An attacker who has gained access
# - A developer debugging (which shouldn't happen in production)
- rule: Shell Spawned in Container
  desc: Detect a shell being spawned inside a running container
  condition: >
    spawned_process and
    container and
    proc.name in (bash, sh, zsh, dash, csh, ksh, fish) and
    not container.image.repository in (
      "docker.io/library/busybox",
      "docker.io/library/alpine"
    )
  output: >
    Shell spawned in container
    (user=%user.name user_loginuid=%user.loginuid container_id=%container.id
     container_name=%container.name image=%container.image.repository
     shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline
     terminal=%proc.tty container_id=%container.id)
  priority: WARNING
  tags: [container, shell, mitre_execution]

# Rule 2: Detect sensitive file access
# Containers should never read /etc/shadow or other host-level
# sensitive files. If they do, it's likely a container escape or
# a misconfigured hostPath mount.
- rule: Read Sensitive File in Container
  desc: Detect reading of sensitive files within a container
  condition: >
    open_read and
    container and
    fd.name in (/etc/shadow, /etc/sudoers, /root/.ssh/authorized_keys,
                /root/.bash_history, /etc/kubernetes/admin.conf) and
    not proc.name in (sshd, sudo)
  output: >
    Sensitive file read in container
    (user=%user.name file=%fd.name container=%container.name
     image=%container.image.repository command=%proc.cmdline)
  priority: CRITICAL
  tags: [container, filesystem, mitre_credential_access]

# Rule 3: Detect unexpected outbound connections
# Production containers should only connect to known services.
# Outbound connections to unusual ports could indicate data exfiltration.
- rule: Unexpected Outbound Connection from Container
  desc: Detect outbound network connections to unusual destinations
  condition: >
    outbound and
    container and
    fd.sport in (4444, 5555, 6666, 8888, 9999, 1337) and
    not container.image.repository startswith "registry.example.com/"
  output: >
    Unexpected outbound connection from container
    (container=%container.name image=%container.image.repository
     connection=%fd.name sport=%fd.sport dport=%fd.dport
     command=%proc.cmdline)
  priority: WARNING
  tags: [container, network, mitre_exfiltration]

# Rule 4: Detect binary modification in container
# If a container's filesystem is read-only (as it should be in
# production), this shouldn't happen. If the filesystem is writable,
# this detects attackers dropping tools.
- rule: Write to Binary Directory in Container
  desc: Detect writing to directories that contain system binaries
  condition: >
    write and
    container and
    fd.directory in (/bin, /sbin, /usr/bin, /usr/sbin, /usr/local/bin)
  output: >
    Binary directory written to in container
    (user=%user.name file=%fd.name container=%container.name
     image=%container.image.repository command=%proc.cmdline)
  priority: CRITICAL
  tags: [container, filesystem, mitre_persistence]
```

### Testing Falco

```bash
# Deploy a test Pod
kubectl run falco-test --image=alpine:3.19 -- sleep 3600

# Trigger a shell detection
kubectl exec -it falco-test -- sh

# Check Falco logs
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=20
# ... Shell spawned in container ...

# Trigger a sensitive file read
kubectl exec falco-test -- cat /etc/shadow

# Check Falco logs again
kubectl logs -n falco -l app.kubernetes.io/name=falco --tail=20
# ... Sensitive file read in container ...
```

---

## 28.8 Hands-on: Hardening a Complete Application

Let's take a real application from insecure defaults to fully hardened,
explaining every change along the way.

### Step 0: The Insecure Default

```yaml
# INSECURE — This is what a typical "it works" deployment looks like.
# Every single field missing here is a security gap.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: default          # ← Using default namespace
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
      - name: web
        image: myapp:latest   # ← :latest tag, no digest
        ports:
        - containerPort: 80   # ← Binding to privileged port 80
```

**What's wrong:**
1. Running in `default` namespace (no isolation)
2. Using `:latest` tag (no version pinning)
3. No image digest (mutable reference)
4. No ServiceAccount specified (uses `default`)
5. No SecurityContext (runs as root)
6. No resource limits (can consume all node resources)
7. No health probes (no automatic recovery)
8. No readOnlyRootFilesystem
9. No capabilities dropped
10. Privilege escalation allowed by default
11. Service account token mounted (unnecessary API access)

### Step 1: Namespace Isolation

```yaml
# Create a dedicated namespace with PSA enforcement
apiVersion: v1
kind: Namespace
metadata:
  name: web-team
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Step 2: Dedicated ServiceAccount

```yaml
# ServiceAccount with token mounting disabled
apiVersion: v1
kind: ServiceAccount
metadata:
  name: web-app-sa
  namespace: web-team
automountServiceAccountToken: false
```

### Step 3: RBAC (Minimal Permissions)

```yaml
# The web app only needs to read its own ConfigMap
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: web-app-role
  namespace: web-team
rules:
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get"]
  resourceNames: ["web-app-config"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: web-app-binding
  namespace: web-team
subjects:
- kind: ServiceAccount
  name: web-app-sa
  namespace: web-team
roleRef:
  kind: Role
  name: web-app-role
  apiGroup: rbac.authorization.k8s.io
```

### Step 4: NetworkPolicy

```yaml
# Restrict network access
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: web-app-netpol
  namespace: web-team
spec:
  podSelector:
    matchLabels:
      app: web-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Only allow traffic from the ingress controller
  - from:
    - namespaceSelector:
        matchLabels:
          kubernetes.io/metadata.name: ingress-nginx
    ports:
    - port: 8080
      protocol: TCP
  egress:
  # Allow DNS
  - to:
    - namespaceSelector: {}
    ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
  # Allow traffic to the database
  - to:
    - podSelector:
        matchLabels:
          app: postgres
    ports:
    - port: 5432
      protocol: TCP
```

### Step 5: The Fully Hardened Deployment

```yaml
# SECURE — Production-hardened deployment with every security control.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: web-team                # ✓ Dedicated namespace
  labels:
    app: web-app
    version: v1.2.3
    team: web-team
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0              # Zero-downtime deployments
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
        version: v1.2.3
        team: web-team
    spec:
      # ── ServiceAccount ──────────────────────────────────
      serviceAccountName: web-app-sa           # ✓ Dedicated SA
      automountServiceAccountToken: false       # ✓ No API access

      # ── Pod-Level Security ──────────────────────────────
      securityContext:
        runAsNonRoot: true                      # ✓ No root
        runAsUser: 10000                        # ✓ Explicit UID
        runAsGroup: 10000                       # ✓ Explicit GID
        fsGroup: 10000                          # ✓ Volume group
        seccompProfile:
          type: RuntimeDefault                  # ✓ Seccomp

      # ── Anti-Affinity ───────────────────────────────────
      # Spread replicas across nodes for resilience.
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values: ["web-app"]
              topologyKey: kubernetes.io/hostname

      # ── Topology Spread ─────────────────────────────────
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: web-app

      # ── Containers ──────────────────────────────────────
      containers:
      - name: web
        # ✓ Pinned version + image digest for immutability
        image: registry.example.com/web-app:v1.2.3@sha256:a1b2c3d4e5f6...
        imagePullPolicy: IfNotPresent

        ports:
        - containerPort: 8080                   # ✓ Unprivileged port
          protocol: TCP

        # ── Container Security ────────────────────────────
        securityContext:
          runAsNonRoot: true                    # ✓ No root
          runAsUser: 10000                      # ✓ Non-root UID
          runAsGroup: 10000                     # ✓ Non-root GID
          allowPrivilegeEscalation: false       # ✓ No escalation
          readOnlyRootFilesystem: true          # ✓ Immutable fs
          capabilities:
            drop: ["ALL"]                       # ✓ Drop all caps
            # No add — the app doesn't need any
          seccompProfile:
            type: RuntimeDefault                # ✓ Seccomp

        # ── Resources ─────────────────────────────────────
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"

        # ── Health Probes ─────────────────────────────────
        startupProbe:
          httpGet:
            path: /healthz
            port: 8080
          failureThreshold: 30
          periodSeconds: 2
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /readyz
            port: 8080
          initialDelaySeconds: 0
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3

        # ── Environment ───────────────────────────────────
        env:
        - name: APP_PORT
          value: "8080"
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: web-app-config
              key: db-host

        # ── Volume Mounts ─────────────────────────────────
        volumeMounts:
        - name: tmp
          mountPath: /tmp
        - name: cache
          mountPath: /var/cache
        - name: app-config
          mountPath: /etc/config
          readOnly: true

      # ── Volumes ─────────────────────────────────────────
      volumes:
      - name: tmp
        emptyDir:
          sizeLimit: "100Mi"                   # ✓ Limit tmp size
      - name: cache
        emptyDir:
          sizeLimit: "200Mi"
      - name: app-config
        configMap:
          name: web-app-config
```

### The Security Checklist

| Control                         | Insecure Default       | Hardened Value                    |
|---------------------------------|------------------------|-----------------------------------|
| Namespace                       | `default`              | `web-team` (with PSA restricted)  |
| ServiceAccount                  | `default` (auto)       | `web-app-sa` (dedicated)          |
| SA Token Mount                  | `true` (auto)          | `false`                           |
| Image Tag                       | `:latest`              | `:v1.2.3@sha256:...`             |
| runAsNonRoot                    | `false` (default)      | `true`                            |
| runAsUser                       | `0` (root)             | `10000`                           |
| readOnlyRootFilesystem          | `false` (default)      | `true`                            |
| allowPrivilegeEscalation        | `true` (default)       | `false`                           |
| capabilities                    | Default set            | `drop: ALL`                       |
| seccompProfile                  | Unconfined             | `RuntimeDefault`                  |
| Resource limits                 | None                   | CPU + Memory set                  |
| Health probes                   | None                   | Startup + Liveness + Readiness    |
| NetworkPolicy                   | None (allow all)       | Ingress + Egress restricted       |
| RBAC                            | None (no permissions)  | Minimal ConfigMap read access     |
| PSA enforcement                 | None                   | `restricted` level enforced       |

---

## 28.9 Go: Pod Security Validation

```go
// Package podsecurity provides utilities for validating Pod specifications
// against Pod Security Standards programmatically. This enables pre-flight
// checks in CI/CD pipelines, admission webhooks, and audit tools without
// relying on the built-in PSA controller.
//
// The validator checks a Pod spec against the Restricted PSS level and
// returns a detailed list of violations. This is useful for:
//   - CI/CD pre-deployment checks
//   - Custom admission webhooks with better error messages
//   - Audit tools that scan existing Pods for compliance
//   - Developer tools that validate manifests before apply
package podsecurity

import (
	"fmt"
	"strings"

	corev1 "k8s.io/api/core/v1"
)

// Violation represents a single Pod Security Standards violation.
// It contains the field path that violates the standard, the current
// value of the field, and a human-readable message explaining the
// required value.
type Violation struct {
	// Field is the JSON path to the violating field in the Pod spec
	// (e.g., "spec.containers[0].securityContext.runAsNonRoot").
	Field string

	// Message is a human-readable description of the violation and
	// how to fix it.
	Message string

	// Level is the PSS level that this check belongs to
	// ("baseline" or "restricted").
	Level string
}

// ValidateRestrictedLevel checks a Pod spec against the Restricted PSS
// level. It returns all violations found. An empty slice means the Pod
// is fully compliant with the Restricted level.
//
// This function checks both Pod-level and container-level security
// settings, including init containers and ephemeral containers.
//
// Parameters:
//   - pod: The Pod specification to validate. Only the spec is examined;
//     metadata and status are ignored.
//
// Returns a slice of Violations. Each Violation describes a single
// non-compliant field with instructions for remediation.
func ValidateRestrictedLevel(pod *corev1.Pod) []Violation {
	var violations []Violation

	// ── Pod-Level Checks ─────────────────────────────────────────

	// Check Pod-level securityContext exists.
	if pod.Spec.SecurityContext == nil {
		violations = append(violations, Violation{
				Field:   "spec.securityContext",
				Message: "Pod-level securityContext is not set. Set runAsNonRoot: true and seccompProfile.",
				Level:   "restricted",
			})
	} else {
		// Check runAsNonRoot at Pod level.
		if pod.Spec.SecurityContext.RunAsNonRoot == nil || !*pod.Spec.SecurityContext.RunAsNonRoot {
			violations = append(violations, Violation{
					Field:   "spec.securityContext.runAsNonRoot",
					Message: "Pod must set runAsNonRoot: true at the Pod level.",
					Level:   "restricted",
				})
		}

		// Check seccomp profile at Pod level.
		if pod.Spec.SecurityContext.SeccompProfile == nil {
			violations = append(violations, Violation{
					Field:   "spec.securityContext.seccompProfile",
					Message: "Pod must set seccompProfile.type to RuntimeDefault or Localhost.",
					Level:   "restricted",
				})
		} else if pod.Spec.SecurityContext.SeccompProfile.Type != corev1.SeccompProfileTypeRuntimeDefault &&
		pod.Spec.SecurityContext.SeccompProfile.Type != corev1.SeccompProfileTypeLocalhost {
			violations = append(violations, Violation{
					Field:   "spec.securityContext.seccompProfile.type",
					Message: fmt.Sprintf("Pod seccompProfile.type must be RuntimeDefault or Localhost, got %q.", pod.Spec.SecurityContext.SeccompProfile.Type),
					Level:   "restricted",
				})
		}
	}

	// ── Baseline Checks (hostNetwork, hostPID, etc.) ─────────────

	// Check hostNetwork.
	if pod.Spec.HostNetwork {
		violations = append(violations, Violation{
				Field:   "spec.hostNetwork",
				Message: "hostNetwork must be false or unset.",
				Level:   "baseline",
			})
	}

	// Check hostPID.
	if pod.Spec.HostPID {
		violations = append(violations, Violation{
				Field:   "spec.hostPID",
				Message: "hostPID must be false or unset.",
				Level:   "baseline",
			})
	}

	// Check hostIPC.
	if pod.Spec.HostIPC {
		violations = append(violations, Violation{
				Field:   "spec.hostIPC",
				Message: "hostIPC must be false or unset.",
				Level:   "baseline",
			})
	}

	// ── Volume Checks ────────────────────────────────────────────

	// Restricted level only allows specific volume types.
	allowedVolumeTypes := map[string]bool{
		"configMap":              true,
		"csi":                   true,
		"downwardAPI":           true,
		"emptyDir":              true,
		"ephemeral":             true,
		"persistentVolumeClaim": true,
		"projected":             true,
		"secret":                true,
	}

	for i, vol := range pod.Spec.Volumes {
		volType := getVolumeType(vol)
		if !allowedVolumeTypes[volType] {
			violations = append(violations, Violation{
					Field:   fmt.Sprintf("spec.volumes[%d]", i),
					Message: fmt.Sprintf("Volume %q uses type %q which is not allowed under Restricted level. Allowed types: configMap, csi, downwardAPI, emptyDir, ephemeral, persistentVolumeClaim, projected, secret.", vol.Name, volType),
					Level:   "restricted",
				})
		}
	}

	// ── Container Checks ─────────────────────────────────────────

	// Check all container types.
	allContainers := []struct {
		containers []corev1.Container
		prefix     string
	}{
		{pod.Spec.Containers, "spec.containers"},
		{pod.Spec.InitContainers, "spec.initContainers"},
	}

	for _, group := range allContainers {
		for i, c := range group.containers {
			prefix := fmt.Sprintf("%s[%d]", group.prefix, i)
			violations = append(violations, validateContainer(c, prefix)...)
		}
	}

	return violations
}

// validateContainer checks a single container's SecurityContext against
// the Restricted PSS level. It returns violations for any non-compliant
// settings.
//
// Parameters:
//   - c: The container specification to validate.
//   - prefix: The JSON path prefix for this container (e.g.,
	//     "spec.containers[0]") used in violation field paths.
//
// Returns a slice of Violations for this specific container.
func validateContainer(c corev1.Container, prefix string) []Violation {
	var violations []Violation

	// Check that securityContext exists.
	if c.SecurityContext == nil {
		violations = append(violations, Violation{
				Field:   prefix + ".securityContext",
				Message: fmt.Sprintf("Container %q must have a securityContext.", c.Name),
				Level:   "restricted",
			})
		return violations // Can't check fields on nil securityContext
	}

	sc := c.SecurityContext

	// ── Baseline: privileged ─────────────────────────────────────
	if sc.Privileged != nil && *sc.Privileged {
		violations = append(violations, Violation{
				Field:   prefix + ".securityContext.privileged",
				Message: fmt.Sprintf("Container %q must not set privileged: true.", c.Name),
				Level:   "baseline",
			})
	}

	// ── Restricted: runAsNonRoot ─────────────────────────────────
	if sc.RunAsNonRoot == nil || !*sc.RunAsNonRoot {
		violations = append(violations, Violation{
				Field:   prefix + ".securityContext.runAsNonRoot",
				Message: fmt.Sprintf("Container %q must set runAsNonRoot: true.", c.Name),
				Level:   "restricted",
			})
	}

	// ── Restricted: runAsUser ────────────────────────────────────
	if sc.RunAsUser != nil && *sc.RunAsUser == 0 {
		violations = append(violations, Violation{
				Field:   prefix + ".securityContext.runAsUser",
				Message: fmt.Sprintf("Container %q must not run as root (UID 0).", c.Name),
				Level:   "restricted",
			})
	}

	// ── Restricted: allowPrivilegeEscalation ─────────────────────
	if sc.AllowPrivilegeEscalation == nil || *sc.AllowPrivilegeEscalation {
		violations = append(violations, Violation{
				Field:   prefix + ".securityContext.allowPrivilegeEscalation",
				Message: fmt.Sprintf("Container %q must set allowPrivilegeEscalation: false.", c.Name),
				Level:   "restricted",
			})
	}

	// ── Restricted: capabilities ─────────────────────────────────
	if sc.Capabilities == nil {
		violations = append(violations, Violation{
				Field:   prefix + ".securityContext.capabilities",
				Message: fmt.Sprintf("Container %q must set capabilities.drop: [\"ALL\"].", c.Name),
				Level:   "restricted",
			})
	} else {
		// Check that ALL is dropped.
		hasDropAll := false
		for _, cap := range sc.Capabilities.Drop {
			if string(cap) == "ALL" {
				hasDropAll = true
				break
			}
		}
		if !hasDropAll {
			violations = append(violations, Violation{
					Field:   prefix + ".securityContext.capabilities.drop",
					Message: fmt.Sprintf("Container %q must include \"ALL\" in capabilities.drop.", c.Name),
					Level:   "restricted",
				})
		}

		// Check that only NET_BIND_SERVICE is added (if any).
		for _, cap := range sc.Capabilities.Add {
			if string(cap) != "NET_BIND_SERVICE" {
				violations = append(violations, Violation{
						Field:   prefix + ".securityContext.capabilities.add",
						Message: fmt.Sprintf("Container %q adds capability %q. Only NET_BIND_SERVICE is allowed under Restricted level.", c.Name, cap),
						Level:   "restricted",
					})
			}
		}
	}

	// ── Restricted: seccompProfile ───────────────────────────────
	if sc.SeccompProfile == nil {
		// Only a violation if Pod-level seccomp is also not set
		// (caller should check Pod-level separately)
	} else if sc.SeccompProfile.Type != corev1.SeccompProfileTypeRuntimeDefault &&
	sc.SeccompProfile.Type != corev1.SeccompProfileTypeLocalhost {
		violations = append(violations, Violation{
				Field:   prefix + ".securityContext.seccompProfile.type",
				Message: fmt.Sprintf("Container %q seccompProfile.type must be RuntimeDefault or Localhost, got %q.", c.Name, sc.SeccompProfile.Type),
				Level:   "restricted",
			})
	}

	return violations
}

// getVolumeType returns a string identifying the type of a Volume.
// Kubernetes volumes are a union type — exactly one field in the
// VolumeSource should be set.
func getVolumeType(vol corev1.Volume) string {
	switch {
	case vol.ConfigMap != nil:
		return "configMap"
	case vol.Secret != nil:
		return "secret"
	case vol.EmptyDir != nil:
		return "emptyDir"
	case vol.PersistentVolumeClaim != nil:
		return "persistentVolumeClaim"
	case vol.HostPath != nil:
		return "hostPath"
	case vol.Projected != nil:
		return "projected"
	case vol.DownwardAPI != nil:
		return "downwardAPI"
	case vol.CSI != nil:
		return "csi"
	case vol.Ephemeral != nil:
		return "ephemeral"
	default:
		return "unknown"
	}
}

// FormatViolations returns a human-readable, multi-line string
// summarizing all violations. Suitable for printing to the console
// or including in webhook denial messages.
//
// Parameters:
//   - violations: The slice of Violations to format.
//
// Returns a formatted string with one line per violation, grouped
// by PSS level (baseline violations first, then restricted).
func FormatViolations(violations []Violation) string {
	if len(violations) == 0 {
		return "✓ Pod is compliant with Restricted PSS level"
	}

	var sb strings.Builder
	sb.WriteString(fmt.Sprintf("✗ %d violation(s) found:\n", len(violations)))

	// Group by level for clearer output.
	var baseline, restricted []Violation
	for _, v := range violations {
		if v.Level == "baseline" {
			baseline = append(baseline, v)
		} else {
			restricted = append(restricted, v)
		}
	}

	if len(baseline) > 0 {
		sb.WriteString("\n  Baseline violations:\n")
		for _, v := range baseline {
			sb.WriteString(fmt.Sprintf("    • [%s] %s\n", v.Field, v.Message))
		}
	}

	if len(restricted) > 0 {
		sb.WriteString("\n  Restricted violations:\n")
		for _, v := range restricted {
			sb.WriteString(fmt.Sprintf("    • [%s] %s\n", v.Field, v.Message))
		}
	}

	return sb.String()
}
```

---

## 28.10 Summary

| Layer                    | Tool                  | When It Acts          | What It Does                     |
|--------------------------|-----------------------|-----------------------|----------------------------------|
| Authorization            | RBAC (Ch 27)          | Every API request     | Controls who can create Pods     |
| Admission (built-in)     | PSA                   | Pod creation          | Enforces PSS levels              |
| Admission (webhook)      | Gatekeeper / Kyverno  | Resource creation     | Custom policies (labels, images) |
| Runtime                  | Falco                 | While Pod runs        | Detects unexpected behavior      |

**The security progression:**
1. **RBAC** — control who can deploy
2. **PSA / Gatekeeper / Kyverno** — control what they can deploy
3. **SecurityContext** — configure how containers run
4. **NetworkPolicy** — control network access
5. **Falco** — detect runtime anomalies

**The PSS levels:**
- **Privileged** — no restrictions (system workloads only)
- **Baseline** — blocks known privilege escalations (minimum for production)
- **Restricted** — maximum security (target for all application workloads)

> **Interview tip**: When asked about Pod security, explain the evolution
> (PSP → PSS/PSA), then describe the three levels. Show you understand
> SecurityContext by listing the key fields: `runAsNonRoot`, `readOnlyRootFilesystem`,
> `allowPrivilegeEscalation: false`, `capabilities.drop: ALL`, and
> `seccompProfile: RuntimeDefault`. Mention that PSA uses namespace labels
> and has three modes (enforce, audit, warn). Bonus points for mentioning
> Gatekeeper/Kyverno for custom policies and Falco for runtime detection.

---

*Next: Chapter 29 — Network Policies, where we control the network layer
to complete the defense-in-depth picture.*
