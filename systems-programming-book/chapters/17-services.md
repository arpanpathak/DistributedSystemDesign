# Chapter 17: Services, Endpoints & Service Discovery

> *"A Pod without a Service is like a phone without a number — reachable only if you
> happen to know exactly where it is right now."*

In Chapter 16 we deployed Pods and watched them come and go. Every time a Pod
restarts, moves to a different node, or scales up, it receives a **brand-new IP
address**. If your front-end hard-codes the IP of your back-end Pod, the
connection breaks the instant Kubernetes reschedules that Pod.

**Services** solve this problem. They give a *stable identity* — a virtual IP
and a DNS name — to a logical set of Pods. Clients talk to the Service; the
Service routes traffic to whichever Pods are healthy right now.

This chapter takes you from "why Services exist" all the way through kube-proxy
internals, EndpointSlices, DNS-based discovery, session affinity, and a
hands-on Go client that discovers endpoints at runtime.

---

## 17.1 Why Services Exist

### 17.1.1 The Ephemeral Nature of Pods

Every Pod in Kubernetes is **mortal**. When a Pod dies, the ReplicaSet (or
Deployment, or StatefulSet) creates a replacement — but that replacement gets a
completely new IP address. Consider a simple three-replica Deployment:

```
Time T0:
  pod-abc  →  10.244.1.5
  pod-def  →  10.244.2.8
  pod-ghi  →  10.244.3.2

Time T1 (pod-def restarted on a different node):
  pod-abc  →  10.244.1.5
  pod-xyz  →  10.244.1.9     ← new Pod, new IP
  pod-ghi  →  10.244.3.2
```

Any client that cached `10.244.2.8` now sends packets into the void. This is
not a bug — it is a fundamental property of container orchestration. Pods are
cattle, not pets.

### 17.1.2 Stable Virtual IP and DNS Name

A Kubernetes **Service** provides:

| Property | What it gives you |
|---|---|
| **ClusterIP** | A virtual IP that never changes for the lifetime of the Service |
| **DNS name** | `<service>.<namespace>.svc.cluster.local` — resolvable from any Pod |
| **Port mapping** | Decouples the port clients use from the port containers listen on |
| **Load balancing** | Distributes traffic across all healthy backend Pods |

The Service object itself is stored in etcd. The moment you create it,
Kubernetes allocates a ClusterIP from the service CIDR (e.g., `10.96.0.0/12`)
and starts routing traffic.

### 17.1.3 Label Selectors → Dynamic Endpoint Lists

A Service does not hard-code Pod IPs. Instead, it carries a **label selector**:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
  namespace: production
spec:
  selector:
    app: backend          # ← any Pod with this label is a backend
    tier: api
  ports:
    - protocol: TCP
      port: 80            # ← the port clients use
      targetPort: 8080    # ← the port the container listens on
```

The **Endpoints controller** (and the newer **EndpointSlice controller**)
continuously watches Pods. Whenever a Pod matches the selector *and* passes its
readiness probe, it is added to the endpoint list. When it fails or is deleted,
it is removed. This is entirely dynamic — no manual registration required.

```
┌──────────────────────────────────────────────────┐
│                   Service: backend                │
│          ClusterIP: 10.96.45.12:80               │
│          Selector: app=backend, tier=api          │
│                                                   │
│   ┌────────────────────────────────────────────┐  │
│   │ EndpointSlice                              │  │
│   │  10.244.1.5:8080  (Ready)                  │  │
│   │  10.244.1.9:8080  (Ready)                  │  │
│   │  10.244.3.2:8080  (Ready)                  │  │
│   └────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────┘
```

---

## 17.2 Service Types

Kubernetes defines four Service types. Each one builds on the previous,
forming a layered model:

```
ExternalName   (DNS CNAME — no proxying)

LoadBalancer   = cloud LB  + NodePort + ClusterIP
NodePort       = node port + ClusterIP
ClusterIP      = virtual IP only (internal)
```

### 17.2.1 ClusterIP (Default) — Internal Cluster Access Only

#### How ClusterIP Works

When you create a ClusterIP Service, the following happens:

1. **API server** allocates a virtual IP from the service CIDR and stores the
   Service object in etcd.
2. **EndpointSlice controller** creates EndpointSlice resources that list the
   IPs and ports of matching, ready Pods.
3. **kube-proxy** (running on every node) watches Services and EndpointSlices.
   It programs the node's packet filter to intercept traffic destined for the
   ClusterIP and DNAT it to a backend Pod IP.

In **iptables mode** (the default), the flow looks like this:

```
┌─────────────────────────────────────────────────────────────────┐
│ Node                                                             │
│                                                                  │
│  Client Pod ──► [iptables PREROUTING / OUTPUT]                   │
│                     │                                            │
│                     ▼                                            │
│            KUBE-SERVICES chain                                   │
│                     │                                            │
│           match dst=10.96.45.12:80                               │
│                     │                                            │
│                     ▼                                            │
│         KUBE-SVC-XXXXXXXX chain                                  │
│           ┌─────────┼─────────┐                                  │
│           │ 33%     │ 33%     │ 33%       (probability-based)    │
│           ▼         ▼         ▼                                  │
│        KUBE-SEP-A  KUBE-SEP-B  KUBE-SEP-C                       │
│        DNAT to     DNAT to     DNAT to                           │
│       10.244.1.5  10.244.1.9  10.244.3.2                        │
│                                                                  │
│  Packet now has dst=10.244.x.x → routed via CNI overlay/routing │
└─────────────────────────────────────────────────────────────────┘
```

The virtual IP `10.96.45.12` never appears on any network interface. It exists
only inside the iptables rules. That is why you cannot `ping` a ClusterIP — ICMP
packets are not matched by the DNAT rules (which are TCP/UDP-specific).

In **IPVS mode**, the virtual IP is bound to a dummy interface (`kube-ipvs0`),
and the IPVS kernel module performs the DNAT. This is more efficient at scale
(O(1) lookup vs. O(n) iptables chain traversal).

#### Hands-on: Create a ClusterIP Service and Trace Packet Flow

First, deploy a simple backend:

```yaml
# File: backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  labels:
    app: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: hashicorp/http-echo:0.2.3
          args:
            - "-text=Hello from backend"
            - "-listen=:8080"
          ports:
            - containerPort: 8080
```

Now create the Service:

```yaml
# File: backend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  type: ClusterIP          # default — can be omitted
  selector:
    app: backend
  ports:
    - name: http
      protocol: TCP
      port: 80              # clients connect to this port
      targetPort: 8080      # container listens on this port
```

Apply and inspect:

```bash
$ kubectl apply -f backend-deployment.yaml -f backend-service.yaml

$ kubectl get svc backend
NAME      TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)   AGE
backend   ClusterIP   10.96.45.12   <none>        80/TCP    5s

$ kubectl get endpointslices -l kubernetes.io/service-name=backend
NAME            ADDRESSTYPE   PORTS   ENDPOINTS                             AGE
backend-abc12   IPv4          8080    10.244.1.5,10.244.1.9,10.244.3.2     5s
```

Trace the packet path from a debug Pod:

```bash
# Launch a debug Pod
$ kubectl run debug --image=nicolaka/netshoot --rm -it -- bash

# Inside the debug Pod:
$ curl http://backend.default.svc.cluster.local:80
Hello from backend

# Watch DNS resolution
$ nslookup backend.default.svc.cluster.local
Server:    10.96.0.10
Address:   10.96.0.10#53

