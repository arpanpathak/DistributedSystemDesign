# Chapter 13: Scheduler Deep Dive

> *"The scheduler is the matchmaker of Kubernetes — it decides which Pod runs where,
> balancing constraints, preferences, and resource realities across your entire cluster."*

The Kubernetes scheduler is one of the most critical control-plane components. Every Pod
that enters the cluster without an explicit node assignment passes through the scheduler.
It evaluates dozens of constraints — resource requests, affinity rules, taints,
topology spread, volume locality — and produces a single binding decision. In this
chapter, we dissect the scheduler from its watch loop to its plugin framework, build
custom scheduling plugins in Go, and learn to tune scheduling for clusters of every size.

---

## 13.1 What the Scheduler Does

At its core, the scheduler performs one job: **assign unscheduled Pods to Nodes**.

### 13.1.1 Watching for Unscheduled Pods

The scheduler runs a continuous **informer-based watch** against the API server.
It filters for Pods where `spec.nodeName` is empty — these are the Pods that need
scheduling.

```text
┌─────────────────────────────────────────────────────┐
│                    API Server                        │
│                                                      │
│   Pod A  (spec.nodeName: "")     ← Unscheduled      │
│   Pod B  (spec.nodeName: "node-2") ← Already bound  │
│   Pod C  (spec.nodeName: "")     ← Unscheduled      │
└──────────┬──────────────────────────────────────────┘
           │  Watch (fieldSelector: spec.nodeName="")
           ▼
┌─────────────────────────────────────────────────────┐
│              kube-scheduler                          │
│                                                      │
│   Scheduling Queue:                                  │
│     ┌───────┐  ┌───────┐                            │
│     │ Pod A │  │ Pod C │                             │
│     └───────┘  └───────┘                            │
└─────────────────────────────────────────────────────┘
```

The informer pushes new unscheduled Pods into the **scheduling queue** — a priority
queue that orders Pods by priority class, creation timestamp, and other factors.

### 13.1.2 Assigning Pods to Nodes

Once the scheduler selects a Pod from the queue, it runs a two-phase algorithm:

1. **Filtering** — eliminate nodes that *cannot* run the Pod.
2. **Scoring** — rank the remaining nodes to find the *best* one.

The winner node is recorded, and the scheduler proceeds to **binding**.

### 13.1.3 Binding — The Final Act

Binding is the act of telling the API server "this Pod runs on this Node." The
scheduler creates a `Binding` object:

```yaml
apiVersion: v1
kind: Binding
metadata:
  name: my-pod
  namespace: default
target:
  apiVersion: v1
  kind: Node
  name: node-3
```

The API server receives the binding, sets `spec.nodeName` on the Pod, and the kubelet
on `node-3` picks it up. The Pod is now scheduled.

**Key insight:** The scheduler does *not* start containers. It only sets
`spec.nodeName`. The kubelet does the rest.

---

## 13.2 The Scheduling Cycle

Every Pod goes through a well-defined **scheduling cycle**. Understanding this cycle
is essential for debugging scheduling failures and writing custom plugins.

### 13.2.1 The Scheduling Pipeline

```
                         Scheduling Cycle
  ┌──────────────────────────────────────────────────────────┐
  │                                                          │
  │  ┌───────────┐    ┌───────────┐    ┌───────────────┐    │
  │  │ PreFilter  │───▶│  Filter   │───▶│  PostFilter   │    │
  │  │           │    │           │    │ (if no nodes  │    │
  │  │ Validate  │    │ Eliminate │    │  passed)      │    │
  │  │ Pod data  │    │ unsuitable│    │               │    │
  │  └───────────┘    │ nodes     │    └───────────────┘    │
  │                    └─────┬─────┘                         │
  │                          │ feasible nodes                │
  │                          ▼                               │
  │                    ┌───────────┐    ┌───────────────┐    │
  │                    │ PreScore  │───▶│    Score      │    │
  │                    │           │    │               │    │
  │                    │ Prepare   │    │ Rank nodes    │    │
  │                    │ scoring   │    │ 0-100 per     │    │
  │                    │ state     │    │ plugin        │    │
  │                    └───────────┘    └──────┬────────┘    │
  │                                            │             │
  │                                            ▼             │
  │                                     ┌──────────────┐     │
  │                                     │NormalizeScore│     │
  │                                     │              │     │
  │                                     │ Scale scores │     │
  │                                     │ to 0-100     │     │
  │                                     └──────┬───────┘     │
  │                                            │             │
  └────────────────────────────────────────────┼─────────────┘
                                               │
                         Binding Cycle         │
  ┌────────────────────────────────────────────┼─────────────┐
  │                                            ▼             │
  │  ┌───────────┐  ┌─────────┐  ┌─────────┐  ┌──────────┐  │
  │  │  Reserve  │─▶│ Permit  │─▶│ PreBind │─▶│   Bind   │  │
  │  │           │  │         │  │         │  │          │  │
  │  │ Reserve   │  │ Allow / │  │ Pre-    │  │ Create   │  │
  │  │ resources │  │ Deny /  │  │ binding │  │ Binding  │  │
  │  │ on node   │  │ Wait    │  │ work    │  │ object   │  │
  │  └───────────┘  └─────────┘  └─────────┘  └────┬─────┘  │
  │                                                  │       │
  │                                                  ▼       │
  │                                           ┌──────────┐   │
  │                                           │ PostBind │   │
  │                                           │          │   │
  │                                           │ Cleanup  │   │
  │                                           └──────────┘   │
  └──────────────────────────────────────────────────────────┘
```

### 13.2.2 Filtering Phase — Which Nodes CAN Run This Pod?

The filtering phase applies hard constraints. Each filter plugin returns one of:
- **Success** — the node passes this filter.
- **Unschedulable** — the node cannot run this Pod.
- **UnschedulableAndUnresolvable** — the node cannot run this Pod and preemption
  will not help.

