# Chapter 15: Pods — The Atomic Unit

> *"The Pod is to Kubernetes what the process is to Unix — the fundamental building
> block upon which everything else is composed."*

In the previous chapters, we explored how Linux namespaces, cgroups, and container
runtimes collaborate to isolate processes. We built containers from scratch, understood
the OCI runtime specification, and saw how containerd and runc work together to run
workloads. Now we ascend one level of abstraction: we enter the domain of Kubernetes,
and the first citizen we meet is the **Pod**.

This chapter is exhaustive. We will dissect every field in a PodSpec, explore
multi-container patterns, probe mechanisms, security contexts, disruption budgets,
topology constraints, and resource management. By the end, you will understand Pods
not as a YAML incantation, but as a carefully orchestrated collaboration of Linux
primitives you already know.

---

## 15.1 What Is a Pod?

### 15.1.1 The Smallest Deployable Unit

In Kubernetes, you never deploy a container directly. You deploy a **Pod**. A Pod is
the smallest, most basic deployable object. It represents a single instance of a
running process in your cluster — but that "process" may in fact be multiple tightly
coupled containers.

Think of a Pod as a **logical host**. On a traditional server, you might run an
application process alongside a logging daemon, a metrics exporter, and a local proxy
— all sharing the same network stack, the same filesystem mounts, and the same IPC
namespace. A Pod recreates exactly this environment, but within the Kubernetes
orchestration model.

### 15.1.2 Shared Namespaces

When Kubernetes creates a Pod, it instructs the container runtime to set up a group
of containers that share:

| Namespace | What It Means |
|-----------|---------------|
| **Network** | All containers in the Pod share the same IP address, port space, and network interfaces. Container A can reach container B on `localhost`. |
| **IPC** | Containers can communicate via System V IPC or POSIX message queues. |
| **PID** (optional) | With `shareProcessNamespace: true`, containers can see each other's processes. Process 1 in one container can send signals to processes in another. |
| **UTS** | All containers share the same hostname (the Pod name). |

They do **not** share:

- **Mount namespace** — each container has its own filesystem root, though they can
  share specific directories via volumes.
- **User namespace** — each container has its own UID/GID mapping (though user
  namespace support in Kubernetes is evolving).

### 15.1.3 Shared Storage via Volumes

Volumes are the mechanism through which containers in a Pod share data on disk.
A volume is declared at the Pod level and mounted into one or more containers at
specified paths. The lifecycle of a volume depends on its type:

- **emptyDir** — created when the Pod is assigned to a node, deleted when the Pod
  is removed. Perfect for scratch space and inter-container communication.
- **hostPath** — maps a file or directory from the host node's filesystem. Survives
  Pod restarts on the same node, but not rescheduling.
- **persistentVolumeClaim** — backed by a PersistentVolume in the cluster. Survives
  Pod deletion entirely.
- **configMap**, **secret**, **downwardAPI** — project cluster data into the Pod.

### 15.1.4 The Pod as a Logical Host

The mental model is simple: **a Pod is a virtual machine, and containers are processes
inside it.** They share the network stack just as processes on a VM share the NIC.
They share IPC just as processes on a VM share System V semaphores. They share
volumes just as processes on a VM share mounted filesystems.

```text
┌─────────────────────────────────────────────────┐
│                   POD (logical host)             │
│                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │ Container A  │  │ Container B  │  │ Init Ctr │ │
│  │ (app server) │  │ (log agent)  │  │ (setup)  │ │
│  └──────┬───────┘  └──────┬───────┘  └──────────┘ │
│         │                 │                        │
│  ═══════╪═════════════════╪═══════════════════     │
│         │   Shared Network Namespace  │            │
│         │   IP: 10.244.1.5            │            │
│  ═══════╪═════════════════╪═══════════════════     │
│         │                 │                        │
│  ┌──────┴─────────────────┴───────┐                │
│  │        Shared Volumes          │                │
│  │    /var/log  /config  /data    │                │
│  └────────────────────────────────┘                │
└─────────────────────────────────────────────────┘
```

---

## 15.2 Pod Spec Deep Dive

The PodSpec is the beating heart of nearly every workload object in Kubernetes.
Deployments, StatefulSets, DaemonSets, Jobs, and CronJobs all embed a PodSpec in
their `.spec.template.spec`. Understanding every field here gives you leverage
across the entire API.

### 15.2.1 containers[]

The `containers` array is required and must have at least one entry. Each entry
defines a container to run:

```yaml
containers:
- name: app                      # Unique name within the Pod
  image: myregistry/myapp:v1.2   # Container image reference
  imagePullPolicy: IfNotPresent  # Always | Never | IfNotPresent
  command: ["/app/server"]       # Overrides ENTRYPOINT
  args: ["--port=8080"]          # Overrides CMD
  workingDir: /app               # Working directory
  ports:
  - name: http
    containerPort: 8080          # Informational — does not open ports
    protocol: TCP                # TCP | UDP | SCTP
  env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: url
  envFrom:
  - configMapRef:
      name: app-config
  resources:
    requests:
      cpu: "250m"
      memory: "128Mi"
    limits:
      cpu: "1"
      memory: "512Mi"
  volumeMounts:
  - name: data
    mountPath: /data
    readOnly: false
    subPath: app-data            # Mount a subdirectory of the volume
  livenessProbe: { ... }
  readinessProbe: { ... }
  startupProbe: { ... }
  lifecycle:
    postStart:
      exec:
        command: ["/bin/sh", "-c", "echo started"]
    preStop:
      httpGet:
        path: /shutdown
        port: 8080
  securityContext: { ... }       # Container-level security context
  stdin: false
  stdinOnce: false
  tty: false
  terminationMessagePath: /dev/termination-log
  terminationMessagePolicy: File  # File | FallbackToLogsOnError
```

**Key considerations:**

- `command` and `args` override the image's `ENTRYPOINT` and `CMD` respectively.
  If you specify `command` but not `args`, the image's `CMD` is ignored entirely.
- `imagePullPolicy: Always` forces a pull every time. Use for `:latest` tags.
  `IfNotPresent` is the default for tagged images. `Never` uses only local images.
- `ports` is purely informational for documentation and service discovery. Containers
  listen on whatever ports they choose regardless of this field.

### 15.2.2 initContainers[]

Init containers run **sequentially** before any regular container starts. Each must
complete successfully (exit code 0) before the next one begins. If any init container
fails, the Pod restarts according to its `restartPolicy`.

```yaml
initContainers:
- name: wait-for-db
  image: busybox:1.36
  command: ['sh', '-c', 'until nc -z db-service 5432; do sleep 2; done']
- name: migrate
  image: myapp/migrate:v1.2
  command: ['./migrate', '--direction=up']
```

Use cases:
- Wait for dependent services to become available
- Run database migrations before the app starts
- Clone a git repository or download configuration files
- Set up file permissions on shared volumes

Init containers have the same spec as regular containers but do not support
`livenessProbe`, `readinessProbe`, or `startupProbe` — they are expected to run
to completion.

### 15.2.3 ephemeralContainers[]

Ephemeral containers are a debugging feature. They cannot be specified at Pod
creation time but can be injected into a running Pod via the API (or
`kubectl debug`). They are never restarted and have no probes or ports.

### 15.2.4 volumes[]

Volumes are declared at the Pod level and referenced by containers via `volumeMounts`:

```yaml
volumes:
- name: shared-data
  emptyDir:
    medium: ""          # "" (disk) or "Memory" (tmpfs)
    sizeLimit: "1Gi"    # For Memory medium, counts against container memory limit
- name: config
  configMap:
    name: app-config
    items:              # Selective key projection
    - key: config.yaml
      path: config.yaml
    defaultMode: 0644
- name: secrets
  secret:
    secretName: app-secrets
    defaultMode: 0400
- name: pod-info
  downwardAPI:
    items:
    - path: "labels"
      fieldRef:
        fieldPath: metadata.labels
    - path: "cpu-limit"
      resourceFieldRef:
        containerName: app
        resource: limits.cpu
- name: persistent
  persistentVolumeClaim:
    claimName: my-pvc
    readOnly: false
- name: host-log
  hostPath:
    path: /var/log
    type: Directory      # DirectoryOrCreate | File | FileOrCreate | Socket | CharDevice | BlockDevice
- name: projected
  projected:
    sources:
    - serviceAccountToken:
        path: token
        expirationSeconds: 3600
        audience: my-api
    - configMap:
        name: app-config
    - secret:
        name: app-secrets
```

### 15.2.5 nodeSelector

The simplest form of node constraint. The Pod will only be scheduled on nodes
whose labels match **all** entries:

```yaml
nodeSelector:
  disktype: ssd
  kubernetes.io/arch: amd64
```

### 15.2.6 affinity

Affinity rules provide far more expressive node and pod placement:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: topology.kubernetes.io/zone
          operator: In
          values: [us-east-1a, us-east-1b]
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      preference:
        matchExpressions:
        - key: node-type
          operator: In
          values: [high-memory]
  podAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
    - labelSelector:
        matchExpressions:
        - key: app
          operator: In
          values: [cache]
      topologyKey: kubernetes.io/hostname
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: web
        topologyKey: topology.kubernetes.io/zone
