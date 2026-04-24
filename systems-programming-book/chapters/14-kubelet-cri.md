# Chapter 14: Kubelet, CRI & Pod Lifecycle

> *"The kubelet is where Kubernetes meets reality — it transforms the desired state
> written in etcd into actual running containers on actual Linux machines."*

Every node in a Kubernetes cluster runs a single, critical agent: the **kubelet**.
The kubelet watches the API server for Pods assigned to its node, then drives the
container runtime to create, start, monitor, and stop containers. It manages volumes,
reports node status, runs health probes, and handles graceful shutdown. Between the
kubelet and the container runtime sits the **Container Runtime Interface (CRI)** — a
gRPC API that decouples Kubernetes from any specific container runtime.

In this chapter, we trace the entire journey of a Pod — from API server to running
containers — and dissect every component the kubelet uses along the way.

---

## 14.1 Kubelet — The Node Agent

### 14.1.1 What the Kubelet Does

The kubelet is the **node-level reconciliation engine**. Its job:

1. **Watch** for Pods assigned to this node (via the API server).
2. **Reconcile** the actual state of containers with the desired state in each Pod spec.
3. **Report** node status and Pod status back to the API server.

The kubelet does NOT:
- Schedule Pods (that's the scheduler).
- Route network traffic (that's kube-proxy / CNI).
- Manage cluster state (that's the controller manager).

### 14.1.2 How the Kubelet Runs

The kubelet runs as a **systemd service** on each node (not as a container):

```bash
# Typical systemd unit
[Unit]
Description=Kubernetes Kubelet
After=containerd.service

[Service]
ExecStart=/usr/bin/kubelet \
  --config=/var/lib/kubelet/config.yaml \
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock \
  --kubeconfig=/etc/kubernetes/kubelet.conf \
  --node-ip=10.0.1.5
Restart=always

[Install]
WantedBy=multi-user.target
```

Why not a container? The kubelet must manage the container runtime itself — it
cannot depend on the thing it manages to run.

---

## 14.2 Kubelet Architecture

### 14.2.1 Internal Components

```
┌──────────────────────────────────────────────────────────────────────┐
│                           KUBELET                                    │
│                                                                      │
│  ┌──────────────┐    ┌──────────────────────────────────────────┐    │
│  │  API Server   │    │             Sync Loop                    │    │
│  │  Watch/List   │───▶│  (Main reconciliation goroutine)         │    │
│  └──────────────┘    │                                          │    │
│                       │  For each Pod:                           │    │
│  ┌──────────────┐    │    Compare desired vs actual state        │    │
│  │ Static Pod    │───▶│    Take action (create/start/stop/kill)  │    │
│  │ File Watcher  │    │                                          │    │
│  └──────────────┘    └──────────┬───────────────────────────────┘    │
│                                  │                                    │
│  ┌──────────────┐               │ drives                             │
│  │ HTTP Source   │───────────┐  │                                    │
│  └──────────────┘            │  │                                    │
│                               ▼  ▼                                    │
│  ┌───────────────────────────────────────────────────────────────┐   │
│  │                    Pod Lifecycle Manager                       │   │
│  │                                                               │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌────────────────────┐  │   │
│  │  │  Pod Workers  │  │ Probe Manager│  │ Container Runtime  │  │   │
│  │  │ (per-pod      │  │              │  │ (via CRI gRPC)     │  │   │
│  │  │  goroutines)  │  │ - Liveness   │  │                    │  │   │
│  │  │              │  │ - Readiness  │  │ - RunPodSandbox    │  │   │
│  │  │              │  │ - Startup    │  │ - CreateContainer  │  │   │
│  │  └──────────────┘  └──────────────┘  │ - StartContainer   │  │   │
│  │                                       │ - StopContainer    │  │   │
│  └───────────────────────────────────────┴────────────────────┘─┘   │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                     Supporting Managers                         │  │
│  │                                                                 │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │  │
│  │  │ Volume       │  │ Image        │  │ Status Manager       │ │  │
│  │  │ Manager      │  │ Manager      │  │                      │ │  │
│  │  │              │  │              │  │ Reports Pod/Node     │ │  │
│  │  │ Mount/       │  │ Pull/GC      │  │ status to API server │ │  │
│  │  │ Unmount      │  │ images       │  │                      │ │  │
│  │  └──────────────┘  └──────────────┘  └──────────────────────┘ │  │
│  │                                                                 │  │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │  │
│  │  │ PLEG         │  │ Certificate  │  │ Node Status          │ │  │
│  │  │ (Pod Life-   │  │ Manager      │  │ Manager              │ │  │
│  │  │ cycle Event  │  │              │  │                      │ │  │
│  │  │ Generator)   │  │ Rotate       │  │ Node conditions,     │ │  │
│  │  │              │  │ kubelet      │  │ capacity, allocatable│ │  │
│  │  │ Detect       │  │ certs        │  │                      │ │  │
│  │  │ container    │  │              │  │                      │ │  │
│  │  │ state changes│  └──────────────┘  └──────────────────────┘ │  │
│  │  └──────────────┘                                              │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                    CRI (gRPC)                                   │  │
│  │                                                                 │  │
│  │  RuntimeService        │  ImageService                         │  │
│  │  - RunPodSandbox       │  - PullImage                          │  │
│  │  - StopPodSandbox      │  - ListImages                         │  │
│  │  - CreateContainer     │  - RemoveImage                        │  │
│  │  - StartContainer      │  - ImageFsInfo                        │  │
│  │  - StopContainer       │                                       │  │
│  │  - RemoveContainer     │                                       │  │
│  │  - ListContainers      │                                       │  │
│  │  - ContainerStatus     │                                       │  │
│  └────────────────────────┴───────────────────────────────────────┘  │
│                          │                                           │
└──────────────────────────┼───────────────────────────────────────────┘
                           │ gRPC over unix socket
                           ▼
              ┌────────────────────────┐
              │   Container Runtime    │
              │   (containerd / CRI-O) │
              └────────────────────────┘
```

### 14.2.2 PLEG — Pod Lifecycle Event Generator

The PLEG is a critical internal component that detects container state changes.
It periodically (every 1 second) calls `ListContainers` and `ListSandboxes` via
CRI, compares the result with its cached state, and generates events:

- `ContainerStarted`
- `ContainerDied`
- `ContainerRemoved`
- `ContainerChanged`

These events drive the sync loop to reconcile Pods. If the PLEG falls behind
(takes more than 3 minutes to relist), the node is marked `NotReady` with the
condition `PLEGNotHealthy`.

**Newer alternative:** Starting in Kubernetes 1.26, **Evented PLEG** (beta in 1.27)
uses CRI events (container runtime pushes events) instead of periodic polling. This
reduces CPU usage and improves responsiveness.

### 14.2.3 Volume Manager

The volume manager handles the lifecycle of volumes attached to Pods:

1. **Attach** — for network volumes (EBS, GCE PD, Azure Disk), coordinate with the
   `AttachDetachController` in the controller manager.
2. **Mount** — mount the volume into the Pod's directory on the host filesystem.
3. **Unmount** — when the Pod terminates.
4. **Detach** — release the volume from the node.

The volume manager runs a reconciliation loop independent of the sync loop, because
volume operations can take significant time (attaching an EBS volume may take 30+
seconds).

### 14.2.4 Image Manager

Manages container image lifecycle:
- **Pull** images required by Pods (with configurable pull policy).
- **Garbage collect** unused images when disk space is low.
- Configurable thresholds: `imageGCHighThresholdPercent` (default 85%) and
  `imageGCLowThresholdPercent` (default 80%).

### 14.2.5 Status Manager

Batches Pod status updates and sends them to the API server. This prevents
overwhelming the API server with status updates during burst activity (e.g.,
node startup when many containers start simultaneously).

### 14.2.6 Certificate Manager

Handles kubelet certificate rotation. The kubelet needs a client certificate to
authenticate to the API server and a serving certificate for the kubelet's HTTPS
endpoint (used by `kubectl logs`, `kubectl exec`, and the metrics endpoint).

### 14.2.7 Node Status Manager

Periodically reports node conditions to the API server:

| Condition | Meaning |
|-----------|---------|
| `Ready` | Kubelet is healthy and ready to accept Pods |
| `MemoryPressure` | Node memory is low |
| `DiskPressure` | Node disk is low |
| `PIDPressure` | Too many processes on the node |
| `NetworkUnavailable` | Node network is not configured |

Also reports node capacity and allocatable resources:

```yaml
status:
  capacity:
    cpu: "8"
    memory: "32Gi"
    pods: "110"
  allocatable:
    cpu: "7800m"       # capacity minus reserved
    memory: "31Gi"
    pods: "110"
```

---

## 14.3 The Sync Loop

The sync loop is the beating heart of the kubelet. It runs continuously,
reconciling desired Pod state with actual container state.

### 14.3.1 Three Pod Sources

The kubelet accepts Pod definitions from three sources:

| Source | Description | Use Case |
|--------|-------------|----------|
| **API Server** | Watch for Pods with `spec.nodeName` matching this node | Normal Pods |
| **Static Pod (File)** | Watch a local directory (default `/etc/kubernetes/manifests/`) | Control plane components |
| **HTTP Endpoint** | Poll a URL for Pod manifests | Legacy, rarely used |

All three sources feed into the same sync loop. Static Pods are special: the
kubelet creates a **mirror Pod** in the API server so they appear in `kubectl get pods`,
but the kubelet is the sole manager.

### 14.3.2 Reconciliation Logic

On each sync iteration, for each Pod assigned to this node:

```
1. Get desired Pod spec (from API server / file / HTTP)
2. Get actual state (from PLEG / CRI)
3. Compare:
   a. Pod should exist but doesn't → Create sandbox + containers
   b. Pod exists but spec changed → Update (recreate containers)
   c. Pod should be deleted → Graceful shutdown sequence
   d. Container died unexpectedly → Apply restart policy
   e. Probes failing → Take action (restart / remove from service)
4. Report status back to API server
```

### 14.3.3 Pod Worker Goroutines

The kubelet spawns one **pod worker goroutine** per Pod. This goroutine serializes
all operations for its Pod (creation, updates, deletion) while allowing different
Pods to be processed in parallel.

```
Sync Loop
    │
    ├── Pod Worker (Pod A) ─── Create sandbox ─── Pull image ─── Start container
    │
    ├── Pod Worker (Pod B) ─── Kill container ─── Remove sandbox
    │
    ├── Pod Worker (Pod C) ─── Run liveness probe ─── Restart container
    │
    └── Pod Worker (Pod D) ─── (idle — Pod is healthy)
```

---

## 14.4 CRI — Container Runtime Interface

### 14.4.1 What CRI Is

CRI is a **gRPC API** that the kubelet uses to communicate with the container
runtime. It was introduced in Kubernetes 1.5 to decouple the kubelet from specific
runtime implementations.

Before CRI, the kubelet had hardcoded Docker support. Now, any runtime that
implements the CRI gRPC services can work with Kubernetes.

```
┌──────────┐     gRPC (unix socket)     ┌────────────────┐
│  Kubelet │ ◄─────────────────────────▶ │ Container      │
│          │  RuntimeService             │ Runtime        │
│          │  ImageService               │                │
│          │                             │ (containerd,   │
│          │                             │  CRI-O, etc.)  │
└──────────┘                             └────────────────┘
```

### 14.4.2 RuntimeService API

The RuntimeService handles Pod sandboxes and containers:

| Method | Purpose |
|--------|---------|
| `RunPodSandbox` | Create and start a Pod sandbox (network namespace, etc.) |
| `StopPodSandbox` | Stop all containers in the sandbox and tear down networking |
| `RemovePodSandbox` | Remove the sandbox after stopping |
| `PodSandboxStatus` | Get status of a sandbox |
| `ListPodSandbox` | List all sandboxes |
| `CreateContainer` | Create a container within a sandbox (but don't start it) |
| `StartContainer` | Start a created container |
| `StopContainer` | Stop a running container (with timeout) |
| `RemoveContainer` | Remove a stopped container |
| `ListContainers` | List all containers |
| `ContainerStatus` | Get status of a container |
| `UpdateContainerResources` | Update CPU/memory limits of a running container |
| `ExecSync` | Execute a command in a container synchronously |
| `Exec` | Execute a command in a container (streaming) |
| `Attach` | Attach to a running container (streaming) |
| `PortForward` | Forward ports to a Pod sandbox |

### 14.4.3 ImageService API

| Method | Purpose |
|--------|---------|
| `PullImage` | Pull an image from a registry |
| `ListImages` | List images on the node |
| `RemoveImage` | Remove an image |
| `ImageStatus` | Get status of an image |
| `ImageFsInfo` | Get filesystem info for image storage |

### 14.4.4 Hands-on: Using crictl

`crictl` is a command-line tool for interacting directly with CRI. It speaks the
same gRPC protocol the kubelet uses.

```bash
# Configure crictl to talk to containerd
cat > /etc/crictl.yaml <<EOF
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 10
debug: false
EOF

# List all Pods (sandboxes)
crictl pods
# Output:
# POD ID         CREATED       STATE   NAME                        NAMESPACE     ATTEMPT
# a1b2c3d4e5f6   2 hours ago   Ready   nginx-7c658794b9-x4q2z    default       0
# f6e5d4c3b2a1   3 hours ago   Ready   coredns-787d4945fb-9xk2p  kube-system   0

# List all containers
crictl ps
# Output:
# CONTAINER ID   IMAGE          CREATED       STATE     NAME      ATTEMPT   POD ID
# abc123def456   nginx:latest   2 hours ago   Running   nginx     0         a1b2c3d4e5f6

# Inspect a container
crictl inspect abc123def456

# Pull an image
crictl pull nginx:1.25

# List images
crictl images
# Output:
# IMAGE                    TAG       DIGEST       SIZE
# docker.io/library/nginx  1.25      sha256:...   67.3MB
# docker.io/library/nginx  latest    sha256:...   67.3MB

# Execute a command in a container
crictl exec -it abc123def456 /bin/sh

# View container logs
crictl logs abc123def456

# Get Pod sandbox stats
crictl statsp a1b2c3d4e5f6

# Stop and remove a container
crictl stop abc123def456
crictl rm abc123def456

# Stop and remove a Pod sandbox
crictl stopp a1b2c3d4e5f6
crictl rmp a1b2c3d4e5f6
```

### 14.4.5 Hands-on: Examining CRI Calls

To see exactly what CRI calls the kubelet makes, enable CRI-level logging:

```bash
# For containerd: enable debug logging
# Edit /etc/containerd/config.toml
[debug]
  level = "debug"

# Restart containerd
systemctl restart containerd

# Watch containerd logs for CRI calls
journalctl -u containerd -f | grep -E "RunPodSandbox|CreateContainer|StartContainer"
```

You'll see entries like:

```
RunPodSandbox for &PodSandboxMetadata{Name:nginx-pod,Uid:abc-123,Namespace:default}
CreateContainer for container nginx in sandbox abc-123
StartContainer for container-id def-456
```

---

## 14.5 Pod Sandbox

### 14.5.1 What a Sandbox Is

A **Pod sandbox** is the shared execution environment for all containers in a Pod.
It provides:

- **Network namespace** — all containers share the same IP address and port space.
- **IPC namespace** — containers can communicate via System V IPC and POSIX message queues.
- **PID namespace** (optional, if `shareProcessNamespace: true`) — containers can
  see each other's processes.
- **UTS namespace** — shared hostname.

The sandbox is created *before* any application containers and destroyed *after*
all containers stop.

### 14.5.2 The Pause Container

The sandbox is implemented as a special **pause container** (also called the
"infrastructure container"). It runs the `pause` binary — a minimal program that
does nothing but sleep:

```c
// Simplified pause.c — the actual Kubernetes pause container
#include <signal.h>
#include <unistd.h>

static void sigdown(int signo) { _exit(0); }

int main() {
    signal(SIGINT, sigdown);
    signal(SIGTERM, sigdown);
    for (;;) pause();  // sleep forever
    return 0;
}
```

**Why does the pause container exist?**

1. **Namespace holder:** Linux namespaces are destroyed when the last process
   using them exits. The pause container keeps the namespaces alive even if all
   application containers crash.

2. **PID 1:** In the PID namespace, the pause container is PID 1. It reaps
   zombie processes (orphaned children), which prevents PID exhaustion.

3. **Network anchor:** The network interface and IP address belong to the pause
   container's network namespace. Application containers join this namespace.

```
┌─────────────────── Pod Sandbox ───────────────────────┐
│                                                        │
│  Network Namespace: 10.244.1.5                        │
│  IPC Namespace: shared                                │
│                                                        │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────┐    │
│  │ pause         │  │ app           │  │ sidecar  │    │
│  │ (PID 1)      │  │ container     │  │ container│    │
│  │              │  │              │  │          │    │
│  │ Holds the    │  │ eth0 is from │  │ Shares   │    │
│  │ namespaces   │  │ sandbox      │  │ localhost│    │
│  └──────────────┘  └──────────────┘  └──────────┘    │
│                                                        │
│  All containers share:                                 │
│    - IP address (10.244.1.5)                          │
│    - Port space                                        │
│    - localhost                                         │
│    - IPC                                               │
│    - Volumes (if mounted)                              │
└────────────────────────────────────────────────────────┘
```

### 14.5.3 How Containers Share Localhost

Because all containers in a Pod share the same network namespace, they
communicate via `localhost`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-net
spec:
  containers:
  - name: web
    image: nginx:latest
    ports:
    - containerPort: 80
  - name: log-collector
    image: fluentd:latest
    # Can reach nginx at localhost:80
    env:
    - name: WEB_ENDPOINT
      value: "http://localhost:80"
```

---

## 14.6 Pod Lifecycle — End to End

### 14.6.1 The Complete Pod Startup Sequence

```
User/Controller                  API Server              etcd
     │                                │                     │
     │  kubectl apply -f pod.yaml     │                     │
     │───────────────────────────────▶│                     │
     │                                │  store Pod          │
     │                                │  (nodeName="")      │
     │                                │────────────────────▶│
     │                                │                     │
     │                                │◄────────────────────│
     │   201 Created                  │                     │
     │◄───────────────────────────────│                     │
     │                                │                     │
                                      │                     │
Scheduler                            │                     │
     │  Watch: Pod with nodeName=""   │                     │
     │◄───────────────────────────────│                     │
     │                                │                     │
     │  Filter → Score → Select node  │                     │
     │                                │                     │
     │  Bind: set nodeName="node-2"   │                     │
     │───────────────────────────────▶│                     │
     │                                │  update Pod         │
     │                                │────────────────────▶│
     │                                │                     │
                                      │                     │
Kubelet (on node-2)                  │                     │
     │  Watch: Pod with nodeName=     │                     │
     │         "node-2"              │                     │
     │◄───────────────────────────────│                     │
     │                                │                     │
     │  ┌─────────────────────────────┤                     │
     │  │ 1. Volume Setup             │                     │
     │  │    Attach + Mount volumes   │                     │
     │  │                             │                     │
     │  │ 2. Image Pull               │                     │
     │  │    Pull images for all      │                     │
     │  │    containers               │                     │
     │  │                             │                     │
     │  │ 3. Create Pod Sandbox       │                     │
     │  │    CRI: RunPodSandbox()     │                     │
     │  │    → pause container        │                     │
     │  │    → network namespace      │                     │
     │  │    → CNI: assign IP         │                     │
     │  │                             │                     │
     │  │ 4. Run Init Containers      │                     │
     │  │    (sequentially)           │                     │
     │  │    CreateContainer()        │                     │
     │  │    StartContainer()         │                     │
     │  │    Wait for completion      │                     │
     │  │                             │                     │
     │  │ 5. Run App Containers       │                     │
     │  │    (in parallel)            │                     │
     │  │    CreateContainer()        │                     │
     │  │    StartContainer()         │                     │
     │  │    Start postStart hook     │                     │
     │  │                             │                     │
     │  │ 6. Start Probes             │                     │
     │  │    Startup probe (if any)   │                     │
     │  │    Then liveness + readiness│                     │
     │  │                             │                     │
     │  │ 7. Pod becomes Ready        │                     │
     │  │    (all readiness probes    │                     │
     │  │     pass)                   │                     │
     │  └─────────────────────────────┤                     │
     │                                │                     │
     │  Update Pod status:            │                     │
     │  phase=Running, Ready=True     │                     │
     │───────────────────────────────▶│                     │
     │                                │────────────────────▶│
```

### 14.6.2 Pod Phases

| Phase | Description |
|-------|-------------|
| `Pending` | Pod accepted but not yet running. Waiting for scheduling, image pull, or volume setup. |
| `Running` | At least one container is running. |
| `Succeeded` | All containers terminated successfully (exit code 0). Won't restart. |
| `Failed` | All containers terminated. At least one failed (non-zero exit or killed by system). |
| `Unknown` | Pod status cannot be determined. Usually a communication issue with the node. |

### 14.6.3 Pod Conditions

| Condition | Meaning |
|-----------|---------|
| `PodScheduled` | Pod has been assigned to a node |
| `ContainersReady` | All containers in the Pod are ready |
| `Initialized` | All init containers completed successfully |
| `Ready` | Pod is ready to serve traffic (all readiness probes pass, all conditions met) |

---

## 14.7 Container States

Each container in a Pod is in one of three states:

### 14.7.1 Waiting

The container is not yet running. Common reasons:

| Reason | Description |
|--------|-------------|
| `ContainerCreating` | Container is being created (image pull in progress) |
| `ErrImagePull` | Failed to pull the image |
| `ImagePullBackOff` | Image pull failed repeatedly; backing off |
| `CrashLoopBackOff` | Container keeps crashing; kubelet delays restart |
| `CreateContainerConfigError` | Invalid container config (e.g., missing Secret) |

### 14.7.2 Running

The container is executing. The `startedAt` timestamp indicates when it started.

### 14.7.3 Terminated

The container exited. Key fields:
- `exitCode` — the process exit code (0 = success).
- `reason` — why it terminated (`Completed`, `Error`, `OOMKilled`).
- `startedAt` / `finishedAt` — timestamps.

### 14.7.4 Last Termination State

For debugging restarts, check `lastState`:

```bash
kubectl get pod my-pod -o jsonpath='{.status.containerStatuses[0].lastState}'
```

```json
{
  "terminated": {
    "exitCode": 137,
    "reason": "OOMKilled",
    "startedAt": "2024-01-15T10:30:00Z",
    "finishedAt": "2024-01-15T10:32:45Z"
  }
}
```

Exit code 137 = killed by SIGKILL (128 + 9). Often caused by OOM killer when the
container exceeds its memory limit.

---

## 14.8 Init Containers

### 14.8.1 What Init Containers Are

Init containers run **before** app containers start, and they run **sequentially**.
Each init container must complete successfully (exit 0) before the next one starts.
If any init container fails, the kubelet restarts it (subject to `restartPolicy`).

### 14.8.2 Use Cases

1. **Wait for dependencies:** Wait for a database to become available.
2. **Configuration generation:** Generate config files from templates or Secrets.
3. **Database migration:** Run schema migrations before the app starts.
4. **Security setup:** Set file permissions or download certificates.
5. **Data preloading:** Clone a git repository or download static assets.

### 14.8.3 Hands-on: Full Init Container Example

```yaml
# Pod with two init containers:
# 1. Wait for a service to be available
# 2. Fetch configuration from an external source
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
  # First init container: wait for the database service
  - name: wait-for-db
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      echo "Waiting for database service..."
      until nslookup postgres-service.default.svc.cluster.local; do
        echo "  Database service not found. Sleeping 2s..."
        sleep 2
      done
      echo "Database service is available!"

  # Second init container: generate config
  - name: generate-config
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      echo "Generating configuration..."
      cat > /config/app.conf <<'CONF'
      database.host=postgres-service
      database.port=5432
      database.name=myapp
      log.level=info
      CONF
      echo "Configuration written to /config/app.conf"
    volumeMounts:
    - name: config-volume
      mountPath: /config

  # App container: starts only after both init containers succeed
  containers:
  - name: app
    image: myapp:latest
    volumeMounts:
    - name: config-volume
      mountPath: /etc/app
      readOnly: true
    ports:
    - containerPort: 8080

  volumes:
  - name: config-volume
    emptyDir: {}
```

**Execution order:**
1. `wait-for-db` starts. Loops until DNS resolves `postgres-service`. Exits 0.
2. `generate-config` starts. Writes `/config/app.conf`. Exits 0.
3. `app` starts. Reads config from `/etc/app/app.conf`.

If `wait-for-db` keeps failing (service doesn't exist), the Pod stays in `Init:0/2`
state indefinitely. Check with:

```bash
kubectl get pod init-demo
# NAME        READY   STATUS     RESTARTS   AGE
# init-demo   0/1     Init:0/2   0          5m
```

---

## 14.9 Probes — Health Checking

Probes are the kubelet's mechanism for determining container health. They run
inside the kubelet process (not in the container) and make periodic checks.

### 14.9.1 Three Types of Probes

| Probe | Question | Action on Failure |
|-------|----------|-------------------|
| **Liveness** | Is the container alive? | Restart the container |
| **Readiness** | Is the container ready for traffic? | Remove from Service endpoints |
| **Startup** | Has the container finished starting? | Kill and restart (subject to restartPolicy) |

### 14.9.2 Probe Mechanisms

| Type | How It Works |
|------|-------------|
| `httpGet` | Send HTTP GET to container. 200-399 = success. |
| `tcpSocket` | Open TCP connection. Connection established = success. |
| `grpc` | Send gRPC health check. `SERVING` status = success. |
| `exec` | Run a command in the container. Exit code 0 = success. |

### 14.9.3 Probe Configuration

| Parameter | Default | Description |
|-----------|---------|-------------|
| `initialDelaySeconds` | 0 | Seconds to wait before first probe |
| `periodSeconds` | 10 | How often to probe |
| `timeoutSeconds` | 1 | Probe timeout |
| `failureThreshold` | 3 | Consecutive failures before taking action |
| `successThreshold` | 1 | Consecutive successes to be considered healthy |

### 14.9.4 How Probes Interact

```
Container starts
     │
     ▼
┌─────────────────┐      Startup probe defined?
│  Startup Probe   │──── NO ───▶ Skip to liveness/readiness
│  (if defined)    │
│                  │──── YES ──▶ Run startup probe repeatedly
│                  │              │
│                  │              ├── Success → Startup complete
│                  │              │              Start liveness + readiness
│                  │              │
│                  │              └── Failure (exceeds threshold) → Kill container
└─────────────────┘
     │ Startup succeeded
     ▼
┌──────────────────────────────────────┐
│  Liveness Probe     Readiness Probe  │
│  (runs periodically) (runs periodically)│
│                                      │
│  Failure → Restart  Failure → Remove │
│  container          from Service     │
│                     endpoints        │
│                                      │
│  Success → No       Success → Add to│
│  action             Service          │
│                     endpoints        │
└──────────────────────────────────────┘
```

### 14.9.5 Hands-on: Complete Probe Examples with Go HTTP Server

First, a Go server that implements proper health endpoints:

```go
// Package main implements a sample HTTP server with liveness, readiness,
// and startup health check endpoints suitable for Kubernetes probe
// configuration. The server demonstrates realistic patterns:
//
//   - Startup delay: the server needs time to load data before it can serve.
//   - Readiness toggle: the server can become temporarily unready (e.g.,
//     during a cache rebuild or database reconnection).
//   - Liveness: as long as the process is not deadlocked, it reports alive.
//
// These endpoints return HTTP 200 for healthy and HTTP 503 for unhealthy,
// which the kubelet's httpGet probe interprets as success/failure.
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

// server holds the shared state for the health-check demo server.
// Fields are protected by either atomic operations or a mutex to ensure
// safe concurrent access from the HTTP handlers and background goroutines.
type server struct {
	// started is set to 1 (atomically) once the server has finished
	// its startup sequence (loading data, warming caches, etc.).
	// The startup probe checks this field.
	started atomic.Int32

	// ready is set to 1 (atomically) when the server can handle traffic.
	// It may toggle between 0 and 1 during the server's lifetime (e.g.,
	// when a backend dependency becomes temporarily unavailable).
	// The readiness probe checks this field.
	ready atomic.Int32

	// mu protects mutable non-atomic state (currently unused but
	// reserved for future extension, e.g., connection pool state).
	mu sync.Mutex
}

// handleStartup is the startup probe handler. It returns 200 OK once
// the server has completed its initialization sequence, and 503 Service
// Unavailable before that.
//
// Kubernetes startup probes call this endpoint periodically. Until this
// endpoint returns 200, the kubelet will NOT run liveness or readiness
// probes, giving the server time to initialize without being killed.
func (s *server) handleStartup(w http.ResponseWriter, r *http.Request) {
	if s.started.Load() == 1 {
		w.WriteHeader(http.StatusOK)
		fmt.Fprintln(w, "started")
		return
	}
	w.WriteHeader(http.StatusServiceUnavailable)
	fmt.Fprintln(w, "not started")
}

// handleLiveness is the liveness probe handler. It always returns 200 OK
// as long as the HTTP server goroutine is running — which is a reasonable
// liveness signal. If the server were deadlocked, this handler would not
// respond, and the probe would time out (counted as a failure).
//
// In production, you might add checks here for:
//   - Goroutine deadlock detection (runtime.NumGoroutine() thresholds).
//   - Critical background worker health.
//   - File descriptor exhaustion.
//
// Do NOT check external dependencies in the liveness probe — that causes
// cascading restarts when a dependency goes down.
func (s *server) handleLiveness(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	fmt.Fprintln(w, "alive")
}

// handleReadiness is the readiness probe handler. It returns 200 OK when
// the server is ready to handle business traffic, and 503 when it is not.
//
// Unlike liveness, readiness CAN (and often should) check dependencies:
//   - Database connectivity.
//   - Cache warmth.
//   - Required config loaded.
//
// When readiness fails, the kubelet removes the Pod from Service endpoints,
// stopping traffic flow to this instance — but does NOT restart it.
func (s *server) handleReadiness(w http.ResponseWriter, r *http.Request) {
	if s.ready.Load() == 1 {
		w.WriteHeader(http.StatusOK)
		fmt.Fprintln(w, "ready")
		return
	}
	w.WriteHeader(http.StatusServiceUnavailable)
	fmt.Fprintln(w, "not ready")
}

// handleBusiness is the main application endpoint. It simulates serving
// real traffic by returning a simple response.
func (s *server) handleBusiness(w http.ResponseWriter, r *http.Request) {
	w.WriteHeader(http.StatusOK)
	fmt.Fprintln(w, "Hello from the application!")
}

// simulateStartup mimics a slow startup sequence — loading data, warming
// caches, establishing connections, etc. After the delay, the server
// transitions to started AND ready.
func (s *server) simulateStartup(delay time.Duration) {
	log.Printf("Starting initialization (will take %v)...", delay)
	time.Sleep(delay)
	s.started.Store(1)
	s.ready.Store(1)
	log.Println("Initialization complete. Server is started and ready.")
}

// main initializes the server, registers HTTP handlers, starts the
// startup simulation, and listens for graceful shutdown signals.
func main() {
	s := &server{}

	mux := http.NewServeMux()
	mux.HandleFunc("/healthz/startup", s.handleStartup)
	mux.HandleFunc("/healthz/live", s.handleLiveness)
	mux.HandleFunc("/healthz/ready", s.handleReadiness)
	mux.HandleFunc("/", s.handleBusiness)

	httpServer := &http.Server{
		Addr:    ":8080",
		Handler: mux,
	}

	// Simulate a slow startup in the background.
	go s.simulateStartup(15 * time.Second)

	// Graceful shutdown on SIGTERM/SIGINT.
	go func() {
		sigCh := make(chan os.Signal, 1)
		signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)
		sig := <-sigCh
		log.Printf("Received signal %v. Starting graceful shutdown...", sig)

		// Mark as not ready immediately — stop receiving new traffic.
		s.ready.Store(0)

		// Give in-flight requests time to complete.
		ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
		defer cancel()

		if err := httpServer.Shutdown(ctx); err != nil {
			log.Printf("HTTP server shutdown error: %v", err)
		}
		log.Println("Server stopped.")
	}()

	log.Println("Starting HTTP server on :8080")
	if err := httpServer.ListenAndServe(); err != http.ErrServerClosed {
		log.Fatalf("HTTP server error: %v", err)
	}
}
```

Now the Pod spec with all three probes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
  - name: app
    image: probe-demo:latest
    ports:
    - containerPort: 8080

    # Startup probe: give the app up to 60 seconds to start
    # (failureThreshold × periodSeconds = 30 × 2 = 60s)
    startupProbe:
      httpGet:
        path: /healthz/startup
        port: 8080
      periodSeconds: 2
      failureThreshold: 30
      timeoutSeconds: 1

    # Liveness probe: restart if the app is deadlocked
    # Starts AFTER startup probe succeeds
    livenessProbe:
      httpGet:
        path: /healthz/live
        port: 8080
      periodSeconds: 10
      failureThreshold: 3
      timeoutSeconds: 1

    # Readiness probe: remove from Service if not ready
    # Starts AFTER startup probe succeeds
    readinessProbe:
      httpGet:
        path: /healthz/ready
        port: 8080
      periodSeconds: 5
      failureThreshold: 2
      successThreshold: 1
      timeoutSeconds: 1
```

### 14.9.6 Other Probe Types

**TCP Socket Probe:**

```yaml
# Useful for non-HTTP services (databases, message brokers)
readinessProbe:
  tcpSocket:
    port: 5432
  periodSeconds: 10
```

**gRPC Probe (requires Kubernetes 1.24+):**

```yaml
# For gRPC services implementing grpc.health.v1.Health
readinessProbe:
  grpc:
    port: 50051
    service: "my.service.Name"  # optional: check specific service
  periodSeconds: 10
```

**Exec Probe:**

```yaml
# Run a command inside the container
livenessProbe:
  exec:
    command:
    - cat
    - /tmp/healthy
  periodSeconds: 5
```

### 14.9.7 Common Probe Mistakes

1. **Checking external dependencies in liveness probes.** If your database goes
   down, all your Pods restart — making the outage worse.

2. **Setting `initialDelaySeconds` instead of using a startup probe.** The startup
   probe is more flexible and doesn't delay liveness/readiness checks after restarts.

3. **Too aggressive thresholds.** `failureThreshold: 1` with `periodSeconds: 1`
   restarts containers on a single slow response.

4. **Forgetting `timeoutSeconds`.** A dangling connection can make probes pass
   (the connection is established) even though the application is hanging.

---

## 14.10 Resource Management

### 14.10.1 Requests vs Limits

Every container can specify resource **requests** and **limits**:

```yaml
resources:
  requests:
    cpu: "250m"      # 0.25 CPU cores
    memory: "128Mi"  # 128 mebibytes
  limits:
    cpu: "1"         # 1 CPU core
    memory: "512Mi"  # 512 mebibytes
```

| | Requests | Limits |
|---|---------|--------|
| **Purpose** | Scheduling guarantee | Maximum allowed usage |
| **Scheduler** | Uses requests to find a node | Ignores limits |
| **Runtime** | Sets baseline allocation | Enforces ceiling |
| **CPU** | `cpu.shares` in cgroup | `cpu.max` (CFS quota) |
| **Memory** | Used by scheduler only | `memory.max` (OOM kill) |

### 14.10.2 CPU: From Requests to Cgroups

**Requests → `cpu.shares`:**

```
cpu.shares = request_in_millicores × 1024 / 1000

Example: 250m → 256 shares
         500m → 512 shares
         1000m → 1024 shares (1 core)
```

CPU shares are **relative weights**. A container with 512 shares gets twice the
CPU time of a container with 256 shares — but only when there's contention. When
the CPU is idle, any container can use all available CPU.

**Limits → `cpu.max` (CFS quota):**

```
# cgroup v2: cpu.max = "$QUOTA $PERIOD"
# PERIOD defaults to 100000 (100ms)
# QUOTA = limit_in_millicores × PERIOD / 1000

Example: 1 CPU  → cpu.max = "100000 100000"  (100ms every 100ms)
         500m   → cpu.max = "50000 100000"    (50ms every 100ms)
         2 CPUs → cpu.max = "200000 100000"   (200ms every 100ms)
```

**CPU throttling:** When a container exceeds its CPU quota, the kernel suspends
it until the next period. This causes latency spikes.

### 14.10.3 Memory: From Requests to Cgroups

**Requests:** Memory requests are used only by the scheduler. They guarantee that
the node has at least that much memory available.

**Limits → `memory.max`:**

```
# cgroup v2: memory.max = limit_in_bytes

Example: 512Mi → memory.max = 536870912
```

When a container tries to allocate memory beyond its limit, the kernel's OOM killer
terminates the container. The kubelet reports this as `OOMKilled` in the container's
termination reason.

### 14.10.4 QoS Classes

Kubernetes assigns a **Quality of Service (QoS) class** to each Pod based on
resource configuration:

| QoS Class | Criteria | OOM Kill Priority |
|-----------|----------|-------------------|
| **Guaranteed** | Every container has requests = limits for both CPU and memory | Lowest (last to be killed) |
| **Burstable** | At least one container has a request or limit set, but not all equal | Middle |
| **BestEffort** | No container has any requests or limits | Highest (first to be killed) |

```yaml
# Guaranteed QoS
resources:
  requests:
    cpu: "500m"
    memory: "256Mi"
  limits:
    cpu: "500m"      # same as request
    memory: "256Mi"  # same as request

# Burstable QoS
resources:
  requests:
    cpu: "250m"
    memory: "128Mi"
  limits:
    cpu: "1"         # different from request
    memory: "512Mi"  # different from request

# BestEffort QoS — no resources specified at all
# (not recommended for production)
```

### 14.10.5 How QoS Affects OOM Kill Priority

When the node runs out of memory, the kernel OOM killer selects victims based on
the `oom_score_adj` value:

| QoS Class | oom_score_adj | Kill Priority |
|-----------|---------------|---------------|
| BestEffort | 1000 | First (highest score) |
| Burstable | 2-999 (based on memory request/capacity ratio) | Middle |
| Guaranteed | -997 | Last (lowest score) |

### 14.10.6 Hands-on: Observing Cgroup Settings

```bash
# Find the cgroup path for a container
CONTAINER_ID=$(crictl ps --name my-container -q)
CGROUP_PATH=$(crictl inspect $CONTAINER_ID | jq -r '.info.runtimeSpec.linux.cgroupsPath')

# For cgroup v2 (modern systems)
# CPU settings
cat /sys/fs/cgroup/$CGROUP_PATH/cpu.max
# Output: 100000 100000 (limit=1 CPU)

cat /sys/fs/cgroup/$CGROUP_PATH/cpu.weight
# Output: 39 (corresponds to ~1000 shares / 1024 * 39.32)

# Memory settings
cat /sys/fs/cgroup/$CGROUP_PATH/memory.max
# Output: 536870912 (512Mi)

cat /sys/fs/cgroup/$CGROUP_PATH/memory.current
# Output: 123456789 (current usage)

# OOM kill counter
cat /sys/fs/cgroup/$CGROUP_PATH/memory.events
# Output includes: oom_kill 0 (number of OOM kills)
```

---

## 14.11 Device Plugins

### 14.11.1 What Device Plugins Do

Device plugins allow nodes to advertise custom hardware resources (GPUs, FPGAs,
SR-IOV NICs, InfiniBand) to the kubelet. The kubelet then includes these resources
in the node's capacity, and the scheduler can use them for scheduling decisions.

### 14.11.2 Device Plugin Architecture

```
┌──────────────┐     gRPC (unix socket)     ┌──────────────────┐
│   Kubelet     │ ◄────────────────────────▶ │  Device Plugin   │
│               │                            │  (e.g., NVIDIA)  │
│  device       │  1. Register(ResourceName) │                  │
│  plugin       │  2. ListAndWatch(devices)  │  Monitors        │
│  manager      │  3. Allocate(deviceIDs)    │  hardware and    │
│               │                            │  reports devices │
└──────────────┘                             └──────────────────┘
```

### 14.11.3 Device Plugin API

The device plugin registers with the kubelet via a Unix socket at
`/var/lib/kubelet/device-plugins/`. The protocol:

1. **Register:** Plugin tells kubelet its resource name (e.g., `nvidia.com/gpu`).
2. **ListAndWatch:** Plugin streams the list of available devices to kubelet.
   Updates are sent when devices become healthy/unhealthy.
3. **Allocate:** When a Pod requests the device, kubelet calls Allocate. The plugin
   returns the device files, environment variables, and mounts to inject into
   the container.

### 14.11.4 Hands-on: Examining Device Plugin Registration

```bash
# List registered device plugins
ls /var/lib/kubelet/device-plugins/

# Check node capacity for device resources
kubectl describe node <node-name> | grep -A 5 "Capacity"
# Capacity:
#   cpu:                8
#   memory:             32Gi
#   nvidia.com/gpu:     2

# Request a GPU in a Pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  containers:
  - name: cuda
    image: nvidia/cuda:12.0-base
    resources:
      limits:
        nvidia.com/gpu: 1
EOF

# Verify GPU allocation
kubectl exec gpu-pod -- nvidia-smi
```

---

## 14.12 Static Pods

### 14.12.1 What Static Pods Are

Static Pods are managed directly by the kubelet without the API server's
involvement. The kubelet watches a local directory (default:
`/etc/kubernetes/manifests/`) and creates/destroys Pods based on the files
it finds there.

### 14.12.2 How Static Pods Work

1. Kubelet scans the static Pod directory every 20 seconds (configurable via
   `fileCheckFrequency`).
2. For each YAML/JSON file found, kubelet creates the Pod.
3. Kubelet creates a **mirror Pod** in the API server — a read-only copy that
   appears in `kubectl get pods`. The mirror Pod cannot be edited or deleted
   through the API; deleting it just recreates it.
4. To remove a static Pod, delete the file from the manifest directory.

### 14.12.3 Use Case: Control Plane Components

In a `kubeadm`-bootstrapped cluster, control plane components are static Pods:

```bash
ls /etc/kubernetes/manifests/
# etcd.yaml
# kube-apiserver.yaml
# kube-controller-manager.yaml
# kube-scheduler.yaml
```

This creates a chicken-and-egg solution: the kubelet can start the API server
without the API server being available yet.

### 14.12.4 Hands-on: Creating a Static Pod

```bash
# Create a static Pod manifest
cat > /etc/kubernetes/manifests/static-nginx.yaml <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: static-nginx
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "200m"
        memory: "128Mi"
EOF

# Wait a few seconds for kubelet to pick it up
sleep 5

# The Pod appears with the node name appended
kubectl get pods
# NAME                       READY   STATUS    RESTARTS   AGE
# static-nginx-node-1        1/1     Running   0          10s

# The Pod has an annotation indicating it's a mirror Pod
kubectl get pod static-nginx-node-1 -o jsonpath='{.metadata.annotations}'
# {"kubernetes.io/config.mirror":"...","kubernetes.io/config.source":"file"}

# To remove: delete the file
rm /etc/kubernetes/manifests/static-nginx.yaml
```

---

## 14.13 Graceful Shutdown

### 14.13.1 Pod Termination Flow

When a Pod is deleted, the kubelet orchestrates a graceful shutdown:

```
kubectl delete pod my-pod
         │
         ▼
┌─────────────────────┐
│ API Server sets      │
│ deletionTimestamp    │
│ and                  │
│ deletionGracePeriod │
│ (default: 30s)       │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Kubelet receives     │
│ Pod update           │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────────────────────────────────┐
│ 1. Mark Pod as Terminating                       │
│    - Remove from Service endpoints              │
│    - readinessGates become false                │
│                                                  │
│ 2. Run preStop hooks (all containers in parallel)│
│    - exec, httpGet, or sleep                    │
│    - Blocks until hook completes or             │
│      grace period expires                        │
│                                                  │
│ 3. Send SIGTERM to PID 1 of each container      │
│    - Container processes receive the signal      │
│    - They should begin graceful shutdown          │
│                                                  │
│ 4. Wait for containers to exit                   │
│    - Up to terminationGracePeriodSeconds         │
│    - (minus time spent on preStop hooks)         │
│                                                  │
│ 5. Send SIGKILL if still running                 │
│    - Forceful termination — cannot be caught     │
│                                                  │
│ 6. Clean up:                                     │
│    - Remove containers                           │
│    - Remove sandbox                              │
│    - Unmount volumes                             │
│    - Remove Pod from API server                  │
└─────────────────────────────────────────────────┘
```

**Critical timing detail:** The preStop hook and SIGTERM share the same grace period.
If your preStop hook takes 25 seconds and the grace period is 30 seconds, your
application only has 5 seconds to shut down after receiving SIGTERM.

### 14.13.2 terminationGracePeriodSeconds

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: graceful-pod
spec:
  terminationGracePeriodSeconds: 60  # default is 30
  containers:
  - name: app
    image: myapp:latest
    lifecycle:
      preStop:
        exec:
          command:
          - /bin/sh
          - -c
          - "sleep 5"  # give LB time to drain connections
```

### 14.13.3 Hands-on: Implementing Graceful Shutdown in Go

```go
// Package main demonstrates a production-ready HTTP server with proper
// graceful shutdown handling for Kubernetes. The server:
//
//  1. Traps SIGTERM (sent by kubelet) and SIGINT (for local development).
//  2. Immediately stops accepting new connections.
//  3. Waits for in-flight requests to complete (up to a deadline).
//  4. Flushes any buffered data (logs, metrics, message queues).
//  5. Closes database connections and releases resources.
//  6. Exits with code 0 (clean shutdown).
//
// This pattern ensures zero-downtime deployments when combined with
// a readiness probe and a preStop hook that allows load balancers
// to drain connections before the server stops.
package main

import (
	"context"
	"errors"
	"fmt"
	"log"
	"net/http"
	"os"
	"os/signal"
	"sync/atomic"
	"syscall"
	"time"
)

// activeRequests tracks the number of in-flight HTTP requests.
// This is used for observability during shutdown — you can log
// how many requests were still being processed when shutdown began.
var activeRequests atomic.Int64

// requestTracker is middleware that increments/decrements the active
// request counter. This gives the shutdown handler visibility into
// how many requests are still in progress.
func requestTracker(next http.Handler) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		activeRequests.Add(1)
		defer activeRequests.Add(-1)
		next.ServeHTTP(w, r)
	})
}