If a Pod fails all nodes in filtering, the scheduler invokes **PostFilter** plugins
(which may trigger preemption). If no PostFilter plugin can resolve the situation,
the Pod goes back to the queue with an `Unschedulable` condition.

Common filtering reasons:
| Reason | Plugin |
|--------|--------|
| Insufficient CPU/memory | `NodeResourcesFit` |
| Node is unschedulable | `NodeUnschedulable` |
| Node has a taint the Pod doesn't tolerate | `TaintToleration` |
| Port conflict | `NodePorts` |
| Node affinity mismatch | `NodeAffinity` |
| Volume topology mismatch | `VolumeBinding` |

### 13.2.3 Scoring Phase — Which Node is BEST?

After filtering, the scheduler scores each feasible node. Each scoring plugin
returns a value from 0 to 100 for each node. Scores are weighted by the plugin's
configured weight and summed.

```
Final Score(node) = Σ (plugin_score × plugin_weight)
```

The node with the highest total score wins. Ties are broken randomly.

### 13.2.4 Binding Phase — Tell the API Server

Once a node is selected, the scheduler:
1. **Reserves** the Pod's resources on the selected node (in the scheduler's cache).
2. Runs **Permit** plugins (which may approve, deny, or ask the scheduler to wait).
3. Runs **PreBind** plugins (e.g., provision volumes).
4. Runs the **Bind** plugin — creates the `Binding` object in the API server.
5. Runs **PostBind** plugins — cleanup and notification.

The binding cycle runs **asynchronously** from the scheduling cycle, so the scheduler
can continue scheduling other Pods while waiting for binding to complete.

---

## 13.3 Scheduling Framework — Extension Points

Kubernetes 1.19+ uses the **Scheduling Framework**, a pluggable architecture that
replaces the old predicates/priorities system. Every scheduling decision flows through
well-defined **extension points**, and each extension point is implemented by one or
more **plugins**.

### 13.3.1 Extension Point Reference

| Extension Point | Phase | Purpose | Called Per |
|----------------|-------|---------|-----------|
| **QueueSort** | Queue | Determine Pod ordering in the scheduling queue | Pod pair |
| **PreFilter** | Scheduling | Validate and precompute data for filtering | Pod |
| **Filter** | Scheduling | Eliminate infeasible nodes | Node × Pod |
| **PostFilter** | Scheduling | Handle scheduling failures (e.g., preemption) | Pod |
| **PreScore** | Scheduling | Precompute data for scoring | Pod |
| **Score** | Scheduling | Score each feasible node | Node × Pod |
| **NormalizeScore** | Scheduling | Normalize scores to [0, 100] | Plugin |
| **Reserve** | Binding | Reserve resources before binding | Pod |
| **Permit** | Binding | Approve, deny, or delay binding | Pod |
| **PreBind** | Binding | Prepare for binding (e.g., provision PVs) | Pod |
| **Bind** | Binding | Actually bind the Pod to the Node | Pod |
| **PostBind** | Binding | Notify after successful binding | Pod |

### 13.3.2 PreFilter

**When:** Before any Filter plugin runs, once per scheduling cycle.

**Purpose:** Validate Pod data and precompute information that Filter plugins will need.
If PreFilter fails, the Pod is immediately marked as unschedulable.

**Use cases:**
- Check that the Pod's resource requests are valid.
- Compute the set of nodes that have the required topology domains.
- Build a mapping of ports needed by the Pod.

**Example:** The `InterPodAffinity` plugin uses PreFilter to compute the set of
topology domains that satisfy the Pod's affinity/anti-affinity rules. This avoids
recomputing the same data for every node during filtering.

### 13.3.3 Filter

**When:** For each node in the cluster (or a subset), once per node per scheduling cycle.

**Purpose:** Return whether a specific node can run the Pod. This is a hard constraint —
if any filter plugin rejects a node, that node is eliminated.

**Use cases:**
- Check if the node has enough resources (`NodeResourcesFit`).
- Check if the node has the right labels (`NodeAffinity`).
- Check if the Pod tolerates the node's taints (`TaintToleration`).
- Check for port conflicts (`NodePorts`).

### 13.3.4 PostFilter

**When:** Only if no nodes pass the Filter phase.

**Purpose:** Attempt to make the Pod schedulable. The most common PostFilter
implementation is the **DefaultPreemption** plugin, which tries to evict
lower-priority Pods to free up resources.

**Use cases:**
- Preemption — find the cheapest set of Pods to evict.
- Logging — record why scheduling failed.

### 13.3.5 PreScore

**When:** After filtering succeeds, before scoring, once per scheduling cycle.

**Purpose:** Precompute data that scoring plugins will need.

**Use cases:**
- Compute the current topology spread of existing Pods.
- Build affinity/anti-affinity mappings.

### 13.3.6 Score

**When:** For each feasible node, once per node per scoring plugin.

**Purpose:** Return a score from 0 to `framework.MaxNodeScore` (100) indicating how
well the node fits the Pod. Higher is better.

**Use cases:**
- Prefer nodes with more available resources (`NodeResourcesFit` with
  `MostAllocated` or `LeastAllocated` strategy).
- Prefer nodes that satisfy preferred affinity rules (`NodeAffinity`).
- Prefer nodes that improve topology spread (`PodTopologySpread`).

### 13.3.7 NormalizeScore

**When:** After all Score plugins run, once per scoring plugin.

**Purpose:** Normalize raw scores to the [0, 100] range. Some plugins produce
scores that are not naturally in this range and need normalization.

### 13.3.8 Reserve

**When:** After a node is selected, before Permit.

**Purpose:** Reserve resources on the selected node in the scheduler's cache.
This prevents another Pod from being scheduled on the same node based on stale
resource data. If binding later fails, `Unreserve` is called to roll back.

**Use cases:**
- Reserve volume claims.
- Reserve device plugin resources.

### 13.3.9 Permit

**When:** After Reserve, before PreBind.

**Purpose:** Approve, deny, or delay the binding decision. This is the last chance
to prevent binding.

**Use cases:**
- **Gang scheduling** — wait until all Pods in a group are scheduled before
  allowing any of them to bind.