Name:   backend.default.svc.cluster.local
Address: 10.96.45.12          ← the ClusterIP

# Examine iptables on a node (requires SSH or nsenter)
$ sudo iptables -t nat -L KUBE-SERVICES -n | grep 10.96.45.12
KUBE-SVC-XXXXXXXX  tcp  --  0.0.0.0/0  10.96.45.12  tcp dpt:80
```

---

### 17.2.2 NodePort — Expose on Every Node's IP

A **NodePort** Service extends ClusterIP by opening a static port on **every
node** in the cluster. External clients can reach the service at
`<any-node-ip>:<node-port>`.

```
External Client
       │
       ▼
  NodeIP:31234         ← any node in the cluster
       │
       ▼
  ClusterIP:80         ← internal ClusterIP is still created
       │
       ▼
  Pod IP:8080          ← one of the backend Pods
```

**Port range:** By default, NodePort allocates from **30000–32767**. This range
is configured via `--service-node-port-range` on the API server.

#### How NodePort Builds on ClusterIP

When you create a NodePort Service, Kubernetes:

1. Allocates a ClusterIP (same as a ClusterIP Service).
2. Allocates a port from the NodePort range (or uses the one you specified).
3. Instructs kube-proxy on every node to listen on that port and forward
   traffic to the ClusterIP, which then DNATs to a Pod.

The iptables chains gain an additional rule in `KUBE-NODEPORTS` that matches
traffic arriving on the node port and jumps to the same `KUBE-SVC-*` chain.

#### Hands-on: Create a NodePort Service

```yaml
# File: backend-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-nodeport
spec:
  type: NodePort
  selector:
    app: backend
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
      nodePort: 31234      # optional — Kubernetes assigns one if omitted
```

```bash
$ kubectl apply -f backend-nodeport.yaml

$ kubectl get svc backend-nodeport
NAME               TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
backend-nodeport   NodePort   10.96.78.200   <none>        80:31234/TCP   3s

# Access from outside the cluster (replace NODE_IP with any node's IP)
$ curl http://${NODE_IP}:31234
Hello from backend
```

> **Caveat:** NodePort exposes the service on *all* nodes, even nodes that do
> not run any backend Pods. This means traffic may take an extra hop from the
> receiving node to a node that actually has a Pod. See `externalTrafficPolicy`
> in §17.9 for how to avoid this.

---

### 17.2.3 LoadBalancer — External Load Balancer (Cloud)

A **LoadBalancer** Service is the standard way to expose a service to the
internet on cloud providers (AWS, GCP, Azure). It provisions a cloud load
balancer that forwards traffic to the NodePort.

#### The Layered Model

```
Internet Client
       │
       ▼
  Cloud Load Balancer   ← provisioned by cloud controller manager
       │
       ▼
  NodeIP:31234          ← NodePort on one of the nodes
       │
       ▼
  ClusterIP:80          ← internal virtual IP
       │
       ▼
  Pod IP:8080           ← backend Pod
```

Every LoadBalancer Service *also* creates a NodePort *and* a ClusterIP. You
get all three layers.

#### externalTrafficPolicy: Cluster vs. Local

This field controls how traffic is distributed once it reaches a node:

| Policy | Behavior | Source IP | Distribution |
|---|---|---|---|
| `Cluster` (default) | kube-proxy may forward to a Pod on a *different* node | SNAT'd (lost) | Even across all Pods |
| `Local` | kube-proxy only forwards to Pods on *this* node | Preserved | Uneven if Pods are not balanced |

With `externalTrafficPolicy: Local`:
- If a node has **no** backend Pods, the health check fails and the load
  balancer stops sending traffic to that node.
- Source IP is preserved because there is no SNAT hop between nodes.
- Useful for geo-aware load balancing and logging real client IPs.

#### Hands-on: LoadBalancer with MetalLB (Bare-Metal / kind)

On bare-metal or `kind` clusters, there is no cloud controller to provision a
load balancer. **MetalLB** fills this gap.

Install MetalLB:

```bash
$ kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.12/config/manifests/metallb-native.yaml
```

Configure an IP address pool (adjust the range to your network):

```yaml
# File: metallb-pool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: default-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.240-192.168.1.250
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: default
  namespace: metallb-system
```

Now create a LoadBalancer Service:

```yaml
# File: backend-lb.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-lb
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local    # preserve source IP
  selector:
    app: backend
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 8080
```

```bash
$ kubectl apply -f metallb-pool.yaml -f backend-lb.yaml

$ kubectl get svc backend-lb
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP     PORT(S)        AGE
backend-lb   LoadBalancer   10.96.90.100   192.168.1.240   80:31999/TCP   10s

# Access via the external IP
$ curl http://192.168.1.240
Hello from backend
```

MetalLB responds to ARP requests for `192.168.1.240`, directing traffic to a
node that has a ready backend Pod (because `externalTrafficPolicy: Local`
causes nodes without Pods to fail health checks).

---

### 17.2.4 ExternalName — CNAME Alias to External Service

An **ExternalName** Service creates a DNS `CNAME` record that points to an
external hostname. No proxying, no ClusterIP, no endpoints — just DNS.

#### Use Case: Database Migration Without App Changes

Suppose your application connects to `database.production.svc.cluster.local`.
Initially, the database runs outside the cluster at `db.legacy.example.com`.
Later, you migrate it into the cluster.

**Phase 1 — external database:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
  namespace: production
spec:
  type: ExternalName
  externalName: db.legacy.example.com
```

Your application resolves `database.production.svc.cluster.local` and gets
back `db.legacy.example.com` via CNAME. No IP allocation, no kube-proxy rules.

**Phase 2 — migrate to in-cluster:** Replace the ExternalName Service with a
regular ClusterIP Service that selects the new in-cluster database Pods. The
application code never changes — it still connects to
`database.production.svc.cluster.local`.

#### Hands-on: ExternalName Service

```yaml
# File: external-db.yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: db.example.com
```

```bash
$ kubectl apply -f external-db.yaml

$ kubectl run debug --image=nicolaka/netshoot --rm -it -- nslookup external-db.default.svc.cluster.local
Server:    10.96.0.10
Address:   10.96.0.10#53

external-db.default.svc.cluster.local  canonical name = db.example.com
```

> **Warning:** ExternalName does not support ports. The client must know which
> port to use. It also does not work well with TLS because the certificate on
> the external server will not match the Kubernetes DNS name.

---

### 17.2.5 Headless Services (clusterIP: None)

A **headless** Service has no ClusterIP. Instead of returning a single virtual
IP, DNS returns the individual Pod IPs as A/AAAA records.

#### When to Use Headless Services

- **StatefulSets:** Each Pod needs a stable, unique DNS name
  (`pod-0.service.namespace.svc.cluster.local`).
- **Client-side load balancing:** The client (or a library like gRPC) wants to
  know all endpoints and balance traffic itself.
- **Service discovery for peer-to-peer protocols:** e.g., Cassandra gossip,
  Elasticsearch cluster formation.

#### Headless Service with a Selector

```yaml
# File: backend-headless.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-headless
spec:
  clusterIP: None           # ← this makes it headless
  selector:
    app: backend
  ports:
    - name: http
      port: 8080
      targetPort: 8080
```

#### Hands-on: Headless Service DNS Lookup

