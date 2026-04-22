# Chapter 10: Kubernetes Architecture Deep Dive

> *"Kubernetes is a platform for building platforms."* — Kelsey Hightower

Kubernetes has become the de facto standard for container orchestration, but beneath the
`kubectl apply` commands lies a sophisticated distributed system built on decades of systems
programming wisdom. This chapter dissects Kubernetes from the inside out — every component,
every API call, every reconciliation loop.

We will trace how a simple `kubectl apply -f deployment.yaml` triggers a cascade of events
across a dozen components, each communicating through well-defined interfaces. By the end
of this chapter, you will understand Kubernetes not as a magic black box, but as an elegantly
designed collection of control loops, watches, and state machines.

**Prerequisites:** Chapter 9 (Containers & Container Runtimes), basic understanding of
distributed systems concepts.

---

## 10.1 The Declarative Model — Kubernetes' Core Philosophy

Before diving into components, we must understand the single most important idea in
Kubernetes: **declarative configuration**. Every design decision in Kubernetes flows from
this principle.

### 10.1.1 Desired State vs Current State

In an imperative system, you tell the system *what to do*:

```bash
# Imperative: a sequence of commands
docker run nginx
docker run nginx
docker run nginx
# Now I have 3 nginx containers... maybe. What if one failed?
```

In a declarative system, you tell the system *what you want*:

```yaml
# Declarative: desired state
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3           # I want 3 replicas. Period.
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
```

The system's job is to **continuously** make reality match the declaration. If a pod dies,
Kubernetes doesn't need a human to notice and run `docker run` again — the Deployment
controller sees the discrepancy and fixes it automatically.

This is not just a convenience feature. It is the **architectural foundation** of
Kubernetes' resilience.

### 10.1.2 The Control Loop: Observe → Diff → Act

Every controller in Kubernetes implements the same fundamental pattern:

```
┌─────────────────────────────────────────────────────────┐
│                   THE CONTROL LOOP                       │
│                                                          │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│   │          │    │          │    │          │          │
│   │ OBSERVE  │───▶│   DIFF   │───▶│   ACT    │          │
│   │          │    │          │    │          │          │
│   │ Read the │    │ Compare  │    │ Take     │          │
│   │ current  │    │ current  │    │ action   │          │
│   │ state of │    │ state to │    │ to move  │          │
│   │ the world│    │ desired  │    │ toward   │          │
│   │          │    │ state    │    │ desired  │          │
│   └──────────┘    └──────────┘    │ state    │          │
│        ▲                          └────┬─────┘          │
│        │                               │                │
│        └───────────────────────────────┘                │
│                  (repeat forever)                        │
└─────────────────────────────────────────────────────────┘
```

This pattern, borrowed from control theory (think: a thermostat), is implemented by every
single controller:

| Controller | Observes | Desired State | Action |
|---|---|---|---|
| ReplicaSet controller | Running pods | `spec.replicas: 3` | Create/delete pods |
| Deployment controller | ReplicaSets | `spec.template` | Create new RS, scale old RS down |
| Node controller | Node heartbeats | All nodes healthy | Mark unresponsive nodes as `NotReady` |
| Scheduler | Unscheduled pods | Every pod on a node | Bind pod to a node |
| kubelet | Pod specs for its node | Containers running | Start/stop containers via CRI |

### 10.1.3 Level-Triggered vs Edge-Triggered Reconciliation

This distinction is critical to understanding Kubernetes' reliability model.

**Edge-triggered** systems react to *changes* (events):
- "A pod was deleted" → create a new one
- Problem: What if you miss the event? The system becomes inconsistent.

**Level-triggered** systems react to *state* (levels):
- "There should be 3 pods, but I see 2" → create one more
- Even if you miss events, the next reconciliation fixes things.

```
Edge-Triggered (Fragile):
    Event: "Pod deleted"  ──▶  Action: "Create pod"
    
    What if the event is lost?  💥  System stays broken.

Level-Triggered (Robust):
    State: "desired=3, actual=2" ──▶  Action: "Create 1 pod"
    
    Even if events are lost, the next check fixes it. ✅
```

Kubernetes is **level-triggered**. Controllers don't just react to watch events — they
periodically re-list all resources and reconcile. This is why Kubernetes self-heals:

```go
// Simplified reconciliation — level-triggered, not edge-triggered.
// The controller doesn't care *why* the count is wrong, only *that* it is wrong.
func (c *ReplicaSetController) Reconcile(rs *appsv1.ReplicaSet) error {
	// OBSERVE: What is the current state?
	currentPods, err := c.podLister.Pods(rs.Namespace).List(
		labels.SelectorFromSet(rs.Spec.Selector.MatchLabels),
	)
	if err != nil {
		return err
	}

	// DIFF: How does current state compare to desired state?
	desired := int(*rs.Spec.Replicas)
	current := len(currentPods)
	diff := desired - current

	// ACT: Take action to converge toward desired state.
	switch {
	case diff > 0:
		// Too few pods — create more.
		for i := 0; i < diff; i++ {
			c.createPod(rs)
		}
	case diff < 0:
		// Too many pods — delete extras.
		// Select pods to delete using a deterministic policy
		// (e.g., newest first, or spread across nodes).
		podsToDelete := selectPodsToDelete(currentPods, -diff)
		for _, pod := range podsToDelete {
			c.deletePod(pod)
		}
	default:
		// Perfect — nothing to do.
	}
	return nil
}
```

### 10.1.4 Why Declarative Beats Imperative for Distributed Systems

In a distributed system, **anything can fail at any time**: networks partition, nodes
crash, processes restart. The declarative model handles this gracefully because:

1. **Idempotency**: Applying the same desired state multiple times has the same effect.
   `kubectl apply` is safe to run repeatedly.

2. **Convergence**: No matter what state the system is in, the control loops always push
   toward the desired state. There is no sequence of commands to replay.

3. **Self-healing**: If a node dies and takes pods with it, controllers notice the
   discrepancy and recreate pods elsewhere. No human intervention needed.

4. **Auditability**: The desired state is a YAML file in version control. You can diff
   it, review it, roll it back. Try doing that with a sequence of imperative commands.

5. **Concurrency safety**: Multiple controllers can operate simultaneously without
   stepping on each other, because each only cares about its own slice of desired state
   and uses optimistic concurrency (resourceVersion) to avoid conflicts.

---

## 10.2 High-Level Architecture

Kubernetes separates into two planes:

- **Control Plane**: The "brain" — makes decisions about the cluster.
- **Data Plane** (Worker Nodes): The "body" — runs the actual workloads.

```
┌─────────────────────────────── KUBERNETES CLUSTER ────────────────────────────────┐
│                                                                                    │
│  ┌────────────────────────── CONTROL PLANE ──────────────────────────┐             │
│  │                                                                    │             │
│  │  ┌──────────────┐   ┌──────────────────┐   ┌──────────────────┐  │             │
│  │  │              │   │                  │   │                  │  │             │
│  │  │    etcd      │◀──│  kube-apiserver  │──▶│  kube-scheduler  │  │             │
│  │  │              │   │                  │   │                  │  │             │
│  │  │  (cluster    │   │  (central hub,   │   │  (assigns pods   │  │             │
│  │  │   state      │   │   REST API,      │   │   to nodes)      │  │             │
│  │  │   store)     │   │   authn/authz)   │   │                  │  │             │
│  │  └──────────────┘   └────────┬─────────┘   └──────────────────┘  │             │
│  │                              │                                    │             │
│  │                              │  ┌──────────────────────────────┐  │             │
│  │                              │  │                              │  │             │
│  │                              ├──│  kube-controller-manager     │  │             │
│  │                              │  │                              │  │             │
│  │                              │  │  (deployment, replicaset,    │  │             │
│  │                              │  │   node, job, sa controllers) │  │             │
│  │                              │  └──────────────────────────────┘  │             │
│  │                              │                                    │             │
│  │                              │  ┌──────────────────────────────┐  │             │
│  │                              └──│  cloud-controller-manager    │  │             │
│  │                                 │  (cloud LB, routes, nodes)   │  │             │
│  │                                 └──────────────────────────────┘  │             │
│  └────────────────────────────────────────────────────────────────────┘             │
│                              │ HTTPS (Watch/List)                                  │
│                              ▼                                                     │
│  ┌────────────────────────── DATA PLANE (Worker Nodes) ──────────────────────────┐ │
│  │                                                                                │ │
│  │  ┌──── Node 1 ────────────────┐    ┌──── Node 2 ────────────────┐            │ │
│  │  │                            │    │                            │            │ │
│  │  │  ┌────────┐  ┌──────────┐ │    │  ┌────────┐  ┌──────────┐ │            │ │
│  │  │  │kubelet │  │kube-proxy│ │    │  │kubelet │  │kube-proxy│ │            │ │
│  │  │  └───┬────┘  └──────────┘ │    │  └───┬────┘  └──────────┘ │            │ │
│  │  │      │ CRI                │    │      │ CRI                │            │ │
│  │  │  ┌───▼──────────────────┐ │    │  ┌───▼──────────────────┐ │            │ │
│  │  │  │  Container Runtime   │ │    │  │  Container Runtime   │ │            │ │
│  │  │  │  (containerd/CRI-O) │ │    │  │  (containerd/CRI-O) │ │            │ │
│  │  │  └──────────────────────┘ │    │  └──────────────────────┘ │            │ │
│  │  │                            │    │                            │            │ │
│  │  │  ┌─Pod─┐ ┌─Pod─┐ ┌─Pod─┐ │    │  ┌─Pod─┐ ┌─Pod─┐        │            │ │
│  │  │  │nginx│ │redis│ │app  │ │    │  │app  │ │db   │        │            │ │
│  │  │  └─────┘ └─────┘ └─────┘ │    │  └─────┘ └─────┘        │            │ │
│  │  └────────────────────────────┘    └────────────────────────────┘            │ │
│  └────────────────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

**Every arrow in this diagram is an API call or watch connection.** Let's trace each one:

| Connection | Protocol | Direction | Purpose |
|---|---|---|---|
| apiserver → etcd | gRPC | apiserver initiates | Read/write all cluster state |
| apiserver ← etcd | gRPC watch | etcd pushes changes | Notify apiserver of state changes |
| scheduler → apiserver | HTTPS watch | scheduler watches | Watch for unscheduled pods |
| scheduler → apiserver | HTTPS POST | scheduler writes | Bind pod to node |
| controller-manager → apiserver | HTTPS watch | CM watches | Watch resources to reconcile |
| controller-manager → apiserver | HTTPS CRUD | CM writes | Create/update/delete resources |
| kubelet → apiserver | HTTPS watch | kubelet watches | Watch for pods assigned to its node |
| kubelet → apiserver | HTTPS PATCH | kubelet writes | Update pod/node status |
| kubelet → container runtime | gRPC (CRI) | kubelet initiates | Start/stop containers |
| cloud-controller → apiserver | HTTPS watch | CCM watches | Watch nodes, services |
| cloud-controller → cloud API | HTTPS | CCM initiates | Provision LBs, routes |
| kube-proxy → apiserver | HTTPS watch | proxy watches | Watch Services and Endpoints |

**Critical design point:** The API server is the **only** component that talks to etcd.
Every other component talks exclusively to the API server. This creates a clean
architecture with a single source of truth and a single point of access control.

---

## 10.3 Control Plane Components — Deep Dive

### 10.3.1 etcd — The Cluster State Store

etcd is a distributed key-value store that holds **all** Kubernetes cluster state.
When you create a Deployment, a Service, a ConfigMap — it all lives in etcd.

#### What etcd Stores

etcd stores every Kubernetes object as a serialized Protocol Buffer (or JSON) at a
well-defined key path:

```
/registry/<resource-type>/<namespace>/<name>