- **Quota enforcement** — check if binding this Pod would exceed a quota.

Permit can return three results:
- `Approve` — proceed to PreBind.
- `Deny` — reject the scheduling decision; the Pod goes back to the queue.
- `Wait` — hold the Pod for up to a configured timeout. Another Permit call
  (or a timeout) resolves the wait.

### 13.3.10 PreBind

**When:** After Permit approves, before Bind.

**Purpose:** Perform preparatory work needed before binding.

**Use cases:**
- Provision a PersistentVolume for the Pod (`VolumeBinding` plugin).
- Register the Pod with an external system.

### 13.3.11 Bind

**When:** After PreBind succeeds.

**Purpose:** Actually create the `Binding` object in the API server.

The default Bind plugin calls `clientset.CoreV1().Pods(namespace).Bind(ctx, binding)`.
Custom Bind plugins can implement alternative binding mechanisms.

### 13.3.12 PostBind

**When:** After Bind succeeds.

**Purpose:** Notification and cleanup after a successful binding.

**Use cases:**
- Update metrics.
- Notify external systems.
- Clean up temporary state from PreBind/Reserve.

---

## 13.4 Default Scheduling Plugins

The default scheduler profile (`default-scheduler`) registers a comprehensive set
of plugins. Understanding what each plugin does helps you predict scheduling behavior
and debug failures.

### 13.4.1 NodeResourcesFit

**Extension points:** PreFilter, Filter, Score

**What it does:** Ensures the node has enough allocatable resources (CPU, memory,
ephemeral storage, extended resources) to satisfy the Pod's requests. In scoring,
it can prefer nodes using different strategies:

| Strategy | Behavior |
|----------|----------|
| `LeastAllocated` | Prefer nodes with more free resources (spread load) |
| `MostAllocated` | Prefer nodes with fewer free resources (bin-pack) |
| `RequestedToCapacityRatio` | Custom scoring curve |

**Example effect:** A Pod requesting 2 CPU and 4Gi memory will be filtered out of
any node with less than 2 CPU or 4Gi memory allocatable.

```yaml
# Pod that requires specific resources
apiVersion: v1
kind: Pod
metadata:
  name: resource-hungry
spec:
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        cpu: "2"
        memory: "4Gi"
      limits:
        cpu: "4"
        memory: "8Gi"
```

### 13.4.2 NodeAffinity

**Extension points:** Filter, Score

**What it does:** Implements `spec.affinity.nodeAffinity`. In the Filter phase, it
enforces `requiredDuringSchedulingIgnoredDuringExecution`. In the Score phase, it
prefers nodes matching `preferredDuringSchedulingIgnoredDuringExecution`.

### 13.4.3 NodePorts

**Extension points:** PreFilter, Filter

**What it does:** Checks for host port conflicts. If a Pod requests `hostPort: 80`
and another Pod on the node already uses host port 80, the node is filtered out.

### 13.4.4 NodeUnschedulable

**Extension points:** Filter

**What it does:** Filters out nodes with `spec.unschedulable: true` (i.e., nodes
that have been cordoned with `kubectl cordon`). DaemonSet Pods are exempt.

### 13.4.5 TaintToleration

**Extension points:** Filter, Score

**What it does:** Filters out nodes whose taints are not tolerated by the Pod.
In scoring, prefers nodes with fewer untolerated taints (among preferred taints).

### 13.4.6 InterPodAffinity

**Extension points:** PreFilter, Filter, PreScore, Score

**What it does:** Implements Pod affinity and anti-affinity rules. In filtering,
enforces required rules. In scoring, prefers nodes that satisfy preferred rules.

This is one of the most computationally expensive plugins because it must evaluate
the existing Pods on every candidate node.

### 13.4.7 PodTopologySpread

**Extension points:** PreFilter, Filter, PreScore, Score

**What it does:** Implements `spec.topologySpreadConstraints`. Ensures Pods are
evenly spread across topology domains (zones, nodes, etc.).

### 13.4.8 VolumeBinding

**Extension points:** PreFilter, Filter, Reserve, PreBind

**What it does:** Checks that the node can satisfy the Pod's PersistentVolumeClaim
requirements. For `WaitForFirstConsumer` storage classes, it delays volume binding
until the Pod is scheduled, then provisions the volume on the selected node.

### 13.4.9 Plugin Interaction Example

Consider a Pod with these constraints:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: constrained-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values: ["us-east-1a", "us-east-1b"]
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchLabels:
            app: constrained-pod
        topologyKey: kubernetes.io/hostname
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "special"
    effect: "NoSchedule"
  containers:
  - name: app
    image: myapp:latest
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
```

The scheduling pipeline processes this Pod as follows:

```
Filter phase:
  NodeAffinity      → Only us-east-1a and us-east-1b nodes pass
  TaintToleration   → Only nodes without NoSchedule taints pass
                      (unless taint key=dedicated, value=special)
  NodeResourcesFit  → Only nodes with ≥500m CPU and ≥512Mi memory pass
  InterPodAffinity  → Only nodes NOT already running app=constrained-pod pass
  NodeUnschedulable → Only nodes that are not cordoned pass

Score phase:
  NodeResourcesFit  → Prefer nodes with more (or fewer) free resources
  PodTopologySpread → Prefer nodes that improve even distribution
  NodeAffinity      → Score preferred affinity rules (if any)
  TaintToleration   → Prefer nodes with fewer untolerated taints
```

---

## 13.5 Node Affinity & Anti-Affinity

Node affinity lets you constrain which nodes a Pod can be scheduled on based on
**node labels**. It replaces the older `nodeSelector` with a more expressive syntax.

### 13.5.1 Required Node Affinity

`requiredDuringSchedulingIgnoredDuringExecution` — the Pod **must** be placed on a
node matching the rules. If no node matches, the Pod stays unscheduled.

The name tells you exactly what happens:
- **Required during scheduling** — enforced when the scheduler makes its decision.
- **Ignored during execution** — if the node's labels change after scheduling,
  the Pod is NOT evicted.

```yaml
# Pod that MUST run on an SSD node in us-east-1
apiVersion: v1
kind: Pod
metadata:
  name: ssd-required
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values: ["ssd"]
          - key: topology.kubernetes.io/zone
            operator: In
            values: ["us-east-1a", "us-east-1b", "us-east-1c"]
  containers:
  - name: app
    image: myapp:latest
