# Chapter 19: StatefulSets — Stateful Workloads Done Right

> *"In a world of cattle, some workloads insist on being pets.
> StatefulSets are Kubernetes' answer to that stubbornness."*

Up to this point, every workload controller we have studied — Deployments,
ReplicaSets, even bare Pods — has treated its Pods as interchangeable units of
compute. If one Pod dies, another rises in its place with a new name, a new IP,
and no memory of its predecessor. For a stateless web server that reads
configuration from environment variables and queries a remote database, this
model is perfect.

But the moment your workload *is* the database — or a distributed cache, a
message broker, a consensus cluster — interchangeability becomes a liability.
These systems need **stable identities**, **stable storage**, and **ordered
lifecycle operations**. Kubernetes provides a first-class primitive for exactly
this class of workload: the **StatefulSet**.

This chapter takes you from first principles to production-grade patterns. We
will deploy a Redis cluster, a PostgreSQL primary-replica setup, and a custom
distributed key-value store written in Go — all powered by StatefulSets.

---

## 19.1 Why StatefulSets Exist

### 19.1.1 The Limits of Deployments for Stateful Workloads

A Deployment manages a ReplicaSet, which in turn manages a set of Pods. Every
Pod in a ReplicaSet is **fungible**: the controller does not care which
particular Pod is running, only that the *count* matches the desired replica
count. This means:

| Property                  | Deployment Behavior                              |
|---------------------------|--------------------------------------------------|
| **Pod name**              | Random suffix (`web-7d4f8b6c9-xk2lz`)           |
| **Network identity**      | Ephemeral — changes on every reschedule          |
| **Storage**               | Shared or ephemeral; no per-Pod guarantee        |
| **Startup/shutdown order**| Arbitrary — all Pods start/stop concurrently     |
| **Rolling update order**  | Arbitrary                                        |

For a stateless HTTP API behind a load balancer, none of these properties
matter. For a three-node etcd cluster, *every single one* matters.

Consider what happens when you run a three-node etcd cluster using a
Deployment:

1. **Pod names are random.** etcd nodes need to know each other by stable
   names to form a cluster. If `etcd-abc123` restarts as `etcd-def456`, the
   cluster's peer list is broken.

2. **Network identities are ephemeral.** Even if you discover peers via a
   Service, the individual Pod IPs change on reschedule. etcd's peer URLs
   become stale.

3. **Storage is shared or lost.** If all three Pods mount the same
   PersistentVolume, you have a data corruption nightmare. If they use
   `emptyDir`, data is lost on Pod restart.

4. **Startup order is undefined.** etcd's bootstrap procedure requires the
   first node to initialize the cluster before others join. A Deployment
   starts all three concurrently, leading to split-brain or boot loops.

### 19.1.2 What StatefulSets Provide

A StatefulSet solves each of these problems with four guarantees:

1. **Stable, unique Pod names.** Pods are named
   `<statefulset-name>-<ordinal>`: `etcd-0`, `etcd-1`, `etcd-2`. The ordinal
   is deterministic and survives rescheduling.

2. **Stable DNS names via a headless Service.** Each Pod gets a DNS record:
   `<pod-name>.<service-name>.<namespace>.svc.cluster.local`. `etcd-0` is
   always reachable at `etcd-0.etcd-headless.default.svc.cluster.local`.

3. **Stable, per-Pod storage.** Each Pod gets its own PersistentVolumeClaim.
   When `etcd-1` is rescheduled to a different Node, it reattaches to the
   same PVC and finds its data intact.

4. **Ordered lifecycle operations.** Pods are created in ascending ordinal
   order (0, 1, 2) and deleted in descending order (2, 1, 0). Rolling
   updates proceed in reverse ordinal order by default.

### 19.1.3 Canonical Use Cases

| Use Case                     | Why StatefulSet?                                  |
|------------------------------|---------------------------------------------------|
| **Databases** (PostgreSQL, MySQL, MongoDB) | Stable storage, primary/replica ordering |
| **Distributed caches** (Redis Cluster, Memcached) | Stable identity for hash slots    |
| **Consensus systems** (etcd, ZooKeeper)    | Ordered bootstrap, stable peer list      |
| **Message brokers** (Kafka, RabbitMQ)      | Stable broker IDs, persistent log storage|
| **Search engines** (Elasticsearch)         | Stable node roles, persistent indices    |

> **Rule of thumb:** If your application needs to know *which instance it is*
> or needs *data to survive Pod restarts*, reach for a StatefulSet.

---

## 19.2 StatefulSet Guarantees — In Depth

Let us examine each guarantee in detail, because understanding the *mechanics*
is essential for debugging production issues.

### 19.2.1 Stable, Unique Pod Names

Every Pod managed by a StatefulSet is assigned an **ordinal index** starting
from 0. The Pod's name is constructed as:

```
<statefulset-name>-<ordinal>
```

For a StatefulSet named `web` with 3 replicas:

```
web-0
web-1
web-2
```

These names are **deterministic**. If `web-1` is deleted (by the controller, by
a node failure, or by you), the replacement Pod is also named `web-1`. This is
fundamentally different from a ReplicaSet, which would create a Pod with a new
random suffix.

The ordinal is also injected into the Pod's **hostname**:

```bash
# Inside web-1:
$ hostname
web-1
```

This means your application can discover its own identity by reading its
hostname — a pattern we will exploit extensively in the hands-on examples.

### 19.2.2 Stable DNS Names via Headless Service

A **headless Service** (one with `clusterIP: None`) is the foundation of
StatefulSet networking. When you create a headless Service that selects
StatefulSet Pods, Kubernetes creates individual DNS records for each Pod:

```
<pod-name>.<service-name>.<namespace>.svc.cluster.local
```

For our `web` StatefulSet with a headless Service named `web-headless`:

```
web-0.web-headless.default.svc.cluster.local → 10.244.1.5
web-1.web-headless.default.svc.cluster.local → 10.244.2.8
web-2.web-headless.default.svc.cluster.local → 10.244.3.2
```

These DNS names are **stable** — they always resolve to the current IP of the
named Pod, even after rescheduling. This is how StatefulSet Pods discover and
communicate with each other.

Additionally, **SRV records** are created for named ports:

```bash
$ dig SRV _http._tcp.web-headless.default.svc.cluster.local

;; ANSWER SECTION:
_http._tcp.web-headless.default.svc.cluster.local. 30 IN SRV 0 33 80 web-0.web-headless.default.svc.cluster.local.
_http._tcp.web-headless.default.svc.cluster.local. 30 IN SRV 0 33 80 web-1.web-headless.default.svc.cluster.local.
_http._tcp.web-headless.default.svc.cluster.local. 30 IN SRV 0 33 80 web-2.web-headless.default.svc.cluster.local.
```

### 19.2.3 Stable Storage — PVC per Pod

A StatefulSet's `volumeClaimTemplates` field is a list of PVC templates. For
each replica, the controller creates a PVC named:

```
<template-name>-<statefulset-name>-<ordinal>
```

For a template named `data` in a StatefulSet named `web`:

```
data-web-0
data-web-1
data-web-2
```

When `web-1` is deleted and recreated, the new `web-1` Pod is bound to the
*same* PVC `data-web-1`. The data survives because the PVC — and its backing
PersistentVolume — persists independently of the Pod.

**Critical detail:** When you scale a StatefulSet *down* (say from 3 to 2),
the controller deletes `web-2` but does **not** delete `data-web-2`. This is
by design — Kubernetes errs on the side of data preservation. You must delete
orphaned PVCs manually if you want to reclaim storage.

### 19.2.4 Ordered, Graceful Deployment and Scaling

With the default `podManagementPolicy: OrderedReady`:

- **Scale up:** Pods are created one at a time in ascending ordinal order.
  `web-0` must be **Running and Ready** before `web-1` is created. `web-1`
  must be Running and Ready before `web-2` is created.

- **Scale down:** Pods are terminated one at a time in descending ordinal
  order. `web-2` is terminated first. Only after it is fully terminated is
  `web-1` terminated.

- **Updates:** Pods are updated in reverse ordinal order by default. `web-2`
  is updated first, then `web-1`, then `web-0`.

This ordering is critical for distributed systems. For example, in a
ZooKeeper ensemble, the leader is typically the lowest-ordinal node. Updating
or scaling down from the highest ordinal ensures the leader is disrupted last.

### 19.2.5 What StatefulSets Do NOT Guarantee

StatefulSets provide ordering and identity, but they do **not** provide:

- **Data replication.** Your application must handle replication (or you must
  use an operator that does).
- **Backup and restore.** PVCs persist, but backup is your responsibility.
- **Automated failover.** If `web-0` is the primary database and it fails,
  the StatefulSet will restart it, but it will not promote `web-1`.
- **Application-level configuration.** You must wire up primary/replica
  configuration yourself (often via init containers).

For these higher-level concerns, the **operator pattern** is the standard
solution. We will touch on this at the end of the chapter and dive deep in
Chapters 32–35.

---