```

- **nodeAffinity** — constrain to nodes with specific labels.
  - `required...` — hard constraint, Pod won't schedule without it.
  - `preferred...` — soft constraint, scheduler tries but doesn't guarantee.
- **podAffinity** — schedule near Pods with specific labels.
- **podAntiAffinity** — schedule away from Pods with specific labels.

### 15.2.7 tolerations

Tolerations allow Pods to schedule onto nodes with matching taints:

```yaml
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "gpu-workload"
  effect: "NoSchedule"
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300     # Tolerate for 5 minutes before eviction
```

The `operator` is either `Equal` (key and value must match) or `Exists` (only key
must match — value is ignored). The `effect` can be `NoSchedule`, `PreferNoSchedule`,
or `NoExecute`.

### 15.2.8 serviceAccountName

Every Pod runs as a service account. If not specified, it uses the `default` service
account in the namespace. The service account controls what API permissions the Pod
has via RBAC bindings:

```yaml
serviceAccountName: my-app-sa
automountServiceAccountToken: false  # Opt out of mounting the SA token
```

### 15.2.9 securityContext (Pod-level)

Pod-level security context applies to all containers and init containers:

```yaml
securityContext:
  runAsUser: 1000
  runAsGroup: 3000
  runAsNonRoot: true
  fsGroup: 2000          # All volumes are owned by this group
  fsGroupChangePolicy: OnRootMismatch  # Always | OnRootMismatch
  supplementalGroups: [4000, 5000]
  sysctls:
  - name: net.core.somaxconn
    value: "1024"
  seccompProfile:
    type: RuntimeDefault   # Unconfined | RuntimeDefault | Localhost
  seLinuxOptions:
    level: "s0:c123,c456"
```

### 15.2.10 dnsPolicy and dnsConfig

```yaml
dnsPolicy: ClusterFirst   # Default | ClusterFirst | ClusterFirstWithHostNet | None
dnsConfig:
  nameservers:
  - 8.8.8.8
  searches:
  - my-namespace.svc.cluster.local
  options:
  - name: ndots
    value: "5"
```

- `ClusterFirst` — use cluster DNS (CoreDNS), fall back to upstream.
- `Default` — inherit the node's DNS configuration.
- `None` — ignore all and use only `dnsConfig`.

### 15.2.11 restartPolicy

```yaml
restartPolicy: Always  # Always | OnFailure | Never
```

- `Always` — default for long-running services (used in Deployments).
- `OnFailure` — restart only on non-zero exit code (used in Jobs).
- `Never` — don't restart (used for one-shot debugging pods).

### 15.2.12 terminationGracePeriodSeconds

```yaml
terminationGracePeriodSeconds: 60  # Default: 30
```

When a Pod is told to stop, Kubernetes:
1. Sends `SIGTERM` to PID 1 in each container.
2. Runs any `preStop` hooks in parallel.
3. Waits up to `terminationGracePeriodSeconds`.
4. Sends `SIGKILL` if the container is still running.

Set this high enough for your application to drain connections, flush buffers, and
shut down cleanly. For HTTP servers, 30–60 seconds is typical. For batch processors
with large in-memory state, you may need 300+ seconds.

### 15.2.13 hostNetwork, hostPID, hostIPC

```yaml
hostNetwork: true   # Use the host's network namespace (Pod gets host IP)
hostPID: true       # See all processes on the host
hostIPC: true       # Share host IPC namespace
```

These are **privileged operations** and should be used sparingly. Typical use cases:
- `hostNetwork` — CNI plugins, kube-proxy, monitoring agents that need to bind
  to the host's network interfaces.
- `hostPID` — debugging tools, process monitors.
- `hostIPC` — legacy applications that use shared memory with host processes.

### 15.2.14 priorityClassName and priority

```yaml
priorityClassName: system-cluster-critical
# or
priority: 1000000
```

Higher-priority Pods can preempt lower-priority ones when cluster resources are
exhausted.

### 15.2.15 Hands-on: Fully Annotated Pod YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: comprehensive-pod
  namespace: default
  labels:
    app: myapp
    version: v1.2.0
    tier: backend
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
spec:
  # --- Service Account ---
  serviceAccountName: myapp-sa
  automountServiceAccountToken: true

  # --- Pod-level Security ---
  securityContext:
    runAsUser: 1000
    runAsGroup: 3000
    runAsNonRoot: true
    fsGroup: 2000
    seccompProfile:
      type: RuntimeDefault

  # --- Scheduling ---
  nodeSelector:
    kubernetes.io/arch: amd64
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: myapp
          topologyKey: topology.kubernetes.io/zone
  tolerations:
  - key: "workload-type"
    operator: "Equal"
    value: "backend"
    effect: "NoSchedule"

  # --- DNS ---
  dnsPolicy: ClusterFirst
  dnsConfig:
    options:
    - name: ndots
      value: "2"

  # --- Lifecycle ---
  restartPolicy: Always
  terminationGracePeriodSeconds: 60

  # --- Init Containers (run sequentially before main containers) ---
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command: ['sh', '-c', 'until nc -z postgres-svc 5432; do echo waiting; sleep 2; done']
    resources:
      requests:
        cpu: 50m
        memory: 32Mi
      limits:
        cpu: 100m
        memory: 64Mi

  - name: run-migrations
    image: myregistry/myapp-migrate:v1.2.0
    command: ['./migrate', 'up']
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: url
    volumeMounts:
    - name: migration-scripts
      mountPath: /migrations
      readOnly: true
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 256Mi

  # --- Main Containers ---
  containers:
  - name: app
    image: myregistry/myapp:v1.2.0
    imagePullPolicy: IfNotPresent
    command: ["/app/server"]
    args: ["--config=/config/app.yaml"]
    workingDir: /app

    ports:
    - name: http
      containerPort: 8080
      protocol: TCP
    - name: metrics
      containerPort: 9090
      protocol: TCP

    env:
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name
    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: url

    envFrom:
    - configMapRef:
        name: app-environment

    resources:
      requests:
        cpu: "250m"
        memory: "256Mi"
      limits:
        cpu: "1"
        memory: "512Mi"

    volumeMounts:
    - name: config
      mountPath: /config
      readOnly: true
    - name: data
      mountPath: /data
    - name: tmp
      mountPath: /tmp

    # --- Probes ---
    startupProbe:
      httpGet:
        path: /healthz/startup
        port: http
      failureThreshold: 30
      periodSeconds: 2

    livenessProbe:
      httpGet:
        path: /healthz/live
        port: http
      initialDelaySeconds: 0
      periodSeconds: 10
      timeoutSeconds: 3
      failureThreshold: 3

    readinessProbe:
      httpGet:
        path: /healthz/ready
        port: http
      initialDelaySeconds: 0
      periodSeconds: 5
      timeoutSeconds: 2
      failureThreshold: 2
      successThreshold: 1

    # --- Lifecycle Hooks ---
    lifecycle:
      preStop:
        httpGet:
          path: /shutdown
          port: http

    # --- Container-level Security ---
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]

  # --- Sidecar: Log Forwarder ---
  - name: log-forwarder
    image: fluent/fluent-bit:2.2
    volumeMounts:
    - name: data
      mountPath: /data
      readOnly: true
    - name: fluentbit-config
      mountPath: /fluent-bit/etc
      readOnly: true
    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 128Mi
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]

  # --- Volumes ---
  volumes:
  - name: config
    configMap:
      name: app-config
  - name: data
    emptyDir: {}
  - name: tmp
    emptyDir:
      medium: Memory
      sizeLimit: 100Mi
  - name: migration-scripts
    configMap:
      name: migration-files
  - name: fluentbit-config
    configMap:
      name: fluentbit-config

  # --- Image Pull Secrets ---
  imagePullSecrets:
  - name: registry-credentials
```

---

## 15.3 Multi-Container Pod Patterns

### 15.3.1 The Sidecar Pattern

A sidecar container augments the main application container with supplementary
functionality. The sidecar runs alongside the main container for the entire lifetime
of the Pod.

```
┌────────────────────────────────────────────────┐
│                     POD                         │
│                                                 │
│  ┌──────────────────┐  ┌─────────────────────┐  │
│  │   Main Container  │  │   Sidecar Container │  │
│  │                   │  │                     │  │
│  │   Application     │──│   Logging Agent     │  │
│  │   writes to       │  │   reads from        │  │
│  │   /var/log/app    │  │   /var/log/app      │  │
│  └──────────────────┘  └─────────────────────┘  │
│           │                      │               │
│     ┌─────┴──────────────────────┴─────┐         │
│     │     Shared Volume: /var/log/app  │         │
│     └──────────────────────────────────┘         │
└────────────────────────────────────────────────┘
```

**Real-world scenarios:**

1. **Logging agent** — Fluent Bit or Filebeat reads logs from a shared volume and
   ships them to Elasticsearch or a cloud logging service.
2. **Service mesh proxy** — Envoy or Linkerd proxy intercepts all network traffic
   for mTLS, observability, and traffic management.
3. **Config watcher** — A container watches a ConfigMap-backed volume for changes
   and signals the main container to reload.
4. **TLS certificate manager** — A container that periodically renews TLS
   certificates and places them in a shared volume.

**Example: Application with logging sidecar**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-logging
spec:
  containers:
  - name: app
    image: myapp:v1.0
    command: ["/app/server"]
    volumeMounts:
    - name: logs
      mountPath: /var/log/app

  - name: log-shipper
    image: fluent/fluent-bit:2.2
    volumeMounts:
    - name: logs
      mountPath: /var/log/app
      readOnly: true
    - name: fluentbit-config
      mountPath: /fluent-bit/etc

  volumes:
  - name: logs
    emptyDir: {}
  - name: fluentbit-config
    configMap:
      name: fluentbit-cfg