// handleWork simulates a slow request handler. In production, this
// would be your actual business logic — database queries, API calls, etc.
func handleWork(w http.ResponseWriter, r *http.Request) {
	// Simulate work that takes 1-5 seconds
	time.Sleep(2 * time.Second)
	w.WriteHeader(http.StatusOK)
	fmt.Fprintf(w, "Request processed at %s\n", time.Now().Format(time.RFC3339))
}

// flushBuffers simulates flushing any buffered data before shutdown.
// In production, this would flush log buffers, metric collectors,
// message queue producers, etc.
func flushBuffers() {
	log.Println("Flushing buffered data...")
	time.Sleep(500 * time.Millisecond) // Simulate flush time
	log.Println("Buffers flushed.")
}

// closeDatabaseConnections simulates closing database connections
// gracefully. In production, call db.Close() on your *sql.DB instances.
func closeDatabaseConnections() {
	log.Println("Closing database connections...")
	time.Sleep(200 * time.Millisecond) // Simulate close time
	log.Println("Database connections closed.")
}

// main sets up the HTTP server and orchestrates graceful shutdown.
// The shutdown sequence is:
//
//  1. Receive SIGTERM from kubelet.
//  2. Stop accepting new connections (http.Server.Shutdown).
//  3. Wait for in-flight requests to complete.
//  4. Flush buffers and close connections.
//  5. Exit cleanly.
func main() {
	mux := http.NewServeMux()
	mux.HandleFunc("/work", handleWork)
	mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		fmt.Fprintln(w, "ok")
	})

	srv := &http.Server{
		Addr:    ":8080",
		Handler: requestTracker(mux),

		// ReadHeaderTimeout prevents slowloris attacks and ensures
		// shutdown doesn't wait forever on misbehaving clients.
		ReadHeaderTimeout: 10 * time.Second,

		// IdleTimeout controls how long keep-alive connections stay open.
		// During shutdown, idle connections are closed immediately.
		IdleTimeout: 120 * time.Second,
	}

	// Channel to signal that shutdown is complete.
	shutdownComplete := make(chan struct{})

	go func() {
		// Create a channel that receives SIGTERM and SIGINT.
		// SIGTERM: sent by kubelet during Pod termination.
		// SIGINT:  sent by Ctrl+C during local development.
		sigCh := make(chan os.Signal, 1)
		signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)

		// Block until we receive a signal.
		sig := <-sigCh
		log.Printf("Received signal: %v", sig)
		log.Printf("Active requests at shutdown: %d", activeRequests.Load())

		// Create a deadline for the entire shutdown sequence.
		// This should be less than terminationGracePeriodSeconds minus
		// the preStop hook duration to ensure clean exit before SIGKILL.
		//
		// Example: terminationGracePeriodSeconds=60, preStop=5s
		//          → shutdown deadline should be ~50s (with margin)
		ctx, cancel := context.WithTimeout(context.Background(), 50*time.Second)
		defer cancel()

		// Phase 1: Stop accepting new connections and wait for in-flight
		// requests to complete. This is the key method for graceful shutdown.
		log.Println("Shutting down HTTP server (waiting for in-flight requests)...")
		if err := srv.Shutdown(ctx); err != nil {
			log.Printf("HTTP server shutdown error: %v", err)
		}
		log.Println("HTTP server stopped. All in-flight requests completed.")

		// Phase 2: Flush any buffered data.
		flushBuffers()

		// Phase 3: Close external connections.
		closeDatabaseConnections()

		log.Println("Graceful shutdown complete.")
		close(shutdownComplete)
	}()

	// Start serving.
	log.Println("Starting HTTP server on :8080")
	if err := srv.ListenAndServe(); !errors.Is(err, http.ErrServerClosed) {
		log.Fatalf("HTTP server error: %v", err)
	}

	// Wait for shutdown to complete before exiting.
	<-shutdownComplete
	log.Println("Process exiting.")
}
```

Matching Pod spec with preStop hook for connection draining:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: graceful-app
spec:
  terminationGracePeriodSeconds: 60
  containers:
  - name: app
    image: graceful-app:latest
    ports:
    - containerPort: 8080
    lifecycle:
      preStop:
        exec:
          command:
          - /bin/sh
          - -c
          # Sleep gives the Endpoints controller and kube-proxy time to
          # remove this Pod from Service endpoints and update iptables/IPVS
          # rules. Without this delay, the Pod may receive traffic after
          # it begins shutting down.
          - "sleep 5"
    readinessProbe:
      httpGet:
        path: /healthz
        port: 8080
      periodSeconds: 5
    resources:
      requests:
        cpu: "100m"
        memory: "64Mi"
      limits:
        cpu: "500m"
        memory: "256Mi"
```