## 19.3 StatefulSet Spec Deep Dive

Let us examine every significant field in a StatefulSet manifest. Here is a
comprehensive example:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
  namespace: default
  labels:
    app: web
spec:
  # -----------------------------------------------------------
  # serviceName: REQUIRED. Must match the name of a headless
  # Service that governs this StatefulSet's network identity.
  # -----------------------------------------------------------
  serviceName: "web-headless"

  # -----------------------------------------------------------
  # replicas: Desired number of Pods. Defaults to 1.
  # -----------------------------------------------------------
  replicas: 3

  # -----------------------------------------------------------
  # selector: Label selector that must match the Pod template's
  # labels. Immutable after creation.
  # -----------------------------------------------------------
  selector:
    matchLabels:
      app: web

  # -----------------------------------------------------------
  # podManagementPolicy: Controls how Pods are created/deleted.
  #   OrderedReady (default): One at a time, in order.
  #   Parallel: All at once (for apps that don't need ordering).
  # -----------------------------------------------------------
  podManagementPolicy: OrderedReady

  # -----------------------------------------------------------
  # updateStrategy: Controls how Pods are updated.
  #   RollingUpdate (default): Updates in reverse ordinal order.
  #     partition: Only Pods with ordinal >= partition are updated.
  #   OnDelete: Pods are only updated when manually deleted.
  # -----------------------------------------------------------
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0

  # -----------------------------------------------------------
  # revisionHistoryLimit: Number of old controller revisions to
  # retain. Defaults to 10.
  # -----------------------------------------------------------
  revisionHistoryLimit: 10

  # -----------------------------------------------------------
  # minReadySeconds: Minimum seconds a Pod must be Ready without
  # crashing before it is considered Available. Defaults to 0.
  # -----------------------------------------------------------
  minReadySeconds: 10

  # -----------------------------------------------------------
  # ordinals: Starting ordinal for Pod naming. Allows starting
  # from a number other than 0 (e.g., start: 1 gives web-1,
  # web-2, web-3 for 3 replicas). Requires feature gate
  # StatefulSetStartOrdinal (stable in 1.31).
  # -----------------------------------------------------------
  ordinals:
    start: 0

  # -----------------------------------------------------------
  # persistentVolumeClaimRetentionPolicy: Controls PVC lifecycle
  # when StatefulSet is scaled down or deleted.
  #   whenScaled: Retain (default) or Delete
  #   whenDeleted: Retain (default) or Delete
  # Requires StatefulSetAutoDeletePVC feature gate (beta in 1.27).
  # -----------------------------------------------------------
  persistentVolumeClaimRetentionPolicy:
    whenScaled: Retain
    whenDeleted: Retain

  # -----------------------------------------------------------
  # template: Pod template — same as a Deployment's template.
  # -----------------------------------------------------------
  template:
    metadata:
      labels:
        app: web
    spec:
      terminationGracePeriodSeconds: 30
      containers:
        - name: web
          image: nginx:1.25
          ports:
            - containerPort: 80
              name: http
          volumeMounts:
            - name: data
              mountPath: /usr/share/nginx/html
          readinessProbe:
            httpGet:
              path: /
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi

  # -----------------------------------------------------------
  # volumeClaimTemplates: List of PVC templates. One PVC is
  # created per replica per template. PVCs are named:
  #   <template-name>-<statefulset-name>-<ordinal>
  # -----------------------------------------------------------
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 1Gi
```

### 19.3.1 serviceName

This field is **required** and must reference a headless Service that already
exists (or is created alongside the StatefulSet). The headless Service provides
the DNS domain for Pod network identities.

If you forget to create the headless Service, the StatefulSet will be created
but DNS resolution for individual Pods will fail.

### 19.3.2 podManagementPolicy

Two options:

- **OrderedReady** (default): Pods are created strictly in order (0, 1, 2, …)
  and each must be Running+Ready before the next is created. Deletion is in
  reverse order. This is the safe choice for most stateful workloads.

- **Parallel**: All Pods are created or deleted simultaneously. Use this when
  your application handles its own coordination (e.g., Cassandra with gossip
  protocol) and you want faster scaling.

### 19.3.3 updateStrategy

Two types:

**RollingUpdate** (default):

- Updates Pods in reverse ordinal order: highest ordinal first.
- The `partition` field enables **staged rollouts**: only Pods with ordinal ≥
  `partition` are updated. This is incredibly useful for canary deployments of
  stateful workloads.

**OnDelete**:

- The controller does **not** automatically update Pods. You must manually
  delete each Pod to trigger recreation with the new spec.
- Useful when you need full control over the update sequence (e.g., database
  migrations that require manual validation between nodes).

### 19.3.4 volumeClaimTemplates

Each entry in `volumeClaimTemplates` is a PVC template. The controller creates
one PVC per replica per template. A StatefulSet with 3 replicas and two
templates (`data` and `wal`) produces six PVCs:

```
data-web-0, data-web-1, data-web-2
wal-web-0,  wal-web-1,  wal-web-2
```

PVCs are bound to the Pod by ordinal. This binding is **permanent** for the
lifetime of the PVC — even if the StatefulSet is deleted (with the default
retention policy).

### 19.3.5 persistentVolumeClaimRetentionPolicy

This newer field (beta since Kubernetes 1.27) gives you control over what
happens to PVCs when a StatefulSet is scaled down or deleted:

| Policy        | whenScaled | whenDeleted |
|---------------|------------|-------------|
| **Retain** (default) | PVCs survive scale-down | PVCs survive StatefulSet deletion |
| **Delete**    | PVCs deleted on scale-down | PVCs deleted when StatefulSet is deleted |

> **Warning:** Setting `whenDeleted: Delete` means deleting a StatefulSet
> destroys all its data. Use with extreme caution.

### 19.3.6 ordinals

The `ordinals.start` field (stable since Kubernetes 1.31) allows you to start
Pod numbering from a value other than 0. For example, `start: 5` with 3
replicas produces Pods named `web-5`, `web-6`, `web-7`.

This is useful for:
- Migrating from one StatefulSet to another without name collisions.
- Advanced multi-cluster topologies where ordinal ranges are partitioned.

---

## 19.4 How the StatefulSet Controller Works

Understanding the controller's reconciliation loop is essential for debugging.
Here is the algorithm in pseudocode:

```
LOOP forever:
    desired = statefulSet.Spec.Replicas
    current = list Pods matching statefulSet.Spec.Selector, owned by this StatefulSet

    sort current by ordinal ascending

    # --- SCALE UP ---
    for ordinal in 0..(desired - 1):
        if Pod with this ordinal does not exist:
            create Pod <name>-<ordinal>
            if podManagementPolicy == OrderedReady:
                WAIT until Pod is Running + Ready
                (do not create next Pod until this one is ready)

    # --- SCALE DOWN ---
    for ordinal in (len(current) - 1) down to desired:
        delete Pod <name>-<ordinal>
        if podManagementPolicy == OrderedReady:
            WAIT until Pod is fully terminated
            (do not delete next Pod until this one is gone)

    # --- ROLLING UPDATE ---
    if updateStrategy == RollingUpdate:
        for ordinal in (desired - 1) down to partition:
            if Pod at this ordinal is not at current revision:
                delete Pod <name>-<ordinal>
                # Controller will recreate it with new spec on next loop
                if podManagementPolicy == OrderedReady:
                    WAIT until new Pod is Running + Ready

    # --- HANDLE FAILURES ---
    for each Pod in current:
        if Pod is Failed and not being deleted:
            recreate Pod
```

### 19.4.1 Key Observations

1. **The controller never creates or deletes more than one Pod at a time**
   (with `OrderedReady` policy). This serial behavior is intentional — it
   prevents thundering-herd issues in stateful systems.

2. **A stuck Pod blocks everything.** If `web-1` never becomes Ready during
   scale-up, `web-2` will never be created. This is both a safety feature
   (it prevents cascading failures) and a debugging challenge (you must fix
   the stuck Pod to unblock progress).

3. **The controller is level-triggered, not edge-triggered.** It reconciles
   the entire state on every loop iteration, not just in response to events.
   This means it is self-healing: if you manually delete a Pod, the
   controller detects the gap and recreates it.

4. **PVC creation is separate from Pod creation.** The controller creates
   PVCs before creating Pods. If PVC creation fails (e.g., no available
   storage), the Pod is not created.

### 19.4.2 Identity and the Sticky Relationship

Each Pod in a StatefulSet has a **sticky identity** consisting of:

- **Ordinal:** An integer starting from 0 (or `ordinals.start`).
- **Stable hostname:** `<statefulset-name>-<ordinal>`.
- **Stable DNS name:** `<pod-name>.<service-name>.<namespace>.svc.cluster.local`.
- **Stable storage:** PVC `<template-name>-<statefulset-name>-<ordinal>`.

This identity is **reattached** when a Pod is rescheduled. If `web-1` is
evicted from Node A and rescheduled to Node B, the new Pod on Node B:

- Is still named `web-1`.
- Still resolves at `web-1.web-headless.default.svc.cluster.local`.
- Still mounts PVC `data-web-1` (which may need to be detached from Node A
  and attached to Node B — a process that can take time with cloud volumes).

---

## 19.5 Headless Service — The Foundation

A headless Service is the DNS backbone of a StatefulSet. Without it,
StatefulSet Pods have no stable network identity.

### 19.5.1 What Makes a Service "Headless"

A Service is headless when `spec.clusterIP` is set to `None`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-headless
  labels:
    app: web
spec:
  # -----------------------------------------------------------
  # clusterIP: None makes this a headless Service.
  # Instead of a single virtual IP, DNS returns individual
  # Pod IPs directly.
  # -----------------------------------------------------------
  clusterIP: None
  selector:
    app: web
  ports:
    - port: 80
      name: http
      targetPort: 80
```