```bash
$ kubectl apply -f backend-headless.yaml

# DNS lookup returns all Pod IPs, not a single ClusterIP
$ kubectl run debug --image=nicolaka/netshoot --rm -it -- \
    nslookup backend-headless.default.svc.cluster.local

Server:    10.96.0.10
Address:   10.96.0.10#53

Name:   backend-headless.default.svc.cluster.local
Address: 10.244.1.5
Name:   backend-headless.default.svc.cluster.local
Address: 10.244.1.9
Name:   backend-headless.default.svc.cluster.local
Address: 10.244.3.2
```

Compare this with a regular ClusterIP Service, which returns a *single* IP:

```bash
$ nslookup backend.default.svc.cluster.local
Name:   backend.default.svc.cluster.local
Address: 10.96.45.12         ← single virtual IP
```

#### Headless + StatefulSet

When a headless Service fronts a StatefulSet, each Pod gets a unique DNS entry:

```
web-0.backend-headless.default.svc.cluster.local → 10.244.1.5
web-1.backend-headless.default.svc.cluster.local → 10.244.1.9
web-2.backend-headless.default.svc.cluster.local → 10.244.3.2
```

This is how databases (PostgreSQL, MySQL, CockroachDB) discover their peers.

---

## 17.3 EndpointSlices (Replacing Endpoints)

### 17.3.1 Why EndpointSlices Replaced Endpoints

The original `Endpoints` resource stored *all* backend addresses in a single
object. For a Service with 5,000 Pods, that single Endpoints object became
enormous. Every time a single Pod changed state, the *entire* object was
re-serialized and pushed to every kube-proxy via the watch stream.

**EndpointSlices** (GA since Kubernetes 1.21) solve this by splitting endpoints
into smaller chunks (default: 100 endpoints per slice). When one Pod changes,
only the affected slice is updated.

```
Before (Endpoints):
┌─────────────────────────────────────────────┐
│ Endpoints: backend                           │
│   5000 addresses in one object              │
│   Any change → full object reserialized     │
└─────────────────────────────────────────────┘

After (EndpointSlices):
┌──────────────┐ ┌──────────────┐     ┌──────────────┐
│ Slice 1      │ │ Slice 2      │ ... │ Slice 50     │
│ 100 addrs    │ │ 100 addrs    │     │ 100 addrs    │
│              │ │              │     │              │
│ Change in    │ │              │     │              │
│ this slice   │ │ (untouched)  │     │ (untouched)  │
│ → only this  │ │              │     │              │
│   resent     │ │              │     │              │
└──────────────┘ └──────────────┘     └──────────────┘
```

### 17.3.2 EndpointSlice Structure

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: backend-abc12
  namespace: default
  labels:
    kubernetes.io/service-name: backend    # links to Service
  ownerReferences:
    - apiVersion: v1
      kind: Service
      name: backend
addressType: IPv4
ports:
  - name: http
    protocol: TCP
    port: 8080
endpoints:
  - addresses:
      - "10.244.1.5"
    conditions:
      ready: true
      serving: true
      terminating: false
    targetRef:
      kind: Pod
      name: backend-abc
      namespace: default
    nodeName: worker-1
    zone: us-east-1a
  - addresses:
      - "10.244.1.9"
    conditions:
      ready: true
      serving: true
      terminating: false
    targetRef:
      kind: Pod
      name: backend-xyz
      namespace: default
    nodeName: worker-2
    zone: us-east-1b
```

Key fields:

| Field | Purpose |
|---|---|
| `addressType` | `IPv4`, `IPv6`, or `FQDN` |
| `endpoints[].conditions.ready` | Whether the Pod is passing readiness probes |
| `endpoints[].conditions.serving` | Whether the Pod is able to serve (may be true during termination) |
| `endpoints[].conditions.terminating` | Whether the Pod is in the process of shutting down |
| `endpoints[].nodeName` | Which node the Pod runs on (used for topology-aware routing) |
| `endpoints[].zone` | Availability zone (used for topology-aware routing) |

### 17.3.3 How the EndpointSlice Controller Manages Them

The **EndpointSlice controller** runs inside `kube-controller-manager`. It:

1. Watches all Services and all Pods.
2. For each Service with a selector, it identifies matching, ready Pods.
3. It creates, updates, or deletes EndpointSlice objects as Pods come and go.
4. It respects the maximum endpoints per slice (default 100) and will create
   additional slices as needed.
5. It sets `ownerReferences` so that deleting the Service garbage-collects its
   EndpointSlices.

**For Services without a selector**, no EndpointSlices are auto-created. You
can manually create EndpointSlice objects to point at external IPs — useful for
integrating non-Kubernetes services.

### 17.3.4 Hands-on: Examining EndpointSlices with kubectl

```bash
# List all EndpointSlices for a service
$ kubectl get endpointslices -l kubernetes.io/service-name=backend
NAME            ADDRESSTYPE   PORTS   ENDPOINTS                             AGE
backend-abc12   IPv4          8080    10.244.1.5,10.244.1.9,10.244.3.2     2m

# Detailed view
$ kubectl describe endpointslice backend-abc12
Name:         backend-abc12
Namespace:    default
Labels:       kubernetes.io/service-name=backend
AddressType:  IPv4
Ports:
  Name  Port  Protocol
  ----  ----  --------
  http  8080  TCP
Endpoints:
  - Addresses:  10.244.1.5
    Conditions:
      Ready:        true
      Serving:      true
      Terminating:  false
    TargetRef:      Pod/backend-abc
    NodeName:       worker-1
    Zone:           us-east-1a
  ...

# Watch for changes in real-time
$ kubectl get endpointslices -l kubernetes.io/service-name=backend -w

# Scale the deployment and watch slices update
$ kubectl scale deployment backend --replicas=5
# (In the watch window, you'll see the EndpointSlice update with new endpoints)
```

---

## 17.4 kube-proxy Deep Dive

**kube-proxy** is the component responsible for implementing Services on each
node. It runs as a DaemonSet (or static Pod) and watches the API server for
Service and EndpointSlice changes. It supports three modes.

### 17.4.1 iptables Mode (Default)

In iptables mode, kube-proxy programs the Linux netfilter framework
(specifically the `nat` table) to intercept and redirect Service traffic.

#### NAT Chains and Probability-Based Load Balancing

For each Service, kube-proxy creates a chain of rules:

```
KUBE-SERVICES
  └─► KUBE-SVC-XXXXXXXX           (matches ClusterIP:port)
        ├─► KUBE-SEP-AAAA         (33% probability → DNAT to Pod 1)
        ├─► KUBE-SEP-BBBB         (50% of remaining = 33% overall → DNAT to Pod 2)
        └─► KUBE-SEP-CCCC         (100% of remaining = 33% overall → DNAT to Pod 3)
```

The probability math works like this for N backends:

- Rule 1: probability = `1/N` → matches 33.3%
- Rule 2: probability = `1/(N-1)` → of the 66.7% that passed rule 1, 50% match = 33.3%
- Rule 3: probability = `1/(N-2)` = 100% → catches the remaining 33.3%

This gives uniform distribution. iptables evaluates rules sequentially, so with
many backends the chain becomes long and slow — O(n) per packet.

#### Additional iptables Rules

kube-proxy also creates:

- **KUBE-MARK-MASQ** rules: Mark packets that need source NAT (masquerading)
  so that return traffic comes back to the correct node.
- **KUBE-NODEPORTS** chain: Matches traffic arriving on NodePort ports.
- **KUBE-POSTROUTING** chain: Performs SNAT on marked packets.
- **KUBE-FW-*** chains: For LoadBalancer Services with `loadBalancerSourceRanges`.

### 17.4.2 IPVS Mode — Kernel-Level L4 Load Balancing

**IPVS** (IP Virtual Server) is a transport-layer load balancer built into the
Linux kernel. Unlike iptables (which evaluates rules sequentially), IPVS uses
hash tables for O(1) lookup.

To enable IPVS mode:

```yaml
# kube-proxy ConfigMap
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-proxy
  namespace: kube-system