**Why the preStop sleep?**

When a Pod is deleted:
1. The API server sets `deletionTimestamp`.
2. The endpoints controller removes the Pod from Service endpoints.
3. kube-proxy updates iptables/IPVS rules.

Steps 2 and 3 happen *asynchronously*. Without the preStop sleep, SIGTERM may
arrive before kube-proxy has finished updating rules, causing some traffic to
still reach the terminating Pod. The 5-second sleep gives kube-proxy time to
catch up.

---

## 14.14 Ephemeral Containers

### 14.14.1 What Ephemeral Containers Are

Ephemeral containers are temporary containers added to a running Pod for
**debugging purposes**. They are added via the Pod's `ephemeralContainers`
subresource — no restart required.

Key properties:
- They are never automatically restarted.
- They have no ports, readiness probes, or liveness probes.
- They share the Pod's namespaces (network, IPC, PID if enabled).
- They cannot be removed once added (but they can be stopped).
- Resource limits are not enforced on them in most configurations.

### 14.14.2 Why Ephemeral Containers Exist

Production containers are often built from minimal images (distroless, scratch)
that lack debugging tools. Ephemeral containers let you inject a debugging
container with `bash`, `curl`, `tcpdump`, `strace`, etc., without modifying the
production image.

### 14.14.3 Hands-on: Using Ephemeral Containers

