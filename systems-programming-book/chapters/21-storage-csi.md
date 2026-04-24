# Chapter 21: Persistent Volumes, Storage Classes & CSI

> *"Data is the new oil, but without persistent storage in Kubernetes, your oil evaporates every time a container restarts."*

Storage is one of the most complex subsystems in Kubernetes. Containers were designed
to be ephemeral — lightweight, disposable, and stateless. But real-world applications
need databases, file uploads, configuration state, and logs that survive container
restarts, Pod rescheduling, and even node failures. This chapter takes you from the
fundamental problem of container storage through the full Kubernetes storage stack:
Volumes, Persistent Volumes, StorageClasses, and the Container Storage Interface (CSI)
that makes it all extensible.

By the end of this chapter, you will understand how storage works at every layer —
from the kubelet mounting a filesystem into a container's mount namespace, to a CSI
driver calling a cloud provider API to provision a block device, to the kernel
completing an NFS read on a remote server.

---

## The Storage Problem in Kubernetes

### Containers Are Ephemeral

Let's start with first principles. A container's filesystem is built from a layered
image — a stack of read-only layers with a thin writable layer on top (the "upper"
layer in an OverlayFS). When the container process exits, that writable layer is
discarded. Everything written during the container's lifetime — log files, database
records, uploaded images, temporary computations — vanishes.

```text
┌─────────────────────────────────┐
│   Writable Layer (container)    │ ← Discarded on container restart
├─────────────────────────────────┤
│   Read-Only Layer 3 (app code)  │
├─────────────────────────────────┤
│   Read-Only Layer 2 (packages)  │
├─────────────────────────────────┤
│   Read-Only Layer 1 (base OS)   │
└─────────────────────────────────┘
         Container Filesystem
```

This is by design. Immutable infrastructure means we rebuild rather than patch. But
it creates a fundamental tension: **applications need state**.

Consider a PostgreSQL database. If the container running PostgreSQL restarts — due to
an OOM kill, a node failure, or a rolling update — every row in every table is gone.
The database might as well have never existed.

### The Naive Solutions and Their Limitations

Kubernetes provides several built-in volume types that address simple use cases:

**`emptyDir`**: A directory created when a Pod is assigned to a node, shared among
all containers in that Pod, and deleted when the Pod is removed from the node. It
solves container-to-container communication within a single Pod, but the data dies
with the Pod.

**`hostPath`**: A directory on the host node's filesystem mounted into the Pod. The
data survives container restarts (since it lives on the host), but it's pinned to a
specific node. If the Pod is rescheduled to a different node, the data is gone. Worse,
`hostPath` is a security risk — a misconfigured `hostPath` can give a container access
to the host's `/etc`, `/var/run/docker.sock`, or even `/`.

Neither of these solves the real problem: **persistent, network-attached, provider-
agnostic storage that survives Pod rescheduling across nodes**.

### What We Need

A proper storage solution for Kubernetes must:

1. **Persist beyond Pod lifetime** — data survives Pod deletion and rescheduling
2. **Be network-attached** — accessible from any node in the cluster
3. **Be provider-agnostic** — the same PVC YAML works on AWS (EBS), GCP (Persistent
   Disk), Azure (Managed Disk), or bare metal (Ceph, NFS)
4. **Support dynamic provisioning** — storage is created on-demand, not pre-allocated
5. **Support access modes** — some storage can be shared (NFS), some cannot (block)
6. **Support snapshots and expansion** — operational necessities

Kubernetes achieves this through the **PersistentVolume** subsystem and the
**Container Storage Interface (CSI)**.

---

## Volume Types

Before we dive into PersistentVolumes, let's catalog the volume types available in
Kubernetes. Understanding these is essential — even in a fully CSI-driven cluster,
you'll use `emptyDir` for scratch space and `configMap`/`secret` for configuration.

### emptyDir — Shared Temporary Storage

An `emptyDir` volume is created when a Pod is assigned to a node. It starts empty.
All containers in the Pod can read and write the same files in the `emptyDir` volume.
When the Pod is removed from the node — for any reason — the data in the `emptyDir`
is deleted permanently.

**Use cases:**
- Scratch space for disk-based merge sort
- Checkpointing a long computation for crash recovery
- Holding files that a content-manager container fetches for a web-server container

**Medium options:**
- Default: backed by the node's disk (whatever the kubelet root directory uses)
- `medium: Memory`: backed by a tmpfs (RAM-backed filesystem). Fast, but counts
  against the container's memory limit.

```yaml
# emptyDir-example.yaml
# Demonstrates emptyDir shared between two containers.
# The 'writer' container writes a timestamp every second.
# The 'reader' container tails the file continuously.
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
  # First container: writes timestamps to a shared file
  - name: writer
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      while true; do
        echo "$(date): Hello from writer" >> /data/log.txt
        sleep 1
      done
    volumeMounts:
    - name: shared-data
      mountPath: /data

  # Second container: reads the shared file
  - name: reader
    image: busybox:1.36
    command:
    - sh
    - -c
    - tail -f /data/log.txt
    volumeMounts:
    - name: shared-data
      mountPath: /data

  volumes:
  # emptyDir volume — created when Pod starts, deleted when Pod dies
  - name: shared-data
    emptyDir: {}
```

```yaml
# emptyDir with tmpfs (RAM-backed)
# WARNING: tmpfs counts against the container's memory limit.
# If the container writes 500Mi to tmpfs and has a 256Mi memory limit,
# the container will be OOM-killed.
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-tmpfs
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "3600"]
    volumeMounts:
    - name: cache
      mountPath: /cache
    resources:
      limits:
        memory: "512Mi"
  volumes:
  - name: cache
    emptyDir:
      medium: Memory
      sizeLimit: 256Mi    # Limit tmpfs size to prevent OOM
```

**What happens on disk:**

When a Pod with an `emptyDir` is created, the kubelet creates a directory under:
```
/var/lib/kubelet/pods/<pod-uid>/volumes/kubernetes.io~empty-dir/<volume-name>/
```

For `medium: Memory`, the kubelet mounts a tmpfs at that path instead.

### hostPath — Mount Host Directory