```

**How `nodeSelectorTerms` work:**
- Multiple `nodeSelectorTerms` are **OR**-ed — the node must match at least one term.
- Within a single term, multiple `matchExpressions` are **AND**-ed — the node must
  match all expressions in the term.

```yaml
# OR semantics: node must be EITHER in us-east-1a OR have disktype=ssd
nodeSelectorTerms:
- matchExpressions:
  - key: topology.kubernetes.io/zone
    operator: In
    values: ["us-east-1a"]
- matchExpressions:
  - key: disktype
    operator: In
    values: ["ssd"]
```

Available operators: `In`, `NotIn`, `Exists`, `DoesNotExist`, `Gt`, `Lt`.

### 13.5.2 Preferred Node Affinity

`preferredDuringSchedulingIgnoredDuringExecution` — the scheduler **tries** to place
the Pod on a matching node, but will place it elsewhere if no matching node is available.

Each preference has a `weight` (1-100) that influences scoring:

```yaml
# Pod that PREFERS GPU nodes but can run anywhere
apiVersion: v1
kind: Pod
metadata:
  name: gpu-preferred
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: accelerator
            operator: In
            values: ["nvidia-tesla-v100"]
      - weight: 20
        preference:
          matchExpressions:
          - key: accelerator
            operator: In
            values: ["nvidia-tesla-t4"]
  containers:
  - name: ml-training
    image: ml-trainer:latest
```

In this example:
- Nodes with `nvidia-tesla-v100` get a score bonus of 80.
- Nodes with `nvidia-tesla-t4` get a score bonus of 20.
- Nodes without any accelerator label get no bonus but are still eligible.

### 13.5.3 Combining Required and Preferred

You can combine both types. Required rules act as a hard filter; preferred rules
rank the remaining nodes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: combined-affinity
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values: ["us-east-1a", "us-east-1b"]
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: node-type
            operator: In
            values: ["high-memory"]
  containers:
  - name: app
    image: myapp:latest
```

This Pod **must** run in `us-east-1a` or `us-east-1b`, and **prefers** high-memory
nodes within those zones.

### 13.5.4 Node Anti-Affinity

There is no explicit "node anti-affinity" API. Instead, use the `NotIn` or
`DoesNotExist` operators:

```yaml
# Avoid nodes labeled with maintenance=true
requiredDuringSchedulingIgnoredDuringExecution:
  nodeSelectorTerms:
  - matchExpressions:
    - key: maintenance
      operator: DoesNotExist
```

---

## 13.6 Pod Affinity & Anti-Affinity

Pod affinity/anti-affinity lets you constrain scheduling based on **which other Pods
are already running** on candidate nodes (or more precisely, in the same topology domain).

### 13.6.1 Key Concepts

- **`labelSelector`** — selects the set of Pods to consider.
- **`topologyKey`** — the node label that defines a topology domain. Common values:
  - `kubernetes.io/hostname` — per-node.
  - `topology.kubernetes.io/zone` — per-zone.
  - `topology.kubernetes.io/region` — per-region.

**Pod affinity:** "Schedule me in the same topology domain as Pods matching this selector."

**Pod anti-affinity:** "Schedule me in a *different* topology domain from Pods matching
this selector."

### 13.6.2 Hands-on: Co-locating Pods

A web frontend that should run on the same node as its Redis cache:

```yaml
# Redis deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 3
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7
        ports:
        - containerPort: 6379
---
# Web frontend that should be co-located with Redis
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-frontend
  template:
    metadata:
      labels:
        app: web-frontend
    spec:
      affinity:
        podAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: redis
            topologyKey: kubernetes.io/hostname
      containers:
      - name: web
        image: nginx:latest
```

Each `web-frontend` Pod will only be scheduled on a node that already runs a `redis`
Pod. If no node runs Redis, the web frontend Pod will be unschedulable.

### 13.6.3 Hands-on: Spreading Pods with Anti-Affinity

Ensure no two replicas of the same app run on the same node:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spread-app
spec:
  replicas: 5
  selector:
    matchLabels:
      app: spread-app
  template:
    metadata:
      labels:
        app: spread-app
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: spread-app
            topologyKey: kubernetes.io/hostname
      containers:
      - name: app
        image: myapp:latest
```

**Warning:** With required anti-affinity, you need at least as many nodes as replicas.
If you have 5 replicas and only 3 nodes, 2 replicas will be unschedulable.

Use `preferredDuringSchedulingIgnoredDuringExecution` for soft spreading:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: spread-app
        topologyKey: kubernetes.io/hostname
```

### 13.6.4 Zone-Level Anti-Affinity

Spread replicas across availability zones:

```yaml
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
    - weight: 100
      podAffinityTerm:
        labelSelector:
          matchLabels:
            app: my-service
        topologyKey: topology.kubernetes.io/zone
```

### 13.6.5 Performance Considerations

Pod affinity/anti-affinity is **O(existing_pods × candidate_nodes)** — it can be
expensive in large clusters. For spreading Pods, prefer
`topologySpreadConstraints` (Section 13.8), which is more efficient.

---

## 13.7 Taints and Tolerations

Taints and tolerations work together to ensure that Pods are not scheduled onto
inappropriate nodes. While node affinity *attracts* Pods, taints *repel* them.

### 13.7.1 How Taints Work

A **taint** is applied to a node:

```bash
kubectl taint nodes node-1 dedicated=gpu:NoSchedule
```

This creates a taint with:
- **Key:** `dedicated`
- **Value:** `gpu`
- **Effect:** `NoSchedule`