data:
  config.conf: |
    mode: "ipvs"
    ipvs:
      scheduler: "rr"    # round-robin
```

#### Load Balancing Algorithms

IPVS supports multiple scheduling algorithms:

| Algorithm | Flag | Description |
|---|---|---|
| Round Robin | `rr` | Distributes sequentially across backends |
| Weighted Round Robin | `wrr` | Like `rr` but respects backend weights |
| Least Connections | `lc` | Sends to the backend with fewest active connections |
| Weighted Least Connections | `wlc` | `lc` with weights |
| Source Hashing | `sh` | Hashes source IP for sticky routing |
| Destination Hashing | `dh` | Hashes destination IP |
| Shortest Expected Delay | `sed` | Minimizes expected delay based on connections + weight |
| Never Queue | `nq` | Sends to idle server; if none idle, uses `sed` |

With IPVS, the ClusterIP is actually bound to the `kube-ipvs0` dummy
interface, making it visible to the kernel's routing and IPVS subsystems:

```bash
$ ip addr show kube-ipvs0
5: kube-ipvs0: <BROADCAST,NOARP> mtu 1500
    inet 10.96.45.12/32 scope global kube-ipvs0
    inet 10.96.78.200/32 scope global kube-ipvs0
    ...

$ sudo ipvsadm -Ln
IP Virtual Server version 1.2.1
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  10.96.45.12:80 rr
  -> 10.244.1.5:8080              Masq    1      3          12
  -> 10.244.1.9:8080              Masq    1      2          15
  -> 10.244.3.2:8080              Masq    1      4          11
```

### 17.4.3 Why eBPF (Cilium) Is Replacing kube-proxy

Both iptables and IPVS have limitations:

| Limitation | iptables | IPVS |
|---|---|---|
| Connection tracking overhead | High | Moderate |
| Rule update latency | O(n) rebuild | Better, but still uses some iptables |
| Observability | Limited | Limited |
| Policy integration | Separate from networking | Separate |

**eBPF** (extended Berkeley Packet Filter) allows running sandboxed programs
*inside the kernel* at various hook points (XDP, TC, socket). **Cilium**, the
most popular eBPF-based CNI, can replace kube-proxy entirely:

```
Traditional:
  Packet → netfilter/iptables → DNAT → routing → Pod

eBPF (Cilium):
  Packet → eBPF program at socket/TC layer → direct redirect → Pod
```

Benefits of eBPF over kube-proxy:

- **Faster:** Short-circuits the kernel networking stack. Socket-level
  redirection avoids conntrack entirely for east-west traffic.
- **Scalable:** No iptables chains that grow with the number of Services.
- **Observable:** eBPF programs can emit rich telemetry (Hubble).
- **Unified:** Network policy, load balancing, and encryption in one data plane.

To use Cilium as a kube-proxy replacement:

```bash
$ helm install cilium cilium/cilium \
    --namespace kube-system \
    --set kubeProxyReplacement=true \
    --set k8sServiceHost=${API_SERVER_IP} \
    --set k8sServicePort=6443
```

### 17.4.4 Hands-on: Examining iptables Rules for a Service

```bash
# SSH into a node (or use kubectl debug node)
$ kubectl debug node/worker-1 -it --image=nicolaka/netshoot

# List all KUBE-SERVICES rules in the nat table
$ iptables -t nat -L KUBE-SERVICES -n --line-numbers | head -30

# Find the chain for our backend service (ClusterIP 10.96.45.12)
$ iptables -t nat -L KUBE-SERVICES -n | grep 10.96.45.12
KUBE-SVC-ABC123  tcp  --  0.0.0.0/0  10.96.45.12  /* default/backend:http */  tcp dpt:80

# Inspect the service chain (shows probability-based routing)
$ iptables -t nat -L KUBE-SVC-ABC123 -n
Chain KUBE-SVC-ABC123 (1 references)
target          prot opt source      destination
KUBE-SEP-AAA1   all  --  0.0.0.0/0  0.0.0.0/0   statistic mode random probability 0.33333
KUBE-SEP-BBB2   all  --  0.0.0.0/0  0.0.0.0/0   statistic mode random probability 0.50000
KUBE-SEP-CCC3   all  --  0.0.0.0/0  0.0.0.0/0

# Inspect an individual endpoint chain (shows the DNAT)
$ iptables -t nat -L KUBE-SEP-AAA1 -n
Chain KUBE-SEP-AAA1 (1 references)
target     prot opt source        destination
KUBE-MARK-MASQ  all  --  10.244.1.5   0.0.0.0/0
DNAT       tcp  --  0.0.0.0/0     0.0.0.0/0    tcp to:10.244.1.5:8080
```

### 17.4.5 Hands-on: ASCII Diagram of iptables Chain Flow

Here is the complete flow for a packet from a Pod on the same node hitting a
ClusterIP Service:

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        PACKET FLOW (same node)                          │
│                                                                          │
│  Client Pod sends to 10.96.45.12:80                                      │
│       │                                                                  │
│       ▼                                                                  │
│  OUTPUT chain (nat table)                                                │
│       │                                                                  │
│       ▼                                                                  │
│  KUBE-SERVICES                                                           │
│  ┌─ match dst 10.96.45.12:80 ──► KUBE-SVC-ABC123                       │
│  │                                    │                                  │
│  │                        ┌───────────┼───────────┐                      │
│  │                        ▼           ▼           ▼                      │
│  │                   KUBE-SEP-A  KUBE-SEP-B  KUBE-SEP-C                 │
│  │                   p=0.333    p=0.500    p=1.000                      │
│  │                        │           │           │                      │
│  │                   DNAT to     DNAT to     DNAT to                    │
│  │                  10.244.1.5  10.244.1.9  10.244.3.2                  │
│  │                        │           │           │                      │
│  │                        └───────────┼───────────┘                      │
│  │                                    ▼                                  │
│  │                           Packet now has:                             │
│  │                           src=10.244.x.x (client Pod)                │
│  │                           dst=10.244.y.y:8080 (backend Pod)          │
│  │                                    │                                  │
│  │                                    ▼                                  │
│  │                           POSTROUTING chain                           │
│  │                           KUBE-POSTROUTING                            │
│  │                           (MASQUERADE if marked)                      │
│  │                                    │                                  │
│  │                                    ▼                                  │
│  │                           Routed via CNI to backend Pod               │
│  └───────────────────────────────────────────────────────────────────── │
│                                                                          │
│  For EXTERNAL traffic (NodePort), the entry point is:                    │
│  PREROUTING → KUBE-SERVICES → KUBE-NODEPORTS → KUBE-SVC-*              │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 17.5 Service Discovery

Kubernetes provides two mechanisms for Pods to discover Services.

### 17.5.1 DNS-Based Discovery

Every Service gets a DNS entry managed by **CoreDNS** (or kube-dns in older
clusters). The naming convention:

```
<service-name>.<namespace>.svc.cluster.local
```

DNS record types:

| Service Type | DNS Record | Response |
|---|---|---|
| ClusterIP | A/AAAA | ClusterIP address |
| Headless (clusterIP: None) | A/AAAA | Set of Pod IP addresses |
| ExternalName | CNAME | External hostname |

**SRV records** are also created for named ports:

```
_http._tcp.backend.default.svc.cluster.local → SRV 0 33 8080 pod-abc.backend.default.svc.cluster.local
```

The Pod's `/etc/resolv.conf` is configured automatically:

```
nameserver 10.96.0.10          ← CoreDNS ClusterIP
search default.svc.cluster.local svc.cluster.local cluster.local
ndots: 5
```

The `ndots: 5` setting means that any name with fewer than 5 dots is first
tried with each search domain appended. So `backend` is resolved as:

1. `backend.default.svc.cluster.local` ← **matches!**
2. `backend.svc.cluster.local` (would try if #1 failed)
3. `backend.cluster.local` (would try if #2 failed)
4. `backend` (absolute lookup, would try if #3 failed)

> **Performance tip:** For external domains, use a trailing dot
> (`api.example.com.`) to skip the search domain expansion. Otherwise, every
> external DNS lookup generates 4 extra queries.

### 17.5.2 Environment Variable-Based Discovery

When a Pod starts, Kubernetes injects environment variables for every Service
that exists at that time in the **same namespace**:

```bash
# For a Service named "backend" on port 80
BACKEND_SERVICE_HOST=10.96.45.12
BACKEND_SERVICE_PORT=80
BACKEND_PORT=tcp://10.96.45.12:80
BACKEND_PORT_80_TCP=tcp://10.96.45.12:80
BACKEND_PORT_80_TCP_PROTO=tcp
BACKEND_PORT_80_TCP_PORT=80
BACKEND_PORT_80_TCP_ADDR=10.96.45.12
```

**Limitations of environment variable discovery:**

1. Services must exist **before** the Pod starts. If you create the Service
   after the Pod, the Pod will not have the variables.
2. Variable names follow a convention that can clash with user-defined env
   vars (dashes are converted to underscores, names are uppercased).
3. Cross-namespace services are not included.

For these reasons, **DNS is the preferred discovery mechanism** in modern
Kubernetes.

### 17.5.3 Hands-on: DNS Resolution from Within a Pod

```bash
# Start an interactive debug Pod
$ kubectl run dns-test --image=nicolaka/netshoot --rm -it -- bash