Examples:
/registry/pods/default/nginx-7d9b8c6f5-abc12
/registry/deployments/kube-system/coredns
/registry/services/specs/default/kubernetes
/registry/secrets/default/my-secret
/registry/configmaps/default/my-config
/registry/nodes/worker-1
/registry/namespaces/default
```

You can inspect etcd directly (though you should never modify it by hand in production):

```bash
# List all keys in etcd (NEVER do this in production — it dumps all secrets!)
ETCDCTL_API=3 etcdctl get /registry --prefix --keys-only | head -50

# Get a specific pod's stored data
ETCDCTL_API=3 etcdctl get /registry/pods/default/nginx-7d9b8c6f5-abc12

# Watch for changes to all pods
ETCDCTL_API=3 etcdctl watch /registry/pods --prefix
```

#### The Watch Mechanism

etcd's watch feature is the nervous system of Kubernetes. When a controller wants to
know about changes to pods, it doesn't poll — it establishes a **watch** on the
relevant key prefix. etcd streams changes in real time:

```
┌─────────────┐         watch /registry/pods/*         ┌─────────┐
│  Controller  │ ──────────────────────────────────────▶ │  etcd   │
│  (via API    │ ◀─────────────────────────────────────  │         │
│   server)    │   Event: PUT /registry/pods/default/x   │         │
│              │   Event: DELETE /registry/pods/ns/y      │         │
└─────────────┘                                          └─────────┘
```

Each watch event includes:
- **Type**: PUT (create/update) or DELETE
- **Key**: The full key path
- **Value**: The new value (for PUT) or previous value (for DELETE)
- **ModRevision**: A monotonically increasing revision number

The revision number is crucial — it allows clients to resume watches from where they
left off, even after disconnection. This is how Kubernetes achieves reliable event
delivery without message queues.

#### Why etcd is the Single Source of Truth

etcd uses the **Raft consensus protocol** to replicate data across multiple nodes.
This guarantees:

- **Linearizable reads**: Every read sees the most recent write.
- **Consistent ordering**: All nodes agree on the order of operations.
- **Durability**: Committed writes survive node failures.

This means every Kubernetes component can trust that what it reads from etcd (via the
API server) is the true, consistent state of the cluster.

> **📖 Reference:** Chapter 11 covers etcd in full depth — Raft consensus, compaction,
> defragmentation, performance tuning, and disaster recovery.

---

### 10.3.2 kube-apiserver — The Central Hub

The API server is the **front door** to the Kubernetes cluster. Every component — kubectl,
controllers, kubelet, scheduler, external tools — communicates through it. Nothing talks
to etcd directly except the API server.

#### Why Everything Goes Through the API Server

```
                        ┌──────────────────────┐
                        │                      │
         kubectl ──────▶│                      │◀────── kubelet
                        │                      │
      Prometheus ──────▶│   kube-apiserver     │◀────── scheduler
                        │                      │
    CI/CD tools ──────▶│                      │◀────── controller-mgr
                        │                      │
  Custom controllers ──▶│                      │◀────── cloud-controller
                        │                      │
                        └──────────┬───────────┘
                                   │
                                   ▼
                            ┌──────────────┐
                            │    etcd      │
                            └──────────────┘
```

This single-hub design provides:
1. **Unified authentication/authorization** — one place to enforce RBAC.
2. **Admission control** — one place to validate and mutate requests.
3. **Audit logging** — one place to log all cluster operations.
4. **API versioning** — one place to handle version conversion.
5. **Watch multiplexing** — etcd handles fewer connections.

#### Request Lifecycle

Every request to the API server passes through this pipeline:

```
  Client Request (e.g., kubectl apply -f pod.yaml)
        │
        ▼
  ┌─────────────────────────────────────────────────────────┐
  │  1. AUTHENTICATION                                       │
  │     - Who are you?                                       │
  │     - x509 certs, bearer tokens, OIDC, webhook           │
  │     - Result: "This is user admin, groups [system:masters]" │
  └──────────────────────┬──────────────────────────────────┘
                         ▼
  ┌─────────────────────────────────────────────────────────┐
  │  2. AUTHORIZATION                                        │
  │     - Are you allowed to do this?                        │
  │     - RBAC (Role-Based Access Control)                    │
  │     - "Can user admin CREATE pods in namespace default?" │
  │     - Result: Allow / Deny                                │
  └──────────────────────┬──────────────────────────────────┘
                         ▼
  ┌─────────────────────────────────────────────────────────┐
  │  3. MUTATING ADMISSION                                   │
  │     - Modify the request before validation                │
  │     - Inject sidecars (Istio), add labels, set defaults   │
  │     - MutatingAdmissionWebhook calls external services    │
  └──────────────────────┬──────────────────────────────────┘
                         ▼
  ┌─────────────────────────────────────────────────────────┐
  │  4. OBJECT SCHEMA VALIDATION                             │
  │     - Is this a valid Kubernetes object?                  │
  │     - Does it conform to the OpenAPI schema?              │
  │     - Are required fields present?                        │
  └──────────────────────┬──────────────────────────────────┘
                         ▼
  ┌─────────────────────────────────────────────────────────┐
  │  5. VALIDATING ADMISSION                                 │
  │     - Final validation by webhooks                        │
  │     - Policy engines (OPA/Gatekeeper, Kyverno)            │
  │     - Cannot modify the object, only allow/deny           │
  └──────────────────────┬──────────────────────────────────┘
                         ▼
  ┌─────────────────────────────────────────────────────────┐
  │  6. PERSIST TO ETCD                                      │
  │     - Serialize to protobuf                               │
  │     - Write to etcd with resourceVersion                  │
  │     - etcd notifies watchers                              │
  └──────────────────────┬──────────────────────────────────┘
                         ▼
  ┌─────────────────────────────────────────────────────────┐
  │  7. RESPONSE                                             │
  │     - Return the created/updated object to the client     │
  │     - Include the new resourceVersion                     │
  └─────────────────────────────────────────────────────────┘
```

#### API Groups, Versions, and Resources

The Kubernetes API is organized into groups:

```
/api/v1/                          # Core (legacy) group
    /api/v1/pods
    /api/v1/services
    /api/v1/configmaps
    /api/v1/namespaces

/apis/apps/v1/                    # Apps group
    /apis/apps/v1/deployments
    /apis/apps/v1/replicasets
    /apis/apps/v1/statefulsets
    /apis/apps/v1/daemonsets

/apis/batch/v1/                   # Batch group
    /apis/batch/v1/jobs
    /apis/batch/v1/cronjobs

/apis/networking.k8s.io/v1/       # Networking group
    /apis/networking.k8s.io/v1/ingresses
    /apis/networking.k8s.io/v1/networkpolicies

/apis/rbac.authorization.k8s.io/v1/  # RBAC group
    /apis/rbac.authorization.k8s.io/v1/roles
    /apis/rbac.authorization.k8s.io/v1/clusterroles
```

Full URL pattern for a namespaced resource:
```
/apis/<group>/<version>/namespaces/<namespace>/<resource>/<name>

Example:
/apis/apps/v1/namespaces/default/deployments/nginx
```

#### Watch/List — Long-Polling for Changes

The watch mechanism is how controllers stay informed without constant polling:

```bash
# List all pods (one-time query)
GET /api/v1/namespaces/default/pods

# Watch for changes starting from a specific resourceVersion
GET /api/v1/namespaces/default/pods?watch=true&resourceVersion=12345
```

A watch response is a stream of newline-delimited JSON events:

```json
{"type":"ADDED","object":{"kind":"Pod","metadata":{"name":"nginx-abc","resourceVersion":"12346"}}}
{"type":"MODIFIED","object":{"kind":"Pod","metadata":{"name":"nginx-abc","resourceVersion":"12347"}}}
{"type":"DELETED","object":{"kind":"Pod","metadata":{"name":"nginx-abc","resourceVersion":"12348"}}}
```

#### Aggregation Layer — Extending the API

The API server can be extended without modifying its code by registering
**APIService** objects that proxy requests to external API servers:

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
spec:
  service:
    name: metrics-server
    namespace: kube-system
  group: metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 100
  versionPriority: 100
```

This is how `kubectl top pods` works — the metrics API is served by the metrics-server,
not the core API server.

#### Hands-On: Talking to the API Server with curl

```bash
# Get API server address and credentials from kubeconfig
APISERVER=$(kubectl config view --minify -o jsonpath='{.clusters[0].cluster.server}')
TOKEN=$(kubectl create token default)

# List all pods in the default namespace
curl -s -k -H "Authorization: Bearer $TOKEN" \
  "$APISERVER/api/v1/namespaces/default/pods" | jq '.items[].metadata.name'

# Get API server version
curl -s -k -H "Authorization: Bearer $TOKEN" \
  "$APISERVER/version" | jq .

# Discover all available API groups
curl -s -k -H "Authorization: Bearer $TOKEN" \
  "$APISERVER/apis" | jq '.groups[].name'

# Watch for pod changes (streaming response)
curl -s -k -H "Authorization: Bearer $TOKEN" \
  "$APISERVER/api/v1/namespaces/default/pods?watch=true"

# Create a pod directly via the API
curl -s -k -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  "$APISERVER/api/v1/namespaces/default/pods" \
  -d '{
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {"name": "curl-test"},
    "spec": {
      "containers": [{
        "name": "nginx",
        "image": "nginx:1.25"
      }]
    }
  }'
```

#### Hands-On: Using client-go to Interact with the API Server

```go
package main

import (
	"context"
	"fmt"
	"os"
	"path/filepath"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
)

// listPodsWithClientGo demonstrates how to connect to the Kubernetes API server
// using client-go and list all pods in a given namespace.
//
// client-go handles all the complexity of:
//   - Loading kubeconfig and authenticating
//   - Serialization/deserialization of API objects
//   - HTTP connection management and retries
//   - TLS certificate verification
func main() {
	// Step 1: Build the rest.Config from the default kubeconfig location.
	// This reads ~/.kube/config (or the path in KUBECONFIG env var)
	// and extracts the server URL, TLS certs, and auth credentials.
	home, _ := os.UserHomeDir()
	kubeconfig := filepath.Join(home, ".kube", "config")

	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error building kubeconfig: %v\n", err)
		os.Exit(1)
	}

	// Step 2: Create a kubernetes.Clientset from the config.
	// The Clientset provides typed client interfaces for every built-in
	// API group (CoreV1, AppsV1, BatchV1, etc.), each with methods like
	// Get(), List(), Create(), Update(), Delete(), Watch().
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error creating clientset: %v\n", err)
		os.Exit(1)
	}

	// Step 3: Use the CoreV1 typed client to list pods in the "default" namespace.
	// This translates to: GET /api/v1/namespaces/default/pods
	// The ListOptions allow filtering by labels, fields, resource version, etc.
	pods, err := clientset.CoreV1().Pods("default").List(
		context.Background(),
		metav1.ListOptions{},
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error listing pods: %v\n", err)
		os.Exit(1)
	}

	// Step 4: Print each pod's name, status, and IP address.
	fmt.Printf("Found %d pods in namespace 'default':\n\n", len(pods.Items))
	for _, pod := range pods.Items {
		fmt.Printf("  %-40s  Phase=%-12s  IP=%s\n",
			pod.Name,
			pod.Status.Phase,
			pod.Status.PodIP,
		)
	}
}
```

---

### 10.3.3 kube-controller-manager — The Reconciliation Engine

The kube-controller-manager is a single binary that runs **dozens** of independent
controllers, each implementing the Observe → Diff → Act loop for a specific resource type.

#### Why One Binary for All Controllers?

Running all controllers in a single process simplifies deployment and reduces resource
overhead. Each controller runs as a separate goroutine within the process, sharing the
same API server connection and informer caches.

#### Key Controllers

```
┌─────────────────────────────────────────────────────────────────┐
│                  kube-controller-manager                         │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │  Deployment      │  │  ReplicaSet      │  │  StatefulSet  │  │
│  │  Controller      │  │  Controller      │  │  Controller   │  │
│  └──────────────────┘  └──────────────────┘  └──────────────┘  │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │  DaemonSet       │  │  Job             │  │  CronJob      │  │
│  │  Controller      │  │  Controller      │  │  Controller   │  │
│  └──────────────────┘  └──────────────────┘  └──────────────┘  │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │  Node            │  │  Namespace       │  │  ServiceAcct  │  │
│  │  Controller      │  │  Controller      │  │  Controller   │  │
│  └──────────────────┘  └──────────────────┘  └──────────────┘  │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │  EndpointSlice   │  │  GarbageCollect  │  │  PV/PVC      │  │
│  │  Controller      │  │  Controller      │  │  Controller   │  │
│  └──────────────────┘  └──────────────────┘  └──────────────┘  │
│                                                                  │
│  ... and 20+ more controllers                                   │
└─────────────────────────────────────────────────────────────────┘
```

Here's what the major controllers do:

| Controller | Watches | Creates/Manages | Reconciliation Logic |
|---|---|---|---|
| **Deployment** | Deployments | ReplicaSets | Manages rolling updates, rollbacks by scaling RS up/down |
| **ReplicaSet** | ReplicaSets, Pods | Pods | Ensures `spec.replicas` pods exist with correct template |
| **StatefulSet** | StatefulSets, Pods | Pods, PVCs | Ordered create/delete with stable network identity |
| **DaemonSet** | DaemonSets, Nodes | Pods | Ensures one pod per node (or per matching node) |
| **Job** | Jobs, Pods | Pods | Runs pods to completion, tracks success/failure count |
| **CronJob** | CronJobs | Jobs | Creates Jobs on a cron schedule |
| **Node** | Nodes | — | Monitors heartbeats, marks unhealthy nodes NotReady |
| **Namespace** | Namespaces | — | Cleans up resources when namespace is deleted |
| **ServiceAccount** | Namespaces | ServiceAccounts | Creates default SA in each new namespace |
| **EndpointSlice** | Services, Pods | EndpointSlices | Tracks which pods back each Service |
| **GarbageCollector** | All resources | — | Deletes objects whose owner has been deleted |
| **PV/PVC** | PVs, PVCs | — | Binds PersistentVolumeClaims to PersistentVolumes |

#### Leader Election

In a high-availability setup with multiple control plane nodes, only **one** instance
of the controller-manager should be actively reconciling at any time. Kubernetes uses
**lease-based leader election** to ensure this:

```
┌────────────────────┐   ┌────────────────────┐   ┌────────────────────┐
│  controller-mgr-1  │   │  controller-mgr-2  │   │  controller-mgr-3  │
│                    │   │                    │   │                    │
│  Status: LEADER ★  │   │  Status: STANDBY   │   │  Status: STANDBY   │
│  (actively         │   │  (watching leader   │   │  (watching leader   │
│   reconciling)     │   │   lease, ready to   │   │   lease, ready to   │
│                    │   │   take over)        │   │   take over)        │
└────────────────────┘   └────────────────────┘   └────────────────────┘
         │
         ▼
   Lease object in kube-system namespace
   (renewed every ~2 seconds)
```

The leader acquires a **Lease** object in the `kube-system` namespace. If the leader
fails to renew the lease within the timeout period, another instance acquires it:

```bash
# View the leader election lease
kubectl -n kube-system get lease kube-controller-manager -o yaml
```

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: kube-controller-manager
  namespace: kube-system
spec:
  holderIdentity: control-plane-node-1_<uuid>
  leaseDurationSeconds: 15
  renewTime: "2024-01-15T10:30:45.123456Z"
  acquireTime: "2024-01-15T08:00:00.000000Z"
```

#### Hands-On: Examining Controller Source Code Flow

The Deployment controller's reconciliation loop (simplified from the actual source):

```go
// syncDeployment is the core reconciliation function for the Deployment controller.
// It is called whenever a Deployment or one of its ReplicaSets changes.
//
// The function examines the current state (existing ReplicaSets and their pods)
// and takes action to progress toward the desired state (the Deployment's spec).
//
// Key design: This function is LEVEL-TRIGGERED — it doesn't care what event
// triggered it, only what the current state vs desired state looks like.
func (dc *DeploymentController) syncDeployment(ctx context.Context, key string) error {
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return err
	}

	// OBSERVE: Read the Deployment from the informer cache (not the API server).
	// The informer cache is kept up-to-date by the watch mechanism.
	deployment, err := dc.dLister.Deployments(namespace).Get(name)
	if errors.IsNotFound(err) {
		// The Deployment was deleted. Nothing to reconcile.
		return nil
	}
	if err != nil {
		return err
	}

	// Find all ReplicaSets owned by this Deployment.
	// OwnerReferences link child objects to their parent.
	rsList, err := dc.getReplicaSetsForDeployment(ctx, deployment)
	if err != nil {
		return err
	}

	// DIFF + ACT: Choose the right strategy based on the Deployment's spec.
	switch deployment.Spec.Strategy.Type {
	case apps.RollingUpdateDeploymentStrategyType:
		// Rolling update: gradually shift traffic from old RS to new RS.
		// Respects maxSurge (how many extra pods allowed) and
		// maxUnavailable (how many pods can be down during update).
		return dc.rolloutRolling(ctx, deployment, rsList)

	case apps.RecreateDeploymentStrategyType:
		// Recreate: kill all old pods first, then create new ones.
		// Causes downtime but is simpler and avoids running two versions.
		return dc.rolloutRecreate(ctx, deployment, rsList)
	}

	return fmt.Errorf("unexpected deployment strategy type: %s",
		deployment.Spec.Strategy.Type)
}
```

---

### 10.3.4 kube-scheduler — Assigning Pods to Nodes

The scheduler watches for newly created pods that have no `.spec.nodeName` set
(unscheduled pods) and selects the best node for each one.

#### Scheduling Cycle: Filtering → Scoring → Binding

```
  Unscheduled Pod arrives
        │
        ▼
  ┌──────────────────────────────────────────────────────┐
  │  FILTERING PHASE                                      │
  │  "Which nodes CAN run this pod?"                      │
  │                                                        │
  │  ┌─ NodeResourcesFit ─ Does the node have enough     │
  │  │                      CPU/memory?                   │
  │  ├─ NodeAffinity ────── Does the pod's nodeAffinity   │
  │  │                      match this node?              │
  │  ├─ PodTopologySpread ─ Would this violate topology   │
  │  │                      spread constraints?           │
  │  ├─ TaintToleration ── Does the pod tolerate the      │
  │  │                      node's taints?                │
  │  ├─ NodePorts ──────── Are the requested hostPorts    │
  │  │                      available?                    │
  │  └─ VolumeZone ─────── Is the pod's PV in this       │
  │                         node's zone?                  │
  │                                                        │
  │  Result: [node-1, node-3, node-5] (feasible nodes)    │
  └──────────────────────┬───────────────────────────────┘
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │  SCORING PHASE                                        │
  │  "Which feasible node is BEST?"                       │
  │                                                        │
  │  Each plugin scores each node from 0-100:              │
  │                                                        │
  │  ┌─ LeastAllocated ── Prefer nodes with more free    │
  │  │                     resources (spread workload)     │
  │  ├─ MostAllocated ─── Prefer nodes already busy       │
  │  │                     (bin-packing for efficiency)    │
  │  ├─ InterPodAffinity ─ Prefer nodes running pods     │
  │  │                      the pod has affinity for      │
  │  ├─ ImageLocality ──── Prefer nodes that already     │
  │  │                      have the container image      │
  │  └─ NodeAffinity ───── Score by preferred affinity   │
  │                                                        │
  │  Scores:  node-1=72   node-3=85   node-5=61           │
  │  Winner:  node-3  ★                                    │
  └──────────────────────┬───────────────────────────────┘
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │  BINDING PHASE                                        │
  │  "Assign the pod to the winning node."                │
  │                                                        │
  │  POST /api/v1/namespaces/default/pods/nginx/binding   │
  │  Body: { "target": { "name": "node-3" } }             │
  │                                                        │
  │  This sets pod.spec.nodeName = "node-3"                │
  │  The kubelet on node-3 sees this and starts the pod.   │
  └──────────────────────────────────────────────────────┘
```

#### Scheduling Framework — Extension Points

The scheduler is built as a **framework** with pluggable extension points:

```
┌─────────────────── Scheduling Framework ───────────────────────┐
│                                                                 │
│  PreFilter → Filter → PostFilter → PreScore → Score →          │
│  NormalizeScore → Reserve → Permit → PreBind → Bind → PostBind │
│                                                                 │
│  Each extension point can have multiple plugins.                │
│  Plugins are composable — enable/disable/reorder as needed.     │
└─────────────────────────────────────────────────────────────────┘
```

> **📖 Reference:** Chapter 13 covers the scheduler in full depth — writing custom
> scheduler plugins, multi-scheduler setups, and scheduling performance optimization.

---

### 10.3.5 cloud-controller-manager — Cloud Integration

The cloud-controller-manager (CCM) runs controllers that interact with the underlying
cloud provider (AWS, GCP, Azure, etc.):

| Controller | Responsibility |
|---|---|
| **Node** | Detects when a cloud VM is deleted and removes the Kubernetes Node object |
| **Route** | Configures cloud network routes so pods on different nodes can communicate |
| **Service** | Provisions cloud load balancers for `type: LoadBalancer` Services |

#### Why It Was Split from kube-controller-manager

Originally, cloud-specific code lived inside the main controller manager. This meant:
- Every cloud provider had to upstream their code into the main Kubernetes repo.
- The controller manager binary contained code for ALL cloud providers.
- Cloud providers couldn't iterate independently.

The CCM split allows cloud providers to maintain their own out-of-tree controller
managers that implement a well-defined interface.

---

## 10.4 Data Plane Components — Deep Dive

### 10.4.1 kubelet — The Node Agent

The kubelet is the most complex component on the data plane. It runs on every worker
node and is responsible for managing the complete lifecycle of pods on that node.

```
┌──────────────────────────── Worker Node ────────────────────────────────┐
│                                                                          │
│  ┌──────────────────────────── kubelet ─────────────────────────────┐   │
│  │                                                                    │   │
│  │  ┌─────────────────┐   ┌──────────────────┐   ┌──────────────┐  │   │
│  │  │  Pod Lifecycle   │   │  Volume Manager   │   │  Status       │  │   │
│  │  │  Manager         │   │  (CSI calls)      │   │  Manager      │  │   │
│  │  └────────┬─────────┘   └──────────────────┘   └──────────────┘  │   │
│  │           │                                                        │   │
│  │           │ CRI (gRPC)                                             │   │
│  │           ▼                                                        │   │
│  │  ┌─────────────────────────────────────────────────────────────┐  │   │
│  │  │  Container Runtime Interface (CRI)                          │  │   │
│  │  │                                                              │  │   │
│  │  │  RunPodSandbox()    CreateContainer()    StartContainer()    │  │   │
│  │  │  StopContainer()    RemoveContainer()    PodSandboxStatus()  │  │   │
│  │  └──────────────────────────┬──────────────────────────────────┘  │   │
│  │                             │                                      │   │
│  │                             ▼                                      │   │
│  │  ┌──────────────────────────────────────────────────────────────┐ │   │
│  │  │  containerd / CRI-O                                          │ │   │
│  │  │                                                              │ │   │
│  │  │  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌──────────────┐  │ │   │
│  │  │  │ runc    │  │ runc    │  │ runc    │  │ Device Plugins│  │ │   │
│  │  │  │ (nginx) │  │ (redis) │  │ (app)   │  │ (GPU, etc.)  │  │ │   │
│  │  │  └─────────┘  └─────────┘  └─────────┘  └──────────────┘  │ │   │
│  │  └──────────────────────────────────────────────────────────────┘ │   │
│  └────────────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

#### Pod Lifecycle Management

The kubelet watches the API server for pods assigned to its node (`spec.nodeName`
matches). For each pod, it:

1. **Pulls images** (via CRI `PullImage`)
2. **Creates the pod sandbox** (a pause container that holds the network namespace)
3. **Creates and starts containers** (via CRI `CreateContainer` + `StartContainer`)
4. **Runs health checks** (liveness, readiness, startup probes)
5. **Reports status** back to the API server
6. **Handles graceful shutdown** when a pod is deleted

#### Node Status Reporting

The kubelet periodically reports its node's status to the API server:

```yaml
# Node status includes:
status:
  conditions:
  - type: Ready
    status: "True"
    lastHeartbeatTime: "2024-01-15T10:30:00Z"
  - type: MemoryPressure
    status: "False"
  - type: DiskPressure
    status: "False"
  - type: PIDPressure
    status: "False"
  capacity:
    cpu: "8"
    memory: "32Gi"
    pods: "110"
  allocatable:
    cpu: "7800m"          # 8 CPUs minus system reserved
    memory: "31Gi"
    pods: "110"
  nodeInfo:
    containerRuntimeVersion: "containerd://1.7.2"
    kubeletVersion: "v1.29.0"
    osImage: "Ubuntu 22.04 LTS"
    architecture: "amd64"
```

> **📖 Reference:** Chapter 14 covers the kubelet in full depth — PLEG, cAdvisor,
> eviction manager, and the full pod lifecycle state machine.

---

### 10.4.2 kube-proxy — Service Networking

kube-proxy runs on every node and implements the **Service** abstraction — translating
a stable virtual IP (ClusterIP) into actual pod IPs.

#### iptables Mode (Default)

In iptables mode, kube-proxy programs kernel iptables rules to intercept traffic
destined for Service ClusterIPs and redirect (DNAT) it to a healthy backend pod:

```
  Client Pod (10.244.1.5)
      │
      │ dst: 10.96.0.100:80 (Service ClusterIP)
      ▼
  ┌─────────────────────────────────────────────────┐
  │  iptables (PREROUTING / OUTPUT chain)            │
  │                                                   │
  │  -A KUBE-SERVICES -d 10.96.0.100/32 -p tcp       │
  │    --dport 80 -j KUBE-SVC-XXXX                    │
  │                                                   │
  │  # Random load balancing across endpoints:         │
  │  -A KUBE-SVC-XXXX -m statistic --mode random      │
  │    --probability 0.333 -j KUBE-SEP-AAA             │
  │  -A KUBE-SVC-XXXX -m statistic --mode random      │
  │    --probability 0.500 -j KUBE-SEP-BBB             │
  │  -A KUBE-SVC-XXXX -j KUBE-SEP-CCC                 │
  │                                                   │
  │  # Each endpoint does DNAT to pod IP:              │
  │  -A KUBE-SEP-AAA -j DNAT --to-dest 10.244.1.10:80 │
  │  -A KUBE-SEP-BBB -j DNAT --to-dest 10.244.2.20:80 │
  │  -A KUBE-SEP-CCC -j DNAT --to-dest 10.244.3.30:80 │
  └─────────────────────────────────────────────────┘
      │
      │ dst: 10.244.2.20:80 (Pod IP — after DNAT)
      ▼
  Backend Pod (10.244.2.20)
```

#### IPVS Mode

IPVS (IP Virtual Server) is a kernel-level L4 load balancer that scales much better
than iptables for clusters with thousands of Services:

```
  iptables mode:  O(n) rule matching — linear scan through rules
  IPVS mode:      O(1) hash-based lookup — constant time regardless of services
```

IPVS supports multiple load balancing algorithms:
- **Round Robin** (rr) — default
- **Least Connections** (lc)
- **Source Hashing** (sh) — session affinity
- **Weighted Round Robin** (wrr)

#### eBPF Mode (Cilium)

Cilium can completely replace kube-proxy using eBPF programs attached directly to the
network stack, bypassing iptables/IPVS entirely:

```
Traditional:  Pod → iptables → conntrack → routing → Pod
Cilium eBPF:  Pod → eBPF program → Pod  (much shorter path)
```

Benefits: Lower latency, better observability, no conntrack table exhaustion.

#### Hands-On: Examining iptables Rules Created by kube-proxy

```bash
# List all KUBE-SERVICES rules (one per Service)
sudo iptables -t nat -L KUBE-SERVICES -n --line-numbers

# Find rules for a specific service (e.g., the kubernetes API service)
sudo iptables -t nat -L KUBE-SERVICES -n | grep "10.96.0.1"

# Trace the full chain for a service
# First, find the KUBE-SVC chain name:
sudo iptables -t nat -L KUBE-SERVICES -n | grep "10.96.0.100"
# Output: KUBE-SVC-XXXX tcp -- 0.0.0.0/0 10.96.0.100 tcp dpt:80

# Then list endpoints in that chain:
sudo iptables -t nat -L KUBE-SVC-XXXX -n
# Output: Shows KUBE-SEP chains with probability-based load balancing

# Show the DNAT targets (actual pod IPs):
sudo iptables -t nat -L KUBE-SEP-AAA -n
# Output: DNAT to 10.244.1.10:80

# Count total iptables rules (can be thousands in large clusters)
sudo iptables -t nat -L -n | wc -l
```

#### Hands-On: Examining IPVS Rules

```bash
# Enable IPVS mode in kube-proxy config
# (in kube-proxy ConfigMap, set mode: "ipvs")

# List all virtual servers (one per Service ClusterIP)
sudo ipvsadm -Ln

# Output looks like:
# TCP  10.96.0.1:443 rr
#   -> 192.168.1.10:6443    Masq    1      0      0
#   -> 192.168.1.11:6443    Masq    1      0      0
# TCP  10.96.0.100:80 rr
#   -> 10.244.1.10:80       Masq    1      0      0
#   -> 10.244.2.20:80       Masq    1      0      0
#   -> 10.244.3.30:80       Masq    1      0      0

# Show connection statistics
sudo ipvsadm -Ln --stats

# Show the rate of connections
sudo ipvsadm -Ln --rate
```

---

### 10.4.3 Container Runtime

The container runtime is the component that actually runs containers. Kubernetes uses
the **Container Runtime Interface (CRI)** — a gRPC API — to communicate with it.

```
┌──────────┐    gRPC (CRI)    ┌─────────────────┐     ┌───────┐
│  kubelet  │ ───────────────▶ │  containerd      │ ──▶ │ runc  │
│           │                  │  or CRI-O        │     │       │
└──────────┘                  └─────────────────┘     └───────┘
```

Key CRI operations:
- `RunPodSandbox` — Create the pod's network namespace (the "pause" container)
- `CreateContainer` — Prepare a container (pull image, set up rootfs)
- `StartContainer` — Actually start the process
- `StopContainer` — Send SIGTERM, wait, then SIGKILL
- `RemoveContainer` — Clean up container resources

> **📖 Reference:** Chapter 9 covers container runtimes in detail — containerd
> architecture, CRI-O, image management, and the OCI runtime spec.

---

## 10.5 Kubernetes Objects & API Machinery

### 10.5.1 Resource Types, API Groups, and Versions

Every Kubernetes object has a **GVR** (Group, Version, Resource) that uniquely identifies
its type in the API:

```
Group:    apps                    (or "" for core resources)
Version:  v1                      (the API version)
Resource: deployments             (plural, lowercase name)

Full API path: /apis/apps/v1/namespaces/default/deployments
```

And a **GVK** (Group, Version, Kind) that identifies the Go type:

```
Group:    apps
Version:  v1
Kind:     Deployment              (singular, CamelCase)
```

GVR is used for REST API paths; GVK is used in YAML manifests (`apiVersion` + `kind`).

```yaml
# GVK in a manifest:
apiVersion: apps/v1        # Group/Version  (core group omits the group: "v1")
kind: Deployment            # Kind

# This maps to GVR:
# Group=apps, Version=v1, Resource=deployments
# API path: /apis/apps/v1/namespaces/<ns>/deployments/<name>
```

### 10.5.2 Object Metadata

Every Kubernetes object has standard metadata:

```yaml
metadata:
  # === Identity ===
  name: nginx-deployment          # Unique within namespace
  namespace: default              # Namespace scope (cluster-scoped objects omit this)
  uid: "a1b2c3d4-e5f6-..."       # Globally unique ID, set by API server

  # === Versioning ===
  resourceVersion: "12345"        # etcd revision — used for optimistic concurrency
  generation: 3                   # Incremented on spec changes (not status changes)

  # === Labeling ===
  labels:                         # Key-value pairs for selection and grouping
    app: nginx
    environment: production
    version: "1.25"

  # === Annotations ===
  annotations:                    # Key-value pairs for non-identifying metadata
    deployment.kubernetes.io/revision: "3"
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment",...}

  # === Ownership ===
  ownerReferences:                # Links to parent objects (for garbage collection)
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: nginx-deploy-7d9b8c6f5
    uid: "x1y2z3..."
    controller: true
    blockOwnerDeletion: true

  # === Deletion Control ===
  finalizers:                     # Prevent deletion until controllers clean up
  - kubernetes.io/pv-protection
  deletionTimestamp: null         # Set when deletion is requested (with finalizers)
```

### 10.5.3 OwnerReferences — The Garbage Collection Tree

Kubernetes objects form a tree of ownership:

```
Deployment (nginx-deploy)
    │
    ├── ReplicaSet (nginx-deploy-7d9b8c6f5)    ownerRef → Deployment
    │       │
    │       ├── Pod (nginx-deploy-7d9b8c6f5-abc12)   ownerRef → ReplicaSet
    │       ├── Pod (nginx-deploy-7d9b8c6f5-def34)   ownerRef → ReplicaSet
    │       └── Pod (nginx-deploy-7d9b8c6f5-ghi56)   ownerRef → ReplicaSet
    │
    └── ReplicaSet (nginx-deploy-6c8b7a5e4)    ownerRef → Deployment (old revision)
            │
            └── (scaled to 0 — no pods)
```

When you delete a Deployment, the **garbage collector controller** cascades the
deletion: it finds all objects with an `ownerReference` pointing to the deleted
Deployment, deletes them, then finds their children, and so on.

Two deletion propagation policies:
- **Foreground**: Children deleted first, then parent.
- **Background** (default): Parent deleted immediately, children cleaned up async.
- **Orphan**: Children left alive (ownerReference removed).

### 10.5.4 Finalizers — Preventing Premature Deletion

Finalizers are a list of strings on an object that prevent its deletion until specific
cleanup is performed. When you delete an object with finalizers:

1. Kubernetes sets `deletionTimestamp` but does NOT remove the object.
2. Controllers see the `deletionTimestamp` and perform cleanup.
3. Each controller removes its finalizer from the list.
4. When the finalizers list is empty, Kubernetes actually deletes the object.

```
DELETE /api/v1/namespaces/default/persistentvolumeclaims/my-data
    │
    ▼
┌─────────────────────────────────────────────┐
│  PVC object (still exists!)                  │
│  metadata:                                   │
│    deletionTimestamp: "2024-01-15T10:30:00Z" │
│    finalizers:                                │
│    - kubernetes.io/pv-protection             │  ← PV controller must remove this
└─────────────────────────────────────────────┘
    │
    │  PV controller detects deletionTimestamp,
    │  checks no pods are using the PVC,
    │  removes the finalizer.
    ▼
┌─────────────────────────────────────────────┐
│  PVC object                                  │
│  metadata:                                   │
│    deletionTimestamp: "2024-01-15T10:30:00Z" │
│    finalizers: []                             │  ← Empty! Now it can be deleted.
└─────────────────────────────────────────────┘
    │
    ▼
  Object deleted from etcd.
```

### 10.5.5 Hands-On: Exploring the API with kubectl

```bash
# List ALL resource types available in the cluster
kubectl api-resources

# List only namespaced resources
kubectl api-resources --namespaced=true

# List resources in the "apps" API group
kubectl api-resources --api-group=apps

# Show the full API path for deployments
kubectl api-resources -o wide | grep deployments

# Discover API versions
kubectl api-versions

# Get the OpenAPI schema for a specific resource
kubectl explain deployment.spec.strategy --recursive

# View the raw API response (shows resourceVersion, uid, etc.)
kubectl get pod nginx -o json | jq '.metadata'

# View ownerReferences of a pod (shows parent ReplicaSet)
kubectl get pod nginx-deploy-7d9b8c6f5-abc12 -o json | \
  jq '.metadata.ownerReferences'

# View the resource tree (requires krew plugin 'tree')
kubectl tree deployment nginx-deploy
```

### 10.5.6 Hands-On: Using Unstructured Objects in Go with client-go

```go
package main

import (
	"context"
	"fmt"
	"os"
	"path/filepath"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/tools/clientcmd"
)

// unstructuredDemo demonstrates working with Kubernetes objects without
// compile-time type definitions. This is essential when working with:
//   - Custom Resource Definitions (CRDs) — no Go types exist in client-go
//   - Dynamic controllers that handle arbitrary resource types
//   - Tools that need to work with any Kubernetes object generically
//
// The unstructured.Unstructured type wraps a map[string]interface{}, allowing
// you to read/write any field using nested map access or helper methods.
func main() {
	home, _ := os.UserHomeDir()
	kubeconfig := filepath.Join(home, ".kube", "config")

	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}

	// Create a dynamic client — unlike the typed Clientset, this client
	// can work with ANY resource type using GVR (Group, Version, Resource).
	dynClient, err := dynamic.NewForConfig(config)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error creating dynamic client: %v\n", err)
		os.Exit(1)
	}

	// Define the GVR for the resource we want to work with.
	// This could be a built-in resource or a CRD — the dynamic client
	// doesn't need compile-time Go types.
	deploymentGVR := schema.GroupVersionResource{
		Group:    "apps",
		Version:  "v1",
		Resource: "deployments",
	}

	// List all deployments in the "default" namespace.
	// This returns a list of unstructured.Unstructured objects.
	deployments, err := dynClient.Resource(deploymentGVR).Namespace("default").List(
		context.Background(),
		metav1.ListOptions{},
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error listing deployments: %v\n", err)
		os.Exit(1)
	}

	fmt.Printf("Deployments in 'default' namespace:\n\n")
	for _, d := range deployments.Items {
		// Access fields using the unstructured helper methods.
		// These navigate the nested map[string]interface{} structure.
		name := d.GetName()
		namespace := d.GetNamespace()

		// Access nested fields using NestedInt64 / NestedString / etc.
		// The path is a slice of map keys to traverse.
		replicas, found, err := unstructured.NestedInt64(
			d.Object, "spec", "replicas",
		)
		if err != nil || !found {
			replicas = 0
		}

		// Access the container image from the pod template.
		// Path: spec.template.spec.containers[0].image
		containers, found, err := unstructured.NestedSlice(
			d.Object, "spec", "template", "spec", "containers",
		)
		image := "unknown"
		if err == nil && found && len(containers) > 0 {
			if c, ok := containers[0].(map[string]interface{}); ok {
				if img, ok := c["image"].(string); ok {
					image = img
				}
			}
		}

		fmt.Printf("  %-30s  ns=%-12s  replicas=%d  image=%s\n",
			name, namespace, replicas, image,
		)
	}

	// Create an unstructured object from scratch.
	// This is useful when you don't have a Go struct for a CRD.
	newConfigMap := &unstructured.Unstructured{
		Object: map[string]interface{}{
			"apiVersion": "v1",
			"kind":       "ConfigMap",
			"metadata": map[string]interface{}{
				"name":      "dynamic-config",
				"namespace": "default",
				"labels": map[string]interface{}{
					"created-by": "dynamic-client",
				},
			},
			"data": map[string]interface{}{
				"key1": "value1",
				"key2": "value2",
			},
		},
	}

	// Create the ConfigMap using the dynamic client.
	configMapGVR := schema.GroupVersionResource{
		Group:    "",       // Core group (empty string)
		Version:  "v1",
		Resource: "configmaps",
	}

	created, err := dynClient.Resource(configMapGVR).Namespace("default").Create(
		context.Background(),
		newConfigMap,
		metav1.CreateOptions{},
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error creating ConfigMap: %v\n", err)
		os.Exit(1)
	}

	fmt.Printf("\nCreated ConfigMap: %s (resourceVersion=%s)\n",
		created.GetName(),
		created.GetResourceVersion(),
	)
}
```

---

## 10.6 How a Pod Gets Created — End-to-End Flow

Let's trace every single step from `kubectl apply` to a running container. This is the
most important flow to understand in Kubernetes.

```
  kubectl apply -f pod.yaml
        │
        │  1. kubectl reads pod.yaml, sends HTTP POST to API server
        │     POST /api/v1/namespaces/default/pods
        ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  kube-apiserver                                               │
  │                                                                │
  │  2. Authentication → "User: admin"                             │
  │  3. Authorization  → "admin can create pods in default? Yes"   │
  │  4. Mutating Admission → Add default service account,          │
  │                          inject tolerations, set defaults       │
  │  5. Validation → Schema valid? Required fields present?        │
  │  6. Validating Admission → Policy checks pass?                 │
  │  7. Write to etcd → /registry/pods/default/nginx               │
  │     Pod status: Pending, nodeName: ""                          │
  │  8. Return 201 Created to kubectl                              │
  └──────────────────────────┬───────────────────────────────────┘
                             │
                             │  9. Watch event fires:
                             │     "New pod with nodeName=''"
                             ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  kube-scheduler                                               │
  │                                                                │
  │  10. Filter nodes → Which nodes have enough resources?         │
  │  11. Score nodes → Which node is best?                         │
  │  12. Bind: POST /api/v1/namespaces/default/pods/nginx/binding │
  │      Body: {"target": {"name": "worker-2"}}                    │
  │  13. API server sets pod.spec.nodeName = "worker-2"            │
  │      Writes update to etcd.                                    │
  └──────────────────────────┬───────────────────────────────────┘
                             │
                             │  14. Watch event fires:
                             │      "Pod nginx now assigned to worker-2"
                             ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  kubelet (on worker-2)                                        │
  │                                                                │
  │  15. Sees pod assigned to its node via watch                   │
  │  16. Pulls container image (if not cached)                     │
  │      CRI: PullImage("nginx:1.25")                              │
  │  17. Creates pod sandbox (pause container)                     │
  │      CRI: RunPodSandbox(PodSandboxConfig)                     │
  │      → Creates network namespace, gets IP from CNI             │
  │  18. Creates container                                         │
  │      CRI: CreateContainer(sandboxID, ContainerConfig)          │
  │  19. Starts container                                          │
  │      CRI: StartContainer(containerID)                          │
  │  20. Reports pod status back to API server                     │
  │      PATCH /api/v1/namespaces/default/pods/nginx/status        │
  │      Status: Running, IP: 10.244.2.15                          │
  └──────────────────────────────────────────────────────────────┘
```

### Detailed Sequence Diagram

```
  kubectl         API Server        etcd         Scheduler         kubelet
    │                  │               │              │               │
    │ POST /pods       │               │              │               │
    │─────────────────▶│               │              │               │
    │                  │ PUT /pods/... │              │               │
    │                  │──────────────▶│              │               │
    │                  │     OK        │              │               │
    │                  │◀──────────────│              │               │
    │  201 Created     │               │              │               │
    │◀─────────────────│               │              │               │
    │                  │               │              │               │
    │                  │  Watch event: new pod        │               │
    │                  │  (nodeName="")               │               │
    │                  │─────────────────────────────▶│               │
    │                  │               │              │               │
    │                  │               │    Filter    │               │
    │                  │               │    Score     │               │
    │                  │               │    Select    │               │
    │                  │               │              │               │
    │                  │ POST /pods/nginx/binding     │               │
    │                  │◀─────────────────────────────│               │
    │                  │ PUT /pods/... │              │               │
    │                  │──────────────▶│              │               │
    │                  │     OK        │              │               │
    │                  │◀──────────────│              │               │
    │                  │               │              │               │
    │                  │  Watch event: pod assigned to worker-2       │
    │                  │────────────────────────────────────────────▶│
    │                  │               │              │               │
    │                  │               │              │  PullImage    │
    │                  │               │              │  RunSandbox   │
    │                  │               │              │  CreateCtnr   │
    │                  │               │              │  StartCtnr    │
    │                  │               │              │               │
    │                  │ PATCH /pods/nginx/status     │               │
    │                  │◀────────────────────────────────────────────│
    │                  │ PUT /pods/... │              │               │
    │                  │──────────────▶│              │               │
    │                  │     OK        │              │               │
    │                  │◀──────────────│              │               │
    │                  │               │              │               │
```

**Key insight:** No component directly calls another (except via the API server). The
scheduler doesn't tell the kubelet to start a pod. Instead, the scheduler writes to the
API server, and the kubelet independently watches for changes. This **decoupled**
design means any component can restart independently without losing state.

---

## 10.7 Kubernetes Networking Model

Kubernetes imposes three fundamental networking requirements:

```
┌─────────────────────────────────────────────────────────────────┐
│  THE KUBERNETES NETWORKING MODEL                                 │
│                                                                   │
│  1. Every Pod gets its own unique IP address.                     │
│                                                                   │
│  2. Pods on ANY node can communicate with pods on ANY other       │
│     node directly — without NAT.                                  │
│                                                                   │
│  3. Agents on a node (kubelet, kube-proxy) can communicate        │
│     with all pods on that node.                                   │
│                                                                   │
│  These rules create a FLAT NETWORK — every pod can reach every    │
│  other pod using its IP, regardless of which node it's on.        │
└─────────────────────────────────────────────────────────────────┘
```

This is implemented by **CNI plugins** (Calico, Cilium, Flannel, Weave, etc.) which
configure the actual network plumbing:

```
┌───── Node 1 ──────────────┐        ┌───── Node 2 ──────────────┐
│                            │        │                            │
│  Pod A (10.244.1.2)       │        │  Pod C (10.244.2.2)       │
│  Pod B (10.244.1.3)       │        │  Pod D (10.244.2.3)       │
│                            │        │                            │
│  cbr0 bridge (10.244.1.1) │        │  cbr0 bridge (10.244.2.1) │
│         │                  │        │         │                  │
│         │ veth pairs       │        │         │ veth pairs       │
│         │                  │        │         │                  │
└─────────┼──────────────────┘        └─────────┼──────────────────┘
          │                                     │
          │      VXLAN / BGP / IPinIP           │
          └─────────────────────────────────────┘
          
Pod A (10.244.1.2) → Pod C (10.244.2.2):  Direct, no NAT ✅
```

> **📖 Reference:** Chapters 22-26 cover Kubernetes networking in full depth — CNI
> plugins, Service mesh, Ingress, NetworkPolicy, DNS, and eBPF networking.

---

## 10.8 How a Deployment Rolling Update Works — End-to-End

Let's trace what happens when you update a Deployment's image from `nginx:1.24` to
`nginx:1.25`. This involves two controllers working in concert.

### Initial State

```yaml
Deployment: nginx-deploy (replicas: 3, image: nginx:1.24)
    └── ReplicaSet: nginx-deploy-abc (replicas: 3, image: nginx:1.24)
            ├── Pod: nginx-deploy-abc-111 (Running, nginx:1.24)
            ├── Pod: nginx-deploy-abc-222 (Running, nginx:1.24)
            └── Pod: nginx-deploy-abc-333 (Running, nginx:1.24)
```

### Update Command

```bash
kubectl set image deployment/nginx-deploy nginx=nginx:1.25
# Or: kubectl apply -f deployment.yaml (with updated image)
```

### Step-by-Step Sequence

```
Step 1: kubectl PATCHes the Deployment spec
  PATCH /apis/apps/v1/namespaces/default/deployments/nginx-deploy
  Body: {"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.25"}]}}}}

Step 2: Deployment controller detects the spec change.
  The pod template hash changed → a new ReplicaSet is needed.

Step 3: Deployment controller creates a NEW ReplicaSet (nginx-deploy-xyz)
  with the updated pod template (image: nginx:1.25), replicas: 0.

Step 4: Rolling update begins.
  Strategy: maxSurge=25%, maxUnavailable=25%
  With 3 replicas: maxSurge=1, maxUnavailable=1

  ┌──────────────────────────────────────────────────────────────┐
  │  Round 1:                                                     │
  │    - Scale UP new RS:   nginx-deploy-xyz replicas: 0 → 1     │
  │    - Scale DOWN old RS: nginx-deploy-abc replicas: 3 → 2     │
  │                                                                │
  │  State: 2 old pods + 1 new pod = 3 running (1 unavailable OK) │
  └──────────────────────────────────────────────────────────────┘
        │
        │  Wait for new pod to become Ready...
        ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  Round 2:                                                     │
  │    - Scale UP new RS:   nginx-deploy-xyz replicas: 1 → 2     │
  │    - Scale DOWN old RS: nginx-deploy-abc replicas: 2 → 1     │
  │                                                                │
  │  State: 1 old pod + 2 new pods                                │
  └──────────────────────────────────────────────────────────────┘
        │
        │  Wait for new pod to become Ready...
        ▼
  ┌──────────────────────────────────────────────────────────────┐
  │  Round 3:                                                     │
  │    - Scale UP new RS:   nginx-deploy-xyz replicas: 2 → 3     │
  │    - Scale DOWN old RS: nginx-deploy-abc replicas: 1 → 0     │
  │                                                                │
  │  State: 0 old pods + 3 new pods (COMPLETE ✅)                 │
  └──────────────────────────────────────────────────────────────┘
```

### Final State

```yaml
Deployment: nginx-deploy (replicas: 3, image: nginx:1.25)
    ├── ReplicaSet: nginx-deploy-xyz (replicas: 3, image: nginx:1.25)  ← ACTIVE
    │       ├── Pod: nginx-deploy-xyz-aaa (Running, nginx:1.25)
    │       ├── Pod: nginx-deploy-xyz-bbb (Running, nginx:1.25)
    │       └── Pod: nginx-deploy-xyz-ccc (Running, nginx:1.25)
    │
    └── ReplicaSet: nginx-deploy-abc (replicas: 0, image: nginx:1.24)  ← KEPT (for rollback)
```

Note: The old ReplicaSet is kept (with 0 replicas) to enable rollback:
```bash
kubectl rollout undo deployment/nginx-deploy
# This simply scales the old RS back up and the new RS back down.
```

### API Calls Made During Rolling Update

```
1. Deployment controller watches Deployments, sees spec change
2. GET  /apis/apps/v1/namespaces/default/replicasets?labelSelector=app=nginx
3. POST /apis/apps/v1/namespaces/default/replicasets  (create new RS)
4. PATCH /apis/apps/v1/namespaces/default/replicasets/nginx-deploy-xyz
   (scale to 1)
5. PATCH /apis/apps/v1/namespaces/default/replicasets/nginx-deploy-abc
   (scale to 2)
6. ReplicaSet controller watches ReplicaSets, sees replicas mismatch
7. POST /api/v1/namespaces/default/pods  (create new pod)
8. DELETE /api/v1/namespaces/default/pods/nginx-deploy-abc-333
9. ... (repeat until rollout complete)
10. PATCH /apis/apps/v1/namespaces/default/deployments/nginx-deploy/status
    (update rollout status)
```

---

## 10.9 High Availability Architecture

Production Kubernetes clusters run multiple copies of every control plane component
for fault tolerance.

```
┌─────────────────────────────── HA Control Plane ─────────────────────────────────┐
│                                                                                    │
│                        ┌──────────────────┐                                       │
│                        │  Load Balancer   │                                       │
│                        │  (HAProxy/NLB)   │                                       │
│                        └────────┬─────────┘                                       │
│                    ┌────────────┼────────────┐                                    │
│                    ▼            ▼            ▼                                    │
│  ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐                  │
│  │  Control Plane 1  │ │  Control Plane 2  │ │  Control Plane 3  │                  │
│  │                    │ │                    │ │                    │                  │
│  │  API Server ★     │ │  API Server ★     │ │  API Server ★     │                  │
│  │  (all active)     │ │  (all active)     │ │  (all active)     │                  │
│  │                    │ │                    │ │                    │                  │
│  │  Controller Mgr   │ │  Controller Mgr   │ │  Controller Mgr   │                  │
│  │  (LEADER) ★       │ │  (standby)        │ │  (standby)        │                  │
│  │                    │ │                    │ │                    │                  │
│  │  Scheduler        │ │  Scheduler        │ │  Scheduler        │                  │
│  │  (LEADER) ★       │ │  (standby)        │ │  (standby)        │                  │
│  │                    │ │                    │ │                    │                  │
│  │  etcd (member)    │ │  etcd (member)    │ │  etcd (member)    │                  │
│  │  ★ (leader)       │ │                    │ │                    │                  │
│  └──────────────────┘ └──────────────────┘ └──────────────────┘                  │
│                                                                                    │
│  API Server:         ALL instances active (stateless, load balanced)                │
│  Controller Manager: ONE active leader (lease-based election)                      │
│  Scheduler:          ONE active leader (lease-based election)                      │
│  etcd:               Raft consensus (one leader, all replicas serve reads)         │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### Component HA Strategies

| Component | HA Strategy | Why |
|---|---|---|
| **API Server** | Active-active, all instances serve traffic | Stateless — all state in etcd |
| **Controller Manager** | Active-passive with leader election | Avoid duplicate reconciliation |
| **Scheduler** | Active-passive with leader election | Avoid double-scheduling pods |
| **etcd** | Raft consensus (3 or 5 members) | Tolerates 1 or 2 member failures |

### Why 3 or 5 etcd Nodes?

Raft requires a **quorum** (majority) to commit writes:
- 3 nodes: quorum = 2, tolerates 1 failure
- 5 nodes: quorum = 3, tolerates 2 failures
- 7 nodes: quorum = 4, tolerates 3 failures (diminishing returns, more write latency)

Even numbers are never recommended — 4 nodes tolerates only 1 failure (same as 3) but
requires more resources.

### Hands-On: Setting Up HA Control Plane with kubeadm

```bash
# === Prerequisites ===
# 3 control plane nodes: cp1 (192.168.1.10), cp2 (192.168.1.11), cp3 (192.168.1.12)
# 1 load balancer: lb (192.168.1.100) — HAProxy or cloud NLB
# The load balancer forwards port 6443 to all 3 control plane nodes.

# === Step 1: Initialize the first control plane node ===
# The --control-plane-endpoint flag is crucial for HA — it tells all components
# to talk to the load balancer address, not a specific node.
sudo kubeadm init \
  --control-plane-endpoint "192.168.1.100:6443" \
  --upload-certs \
  --pod-network-cidr=10.244.0.0/16

# This outputs a join command with a certificate key:
# kubeadm join 192.168.1.100:6443 --token <token> \
#   --discovery-token-ca-cert-hash sha256:<hash> \
#   --control-plane --certificate-key <cert-key>

# === Step 2: Install CNI (e.g., Calico) ===
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

# === Step 3: Join additional control plane nodes ===
# Run on cp2 and cp3:
sudo kubeadm join 192.168.1.100:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane \
  --certificate-key <cert-key>

# === Step 4: Verify HA ===
kubectl get nodes
# NAME   STATUS   ROLES           AGE   VERSION
# cp1    Ready    control-plane   10m   v1.29.0
# cp2    Ready    control-plane   5m    v1.29.0
# cp3    Ready    control-plane   3m    v1.29.0

# Check etcd cluster health
kubectl -n kube-system exec etcd-cp1 -- etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list -w table

# Check leader election
kubectl -n kube-system get lease kube-controller-manager -o jsonpath='{.spec.holderIdentity}'
kubectl -n kube-system get lease kube-scheduler -o jsonpath='{.spec.holderIdentity}'

# === Test failover ===
# Stop one control plane node and verify the cluster continues to function:
# ssh cp1 "sudo systemctl stop kubelet"
# kubectl get nodes  # cp1 becomes NotReady, but cluster works fine
```

---

## 10.10 client-go — The Go Client Library

client-go is the official Go client library for Kubernetes. It's used by kubectl,
controllers, operators, and any Go program that interacts with Kubernetes. Understanding
client-go is essential for building Kubernetes-native applications.

### 10.10.1 Architecture Overview

```
┌──────────────────────────────────────────────────────────────────────┐
│                        client-go Architecture                         │
│                                                                        │
│  ┌──────────────┐                                                      │
│  │  rest.Config  │  ← Server URL, TLS certs, auth token               │
│  └──────┬───────┘                                                      │
│         │                                                              │
│         ▼                                                              │
│  ┌──────────────────┐                                                  │
│  │  rest.RESTClient  │  ← Low-level HTTP client (GET/POST/PUT/DELETE)  │
│  └──────┬───────────┘                                                  │
│         │                                                              │
│         ▼                                                              │
│  ┌──────────────────────────────────────────────┐                     │
│  │  kubernetes.Clientset                         │                     │
│  │                                                │                     │
│  │  ┌─ CoreV1()  → pods, services, configmaps    │                     │
│  │  ├─ AppsV1()  → deployments, replicasets      │                     │
│  │  ├─ BatchV1() → jobs, cronjobs                │                     │
│  │  └─ ...       → every built-in API group      │                     │
│  └──────────────────────────────────────────────┘                     │
│                                                                        │
│  For Custom Resources (CRDs):                                         │
│  ┌──────────────────┐                                                  │
│  │  dynamic.Client   │  ← Works with any GVR using unstructured types  │
│  └──────────────────┘                                                  │
│                                                                        │
│  For Caching (used by controllers):                                   │
│  ┌──────────────────────────────────────────────────────────────┐     │
│  │  Informer System                                              │     │
│  │                                                                │     │
│  │  ┌───────────┐    ┌──────────┐    ┌────────────┐             │     │
│  │  │ Reflector  │───▶│  Store   │───▶│   Lister    │             │     │
│  │  │ (List+Watch│    │ (cache)  │    │ (read from  │             │     │
│  │  │  from API) │    │          │    │  cache)     │             │     │
│  │  └───────────┘    └──────────┘    └────────────┘             │     │
│  │        │                                                      │     │
│  │        └──▶ Event Handlers (AddFunc, UpdateFunc, DeleteFunc)  │     │
│  │                │                                               │     │
│  │                ▼                                               │     │
│  │  ┌────────────────────┐                                       │     │
│  │  │  Work Queue         │  ← Rate-limited, delayed, deduped    │     │
│  │  │  (key = ns/name)   │                                       │     │
│  │  └────────────────────┘                                       │     │
│  └──────────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────────┘
```

### 10.10.2 Informers — Cached Watch with Indexing

Informers are the backbone of every Kubernetes controller. They provide:

1. **List + Watch**: Initial list of all objects, then watch for changes.
2. **In-memory cache**: Controllers read from cache, not the API server.
3. **Event handlers**: Callbacks when objects are added, updated, or deleted.
4. **Resync**: Periodic full reconciliation (level-triggered safety net).

```
  ┌──────────────┐       List + Watch        ┌───────────────┐
  │   Informer    │ ◀───────────────────────── │  API Server   │
  │               │                            │               │
  │  ┌──────────┐│                            └───────────────┘
  │  │ Reflector ││  1. Initial: LIST /api/v1/pods
  │  │          ││     → Gets all pods + resourceVersion
  │  │          ││  2. Then: WATCH /api/v1/pods?resourceVersion=X
  │  │          ││     → Streams changes as they happen
  │  └────┬─────┘│
  │       │      │
  │       ▼      │
  │  ┌──────────┐│   In-memory store (thread-safe map)
  │  │  Store    ││   key: "namespace/name" → value: *v1.Pod
  │  │ (cache)  ││
  │  └────┬─────┘│
  │       │      │
  │       ▼      │
  │  Event Handlers:
  │    OnAdd(pod)     → enqueue "default/nginx" to work queue
  │    OnUpdate(old, new) → enqueue "default/nginx"
  │    OnDelete(pod)  → enqueue "default/nginx"
  └──────────────┘
```

### 10.10.3 Listers — Reading from Cache

Listers provide read access to the informer's cache. They never hit the API server:

```go
// Using a Lister — reads from in-memory cache, not API server.
// This is O(1) and doesn't create any network traffic.
pod, err := podLister.Pods("default").Get("nginx")

// vs. using the Clientset directly — makes an HTTP request to API server.
// This creates network traffic and load on the API server.
pod, err := clientset.CoreV1().Pods("default").Get(ctx, "nginx", metav1.GetOptions{})
```

Controllers should **always** use Listers for reads and the Clientset for writes.

### 10.10.4 Work Queues — Rate-Limited and Delayed

client-go provides a sophisticated work queue that:
- **Deduplicates**: If the same key is added multiple times, it's processed once.
- **Rate-limits**: Prevents controllers from hammering the API server.
- **Delays**: Allows re-queueing an item to retry later.

```go
// Create a rate-limited work queue.
// The rate limiter uses exponential backoff:
//   - First retry:  5ms
//   - Second retry: 10ms
//   - Third retry:  20ms
//   - ... up to maxDelay (1000 seconds)
queue := workqueue.NewRateLimitingQueue(
	workqueue.DefaultControllerRateLimiter(),
)

// Add an item (deduplicated — adding "default/nginx" twice results in one processing)
queue.Add("default/nginx")

// Get an item to process
item, shutdown := queue.Get()
key := item.(string)

// Process the item...
err := processItem(key)
if err != nil {
	// Requeue with rate limiting (exponential backoff)
	queue.AddRateLimited(key)
} else {
	// Mark as done (removes from tracking)
	queue.Done(key)
	queue.Forget(key) // Reset rate limit counter
}
```

### 10.10.5 Hands-On: Full client-go Example — List/Watch Pods with Informers

```go
package main

import (
	"fmt"
	"os"
	"os/signal"
	"path/filepath"
	"syscall"
	"time"

	v1 "k8s.io/api/core/v1"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
)

// podInformerDemo demonstrates using client-go informers to watch for pod
// changes across the entire cluster.
//
// Informers are the preferred way to watch Kubernetes resources because they:
//   - Maintain an in-memory cache (reducing API server load)
//   - Automatically reconnect on watch errors
//   - Provide event handlers for add/update/delete
//   - Support periodic resync for level-triggered reconciliation
//
// This is the same mechanism used by every built-in Kubernetes controller.
func main() {
	// Load kubeconfig from the default location (~/.kube/config).
	home, _ := os.UserHomeDir()
	kubeconfig := filepath.Join(home, ".kube", "config")

	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error building kubeconfig: %v\n", err)
		os.Exit(1)
	}

	// Create a Clientset — this is the typed client for all built-in resources.
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error creating clientset: %v\n", err)
		os.Exit(1)
	}

	// Create a SharedInformerFactory with a 30-second resync period.
	//
	// The factory creates informers for all resource types. SharedInformers
	// ensure that multiple controllers watching the same resource type share
	// the same underlying watch connection and cache.
	//
	// The resync period (30s here) controls how often the informer delivers
	// synthetic "update" events for ALL cached objects. This is the
	// level-triggered safety net — even if watch events are missed, the
	// controller will reconcile every object every 30 seconds.
	factory := informers.NewSharedInformerFactory(clientset, 30*time.Second)

	// Get a pod informer from the factory.
	// This doesn't start watching yet — it just registers the informer.
	podInformer := factory.Core().V1().Pods().Informer()

	// Register event handlers. These callbacks fire when pods are
	// added, updated, or deleted. In a real controller, these would
	// enqueue the pod's key (namespace/name) into a work queue.
	podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
			// AddFunc is called when a new pod appears in the cache.
			// This fires for ALL existing pods on initial list, and for
			// newly created pods after that.
			AddFunc: func(obj interface{}) {
				pod := obj.(*v1.Pod)
				fmt.Printf("[ADD]    %-50s  Phase=%-12s  Node=%s\n",
					pod.Namespace+"/"+pod.Name,
					pod.Status.Phase,
					pod.Spec.NodeName,
				)
			},

			// UpdateFunc is called when a pod in the cache is modified.
			// It receives both the old and new versions of the object.
			// In a real controller, you'd compare them to decide whether
			// to reconcile.
			UpdateFunc: func(oldObj, newObj interface{}) {
				oldPod := oldObj.(*v1.Pod)
				newPod := newObj.(*v1.Pod)

				// Only print if something meaningful changed.
				if oldPod.Status.Phase != newPod.Status.Phase {
					fmt.Printf("[UPDATE] %-50s  Phase: %s → %s\n",
						newPod.Namespace+"/"+newPod.Name,
						oldPod.Status.Phase,
						newPod.Status.Phase,
					)
				}
			},

			// DeleteFunc is called when a pod is removed from the cache.
			// The object may be a *v1.Pod or a DeletedFinalStateUnknown
			// (if the watch was disconnected when the delete happened).
			DeleteFunc: func(obj interface{}) {
				pod := obj.(*v1.Pod)
				fmt.Printf("[DELETE] %-50s\n",
					pod.Namespace+"/"+pod.Name,
				)
			},
		})

	// Create a stop channel. When this channel is closed, all informers
	// will stop their list/watch loops and shut down cleanly.
	stopCh := make(chan struct{})

	// Handle SIGINT/SIGTERM for graceful shutdown.
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		<-sigCh
		fmt.Println("\nShutting down...")
		close(stopCh)
	}()

	// Start ALL registered informers. This kicks off:
	// 1. LIST /api/v1/pods (get all pods + resourceVersion)
	// 2. WATCH /api/v1/pods?resourceVersion=X (stream changes)
	fmt.Println("Starting pod informer (Ctrl+C to stop)...")
	factory.Start(stopCh)

	// Wait for the cache to sync — i.e., the initial LIST has completed
	// and all existing pods are in the cache. Never process events before
	// the cache is synced, or you'll see "add" events for objects that
	// already existed.
	fmt.Println("Waiting for cache sync...")
	factory.WaitForCacheSync(stopCh)
	fmt.Println("Cache synced! Watching for changes...\n")

	// Block until stop signal.
	<-stopCh
}
```

### 10.10.6 Hands-On: Building a Simple Pod Monitor in Go

```go
package main

import (
	"context"
	"fmt"
	"os"
	"os/signal"
	"path/filepath"
	"syscall"
	"time"

	v1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/util/wait"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/workqueue"
	"k8s.io/klog/v2"
)

// PodMonitor is a simple controller that watches for pods in a
// CrashLoopBackOff state and logs alerts. It demonstrates the
// full controller pattern used by every Kubernetes controller:
//
//   Informer (List+Watch) → Event Handler → Work Queue → Worker → Reconcile
//
// This is production-quality structure — the same pattern used in
// kube-controller-manager, the scheduler, and most operators.
type PodMonitor struct {
	// clientset provides typed access to the Kubernetes API.
	// Used for reads that bypass the cache (e.g., getting the very latest
		// status) and for writes (creating, updating, deleting objects).
	clientset kubernetes.Interface

	// podLister provides cached, indexed read access to pods.
	// Reads from the informer cache — never hits the API server.
	// Thread-safe and very fast (O(1) by namespace+name).
	podLister cache.Indexer

	// podsSynced returns true when the informer cache has completed
	// its initial LIST and the cache is populated. Workers must wait
	// for this before processing items to avoid false "not found" errors.
	podsSynced cache.InformerSynced

	// queue is a rate-limited work queue that holds pod keys (namespace/name)
	// waiting to be reconciled. It provides:
	//   - Deduplication: adding the same key twice results in one processing
	//   - Rate limiting: exponential backoff on requeue
	//   - Graceful shutdown: queue.ShutDown() signals workers to stop
	queue workqueue.RateLimitingInterface
}

// NewPodMonitor creates a new PodMonitor controller.
// It sets up the informer event handlers and work queue, but does not
// start processing — call Run() for that.
func NewPodMonitor(clientset kubernetes.Interface, factory informers.SharedInformerFactory) *PodMonitor {
	podInformer := factory.Core().V1().Pods().Informer()

	monitor := &PodMonitor{
		clientset:  clientset,
		podLister:  podInformer.GetIndexer(),
		podsSynced: podInformer.HasSynced,
		queue: workqueue.NewRateLimitingQueue(
			workqueue.DefaultControllerRateLimiter(),
		),
	}

	// Register event handlers that enqueue pod keys.
	// We don't do any heavy processing here — just extract the key
	// and add it to the work queue. The actual reconciliation logic
	// runs in the worker goroutines.
	podInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
			AddFunc: func(obj interface{}) {
				key, err := cache.MetaNamespaceKeyFunc(obj)
				if err == nil {
					monitor.queue.Add(key)
				}
			},
			UpdateFunc: func(_, newObj interface{}) {
				key, err := cache.MetaNamespaceKeyFunc(newObj)
				if err == nil {
					monitor.queue.Add(key)
				}
			},
			DeleteFunc: func(obj interface{}) {
				key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
				if err == nil {
					monitor.queue.Add(key)
				}
			},
		})

	return monitor
}

// Run starts the controller's worker goroutines.
// It blocks until the stopCh is closed.
//
// Parameters:
//   - workers: number of concurrent worker goroutines to process the queue.
//     More workers = faster processing, but also more concurrent API calls.
//   - stopCh: closing this channel triggers graceful shutdown.
func (m *PodMonitor) Run(workers int, stopCh <-chan struct{}) error {
	defer m.queue.ShutDown()

	klog.Info("Starting PodMonitor controller")

	// Wait for the informer cache to sync before processing items.
	// If we start processing before the cache is populated, we might
	// see "not found" for objects that actually exist — they just
	// haven't been listed yet.
	klog.Info("Waiting for informer caches to sync...")
	if ok := cache.WaitForCacheSync(stopCh, m.podsSynced); !ok {
		return fmt.Errorf("failed to wait for caches to sync")
	}
	klog.Info("Informer caches synced")

	// Start worker goroutines. Each worker pulls items from the queue
	// and calls processNextWorkItem in a loop.
	klog.Infof("Starting %d workers", workers)
	for i := 0; i < workers; i++ {
		// wait.Until runs the function repeatedly (with no delay between
			// iterations) until the stop channel is closed.
		go wait.Until(m.runWorker, time.Second, stopCh)
	}

	klog.Info("PodMonitor controller started")
	<-stopCh
	klog.Info("Shutting down PodMonitor controller")
	return nil
}

// runWorker processes items from the work queue in a loop.
// It runs until the queue is shut down.
func (m *PodMonitor) runWorker() {
	for m.processNextWorkItem() {
	}
}

// processNextWorkItem pulls a single item from the work queue and
// processes it. Returns false when the queue is shut down.
//
// This method handles the full lifecycle of a queue item:
// 1. Get item from queue (blocks until one is available)
// 2. Process it (call reconcile)
// 3. On success: mark Done + Forget (reset rate limit)
// 4. On error: mark Done + AddRateLimited (retry with backoff)
func (m *PodMonitor) processNextWorkItem() bool {
	// Get blocks until an item is available or the queue is shut down.
	obj, shutdown := m.queue.Get()
	if shutdown {
		return false
	}

	// Always call Done to tell the queue we finished processing this item.
	// This is separate from Forget (which resets the rate limit counter).
	defer m.queue.Done(obj)

	key, ok := obj.(string)
	if !ok {
		// Invalid item type — forget it so it's not requeued.
		m.queue.Forget(obj)
		klog.Errorf("expected string in queue but got %T", obj)
		return true
	}

	// Run the actual reconciliation logic.
	if err := m.reconcile(key); err != nil {
		// Requeue with exponential backoff.
		// The rate limiter increases the delay each time:
		// 5ms → 10ms → 20ms → 40ms → ... up to 1000s.
		m.queue.AddRateLimited(key)
		klog.Errorf("error reconciling %s: %v (requeuing)", key, err)
		return true
	}

	// Success — reset the rate limit counter for this key.
	// Next time it's enqueued, it starts with the base delay.
	m.queue.Forget(obj)
	return true
}

// reconcile is the core business logic of the controller.
// Given a pod key (namespace/name), it checks the pod's status and
// alerts if the pod is in a CrashLoopBackOff state.
//
// This function is LEVEL-TRIGGERED: it doesn't care what event caused
// it to run. It simply examines the current state and acts accordingly.
func (m *PodMonitor) reconcile(key string) error {
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return fmt.Errorf("invalid resource key: %s", key)
	}

	// Read the pod from the informer cache (not the API server).
	// This is fast and doesn't create API server load.
	obj, exists, err := m.podLister.GetByKey(key)
	if err != nil {
		return fmt.Errorf("error fetching pod %s from cache: %v", key, err)
	}
	if !exists {
		// Pod was deleted — nothing to do.
		klog.V(4).Infof("Pod %s/%s no longer exists", namespace, name)
		return nil
	}

	pod := obj.(*v1.Pod)

	// Check each container's status for CrashLoopBackOff.
	for _, cs := range pod.Status.ContainerStatuses {
		if cs.State.Waiting != nil &&
		cs.State.Waiting.Reason == "CrashLoopBackOff" {

			// Alert! In production, this could send to PagerDuty, Slack, etc.
			klog.Warningf(
				"🚨 ALERT: Pod %s/%s container %q is in CrashLoopBackOff "+
				"(restarts: %d, message: %s)",
				namespace, name,
				cs.Name,
				cs.RestartCount,
				cs.State.Waiting.Message,
			)

			// Optionally, annotate the pod with the alert timestamp.
			// This demonstrates writing back to the API server.
			m.annotatePod(namespace, name)
		}
	}

	return nil
}

// annotatePod adds an annotation to the pod recording when the
// CrashLoopBackOff alert was triggered. This uses the Clientset
// (not the cache) because it's a write operation.
func (m *PodMonitor) annotatePod(namespace, name string) {
	ctx := context.Background()

	pod, err := m.clientset.CoreV1().Pods(namespace).Get(
		ctx, name, metav1.GetOptions{},
	)
	if err != nil {
		klog.Errorf("Failed to get pod %s/%s for annotation: %v",
			namespace, name, err)
		return
	}

	if pod.Annotations == nil {
		pod.Annotations = make(map[string]string)
	}
	pod.Annotations["pod-monitor/last-crashloop-alert"] = time.Now().UTC().Format(time.RFC3339)

	_, err = m.clientset.CoreV1().Pods(namespace).Update(
		ctx, pod, metav1.UpdateOptions{},
	)
	if err != nil {
		klog.Errorf("Failed to annotate pod %s/%s: %v",
			namespace, name, err)
	}
}

func main() {
	home, _ := os.UserHomeDir()
	kubeconfig := filepath.Join(home, ".kube", "config")

	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		klog.Fatalf("Error building kubeconfig: %v", err)
	}

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		klog.Fatalf("Error creating clientset: %v", err)
	}

	// Create the shared informer factory with a 1-minute resync period.
	// The resync period triggers synthetic "update" events for ALL cached
	// objects, ensuring level-triggered reconciliation even if watch events
	// are missed.
	factory := informers.NewSharedInformerFactory(clientset, 1*time.Minute)

	// Create the PodMonitor controller.
	monitor := NewPodMonitor(clientset, factory)

	// Set up graceful shutdown.
	stopCh := make(chan struct{})
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		<-sigCh
		close(stopCh)
	}()

	// Start informers (begins List+Watch against the API server).
	factory.Start(stopCh)

	// Run the controller with 2 worker goroutines.
	if err := monitor.Run(2, stopCh); err != nil {
		klog.Fatalf("Error running controller: %v", err)
	}
}
```

---

## 10.11 Summary

Kubernetes is a masterpiece of distributed systems engineering, built on a few key ideas:

```
┌─────────────────────────────────────────────────────────────────────┐
│                      KEY TAKEAWAYS                                   │
│                                                                       │
│  1. DECLARATIVE MODEL: Tell Kubernetes what you want, not what to    │
│     do. Control loops continuously reconcile toward desired state.    │
│                                                                       │
│  2. LEVEL-TRIGGERED: Controllers react to STATE, not events. Even    │
│     if events are lost, the next reconciliation fixes everything.    │
│                                                                       │
│  3. API SERVER IS THE HUB: All communication flows through the API   │
│     server. No component talks directly to another. This enables     │
│     clean auth, auditing, and decoupling.                            │
│                                                                       │
│  4. WATCH MECHANISM: Controllers don't poll — they watch for          │
│     changes via long-lived HTTP connections. etcd's watch is the     │
│     foundation of this entire system.                                │
│                                                                       │
│  5. OWNERSHIP TREE: Objects own other objects via ownerReferences.    │
│     Garbage collection cascades automatically.                        │
│                                                                       │
│  6. OPTIMISTIC CONCURRENCY: resourceVersion prevents conflicts       │
│     without locks. If two controllers update the same object,         │
│     one gets a conflict error and retries.                            │
│                                                                       │
│  7. client-go PATTERN: Informer → Event Handler → Work Queue →      │
│     Worker → Reconcile. Every controller follows this pattern.       │
└─────────────────────────────────────────────────────────────────────┘
```

### What's Next

| Chapter | Topic | Why |
|---|---|---|
| **11** | etcd Deep Dive | Raft consensus, compaction, performance tuning |
| **12** | Writing Custom Controllers | Build your own operator with controller-runtime |
| **13** | Scheduler Deep Dive | Custom scheduling plugins, multi-scheduler |
| **14** | kubelet Deep Dive | Pod lifecycle, PLEG, eviction, device plugins |
| **22-26** | Kubernetes Networking | CNI, Services, Ingress, DNS, eBPF |

---

## References

1. **Kubernetes Official Documentation** — https://kubernetes.io/docs/
2. **Kubernetes Source Code** — https://github.com/kubernetes/kubernetes
3. **client-go Source Code** — https://github.com/kubernetes/client-go
4. **"Kubernetes: Up and Running"** — Brendan Burns, Joe Beda, Kelsey Hightower
5. **"Programming Kubernetes"** — Michael Hausenblas, Stefan Schimanski
6. **Kubernetes Design Proposals** — https://github.com/kubernetes/design-proposals-archive
7. **etcd Documentation** — https://etcd.io/docs/
8. **CNCF Technical Reports** — https://www.cncf.io/reports/