### 19.5.2 DNS Behavior Comparison

| Service Type        | DNS A Record Returns             | Load Balancing    |
|---------------------|----------------------------------|-------------------|
| ClusterIP (normal)  | Single virtual IP                | kube-proxy rules  |
| Headless            | Set of individual Pod IPs        | Client-side       |

When you query a normal Service, you get one IP:

```bash
$ nslookup web-service.default.svc.cluster.local
Address: 10.96.42.17
```

When you query a headless Service, you get all Pod IPs:

```bash
$ nslookup web-headless.default.svc.cluster.local
Address: 10.244.1.5
Address: 10.244.2.8
Address: 10.244.3.2
```

And you can query individual Pods by name:

```bash
$ nslookup web-0.web-headless.default.svc.cluster.local
Address: 10.244.1.5
```

### 19.5.3 SRV Records for Port Discovery

SRV records allow clients to discover both the hostname and port of each Pod:

```bash
$ dig SRV _http._tcp.web-headless.default.svc.cluster.local

;; ANSWER SECTION:
_http._tcp.web-headless.default.svc.cluster.local. 30 IN SRV 0 33 80 web-0.web-headless.default.svc.cluster.local.
_http._tcp.web-headless.default.svc.cluster.local. 30 IN SRV 0 33 80 web-1.web-headless.default.svc.cluster.local.
_http._tcp.web-headless.default.svc.cluster.local. 30 IN SRV 0 33 80 web-2.web-headless.default.svc.cluster.local.
```

This is particularly useful for applications like Kafka, where brokers
advertise their endpoints to clients.

### 19.5.4 Hands-on: Verifying DNS Records

```bash
# Create the headless Service and StatefulSet (using the YAML from section 19.3)
kubectl apply -f web-headless-service.yaml
kubectl apply -f web-statefulset.yaml

# Wait for all Pods to be ready
kubectl wait --for=condition=Ready pod/web-0 pod/web-1 pod/web-2 --timeout=120s

# Launch a DNS debugging Pod
kubectl run dns-debug --image=busybox:1.36 --rm -it --restart=Never -- sh

# Inside the debug Pod:
# Query the headless Service — returns all Pod IPs
nslookup web-headless.default.svc.cluster.local

# Query individual Pods
nslookup web-0.web-headless.default.svc.cluster.local
nslookup web-1.web-headless.default.svc.cluster.local
nslookup web-2.web-headless.default.svc.cluster.local

# Verify SRV records
nslookup -type=srv _http._tcp.web-headless.default.svc.cluster.local
```

---

## 19.6 Persistent Volume Claims — Data That Survives

### 19.6.1 How volumeClaimTemplates Work

The `volumeClaimTemplates` field in a StatefulSet is a list of PVC
specifications. For each template and each replica, the StatefulSet controller
creates a PVC before creating the corresponding Pod.

The naming convention is:

```
<volumeClaimTemplate.metadata.name>-<statefulSet.metadata.name>-<ordinal>
```

For a StatefulSet `postgres` with a template named `pgdata` and 3 replicas:

```
pgdata-postgres-0   →  bound to PV pv-abc123
pgdata-postgres-1   →  bound to PV pv-def456
pgdata-postgres-2   →  bound to PV pv-ghi789
```

### 19.6.2 PVC Lifecycle — The Retention Guarantee

Here is the key insight that trips up many engineers:

> **PVCs created by a StatefulSet are NEVER automatically deleted** (with the
> default retention policy).

This means:

1. **Pod restart:** Pod is deleted and recreated → same PVC is reattached.
   Data intact.
2. **Pod rescheduling:** Pod moves to a different Node → same PVC is
   reattached (volume is detached from old Node and attached to new Node).
   Data intact.
3. **Scale down:** StatefulSet scaled from 3 to 2 → `web-2` Pod is deleted,
   but `data-web-2` PVC is **retained**. Data intact.
4. **Scale back up:** StatefulSet scaled from 2 to 3 → new `web-2` Pod is
   created and **reattaches to existing `data-web-2` PVC**. Data intact.
5. **StatefulSet deletion:** StatefulSet is deleted → all Pods are deleted,
   but all PVCs are **retained**. Data intact.

This behavior prioritizes data safety over resource cleanup. In production,
you should have a process (manual or automated) to review and clean up
orphaned PVCs.

### 19.6.3 Hands-on: Verifying PVC Persistence

```bash
# Deploy a StatefulSet with a volumeClaimTemplate
kubectl apply -f web-statefulset.yaml

# List PVCs — one per replica
kubectl get pvc -l app=web
# NAME         STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS
# data-web-0   Bound    pv-abc123    1Gi        RWO            standard
# data-web-1   Bound    pv-def456    1Gi        RWO            standard
# data-web-2   Bound    pv-ghi789    1Gi        RWO            standard

# Write data to web-1's volume
kubectl exec web-1 -- sh -c 'echo "I am web-1 and my data persists" > /usr/share/nginx/html/index.html'

# Delete web-1 Pod (the controller will recreate it)
kubectl delete pod web-1

# Wait for the new web-1 to be ready
kubectl wait --for=condition=Ready pod/web-1 --timeout=60s

# Verify data survived
kubectl exec web-1 -- cat /usr/share/nginx/html/index.html
# Output: I am web-1 and my data persists

# Scale down to 2 replicas
kubectl scale statefulset web --replicas=2

# web-2 Pod is deleted, but PVC survives
kubectl get pvc data-web-2
# NAME         STATUS   VOLUME       CAPACITY   ACCESS MODES   STORAGECLASS
# data-web-2   Bound    pv-ghi789    1Gi        RWO            standard

# Scale back up to 3 replicas — web-2 reattaches to its old PVC
kubectl scale statefulset web --replicas=3
kubectl wait --for=condition=Ready pod/web-2 --timeout=60s
```

---

## 19.7 Hands-on: Deploying a Complete Stateful Application

### 19.7.1 Example 1: Redis Cluster with StatefulSet

A production Redis Cluster requires at least 6 nodes (3 primaries + 3
replicas) with stable identities and persistent storage. Here is the complete
setup:

```yaml
# ------------------------------------------------------------------
# redis-configmap.yaml
# Configuration for Redis Cluster nodes.
# ------------------------------------------------------------------
apiVersion: v1
kind: ConfigMap
metadata:
  name: redis-cluster-config
data:
  redis.conf: |
    # Enable Redis Cluster mode
    cluster-enabled yes
    # File where cluster topology is persisted
    cluster-config-file nodes.conf
    # Timeout for node failure detection (ms)
    cluster-node-timeout 5000
    # Enable AOF persistence for durability
    appendonly yes
    appendfilename "appendonly.aof"
    # Bind to all interfaces (Pod networking)
    bind 0.0.0.0
    # Disable protected mode (within cluster network)
    protected-mode no
    # Port for client connections
    port 6379
    # Port for cluster bus (gossip protocol)
    cluster-announce-port 6379
    cluster-announce-bus-port 16379

---
# ------------------------------------------------------------------
# redis-headless-service.yaml
# Headless Service for Redis Cluster — provides stable DNS for each
# Pod so nodes can discover each other by hostname.
# ------------------------------------------------------------------
apiVersion: v1
kind: Service
metadata:
  name: redis-cluster
  labels:
    app: redis-cluster
spec:
  clusterIP: None
  selector:
    app: redis-cluster
  ports:
    - port: 6379
      name: client
      targetPort: 6379
    - port: 16379
      name: gossip
      targetPort: 16379

---
# ------------------------------------------------------------------
# redis-statefulset.yaml
# StatefulSet for a 6-node Redis Cluster (3 primaries + 3 replicas).
# Each Pod gets persistent storage and a stable DNS identity.
# ------------------------------------------------------------------
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: redis-cluster
spec:
  serviceName: redis-cluster
  replicas: 6
  selector:
    matchLabels:
      app: redis-cluster
  podManagementPolicy: OrderedReady
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: redis-cluster
    spec:
      # --------------------------------------------------------
      # Init container: Configures the node's announce IP using
      # the Pod's hostname and the headless Service DNS.
      # --------------------------------------------------------
      initContainers:
        - name: configure
          image: redis:7.2
          command:
            - sh
            - -c
            - |
              # Resolve this Pod's FQDN to get its IP address.
              # Redis needs the IP for cluster-announce-ip because
              # it communicates the IP to other nodes in gossip.
              HOSTNAME=$(hostname)
              FQDN="${HOSTNAME}.redis-cluster.default.svc.cluster.local"

              # Wait for DNS to be resolvable (can take a moment after Pod start)
              until getent hosts "${FQDN}"; do
                echo "Waiting for DNS resolution of ${FQDN}..."
                sleep 1
              done

              POD_IP=$(getent hosts "${FQDN}" | awk '{print $1}')

              # Write the announce-ip directive to a config file
              # that Redis will include at startup.
              echo "cluster-announce-ip ${POD_IP}" > /etc/redis/announce.conf
              echo "Configured announce IP: ${POD_IP}"
          volumeMounts:
            - name: redis-config
              mountPath: /etc/redis
      containers:
        - name: redis
          image: redis:7.2
          ports:
            - containerPort: 6379
              name: client
            - containerPort: 16379
              name: gossip
          command:
            - redis-server
            - /conf/redis.conf
            - --include
            - /etc/redis/announce.conf
          readinessProbe:
            exec:
              command:
                - redis-cli
                - ping
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            exec:
              command:
                - redis-cli
                - ping
            initialDelaySeconds: 15
            periodSeconds: 10
          volumeMounts:
            - name: data
              mountPath: /data
            - name: conf
              mountPath: /conf
              readOnly: true
            - name: redis-config
              mountPath: /etc/redis
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 512Mi
      volumes:
        - name: conf
          configMap:
            name: redis-cluster-config
        - name: redis-config
          emptyDir: {}
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 5Gi
```