# A record for ClusterIP service
bash-5.1# dig backend.default.svc.cluster.local +short
10.96.45.12

# A records for headless service (returns all Pod IPs)
bash-5.1# dig backend-headless.default.svc.cluster.local +short
10.244.1.5
10.244.1.9
10.244.3.2

# SRV record for a named port
bash-5.1# dig _http._tcp.backend.default.svc.cluster.local SRV +short
0 33 8080 10-244-1-5.backend.default.svc.cluster.local.
0 33 8080 10-244-1-9.backend.default.svc.cluster.local.
0 33 8080 10-244-3-2.backend.default.svc.cluster.local.

# Cross-namespace lookup
bash-5.1# dig backend.production.svc.cluster.local +short
10.96.99.42

# Check resolv.conf
bash-5.1# cat /etc/resolv.conf
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

### 17.5.4 Hands-on: Environment Variable Discovery

```bash
$ kubectl run env-test --image=busybox --rm -it --restart=Never -- env | sort | grep -i backend
BACKEND_PORT=tcp://10.96.45.12:80
BACKEND_PORT_80_TCP=tcp://10.96.45.12:80
BACKEND_PORT_80_TCP_ADDR=10.96.45.12
BACKEND_PORT_80_TCP_PORT=80
BACKEND_PORT_80_TCP_PROTO=tcp
BACKEND_SERVICE_HOST=10.96.45.12
BACKEND_SERVICE_PORT=80
```

---

## 17.6 Session Affinity

By default, Services distribute traffic randomly (iptables probability) or via
round-robin (IPVS). Sometimes you need requests from the same client to reach
the same Pod — for example, if the backend uses in-memory sessions.

### 17.6.1 sessionAffinity: ClientIP

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-sticky
spec:
  selector:
    app: backend
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 1800    # 30 minutes (default: 10800 = 3 hours)
  ports:
    - port: 80
      targetPort: 8080
```

When `sessionAffinity: ClientIP` is set:

- **iptables mode:** kube-proxy adds the `--persist` flag (recent module) so
  that packets from the same source IP are DNAT'd to the same backend.
- **IPVS mode:** IPVS uses its built-in persistence feature with the
  configured timeout.

### 17.6.2 Hands-on: Configuring and Testing Session Affinity

```bash
$ kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: backend-sticky
spec:
  selector:
    app: backend
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 300
  ports:
    - port: 80
      targetPort: 8080
EOF

# Test from a debug Pod — all requests should hit the same backend
$ kubectl run curl-test --image=curlimages/curl --rm -it -- sh
$ for i in $(seq 1 10); do curl -s http://backend-sticky; done
Hello from backend    # Pod 1
Hello from backend    # Pod 1  (same!)
Hello from backend    # Pod 1  (same!)
Hello from backend    # Pod 1  (same!)
...

# Compare with the non-sticky service
$ for i in $(seq 1 10); do curl -s http://backend; done
Hello from backend    # Pod 2
Hello from backend    # Pod 1
Hello from backend    # Pod 3
Hello from backend    # Pod 1
...
```

> **Note:** Kubernetes only supports `ClientIP` session affinity, not
> cookie-based affinity. For cookie-based affinity, use an Ingress controller
> (e.g., NGINX Ingress with `nginx.ingress.kubernetes.io/affinity: cookie`).

---

## 17.7 Multi-Port Services

A single Service can expose multiple ports. Each port **must** have a unique
name when there is more than one.

### 17.7.1 Port Naming and targetPort Mapping

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app
spec:
  selector:
    app: web
  ports:
    - name: http             # required when multiple ports
      protocol: TCP
      port: 80               # port clients connect to
      targetPort: web-http   # can reference a named port on the container!
    - name: https
      protocol: TCP
      port: 443
      targetPort: web-https
    - name: metrics
      protocol: TCP
      port: 9090
      targetPort: 9090
```

Using **named targetPorts** (`web-http` instead of `8080`) allows different
Pods to listen on different port numbers while the Service maps them
correctly. This is especially useful during rolling updates where old and new
Pods may use different ports.

The container must define the named port:

```yaml
containers:
  - name: web
    ports:
      - name: web-http       # this name matches targetPort
        containerPort: 8080
      - name: web-https
        containerPort: 8443
```

### 17.7.2 Hands-on: Multi-Port Service

```yaml
# File: multi-port-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: multi-port-app
spec:
  selector:
    app: multi-port
  ports:
    - name: http
      port: 80
      targetPort: 8080
    - name: grpc
      port: 9000
      targetPort: 9000
    - name: metrics
      port: 9090
      targetPort: 9090
```

```bash
$ kubectl apply -f multi-port-svc.yaml
$ kubectl get svc multi-port-app
NAME              TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)                   AGE
multi-port-app    ClusterIP   10.96.55.100   <none>        80/TCP,9000/TCP,9090/TCP  5s

# DNS SRV records for each named port
$ dig _http._tcp.multi-port-app.default.svc.cluster.local SRV +short
0 100 8080 multi-port-app.default.svc.cluster.local.

$ dig _grpc._tcp.multi-port-app.default.svc.cluster.local SRV +short
0 100 9000 multi-port-app.default.svc.cluster.local.
```