```

**Example: Envoy sidecar for mTLS**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-envoy
spec:
  containers:
  - name: app
    image: myapp:v1.0
    # App listens on localhost:8080 — only reachable within the Pod
    env:
    - name: LISTEN_ADDR
      value: "127.0.0.1:8080"

  - name: envoy
    image: envoyproxy/envoy:v1.28
    ports:
    - containerPort: 8443   # External TLS port
    volumeMounts:
    - name: envoy-config
      mountPath: /etc/envoy
    - name: tls-certs
      mountPath: /etc/tls
      readOnly: true

  volumes:
  - name: envoy-config
    configMap:
      name: envoy-config
  - name: tls-certs
    secret:
      secretName: app-tls
```

### 15.3.2 The Ambassador Pattern

An ambassador container proxies network connections from the main container to
external services. The main container connects to `localhost`, and the ambassador
handles service discovery, connection pooling, or protocol translation.

```
┌──────────────────────────────────────────────────────┐
│                        POD                            │
│                                                       │
│  ┌──────────────┐       ┌───────────────┐             │
│  │ Main App     │──────▶│ Ambassador    │────────────▶ External DB
│  │              │       │ (pgbouncer)   │    Cloud     │
│  │ connects to  │       │               │    SQL       │
│  │ localhost    │       │ Connection    │             │
│  │ :5432        │       │ pooling &     │             │
│  └──────────────┘       │ TLS termination│            │
│                         └───────────────┘             │
└──────────────────────────────────────────────────────┘
```

**Example: PgBouncer ambassador for database connection pooling**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ambassador-pgbouncer
spec:
  containers:
  - name: app
    image: myapp:v1.0
    env:
    # Application connects to localhost — ambassador handles the rest
    - name: DATABASE_URL
      value: "postgres://user:pass@localhost:5432/mydb"

  - name: pgbouncer
    image: edoburu/pgbouncer:1.21
    ports:
    - containerPort: 5432
    env:
    - name: DATABASE_URL
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: url
    - name: POOL_MODE
      value: transaction
    - name: MAX_CLIENT_CONN
      value: "100"
    volumeMounts:
    - name: pgbouncer-config
      mountPath: /etc/pgbouncer

  volumes:
  - name: pgbouncer-config
    configMap:
      name: pgbouncer-cfg
```

### 15.3.3 The Adapter Pattern

An adapter container transforms or normalizes output from the main container into
a format expected by external consumers — most commonly, transforming application
metrics into a standard format like Prometheus exposition format.

```
                    ┌──────────────────────────┐
                    │           POD             │
                    │                           │
  Prometheus ◀──────│─ ┌───────────────────┐   │
  scrapes           │  │  Adapter           │   │
  :9090/metrics     │  │  (prom exporter)   │   │
                    │  │  reads /metrics    │   │
                    │  └────────┬───────────┘   │
                    │           │ shared vol     │
                    │  ┌────────┴───────────┐   │
                    │  │  Main App          │   │
                    │  │  writes metrics in │   │
                    │  │  custom format     │   │
                    │  └───────────────────┘   │
                    └──────────────────────────┘
```

**Example: Redis exporter adapter**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: adapter-redis
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9121"
spec:
  containers:
  - name: redis
    image: redis:7.2
    ports:
    - containerPort: 6379

  - name: redis-exporter
    image: oliver006/redis_exporter:v1.55
    ports:
    - containerPort: 9121
    env:
    - name: REDIS_ADDR
      value: "localhost:6379"
```

### 15.3.4 The Init Container Pattern

Init containers run before the main containers start. They are used for
one-time setup tasks. We covered the spec in §15.2.2; here are detailed
real-world examples.

**Example: Git clone and permission setup**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-git-clone
spec:
  initContainers:
  - name: git-clone
    image: alpine/git:2.40
    command:
    - git
    - clone
    - --depth=1
    - https://github.com/myorg/config-repo.git
    - /config
    volumeMounts:
    - name: config
      mountPath: /config

  - name: set-permissions
    image: busybox:1.36
    command: ['sh', '-c', 'chown -R 1000:3000 /config && chmod -R 0550 /config']
    volumeMounts:
    - name: config
      mountPath: /config

  containers:
  - name: app
    image: myapp:v1.0
    volumeMounts:
    - name: config
      mountPath: /config
      readOnly: true

  volumes:
  - name: config
    emptyDir: {}
```

**Example: Schema migration before app start**

```yaml
initContainers:
- name: migrate
  image: myapp/migrate:v1.2
  command: ['./migrate', '--source=file:///migrations', '--database=$(DATABASE_URL)', 'up']
  env:
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: db-creds
        key: url
  volumeMounts:
  - name: migrations
    mountPath: /migrations
```

---

## 15.4 Pod Networking in Detail

### 15.4.1 The Shared Network Namespace

When the kubelet creates a Pod, the container runtime first creates the **pause
container** (also called the infrastructure container or sandbox container). This
container does almost nothing — it calls `pause()` in a loop — but it owns the
network namespace. All other containers in the Pod join this namespace.

```
┌─────────────────────────────────────────────┐
│ Node                                        │
│                                             │
│  ┌────────────────────────────────────────┐  │
│  │ Pod: my-app (IP: 10.244.1.5)          │  │
│  │                                        │  │
│  │   pause container (owns net ns)        │  │
│  │      │                                 │  │
│  │   ┌──┴───┐  ┌──────┐  ┌──────┐        │  │
│  │   │ eth0 │  │ lo   │  │ veth │        │  │
│  │   └──┬───┘  └──────┘  └──┬───┘        │  │
│  │      │                    │            │  │
│  │   Container A          Container B     │  │
│  │   (port 8080)          (port 9090)     │  │
│  │   sees eth0            sees eth0       │  │
│  │   sees lo              sees lo         │  │
│  └──────┬─────────────────────────────────┘  │
│         │                                    │
│   ┌─────┴──────┐                             │
│   │ veth pair  │                             │
│   └─────┬──────┘                             │
│         │                                    │
│   ┌─────┴──────┐                             │
│   │  cbr0/cni0 │  (bridge)                   │
│   └────────────┘                             │
└─────────────────────────────────────────────┘
```

### 15.4.2 localhost Communication

Because all containers share the network namespace, they communicate over
`localhost`:

- Container A listens on `:8080`, Container B connects to `localhost:8080`.
- There is no network hop, no NAT, no overlay. It is literally the same
  loopback interface.
- This means containers in the same Pod **cannot bind to the same port**.

### 15.4.3 Pod IP Assignment

Each Pod receives a unique IP address from the cluster's Pod CIDR range. The CNI
plugin (Calico, Cilium, Flannel, AWS VPC CNI, etc.) is responsible for:

1. Allocating an IP from the node's assigned Pod CIDR subnet.
2. Creating a veth pair — one end in the Pod's network namespace, one on the host.
3. Configuring routes so that other Pods can reach this IP.

The Pod IP is routable within the cluster. Every Pod can reach every other Pod by IP
without NAT (this is the Kubernetes networking model).

### 15.4.4 Hands-on: Demonstrating Inter-Container Communication

```yaml
# pod-networking-demo.yaml
apiVersion: v1
kind: Pod
metadata:
  name: networking-demo
spec:
  containers:
  # Server container: listens on port 8080
  - name: server
    image: hashicorp/http-echo:0.2.3
    args: ["-listen=:8080", "-text=Hello from server container"]
    ports:
    - containerPort: 8080

  # Client container: curls localhost:8080 every 5 seconds
  - name: client
    image: curlimages/curl:8.4.0
    command:
    - sh
    - -c
    - |
      while true; do
        echo "--- $(date) ---"
        curl -s http://localhost:8080
        echo
        sleep 5
      done
```

Apply and observe:

```bash
kubectl apply -f pod-networking-demo.yaml
kubectl logs networking-demo -c client -f
# Output:
# --- Mon Jan 15 10:30:00 UTC 2024 ---
# Hello from server container
# --- Mon Jan 15 10:30:05 UTC 2024 ---
# Hello from server container
```

This proves that `localhost` communication between containers works seamlessly.

---

## 15.5 Resource Management

### 15.5.1 Requests and Limits

Every container can specify **requests** (the guaranteed minimum) and **limits**
(the enforced maximum) for CPU and memory:

```yaml
resources:
  requests:
    cpu: "250m"      # 250 millicores = 0.25 CPU
    memory: "128Mi"  # 128 mebibytes
  limits:
    cpu: "1"         # 1 full CPU core
    memory: "512Mi"  # 512 mebibytes
```

**How they work:**

| Resource | Request | Limit |
|----------|---------|-------|
| **CPU** | Used by scheduler for placement. The container is guaranteed this much CPU via CFS shares. | Enforced via CFS quota. Container is throttled (not killed) when exceeding. |
| **Memory** | Used by scheduler for placement. | Enforced by the OOM killer. If a container exceeds its memory limit, it is killed with OOMKilled. |

**QoS classes** are derived from requests and limits:

| QoS Class | Condition |
|-----------|-----------|
| **Guaranteed** | Every container has requests == limits for both CPU and memory. |
| **Burstable** | At least one container has a request or limit set, but not all are equal. |
| **BestEffort** | No container has any request or limit set. |

When the node is under memory pressure, BestEffort Pods are evicted first, then
Burstable, then Guaranteed.

### 15.5.2 Ephemeral Storage

```yaml
resources:
  requests:
    ephemeral-storage: "1Gi"
  limits:
    ephemeral-storage: "2Gi"