**Initializing the cluster** after all 6 Pods are running:

```bash
# Wait for all Pods
kubectl wait --for=condition=Ready pod -l app=redis-cluster --timeout=300s

# Collect all Pod IPs
PODS=$(kubectl get pods -l app=redis-cluster \
  -o jsonpath='{range .items[*]}{.status.podIP}:6379 {end}')

# Create the cluster (3 primaries, 1 replica per primary)
kubectl exec -it redis-cluster-0 -- redis-cli --cluster create \
  ${PODS} \
  --cluster-replicas 1 --cluster-yes

# Verify cluster status
kubectl exec redis-cluster-0 -- redis-cli cluster info
kubectl exec redis-cluster-0 -- redis-cli cluster nodes
```

### 19.7.2 Example 2: PostgreSQL with Primary-Replica

This example demonstrates a 1-primary, 2-replica PostgreSQL setup using a
StatefulSet. The init container determines whether a Pod is the primary
(ordinal 0) or a replica (ordinal > 0) and configures accordingly.

```yaml
# ------------------------------------------------------------------
# postgres-configmap.yaml
# Configuration files for PostgreSQL primary and replicas.
# ------------------------------------------------------------------
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  # --------------------------------------------------------
  # primary-postgresql.conf: Settings for the primary node.
  # Enables WAL shipping for streaming replication.
  # --------------------------------------------------------
  primary-postgresql.conf: |
    listen_addresses = '*'
    port = 5432
    max_connections = 200
    shared_buffers = 256MB
    wal_level = replica
    max_wal_senders = 10
    wal_keep_size = 256MB
    hot_standby = on
    synchronous_commit = on
    archive_mode = on
    archive_command = 'cp %p /var/lib/postgresql/archive/%f'

  # --------------------------------------------------------
  # replica-postgresql.conf: Settings for replica nodes.
  # Enables hot standby for read queries.
  # --------------------------------------------------------
  replica-postgresql.conf: |
    listen_addresses = '*'
    port = 5432
    max_connections = 200
    shared_buffers = 256MB
    hot_standby = on
    hot_standby_feedback = on

  # --------------------------------------------------------
  # pg_hba.conf: Host-based authentication.
  # Allows replication connections from the Pod network.
  # --------------------------------------------------------
  pg_hba.conf: |
    # TYPE  DATABASE        USER            ADDRESS                 METHOD
    local   all             all                                     trust
    host    all             all             0.0.0.0/0               md5
    host    replication     replicator      0.0.0.0/0               md5

  # --------------------------------------------------------
  # setup-replication.sh: Init container script that
  # configures the Pod as either primary or replica based
  # on its ordinal index.
  # --------------------------------------------------------
  setup-replication.sh: |
    #!/bin/bash
    set -e

    # Extract the ordinal from the hostname (e.g., postgres-0 → 0)
    ORDINAL=$(hostname | grep -o '[0-9]*$')
    PGDATA="/var/lib/postgresql/data/pgdata"

    echo "Pod ordinal: ${ORDINAL}"

    if [ "${ORDINAL}" -eq 0 ]; then
      echo "=== Configuring as PRIMARY ==="

      # If PGDATA is empty, let PostgreSQL initialize it.
      # If it already has data (PVC was retained), just update config.
      if [ ! -s "${PGDATA}/PG_VERSION" ]; then
        echo "Initializing fresh primary database..."
        initdb -D "${PGDATA}" --auth-host=md5 --auth-local=trust

        # Create the replication user
        pg_ctl start -D "${PGDATA}" -w -o "-c listen_addresses='localhost'"
        psql -U postgres -c "CREATE USER replicator WITH REPLICATION ENCRYPTED PASSWORD 'repl_password';"
        pg_ctl stop -D "${PGDATA}" -w
      fi

      # Install primary config
      cp /config/primary-postgresql.conf "${PGDATA}/postgresql.conf"
      cp /config/pg_hba.conf "${PGDATA}/pg_hba.conf"

      # Create archive directory
      mkdir -p /var/lib/postgresql/archive

    else
      echo "=== Configuring as REPLICA (ordinal ${ORDINAL}) ==="

      PRIMARY_HOST="postgres-0.postgres-headless.default.svc.cluster.local"

      # Wait for primary to be available
      until pg_isready -h "${PRIMARY_HOST}" -U postgres; do
        echo "Waiting for primary at ${PRIMARY_HOST}..."
        sleep 2
      done

      # If PGDATA is empty, clone from primary using pg_basebackup
      if [ ! -s "${PGDATA}/PG_VERSION" ]; then
        echo "Cloning from primary..."
        rm -rf "${PGDATA}"
        PGPASSWORD=repl_password pg_basebackup \
          -h "${PRIMARY_HOST}" \
          -U replicator \
          -D "${PGDATA}" \
          -X stream \
          -P \
          -R  # Creates standby.signal and configures primary_conninfo
      fi

      # Install replica config
      cp /config/replica-postgresql.conf "${PGDATA}/postgresql.conf"
      cp /config/pg_hba.conf "${PGDATA}/pg_hba.conf"

      # Ensure standby.signal exists (triggers replica mode)
      touch "${PGDATA}/standby.signal"
    fi

    echo "Configuration complete."

---
# ------------------------------------------------------------------
# postgres-headless-service.yaml
# Headless Service for PostgreSQL StatefulSet.
# ------------------------------------------------------------------
apiVersion: v1
kind: Service
metadata:
  name: postgres-headless
spec:
  clusterIP: None
  selector:
    app: postgres
  ports:
    - port: 5432
      name: postgres

---
# ------------------------------------------------------------------
# postgres-service.yaml
# ClusterIP Service for connecting to the primary (read-write).
# Uses a label selector that only matches the primary Pod.
# ------------------------------------------------------------------
apiVersion: v1
kind: Service
metadata:
  name: postgres-primary
spec:
  selector:
    app: postgres
    role: primary
  ports:
    - port: 5432
      name: postgres

---
# ------------------------------------------------------------------
# postgres-statefulset.yaml
# StatefulSet for PostgreSQL primary-replica cluster.
# Pod ordinal 0 = primary, ordinals 1+ = replicas.
# ------------------------------------------------------------------
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres-headless
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  podManagementPolicy: OrderedReady
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      partition: 0
  template:
    metadata:
      labels:
        app: postgres
    spec:
      initContainers:
        - name: setup
          image: postgres:16
          command: ["bash", "/config/setup-replication.sh"]
          volumeMounts:
            - name: pgdata
              mountPath: /var/lib/postgresql/data
            - name: config
              mountPath: /config
      containers:
        - name: postgres
          image: postgres:16
          ports:
            - containerPort: 5432
              name: postgres
          env:
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
            - name: POSTGRES_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: postgres-secret
                  key: password
          readinessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - postgres
            initialDelaySeconds: 10
            periodSeconds: 5
          livenessProbe:
            exec:
              command:
                - pg_isready
                - -U
                - postgres
            initialDelaySeconds: 30
            periodSeconds: 10
          volumeMounts:
            - name: pgdata
              mountPath: /var/lib/postgresql/data
            - name: config
              mountPath: /config
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: "2"
              memory: 2Gi
      volumes:
        - name: config
          configMap:
            name: postgres-config
            defaultMode: 0755
  volumeClaimTemplates:
    - metadata:
        name: pgdata
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 10Gi
```