Pods that do not have a matching **toleration** will not be scheduled on `node-1`.

### 13.7.2 Taint Effects

| Effect | Behavior |
|--------|----------|
| `NoSchedule` | New Pods without a matching toleration will NOT be scheduled here. Existing Pods are unaffected. |
| `PreferNoSchedule` | The scheduler will TRY to avoid placing Pods here, but it's not guaranteed. Soft version of NoSchedule. |
| `NoExecute` | New Pods won't be scheduled AND existing Pods without a matching toleration will be EVICTED. |

### 13.7.3 Tolerations

A toleration is specified in the Pod spec:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-pod
spec:
  tolerations:
  - key: "dedicated"
    operator: "Equal"
    value: "gpu"
    effect: "NoSchedule"
  containers:
  - name: cuda-app
    image: nvidia/cuda:12.0-base
```

Toleration operators:
- `Equal` — matches if key, value, and effect all match.
- `Exists` — matches if the key exists (ignores value).

```yaml
# Tolerate ALL taints with key "dedicated" regardless of value
tolerations:
- key: "dedicated"
  operator: "Exists"
  effect: "NoSchedule"

# Tolerate ALL taints (wildcard) — use with extreme caution
tolerations:
- operator: "Exists"
```

### 13.7.4 Built-in Taints

Kubernetes automatically applies these taints to nodes based on conditions:

| Taint | Condition | Effect |
|-------|-----------|--------|
| `node.kubernetes.io/not-ready` | Node is not Ready | `NoExecute` |
| `node.kubernetes.io/unreachable` | Node is unreachable from controller | `NoExecute` |
| `node.kubernetes.io/memory-pressure` | Node has memory pressure | `NoSchedule` |
| `node.kubernetes.io/disk-pressure` | Node has disk pressure | `NoSchedule` |
| `node.kubernetes.io/pid-pressure` | Node has PID pressure | `NoSchedule` |
| `node.kubernetes.io/network-unavailable` | Node network is not configured | `NoSchedule` |
| `node.kubernetes.io/unschedulable` | Node is cordoned | `NoSchedule` |

### 13.7.5 NoExecute and Eviction

When a `NoExecute` taint is added to a node:
1. Pods without a matching toleration are **evicted immediately**.
2. Pods with a matching toleration that specifies `tolerationSeconds` are evicted
   after the specified duration.
3. Pods with a matching toleration without `tolerationSeconds` stay forever.

```yaml
# Tolerate not-ready for 300 seconds, then get evicted
tolerations:
- key: "node.kubernetes.io/not-ready"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300
- key: "node.kubernetes.io/unreachable"
  operator: "Exists"
  effect: "NoExecute"
  tolerationSeconds: 300
```

These tolerations are **added by default** to every Pod (by the DefaultTolerationSeconds
admission controller) with `tolerationSeconds: 300`. This means Pods wait 5 minutes
before being evicted from unhealthy nodes.

### 13.7.6 Hands-on: Dedicated Nodes

Create a pool of nodes dedicated to a specific team:

```bash
# Taint the nodes
kubectl taint nodes node-1 team=ml:NoSchedule
kubectl taint nodes node-2 team=ml:NoSchedule

# Label the nodes (for affinity)
kubectl label nodes node-1 team=ml
kubectl label nodes node-2 team=ml
```

Deploy a Pod that uses both toleration and node affinity to ensure it runs
on and ONLY on ML nodes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-workload
spec:
  tolerations:
  - key: "team"
    operator: "Equal"
    value: "ml"
    effect: "NoSchedule"
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: team
            operator: In
            values: ["ml"]
  containers:
  - name: training
    image: ml-trainer:latest
```

**Why both?** The taint *repels* non-ML Pods. The affinity *attracts* ML Pods.
Without affinity, the ML Pod could be scheduled on any untainted node. Without the
taint, non-ML Pods could be scheduled on ML nodes.

---

## 13.8 Topology Spread Constraints

Topology spread constraints provide fine-grained control over how Pods are distributed
across topology domains. They are the recommended replacement for Pod anti-affinity
for spreading use cases.

### 13.8.1 Key Fields

```yaml
topologySpreadConstraints:
- maxSkew: 1
  topologyKey: topology.kubernetes.io/zone
  whenUnsatisfiable: DoNotSchedule
  labelSelector:
    matchLabels:
      app: my-app
```

| Field | Description |
|-------|-------------|
| `maxSkew` | Maximum allowed difference in Pod count between any two topology domains. |
| `topologyKey` | The node label that defines topology domains. |
| `whenUnsatisfiable` | `DoNotSchedule` (hard) or `ScheduleAnyway` (soft). |
| `labelSelector` | Selects which Pods to count for spread calculation. |
| `minDomains` | Minimum number of domains to consider (optional, v1.25+). |
| `matchLabelKeys` | Use Pod label values for spread grouping (optional, v1.27+). |
| `nodeTaintsPolicy` | `Honor` or `Ignore` taints when counting domains (v1.26+). |

### 13.8.2 How maxSkew Works

Given 3 zones with the following Pod distribution:

```
Zone A: 3 Pods     Zone B: 2 Pods     Zone C: 1 Pod
```

With `maxSkew: 1`, the skew between Zone A and Zone C is `3 - 1 = 2`, which
exceeds maxSkew. The next Pod must go to Zone C (or Zone B) to reduce skew.

With `maxSkew: 2`, the distribution is acceptable, and the next Pod could go
anywhere.

### 13.8.3 Hands-on: Spreading Across Zones

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
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: zone-spread
      containers:
      - name: app
        image: myapp:latest