```

Ephemeral storage covers the container's writable layer, log files (`/var/log`),
and `emptyDir` volumes (non-memory medium). If a container exceeds its ephemeral
storage limit, it is evicted.

### 15.5.3 Extended Resources

Extended resources allow you to request non-standard resources like GPUs:

```yaml
resources:
  requests:
    nvidia.com/gpu: 1
  limits:
    nvidia.com/gpu: 1
```

Extended resources must always have requests == limits (they are not overcommittable).

### 15.5.4 LimitRange — Default Limits for a Namespace

A LimitRange sets default, minimum, and maximum resource constraints for Pods and
containers in a namespace:

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: production
spec:
  limits:
  - type: Container
    default:          # Default limits (applied if none specified)
      cpu: "500m"
      memory: "256Mi"
    defaultRequest:   # Default requests (applied if none specified)
      cpu: "100m"
      memory: "128Mi"
    max:              # Maximum allowed
      cpu: "4"
      memory: "8Gi"
    min:              # Minimum allowed
      cpu: "50m"
      memory: "64Mi"
  - type: Pod
    max:
      cpu: "8"
      memory: "16Gi"
  - type: PersistentVolumeClaim
    max:
      storage: "50Gi"
    min:
      storage: "1Gi"
```

### 15.5.5 ResourceQuota — Namespace-Level Quotas

A ResourceQuota limits the total resource consumption in a namespace:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-alpha
spec:
  hard:
    requests.cpu: "20"
    requests.memory: "40Gi"
    limits.cpu: "40"
    limits.memory: "80Gi"
    pods: "50"
    services: "20"
    persistentvolumeclaims: "10"
    requests.storage: "100Gi"
    configmaps: "50"
    secrets: "50"
    count/deployments.apps: "20"
```

### 15.5.6 Hands-on: Setting Up LimitRange and ResourceQuota

```bash
# Create namespace
kubectl create namespace quota-demo

# Apply LimitRange
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: quota-demo
spec:
  limits:
  - type: Container
    default:
      cpu: 500m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: "2"
      memory: 1Gi
    min:
      cpu: 50m
      memory: 64Mi
EOF

# Apply ResourceQuota
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: quota-demo
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "10"
EOF

# Create a Pod without specifying resources — LimitRange defaults apply
kubectl run test-pod --image=nginx:1.25 -n quota-demo

# Inspect the Pod to see default resources applied
kubectl get pod test-pod -n quota-demo -o jsonpath='{.spec.containers[0].resources}'
# Output: {"limits":{"cpu":"500m","memory":"256Mi"},"requests":{"cpu":"100m","memory":"128Mi"}}

# Check quota usage
kubectl describe resourcequota team-quota -n quota-demo
```

### 15.5.7 Hands-on: Go Program That Monitors Resource Usage

```go
// file: resource_monitor.go
//
// Package main implements a Kubernetes resource usage monitor.
//
// This program connects to the Kubernetes Metrics API (provided by
	// metrics-server) and reports CPU and memory usage for every Pod in
// a given namespace. It compares actual usage against the resource
// requests and limits defined in the PodSpec, highlighting containers
// that are close to their limits.
//
// Usage:
//
//	go run resource_monitor.go --namespace=default --interval=10s
package main

import (
	"context"
	"flag"
	"fmt"
	"os"
	"text/tabwriter"
	"time"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	metricsv1beta1 "k8s.io/metrics/pkg/client/clientset/versioned"
)

func main() {
	// namespace is the Kubernetes namespace to monitor.
	namespace := flag.String("namespace", "default", "Kubernetes namespace to monitor")

	// interval is the polling interval between successive metrics checks.
	interval := flag.Duration("interval", 10*time.Second, "Polling interval")

	// kubeconfig is the path to the kubeconfig file. If empty, in-cluster
	// configuration is attempted.
	kubeconfig := flag.String("kubeconfig", os.Getenv("KUBECONFIG"), "Path to kubeconfig file")

	flag.Parse()

	// Build the REST configuration from the kubeconfig file.
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error building kubeconfig: %v\n", err)
		os.Exit(1)
	}

	// Create the standard Kubernetes clientset for reading PodSpecs.
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error creating clientset: %v\n", err)
		os.Exit(1)
	}

	// Create the metrics clientset for reading Pod metrics from metrics-server.
	metricsClient, err := metricsv1beta1.NewForConfig(config)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error creating metrics client: %v\n", err)
		os.Exit(1)
	}

	// Polling loop: every `interval`, fetch pods and their metrics.
	ctx := context.Background()
	for {
		// Fetch all Pods in the target namespace.
		pods, err := clientset.CoreV1().Pods(*namespace).List(ctx, metav1.ListOptions{})
		if err != nil {
			fmt.Fprintf(os.Stderr, "Error listing pods: %v\n", err)
			time.Sleep(*interval)
			continue
		}

		// Fetch current metrics for all Pods in the namespace.
		podMetrics, err := metricsClient.MetricsV1beta1().PodMetricses(*namespace).List(ctx, metav1.ListOptions{})
		if err != nil {
			fmt.Fprintf(os.Stderr, "Error listing pod metrics: %v\n", err)
			time.Sleep(*interval)
			continue
		}

		// Build a lookup map: podName -> containerName -> usage.
		type containerUsage struct {
			cpuMillis int64
			memBytes  int64
		}
		usageMap := make(map[string]map[string]containerUsage)
		for _, pm := range podMetrics.Items {
			containers := make(map[string]containerUsage)
			for _, c := range pm.Containers {
				containers[c.Name] = containerUsage{
					cpuMillis: c.Usage.Cpu().MilliValue(),
					memBytes:  c.Usage.Memory().Value(),
				}
			}
			usageMap[pm.Name] = containers
		}

		// Print a formatted table of resource usage vs limits.
		fmt.Printf("\n=== Resource Usage Report: namespace=%s time=%s ===\n",
			*namespace, time.Now().Format(time.RFC3339))
		w := tabwriter.NewWriter(os.Stdout, 0, 4, 2, ' ', 0)
		fmt.Fprintln(w, "POD\tCONTAINER\tCPU_USED\tCPU_REQ\tCPU_LIM\tMEM_USED\tMEM_REQ\tMEM_LIM\tWARNING")

		for _, pod := range pods.Items {
			podContainers, ok := usageMap[pod.Name]
			if !ok {
				continue // No metrics yet (Pod may be starting)
			}
			for _, c := range pod.Spec.Containers {
				usage, exists := podContainers[c.Name]
				if !exists {
					continue
				}
				// Extract requests and limits from the container spec.
				cpuReq := c.Resources.Requests.Cpu().MilliValue()
				cpuLim := c.Resources.Limits.Cpu().MilliValue()
				memReq := c.Resources.Requests.Memory().Value()
				memLim := c.Resources.Limits.Memory().Value()

				// Determine warning level based on usage vs limits.
				warning := ""
				if memLim > 0 && usage.memBytes > (memLim*90/100) {
					warning = "⚠ MEM >90% of limit"
				}
				if cpuLim > 0 && usage.cpuMillis > (cpuLim*90/100) {
					warning += " ⚠ CPU >90% of limit"
				}

				fmt.Fprintf(w, "%s\t%s\t%dm\t%dm\t%dm\t%dMi\t%dMi\t%dMi\t%s\n",
					pod.Name, c.Name,
					usage.cpuMillis, cpuReq, cpuLim,
					usage.memBytes/(1024*1024), memReq/(1024*1024), memLim/(1024*1024),
					warning)
			}
		}
		w.Flush()

		time.Sleep(*interval)
	}
}
```

---

## 15.6 Probes — Complete Guide

### 15.6.1 The Three Probe Types

Kubernetes provides three types of probes, each serving a distinct purpose:

| Probe | Question It Answers | What Happens on Failure |
|-------|---------------------|-------------------------|
| **Startup** | "Has the application finished starting up?" | Liveness and readiness probes are not started until startup succeeds. Restarts the container after `failureThreshold` failures. |
| **Liveness** | "Is the application still alive and functional?" | Kubernetes kills the container and restarts it. |
| **Readiness** | "Is the application ready to receive traffic?" | Pod is removed from all Service endpoints. No restart. |

The relationship between them:

```
Pod Created
    │
    ▼
┌──────────────┐     ┌──────────────┐     ┌───────────────┐
│ Startup Probe│────▶│Liveness Probe│     │Readiness Probe│
│ (if defined) │     │ (periodic)   │     │ (periodic)    │
│              │     │              │     │               │
│ Protects slow│     │ Detects hangs│     │ Controls      │
│ starting apps│     │ and deadlocks│     │ traffic flow  │
└──────────────┘     └──────────────┘     └───────────────┘
   succeeds ──────────── starts both probes
```

### 15.6.2 Probe Mechanisms

Each probe supports four mechanisms:

**HTTP GET probe:**
```yaml
httpGet:
  path: /healthz
  port: 8080
  httpHeaders:
  - name: X-Custom-Header
    value: healthcheck
  scheme: HTTP  # or HTTPS