### 19.7.3 Example 3: Building a Distributed Key-Value Store in Go

Now let us build something from scratch: a distributed key-value store that
uses the StatefulSet's ordinal for **hash-based sharding** and the headless
Service for **peer discovery**.

**Architecture:**

- Each Pod owns a range of hash keys based on its ordinal.
- When a client sends a request to any Pod, that Pod hashes the key to
  determine the owner and either handles it locally or proxies to the correct
  peer.
- Peers are discovered via DNS lookup of the headless Service.

```go
// ================================================================
// main.go — Distributed Key-Value Store for Kubernetes StatefulSets
// ================================================================
//
// This program implements a simple distributed key-value store
// designed to run as a StatefulSet in Kubernetes. It exploits
// StatefulSet guarantees — stable hostnames, stable DNS, and
// ordered scaling — to implement hash-based key sharding across
// a set of peer Pods.
//
// Each Pod determines its own ordinal index from its hostname
// (e.g., "kvstore-2" → ordinal 2) and discovers the total
// number of peers by querying the headless Service DNS. Keys
// are assigned to Pods using consistent hashing:
//
//     owner = hash(key) % totalPeers
//
// If a request arrives at the wrong Pod, it is transparently
// proxied to the correct owner.
//
// Endpoints:
//   GET  /get?key=<key>          — Retrieve a value
//   POST /set?key=<key>&value=<val> — Store a value
//   GET  /peers                  — List discovered peers
//   GET  /healthz                — Health check
//
// Environment:
//   POD_NAME         — Set via downward API (e.g., "kvstore-2")
//   SERVICE_NAME     — Headless Service name (e.g., "kvstore-headless")
//   NAMESPACE        — Pod namespace (e.g., "default")
//
// Build:
//   go build -o kvstore main.go
//
// ================================================================

package main

import (
	"context"
	"encoding/json"
	"fmt"
	"hash/fnv"
	"io"
	"log"
	"net"
	"net/http"
	"os"
	"os/signal"
	"regexp"
	"sort"
	"strconv"
	"strings"
	"sync"
	"syscall"
	"time"
)

// ----------------------------------------------------------------
// store is a concurrency-safe, in-memory key-value store.
// In a production system, this would be backed by persistent
// storage (e.g., the StatefulSet's PVC-mounted directory), but
// for clarity we keep it in memory.
// ----------------------------------------------------------------
type store struct {
	mu   sync.RWMutex
	data map[string]string
}

// newStore creates and returns a new, empty key-value store.
func newStore() *store {
	return &store{
		data: make(map[string]string),
	}
}

// Get retrieves the value associated with key. The boolean return
// value indicates whether the key was found.
func (s *store) Get(key string) (string, bool) {
	s.mu.RLock()
	defer s.mu.RUnlock()
	val, ok := s.data[key]
	return val, ok
}

// Set stores a key-value pair, overwriting any existing value.
func (s *store) Set(key, value string) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.data[key] = value
}

// ----------------------------------------------------------------
// peerDiscovery handles DNS-based discovery of peer Pods using
// the headless Service. It periodically resolves the Service
// DNS name and maintains a sorted list of peer hostnames.
// ----------------------------------------------------------------
type peerDiscovery struct {
	// serviceName is the headless Service name (e.g., "kvstore-headless").
	serviceName string

	// namespace is the Kubernetes namespace (e.g., "default").
	namespace string

	// mu protects the peers slice.
	mu sync.RWMutex

	// peers is a sorted list of peer FQDNs discovered via DNS.
	// Sorting ensures consistent ordering across all Pods.
	peers []string
}

// newPeerDiscovery creates a peerDiscovery instance and starts
// the background refresh goroutine. The context controls the
// lifetime of the background goroutine.
func newPeerDiscovery(ctx context.Context, serviceName, namespace string) *peerDiscovery {
	pd := &peerDiscovery{
		serviceName: serviceName,
		namespace:   namespace,
	}
	// Perform an initial synchronous discovery.
	pd.refresh()
	// Start periodic refresh in the background.
	go pd.loop(ctx)
	return pd
}

// refresh performs a DNS SRV lookup on the headless Service to
// discover all peer Pods. It updates the peers list atomically.
func (pd *peerDiscovery) refresh() {
	// The headless Service DNS name follows the pattern:
	// <service>.<namespace>.svc.cluster.local
	target := fmt.Sprintf("%s.%s.svc.cluster.local", pd.serviceName, pd.namespace)

	// Perform a standard DNS lookup (A records) for the headless Service.
	// Each Pod backing the Service will have an A record.
	// However, for hostnames, we perform a SRV lookup.
	_, addrs, err := net.LookupSRV("", "", target)
	if err != nil {
		// SRV lookup may not work in all environments.
		// Fall back to A record lookup.
		log.Printf("[discovery] SRV lookup failed for %s: %v; falling back to A records", target, err)
		ips, err := net.LookupHost(target)
		if err != nil {
			log.Printf("[discovery] A record lookup also failed: %v", err)
			return
		}
		pd.mu.Lock()
		pd.peers = ips
		sort.Strings(pd.peers)
		pd.mu.Unlock()
		log.Printf("[discovery] Discovered %d peers via A records: %v", len(ips), ips)
		return
	}

	// Extract hostnames from SRV records and normalize them.
	peers := make([]string, 0, len(addrs))
	for _, addr := range addrs {
		// SRV targets end with a dot; trim it.
		host := strings.TrimSuffix(addr.Target, ".")
		peers = append(peers, host)
	}
	sort.Strings(peers)

	pd.mu.Lock()
	pd.peers = peers
	pd.mu.Unlock()

	log.Printf("[discovery] Discovered %d peers via SRV: %v", len(peers), peers)
}

// loop periodically calls refresh until the context is cancelled.
func (pd *peerDiscovery) loop(ctx context.Context) {
	ticker := time.NewTicker(10 * time.Second)
	defer ticker.Stop()
	for {
		select {
		case <-ctx.Done():
			return
		case <-ticker.C:
			pd.refresh()
		}
	}
}

// Peers returns a copy of the current peer list.
func (pd *peerDiscovery) Peers() []string {
	pd.mu.RLock()
	defer pd.mu.RUnlock()
	result := make([]string, len(pd.peers))
	copy(result, pd.peers)
	return result
}

// PeerCount returns the number of known peers.
func (pd *peerDiscovery) PeerCount() int {
	pd.mu.RLock()
	defer pd.mu.RUnlock()
	return len(pd.peers)
}

// ----------------------------------------------------------------
// hashKey computes a consistent hash of the given key and returns
// the ordinal of the owning Pod. We use FNV-1a for speed and
// reasonable distribution.
// ----------------------------------------------------------------
func hashKey(key string, totalPeers int) int {
	if totalPeers <= 0 {
		return 0
	}
	h := fnv.New32a()
	// hash.Hash.Write never returns an error; safe to ignore.
	_, _ = h.Write([]byte(key))
	return int(h.Sum32()) % totalPeers
}

// ----------------------------------------------------------------
// extractOrdinal parses the ordinal index from a StatefulSet Pod
// hostname. For example, "kvstore-2" returns 2.
// Returns -1 if the hostname does not match the expected pattern.
// ----------------------------------------------------------------
func extractOrdinal(hostname string) int {
	re := regexp.MustCompile(`-(\d+)$`)
	matches := re.FindStringSubmatch(hostname)
	if len(matches) < 2 {
		return -1
	}
	ordinal, err := strconv.Atoi(matches[1])
	if err != nil {
		return -1
	}
	return ordinal
}

// ----------------------------------------------------------------
// server encapsulates the HTTP server, local key-value store,
// peer discovery, and identity information.
// ----------------------------------------------------------------
type server struct {
	// ordinal is this Pod's ordinal index (e.g., 2 for kvstore-2).
	ordinal int

	// podName is the full Pod name (e.g., "kvstore-2").
	podName string

	// fqdn is this Pod's fully-qualified DNS name.
	fqdn string

	// store is the local in-memory key-value store.
	store *store

	// discovery manages peer discovery via DNS.
	discovery *peerDiscovery

	// httpClient is used for proxying requests to peer Pods.
	httpClient *http.Client
}

// newServer creates a server instance from environment variables.
// It extracts the Pod's ordinal, constructs the FQDN, and
// initializes peer discovery.
func newServer(ctx context.Context) (*server, error) {
	podName := os.Getenv("POD_NAME")
	if podName == "" {
		return nil, fmt.Errorf("POD_NAME environment variable is required")
	}

	serviceName := os.Getenv("SERVICE_NAME")
	if serviceName == "" {
		serviceName = "kvstore-headless"
	}

	namespace := os.Getenv("NAMESPACE")
	if namespace == "" {
		namespace = "default"
	}

	ordinal := extractOrdinal(podName)
	if ordinal < 0 {
		return nil, fmt.Errorf("could not extract ordinal from hostname %q", podName)
	}

	fqdn := fmt.Sprintf("%s.%s.%s.svc.cluster.local", podName, serviceName, namespace)

	return &server{
		ordinal:   ordinal,
		podName:   podName,
		fqdn:      fqdn,
		store:     newStore(),
		discovery: newPeerDiscovery(ctx, serviceName, namespace),
		httpClient: &http.Client{
			Timeout: 5 * time.Second,
		},
	}, nil
}

// handleGet processes GET /get?key=<key> requests. If this Pod
// owns the key (based on hash sharding), it serves the value
// from its local store. Otherwise, it proxies the request to
// the owning peer.
func (s *server) handleGet(w http.ResponseWriter, r *http.Request) {
	key := r.URL.Query().Get("key")
	if key == "" {
		http.Error(w, `{"error":"missing 'key' parameter"}`, http.StatusBadRequest)
		return
	}

	totalPeers := s.discovery.PeerCount()
	owner := hashKey(key, totalPeers)

	// If we own this key, serve it locally.
	if owner == s.ordinal {
		val, ok := s.store.Get(key)
		if !ok {
			http.Error(w, fmt.Sprintf(`{"error":"key %q not found"}`, key), http.StatusNotFound)
			return
		}
		writeJSON(w, http.StatusOK, map[string]string{
			"key":       key,
			"value":     val,
			"served_by": s.podName,
		})
		return
	}

	// Proxy to the owning peer.
	s.proxyToOwner(w, r, owner, "/get", fmt.Sprintf("key=%s", key))
}

// handleSet processes POST /set?key=<key>&value=<val> requests.
// Like handleGet, it routes to the owning Pod.
func (s *server) handleSet(w http.ResponseWriter, r *http.Request) {
	key := r.URL.Query().Get("key")
	value := r.URL.Query().Get("value")
	if key == "" || value == "" {
		http.Error(w, `{"error":"missing 'key' or 'value' parameter"}`, http.StatusBadRequest)
		return
	}

	totalPeers := s.discovery.PeerCount()
	owner := hashKey(key, totalPeers)

	if owner == s.ordinal {
		s.store.Set(key, value)
		writeJSON(w, http.StatusOK, map[string]string{
			"status":    "ok",
			"key":       key,
			"stored_by": s.podName,
		})
		return
	}

	s.proxyToOwner(w, r, owner, "/set", fmt.Sprintf("key=%s&value=%s", key, value))
}

// proxyToOwner forwards a request to the peer Pod that owns the
// key. It constructs the peer's FQDN from the ordinal and makes
// an HTTP request.
func (s *server) proxyToOwner(w http.ResponseWriter, _ *http.Request, ownerOrdinal int, path, query string) {
	// Construct the owner's FQDN from the StatefulSet naming convention.
	// Pod name = <statefulset-name>-<ordinal>
	// We extract the StatefulSet name by removing the ordinal suffix from our own name.
	ssName := s.podName[:len(s.podName)-len(fmt.Sprintf("-%d", s.ordinal))]
	serviceName := os.Getenv("SERVICE_NAME")
	if serviceName == "" {
		serviceName = "kvstore-headless"
	}
	namespace := os.Getenv("NAMESPACE")
	if namespace == "" {
		namespace = "default"
	}

	ownerFQDN := fmt.Sprintf("%s-%d.%s.%s.svc.cluster.local",
		ssName, ownerOrdinal, serviceName, namespace)
	url := fmt.Sprintf("http://%s:8080%s?%s", ownerFQDN, path, query)

	log.Printf("[proxy] Forwarding to %s (owner ordinal %d)", ownerFQDN, ownerOrdinal)

	resp, err := s.httpClient.Get(url)
	if err != nil {
		http.Error(w, fmt.Sprintf(`{"error":"proxy failed: %v"}`, err), http.StatusBadGateway)
		return
	}
	defer resp.Body.Close()

	// Copy the response from the owning peer back to the client.
	w.Header().Set("Content-Type", "application/json")
	w.Header().Set("X-Proxied-From", s.podName)
	w.Header().Set("X-Proxied-To", ownerFQDN)
	w.WriteHeader(resp.StatusCode)
	_, _ = io.Copy(w, resp.Body)
}

// handlePeers returns the list of currently discovered peers.
func (s *server) handlePeers(w http.ResponseWriter, _ *http.Request) {
	writeJSON(w, http.StatusOK, map[string]interface{}{
		"self":    s.podName,
		"ordinal": s.ordinal,
		"fqdn":    s.fqdn,
		"peers":   s.discovery.Peers(),
	})
}

// handleHealth returns a simple health check response.
func (s *server) handleHealth(w http.ResponseWriter, _ *http.Request) {
	writeJSON(w, http.StatusOK, map[string]string{"status": "healthy"})
}

// writeJSON is a helper that serializes data as JSON and writes
// it to the ResponseWriter with the appropriate Content-Type.
func writeJSON(w http.ResponseWriter, status int, data interface{}) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	enc := json.NewEncoder(w)
	enc.SetIndent("", "  ")
	_ = enc.Encode(data)
}

// ----------------------------------------------------------------
// main sets up the HTTP server, registers handlers, and starts
// listening. It also handles graceful shutdown on SIGTERM/SIGINT,
// which is important in Kubernetes where Pods receive SIGTERM
// during termination.
// ----------------------------------------------------------------
func main() {
	log.SetFlags(log.Ldate | log.Ltime | log.Lshortfile)
	log.Println("Starting distributed key-value store...")

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	srv, err := newServer(ctx)
	if err != nil {
		log.Fatalf("Failed to create server: %v", err)
	}

	mux := http.NewServeMux()
	mux.HandleFunc("/get", srv.handleGet)
	mux.HandleFunc("/set", srv.handleSet)
	mux.HandleFunc("/peers", srv.handlePeers)
	mux.HandleFunc("/healthz", srv.handleHealth)

	httpServer := &http.Server{
		Addr:         ":8080",
		Handler:      mux,
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 10 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	// Graceful shutdown: listen for termination signals.
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)

	go func() {
		sig := <-sigCh
		log.Printf("Received signal %v; shutting down gracefully...", sig)
		cancel()
		shutdownCtx, shutdownCancel := context.WithTimeout(context.Background(), 15*time.Second)
		defer shutdownCancel()
		if err := httpServer.Shutdown(shutdownCtx); err != nil {
			log.Printf("HTTP server shutdown error: %v", err)
		}
	}()

	log.Printf("Pod %s (ordinal %d) listening on :8080", srv.podName, srv.ordinal)
	if err := httpServer.ListenAndServe(); err != http.ErrServerClosed {
		log.Fatalf("HTTP server error: %v", err)
	}
	log.Println("Server stopped.")
}
```