---

## 17.8 External IPs

### 17.8.1 The externalIPs Field

You can manually assign external IP addresses to any Service type:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-external
spec:
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
  externalIPs:
    - 203.0.113.10           # an IP routable to a cluster node
```

kube-proxy will program rules so that traffic arriving at `203.0.113.10:80` on
any node is forwarded to the backend Pods.

> **Security warning:** `externalIPs` is a powerful feature that can be
> exploited to hijack traffic. Ensure RBAC policies restrict who can set this
> field (or use admission webhooks to validate it).

### 17.8.2 externalTrafficPolicy: Local — Preserving Source IP

When external traffic enters the cluster through a NodePort or LoadBalancer,
kube-proxy may SNAT the packet — replacing the source IP with the node's IP —
so that the return path is correct. This means backend Pods see the node IP,
not the real client IP.

Setting `externalTrafficPolicy: Local` changes this behavior:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-preserve-ip
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local   # ← preserve source IP
  selector:
    app: backend
  ports:
    - port: 80
      targetPort: 8080
```

With `externalTrafficPolicy: Local`:

1. kube-proxy only routes to Pods **on the local node** (no cross-node SNAT).
2. Nodes without backend Pods **fail the health check** and the load balancer
   stops sending traffic to them.
3. Source IP is preserved.

Trade-off: traffic distribution may be **uneven** if Pods are not spread
evenly across nodes. A node with 3 Pods gets the same amount of traffic as a
node with 1 Pod (the 3-Pod node spreads it further).

### 17.8.3 Hands-on: Demonstrating Source IP Preservation

Deploy an echo server that shows the source IP:

```yaml
# File: echoserver.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echoserver
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echoserver
  template:
    metadata:
      labels:
        app: echoserver
    spec:
      containers:
        - name: echoserver
          image: registry.k8s.io/echoserver:1.10
          ports:
            - containerPort: 8080
```

Create two Services — one with `Cluster` policy, one with `Local`:

```yaml
# File: traffic-policies.yaml
apiVersion: v1
kind: Service
metadata:
  name: echo-cluster
spec:
  type: NodePort
  externalTrafficPolicy: Cluster
  selector:
    app: echoserver
  ports:
    - port: 80
      targetPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: echo-local
spec:
  type: NodePort
  externalTrafficPolicy: Local
  selector:
    app: echoserver
  ports:
    - port: 80
      targetPort: 8080
```

```bash
$ kubectl apply -f echoserver.yaml -f traffic-policies.yaml

# Get the NodePort for each service
$ kubectl get svc echo-cluster echo-local
NAME            TYPE       CLUSTER-IP     PORT(S)        AGE
echo-cluster    NodePort   10.96.11.22    80:31001/TCP   5s
echo-local      NodePort   10.96.33.44    80:31002/TCP   5s

# Access from outside the cluster
$ curl http://${NODE_IP}:31001 2>/dev/null | grep "client_address"
client_address=10.244.0.1        ← SNAT'd! This is the node IP, not yours.

$ curl http://${NODE_IP}:31002 2>/dev/null | grep "client_address"
client_address=192.168.1.100     ← Your real client IP is preserved!
```

---

## 17.9 Service Mesh Concepts (Brief)

Services and kube-proxy handle basic L4 load balancing and service discovery.
But modern microservice architectures often need more:

- **Mutual TLS (mTLS):** Encrypt all service-to-service traffic and verify
  identity.
- **Observability:** Distributed tracing, request-level metrics, access logs.
- **Traffic management:** Canary deployments, traffic splitting, retries,
  circuit breaking.
- **Authorization policies:** Fine-grained "Service A can call Service B on
  method X."

### 17.9.1 How Service Meshes Work

A **service mesh** inserts a proxy alongside every Pod (as a sidecar container
or, more recently, as a per-node eBPF program). All traffic goes through the
proxy, which can enforce policies, collect telemetry, and manage TLS.

```
Without mesh:
  Pod A ──────────────────► Pod B

With sidecar mesh (Istio, Linkerd):
  Pod A ──► Envoy proxy ──► Envoy proxy ──► Pod B
            (sidecar A)     (sidecar B)

With eBPF mesh (Cilium):
  Pod A ──► eBPF program ──► Pod B
            (kernel-level)
```

### 17.9.2 Popular Service Meshes

| Mesh | Proxy | Approach |
|---|---|---|
| **Istio** | Envoy (sidecar) | Full-featured, complex, widely adopted |
| **Linkerd** | linkerd2-proxy (Rust sidecar) | Simpler, lower overhead |
| **Cilium** | eBPF (no sidecar) | Kernel-level, highest performance |
| **Consul Connect** | Envoy or built-in | HashiCorp ecosystem integration |

> **This is not a full service mesh chapter.** We include it here so you
> understand how Services — the basic building block — evolve into richer
> service-to-service communication in production. A dedicated chapter on
> service meshes would cover Istio VirtualService, DestinationRule, mTLS
> configuration, and observability with Kiali and Jaeger.

---

## 17.10 Hands-on: Go Service Discovery Client

Now let's build something real. This Go program runs *inside* the cluster and
uses client-go to discover service endpoints dynamically.

### 17.10.1 The Discovery Client