```

This deployment:
1. **Hard constraint:** Pods must be evenly spread across zones (maxSkew 1).
2. **Soft constraint:** Pods should be evenly spread across nodes within each zone.

### 13.8.4 Topology Spread vs Pod Anti-Affinity

| Feature | Topology Spread | Pod Anti-Affinity |
|---------|----------------|-------------------|
| Granularity | maxSkew (allows imbalance up to N) | Binary (same/different domain) |
| Performance | O(domains) | O(existing_pods × nodes) |
| Multi-level | Multiple constraints | Requires separate rules |
| Recommended for | Even distribution | Hard exclusion (max 1 per node) |

**Rule of thumb:** Use topology spread constraints for even distribution. Use Pod
anti-affinity for strict "no more than one per node/zone" requirements.

---

## 13.9 Priority and Preemption

When cluster resources are scarce, **priority and preemption** determine which Pods
get resources and which Pods may be evicted to make room.

### 13.9.1 PriorityClass

A `PriorityClass` defines a priority value (integer). Higher values mean higher priority.

```yaml
# System-critical workloads
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: system-critical
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "For system-critical components that must always run."
---
# Default priority for regular workloads
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: default-priority
value: 1000
globalDefault: true
preemptionPolicy: PreemptLowerPriority
description: "Default priority for all workloads."
---
# Low priority for batch jobs
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-low
value: 100
globalDefault: false
preemptionPolicy: Never
description: "Low-priority batch jobs. Will NOT preempt other Pods."
```

Built-in priority classes:
- `system-cluster-critical` (value: 2000000000)
- `system-node-critical` (value: 2000001000)

### 13.9.2 Using PriorityClass

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: system-critical
  containers:
  - name: app
    image: critical-app:latest
```

### 13.9.3 How Preemption Works

When a high-priority Pod cannot be scheduled:

```
1. Scheduler tries to schedule Pod P (priority 1000000)
2. Filter phase: No node has enough resources → scheduling fails
3. PostFilter (DefaultPreemption) triggers:
   a. For each node, find the set of lower-priority Pods to evict
      that would free enough resources for Pod P
   b. Choose the node where eviction causes the least disruption:
      - Fewest PDB violations
      - Lowest total priority of evicted Pods
      - Fewest Pods evicted
      - Latest start times (newest Pods evicted first)
   c. Set Pod P's status.nominatedNodeName to the chosen node
   d. Evict the selected Pods (set deletionTimestamp)
   e. Wait for evicted Pods to terminate
   f. Re-schedule Pod P (it should now fit on the nominated node)
```

### 13.9.4 PreemptionPolicy

- `PreemptLowerPriority` (default) — this Pod can preempt lower-priority Pods.
- `Never` — this Pod will never preempt other Pods, even if it has higher priority.

Use `preemptionPolicy: Never` for batch workloads that should wait for resources
rather than disrupt running workloads:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-patient
value: 500
preemptionPolicy: Never
```

### 13.9.5 Hands-on: Observing Preemption

Set up a scenario where preemption occurs:

```bash
# Create PriorityClasses
kubectl apply -f - <<EOF
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 10000
EOF

# Fill the cluster with low-priority Pods
kubectl apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: low-priority-filler
spec:
  replicas: 20
  selector:
    matchLabels:
      app: filler
  template:
    metadata:
      labels:
        app: filler
    spec:
      priorityClassName: low-priority
      containers:
      - name: pause
        image: registry.k8s.io/pause:3.9
        resources:
          requests:
            cpu: "500m"
            memory: "256Mi"
EOF

# Wait for fillers to be running
kubectl wait --for=condition=available deployment/low-priority-filler --timeout=120s

# Now schedule a high-priority Pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: important-pod
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: nginx:latest
    resources:
      requests:
        cpu: "2"
        memory: "1Gi"
EOF

# Watch preemption happen
kubectl get pods -w
kubectl get events --field-selector reason=Preempted
```

---

## 13.10 Custom Scheduler

Kubernetes supports running multiple schedulers simultaneously. You can write a
custom scheduler for specialized workloads while keeping the default scheduler
for everything else.

### 13.10.1 Running Multiple Schedulers

Specify which scheduler a Pod should use:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: custom-scheduled-pod
spec:
  schedulerName: my-custom-scheduler
  containers:
  - name: app
    image: myapp:latest
```

If `schedulerName` is not set, the Pod uses `default-scheduler`.

### 13.10.2 Scheduler Extender (Legacy)

Before the Scheduling Framework, the **scheduler extender** was the primary extension
mechanism. An extender is an external webhook that the scheduler calls at specific
points in the scheduling cycle.

```json
{
  "urlPrefix": "http://my-extender:8888",
  "filterVerb": "filter",
  "prioritizeVerb": "prioritize",
  "weight": 5,
  "bindVerb": "bind",
  "enableHTTPS": false,
  "nodeCacheCapable": true,
  "managedResources": [
    {
      "name": "example.com/custom-resource",
      "ignoredByScheduler": true
    }
  ]
}
```

**Limitations of extenders:**
- HTTP overhead for every scheduling cycle.
- Limited to Filter and Score extension points.
- Cannot access the scheduler's internal cache.
- Difficult to handle errors and failures.

**The Scheduling Framework is the recommended approach for new plugins.**

### 13.10.3 Hands-on: Building a Custom Scheduler Plugin

The Scheduling Framework lets you write plugins as Go code that runs inside the
scheduler process. Here is a complete example of a custom scoring plugin that
prefers nodes with a specific label.