```
Success: HTTP status 200–399. Failure: any other status or connection error.

**TCP Socket probe:**
```yaml
tcpSocket:
  port: 5432
```
Success: TCP connection established. Failure: connection refused or timeout.

**gRPC probe (Kubernetes 1.27+):**
```yaml
grpc:
  port: 50051
  service: my.health.v1.Health  # Optional: specific service to check
```
Uses the standard gRPC health checking protocol.

**Exec probe:**
```yaml
exec:
  command:
  - /bin/sh
  - -c
  - pg_isready -U postgres
```
Success: exit code 0. Failure: any non-zero exit code.

### 15.6.3 Tuning Parameters

Every probe accepts these tuning parameters:

```yaml
initialDelaySeconds: 0    # Seconds after container start before first probe
periodSeconds: 10         # How often to run the probe
timeoutSeconds: 1         # Seconds to wait for a response before failure
successThreshold: 1       # Consecutive successes required (readiness/startup only)
failureThreshold: 3       # Consecutive failures before action is taken
```

**Real-world tuning advice:**

1. **Always use a startup probe for slow-starting apps.** Without it, the liveness
   probe may kill your container before it finishes initializing.

2. **Don't set `initialDelaySeconds` on liveness probes.** Use a startup probe
   instead — it provides the same protection with more flexibility.

3. **Keep liveness probes simple.** Check only that your process is alive, not that
   all its dependencies are healthy. A liveness probe that checks the database will
   restart your app when the database is down — making things worse.

4. **Make readiness probes dependency-aware.** A readiness probe should check
   database connectivity, cache warmth, and other prerequisites. Failing readiness
   removes the Pod from traffic — exactly what you want when a dependency is down.

5. **Set `timeoutSeconds` appropriately.** If your health endpoint sometimes takes
   2 seconds under load, set `timeoutSeconds: 5` to avoid false failures.

6. **For startup probes, use `failureThreshold * periodSeconds` to define total
   startup time.** For example: `failureThreshold: 30, periodSeconds: 2` gives
   your app 60 seconds to start.

### 15.6.4 Hands-on: Go HTTP Server With All Three Probe Types

```go
// file: probe_server.go
//
// Package main implements an HTTP server that demonstrates all three
// Kubernetes probe types: startup, liveness, and readiness.
//
// The server simulates a slow startup, periodic health fluctuations,
// and dependency-based readiness checks. This provides a realistic
// model for understanding how each probe type interacts with the
// Kubernetes kubelet.
//
// Endpoints:
//   - /healthz/startup  — Returns 200 once initialization is complete.
//   - /healthz/live     — Returns 200 if the process is healthy.
//   - /healthz/ready    — Returns 200 if dependencies are available.
//   - /                  — Main application endpoint.
//   - /shutdown          — Graceful shutdown endpoint (for preStop hook).
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"sync"
	"sync/atomic"
	"syscall"
	"time"
)

// server holds the shared state for the application's health probes.
// It tracks whether the application has finished starting up, whether
// it is currently healthy (alive), and whether its dependencies are
// available (ready).
type server struct {
	// started indicates whether the application has finished its
	// initialization sequence. The startup probe checks this flag.
	started atomic.Bool

	// alive indicates whether the application process is healthy.
	// The liveness probe checks this flag.
	alive atomic.Bool

	// ready indicates whether the application is prepared to receive
	// traffic. The readiness probe checks this flag. This is separate
	// from liveness because an application can be alive but not yet
	// ready (e.g., waiting for a cache to warm up).
	ready atomic.Bool

	// shutdownCh is closed when the server receives a shutdown signal,
	// allowing in-flight requests to drain gracefully.
	shutdownCh chan struct{}

	// shutdownOnce ensures the shutdown channel is closed exactly once.
	shutdownOnce sync.Once
}

// newServer creates a new server instance with all probes initially
// indicating "not ready." The startup simulation will progress the
// server through its initialization phases.
func newServer() *server {
	return &server{
		shutdownCh: make(chan struct{}),
	}
}

// simulateStartup mimics a slow application startup. In a real app,
// this might involve loading configuration, establishing database
// connections, warming caches, or running initial data syncs.
func (s *server) simulateStartup() {
	log.Println("[startup] Beginning initialization...")
	time.Sleep(5 * time.Second)
	log.Println("[startup] Phase 1/3: Configuration loaded")

	time.Sleep(3 * time.Second)
	log.Println("[startup] Phase 2/3: Database connection established")

	time.Sleep(2 * time.Second)
	log.Println("[startup] Phase 3/3: Cache warmed")

	// Mark the application as started. The startup probe will now
	// return 200, allowing liveness and readiness probes to begin.
	s.started.Store(true)
	s.alive.Store(true)
	log.Println("[startup] Initialization complete — startup probe will pass")

	// After a brief delay, mark as ready to receive traffic.
	time.Sleep(1 * time.Second)
	s.ready.Store(true)
	log.Println("[startup] Ready for traffic — readiness probe will pass")
}

// handleStartupProbe responds to the Kubernetes startup probe.
// Returns 200 if initialization is complete, 503 otherwise.
func (s *server) handleStartupProbe(w http.ResponseWriter, r *http.Request) {
	if s.started.Load() {
		w.WriteHeader(http.StatusOK)
		fmt.Fprintln(w, `{"status":"started"}`)
	} else {
		w.WriteHeader(http.StatusServiceUnavailable)
		fmt.Fprintln(w, `{"status":"starting"}`)
	}
}

// handleLivenessProbe responds to the Kubernetes liveness probe.
// Returns 200 if the process is healthy, 503 otherwise. If this
// probe fails, Kubernetes restarts the container.
func (s *server) handleLivenessProbe(w http.ResponseWriter, r *http.Request) {
	if s.alive.Load() {
		w.WriteHeader(http.StatusOK)
		fmt.Fprintln(w, `{"status":"alive"}`)
	} else {
		w.WriteHeader(http.StatusServiceUnavailable)
		fmt.Fprintln(w, `{"status":"dead"}`)
	}
}

// handleReadinessProbe responds to the Kubernetes readiness probe.
// Returns 200 if the application is ready to receive traffic, 503
// otherwise. Unlike liveness failures, readiness failures only remove
// the Pod from Service endpoints — they don't trigger a restart.
func (s *server) handleReadinessProbe(w http.ResponseWriter, r *http.Request) {
	if s.ready.Load() {
		w.WriteHeader(http.StatusOK)
		fmt.Fprintln(w, `{"status":"ready"}`)
	} else {
		w.WriteHeader(http.StatusServiceUnavailable)
		fmt.Fprintln(w, `{"status":"not_ready"}`)
	}
}

// handleMain is the primary application endpoint. In a real service,
// this would serve business logic.
func (s *server) handleMain(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	fmt.Fprintln(w, `{"message":"Hello from the application"}`)
}

// handleShutdown initiates a graceful shutdown sequence. This is
// typically called by a Kubernetes preStop hook.
func (s *server) handleShutdown(w http.ResponseWriter, r *http.Request) {
	log.Println("[shutdown] Graceful shutdown requested via preStop hook")

	// Immediately mark as not ready so the readiness probe fails and
	// Kubernetes stops sending new traffic to this Pod.
	s.ready.Store(false)

	w.WriteHeader(http.StatusOK)
	fmt.Fprintln(w, `{"status":"shutting_down"}`)

	// Signal the main goroutine to begin shutdown.
	s.shutdownOnce.Do(func() {
			close(s.shutdownCh)
		})
}

func main() {
	srv := newServer()

	// Start the initialization sequence in a background goroutine.
	go srv.simulateStartup()

	mux := http.NewServeMux()
	mux.HandleFunc("/healthz/startup", srv.handleStartupProbe)
	mux.HandleFunc("/healthz/live", srv.handleLivenessProbe)
	mux.HandleFunc("/healthz/ready", srv.handleReadinessProbe)
	mux.HandleFunc("/", srv.handleMain)
	mux.HandleFunc("/shutdown", srv.handleShutdown)

	httpServer := &http.Server{
		Addr:         ":8080",
		Handler:      mux,
		ReadTimeout:  5 * time.Second,
		WriteTimeout: 10 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	// Listen for OS signals (SIGTERM from Kubernetes, SIGINT from Ctrl+C).
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)

	// Start the HTTP server in a background goroutine.
	go func() {
		log.Printf("[server] Listening on %s\n", httpServer.Addr)
		if err := httpServer.ListenAndServe(); err != http.ErrServerClosed {
			log.Fatalf("[server] ListenAndServe error: %v", err)
		}
	}()

	// Block until we receive a signal or the shutdown endpoint is hit.
	select {
	case sig := <-sigCh:
		log.Printf("[shutdown] Received signal: %v\n", sig)
	case <-srv.shutdownCh:
		log.Println("[shutdown] Shutdown triggered via HTTP endpoint")
	}

	// Mark as not ready and give Kubernetes time to update endpoints.
	srv.ready.Store(false)
	log.Println("[shutdown] Marked as not ready, waiting for endpoint propagation...")
	time.Sleep(5 * time.Second)

	// Gracefully shut down the HTTP server, allowing in-flight requests
	// to complete within a 30-second deadline.
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	if err := httpServer.Shutdown(ctx); err != nil {
		log.Printf("[shutdown] HTTP server shutdown error: %v\n", err)
	}
	log.Println("[shutdown] Server stopped cleanly")
}
```

### 15.6.5 Common Probe Misconfigurations and How to Fix Them

**Misconfiguration 1: Liveness probe checks external dependencies**

```yaml
# ❌ BAD: Liveness probe checks database
livenessProbe:
  httpGet:
    path: /healthz/deep  # Checks DB, Redis, etc.
    port: 8080
  failureThreshold: 3