```bash
# Debug a running Pod with a shell container
kubectl debug -it my-pod --image=busybox:1.36 --target=my-container
# This:
# 1. Adds an ephemeral container with image busybox
# 2. Shares PID namespace with "my-container" (--target)
# 3. Opens an interactive terminal

# Inside the ephemeral container, you can:
# - See processes from the target container:
ps aux
# PID   USER     TIME  COMMAND
# 1     root     0:05  /app/server          ← target container's process
# 50    root     0:00  /bin/sh              ← your debug shell

# - Access the target container's filesystem via /proc:
ls /proc/1/root/app/

# - Use networking tools (shared network namespace):
wget -O- localhost:8080/healthz

# - Check environment variables:
cat /proc/1/environ | tr '\0' '\n'

# Debug with a more fully-featured image
kubectl debug -it my-pod --image=nicolaka/netshoot --target=my-container
# netshoot includes: tcpdump, curl, dig, nmap, iperf3, strace, etc.

# Capture network traffic
tcpdump -i eth0 -w /tmp/capture.pcap

# Check DNS resolution
dig kubernetes.default.svc.cluster.local

# Trace system calls of the target process
strace -p 1 -f -e trace=network
```

### 14.14.4 Creating a Copy for Debugging

Sometimes you need to debug with a modified Pod spec (e.g., different command,
added volumes). `kubectl debug` can create a copy:

```bash
# Create a copy of the Pod with a different command
kubectl debug my-pod -it --copy-to=my-pod-debug --container=my-container -- /bin/sh

# Create a copy with the image changed (useful for distroless containers)
kubectl debug my-pod -it --copy-to=my-pod-debug \
  --set-image=my-container=ubuntu:22.04

# Create a copy that shares process namespace
kubectl debug my-pod -it --copy-to=my-pod-debug \
  --share-processes \
  --image=busybox:1.36
```

### 14.14.5 Debugging Node Issues

`kubectl debug` can also create a Pod for debugging node-level issues:

```bash
# Create a debugging Pod on a specific node
kubectl debug node/my-node -it --image=ubuntu:22.04

# This creates a Pod with:
# - hostPID, hostNetwork, hostIPC enabled
# - The node's root filesystem mounted at /host
# - Runs on the specified node

# Inside the debug Pod:
chroot /host   # access the node's filesystem

# Check kubelet logs
journalctl -u kubelet --no-pager -n 50

# Check containerd
crictl ps

# Check node resources
top
df -h
free -m
```

---

## 14.15 Summary

The kubelet is where Kubernetes meets the operating system:

| Component | Responsibility |
|-----------|---------------|
| **Sync Loop** | Main reconciliation loop — desired vs actual state |
| **PLEG** | Detects container state changes via CRI polling |
| **CRI** | gRPC API to container runtime (containerd/CRI-O) |
| **Pod Sandbox** | Shared namespaces via the pause container |
| **Init Containers** | Sequential pre-startup tasks |
| **Probes** | Health checking — startup, liveness, readiness |
| **Volume Manager** | Attach, mount, unmount, detach |
| **Image Manager** | Pull images, garbage collect |
| **Status Manager** | Report Pod/Node status to API server |
| **Device Plugins** | Advertise and allocate custom hardware |
| **Static Pods** | Kubelet-managed Pods without API server |
| **Graceful Shutdown** | SIGTERM → preStop → grace period → SIGKILL |
| **Ephemeral Containers** | Debug running Pods without restart |