```go
// Package labelscore implements a scheduler plugin that scores nodes
// based on the presence and value of a user-defined label. Nodes whose
// label value matches the Pod annotation "prefer-node-label-value"
// receive the maximum score; all other nodes receive zero.
//
// This plugin demonstrates the Score and NormalizeScore extension points
// of the Kubernetes Scheduling Framework.
package labelscore

import (
	"context"
	"fmt"

	v1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/kubernetes/pkg/scheduler/framework"
)

// PluginName is the registered name of this scheduling plugin.
// The scheduler configuration references this name to enable the plugin.
const PluginName = "LabelScore"

// LabelScoreArgs holds the decoded configuration for the LabelScore plugin.
// The scheduler passes these arguments from the KubeSchedulerConfiguration
// when initializing the plugin.
type LabelScoreArgs struct {
	// LabelKey is the node label key to evaluate during scoring.
	// For example, "gpu-type" or "storage-tier".
	LabelKey string `json:"labelKey"`
}

// LabelScorePlugin implements the Score and NormalizeScore extension points.
// It reads a target label value from a Pod annotation and rewards nodes
// whose label value matches.
type LabelScorePlugin struct {
	// handle provides access to the scheduler's shared state, including
	// the snapshot of cluster objects (nodes, pods, etc.).
	handle framework.Handle

	// labelKey is the node label key to check. Loaded from plugin args.
	labelKey string
}

// compile-time assertions: ensure LabelScorePlugin satisfies the
// ScorePlugin interface (which includes NormalizeScore).
var _ framework.ScorePlugin = &LabelScorePlugin{}

// Name returns the registered name of this plugin. The scheduler uses
// this name to match plugin configuration entries.
func (pl *LabelScorePlugin) Name() string {
	return PluginName
}

// Score is called once for each (Pod, Node) pair during the scoring phase.
// It returns a raw score — here, 100 if the node label matches the Pod's
// preferred value, or 0 otherwise. The framework later calls NormalizeScore
// to ensure all plugins produce values in [0, MaxNodeScore].
//
// Parameters:
//   - ctx:  context for cancellation and deadlines.
//   - state: per-cycle state shared across plugins (unused here).
//   - pod:  the Pod being scheduled.
//   - nodeName: the name of the candidate node.
//
// Returns:
//   - int64: the raw score for this (Pod, Node) pair.
//   - *framework.Status: nil on success, or an error status.
func (pl *LabelScorePlugin) Score(
	ctx context.Context,
	state *framework.CycleState,
	pod *v1.Pod,
	nodeName string,
) (int64, *framework.Status) {
	// Retrieve the full Node object from the scheduler's snapshot.
	nodeInfo, err := pl.handle.SnapshotSharedLister().NodeInfos().Get(nodeName)
	if err != nil {
		return 0, framework.AsStatus(
			fmt.Errorf("getting node %q info: %w", nodeName, err),
		)
	}

	// Read the desired label value from the Pod annotation.
	// If the annotation is missing, every node scores zero — neutral.
	preferredValue, ok := pod.Annotations["prefer-node-label-value"]
	if !ok {
		return 0, nil
	}

	// Check if the node's label matches the preferred value.
	node := nodeInfo.Node()
	if node.Labels[pl.labelKey] == preferredValue {
		return framework.MaxNodeScore, nil // 100 — best possible score
	}

	return 0, nil
}

// ScoreExtensions returns the plugin itself, indicating it implements
// NormalizeScore. If no normalization is needed, return nil.
func (pl *LabelScorePlugin) ScoreExtensions() framework.ScoreExtensions {
	return pl
}

// NormalizeScore adjusts raw scores into the [0, MaxNodeScore] range.
// In this simple plugin the Score method already returns 0 or 100,
// so normalization is a no-op. More complex plugins would scale
// their scores here (e.g., map a range of latency values to 0-100).
//
// Parameters:
//   - ctx:    context for cancellation and deadlines.
//   - state:  per-cycle state.
//   - pod:    the Pod being scheduled.
//   - scores: the list of (nodeName, score) pairs to normalize in place.
//
// Returns:
//   - *framework.Status: nil on success.
func (pl *LabelScorePlugin) NormalizeScore(
	ctx context.Context,
	state *framework.CycleState,
	pod *v1.Pod,
	scores framework.NodeScoreList,
) *framework.Status {
	// Scores are already in [0, 100] — nothing to normalize.
	return nil
}

// New creates a new instance of the LabelScorePlugin. The scheduler
// calls this function during initialization, passing the decoded plugin
// arguments and a handle to shared scheduler state.
//
// Parameters:
//   - obj:    runtime.Object holding the decoded plugin args (LabelScoreArgs).
//   - handle: provides access to cluster snapshot and informer caches.
//
// Returns:
//   - framework.Plugin: the initialized plugin.
//   - error: non-nil if arguments are invalid.
func New(obj runtime.Object, handle framework.Handle) (framework.Plugin, error) {
	args, ok := obj.(*LabelScoreArgs)
	if !ok {
		return nil, fmt.Errorf("expected LabelScoreArgs, got %T", obj)
	}

	if args.LabelKey == "" {
		return nil, fmt.Errorf("labelKey must not be empty")
	}

	return &LabelScorePlugin{
		handle:   handle,
		labelKey: args.LabelKey,
	}, nil
}
```

### 13.10.4 Registering the Plugin

Build a custom scheduler binary that includes your plugin:

```go
// Package main builds a custom kube-scheduler binary that includes
// the LabelScore plugin alongside all default plugins.
//
// This is the standard pattern for extending the Kubernetes scheduler
// with custom plugins: create a main.go that registers your plugin(s)
// and then delegates to the scheduler's command infrastructure.
package main

import (
	"os"

	"k8s.io/component-base/cli"
	"k8s.io/kubernetes/cmd/kube-scheduler/app"

	// Import the custom plugin package.
	"example.com/scheduler-plugins/pkg/labelscore"
)

// main is the entry point for the custom scheduler binary.
// It registers the LabelScore plugin with the scheduler framework,
// then starts the scheduler with the full default plugin set plus
// our custom plugin.
func main() {
	// app.NewSchedulerCommand creates the scheduler command with all
	// default plugins. We add our custom plugin via WithPlugin.
	command := app.NewSchedulerCommand(
		app.WithPlugin(labelscore.PluginName, labelscore.New),
	)

	// cli.Run handles signal trapping and returns the exit code.
	code := cli.Run(command)
	os.Exit(code)
}
```

### 13.10.5 Scheduler Configuration

Configure the scheduler to use the custom plugin:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: custom-scheduler
  plugins:
    score:
      enabled:
      - name: LabelScore
        weight: 10
  pluginConfig:
  - name: LabelScore
    args:
      labelKey: "gpu-type"