**Kubernetes manifests for the key-value store:**

```yaml
# ------------------------------------------------------------------
# kvstore-headless-service.yaml
# Headless Service for the distributed key-value store.
# Provides stable DNS names for each Pod.
# ------------------------------------------------------------------
apiVersion: v1
kind: Service
metadata:
  name: kvstore-headless
spec:
  clusterIP: None
  selector:
    app: kvstore
  ports:
    - port: 8080
      name: http
      targetPort: 8080

---
# ------------------------------------------------------------------
# kvstore-client-service.yaml
# Regular ClusterIP Service for client access. Clients connect to
# any Pod; the application handles routing internally.
# ------------------------------------------------------------------
apiVersion: v1
kind: Service
metadata:
  name: kvstore
spec:
  selector:
    app: kvstore
  ports:
    - port: 80
      name: http
      targetPort: 8080

---
# ------------------------------------------------------------------
# kvstore-statefulset.yaml
# StatefulSet for the distributed key-value store.
# Each Pod gets a stable identity and persistent storage.
# ------------------------------------------------------------------
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: kvstore
spec:
  serviceName: kvstore-headless
  replicas: 3
  selector:
    matchLabels:
      app: kvstore
  podManagementPolicy: Parallel  # Our app handles coordination via hashing
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: kvstore
    spec:
      containers:
        - name: kvstore
          image: myregistry/kvstore:latest
          ports:
            - containerPort: 8080
              name: http
          env:
            # ---------------------------------------------------
            # POD_NAME: Injected via the downward API so the app
            # can determine its ordinal index from its hostname.
            # ---------------------------------------------------
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: SERVICE_NAME
              value: "kvstore-headless"
            - name: NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 10
          volumeMounts:
            - name: data
              mountPath: /data
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 256Mi
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: standard
        resources:
          requests:
            storage: 1Gi
```

**Testing the distributed key-value store:**