`hostPath` mounts a file or directory from the host node's filesystem into the Pod.
The data persists across container restarts (since it's on the host), but it's
**pinned to that specific node**.

```yaml
# hostPath-example.yaml
# Mounts the host's /var/log directory into the container.
# WARNING: hostPath is dangerous in production. A container with access
# to the host filesystem can escape its sandbox.
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-demo
spec:
  containers:
  - name: log-reader
    image: busybox:1.36
    command: ["sh", "-c", "ls -la /host-logs && sleep 3600"]
    volumeMounts:
    - name: host-logs
      mountPath: /host-logs
      readOnly: true          # ALWAYS use readOnly for hostPath
  volumes:
  - name: host-logs
    hostPath:
      path: /var/log          # Path on the host
      type: Directory         # Must be an existing directory
```

**hostPath types:**
| Type                | Behavior                                           |
|---------------------|----------------------------------------------------|
| `""`                | No check before mounting (default)                 |
| `DirectoryOrCreate` | Create directory if it doesn't exist               |
| `Directory`         | Must be an existing directory                      |
| `FileOrCreate`      | Create file if it doesn't exist                    |
| `File`              | Must be an existing file                           |
| `Socket`            | Must be an existing Unix socket                    |
| `CharDevice`        | Must be an existing character device               |
| `BlockDevice`       | Must be an existing block device                   |

**Why hostPath is dangerous:**

1. **Security**: A container with `hostPath: /` has access to the entire host. It can
   read `/etc/shadow`, write to `/etc/crontab`, or access the container runtime socket.
2. **Scheduling**: Data is local to one node. If the Pod moves, the data doesn't.
3. **Multi-tenancy**: Different Pods on the same node could access each other's data.

In production, hostPath should be restricted by PodSecurityAdmission policies. The
only legitimate uses are for DaemonSets that need access to node resources (log
collectors, monitoring agents, CSI node plugins).

### configMap, secret, downwardAPI — Projected Volumes

These volume types inject configuration data into Pods:

```yaml
# Projected volume combining configMap, secret, and downwardAPI
apiVersion: v1
kind: Pod
metadata:
  name: projected-demo
  labels:
    app: myapp
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "3600"]
    volumeMounts:
    - name: all-config
      mountPath: /etc/config
      readOnly: true
  volumes:
  - name: all-config
    projected:
      sources:
      # ConfigMap data appears as files
      - configMap:
          name: app-config
          items:
          - key: database.conf
            path: database.conf
      # Secret data appears as files (base64-decoded)
      - secret:
          name: app-secrets
          items:
          - key: tls.crt
            path: tls/cert.pem
          - key: tls.key
            path: tls/key.pem
      # Downward API — Pod metadata as files
      - downwardAPI:
          items:
          - path: labels
            fieldRef:
              fieldPath: metadata.labels
          - path: cpu-limit
            resourceFieldRef:
              containerName: app
              resource: limits.cpu
```

Under the hood, the kubelet watches these resources and updates the mounted files
when the underlying ConfigMap or Secret changes (with a configurable delay, default
~60 seconds). This uses an atomic symlink swap — the mount point is a symlink to a
timestamped directory, and the kubelet creates a new directory with updated content
then atomically swaps the symlink.

### persistentVolumeClaim — The Standard

This is where we're heading — `persistentVolumeClaim` is the volume type that
references a PVC, which in turn binds to a PV. We'll cover this in detail in the
next section.

```yaml
volumes:
- name: database-storage
  persistentVolumeClaim:
    claimName: postgres-data     # References a PVC in the same namespace
```

### Legacy In-Tree Volume Types: nfs, iscsi, cephfs

Kubernetes historically included volume plugins compiled directly into the kubelet
and controller-manager binaries — these are called "in-tree" plugins. Examples include
`nfs`, `iscsi`, `cephfs`, `awsElasticBlockStore`, `gcePersistentDisk`, and
`azureDisk`.

These are all **deprecated** in favor of CSI drivers. The Kubernetes project is
actively migrating all in-tree plugins to out-of-tree CSI drivers through the
CSI Migration feature. As of Kubernetes 1.29+, most in-tree plugins are either
migrated or in the process of migration.

**Do not use in-tree volume types for new deployments.** Use the corresponding
CSI driver instead.

### csi — The Modern Extensible Interface

The `csi` volume type allows any third-party storage provider to expose storage to
Kubernetes workloads through a standard gRPC interface. We'll deep-dive into CSI
later in this chapter.

---

## Persistent Volumes (PV) and Claims (PVC)

The PersistentVolume subsystem is Kubernetes' answer to the storage problem. It
provides an API that abstracts the details of how storage is provided from how it
is consumed.

### The Two Objects

**PersistentVolume (PV)**: A cluster-level resource that represents a piece of
storage. It could be a physical disk in a data center, a cloud block device, an NFS
export, or a Ceph RBD image. PVs are provisioned by a cluster administrator or
dynamically by a StorageClass.

PVs are **not namespaced** — they are cluster-scoped resources, like Nodes.

**PersistentVolumeClaim (PVC)**: A namespaced resource that represents a user's
request for storage. A PVC specifies the desired capacity, access modes, and
StorageClass. The Kubernetes control plane finds a matching PV and binds them
together.

Think of it like this:
- **PV** = the actual storage (a disk, an NFS share)
- **PVC** = a ticket that says "I need 10Gi of ReadWriteOnce storage"

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                       │
│                                                                 │
│  ┌──────────────────┐           ┌──────────────────────┐        │
│  │   PV (10Gi, RWO) │◄─────────│   PVC (10Gi, RWO)    │        │
│  │   NFS Server     │  Binding  │   Namespace: default  │        │
│  │   Status: Bound   │           │   Status: Bound       │        │
│  └──────────────────┘           └──────────────────────┘        │
│                                          │                      │
│                                          │ Referenced by        │
│                                          ▼                      │
│                                 ┌──────────────────┐            │
│                                 │   Pod             │            │
│                                 │   volumes:        │            │
│                                 │   - pvc: my-claim │            │
│                                 └──────────────────┘            │
└─────────────────────────────────────────────────────────────────┘
```

### Hands-On: Create PV and PVC Manually

Let's create a PV backed by NFS and a PVC that binds to it:

```yaml
# pv-nfs.yaml
# A PersistentVolume backed by an NFS server.
# This represents pre-provisioned storage — an admin has already
# set up the NFS server and exported /exports/data.
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-01
  labels:
    type: nfs
    environment: production
spec:
  capacity:
    storage: 50Gi                  # Total capacity of this volume
  accessModes:
  - ReadWriteMany                  # NFS supports multiple writers
  persistentVolumeReclaimPolicy: Retain   # Keep data after PVC is deleted
  storageClassName: nfs-slow       # Must match PVC's storageClassName
  mountOptions:
  - hard                           # NFS mount option: hard mount (retry indefinitely)
  - nfsvers=4.1                    # NFS mount option: use NFSv4.1
  nfs:
    server: 10.0.0.100            # NFS server IP
    path: /exports/data           # Exported path on the NFS server
```

```yaml
# pvc-nfs.yaml
# A PersistentVolumeClaim requesting 20Gi of ReadWriteMany storage
# from the 'nfs-slow' StorageClass.
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-nfs-claim
  namespace: default
spec:
  accessModes:
  - ReadWriteMany                  # Must be a subset of PV's accessModes
  resources:
    requests:
      storage: 20Gi               # Requested capacity (must be ≤ PV capacity)
  storageClassName: nfs-slow       # Must match PV's storageClassName
  selector:                        # Optional: further restrict which PVs can bind
    matchLabels:
      type: nfs
      environment: production
```

```bash
# Apply the PV and PVC
kubectl apply -f pv-nfs.yaml
kubectl apply -f pvc-nfs.yaml

# Check the binding
kubectl get pv nfs-pv-01
# NAME        CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                STORAGECLASS
# nfs-pv-01   50Gi       RWX            Retain           Bound    default/my-nfs-claim  nfs-slow

kubectl get pvc my-nfs-claim
# NAME           STATUS   VOLUME      CAPACITY   ACCESS MODES   STORAGECLASS
# my-nfs-claim   Bound    nfs-pv-01   50Gi       RWX            nfs-slow
```

### Binding: How PVC Finds a PV

The PV controller (part of `kube-controller-manager`) continuously watches for
unbound PVCs and attempts to find a matching PV. The matching criteria are:

1. **StorageClassName**: Must match exactly (or both must be empty)
2. **Access Modes**: The PV must support all access modes requested by the PVC
3. **Capacity**: The PV's capacity must be ≥ the PVC's request
4. **Selector**: If the PVC has a `selector`, the PV's labels must match
5. **Volume Mode**: Must match (Filesystem vs Block)

The controller picks the **smallest PV** that satisfies all criteria to minimize
wasted capacity.

Once bound, the relationship is exclusive — a PV can be bound to only one PVC, and
a PVC can be bound to only one PV. This is a one-to-one mapping enforced by the
control plane by setting `spec.claimRef` on the PV and `spec.volumeName` on the PVC.

### Access Modes

Access modes define how a volume can be mounted:

| Mode               | Abbreviation | Description                                  |
|--------------------|-------------|----------------------------------------------|
| `ReadWriteOnce`    | RWO         | Mounted as read-write by a **single node**   |
| `ReadOnlyMany`     | ROX         | Mounted as read-only by **many nodes**       |
| `ReadWriteMany`    | RWX         | Mounted as read-write by **many nodes**      |
| `ReadWriteOncePod` | RWOP        | Mounted as read-write by a **single Pod**    |

**Important nuances:**

- **RWO means single node, not single Pod.** Multiple Pods on the same node can all
  mount an RWO volume simultaneously. This trips up many people.
- **RWOP** (Kubernetes 1.27+, GA) is the strictest mode — exactly one Pod can mount
  the volume, regardless of node. Use this for databases where you absolutely must
  prevent concurrent writes.
- **RWX** requires a shared filesystem (NFS, CephFS, GlusterFS). Block storage
  (EBS, Persistent Disk) cannot support RWX.
- Access modes are not enforced at the filesystem level — they're enforced by the
  kubelet during mount. If you mount an NFS volume as RWO, the kubelet won't allow
  a second node to mount it, but NFS itself would happily serve multiple clients.

### Reclaim Policies

When a PVC is deleted, what happens to the underlying PV and its data?

| Policy   | Behavior                                                        |
|----------|-----------------------------------------------------------------|
| `Retain` | PV is marked "Released" but data is preserved. Admin must       |
|          | manually reclaim: delete PV, clean data, recreate if needed.   |
| `Delete` | PV and the underlying storage asset (e.g., EBS volume) are     |
|          | both deleted automatically. **Data is lost.**                   |
| `Recycle`| **Deprecated.** Ran `rm -rf /thevolume/*` on the volume.        |
|          | Use dynamic provisioning instead.                               |

**In production, use `Retain` for anything you can't afford to lose, and `Delete`
for ephemeral or reproducible data.**

### PV Lifecycle

```
                  ┌───────────┐
                  │ Available │ ◄── PV created, no PVC bound
                  └─────┬─────┘
                        │ PVC binds to PV
                        ▼
                  ┌───────────┐
                  │   Bound   │ ◄── PV is bound to a PVC
                  └─────┬─────┘
                        │ PVC is deleted
                        ▼
                  ┌───────────┐
                  │ Released  │ ◄── PV retains data, but PVC is gone
                  └─────┬─────┘
                        │
              ┌─────────┴─────────┐
              ▼                   ▼
      ┌───────────┐       ┌───────────┐
      │  Retained │       │  Deleted  │
      │ (manual   │       │ (auto-    │
      │  cleanup) │       │  cleanup) │
      └───────────┘       └───────────┘
```

**Released state trap:** A Released PV cannot be re-bound to a new PVC automatically.
You must either:
1. Delete and recreate the PV (data preserved on the backend)
2. Manually remove the `spec.claimRef` from the PV to make it Available again

### Hands-On: Observing Binding in Action

```bash
# Create a PV with 5Gi capacity
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: demo-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data
EOF

# PV is Available
kubectl get pv demo-pv
# STATUS: Available

# Create a PVC that matches
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
  storageClassName: manual
EOF

# PV is now Bound
kubectl get pv demo-pv
# STATUS: Bound

# PVC is also Bound
kubectl get pvc demo-pvc
# STATUS: Bound, VOLUME: demo-pv

# Use the PVC in a Pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: storage-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "echo 'Hello persistent world' > /data/hello.txt && sleep 3600"]
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: demo-pvc
EOF

# Verify data was written
kubectl exec storage-demo -- cat /data/hello.txt
# Hello persistent world

# Delete the Pod — data persists
kubectl delete pod storage-demo
# pod "storage-demo" deleted

# Recreate the Pod — data is still there
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: storage-demo-2
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "cat /data/hello.txt && sleep 3600"]
    volumeMounts:
    - name: storage
      mountPath: /data
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: demo-pvc
EOF

kubectl logs storage-demo-2
# Hello persistent world  ← Data survived Pod deletion!
```

---

## Dynamic Provisioning with StorageClasses

Manually creating PVs for every PVC is tedious and doesn't scale. In a cloud
environment with hundreds of developers creating databases, each needing their own
volume, you need **dynamic provisioning**.

### StorageClass: Defining Classes of Storage

A StorageClass is a cluster-scoped resource that describes a "class" of storage.
Think of it as a template: when a PVC references a StorageClass, the provisioner
associated with that class automatically creates a PV that matches.

```yaml
# storageclass-fast.yaml
# Defines a "fast" storage class backed by SSD (gp3) on AWS.
# When a PVC references this class, the EBS CSI driver automatically
# creates a gp3 EBS volume.
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: ebs.csi.aws.com       # The CSI driver that provisions volumes
parameters:
  type: gp3                         # EBS volume type
  iops: "4000"                      # Provisioned IOPS
  throughput: "250"                  # Throughput in MiB/s
  encrypted: "true"                 # Encrypt at rest
  fsType: ext4                      # Filesystem type
reclaimPolicy: Delete               # Delete EBS volume when PVC is deleted
allowVolumeExpansion: true          # Allow PVC resize
volumeBindingMode: WaitForFirstConsumer  # Don't provision until Pod is scheduled
mountOptions:
- discard                           # Enable TRIM/discard for SSD
```

### Key StorageClass Fields

**`provisioner`**: The name of the volume plugin that provisions PVs. For CSI
drivers, this is the driver name (e.g., `ebs.csi.aws.com`, `pd.csi.storage.gke.io`,
`disk.csi.azure.com`).

**`parameters`**: Provider-specific key-value pairs passed to the provisioner.
These configure the underlying storage — disk type, IOPS, encryption, replication
factor, etc.

**`reclaimPolicy`**: Either `Delete` (default) or `Retain`. Applied to dynamically
provisioned PVs.

**`volumeBindingMode`**: Controls when volume binding and provisioning occurs:
- `Immediate` (default): PV is provisioned as soon as the PVC is created
- `WaitForFirstConsumer`: PV provisioning is delayed until a Pod using the PVC is
  scheduled. **This is almost always what you want** because it ensures the volume
  is created in the same availability zone as the Pod.

**`allowVolumeExpansion`**: If `true`, PVCs using this class can be expanded by
editing the PVC's `spec.resources.requests.storage`.

### Volume Binding Modes: Immediate vs WaitForFirstConsumer

This is one of the most important StorageClass settings, and getting it wrong causes
subtle scheduling failures.

**`Immediate`**: The PV is created the instant the PVC is created. The problem?
In a multi-zone cluster, the provisioner might create the volume in zone `us-east-1a`,
but the scheduler might place the Pod in `us-east-1b`. The Pod can't mount a volume
in a different zone. The Pod is stuck in `Pending` with an error like:

```
Warning  FailedScheduling  pod/myapp  0/3 nodes are available:
  1 node had volume zone mismatch, 2 node(s) had taint...
```

**`WaitForFirstConsumer`**: The provisioner waits until the scheduler decides which
node the Pod will run on. Then it creates the volume in the same zone as that node.
This is called **topology-aware provisioning**.

```
Immediate mode:                    WaitForFirstConsumer:
                                   
1. PVC created                     1. PVC created → PVC Pending
2. PV provisioned in zone-a        2. Pod created → scheduler picks node in zone-b
3. Pod scheduled in zone-b         3. PV provisioned in zone-b
4. MOUNT FAILS! Zone mismatch.     4. Pod mounts PV successfully ✓
```

### Default StorageClass

One StorageClass can be marked as the default:

```yaml
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
```

PVCs that don't specify a `storageClassName` will use the default class. If no
default exists, PVCs without a class will remain Pending until a matching PV is
manually created.

To explicitly request no StorageClass (for manual binding), set
`storageClassName: ""` in the PVC.

### Hands-On: StorageClass with local-path-provisioner

In development clusters (kind, k3s, minikube), the `local-path-provisioner` is
commonly used. Let's set one up:

```yaml
# storageclass-local-path.yaml
# StorageClass for Rancher's local-path-provisioner.
# This creates volumes as directories on the node's filesystem.
# NOT suitable for production — data is lost if the node dies.
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer    # Required for local storage
reclaimPolicy: Delete
```

### Hands-On: Dynamic PVC Provisioning

```yaml
# dynamic-pvc.yaml
# This PVC references a StorageClass. The provisioner will automatically
# create a PV that matches.
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-claim
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast-ssd    # References the StorageClass we created
```

```bash
# Apply and watch the dynamic provisioning
kubectl apply -f dynamic-pvc.yaml

# Initially, PVC is Pending (WaitForFirstConsumer)
kubectl get pvc dynamic-claim
# STATUS: Pending (waiting for first consumer)

# Create a Pod that uses this PVC
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: dynamic-demo
spec:
  containers:
  - name: app
    image: nginx:1.25
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: dynamic-claim
EOF

# Now the PVC is Bound and a PV was automatically created
kubectl get pvc dynamic-claim
# STATUS: Bound, VOLUME: pvc-a1b2c3d4-...

kubectl get pv
# NAME                                       CAPACITY   STATUS   CLAIM
# pvc-a1b2c3d4-e5f6-7890-abcd-ef1234567890   10Gi       Bound    default/dynamic-claim
```

---

## CSI (Container Storage Interface) Deep Dive

### Why CSI Was Created

In the early days of Kubernetes, every storage provider's code was compiled directly
into the Kubernetes binaries. Want to use AWS EBS? The EBS driver code is in
`k8s.io/kubernetes/pkg/volume/aws_ebs/`. Want Ceph? It's in
`k8s.io/kubernetes/pkg/volume/rbd/`.

This created several problems:

1. **Release coupling**: Storage vendors had to submit changes to the Kubernetes
   repo and wait for the next Kubernetes release. A bug fix in the EBS driver
   required a full Kubernetes release.

2. **Binary bloat**: The kubelet and controller-manager binaries included code for
   every possible storage provider, even if the cluster used none of them.

3. **Privilege creep**: Storage driver code ran inside the kubelet with full node
   privileges. A bug in a storage driver could crash the kubelet.

4. **Testing burden**: Every storage driver had to be tested as part of the
   Kubernetes CI/CD pipeline.

5. **Vendor lock-in**: Adding a new storage provider required getting code merged
   into the Kubernetes project — a high bar.

CSI solves all of these by defining a **standard gRPC interface** that storage
drivers implement as separate processes (containers), communicating with Kubernetes
through Unix domain sockets.

### CSI Architecture

A CSI driver deployment consists of two major components:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Kubernetes Cluster                              │
│                                                                         │
│  ┌───────────────────────────────────────────────────────┐              │
│  │              Controller Plugin (Deployment/StatefulSet)│              │
│  │                                                       │              │
│  │  ┌──────────────┐  ┌────────────┐  ┌──────────────┐  │              │
│  │  │  CSI Driver   │  │ Provisioner│  │  Attacher    │  │              │
│  │  │  Container    │  │  Sidecar   │  │  Sidecar     │  │              │
│  │  │              │  │            │  │              │  │              │
│  │  │ gRPC Server  │◄─┤ Watches    │  │ Watches      │  │              │
│  │  │              │  │ PVCs       │  │ VolumeAttach- │  │              │
│  │  │ • CreateVol  │  │            │  │ ments        │  │              │
│  │  │ • DeleteVol  │  └────────────┘  └──────────────┘  │              │
│  │  │ • CtrlPub    │                                     │              │
│  │  │ • CtrlUnpub  │  ┌────────────┐  ┌──────────────┐  │              │
│  │  │              │  │Snapshotter │  │  Resizer     │  │              │
│  │  │              │  │  Sidecar   │  │  Sidecar     │  │              │
│  │  └──────────────┘  └────────────┘  └──────────────┘  │              │
│  └───────────────────────────────────────────────────────┘              │
│                                                                         │
│  ┌───────────────────────────────────────────────────────┐              │
│  │              Node Plugin (DaemonSet — every node)      │              │
│  │                                                       │              │
│  │  ┌──────────────┐  ┌────────────────┐  ┌───────────┐ │              │
│  │  │  CSI Driver   │  │ Node Driver    │  │ Liveness  │ │              │
│  │  │  Container    │  │ Registrar      │  │ Probe     │ │              │
│  │  │              │  │                │  │           │ │              │
│  │  │ gRPC Server  │  │ Registers CSI  │  │ Health    │ │              │
│  │  │              │  │ driver with    │  │ check     │ │              │
│  │  │ • NodeStage  │  │ kubelet via    │  │ endpoint  │ │              │
│  │  │ • NodePublish│  │ registration   │  │           │ │              │
│  │  │ • NodeUnpub  │  │ API            │  │           │ │              │
│  │  │ • NodeUnstage│  │                │  │           │ │              │
│  │  └──────────────┘  └────────────────┘  └───────────┘ │              │
│  └───────────────────────────────────────────────────────┘              │
└─────────────────────────────────────────────────────────────────────────┘
```

### CSI Services

A CSI driver implements three gRPC services:

#### Identity Service (Required)

Every CSI driver must implement the Identity service:

| RPC                      | Description                                  |
|--------------------------|----------------------------------------------|
| `GetPluginInfo`          | Returns driver name and version               |
| `GetPluginCapabilities`  | Reports what the driver can do                |
| `Probe`                  | Health check — is the driver ready?           |

#### Controller Service (Optional, runs centrally)

The Controller service manages volumes at the storage backend level:

| RPC                        | Description                                        |
|----------------------------|----------------------------------------------------|
| `CreateVolume`             | Create a new volume on the storage backend         |
| `DeleteVolume`             | Delete a volume from the storage backend           |
| `ControllerPublishVolume`  | Attach a volume to a node (e.g., attach EBS to EC2)|
| `ControllerUnpublishVolume`| Detach a volume from a node                        |
| `ValidateVolumeCapabilities`| Check if a volume supports requested capabilities |
| `ListVolumes`              | Enumerate existing volumes                         |
| `GetCapacity`              | Report available capacity                          |
| `CreateSnapshot`           | Create a snapshot of a volume                      |
| `DeleteSnapshot`           | Delete a snapshot                                  |
| `ListSnapshots`            | Enumerate existing snapshots                       |
| `ControllerExpandVolume`   | Expand a volume's capacity                         |

#### Node Service (Required, runs on every node)

The Node service performs operations on the node where the volume will be used:

| RPC                    | Description                                           |
|------------------------|-------------------------------------------------------|
| `NodeStageVolume`      | Mount volume to a staging directory (global mount)    |
| `NodeUnstageVolume`    | Unmount volume from staging directory                 |
| `NodePublishVolume`    | Bind-mount from staging to Pod's target path          |
| `NodeUnpublishVolume`  | Unmount from Pod's target path                        |
| `NodeGetVolumeStats`   | Get volume usage statistics                           |
| `NodeExpandVolume`     | Expand filesystem after volume expansion              |
| `NodeGetCapabilities`  | Report node service capabilities                      |
| `NodeGetInfo`          | Return node ID, topology info                         |

### The Mount Flow

When a Pod that uses a CSI volume is scheduled to a node, the following sequence
occurs:

```
1. Scheduler places Pod on Node-B

2. Controller: ControllerPublishVolume(volume-123, node-B)
   → Storage backend attaches volume to Node-B
   → e.g., AWS API: ec2:AttachVolume(vol-123, i-nodeB)

3. Node: NodeStageVolume(volume-123, /var/lib/kubelet/plugins/.../globalmount)
   → Format the block device if needed (mkfs.ext4 /dev/xvdf)
   → Mount to a staging path (mount /dev/xvdf /staging/path)
   → This is a "global mount" — done once per volume per node

4. Node: NodePublishVolume(volume-123, /var/lib/kubelet/pods/<pod-uid>/volumes/...)
   → Bind-mount from staging path to Pod's volume directory
   → Can be done multiple times (one per Pod using the volume)

5. kubelet bind-mounts the volume into the container's mount namespace

Pod is running with the volume mounted!
```

### CSI Sidecar Containers

CSI sidecars are the "glue" between the Kubernetes API and the CSI driver. They
watch Kubernetes resources and translate them into CSI gRPC calls:

| Sidecar                    | Watches                  | Calls CSI RPC               |
|----------------------------|--------------------------|-----------------------------|
| `csi-provisioner`          | PVCs                     | `CreateVolume`/`DeleteVolume`|
| `csi-attacher`             | VolumeAttachments        | `ControllerPublish`/`Unpub` |
| `csi-snapshotter`          | VolumeSnapshots          | `CreateSnapshot`/`Delete`   |
| `csi-resizer`              | PVCs (capacity change)   | `ControllerExpandVolume`    |
| `node-driver-registrar`    | —                        | Registers CSI driver w/ kubelet|
| `livenessprobe`            | —                        | `Probe` (health checks)     |

### Hands-On: Examining a CSI Driver Deployment

Let's look at the AWS EBS CSI driver as deployed in a real cluster:

```bash
# List all Pods for the EBS CSI driver
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver

# Controller (Deployment) — runs 2 replicas for HA
# NAME                                  READY   STATUS
# ebs-csi-controller-5b4f6c7d8-abc12   6/6     Running    ← 6 containers!
# ebs-csi-controller-5b4f6c7d8-def34   6/6     Running

# Node (DaemonSet) — runs on every node
# ebs-csi-node-g5h6i                   3/3     Running    ← 3 containers
# ebs-csi-node-j7k8l                   3/3     Running
# ebs-csi-node-m9n0o                   3/3     Running

# Let's see the 6 containers in the controller Pod
kubectl get pod ebs-csi-controller-5b4f6c7d8-abc12 -n kube-system \
  -o jsonpath='{range .spec.containers[*]}{.name}{"\n"}{end}'
# ebs-plugin             ← The actual CSI driver
# csi-provisioner        ← Watches PVCs, calls CreateVolume
# csi-attacher           ← Watches VolumeAttachments, calls ControllerPublishVolume
# csi-snapshotter        ← Watches VolumeSnapshots, calls CreateSnapshot
# csi-resizer            ← Watches PVC resize, calls ControllerExpandVolume
# liveness-probe         ← Health checks

# And the 3 containers in a node Pod
kubectl get pod ebs-csi-node-g5h6i -n kube-system \
  -o jsonpath='{range .spec.containers[*]}{.name}{"\n"}{end}'
# ebs-plugin             ← The actual CSI driver (node service)
# node-driver-registrar  ← Registers with kubelet
# liveness-probe         ← Health checks

# The CSI driver exposes its socket at:
kubectl exec -n kube-system ebs-csi-node-g5h6i -c ebs-plugin -- \
  ls -la /csi/csi.sock
# srwxr-xr-x 1 root root 0 ... /csi/csi.sock
```

### Hands-On: Writing a Minimal CSI Driver Skeleton in Go

Let's write the skeleton of a CSI driver that implements the Identity and Controller
services. This is instructive for understanding the CSI protocol.

```go
// main.go
//
// # Minimal CSI Driver Skeleton
//
// This program implements a bare-minimum CSI (Container Storage Interface)
// driver that serves the Identity and Controller gRPC services. It
// demonstrates the structure and protocol of a CSI driver without any
// real storage backend.
//
// In production, the CreateVolume and DeleteVolume methods would call
// a real storage API (AWS EBS, Ceph, etc.) to provision actual volumes.
//
// Usage:
//
//	go run main.go --endpoint=unix:///csi/csi.sock --nodeid=node-1
package main

import (
	"context"
	"flag"
	"fmt"
	"log"
	"net"
	"net/url"
	"os"
	"sync"

	"github.com/container-storage-interface/spec/lib/go/csi"
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
)

const (
	// driverName is the unique name of this CSI driver.
	// It follows the reverse-DNS convention recommended by the CSI spec.
	driverName = "csi.example.com"

	// driverVersion is the semantic version of this driver.
	driverVersion = "0.1.0"
)

// main parses command-line flags and starts the gRPC server that hosts
// the CSI Identity and Controller services.
func main() {
	// endpoint is the Unix domain socket path where the driver listens.
	// CSI sidecar containers connect to this socket.
	endpoint := flag.String("endpoint", "unix:///csi/csi.sock",
		"CSI endpoint (unix socket path)")

	// nodeID is the unique identifier for the node where this driver runs.
	// For the Controller service, this is informational only.
	nodeID := flag.String("nodeid", "",
		"Node ID for this CSI node plugin instance")

	flag.Parse()

	if *nodeID == "" {
		// In production, this would be auto-detected from instance metadata
		hostname, err := os.Hostname()
		if err != nil {
			log.Fatalf("Failed to get hostname: %v", err)
		}
		*nodeID = hostname
	}

	// Parse the endpoint URL to extract the socket path
	u, err := url.Parse(*endpoint)
	if err != nil {
		log.Fatalf("Failed to parse endpoint %q: %v", *endpoint, err)
	}

	socketPath := u.Path
	if u.Host != "" {
		socketPath = u.Host + socketPath
	}

	// Remove the socket file if it already exists (stale from previous run)
	if err := os.Remove(socketPath); err != nil && !os.IsNotExist(err) {
		log.Fatalf("Failed to remove existing socket %q: %v", socketPath, err)
	}

	// Create a Unix domain socket listener
	listener, err := net.Listen("unix", socketPath)
	if err != nil {
		log.Fatalf("Failed to listen on %q: %v", socketPath, err)
	}
	defer listener.Close()

	log.Printf("CSI driver %s v%s listening on %s", driverName, driverVersion, socketPath)

	// Create the gRPC server and register our CSI services
	server := grpc.NewServer()

	// Register the Identity service (required for all CSI drivers)
	csi.RegisterIdentityServer(server, &identityServer{})

	// Register the Controller service (manages volume lifecycle)
	csi.RegisterControllerServer(server, newControllerServer())

	// Serve gRPC requests until the process is terminated
	if err := server.Serve(listener); err != nil {
		log.Fatalf("gRPC server failed: %v", err)
	}
}

// =============================================================================
// Identity Service
// =============================================================================

// identityServer implements the CSI Identity service.
//
// Every CSI driver MUST implement this service. It provides:
//   - GetPluginInfo:         returns the driver name and version
//   - GetPluginCapabilities: reports which optional services the driver supports
//   - Probe:                 health check to verify the driver is operational
//
// The Identity service is used by the Kubernetes CSI sidecars and the kubelet
// to discover and validate the driver.
type identityServer struct {
	csi.UnimplementedIdentityServer
}

// GetPluginInfo returns the name and version of this CSI driver.
//
// The name MUST be unique across all CSI drivers in the cluster and should
// follow reverse-DNS notation (e.g., "ebs.csi.aws.com").
// The Kubernetes CSI provisioner sidecar uses this name to match StorageClass
// provisioner fields to CSI drivers.
func (s *identityServer) GetPluginInfo(
	ctx context.Context,
	req *csi.GetPluginInfoRequest,
) (*csi.GetPluginInfoResponse, error) {
	log.Printf("GetPluginInfo called")
	return &csi.GetPluginInfoResponse{
		Name:          driverName,
		VendorVersion: driverVersion,
	}, nil
}

// GetPluginCapabilities reports what this CSI driver can do.
//
// We advertise CONTROLLER_SERVICE to indicate that this driver implements
// the Controller service (CreateVolume, DeleteVolume, etc.).
// If we also supported volume accessibility constraints (topology),
// we would advertise VOLUME_ACCESSIBILITY_CONSTRAINTS.
func (s *identityServer) GetPluginCapabilities(
	ctx context.Context,
	req *csi.GetPluginCapabilitiesRequest,
) (*csi.GetPluginCapabilitiesResponse, error) {
	log.Printf("GetPluginCapabilities called")
	return &csi.GetPluginCapabilitiesResponse{
		Capabilities: []*csi.PluginCapability{
			{
				Type: &csi.PluginCapability_Service_{
					Service: &csi.PluginCapability_Service{
						// Advertise that we implement the Controller service
						Type: csi.PluginCapability_Service_CONTROLLER_SERVICE,
					},
				},
			},
		},
	}, nil
}

// Probe verifies that the CSI driver is healthy and ready to serve requests.
//
// The liveness probe sidecar calls this RPC periodically. If it fails,
// Kubernetes restarts the driver container.
//
// In a real driver, this might verify connectivity to the storage backend.
func (s *identityServer) Probe(
	ctx context.Context,
	req *csi.ProbeRequest,
) (*csi.ProbeResponse, error) {
	// A minimal probe: we're alive if this code is executing.
	// Production drivers should check backend connectivity here.
	return &csi.ProbeResponse{}, nil
}

// =============================================================================
// Controller Service
// =============================================================================

// controllerServer implements the CSI Controller service.
//
// The Controller service runs centrally (usually as a Deployment with 1-2
// replicas) and handles volume lifecycle operations:
//   - CreateVolume:              provision a new volume on the storage backend
//   - DeleteVolume:              remove a volume from the storage backend
//   - ControllerPublishVolume:   attach a volume to a specific node
//   - ControllerUnpublishVolume: detach a volume from a node
//   - ValidateVolumeCapabilities: verify volume supports requested access modes
//   - ListVolumes:               enumerate all volumes
//   - GetCapacity:               report available storage capacity
//   - ControllerExpandVolume:    resize an existing volume
//
// In this skeleton, we simulate volumes in memory using a map. A real driver
// would call a storage API (AWS EC2, OpenStack Cinder, Ceph, etc.).
type controllerServer struct {
	csi.UnimplementedControllerServer

	// mu protects the volumes map from concurrent access.
	// The CSI sidecars may issue concurrent requests.
	mu sync.Mutex

	// volumes is an in-memory store of created volumes, keyed by volume ID.
	// In a real driver, this state lives in the storage backend.
	volumes map[string]*volume
}

// volume represents a provisioned storage volume.
// In a real CSI driver, this would correspond to a cloud disk,
// an NFS export, a Ceph RBD image, etc.
type volume struct {
	// id is the unique identifier for this volume.
	id string

	// name is the human-readable name (from PVC metadata).
	name string

	// capacityBytes is the size of the volume in bytes.
	capacityBytes int64
}

// newControllerServer creates a new controllerServer with an empty volume store.
func newControllerServer() *controllerServer {
	return &controllerServer{
		volumes: make(map[string]*volume),
	}
}

// CreateVolume provisions a new volume on the storage backend.
//
// The csi-provisioner sidecar calls this when it detects a new unbound PVC
// that references a StorageClass with our driver as the provisioner.
//
// Parameters:
//   - req.Name: suggested volume name (derived from PVC name + uid)
//   - req.CapacityRange: minimum and maximum capacity requested
//   - req.VolumeCapabilities: requested access modes and mount flags
//   - req.Parameters: key-value pairs from the StorageClass parameters
//
// Returns:
//   - The created volume's ID, capacity, and any topology information
//
// Idempotency: If a volume with the same name already exists and has
// compatible parameters, this method MUST return success (not an error).
// This is required by the CSI spec to handle retries.
func (s *controllerServer) CreateVolume(
	ctx context.Context,
	req *csi.CreateVolumeRequest,
) (*csi.CreateVolumeResponse, error) {
	// Validate required parameters
	if req.GetName() == "" {
		return nil, status.Error(codes.InvalidArgument, "volume name is required")
	}
	if len(req.GetVolumeCapabilities()) == 0 {
		return nil, status.Error(codes.InvalidArgument,
			"volume capabilities are required")
	}

	// Determine the requested capacity.
	// If no capacity is specified, default to 1 GiB.
	capacityBytes := int64(1 * 1024 * 1024 * 1024) // 1 GiB default
	if req.GetCapacityRange() != nil {
		if req.GetCapacityRange().GetRequiredBytes() > 0 {
			capacityBytes = req.GetCapacityRange().GetRequiredBytes()
		}
	}

	s.mu.Lock()
	defer s.mu.Unlock()

	// Idempotency check: if a volume with this name already exists,
	// return it (if capacity matches).
	for _, v := range s.volumes {
		if v.name == req.GetName() {
			if v.capacityBytes >= capacityBytes {
				log.Printf("CreateVolume: returning existing volume %s (%s)",
					v.id, v.name)
				return &csi.CreateVolumeResponse{
					Volume: &csi.Volume{
						VolumeId:      v.id,
						CapacityBytes: v.capacityBytes,
					},
				}, nil
			}
			return nil, status.Errorf(codes.AlreadyExists,
				"volume %s exists with smaller capacity %d < %d",
				v.name, v.capacityBytes, capacityBytes)
		}
	}

	// Generate a unique volume ID.
	// In production, this would be the cloud-provider-assigned volume ID
	// (e.g., "vol-0a1b2c3d4e5f67890" for AWS EBS).
	volumeID := fmt.Sprintf("vol-%s", req.GetName())

	vol := &volume{
		id:            volumeID,
		name:          req.GetName(),
		capacityBytes: capacityBytes,
	}
	s.volumes[volumeID] = vol

	log.Printf("CreateVolume: created volume %s (%s, %d bytes)",
		volumeID, req.GetName(), capacityBytes)

	return &csi.CreateVolumeResponse{
		Volume: &csi.Volume{
			VolumeId:      volumeID,
			CapacityBytes: capacityBytes,
		},
	}, nil
}

// DeleteVolume removes a volume from the storage backend.
//
// The csi-provisioner sidecar calls this when a PVC with reclaimPolicy=Delete
// is deleted and the associated PV is released.
//
// Idempotency: If the volume does not exist, this method MUST return success.
// (The volume may have already been deleted by a previous call.)
func (s *controllerServer) DeleteVolume(
	ctx context.Context,
	req *csi.DeleteVolumeRequest,
) (*csi.DeleteVolumeResponse, error) {
	if req.GetVolumeId() == "" {
		return nil, status.Error(codes.InvalidArgument, "volume ID is required")
	}

	s.mu.Lock()
	defer s.mu.Unlock()

	// Delete the volume if it exists. If it doesn't exist, that's fine
	// (idempotency requirement).
	if _, exists := s.volumes[req.GetVolumeId()]; exists {
		delete(s.volumes, req.GetVolumeId())
		log.Printf("DeleteVolume: deleted volume %s", req.GetVolumeId())
	} else {
		log.Printf("DeleteVolume: volume %s not found (already deleted?)",
			req.GetVolumeId())
	}

	return &csi.DeleteVolumeResponse{}, nil
}

// ControllerGetCapabilities reports which Controller RPCs this driver supports.
//
// The csi-provisioner, csi-attacher, and other sidecars call this to discover
// what operations they can delegate to this driver. If we don't advertise
// CREATE_DELETE_VOLUME, the provisioner won't call CreateVolume/DeleteVolume.
func (s *controllerServer) ControllerGetCapabilities(
	ctx context.Context,
	req *csi.ControllerGetCapabilitiesRequest,
) (*csi.ControllerGetCapabilitiesResponse, error) {
	return &csi.ControllerGetCapabilitiesResponse{
		Capabilities: []*csi.ControllerServiceCapability{
			{
				// We support creating and deleting volumes
				Type: &csi.ControllerServiceCapability_Rpc{
					Rpc: &csi.ControllerServiceCapability_RPC{
						Type: csi.ControllerServiceCapability_RPC_CREATE_DELETE_VOLUME,
					},
				},
			},
		},
	}, nil
}

// ValidateVolumeCapabilities checks whether a volume supports the requested
// capabilities (access modes, mount flags, etc.).
//
// This is called by the Kubernetes PV controller after a volume is provisioned
// to verify it actually supports what the PVC requested.
func (s *controllerServer) ValidateVolumeCapabilities(
	ctx context.Context,
	req *csi.ValidateVolumeCapabilitiesRequest,
) (*csi.ValidateVolumeCapabilitiesResponse, error) {
	if req.GetVolumeId() == "" {
		return nil, status.Error(codes.InvalidArgument, "volume ID is required")
	}

	s.mu.Lock()
	_, exists := s.volumes[req.GetVolumeId()]
	s.mu.Unlock()

	if !exists {
		return nil, status.Errorf(codes.NotFound,
			"volume %s not found", req.GetVolumeId())
	}

	// In this skeleton, we accept all requested capabilities.
	// A real driver would check against the storage backend's actual capabilities.
	return &csi.ValidateVolumeCapabilitiesResponse{
		Confirmed: &csi.ValidateVolumeCapabilitiesResponse_Confirmed{
			VolumeCapabilities: req.GetVolumeCapabilities(),
		},
	}, nil
}
```

To build and test this skeleton:

```bash
# Initialize the Go module
mkdir -p my-csi-driver && cd my-csi-driver
go mod init github.com/example/my-csi-driver

# Download dependencies
go get github.com/container-storage-interface/spec@v1.9.0
go get google.golang.org/grpc@v1.62.0

# Build the driver
go build -o my-csi-driver .

# Test the Identity service with csc (CSI command-line client)
# github.com/rexray/gocsi/csc
go install github.com/rexray/gocsi/csc@latest

# Start the driver
./my-csi-driver --endpoint=unix:///tmp/csi.sock &

# Query the Identity service
csc identity plugin-info --endpoint=unix:///tmp/csi.sock
# "csi.example.com" "0.1.0"

# Create a volume
csc controller create-volume --endpoint=unix:///tmp/csi.sock \
  --cap SINGLE_NODE_WRITER,mount,ext4 \
  --req-bytes 10737418240 \
  my-test-volume
# "vol-my-test-volume" 10737418240
```

---

## Volume Snapshots

Volume snapshots provide point-in-time copies of volumes. They're essential for
backup, disaster recovery, and creating new volumes from existing data.

### Snapshot API Resources

The snapshot feature uses three CRDs (Custom Resource Definitions):

| Resource                 | Scope     | Analogous to    | Description                        |
|--------------------------|-----------|------------------|------------------------------------|
| `VolumeSnapshot`         | Namespaced| PVC              | User's request for a snapshot      |
| `VolumeSnapshotContent`  | Cluster   | PV               | Actual snapshot on storage backend |
| `VolumeSnapshotClass`    | Cluster   | StorageClass     | Parameters for snapshot creation   |

The relationship mirrors PV/PVC:
- VolumeSnapshot requests a snapshot (like PVC requests storage)
- VolumeSnapshotContent represents the actual snapshot (like PV represents storage)
- VolumeSnapshotClass defines how snapshots are created (like StorageClass)

### Hands-On: Snapshot and Restore Workflow

```yaml
# 1. Create a VolumeSnapshotClass
# This tells Kubernetes which CSI driver handles snapshots and how.
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
driver: ebs.csi.aws.com              # Must match the CSI driver name
deletionPolicy: Delete               # Delete snapshot when VolumeSnapshot is deleted
# parameters:                        # Provider-specific parameters (optional)
#   encrypted: "true"
```

```yaml
# 2. Create a VolumeSnapshot from an existing PVC
# This triggers the csi-snapshotter sidecar to call CreateSnapshot
# on the CSI driver, which calls the storage backend's snapshot API.
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-data-snapshot
  namespace: default
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: my-data-pvc   # PVC to snapshot
```

```bash
# Check snapshot status
kubectl get volumesnapshot my-data-snapshot
# NAME               READYTOUSE   SOURCEPVC       RESTORESIZE   SNAPSHOTCLASS
# my-data-snapshot   true         my-data-pvc     10Gi          ebs-snapshot-class

# View the underlying VolumeSnapshotContent
kubectl get volumesnapshotcontent
# NAME                                               READYTOUSE   RESTORESIZE   DRIVER
# snapcontent-a1b2c3d4-e5f6-7890-abcd-1234567890ab   true         10Gi          ebs.csi.aws.com
```

```yaml
# 3. Restore: Create a new PVC from the snapshot
# The CSI driver creates a new volume pre-populated with the snapshot data.
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-data-pvc
  namespace: default
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi                  # Must be ≥ snapshot's restoreSize
  storageClassName: fast-ssd
  dataSource:                        # Key field: specifies the snapshot source
    name: my-data-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
```

```yaml
# 4. Use the restored PVC in a Pod
apiVersion: v1
kind: Pod
metadata:
  name: restored-app
spec:
  containers:
  - name: app
    image: postgres:16
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql/data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: restored-data-pvc   # PVC restored from snapshot
```

### Pre-provisioned Snapshots

You can also import existing snapshots from the storage backend:

```yaml
# VolumeSnapshotContent pointing to an existing snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotContent
metadata:
  name: imported-snapshot-content
spec:
  driver: ebs.csi.aws.com
  deletionPolicy: Retain
  source:
    snapshotHandle: snap-0a1b2c3d4e5f67890   # Existing snapshot ID on AWS
  volumeSnapshotRef:
    name: imported-snapshot
    namespace: default
---
# VolumeSnapshot referencing the pre-provisioned content
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: imported-snapshot
  namespace: default
spec:
  source:
    volumeSnapshotContentName: imported-snapshot-content
```

---

## Volume Expansion

As applications grow, they need more storage. Kubernetes supports expanding
PVCs in-place without data loss.

### Prerequisites

1. The StorageClass must have `allowVolumeExpansion: true`
2. The CSI driver must support `ControllerExpandVolume` and/or `NodeExpandVolume`
3. The volume must be using a filesystem that supports online resize (ext4, xfs)

### Online vs Offline Expansion

**Online expansion** (preferred): The volume is expanded while it's mounted and in
use. The CSI driver expands the block device, then the node plugin expands the
filesystem. No downtime required.

**Offline expansion**: The volume must be unmounted before expansion. The Pod using
the volume must be deleted, the volume is expanded, and then a new Pod can mount it.

Whether online or offline expansion is supported depends on the CSI driver and
the underlying storage system.

### Hands-On: Expanding a PVC

```yaml
# StorageClass with volume expansion enabled
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
allowVolumeExpansion: true            # This is the key setting
volumeBindingMode: WaitForFirstConsumer
```

```yaml
# Create a 10Gi PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: expandable-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: expandable-ssd
```

```bash
# Current size
kubectl get pvc expandable-pvc
# CAPACITY: 10Gi

# Expand to 50Gi by editing the PVC
kubectl patch pvc expandable-pvc -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'

# Monitor the expansion
kubectl get pvc expandable-pvc -w
# CAPACITY: 10Gi    (initially — backend is expanding)
# CAPACITY: 50Gi    (after expansion completes)

# Check events for details
kubectl describe pvc expandable-pvc
# Events:
#   Normal  ExternalExpanding   csi-resizer   CSI driver ebs.csi.aws.com expanding volume
#   Normal  Resizing            csi-resizer   External resizer is resizing volume
#   Normal  FileSystemResizeRequired  kubelet  Require file system resize
#   Normal  FileSystemResizeSuccessful  kubelet  MountVolume.NodeExpandVolume succeeded
```

**Note:** You can only expand PVCs. Shrinking is not supported — you cannot decrease
`spec.resources.requests.storage`. To shrink a volume, you must create a new smaller
PVC, copy data to it, and switch your application over.

---

## Ephemeral Volumes

Not all storage needs to be persistent. Some use cases require temporary storage
that is tightly coupled to a Pod's lifecycle but has more flexibility than `emptyDir`.

### Generic Ephemeral Volumes

Generic ephemeral volumes look like regular PVCs but are created and deleted
automatically with the Pod:

```yaml
# generic-ephemeral.yaml
# The PVC is created when the Pod is created and deleted when the Pod is deleted.
# The volume gets a unique PVC name: <pod-name>-<volume-name>.
apiVersion: v1
kind: Pod
metadata:
  name: ephemeral-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sh", "-c", "df -h /scratch && sleep 3600"]
    volumeMounts:
    - name: scratch
      mountPath: /scratch
  volumes:
  - name: scratch
    ephemeral:
      volumeClaimTemplate:
        metadata:
          labels:
            type: ephemeral-scratch
        spec:
          accessModes:
          - ReadWriteOnce
          resources:
            requests:
              storage: 5Gi
          storageClassName: fast-ssd
```

**Key difference from emptyDir:** Ephemeral volumes use the full StorageClass
infrastructure. They can be SSD-backed, have specific IOPS, use specific
filesystems, and respect volume quotas. An `emptyDir` just uses whatever the
node has.

### CSI Ephemeral Volumes

Some CSI drivers support inline ephemeral volumes — volumes specified directly in
the Pod spec without a PVC:

```yaml
# csi-ephemeral.yaml
# The CSI driver manages the volume's lifecycle tied to the Pod.
# Only some CSI drivers support this (e.g., secrets-store-csi-driver).
apiVersion: v1
kind: Pod
metadata:
  name: csi-ephemeral-demo
spec:
  containers:
  - name: app
    image: busybox:1.36
    command: ["sleep", "3600"]
    volumeMounts:
    - name: secrets
      mountPath: /mnt/secrets
      readOnly: true
  volumes:
  - name: secrets
    csi:
      driver: secrets-store.csi.k8s.io
      readOnly: true
      volumeAttributes:
        secretProviderClass: "aws-secrets"
```

**Use cases for CSI ephemeral volumes:**
- Secrets injection (secrets-store-csi-driver)
- Image mounting (warm-metal/csi-image)
- Temporary encrypted storage

---

## Storage Best Practices

### 1. Always Use WaitForFirstConsumer

Unless you have a specific reason not to, set `volumeBindingMode:
WaitForFirstConsumer` on all StorageClasses. This prevents zone mismatch issues
and ensures the volume is created in the same topology domain as the Pod.

```yaml
# GOOD
volumeBindingMode: WaitForFirstConsumer

# BAD (unless you specifically need pre-provisioning)
volumeBindingMode: Immediate
```

### 2. Set Resource Requests on PVCs

Always specify the exact amount of storage you need. Over-provisioning wastes
money; under-provisioning leads to disk-full errors.

```yaml
resources:
  requests:
    storage: 20Gi    # Be specific — not 100Gi "just in case"
```

Use volume expansion (with `allowVolumeExpansion: true`) instead of over-provisioning.
Start small and grow as needed.

### 3. Use ReadWriteOncePod When Possible

If only one Pod should access a volume (which is true for most databases),
use `ReadWriteOncePod` (RWOP) instead of `ReadWriteOnce` (RWO). RWOP provides
stronger guarantees — only one Pod can mount the volume, not just one node.

```yaml
accessModes:
- ReadWriteOncePod    # Strictest: exactly one Pod
# NOT
# - ReadWriteOnce     # One node, but multiple Pods on that node
```

### 4. Backup Strategies for Persistent Data

PersistentVolumes are persistent, not indestructible. Hardware fails, regions go
down, and humans accidentally run `kubectl delete pvc --all`.

**Backup strategy checklist:**

- **Volume Snapshots**: Use VolumeSnapshots for point-in-time backups. Schedule them
  with a CronJob or a tool like Velero.
- **Application-level backups**: For databases, use `pg_dump`, `mysqldump`, or the
  database's native backup tool. These are more portable than volume snapshots.
- **Cross-region replication**: For disaster recovery, replicate snapshots to another
  region.
- **Test your restores**: A backup that you haven't tested restoring is not a backup.

### 5. Monitor Volume Usage

Set up alerts for volume usage. A 100% full PVC will cause application failures
and potentially data corruption.

```yaml
# Prometheus alerting rule for PVC usage
groups:
- name: storage
  rules:
  - alert: PVCNearlyFull
    expr: |
      kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.85
    for: 15m
    labels:
      severity: warning
    annotations:
      summary: "PVC {{ $labels.persistentvolumeclaim }} is >85% full"
```

### 6. Use Retain for Critical Data

For databases and other critical data, use `reclaimPolicy: Retain` on the
StorageClass. This ensures that even if the PVC is accidentally deleted,
the underlying volume and data are preserved.

```yaml
reclaimPolicy: Retain    # Data preserved even if PVC is deleted
```

### 7. Label Everything

Apply labels to PVs and PVCs for organization and selector-based binding:

```yaml
metadata:
  labels:
    app: postgres
    environment: production
    team: backend
    cost-center: engineering
```

### 8. Use CSI Drivers, Not In-Tree Plugins

If you're still using in-tree volume types like `awsElasticBlockStore` or
`gcePersistentDisk`, migrate to the corresponding CSI driver. In-tree plugins
are deprecated and will be removed in future Kubernetes versions.

---

## Summary

| Concept            | Key Takeaway                                               |
|--------------------|------------------------------------------------------------|
| emptyDir           | Ephemeral, Pod-scoped shared storage                      |
| hostPath           | Node-local, dangerous in production                       |
| PersistentVolume   | Cluster-scoped storage resource                           |
| PersistentVolumeClaim | Namespaced request for storage                        |
| StorageClass       | Template for dynamic provisioning                         |
| CSI                | Standard interface for out-of-tree storage drivers        |
| VolumeSnapshot     | Point-in-time backup of a volume                          |
| Volume Expansion   | Grow PVCs in-place                                        |

The storage subsystem is where Kubernetes' "cattle not pets" philosophy meets
reality. Containers are cattle, but data is the farm. The PV/PVC/StorageClass/CSI
stack gives you a powerful, extensible, provider-agnostic way to manage that farm
— but only if you understand each layer.

In the next chapter, we'll dive into CNI — the Container Network Interface — and
see how Kubernetes networking follows a remarkably similar pattern: a standard
interface, plugin architecture, and separation of concerns between the core system
and provider-specific implementations.