```

Deploy the custom scheduler as a Deployment in the cluster:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: custom-scheduler
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      component: custom-scheduler
  template:
    metadata:
      labels:
        component: custom-scheduler
    spec:
      serviceAccountName: custom-scheduler
      containers:
      - name: scheduler
        image: my-registry/custom-scheduler:v1
        command:
        - /custom-scheduler
        - --config=/etc/scheduler/config.yaml
        - --leader-elect=true
        - --leader-elect-resource-name=custom-scheduler
        volumeMounts:
        - name: config
          mountPath: /etc/scheduler
      volumes:
      - name: config
        configMap:
          name: custom-scheduler-config
```

---

## 13.11 Scheduler Performance

In large clusters (1000+ nodes), scheduler throughput becomes critical. Kubernetes
provides several knobs to tune performance.

### 13.11.1 Percentage of Nodes to Score

By default, the scheduler does not score *all* feasible nodes. Once enough nodes
have been found feasible (controlled by `percentageOfNodesToScore`), the scheduler
stops filtering and scores only the feasible nodes found so far.

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
percentageOfNodesToScore: 50
```

Default behavior:
- For clusters with fewer than 50 nodes: score all feasible nodes (100%).
- For larger clusters: the default is `max(50, 50 × numNodes / 5000)` percent,
  with a minimum of 100 nodes.

**Trade-off:** Lower percentages improve scheduling throughput but may not find the
globally optimal node.

### 13.11.2 Scheduling Throughput

The scheduler processes one Pod per scheduling cycle. Throughput is measured in
Pods/second. Typical values:

| Cluster Size | Throughput (default config) |
|-------------|---------------------------|
| 100 nodes | ~100 Pods/sec |
| 1000 nodes | ~50 Pods/sec |
| 5000 nodes | ~10-20 Pods/sec |

Bottlenecks:
- **InterPodAffinity** — O(existing_pods × nodes). Consider `PodTopologySpread` instead.
- **VolumeBinding** — Disk I/O for PV matching.
- **Overly broad label selectors** — increase the working set size.

### 13.11.3 Profile Configuration

You can run multiple scheduling profiles in a single scheduler instance, each
with different plugin configurations:

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
  plugins:
    score:
      disabled:
      - name: NodeResourcesFit
      enabled:
      - name: NodeResourcesFit
        weight: 1
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: LeastAllocated
        resources:
        - name: cpu
          weight: 1
        - name: memory
          weight: 1

- schedulerName: binpack-scheduler
  plugins:
    score:
      disabled:
      - name: NodeResourcesFit
      enabled:
      - name: NodeResourcesFit
        weight: 1
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: MostAllocated
        resources:
        - name: cpu
          weight: 1
        - name: memory
          weight: 1
```

Pods can then choose their scheduler:

```yaml
# This Pod will be bin-packed
spec:
  schedulerName: binpack-scheduler
```

### 13.11.4 Debugging Scheduling Decisions

When a Pod is stuck in `Pending`, debug with:

```bash
# Check scheduler events for the Pod
kubectl describe pod <pod-name> | grep -A 20 Events

# Check scheduler logs (if running as a Pod)
kubectl logs -n kube-system -l component=kube-scheduler --tail=100

# Verbose scheduling: run scheduler with -v=4 or higher

# Common failure messages:
# "0/5 nodes are available: 2 Insufficient cpu, 3 node(s) had taint
#  {dedicated: gpu}, that the pod didn't tolerate."
```

The `Events` section is your first stop. The scheduler records exactly why
a Pod could not be scheduled:

```
Events:
  Type     Reason            Age   Message
  ----     ------            ----  -------
  Warning  FailedScheduling  30s   0/10 nodes are available:
    3 Insufficient cpu,
    4 node(s) had taint {node.kubernetes.io/disk-pressure: },
    3 node(s) didn't match Pod's node affinity/selector.
```

---

## 13.12 Summary

The Kubernetes scheduler is a sophisticated matchmaking engine:

| Concept | Key Takeaway |
|---------|-------------|
| **Scheduling cycle** | Filter → Score → Bind. Hard constraints then soft preferences. |
| **Scheduling Framework** | 12 extension points. Plugins implement one or more points. |
| **Node affinity** | Attract Pods to specific nodes. Required = hard, Preferred = soft. |
| **Pod affinity/anti-affinity** | Schedule Pods relative to other Pods. Expensive — prefer topology spread. |
| **Taints & tolerations** | Repel Pods from nodes. Three effects: NoSchedule, PreferNoSchedule, NoExecute. |
| **Topology spread** | Evenly distribute Pods across domains. More efficient than anti-affinity. |
| **Priority & preemption** | Higher-priority Pods evict lower-priority Pods when resources are scarce. |
| **Custom scheduler** | Write Go plugins using the Scheduling Framework. Deploy as a separate scheduler. |
| **Performance** | Tune `percentageOfNodesToScore`. Avoid expensive plugins in large clusters. |

In the next chapter, we descend from the control plane to the node level, where the
**kubelet** and **Container Runtime Interface** turn scheduling decisions into running
containers.

---

## Exercises

1. **Affinity Lab:** Create a 3-node cluster (using kind or minikube). Label nodes
   with different zones. Deploy a 6-replica Deployment with topology spread constraints
   that ensures even distribution across zones. Verify with `kubectl get pods -o wide`.

2. **Taint Lab:** Taint one node with `NoSchedule`. Deploy a Pod without a toleration
   and observe it staying Pending. Add the toleration and observe it scheduling.

3. **Preemption Lab:** Create a `low-priority` and `high-priority` PriorityClass.
   Fill the cluster with low-priority Pods. Deploy a high-priority Pod and observe
   preemption events.

4. **Custom Plugin:** Extend the `LabelScorePlugin` from Section 13.10.3 to also
   implement the Filter extension point. Filter out nodes that do NOT have the
   target label at all.

5. **Performance Experiment:** In a cluster with 50+ nodes, measure scheduling
   throughput with `percentageOfNodesToScore` set to 10%, 50%, and 100%. Compare
   the time to schedule 100 Pods.