```bash
# Deploy everything
kubectl apply -f kvstore-headless-service.yaml
kubectl apply -f kvstore-client-service.yaml
kubectl apply -f kvstore-statefulset.yaml

# Wait for all Pods
kubectl wait --for=condition=Ready pod -l app=kvstore --timeout=120s

# Store a value (may be proxied to the owning Pod)
kubectl exec kvstore-0 -- wget -qO- 'http://localhost:8080/set?key=greeting&value=hello'
# {"key":"greeting","stored_by":"kvstore-1","status":"ok"}

# Retrieve it (from any Pod — routing is transparent)
kubectl exec kvstore-2 -- wget -qO- 'http://localhost:8080/get?key=greeting'
# {"key":"greeting","value":"hello","served_by":"kvstore-1"}

# List peers
kubectl exec kvstore-0 -- wget -qO- http://localhost:8080/peers
# {"self":"kvstore-0","ordinal":0,"fqdn":"kvstore-0.kvstore-headless.default.svc.cluster.local","peers":["kvstore-0...","kvstore-1...","kvstore-2..."]}
```

---

## 19.8 Scaling StatefulSets

### 19.8.1 Scaling Up

When you increase the replica count, the StatefulSet controller creates new
Pods in ascending ordinal order:

```bash
# Scale from 3 to 5 replicas
kubectl scale statefulset web --replicas=5

# The controller will create:
# 1. web-3 → waits until Running+Ready
# 2. web-4 → waits until Running+Ready (with OrderedReady policy)
```

If PVCs already exist for the new ordinals (from a previous scale-down), the
new Pods reattach to them automatically. This is how data survives scale
down/up cycles.

### 19.8.2 Scaling Down

When you decrease the replica count, Pods are deleted in reverse ordinal order:

```bash
# Scale from 5 to 3 replicas
kubectl scale statefulset web --replicas=3

# The controller will delete:
# 1. web-4 → waits until fully terminated
# 2. web-3 → waits until fully terminated (with OrderedReady policy)
```

**PVCs are retained.** After scaling down:

```bash
kubectl get pvc -l app=web
# data-web-0   Bound   ...
# data-web-1   Bound   ...
# data-web-2   Bound   ...
# data-web-3   Bound   ...  ← Still exists!
# data-web-4   Bound   ...  ← Still exists!
```

### 19.8.3 Hands-on: Scaling and Data Persistence

```bash
# Start with 3 replicas
kubectl apply -f web-statefulset.yaml

# Write unique data to each Pod
for i in 0 1 2; do
  kubectl exec web-$i -- sh -c "echo 'data from web-$i' > /usr/share/nginx/html/identity.txt"
done

# Scale down to 1 replica
kubectl scale statefulset web --replicas=1

# Verify only web-0 is running
kubectl get pods -l app=web
# NAME    READY   STATUS    RESTARTS   AGE
# web-0   1/1     Running   0          5m

# Scale back up to 3
kubectl scale statefulset web --replicas=3

# Wait for Pods to be ready
kubectl wait --for=condition=Ready pod/web-1 pod/web-2 --timeout=120s

# Verify data survived the scale-down/up cycle
for i in 0 1 2; do
  echo "--- web-$i ---"
  kubectl exec web-$i -- cat /usr/share/nginx/html/identity.txt
done
# --- web-0 ---
# data from web-0
# --- web-1 ---
# data from web-1
# --- web-2 ---
# data from web-2
```

---

## 19.9 Updating StatefulSets

### 19.9.1 RollingUpdate Strategy

The default `RollingUpdate` strategy updates Pods in **reverse ordinal order**:

```
web-2 → web-1 → web-0
```

For each Pod:
1. The controller deletes the Pod.
2. The controller recreates it with the updated spec.
3. The controller waits until the new Pod is Running+Ready.
4. Only then does it proceed to the next Pod.

This reverse-order update is deliberate: in many stateful systems, the
lowest-ordinal Pod is the leader or primary. Updating it last minimizes
disruption.

### 19.9.2 Partition — Staged Rollouts

The `partition` field in `rollingUpdate` is a powerful mechanism for canary
deployments of stateful workloads:

```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    partition: 2  # Only update Pods with ordinal >= 2
```

With `partition: 2` and 3 replicas (web-0, web-1, web-2):
- **web-2** is updated immediately (ordinal 2 ≥ partition 2).
- **web-1** is NOT updated (ordinal 1 < partition 2).
- **web-0** is NOT updated (ordinal 0 < partition 2).

This lets you update a single Pod, validate that it works correctly, and then
lower the partition to roll out to more Pods:

```bash
# Stage 1: Update only web-2
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'
kubectl set image statefulset/web web=nginx:1.26

# Validate web-2 is healthy with the new version
kubectl exec web-2 -- nginx -v

# Stage 2: Update web-1 and web-2
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":1}}}}'

# Stage 3: Update all Pods
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
```

### 19.9.3 OnDelete Strategy

With `OnDelete`, the controller never automatically updates Pods. You must
manually delete each Pod to trigger recreation with the new spec:

```yaml
updateStrategy:
  type: OnDelete
```

```bash
# Update the StatefulSet image
kubectl set image statefulset/web web=nginx:1.26

# Nothing happens — all Pods still run the old version
kubectl get pods -l app=web -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.containers[0].image}{"\n"}{end}'
# web-0: nginx:1.25
# web-1: nginx:1.25
# web-2: nginx:1.25

# Manually trigger update for web-2
kubectl delete pod web-2

# web-2 is recreated with the new image
# web-0 and web-1 still run the old version
```

`OnDelete` is useful when:
- You need to perform manual validation between each node update.
- Your stateful application requires a specific update procedure (e.g.,
  database schema migration before updating the next node).
- You want an operator or external controller to manage the update sequence.

### 19.9.4 Hands-on: Partitioned Rolling Update

```bash
# Deploy the StatefulSet with 5 replicas
kubectl apply -f web-statefulset.yaml
kubectl scale statefulset web --replicas=5
kubectl wait --for=condition=Ready pod -l app=web --timeout=120s

# Set partition to 4 (only update web-4)
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":4}}}}'

# Update the image
kubectl set image statefulset/web web=nginx:1.26

# Verify: only web-4 has the new image
kubectl get pods -l app=web -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.containers[0].image}{"\n"}{end}'
# web-0: nginx:1.25
# web-1: nginx:1.25
# web-2: nginx:1.25
# web-3: nginx:1.25
# web-4: nginx:1.26  ← Updated!

# Lower partition to 2 (update web-2, web-3, web-4)
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":2}}}}'

# Verify: web-2, web-3, web-4 have the new image
kubectl get pods -l app=web -o jsonpath='{range .items[*]}{.metadata.name}: {.spec.containers[0].image}{"\n"}{end}'
# web-0: nginx:1.25
# web-1: nginx:1.25
# web-2: nginx:1.26  ← Updated!
# web-3: nginx:1.26  ← Updated!
# web-4: nginx:1.26  ← Updated!

# Complete the rollout
kubectl patch statefulset web -p '{"spec":{"updateStrategy":{"rollingUpdate":{"partition":0}}}}'
```

---

## 19.10 StatefulSet vs Deployment — When to Use What

### 19.10.1 Decision Matrix

| Criterion                       | Deployment           | StatefulSet          |
|---------------------------------|----------------------|----------------------|
| Pods are interchangeable        | ✅ Yes               | ❌ No                |
| Needs stable network identity   | ❌ No                | ✅ Yes               |
| Needs stable persistent storage | ❌ No                | ✅ Yes               |
| Needs ordered startup/shutdown  | ❌ No                | ✅ Yes               |
| Needs per-Pod configuration     | ❌ No                | ✅ Yes               |
| Scaling speed                   | Fast (parallel)      | Slow (sequential)*   |
| Update speed                    | Fast (parallel)      | Slow (sequential)    |
| Complexity                      | Low                  | Higher               |

\* Unless using `podManagementPolicy: Parallel`.

### 19.10.2 Common Mistakes

**Mistake 1: Using a Deployment for a database.**

```yaml
# ❌ WRONG: Database as a Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: postgres
          image: postgres:16
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: postgres-data  # All 3 Pods share one PVC!
```

Problems:
- All three Pods try to write to the same PVC → data corruption.
- Pod names are random → no way to designate a primary.
- No ordered startup → split-brain on bootstrap.

**Mistake 2: Using a StatefulSet for a stateless API.**

```yaml
# ❌ OVERKILL: Stateless API as StatefulSet
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: api-server
spec:
  serviceName: api-headless
  replicas: 10
  # ...
```

Problems:
- Sequential startup means slow scaling (10 Pods take 10× longer).
- PVCs are created unnecessarily.
- Rolling updates are slow (one at a time instead of rolling percentage).

**Mistake 3: Forgetting the headless Service.**

```yaml
# StatefulSet references a Service that doesn't exist
spec:
  serviceName: "my-headless-service"  # But this Service was never created!
```

The StatefulSet will create Pods, but DNS resolution for individual Pods will
fail. Your application's peer discovery will be broken.

### 19.10.3 The Gray Area

Some workloads fall in between:

- **Caching layers (Redis standalone, Memcached):** Often fine with
  Deployments if data loss on restart is acceptable.
- **Batch processors:** Usually Deployments (or Jobs), unless they need
  stable identities for work partitioning.
- **Application servers with local caches:** Usually Deployments, but a
  StatefulSet can help if you want sticky sessions by hostname.