```

When the database goes down, every Pod restarts. This cascading restart makes
recovery slower because now the application must re-initialize connections.

```yaml
# ✅ GOOD: Liveness probe checks only the process itself
livenessProbe:
  httpGet:
    path: /healthz/live  # Only checks: "Is the process responsive?"
    port: 8080
  failureThreshold: 3

# Readiness probe checks dependencies
readinessProbe:
  httpGet:
    path: /healthz/ready  # Checks DB, Redis, cache warmth
    port: 8080
  failureThreshold: 2
```

**Misconfiguration 2: No startup probe for slow-starting app**

```yaml
# ❌ BAD: App takes 60s to start, liveness kills it at 30s
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  failureThreshold: 3
  periodSeconds: 10
```

The app enters a restart loop: start → 30s delay → 3 failures × 10s = 60s → killed
at 60s, just as it was about to become ready.

```yaml
# ✅ GOOD: Startup probe protects slow-starting app
startupProbe:
  httpGet:
    path: /healthz/startup
    port: 8080
  failureThreshold: 30    # 30 × 2s = 60s total startup window
  periodSeconds: 2

livenessProbe:
  httpGet:
    path: /healthz/live
    port: 8080
  periodSeconds: 10
  failureThreshold: 3
```

**Misconfiguration 3: Timeout too short under load**

```yaml
# ❌ BAD: 1s timeout for an endpoint that takes 2s under load
readinessProbe:
  httpGet:
    path: /healthz/ready
    port: 8080
  timeoutSeconds: 1       # Default is 1 second
  periodSeconds: 5
```

Under load, the health endpoint may take 2 seconds. With a 1-second timeout, the
readiness probe fails, traffic is removed, load drops, probe passes, traffic returns,
load spikes, probe fails — an oscillation pattern.

```yaml
# ✅ GOOD: Generous timeout, separate lightweight health endpoint
readinessProbe:
  httpGet:
    path: /healthz/ready
    port: 8080
  timeoutSeconds: 5
  periodSeconds: 10
```

**Misconfiguration 4: successThreshold > 1 on liveness**

```yaml
# ❌ BAD: successThreshold > 1 is only valid for readiness probes
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  successThreshold: 3     # Ignored for liveness — always 1
```

For liveness and startup probes, `successThreshold` must be 1. Only readiness probes
support values greater than 1.

---

## 15.7 Pod Security Context

Security contexts configure the privilege and access control settings for a Pod or
container. In a production cluster, you should always run with the most restrictive
settings possible.

### 15.7.1 runAsUser, runAsGroup, runAsNonRoot

```yaml
securityContext:
  runAsUser: 1000        # UID to run the container process as
  runAsGroup: 3000       # Primary GID for the container process
  runAsNonRoot: true     # Kubelet validates UID != 0; rejects if it is
```

**Why this matters:** Running as root inside a container means a container escape
gives the attacker root on the host. Running as a non-root user limits the blast
radius of container breakouts.

### 15.7.2 fsGroup — Volume Ownership

```yaml
securityContext:
  fsGroup: 2000
  fsGroupChangePolicy: OnRootMismatch
```

When `fsGroup` is set:
1. All files in mounted volumes are chowned to the specified group.
2. New files created in volumes get this group.
3. The supplemental group is added to the container process.

`fsGroupChangePolicy`:
- `Always` — recursively chown every time the volume is mounted (slow for large volumes).
- `OnRootMismatch` — only chown if the root directory's group doesn't match (faster).

### 15.7.3 capabilities: add/drop

Linux capabilities provide fine-grained control over what a process can do.
Instead of running as root (all capabilities), you grant only the specific
capabilities needed:

```yaml
securityContext:
  capabilities:
    drop:
    - ALL                    # Start by dropping everything
    add:
    - NET_BIND_SERVICE       # Allow binding to ports < 1024
```

Common capabilities:

| Capability | What It Allows |
|------------|---------------|
| `NET_BIND_SERVICE` | Bind to privileged ports (< 1024) |
| `NET_RAW` | Use raw sockets (ping, packet capture) |
| `SYS_PTRACE` | Trace processes (for debugging sidecars) |
| `SYS_ADMIN` | Mount filesystems, configure network, etc. (very broad!) |
| `CHOWN` | Change file ownership |
| `DAC_OVERRIDE` | Bypass file permission checks |
| `SETUID/SETGID` | Change process UID/GID |

**Best practice:** Always `drop: ["ALL"]` and add back only what you need.

### 15.7.4 readOnlyRootFilesystem

```yaml
securityContext:
  readOnlyRootFilesystem: true
```

This prevents the container from writing to its own filesystem layer. Any writes
must go to explicitly mounted volumes. This defends against attacks that try to
modify the container's binaries or drop malware on disk.

You'll typically need to mount `emptyDir` volumes for `/tmp` and other writable
paths.

### 15.7.5 allowPrivilegeEscalation

```yaml
securityContext:
  allowPrivilegeEscalation: false
```

This sets the `no_new_privs` flag on the container process, preventing it from
gaining additional privileges via setuid/setgid binaries or filesystem capabilities.
Always set to `false` unless you have a specific reason not to.

### 15.7.6 seccompProfile

```yaml
securityContext:
  seccompProfile:
    type: RuntimeDefault  # Uses the container runtime's default seccomp profile
```

Seccomp (Secure Computing Mode) filters the system calls a process can make.
The `RuntimeDefault` profile blocks dangerous syscalls like `ptrace`, `reboot`,
and `mount` while allowing normal application behavior.

Options:
- `Unconfined` — no seccomp filtering (not recommended).
- `RuntimeDefault` — runtime's default profile (recommended baseline).
- `Localhost` — custom profile stored on the node at
  `/var/lib/kubelet/seccomp/<path>`.

### 15.7.7 seLinuxOptions and apparmorProfile

```yaml
securityContext:
  seLinuxOptions:
    level: "s0:c123,c456"
    role: "spc_t"
    type: "spc_t"
    user: "system_u"
  # Kubernetes 1.30+
  appArmorProfile:
    type: RuntimeDefault  # Unconfined | RuntimeDefault | Localhost
```

These provide Mandatory Access Control on top of Linux's standard Discretionary
Access Control. SELinux is common on RHEL-based systems; AppArmor on Ubuntu-based
systems.

### 15.7.8 Hands-on: Locked-Down Pod Security Context

```yaml
# secure-pod.yaml
#
# This Pod demonstrates a production-hardened security context.
# It follows the principle of least privilege: drop all capabilities,
# run as non-root, use a read-only filesystem, prevent privilege
# escalation, and apply the runtime's default seccomp profile.
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
  labels:
    app: secure-app
spec:
  # Pod-level security context.
  securityContext:
    runAsUser: 65534         # 'nobody' user
    runAsGroup: 65534        # 'nogroup' group
    runAsNonRoot: true       # Reject if image tries to run as root
    fsGroup: 65534           # Volume files owned by this group
    seccompProfile:
      type: RuntimeDefault   # Apply default seccomp profile

  serviceAccountName: secure-app-sa
  automountServiceAccountToken: false  # Don't mount the SA token

  containers:
  - name: app
    image: myregistry/myapp:v1.2.0
    ports:
    - containerPort: 8080

    # Container-level security context (overrides/extends Pod-level).
    securityContext:
      allowPrivilegeEscalation: false  # No setuid/setgid escalation
      readOnlyRootFilesystem: true     # Immutable container filesystem
      capabilities:
        drop: ["ALL"]                  # Drop every capability

    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 256Mi

    # Writable directories mounted as emptyDir.
    volumeMounts:
    - name: tmp
      mountPath: /tmp
    - name: cache
      mountPath: /var/cache/app

  volumes:
  - name: tmp
    emptyDir:
      sizeLimit: 50Mi
  - name: cache
    emptyDir:
      sizeLimit: 100Mi
```

---

## 15.8 Pod Disruption Budgets

### 15.8.1 What Is a PDB?

A **PodDisruptionBudget** (PDB) tells Kubernetes how many Pods from a set must remain
available during voluntary disruptions — node drains, cluster upgrades, or
autoscaler scale-downs. It does not protect against involuntary disruptions like
hardware failures or OOM kills.

### 15.8.2 minAvailable vs maxUnavailable

You specify **one** of:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  # Option A: At least 2 Pods must always be running.
  minAvailable: 2

  # Option B: At most 1 Pod may be down at any time.
  # maxUnavailable: 1

  # Percentage form is also supported:
  # minAvailable: "50%"
  # maxUnavailable: "25%"

  selector:
    matchLabels:
      app: myapp
```

| Field | Meaning |
|-------|---------|
| `minAvailable: 2` | At least 2 Pods must be running. If you have 3, only 1 can be disrupted. |
| `maxUnavailable: 1` | At most 1 Pod can be down. If you have 5, 4 must be running. |
| `minAvailable: "50%"` | At least 50% of matching Pods must be running. |