```go
// File: cmd/discover/main.go
//
// Package main implements a Kubernetes service discovery client that
// demonstrates how to find and connect to services programmatically.
//
// This program is designed to run inside a Kubernetes cluster (as a Pod).
// It uses the in-cluster configuration (service account token mounted at
// /var/run/secrets/kubernetes.io/serviceaccount) to authenticate with the
// API server.
//
// Usage:
//
//	kubectl run discover --image=myregistry/discover:latest \
//	    --env="TARGET_SERVICE=backend" \
//	    --env="TARGET_NAMESPACE=default"
package main

import (
	"context"
	"fmt"
	"io"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"

	discoveryv1 "k8s.io/api/discovery/v1"
	"k8s.io/apimachinery/pkg/watch"
)

// discoverEndpoints queries the Kubernetes API for EndpointSlices
// associated with the named service. It returns a slice of "host:port"
// strings representing all ready endpoints.
//
// Parameters:
//   - ctx:       Context for cancellation and timeouts.
//   - clientset: Kubernetes client authenticated via in-cluster config.
//   - namespace: The Kubernetes namespace where the service resides.
//   - service:   The name of the Kubernetes Service to discover.
//
// Returns:
//   - []string: A list of "ip:port" strings for all ready endpoints.
//   - error:    Non-nil if the API call fails.
//
// Example output: ["10.244.1.5:8080", "10.244.1.9:8080", "10.244.3.2:8080"]
func discoverEndpoints(
	ctx context.Context,
	clientset kubernetes.Interface,
	namespace, service string,
) ([]string, error) {
	// EndpointSlices are labeled with the service name they belong to.
	// We use a label selector to find all slices for our target service.
	labelSelector := fmt.Sprintf("kubernetes.io/service-name=%s", service)

	// List all EndpointSlices matching our label selector.
	sliceList, err := clientset.DiscoveryV1().EndpointSlices(namespace).List(
		ctx,
		metav1.ListOptions{LabelSelector: labelSelector},
	)
	if err != nil {
		return nil, fmt.Errorf("listing EndpointSlices for %s/%s: %w",
			namespace, service, err)
	}

	var endpoints []string

	// Iterate through each EndpointSlice. A service may have multiple
	// slices if it has more than ~100 endpoints (the default max per slice).
	for _, slice := range sliceList.Items {
		// Extract port numbers from the slice. All endpoints in a slice
		// share the same port definitions.
		var ports []int32
		for _, p := range slice.Ports {
			if p.Port != nil {
				ports = append(ports, *p.Port)
			}
		}

		// Iterate through each endpoint in this slice.
		for _, ep := range slice.Endpoints {
			// Only include endpoints that are ready to receive traffic.
			// The "ready" condition indicates the Pod has passed its
			// readiness probe and is not terminating.
			if ep.Conditions.Ready != nil && !*ep.Conditions.Ready {
				continue
			}

			// Each endpoint may have multiple addresses (e.g., dual-stack
			// IPv4 and IPv6). Combine each address with each port.
			for _, addr := range ep.Addresses {
				for _, port := range ports {
					endpoints = append(endpoints,
						fmt.Sprintf("%s:%d", addr, port))
				}
			}
		}
	}

	return endpoints, nil
}

// watchEndpoints sets up a watch on EndpointSlices for the given service
// and logs changes in real time. This demonstrates how to build a
// reactive service discovery client that responds to topology changes
// without polling.
//
// The watch will continue until the context is cancelled or an error occurs.
//
// Parameters:
//   - ctx:       Context for cancellation. Cancel to stop watching.
//   - clientset: Kubernetes client authenticated via in-cluster config.
//   - namespace: The Kubernetes namespace where the service resides.
//   - service:   The name of the Kubernetes Service to watch.
//
// Returns:
//   - error: Non-nil if the watch setup fails or encounters a fatal error.
func watchEndpoints(
	ctx context.Context,
	clientset kubernetes.Interface,
	namespace, service string,
) error {
	labelSelector := fmt.Sprintf("kubernetes.io/service-name=%s", service)

	// Start watching EndpointSlice resources. The watch returns a channel
	// of events (ADDED, MODIFIED, DELETED) that we can range over.
	watcher, err := clientset.DiscoveryV1().EndpointSlices(namespace).Watch(
		ctx,
		metav1.ListOptions{LabelSelector: labelSelector},
	)
	if err != nil {
		return fmt.Errorf("setting up EndpointSlice watch for %s/%s: %w",
			namespace, service, err)
	}
	defer watcher.Stop()

	fmt.Printf("[watch] Watching EndpointSlices for service %s/%s...\n",
		namespace, service)

	// Range over watch events. The channel closes when the watch expires
	// or the context is cancelled.
	for event := range watcher.ResultChan() {
		slice, ok := event.Object.(*discoveryv1.EndpointSlice)
		if !ok {
			continue
		}

		switch event.Type {
		case watch.Added:
			fmt.Printf("[watch] EndpointSlice ADDED: %s (%d endpoints)\n",
				slice.Name, len(slice.Endpoints))
		case watch.Modified:
			fmt.Printf("[watch] EndpointSlice MODIFIED: %s (%d endpoints)\n",
				slice.Name, len(slice.Endpoints))
			// In a real application, you would update your local
			// endpoint cache here and re-balance connections.
		case watch.Deleted:
			fmt.Printf("[watch] EndpointSlice DELETED: %s\n", slice.Name)
		}
	}

	return nil
}

// probeEndpoint sends an HTTP GET request to the given endpoint and
// returns the response body. This simulates a client connecting to a
// discovered service endpoint.
//
// Parameters:
//   - endpoint: The "host:port" string to connect to (e.g., "10.244.1.5:8080").
//   - timeout:  Maximum time to wait for a response.
//
// Returns:
//   - string: The response body as a string.
//   - error:  Non-nil if the HTTP request fails or the response is not 2xx.
func probeEndpoint(endpoint string, timeout time.Duration) (string, error) {
	client := &http.Client{Timeout: timeout}

	resp, err := client.Get(fmt.Sprintf("http://%s/", endpoint))
	if err != nil {
		return "", fmt.Errorf("probing endpoint %s: %w", endpoint, err)
	}
	defer resp.Body.Close()

	body, err := io.ReadAll(resp.Body)
	if err != nil {
		return "", fmt.Errorf("reading response from %s: %w", endpoint, err)
	}

	if resp.StatusCode < 200 || resp.StatusCode >= 300 {
		return "", fmt.Errorf("endpoint %s returned status %d: %s",
			endpoint, resp.StatusCode, string(body))
	}

	return string(body), nil
}

// getEnvOrDefault returns the value of the named environment variable,
// or the provided default if the variable is not set or is empty.
//
// Parameters:
//   - key:          The environment variable name (e.g., "TARGET_SERVICE").
//   - defaultValue: The value to return if the variable is not set.
//
// Returns:
//   - string: The environment variable value or the default.
func getEnvOrDefault(key, defaultValue string) string {
	if value := os.Getenv(key); value != "" {
		return value
	}
	return defaultValue
}

func main() {
	// Read configuration from environment variables. These would typically
	// be set in the Pod spec (from ConfigMaps, Secrets, or literals).
	targetService := getEnvOrDefault("TARGET_SERVICE", "backend")
	targetNamespace := getEnvOrDefault("TARGET_NAMESPACE", "default")

	fmt.Printf("Service Discovery Client\n")
	fmt.Printf("========================\n")
	fmt.Printf("Target: %s/%s\n\n", targetNamespace, targetService)

	// Build the in-cluster Kubernetes client configuration.
	// This reads the service account token from the mounted secret at
	// /var/run/secrets/kubernetes.io/serviceaccount/token and the CA
	// certificate from the same directory. The API server address is
	// discovered via the KUBERNETES_SERVICE_HOST and KUBERNETES_SERVICE_PORT
	// environment variables (injected by Kubernetes automatically).
	config, err := rest.InClusterConfig()
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error building in-cluster config: %v\n", err)
		fmt.Fprintf(os.Stderr, "Are you running inside a Kubernetes Pod?\n")
		os.Exit(1)
	}

	// Create the Kubernetes clientset. This is the main entry point for
	// interacting with the Kubernetes API.
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error creating Kubernetes client: %v\n", err)
		os.Exit(1)
	}

	// Set up a context that cancels on SIGINT or SIGTERM. This allows
	// graceful shutdown when the Pod is terminated.
	ctx, cancel := signal.NotifyContext(
		context.Background(), syscall.SIGINT, syscall.SIGTERM,
	)
	defer cancel()

	// --- Phase 1: Discover current endpoints ---
	fmt.Println("Phase 1: Discovering current endpoints...")
	endpoints, err := discoverEndpoints(ctx, clientset, targetNamespace, targetService)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Discovery failed: %v\n", err)
		os.Exit(1)
	}

	if len(endpoints) == 0 {
		fmt.Println("No ready endpoints found for the service.")
		fmt.Println("Check that the service exists and has ready Pods.")
		os.Exit(1)
	}

	fmt.Printf("Discovered %d endpoints:\n", len(endpoints))
	for i, ep := range endpoints {
		fmt.Printf("  [%d] %s\n", i, ep)
	}

	// --- Phase 2: Probe each endpoint ---
	fmt.Println("\nPhase 2: Probing endpoints...")
	for _, ep := range endpoints {
		body, err := probeEndpoint(ep, 5*time.Second)
		if err != nil {
			fmt.Printf("  ✗ %s: %v\n", ep, err)
		} else {
			fmt.Printf("  ✓ %s: %s\n", ep, body)
		}
	}

	// --- Phase 3: Watch for changes ---
	fmt.Println("\nPhase 3: Watching for endpoint changes (Ctrl+C to stop)...")
	if err := watchEndpoints(ctx, clientset, targetNamespace, targetService); err != nil {
		fmt.Fprintf(os.Stderr, "Watch error: %v\n", err)
	}

	fmt.Println("\nShutting down gracefully.")
}
```