---

## 19.11 The Operator Pattern for Stateful Workloads

### 19.11.1 Why Operators Exist

StatefulSets provide the *infrastructure* for stateful workloads —
stable identity, stable storage, ordered operations — but they do not provide
*application-level* intelligence. Consider what a production database needs
beyond what a StatefulSet offers:

| Requirement                  | StatefulSet Provides? | Operator Provides? |
|------------------------------|----------------------|-------------------|
| Stable Pod identity          | ✅                   | ✅                |
| Persistent storage           | ✅                   | ✅                |
| Ordered scaling              | ✅                   | ✅                |
| Automated failover           | ❌                   | ✅                |
| Backup and restore           | ❌                   | ✅                |
| Schema migrations            | ❌                   | ✅                |
| Connection pooling           | ❌                   | ✅                |
| Monitoring and alerting      | ❌                   | ✅                |
| Multi-cluster replication    | ❌                   | ✅                |

An **operator** is a custom controller that encodes human operational knowledge
into software. It watches Custom Resources (CRDs) and reconciles the actual
state of a stateful application with the desired state — performing tasks that
would otherwise require a human DBA or sysadmin.

### 19.11.2 Notable Operators

| Operator                                 | Stateful Workload   |
|------------------------------------------|---------------------|
| **CloudNativePG** (cnpg.io)              | PostgreSQL          |
| **Crunchy Data PGO**                     | PostgreSQL          |
| **Percona Operator for MySQL**           | MySQL               |
| **MongoDB Community Operator**           | MongoDB             |
| **Strimzi**                              | Apache Kafka        |
| **Redis Operator** (OpsTree Solutions)   | Redis               |
| **etcd Operator**                        | etcd                |
| **Rook**                                 | Ceph (storage)      |
| **Elasticsearch Operator** (ECK)         | Elasticsearch       |

### 19.11.3 The Relationship Between Operators and StatefulSets

Most database operators **use StatefulSets internally**. The operator creates
and manages StatefulSets (and their associated Services, ConfigMaps, Secrets,
etc.) on behalf of the user. The user interacts with a high-level CRD:

```yaml
# User creates this:
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: my-postgres
spec:
  instances: 3
  storage:
    size: 10Gi

# The operator creates (among other things):
# - StatefulSet "my-postgres" with 3 replicas
# - Headless Service "my-postgres-headless"
# - ClusterIP Service "my-postgres-rw" (primary)
# - ClusterIP Service "my-postgres-ro" (replicas)
# - PVCs, ConfigMaps, Secrets, ServiceMonitors, etc.
```

We will dive deep into building operators in **Chapters 32–35**.

---

## 19.12 Debugging StatefulSets

### 19.12.1 Common Issues and Solutions

**Issue: Pod stuck in Pending.**

```bash
kubectl describe pod web-1
# Events:
#   Warning  FailedScheduling  pod/web-1  0/3 nodes are available:
#   3 pod has unbound immediate PersistentVolumeClaims

# Cause: No PV available for the PVC.
# Solution: Check StorageClass and PV availability.
kubectl get pvc data-web-1
kubectl get storageclass
kubectl get pv
```

**Issue: Pod web-1 never starts because web-0 is not Ready.**

```bash
# With OrderedReady policy, web-1 won't be created until web-0 is Ready.
kubectl get pods -l app=web
# NAME    READY   STATUS             RESTARTS   AGE
# web-0   0/1     CrashLoopBackOff   3          5m

# Fix web-0 first. Check logs:
kubectl logs web-0
kubectl describe pod web-0
```

**Issue: DNS not resolving for individual Pods.**

```bash
# Verify the headless Service exists and selects the right Pods
kubectl get svc web-headless
kubectl get endpoints web-headless

# If endpoints are empty, the label selector doesn't match Pod labels.
```

**Issue: PVC from a previous deployment is reattached with stale data.**

```bash
# List PVCs and check their creation time
kubectl get pvc -l app=web -o custom-columns=NAME:.metadata.name,CREATED:.metadata.creationTimestamp

# Delete stale PVCs before redeploying
kubectl delete pvc data-web-0 data-web-1 data-web-2
```

### 19.12.2 Useful Commands

```bash
# View StatefulSet status
kubectl get statefulset web -o wide

# Detailed status including conditions and revision
kubectl describe statefulset web

# Watch Pod creation/deletion in real-time
kubectl get pods -l app=web -w

# Check which revision each Pod is running
kubectl get pods -l app=web -o jsonpath='{range .items[*]}{.metadata.name}: {.metadata.labels.controller-revision-hash}{"\n"}{end}'

# View StatefulSet events
kubectl events --for statefulset/web

# Check PVC status
kubectl get pvc -l app=web
```

---

## 19.13 Production Checklist for StatefulSets

Before deploying a StatefulSet to production, verify:

- [ ] **Headless Service** exists and selects the correct Pods.
- [ ] **StorageClass** supports the required access mode (`ReadWriteOnce` for
      most databases).
- [ ] **PVC size** is adequate and the StorageClass supports volume expansion
      (if needed).
- [ ] **Readiness probes** are configured — they gate Pod ordering.
- [ ] **Liveness probes** are configured — but with generous timeouts for
      stateful workloads (databases can take minutes to start).
- [ ] **Resource requests and limits** are set — stateful workloads are
      typically memory-intensive.
- [ ] **Pod Disruption Budget** is configured to prevent too many Pods from
      being evicted simultaneously.
- [ ] **Anti-affinity rules** spread Pods across Nodes/zones for HA.
- [ ] **Backup strategy** is in place (PVCs are not backups!).
- [ ] **Monitoring** tracks per-Pod metrics (not just aggregate).
- [ ] **`terminationGracePeriodSeconds`** is long enough for graceful shutdown
      (databases may need 60–300 seconds).

```yaml
# Example: Pod Disruption Budget for a StatefulSet
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: web-pdb
spec:
  # At most 1 Pod can be unavailable during voluntary disruptions.
  maxUnavailable: 1
  selector:
    matchLabels:
      app: web
```

```yaml
# Example: Anti-affinity to spread Pods across Nodes
spec:
  template:
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - web
                topologyKey: kubernetes.io/hostname
```

---

## 19.14 Summary

StatefulSets are the Kubernetes primitive for workloads that need **identity**,
**storage**, and **ordering**. They complement Deployments — not replace them —
by addressing the unique requirements of databases, caches, and consensus
systems.

In this chapter, you learned:

1. **Why StatefulSets exist:** Deployments treat Pods as cattle; StatefulSets
   treat them as pets — each with a name, a home (PVC), and an address (DNS).

2. **The four guarantees:** Stable Pod names, stable DNS via headless Service,
   stable per-Pod storage, and ordered lifecycle operations.

3. **How the controller works:** Serial Pod creation/deletion with
   `OrderedReady`, level-triggered reconciliation, and the sticky identity
   model.

4. **Headless Services:** The DNS backbone that gives each Pod a resolvable
   name.

5. **PVC lifecycle:** Volumes survive Pod restarts, rescheduling, scale-down,
   and even StatefulSet deletion (by default).

6. **Real-world deployments:** Redis Cluster, PostgreSQL primary-replica, and
   a custom distributed key-value store in Go.

7. **Scaling and updating:** Ordered scale-up/down, partitioned rolling
   updates, and the OnDelete strategy for manual control.

8. **When to use what:** StatefulSets for stateful workloads, Deployments for
   stateless ones, and operators for production-grade stateful applications.

In the next chapter, we turn to two more workload controllers: **DaemonSets**
(for running exactly one Pod per Node) and **Jobs/CronJobs** (for
run-to-completion and scheduled workloads).

---

## 19.15 Exercises

1. **Headless Service DNS:** Deploy a 3-replica StatefulSet with a headless
   Service. From a debug Pod, resolve the headless Service name and each
   individual Pod name. Explain the difference between querying the Service
   vs. a specific Pod.

2. **PVC persistence:** Deploy a StatefulSet with `volumeClaimTemplates`.
   Write unique data to each Pod's volume. Scale down to 1 replica, then back
   to 3. Verify that data persisted for all three Pods.

3. **Partitioned update:** Deploy a 5-replica StatefulSet. Use partitioned
   rolling updates to canary a new image to only the highest-ordinal Pod.
   Validate it, then roll out to all Pods.

4. **Parallel policy:** Modify the StatefulSet to use
   `podManagementPolicy: Parallel`. Observe how all Pods start simultaneously.
   Discuss when this is safe and when it is not.

5. **Operator exploration:** Install the CloudNativePG operator and create a
   3-instance PostgreSQL cluster using its `Cluster` CRD. Examine what
   Kubernetes resources the operator creates. Compare this to the manual
   PostgreSQL StatefulSet from Section 19.7.2.

6. **Build an operator (preview):** Extend the distributed key-value store
   from Section 19.7.3 with automatic rebalancing when the replica count
   changes. This is a preview of the operator pattern covered in Chapters
   32–35.