### 15.8.3 Voluntary vs Involuntary Disruptions

| Voluntary | Involuntary |
|-----------|-------------|
| `kubectl drain` | Node hardware failure |
| Cluster autoscaler removing a node | Kernel panic |
| Node OS upgrade | OOM kill by the kernel |
| Preemption by higher-priority Pod | |

PDBs protect against voluntary disruptions only. For involuntary disruptions, you
rely on the ReplicaSet/Deployment controller to create replacement Pods.

### 15.8.4 Hands-on: PDB Configuration and Testing

```bash
# Create a Deployment with 3 replicas
kubectl create deployment myapp --image=nginx:1.25 --replicas=3

# Create a PDB requiring at least 2 Pods available
cat <<EOF | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: myapp
EOF

# Check PDB status
kubectl get pdb myapp-pdb
# NAME        MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# myapp-pdb   2               N/A               1                     5s

# Drain a node — PDB will allow at most 1 Pod to be evicted at a time
kubectl drain node-1 --ignore-daemonsets --delete-emptydir-data

# If all 3 Pods are on node-1, the drain will evict 1, wait for it to
# be rescheduled and Running on another node, then evict the next one.
# This ensures minAvailable=2 is always respected.
```

---

## 15.9 Pod Topology Spread Constraints

### 15.9.1 Why Spread Pods?

Without topology spread constraints, the scheduler may place all your Pods in the
same zone or on the same node. If that zone has an outage or that node fails, your
entire service is down.

### 15.9.2 The TopologySpreadConstraint Spec

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: myapp
  matchLabelKeys:
  - pod-template-hash   # Automatically matches the owning ReplicaSet's Pods
```

| Field | Meaning |
|-------|---------|
| `maxSkew` | Maximum allowed difference in Pod count between any two topology domains. `1` means "spread as evenly as possible." |
| `topologyKey` | The node label that defines topology domains. Common values: `topology.kubernetes.io/zone`, `kubernetes.io/hostname`. |
| `whenUnsatisfiable` | `DoNotSchedule` (hard) or `ScheduleAnyway` (soft). |
| `labelSelector` | Identifies the group of Pods to spread. |
| `matchLabelKeys` | Additional label keys for matching (1.27+). |
| `minDomains` | Minimum number of topology domains required (1.27+). |

### 15.9.3 Hands-on: Topology Spread Examples

**Spread across zones:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zone-spread
spec:
  replicas: 6
  selector:
    matchLabels:
      app: zone-spread
  template:
    metadata:
      labels:
        app: zone-spread
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
        labelSelector:
          matchLabels:
            app: zone-spread
      containers:
      - name: app
        image: nginx:1.25
```

With 3 zones, this produces 2 Pods per zone:

```
Zone A: [pod-1] [pod-2]
Zone B: [pod-3] [pod-4]
Zone C: [pod-5] [pod-6]
```

**Spread across nodes within each zone:**

```yaml
topologySpreadConstraints:
# First: spread across zones
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: myapp
# Second: spread across nodes within each zone
- maxSkew: 1
  topologyKey: kubernetes.io/hostname
  whenUnsatisfiable: ScheduleAnyway
  labelSelector:
    matchLabels:
      app: myapp
```

---

## 15.10 Pod Overhead

### 15.10.1 What Is Pod Overhead?

Some container runtimes add overhead beyond the container's own resource usage.
For example, Kata Containers runs each Pod in a lightweight VM, consuming extra
memory for the VM's kernel and agent process. Pod Overhead lets the runtime declare
this cost so the scheduler accounts for it.

### 15.10.2 RuntimeClass and Pod Overhead

```yaml
apiVersion: node.k8s.io/v1
kind: RuntimeClass
metadata:
  name: kata
handler: kata-qemu
overhead:
  podFixed:
    cpu: "250m"
    memory: "160Mi"
scheduling:
  nodeSelector:
    kata-runtime: "true"
```

A Pod using this RuntimeClass:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kata-pod
spec:
  runtimeClassName: kata
  containers:
  - name: app
    image: myapp:v1.0
    resources:
      requests:
        cpu: "500m"
        memory: "256Mi"
```

The scheduler sees the total request as:
- CPU: 500m (container) + 250m (overhead) = **750m**
- Memory: 256Mi (container) + 160Mi (overhead) = **416Mi**

This ensures the node has enough capacity for both the container and the runtime's
overhead.

---

## 15.11 Pod Lifecycle

### 15.11.1 Pod Phases

A Pod's `status.phase` field can be one of five values:

```
                    ┌──────────┐
                    │ Pending  │ ◀── Pod accepted, waiting for scheduling
                    └────┬─────┘     or image pulls
                         │
                         ▼
                    ┌──────────┐
                    │ Running  │ ◀── At least one container is running
                    └────┬─────┘
                         │
                    ┌────┴────┐
                    ▼         ▼
             ┌───────────┐ ┌────────┐
             │ Succeeded │ │ Failed │
             └───────────┘ └────────┘
                  │              │
                  └──────┬───────┘
                         ▼
                    ┌──────────┐
                    │ Unknown  │ ◀── Node lost contact with control plane
                    └──────────┘
```

| Phase | Meaning |
|-------|---------|
| **Pending** | Pod is accepted by the API server but not yet scheduled, or images are being pulled. |
| **Running** | Pod is bound to a node and at least one container is running or starting. |
| **Succeeded** | All containers exited with code 0 and will not restart (`restartPolicy: Never/OnFailure`). |
| **Failed** | At least one container exited with a non-zero code and will not restart. |
| **Unknown** | Pod status cannot be determined, usually due to lost communication with the node. |

### 15.11.2 Pod Conditions

While `phase` is a coarse summary, `conditions` provide detailed status:

| Condition | Meaning |
|-----------|---------|
| `PodScheduled` | Pod has been assigned to a node. |
| `Initialized` | All init containers have completed successfully. |
| `ContainersReady` | All containers in the Pod have passed their readiness probes. |
| `Ready` | Pod is ready to serve traffic. Gates may delay this beyond `ContainersReady`. |

```bash
kubectl get pod myapp -o jsonpath='{range .status.conditions[*]}{.type}={.status}{"\n"}{end}'
# PodScheduled=True
# Initialized=True
# ContainersReady=True
# Ready=True
```

### 15.11.3 Container States

Each container within a Pod has its own state:

- **Waiting** — not yet running (pulling image, waiting for init containers, etc.).
  The `reason` field explains why (e.g., `ContainerCreating`, `CrashLoopBackOff`,
  `ImagePullBackOff`).
- **Running** — executing. The `startedAt` field records when it started.
- **Terminated** — finished execution. Includes `exitCode`, `reason`, `startedAt`,
  and `finishedAt`.

### 15.11.4 The Complete Lifecycle Sequence

```
1. Pod YAML submitted to API server
2. API server validates, persists to etcd
3. Scheduler watches for unscheduled Pods
4. Scheduler selects a node → sets spec.nodeName
5. Kubelet on the node watches for Pods assigned to it
6. Kubelet instructs CRI to create the Pod sandbox (pause container)
7. Init containers run sequentially
   a. kubelet → CRI: create + start init container 1
   b. Wait for exit code 0
   c. kubelet → CRI: create + start init container 2
   d. Wait for exit code 0
8. Main containers start in parallel
   a. kubelet → CRI: create + start each container
   b. postStart hooks execute
9. Probes begin
   a. Startup probe runs until success
   b. Then liveness + readiness probes run periodically
10. Pod is Ready when all containers pass readiness
11. Pod is added to Service endpoints
```

### 15.11.5 Hands-on: Observing Pod Lifecycle Transitions

```bash
# Watch Pod events in real time
kubectl get events --field-selector involvedObject.name=myapp --watch

# Watch Pod conditions change
kubectl get pod myapp -w

# Detailed lifecycle observation script
watch -n 1 'kubectl get pod myapp -o jsonpath="{.status.phase} | \
  Init:{.status.initContainerStatuses[*].ready} | \
  Ready:{.status.containerStatuses[*].ready} | \
  Conditions:{range .status.conditions[*]}{.type}={.status} {end}"'
```

---

## 15.12 Hands-on: Building and Deploying a Go Application as a Pod

### 15.12.1 The Go HTTP Server

```go
// file: main.go
//
// Package main implements a production-ready HTTP server designed
// for deployment as a Kubernetes Pod. It includes health check
// endpoints for all three probe types (startup, liveness, readiness),
// a graceful shutdown sequence that cooperates with Kubernetes
// terminationGracePeriodSeconds, and structured logging.
//
// This server demonstrates the practices described throughout
// this chapter:
//   - Non-root execution (the Dockerfile creates a dedicated user).
//   - Read-only root filesystem (writable dirs are emptyDir volumes).
//   - Proper signal handling (SIGTERM from Kubernetes).
//   - Pre-stop hook integration (the /shutdown endpoint).
//   - Resource-efficient design (small memory footprint, fast startup).
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"runtime"
	"sync/atomic"
	"syscall"
	"time"
)

// appState tracks the application's lifecycle for probe responses.
type appState struct {
	// started is set to true once initialization is complete.
	started atomic.Bool

	// healthy is set to true when the process is functioning
	// correctly. Set to false if a critical internal error occurs.
	healthy atomic.Bool

	// ready is set to true when the application is prepared to
	// serve traffic. Set to false during shutdown to drain connections.
	ready atomic.Bool
}