Key takeaways:
- The kubelet is the **only** component that runs containers. Everything else
  just writes to the API server.
- **CRI decouples** Kubernetes from specific container runtimes.
- **Probes** are the foundation of self-healing. Get them right.
- **Resource requests** affect scheduling; **limits** affect enforcement.
- **QoS classes** determine OOM kill priority — always set resources.
- **Graceful shutdown** requires coordination between preStop hooks, SIGTERM
  handling, and kube-proxy rule updates.

---

## Exercises

1. **CRI Exploration:** Use `crictl` to list all Pods, containers, and images on
   a node. Inspect a container to find its cgroup path, then examine the cgroup
   CPU and memory settings.

2. **Probe Lab:** Deploy the Go health-check server from Section 14.9.5. Configure
   all three probes. Kill the startup endpoint and observe the container restart.
   Make the readiness endpoint fail and verify the Pod is removed from Service
   endpoints.

3. **Resource Lab:** Create two Pods — one Guaranteed (requests=limits) and one
   BestEffort (no resources). Run a memory stress test in each. Observe which
   Pod gets OOM-killed first.

4. **Graceful Shutdown Lab:** Deploy the Go graceful shutdown server from Section
   14.13.3. Send a long-running request, then delete the Pod. Verify that the
   request completes before the server shuts down.

5. **Init Container Lab:** Create a Pod with an init container that waits for a
   Service. Deploy the Pod first (it should be stuck in `Init:0/1`). Then create
   the Service and watch the init container succeed.

6. **Ephemeral Container Lab:** Deploy a Pod using a distroless image. Use
   `kubectl debug` with `--target` to attach and inspect the running process.

7. **Static Pod Lab:** Create a static Pod manifest in `/etc/kubernetes/manifests/`.
   Verify the mirror Pod appears in `kubectl get pods`. Delete the mirror Pod via
   `kubectl` and observe it being recreated by the kubelet.