### 17.10.2 RBAC Configuration

The service account running this Pod needs permission to list and watch
EndpointSlices:

```yaml
# File: rbac.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: discover-sa
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: endpoint-reader
  namespace: default
rules:
  - apiGroups: ["discovery.k8s.io"]
    resources: ["endpointslices"]
    verbs: ["list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: discover-endpoint-reader
  namespace: default
subjects:
  - kind: ServiceAccount
    name: discover-sa
    namespace: default
roleRef:
  kind: Role
  name: endpoint-reader
  apiGroup: rbac.authorization.k8s.io
```

### 17.10.3 Deployment

```yaml
# File: discover-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: discover
spec:
  replicas: 1
  selector:
    matchLabels:
      app: discover
  template:
    metadata:
      labels:
        app: discover
    spec:
      serviceAccountName: discover-sa
      containers:
        - name: discover
          image: myregistry/discover:latest
          env:
            - name: TARGET_SERVICE
              value: "backend"
            - name: TARGET_NAMESPACE
              value: "default"
```

### 17.10.4 DNS-Based Discovery Alternative

If you don't need the full power of the Kubernetes API, you can discover
endpoints using plain DNS lookups from Go:

```go
// File: cmd/discover-dns/main.go
//
// Package main demonstrates DNS-based service discovery in Kubernetes.
// This approach does not require client-go or RBAC configuration —
// it simply resolves DNS names that Kubernetes CoreDNS provides
// automatically.
//
// For a ClusterIP Service, DNS returns the virtual IP.
// For a headless Service, DNS returns all Pod IPs.
package main

import (
	"fmt"
	"net"
	"os"
	"time"
)

// resolveService performs DNS lookups for a Kubernetes service and
// prints the resolved addresses. It demonstrates three types of
// DNS queries commonly used for service discovery:
//
//  1. A/AAAA records — returns IP addresses.
//  2. SRV records — returns host:port pairs for named ports.
//  3. CNAME records — followed automatically by the resolver.
//
// Parameters:
//   - serviceDNS: The fully-qualified DNS name of the service
//     (e.g., "backend.default.svc.cluster.local").
//
// This function prints results to stdout and does not return errors
// for missing records (some services may not have SRV records).
func resolveService(serviceDNS string) {
	fmt.Printf("Resolving: %s\n", serviceDNS)
	fmt.Println("---")

	// A/AAAA record lookup — the most common type.
	// For ClusterIP services, this returns the virtual IP.
	// For headless services, this returns all Pod IPs.
	ips, err := net.LookupHost(serviceDNS)
	if err != nil {
		fmt.Printf("  A/AAAA lookup failed: %v\n", err)
	} else {
		fmt.Printf("  A/AAAA records (%d):\n", len(ips))
		for _, ip := range ips {
			fmt.Printf("    %s\n", ip)
		}
	}

	// SRV record lookup — provides port information.
	// Useful when you don't know which port the service uses.
	// SRV records use the format: _port-name._protocol.service.namespace.svc.cluster.local
	_, srvs, err := net.LookupSRV("", "", serviceDNS)
	if err != nil {
		fmt.Printf("  SRV lookup failed: %v\n", err)
	} else {
		fmt.Printf("  SRV records (%d):\n", len(srvs))
		for _, srv := range srvs {
			fmt.Printf("    %s:%d (priority=%d, weight=%d)\n",
				srv.Target, srv.Port, srv.Priority, srv.Weight)
		}
	}
}

func main() {
	// Default to looking up the "backend" service in the "default" namespace.
	service := "backend"
	namespace := "default"
	if len(os.Args) > 1 {
		service = os.Args[1]
	}
	if len(os.Args) > 2 {
		namespace = os.Args[2]
	}

	// Construct the fully-qualified DNS name following Kubernetes conventions:
	//   <service>.<namespace>.svc.cluster.local
	fqdn := fmt.Sprintf("%s.%s.svc.cluster.local", service, namespace)

	fmt.Println("=== Kubernetes DNS Service Discovery ===")
	fmt.Printf("Time: %s\n\n", time.Now().Format(time.RFC3339))

	// Resolve the ClusterIP service.
	resolveService(fqdn)

	// Also try the headless variant (if it exists).
	headlessFQDN := fmt.Sprintf("%s-headless.%s.svc.cluster.local",
		service, namespace)
	fmt.Println()
	resolveService(headlessFQDN)
}
```

---

## 17.11 Summary

| Concept | Key Takeaway |
|---|---|
| **Service** | Stable virtual IP + DNS for a set of Pods selected by labels |
| **ClusterIP** | Internal-only; iptables/IPVS DNAT to backend Pods |
| **NodePort** | Extends ClusterIP; opens a port on every node (30000-32767) |
| **LoadBalancer** | Extends NodePort; provisions cloud load balancer |
| **ExternalName** | DNS CNAME to external hostname; no proxying |
| **Headless** | No ClusterIP; DNS returns Pod IPs directly |
| **EndpointSlices** | Scalable replacement for Endpoints; split into 100-endpoint chunks |
| **kube-proxy** | Programs iptables/IPVS/eBPF on each node to implement Services |
| **DNS discovery** | `<svc>.<ns>.svc.cluster.local` — the primary discovery mechanism |
| **Env var discovery** | `{SVC}_SERVICE_HOST` / `{SVC}_SERVICE_PORT` — legacy, limited |
| **Session affinity** | `sessionAffinity: ClientIP` for sticky routing |
| **externalTrafficPolicy** | `Local` preserves source IP; `Cluster` distributes evenly |

---

## 17.12 Exercises

1. **ClusterIP tracing:** Create a 3-replica Deployment and ClusterIP Service.
   Use `tcpdump` in a debug Pod to capture packets and verify the DNAT
   translation from ClusterIP to Pod IP.

2. **NodePort and firewalls:** Create a NodePort Service. From outside the
   cluster, verify that the service is accessible on every node (even nodes
   without Pods). Then set `externalTrafficPolicy: Local` and verify that
   only nodes with Pods respond.

3. **Headless discovery:** Create a StatefulSet with a headless Service.
   Verify that each Pod gets a unique DNS name (`pod-0.svc.ns.svc.cluster.local`).
   Write a Go client that resolves all Pod IPs and round-robins requests.

4. **EndpointSlice scaling:** Create a Deployment with 200 replicas. Examine
   how many EndpointSlice objects are created. Scale down to 50 and watch the
   slices consolidate.

5. **kube-proxy mode comparison:** Configure kube-proxy in IPVS mode. Compare
   the iptables rules and `ipvsadm` output with the default iptables mode.
   Measure the time to update rules when scaling from 10 to 1,000 Pods.

6. **Service mesh introduction:** Install Linkerd on a test cluster. Deploy two
   services and verify that mTLS is automatically applied. Check the Linkerd
   dashboard for request-level metrics.

---

*Next chapter: ConfigMaps, Secrets & Environment Management — decoupling
configuration from code and managing sensitive data securely.*