// statusResponse is the JSON structure returned by all health
// check endpoints.
type statusResponse struct {
	Status    string `json:"status"`
	Timestamp string `json:"timestamp"`
	Uptime    string `json:"uptime,omitempty"`
	GoVersion string `json:"go_version,omitempty"`
}

var (
	state     = &appState{}
	startTime = time.Now()
	logger    *slog.Logger
)

func init() {
	// Configure structured JSON logging for production use.
	logger = slog.New(slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
				Level: slog.LevelInfo,
			}))
	slog.SetDefault(logger)
}

// writeJSON is a helper that marshals a value to JSON and writes it
// to the response with the appropriate content type.
func writeJSON(w http.ResponseWriter, code int, v interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(code)
	json.NewEncoder(w).Encode(v)
}

// handleStartup responds to the Kubernetes startup probe.
func handleStartup(w http.ResponseWriter, r *http.Request) {
	if state.started.Load() {
		writeJSON(w, http.StatusOK, statusResponse{
				Status:    "started",
				Timestamp: time.Now().UTC().Format(time.RFC3339),
			})
	} else {
		writeJSON(w, http.StatusServiceUnavailable, statusResponse{
				Status:    "starting",
				Timestamp: time.Now().UTC().Format(time.RFC3339),
			})
	}
}

// handleLiveness responds to the Kubernetes liveness probe.
func handleLiveness(w http.ResponseWriter, r *http.Request) {
	if state.healthy.Load() {
		writeJSON(w, http.StatusOK, statusResponse{
				Status:    "alive",
				Timestamp: time.Now().UTC().Format(time.RFC3339),
				Uptime:    time.Since(startTime).Round(time.Second).String(),
			})
	} else {
		writeJSON(w, http.StatusServiceUnavailable, statusResponse{
				Status:    "unhealthy",
				Timestamp: time.Now().UTC().Format(time.RFC3339),
			})
	}
}

// handleReadiness responds to the Kubernetes readiness probe.
func handleReadiness(w http.ResponseWriter, r *http.Request) {
	if state.ready.Load() {
		writeJSON(w, http.StatusOK, statusResponse{
				Status:    "ready",
				Timestamp: time.Now().UTC().Format(time.RFC3339),
			})
	} else {
		writeJSON(w, http.StatusServiceUnavailable, statusResponse{
				Status:    "not_ready",
				Timestamp: time.Now().UTC().Format(time.RFC3339),
			})
	}
}

// handleRoot serves the main application endpoint.
func handleRoot(w http.ResponseWriter, r *http.Request) {
	hostname, _ := os.Hostname()
	writeJSON(w, http.StatusOK, map[string]string{
			"message":    "Hello from the Pod!",
			"hostname":   hostname,
			"go_version": runtime.Version(),
			"uptime":     time.Since(startTime).Round(time.Second).String(),
		})
}

func main() {
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	// Simulate initialization (loading config, connecting to DB, etc.).
	slog.Info("Starting initialization sequence")
	time.Sleep(3 * time.Second)
	state.started.Store(true)
	state.healthy.Store(true)
	slog.Info("Initialization complete")

	time.Sleep(1 * time.Second)
	state.ready.Store(true)
	slog.Info("Ready for traffic")

	mux := http.NewServeMux()
	mux.HandleFunc("/", handleRoot)
	mux.HandleFunc("/healthz/startup", handleStartup)
	mux.HandleFunc("/healthz/live", handleLiveness)
	mux.HandleFunc("/healthz/ready", handleReadiness)

	srv := &http.Server{
		Addr:         fmt.Sprintf(":%s", port),
		Handler:      mux,
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 15 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	// Start HTTP server.
	go func() {
		slog.Info("HTTP server starting", "addr", srv.Addr)
		if err := srv.ListenAndServe(); err != http.ErrServerClosed {
			slog.Error("HTTP server error", "error", err)
			os.Exit(1)
		}
	}()

	// Wait for shutdown signal.
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGTERM, syscall.SIGINT)
	sig := <-quit
	slog.Info("Shutdown signal received", "signal", sig)

	// Mark as not ready so readiness probe fails.
	state.ready.Store(false)
	slog.Info("Marked not ready, waiting for endpoint removal")
	time.Sleep(5 * time.Second)

	// Graceful shutdown with timeout.
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()
	if err := srv.Shutdown(ctx); err != nil {
		slog.Error("Shutdown error", "error", err)
	}
	slog.Info("Server stopped")
}
```

### 15.12.2 Multi-Stage Dockerfile

```dockerfile
# Dockerfile
#
# Multi-stage build for a Go HTTP server.
# Stage 1: Build the binary in a full Go environment.
# Stage 2: Copy only the binary into a minimal distroless image.
# This produces a small, secure image with no shell, package manager,
# or unnecessary tools.

# --- Build Stage ---
FROM golang:1.22-alpine AS builder

WORKDIR /build

# Copy go.mod and go.sum first for layer caching.
COPY go.mod go.sum ./
RUN go mod download

# Copy source code and build.
COPY . .
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
    -ldflags='-w -s -extldflags "-static"' \
    -o /app/server .

# --- Runtime Stage ---
FROM gcr.io/distroless/static-debian12:nonroot

# Copy the binary from the build stage.
COPY --from=builder /app/server /app/server

# Run as non-root user (UID 65532 is the 'nonroot' user in distroless).
USER nonroot:nonroot

EXPOSE 8080

ENTRYPOINT ["/app/server"]
```

### 15.12.3 Pod YAML with All Best Practices

```yaml
# pod-go-app.yaml
#
# This Pod YAML applies every best practice discussed in this chapter:
# - Non-root security context
# - Read-only root filesystem
# - Drop all capabilities
# - Resource requests and limits
# - All three probe types
# - Graceful shutdown via preStop hook
# - Topology spread across zones
apiVersion: v1
kind: Pod
metadata:
  name: go-app
  labels:
    app: go-app
    version: v1.0.0
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
spec:
  serviceAccountName: go-app-sa
  automountServiceAccountToken: false

  securityContext:
    runAsUser: 65532
    runAsGroup: 65532
    runAsNonRoot: true
    fsGroup: 65532
    seccompProfile:
      type: RuntimeDefault

  terminationGracePeriodSeconds: 45

  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: go-app

  containers:
  - name: app
    image: myregistry/go-app:v1.0.0
    imagePullPolicy: IfNotPresent
    ports:
    - name: http
      containerPort: 8080
      protocol: TCP

    env:
    - name: PORT
      value: "8080"
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name

    resources:
      requests:
        cpu: 100m
        memory: 64Mi
      limits:
        cpu: 500m
        memory: 128Mi

    startupProbe:
      httpGet:
        path: /healthz/startup
        port: http
      failureThreshold: 15
      periodSeconds: 2

    livenessProbe:
      httpGet:
        path: /healthz/live
        port: http
      periodSeconds: 15
      timeoutSeconds: 3
      failureThreshold: 3

    readinessProbe:
      httpGet:
        path: /healthz/ready
        port: http
      periodSeconds: 5
      timeoutSeconds: 2
      failureThreshold: 2
      successThreshold: 1

    lifecycle:
      preStop:
        exec:
          command: ["sh", "-c", "sleep 5"]

    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]

    volumeMounts:
    - name: tmp
      mountPath: /tmp

  volumes:
  - name: tmp
    emptyDir:
      sizeLimit: 50Mi

  imagePullSecrets:
  - name: registry-creds
```

---

## 15.13 Summary

In this chapter, we explored the Pod — the atomic unit of deployment in Kubernetes.
We covered:

1. **What a Pod is:** A group of containers sharing network, IPC, and (optionally)
   PID namespaces, plus shared volumes — functioning as a logical host.

2. **Every important field in PodSpec:** From `containers[]` and `initContainers[]`
   to `affinity`, `tolerations`, `securityContext`, `dnsPolicy`, and
   `terminationGracePeriodSeconds`.

3. **Multi-container patterns:** Sidecar (logging, proxy), Ambassador (connection
   pooling), Adapter (metrics transformation), and Init (setup before main app).

4. **Pod networking:** The pause container, shared network namespace, localhost
   communication, and Pod IP assignment via CNI.

5. **Resource management:** Requests, limits, QoS classes, ephemeral storage,
   extended resources, LimitRange, and ResourceQuota.

6. **Probes:** Startup, liveness, and readiness probes with HTTP, TCP, gRPC, and
   exec mechanisms. Tuning parameters and common misconfigurations.

7. **Security contexts:** Running as non-root, dropping capabilities, read-only
   filesystems, seccomp profiles, and SELinux/AppArmor.

8. **Pod Disruption Budgets:** Protecting availability during voluntary disruptions.

9. **Topology spread constraints:** Distributing Pods across zones and nodes.

10. **Pod overhead:** RuntimeClass overhead for VM-based runtimes.

11. **Pod lifecycle:** Phases, conditions, container states, and the complete
    creation-to-termination sequence.

12. **A complete hands-on:** Go HTTP server with health checks, multi-stage
    Dockerfile, and production-hardened Pod YAML.

In the next chapter, we'll see how Pods are managed at scale through ReplicaSets
and Deployments — the controllers that ensure your desired number of Pod replicas
are always running.
