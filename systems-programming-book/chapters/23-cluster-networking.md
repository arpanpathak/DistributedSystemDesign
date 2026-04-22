# Chapter 23: Cluster Networking — Pod-to-Pod, Pod-to-Service

> *"The network is the computer."* — John Gage, Sun Microsystems

Kubernetes networking is often described as "simple in concept, complex in implementation."
The model itself is elegant: every Pod gets a real IP address, and every Pod can talk to
every other Pod without NAT. But beneath that simplicity lies a sophisticated stack of
virtual Ethernet devices, network namespaces, bridges, overlay tunnels, iptables chains,
IPVS rules, and eBPF programs — all cooperating to make the flat-network illusion real.

This chapter traces every packet, every hop, every translation. We will start inside a
single node, then cross node boundaries, then follow a packet from a Pod through a
Service and back. By the end, you will be able to sit at any node in a cluster and
explain exactly what happens to every byte that leaves a container.

---

## 23.1 The Kubernetes Networking Model — The Requirements

Before diving into implementations, we must understand what Kubernetes *requires* of any
networking solution. The Kubernetes networking model is defined by four fundamental rules:

### Rule 1: Every Pod Gets Its Own IP Address

Unlike Docker's default bridge networking where containers share the host's IP and use
port mapping, Kubernetes assigns a unique IP address to every Pod. This IP is visible
to the Pod itself — if a container inside the Pod runs `hostname -I`, it sees the same
IP that other Pods use to reach it.

```text
┌─────────────────────────────────────────────────────────┐
│  Docker Default Networking                               │
│                                                          │
│  Host IP: 192.168.1.10                                   │
│  Container A: 172.17.0.2  → exposed as 192.168.1.10:8080│
│  Container B: 172.17.0.3  → exposed as 192.168.1.10:8081│
│                                                          │
│  External clients must use host IP + mapped port         │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  Kubernetes Networking                                   │
│                                                          │
│  Pod A: 10.244.1.5:8080   → reachable as 10.244.1.5:8080│
│  Pod B: 10.244.2.9:8080   → reachable as 10.244.2.9:8080│
│                                                          │
│  Every Pod has its own IP — no port conflicts            │
└─────────────────────────────────────────────────────────┘
```

This eliminates the port-mapping headache entirely. Two Pods can both listen on port 80
because they have different IPs. Applications don't need to discover which port they were
mapped to — they just use their natural ports.

### Rule 2: Pods Communicate Without NAT

Any Pod can send packets to any other Pod using the destination Pod's IP address directly.
The network infrastructure must deliver those packets without Network Address Translation.
The source IP seen by the destination Pod is the real IP of the sending Pod.

This is a *strong* requirement. It means:

- Pod A on Node 1 sends a packet with `src=10.244.1.5, dst=10.244.2.9`
- Pod B on Node 2 receives a packet with `src=10.244.1.5, dst=10.244.2.9`
- No translation, no mangling, no rewriting of either address

### Rule 3: Agents on a Node Can Communicate with All Pods on That Node

System daemons (the kubelet, monitoring agents, log collectors) running directly on the
host must be able to reach all Pods scheduled on that node. This ensures the kubelet can
health-check Pods, and that node-level services can scrape metrics from any local Pod.

### Rule 4: The Flat Network

Together, these rules create what we call a "flat" network — a single L3 address space
where every Pod can reach every other Pod. There is no hierarchy, no zones (at the
network level), no multi-tier NAT. It looks like every Pod is on the same enormous
Ethernet segment.

```text
┌─────────────────── Flat Pod Network (10.244.0.0/16) ───────────────────┐
│                                                                         │
│  Node 1 (10.244.1.0/24)      Node 2 (10.244.2.0/24)                   │
│  ┌──────────┐ ┌──────────┐   ┌──────────┐ ┌──────────┐                │
│  │ Pod A    │ │ Pod B    │   │ Pod C    │ │ Pod D    │                │
│  │ 10.244   │ │ 10.244   │   │ 10.244   │ │ 10.244   │                │
│  │ .1.5     │ │ .1.6     │   │ .2.9     │ │ .2.10    │                │
│  └──────────┘ └──────────┘   └──────────┘ └──────────┘                │
│       │            │              │             │                       │
│       └────────────┴──────────────┴─────────────┘                      │
│              All can reach each other directly                          │
└─────────────────────────────────────────────────────────────────────────┘
```

### What the Model Does NOT Specify

The model is deliberately silent on *how* these requirements are met. It does not mandate:

- A specific encapsulation protocol (VXLAN, Geneve, WireGuard, none)
- A specific routing protocol (BGP, static routes, cloud provider routes)
- A specific implementation technology (iptables, IPVS, eBPF)
- A specific IP address management scheme

This is why the CNI (Container Network Interface) plugin ecosystem exists — each plugin
implements the flat network model differently, with different trade-offs.

### Pod CIDR Allocation

Each node is typically assigned a Pod CIDR — a subnet from the cluster's overall Pod
address range. The `--cluster-cidr` flag on the controller manager defines the overall
range, and `--node-cidr-mask-size` defines per-node subnet size.

```
Cluster CIDR:  10.244.0.0/16  (65,536 addresses)
Node 1 CIDR:   10.244.1.0/24  (256 addresses)
Node 2 CIDR:   10.244.2.0/24  (256 addresses)
Node 3 CIDR:   10.244.3.0/24  (256 addresses)
...
```

This allocation is managed by the Node IPAM controller in the kube-controller-manager.

---

## 23.2 Pod-to-Pod Communication — Same Node

Let's begin with the simplest case: two Pods on the same node. This involves Linux network
namespaces, virtual Ethernet pairs (veth), and a bridge device.

### Network Namespaces Revisited

Every Pod runs in its own network namespace (see Chapter 5 for namespace fundamentals).
A network namespace provides an isolated copy of the network stack:

- Its own interfaces (lo, eth0)
- Its own routing table
- Its own iptables rules
- Its own /proc/net

The host (root) namespace is where the physical NIC and the bridge live.

### The veth Pair

A veth (virtual Ethernet) pair is a tunnel: packets sent into one end appear at the
other. One end lives in the Pod's namespace (typically named `eth0`), and the other
end lives in the host namespace (named something like `vethXXXX`). They are created
as a pair and can never be separated — destroying one destroys both.

### The Bridge

The host namespace contains a bridge device (e.g., `cbr0`, `cni0`, `docker0`). All the
host-side veth endpoints are attached to this bridge. The bridge operates at L2 — it
learns MAC addresses and forwards frames between attached interfaces.

### Packet Flow: Same-Node Pod Communication

```text
  Pod A Network Namespace              Pod B Network Namespace
  ┌─────────────────────┐              ┌─────────────────────┐
  │                     │              │                     │
  │  ┌───────────────┐  │              │  ┌───────────────┐  │
  │  │  Application  │  │              │  │  Application  │  │
  │  │  10.244.1.5   │  │              │  │  10.244.1.6   │  │
  │  └───────┬───────┘  │              │  └───────▲───────┘  │
  │          │ ①        │              │          │ ⑥        │
  │  ┌───────▼───────┐  │              │  ┌───────┴───────┐  │
  │  │    eth0       │  │              │  │    eth0       │  │
  │  │  (veth inside)│  │              │  │  (veth inside)│  │
  │  └───────┬───────┘  │              │  └───────▲───────┘  │
  └──────────┼──────────┘              └──────────┼──────────┘
             │ ②                                  │ ⑤
  ═══════════╪════════════════════════════════════╪══════════
             │         Host Network Namespace      │
  ┌──────────▼────────────────────────────────────┴──────────┐
  │  veth_A                                        veth_B    │
  │  ┌────────┐                                ┌────────┐    │
  │  │vethXXX │                                │vethYYY │    │
  │  └───┬────┘                                └───▲────┘    │
  │      │ ③          ┌──────────────┐             │ ④       │
  │      └────────────►   Bridge     ├─────────────┘         │
  │                   │   (cni0)     │                       │
  │                   │              │                       │
  │                   └──────────────┘                       │
  └──────────────────────────────────────────────────────────┘

  Step-by-step:
  ① Application in Pod A sends packet to 10.244.1.6
  ② Packet traverses veth pair from Pod A namespace to host namespace
  ③ Host-side veth (vethXXX) delivers frame to bridge (cni0)
  ④ Bridge looks up MAC of 10.244.1.6, forwards to vethYYY
  ⑤ Packet traverses veth pair from host namespace to Pod B namespace
  ⑥ Application in Pod B receives the packet
```

### Detailed Step-by-Step

**Step 1: Application sends packet.**
Pod A's application calls `connect()` or `sendto()` targeting `10.244.1.6:8080`. The
kernel in Pod A's namespace consults the routing table:

```
# Inside Pod A's namespace:
$ ip route
default via 10.244.1.1 dev eth0
10.244.1.0/24 dev eth0 scope link src 10.244.1.5
```

The destination `10.244.1.6` matches the `10.244.1.0/24` route — it's on the local
subnet. The kernel needs to resolve the MAC address via ARP.

**Step 2: ARP resolution.**
Pod A sends an ARP request: "Who has 10.244.1.6?" This broadcast goes through the veth
pair to the bridge. The bridge floods the ARP request to all attached interfaces. Pod B's
veth receives it, and Pod B responds with its MAC address.

Some CNI plugins (like Calico) use proxy ARP instead of a bridge. The host responds to
ARP requests on behalf of all Pods, returning its own MAC. This means all traffic goes
through the host's routing stack instead of being switched at L2.

**Step 3: Frame sent to bridge.**
With the MAC address resolved, Pod A constructs an Ethernet frame and sends it out eth0.
The frame arrives at the host-side veth, which is attached to the bridge.

**Step 4: Bridge forwards frame.**
The bridge has learned Pod B's MAC address → vethYYY mapping from the ARP response.
It forwards the frame directly to vethYYY.

**Step 5-6: Frame delivered to Pod B.**
The frame exits through Pod B's eth0, the kernel delivers the packet up the stack, and
Pod B's application receives the data.

### Hands-On: Tracing Same-Node Pod Communication with tcpdump

Let's verify this packet flow on a real cluster.

**Setup: Create two Pods on the same node:**

```yaml
# same-node-pods.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-a
  labels:
    app: network-test
spec:
  nodeName: kind-worker    # Force same node
  containers:
  - name: nettools
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-b
spec:
  nodeName: kind-worker    # Force same node
  containers:
  - name: nettools
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
```

```bash
kubectl apply -f same-node-pods.yaml

# Get Pod IPs
kubectl get pods -o wide
# NAME    READY   STATUS    IP           NODE
# pod-a   1/1     Running   10.244.1.5   kind-worker
# pod-b   1/1     Running   10.244.1.6   kind-worker
```

**Trace from the node:**

```bash
# SSH into the node (for kind: docker exec -it kind-worker bash)
docker exec -it kind-worker bash

# Find the bridge
ip link show type bridge
# 5: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> ...

# Find veth pairs
bridge link show
# 6: vethXXX@if2: ... master cni0
# 7: vethYYY@if2: ... master cni0

# Capture traffic on the bridge
tcpdump -i cni0 -nn host 10.244.1.5 and host 10.244.1.6
```

**Send traffic from Pod A to Pod B:**

```bash
# In another terminal
kubectl exec pod-a -- ping -c 3 10.244.1.6
```

**Observe the tcpdump output:**

```
14:23:01.123456 ARP, Request who-has 10.244.1.6 tell 10.244.1.5, length 28
14:23:01.123567 ARP, Reply 10.244.1.6 is-at aa:bb:cc:dd:ee:02, length 28
14:23:01.123678 IP 10.244.1.5 > 10.244.1.6: ICMP echo request, id 1, seq 1
14:23:01.123789 IP 10.244.1.6 > 10.244.1.5: ICMP echo reply, id 1, seq 1
```

**Inspect veth mappings:**

```bash
# On the node, find which veth connects to which Pod
# Each veth in the host namespace has an ifindex that pairs with the Pod's eth0

# Find Pod A's eth0 ifindex
kubectl exec pod-a -- cat /sys/class/net/eth0/iflink
# 6

# On the node, find interface with index 6
ip link show | grep "^6:"
# 6: vethXXX@if2: ...

# This confirms vethXXX connects to Pod A
```

---

## 23.3 Pod-to-Pod Communication — Different Nodes

When Pods are on different nodes, packets must cross the physical (or virtual) network
infrastructure. There are three main approaches: overlay networks, routed networks, and
eBPF-based networking.

### 23.3.1 Overlay Networks (VXLAN)

Overlay networks encapsulate Pod packets inside regular IP packets. The most common
encapsulation is VXLAN (Virtual Extensible LAN), which wraps L2 frames in UDP packets.

**How VXLAN Works:**

Each node runs a VTEP (VXLAN Tunnel Endpoint) — a virtual interface that encapsulates
and decapsulates VXLAN packets. The VTEP has an IP address on the node's physical
network.

```
  Node 1 (192.168.1.10)                   Node 2 (192.168.1.11)
  ┌────────────────────┐                   ┌────────────────────┐
  │ Pod A (10.244.1.5) │                   │ Pod C (10.244.2.9) │
  │ ┌────────────────┐ │                   │ ┌────────────────┐ │
  │ │  Application   │ │                   │ │  Application   │ │
  │ └───────┬────────┘ │                   │ └───────▲────────┘ │
  │         │ ①        │                   │         │ ⑧        │
  │ ┌───────▼────────┐ │                   │ ┌───────┴────────┐ │
  │ │    eth0        │ │                   │ │    eth0        │ │
  │ └───────┬────────┘ │                   │ └───────▲────────┘ │
  │         │ ②        │                   │         │ ⑦        │
  │ ┌───────▼────────┐ │                   │ ┌───────┴────────┐ │
  │ │   Bridge(cni0) │ │                   │ │   Bridge(cni0) │ │
  │ └───────┬────────┘ │                   │ └───────▲────────┘ │
  │         │ ③        │                   │         │ ⑥        │
  │ ┌───────▼────────┐ │                   │ ┌───────┴────────┐ │
  │ │  VTEP          │ │                   │ │  VTEP          │ │
  │ │  (flannel.1)   │ │                   │ │  (flannel.1)   │ │
  │ │  Encapsulate   │ │                   │ │  Decapsulate   │ │
  │ └───────┬────────┘ │                   │ └───────▲────────┘ │
  │         │ ④        │                   │         │ ⑤        │
  │ ┌───────▼────────┐ │                   │ ┌───────┴────────┐ │
  │ │   eth0 (phys)  │ │                   │ │   eth0 (phys)  │ │
  │ │  192.168.1.10  │ │                   │ │  192.168.1.11  │ │
  │ └───────┬────────┘ │                   │ └───────▲────────┘ │
  └─────────┼──────────┘                   └─────────┼──────────┘
            │              Physical Network           │
            └─────────────────────────────────────────┘

  ① Pod A sends packet: src=10.244.1.5 dst=10.244.2.9
  ② Packet goes through veth to bridge
  ③ Bridge routes to VTEP (flannel.1); packet is VXLAN-encapsulated:
     Outer: src=192.168.1.10 dst=192.168.1.11 UDP port 4789
     Inner: src=10.244.1.5   dst=10.244.2.9
  ④ Encapsulated packet sent on physical network
  ⑤ Node 2 receives UDP packet on port 4789
  ⑥ VTEP decapsulates, extracts inner packet
  ⑦ Inner packet delivered to bridge
  ⑧ Bridge forwards to Pod C via veth pair
```

**The VXLAN Header:**

```
  ┌──────────────────────────────────────────────────────┐
  │ Outer Ethernet Header                                │
  │  src MAC: Node1-MAC   dst MAC: Node2-MAC             │
  ├──────────────────────────────────────────────────────┤
  │ Outer IP Header                                      │
  │  src: 192.168.1.10    dst: 192.168.1.11              │
  │  protocol: UDP                                       │
  ├──────────────────────────────────────────────────────┤
  │ Outer UDP Header                                     │
  │  src port: <hash>     dst port: 4789 (VXLAN)         │
  ├──────────────────────────────────────────────────────┤
  │ VXLAN Header (8 bytes)                               │
  │  VNI: 1 (VXLAN Network Identifier)                   │
  ├──────────────────────────────────────────────────────┤
  │ Inner Ethernet Header                                │
  │  src MAC: PodA-MAC    dst MAC: PodC-MAC              │
  ├──────────────────────────────────────────────────────┤
  │ Inner IP Header                                      │
  │  src: 10.244.1.5      dst: 10.244.2.9                │
  ├──────────────────────────────────────────────────────┤
  │ Inner TCP/UDP Header + Payload                       │
  │  src port: 54321      dst port: 8080                 │
  │  [HTTP request data...]                              │
  └──────────────────────────────────────────────────────┘
```

**VXLAN overhead:** 50 bytes (14 outer Ethernet + 20 outer IP + 8 UDP + 8 VXLAN).
This reduces the effective MTU from 1500 to 1450, which is why many CNI plugins
set the Pod MTU to 1450.

**When to use overlay networks:**
- When the underlay network is not under your control (cloud VPCs with limited routing)
- When you need multi-tenancy with network isolation (VNI per tenant)
- When nodes span multiple L2 domains

**CNI plugins using VXLAN:** Flannel (default), Calico (optional), Weave Net, Cilium (optional)

### 23.3.2 Routed Networks (BGP)

Routed networks avoid encapsulation entirely. Instead, each node advertises its Pod CIDR
to the network using BGP (Border Gateway Protocol). Routers in the physical network learn
these routes and forward Pod traffic natively.

```
  Node 1 (192.168.1.10)                   Node 2 (192.168.1.11)
  ┌────────────────────┐                   ┌────────────────────┐
  │ Pod A (10.244.1.5) │                   │ Pod C (10.244.2.9) │
  │ ┌────────────────┐ │                   │ ┌────────────────┐ │
  │ │  Application   │ │                   │ │  Application   │ │
  │ └───────┬────────┘ │                   │ └───────▲────────┘ │
  │         │ ①        │                   │         │ ⑤        │
  │ ┌───────▼────────┐ │                   │ ┌───────┴────────┐ │
  │ │    eth0 (veth) │ │                   │ │    eth0 (veth) │ │
  │ └───────┬────────┘ │                   │ └───────▲────────┘ │
  │         │ ②        │                   │         │ ④        │
  │ ┌───────▼────────┐ │                   │ ┌───────┴────────┐ │
  │ │ Routing Table  │ │                   │ │ Routing Table  │ │
  │ │ 10.244.2.0/24  │ │                   │ │ 10.244.1.0/24  │ │
  │ │ via 192.168.   │ │                   │ │ via 192.168.   │ │
  │ │    1.11        │ │                   │ │    1.10        │ │
  │ └───────┬────────┘ │                   │ └───────▲────────┘ │
  │         │ ③        │                   │         │ ③        │
  │ ┌───────▼────────┐ │                   │ ┌───────┴────────┐ │
  │ │  eth0 (phys)   │ │                   │ │  eth0 (phys)   │ │
  │ └───────┬────────┘ │                   │ └───────▲────────┘ │
  └─────────┼──────────┘                   └─────────┼──────────┘
            │              Physical Network           │
            └─────────────────────────────────────────┘
            No encapsulation — native IP routing!

  ① Pod A sends packet: src=10.244.1.5 dst=10.244.2.9
  ② Routing table on Node 1: 10.244.2.0/24 via 192.168.1.11
  ③ Packet forwarded natively through physical network
     (The packet on the wire: src=10.244.1.5 dst=10.244.2.9)
  ④ Routing table on Node 2: 10.244.2.0/24 is local
  ⑤ Packet delivered to Pod C via veth
```

**Advantages of routed networking:**
- No encapsulation overhead (full 1500 MTU)
- Better performance (no encap/decap CPU cost)
- Standard network troubleshooting tools work (tcpdump sees real Pod IPs)
- Hardware offload capabilities

**Requirements:**
- The physical network must accept BGP routes from nodes
- Or: cloud provider must support custom routes (AWS VPC routes, GCE routes)
- Or: all nodes on the same L2 segment (direct routes work without BGP)

**Calico's approach:**
Calico runs a BGP agent (BIRD) on each node. Each node peers with its neighbors (or with
route reflectors for large clusters) and advertises its Pod CIDR. Calico also uses
iptables or eBPF for network policy enforcement.

```bash
# On a Calico node, see the BGP-learned routes:
ip route | grep bird
# 10.244.1.0/24 via 192.168.1.10 dev eth0 proto bird
# 10.244.2.0/24 via 192.168.1.11 dev eth0 proto bird
# 10.244.3.0/24 via 192.168.1.12 dev eth0 proto bird
```

### 23.3.3 eBPF-Based Networking (Cilium)

Cilium takes a radically different approach: it attaches eBPF programs directly to
network interfaces, bypassing much of the traditional Linux networking stack.

```
  Node 1                                   Node 2
  ┌────────────────────┐                   ┌────────────────────┐
  │ Pod A (10.244.1.5) │                   │ Pod C (10.244.2.9) │
  │ ┌────────────────┐ │                   │ ┌────────────────┐ │
  │ │  Application   │ │                   │ │  Application   │ │
  │ └───────┬────────┘ │                   │ └───────▲────────┘ │
  │         │ ①        │                   │         │ ⑤        │
  │ ┌───────▼────────┐ │                   │ ┌───────┴────────┐ │
  │ │    eth0        │ │                   │ │    eth0        │ │
  │ │  eBPF tc prog  │◄── Packet handling  │ │  eBPF tc prog  │ │
  │ └───────┬────────┘   at tc layer       │ └───────▲────────┘ │
  │         │ ②                            │         │ ④        │
  │ ┌───────▼────────┐                     │ ┌───────┴────────┐ │
  │ │  lxcXXXX       │  No bridge!         │ │  lxcYYYY       │ │
  │ │  eBPF tc prog  │  eBPF routes        │ │  eBPF tc prog  │ │
  │ └───────┬────────┘  directly           │ └───────▲────────┘ │
  │         │ ③                            │         │ ③        │
  │ ┌───────▼────────┐                     │ ┌───────┴────────┐ │
  │ │  eth0 (phys)   │                     │ │  eth0 (phys)   │ │
  │ │  eBPF XDP prog │                     │ │  eBPF XDP prog │ │
  │ └───────┬────────┘                     │ └───────▲────────┘ │
  └─────────┼──────────┘                   └─────────┼──────────┘
            │                                        │
            └────────────────────────────────────────┘

  Key difference: eBPF programs at each hop make forwarding
  decisions. No bridge, no iptables, no conntrack (optional).
```

**Why eBPF networking is faster:**
- Packets are redirected at the tc (traffic control) layer, never going through the
  bridge or iptables
- `bpf_redirect_peer()` can send packets directly from one veth to another without
  going through the host namespace stack
- XDP (eXpress Data Path) programs on the physical NIC can process packets before
  they even enter the kernel's networking stack
- Connection tracking can be done in eBPF maps instead of the kernel's conntrack table

**CNI plugins using eBPF:** Cilium (primary), Calico (eBPF dataplane option)

### Hands-On: Tracing Cross-Node Pod Communication

```yaml
# cross-node-pods.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-node1
spec:
  nodeName: kind-worker
  containers:
  - name: nettools
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-node2
spec:
  nodeName: kind-worker2
  containers:
  - name: nettools
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
```

```bash
kubectl apply -f cross-node-pods.yaml
kubectl get pods -o wide

# Identify Pod IPs and Nodes
# pod-node1: 10.244.1.5 on kind-worker
# pod-node2: 10.244.2.9 on kind-worker2

# Trace on the source node (kind-worker)
docker exec -it kind-worker bash
tcpdump -i eth0 -nn host 10.244.2.9

# In another terminal, send traffic
kubectl exec pod-node1 -- ping -c 3 10.244.2.9

# With VXLAN overlay, you'll see encapsulated UDP packets:
# 192.168.1.10.46372 > 192.168.1.11.4789: VXLAN, ...
#   10.244.1.5 > 10.244.2.9: ICMP echo request

# With BGP routing, you'll see native Pod IP packets:
# 10.244.1.5 > 10.244.2.9: ICMP echo request

# Check routing on the node
ip route show
# 10.244.1.0/24 dev cni0 proto kernel scope link src 10.244.1.1
# 10.244.2.0/24 via 192.168.1.11 dev flannel.1  (VXLAN)
# -- or --
# 10.244.2.0/24 via 192.168.1.11 dev eth0 proto bird  (BGP)
```

---

## 23.4 Pod-to-Service Communication (kube-proxy)

Services provide stable endpoints for a set of Pods. A Service gets a ClusterIP — a
virtual IP address that doesn't correspond to any real interface. The translation from
ClusterIP to actual Pod IPs is handled by kube-proxy (or its replacement).

### Why Services Exist

Pods are ephemeral. They come and go — deployments scale up and down, rolling updates
replace Pods, nodes fail and Pods are rescheduled. A Pod's IP address is not stable.

Services solve this by providing:
1. A stable IP address (ClusterIP) that lives for the lifetime of the Service
2. A DNS name (`my-service.my-namespace.svc.cluster.local`)
3. Load balancing across the set of backend Pods (Endpoints)

```
  Service: my-service
  ClusterIP: 10.96.45.123

  Endpoints (selected by label selector):
  ┌─────────────────────────────────────────────┐
  │  Pod A: 10.244.1.5:8080                     │
  │  Pod B: 10.244.2.9:8080                     │
  │  Pod C: 10.244.3.4:8080                     │
  └─────────────────────────────────────────────┘

  Client Pod sends to 10.96.45.123:80
  → Translated (DNAT) to 10.244.2.9:8080 (random backend)
  → Response comes back from 10.244.2.9:8080
  → Un-DNAT'd back to appear as from 10.96.45.123:80
```

### How kube-proxy Learns About Services

kube-proxy runs on every node and watches the Kubernetes API server for Service and
Endpoints (or EndpointSlice) objects. When a Service or its Endpoints change, kube-proxy
updates the data plane rules on that node.

```
  ┌──────────────┐     watch Services,     ┌──────────────────┐
  │  API Server  │────EndpointSlices────►  │  kube-proxy      │
  │              │                          │  (on each node)  │
  └──────────────┘                          └────────┬─────────┘
                                                     │
                                    Programs data plane:
                                    ┌────────────────┴───────────┐
                                    │  iptables │ IPVS │ eBPF    │
                                    └────────────────────────────┘
```

### 23.4.1 iptables Mode

iptables mode is the default kube-proxy mode. It programs the Linux kernel's netfilter
framework using iptables rules.

#### How kube-proxy Programs iptables Rules

For each Service, kube-proxy creates a chain of iptables rules that intercept packets
destined for the Service's ClusterIP and DNAT them to one of the backend Pod IPs.

The chain structure:

```
  PREROUTING chain (for traffic from other Pods/external)
  OUTPUT chain (for traffic from local processes)
       │
       ▼
  KUBE-SERVICES chain
  (matches on Service ClusterIP:port)
       │
       ▼
  KUBE-SVC-XXXXXXXXXXXXXXXX chain
  (one per Service — selects a backend)
       │
       ├──► KUBE-SEP-AAAAAAAAAAAAAAAA (Pod A endpoint)
       │    Action: DNAT to 10.244.1.5:8080
       │
       ├──► KUBE-SEP-BBBBBBBBBBBBBBBB (Pod B endpoint)
       │    Action: DNAT to 10.244.2.9:8080
       │
       └──► KUBE-SEP-CCCCCCCCCCCCCCCC (Pod C endpoint)
            Action: DNAT to 10.244.3.4:8080
```

#### Probability-Based Load Balancing

How does the `KUBE-SVC-xxx` chain select which `KUBE-SEP-xxx` to use? It uses the
iptables `statistic` module with `--mode random --probability`:

```
Chain KUBE-SVC-XXXXXXXXXXXXXXXX (1 references)
 pkts bytes target     prot opt in   out   source     destination
    0     0 KUBE-SEP-AAA  all  --  *    *   0.0.0.0/0  0.0.0.0/0
                          statistic mode random probability 0.33333
    0     0 KUBE-SEP-BBB  all  --  *    *   0.0.0.0/0  0.0.0.0/0
                          statistic mode random probability 0.50000
    0     0 KUBE-SEP-CCC  all  --  *    *   0.0.0.0/0  0.0.0.0/0
```

**Why the probabilities are 0.333, 0.500, 1.000 (not 0.333, 0.333, 0.333):**

iptables rules are evaluated sequentially. If a rule doesn't match, the next rule is tried.

- Rule 1: probability 1/3 = 0.333 → selects Pod A 33.3% of the time
- Rule 2: probability 1/2 = 0.500 → of the remaining 66.7%, selects Pod B 50% = 33.3%
- Rule 3: no probability (implicit 1.0) → selects Pod C (the remaining 33.3%)

This is the conditional probability trick: `P(B) = 1/(N-position)` where position starts
at 0. For 3 backends: 1/3, 1/2, 1/1.

#### Connection Tracking (conntrack)

The DNAT rewriting happens once — for the first packet of a connection (the SYN packet).
The Linux conntrack module remembers this translation in a connection tracking table:

```
conntrack -L | grep 10.96.45.123
tcp  6 117 TIME_WAIT src=10.244.1.20 dst=10.96.45.123 sport=54321 dport=80
                     src=10.244.2.9  dst=10.244.1.20  sport=8080  dport=54321
```

This entry says:
- Original: client 10.244.1.20:54321 → Service 10.96.45.123:80
- Reply: Pod 10.244.2.9:8080 → client 10.244.1.20:54321

Subsequent packets in the same connection (SYN-ACK, ACK, data, FIN) are handled by
conntrack — they don't re-traverse the iptables DNAT rules. Return packets from the
backend Pod are un-DNAT'd: the source address 10.244.2.9 is rewritten back to
10.96.45.123, so the client never sees the real backend Pod IP.

#### Full iptables Chain Walkthrough

Let's trace a packet from Pod X (10.244.1.20) to Service `my-service` (10.96.45.123:80):

```
  Pod X sends packet:
  src=10.244.1.20:54321 dst=10.96.45.123:80
       │
       ▼
  ┌─────────────────────────────────────────┐
  │ PREROUTING chain (nat table)            │
  │ -j KUBE-SERVICES                        │ ← All traffic jumps here
  └──────────────────┬──────────────────────┘
                     ▼
  ┌─────────────────────────────────────────┐
  │ KUBE-SERVICES chain                     │
  │ -d 10.96.45.123/32 -p tcp --dport 80   │
  │ -j KUBE-SVC-XXXXXXXXXXXXXXXX            │ ← Match! Jump to SVC chain
  └──────────────────┬──────────────────────┘
                     ▼
  ┌─────────────────────────────────────────┐
  │ KUBE-SVC-XXXXXXXXXXXXXXXX               │
  │ -m statistic --mode random --prob 0.333 │
  │ -j KUBE-SEP-AAAAAAAAAAAAAAAA            │ ← 33% chance: Pod A
  │                                         │
  │ -m statistic --mode random --prob 0.500 │
  │ -j KUBE-SEP-BBBBBBBBBBBBBBBB            │ ← 33% chance: Pod B
  │                                         │
  │ -j KUBE-SEP-CCCCCCCCCCCCCCCC            │ ← 33% chance: Pod C
  └──────────────────┬──────────────────────┘
                     ▼ (say Pod B is selected)
  ┌─────────────────────────────────────────┐
  │ KUBE-SEP-BBBBBBBBBBBBBBBB               │
  │ -p tcp                                  │
  │ -j DNAT --to-destination 10.244.2.9:8080│
  └──────────────────┬──────────────────────┘
                     ▼
  Packet is now:
  src=10.244.1.20:54321 dst=10.244.2.9:8080
  (conntrack entry created)
       │
       ▼
  Normal routing delivers packet to 10.244.2.9
  (same-node or cross-node, as described above)
       │
       ▼
  Pod B responds:
  src=10.244.2.9:8080 dst=10.244.1.20:54321
       │
       ▼
  conntrack un-DNATs the response:
  src=10.96.45.123:80 dst=10.244.1.20:54321
       │
       ▼
  Pod X receives response from 10.96.45.123:80 ✓
```

#### Hands-On: Examining iptables Rules for a Service

```bash
# Create a Service with 3 replicas
kubectl create deployment web --image=nginx --replicas=3
kubectl expose deployment web --port=80

# Get the ClusterIP
kubectl get svc web
# NAME   TYPE        CLUSTER-IP     PORT(S)
# web    ClusterIP   10.96.45.123   80/TCP

# SSH into any node
docker exec -it kind-worker bash

# View the KUBE-SERVICES chain
iptables -t nat -L KUBE-SERVICES -n | grep 10.96.45.123
# -d 10.96.45.123/32 -p tcp --dport 80 -j KUBE-SVC-XXXXXXXXXXXXXXXX

# View the KUBE-SVC chain (load balancing rules)
iptables -t nat -L KUBE-SVC-XXXXXXXXXXXXXXXX -n
# Chain KUBE-SVC-XXXXXXXXXXXXXXXX (1 references)
# target                 prot  source    destination
# KUBE-SEP-AAA           all   0.0.0.0/0 0.0.0.0/0  statistic mode random probability 0.33333
# KUBE-SEP-BBB           all   0.0.0.0/0 0.0.0.0/0  statistic mode random probability 0.50000
# KUBE-SEP-CCC           all   0.0.0.0/0 0.0.0.0/0

# View an individual endpoint chain
iptables -t nat -L KUBE-SEP-AAA -n
# Chain KUBE-SEP-AAA (1 references)
# target   prot  source         destination
# DNAT     tcp   0.0.0.0/0      0.0.0.0/0   tcp to:10.244.1.5:80

# Count total iptables rules (grows with number of Services)
iptables -t nat -L -n | wc -l
```

### 23.4.2 IPVS Mode

IPVS (IP Virtual Server) is a kernel-level L4 load balancer built into Linux. kube-proxy
can use IPVS instead of iptables for Service load balancing.

#### How IPVS Mode Works

In IPVS mode, kube-proxy creates an IPVS virtual server for each Service ClusterIP:port
combination. Backend Pods are registered as "real servers" behind the virtual server.

```
  ┌──────────────────────────────────────────────────────┐
  │  IPVS Virtual Server                                  │
  │  VIP: 10.96.45.123:80   Scheduler: rr (round-robin)  │
  │                                                       │
  │  Real Servers:                                        │
  │  ├── 10.244.1.5:8080   Weight: 1  ActiveConn: 3      │
  │  ├── 10.244.2.9:8080   Weight: 1  ActiveConn: 2      │
  │  └── 10.244.3.4:8080   Weight: 1  ActiveConn: 4      │
  └──────────────────────────────────────────────────────┘
```

IPVS hooks into netfilter at the same point as DNAT rules, but it uses a hash table
internally for O(1) lookup instead of linear rule scanning.

#### The kube-ipvs0 Dummy Interface

IPVS needs the virtual IP to be present on a local interface so the kernel accepts
packets destined for it. kube-proxy creates a dummy interface `kube-ipvs0` and adds all
Service ClusterIPs as addresses on this interface:

```bash
ip addr show kube-ipvs0
# 4: kube-ipvs0: <BROADCAST,NOARP>
#     inet 10.96.45.123/32 scope global kube-ipvs0
#     inet 10.96.0.1/32 scope global kube-ipvs0
#     inet 10.96.100.50/32 scope global kube-ipvs0
#     ...
```

#### Load Balancing Algorithms

IPVS supports multiple scheduling algorithms:

| Algorithm | Flag | Description |
|-----------|------|-------------|
| Round Robin | `rr` | Distributes connections in rotation |
| Least Connection | `lc` | Sends to server with fewest active connections |
| Destination Hashing | `dh` | Hash of destination IP selects server |
| Source Hashing | `sh` | Hash of source IP selects server (session affinity) |
| Shortest Expected Delay | `sed` | Minimizes expected delay |
| Never Queue | `nq` | Two-speed version of SED |
| Weighted Round Robin | `wrr` | Round robin with weight support |
| Weighted Least Connection | `wlc` | Least connection with weight support |

kube-proxy defaults to round-robin (`rr`). You can set it with:

```
--ipvs-scheduler=lc  # least-connection
```

#### Why IPVS Scales Better Than iptables

With iptables, every packet to a Service must traverse the iptables rules linearly until
a match is found. For a cluster with 10,000 Services:

```
  iptables: O(n) per packet
  ┌──────────────────────────────────────────────────────┐
  │ KUBE-SERVICES chain: 10,000+ rules                   │
  │ Rule 1: -d 10.96.0.1/32 -j KUBE-SVC-AAA  → miss     │
  │ Rule 2: -d 10.96.0.2/32 -j KUBE-SVC-BBB  → miss     │
  │ Rule 3: -d 10.96.0.3/32 -j KUBE-SVC-CCC  → miss     │
  │ ...                                                   │
  │ Rule 5000: -d 10.96.45.123 -j KUBE-SVC-XXX → MATCH!  │
  │ (5000 rule comparisons for this one packet!)          │
  └──────────────────────────────────────────────────────┘

  IPVS: O(1) per packet
  ┌──────────────────────────────────────────────────────┐
  │ IPVS hash table                                       │
  │ hash(10.96.45.123:80) → bucket → virtual server       │
  │ (1 hash lookup, regardless of number of services!)    │
  └──────────────────────────────────────────────────────┘
```

Additionally, iptables rule updates are expensive — the entire ruleset must be rewritten
atomically. IPVS allows incremental updates.

**Benchmarks (approximate):**

| Services | iptables rule count | iptables latency | IPVS latency |
|----------|--------------------:|----------------:|-------------:|
| 100      | ~800               | ~0.1 ms         | ~0.05 ms     |
| 1,000    | ~8,000             | ~1 ms           | ~0.05 ms     |
| 10,000   | ~80,000            | ~10 ms          | ~0.05 ms     |
| 50,000   | ~400,000           | ~50 ms          | ~0.05 ms     |

#### Hands-On: Examining IPVS Rules with ipvsadm

```bash
# Enable IPVS mode in kube-proxy
# Edit kube-proxy ConfigMap:
kubectl edit configmap kube-proxy -n kube-system
# Set mode: "ipvs"
# Restart kube-proxy pods:
kubectl rollout restart daemonset kube-proxy -n kube-system

# On a node, install ipvsadm
apt-get install -y ipvsadm

# List all virtual servers
ipvsadm -Ln
# IP Virtual Server version 1.2.1 (size=4096)
# Prot LocalAddress:Port Scheduler Flags
#   -> RemoteAddress:Port    Forward Weight ActiveConn InActConn
# TCP  10.96.45.123:80 rr
#   -> 10.244.1.5:8080       Masq    1      3          0
#   -> 10.244.2.9:8080       Masq    1      2          0
#   -> 10.244.3.4:8080       Masq    1      4          0

# Show statistics
ipvsadm -Ln --stats
# Prot LocalAddress:Port  Conns   InPkts  OutPkts  InBytes  OutBytes
# TCP  10.96.45.123:80      150     4500    4500     540000   2700000

# Show connection table
ipvsadm -Lnc
# Pro  ExpireTime  State       Source:Port     Virtual:Port    Dest:Port
# TCP  00:59       ESTABLISHED 10.244.1.20:54321 10.96.45.123:80 10.244.2.9:8080

# Check the kube-ipvs0 interface
ip addr show kube-ipvs0
```

### 23.4.3 eBPF Mode (Cilium — Replacing kube-proxy)

Cilium can replace kube-proxy entirely, handling all Service load balancing in eBPF
programs attached to Pod interfaces and the physical NIC.

#### How Cilium Handles Services

Cilium implements Service load balancing at two levels:

**1. Socket-level load balancing (connect-time):**

When a Pod calls `connect()` to a Service ClusterIP, Cilium's eBPF program intercepts
the system call and rewrites the destination address *before the packet is even created*:

```
  Traditional kube-proxy (packet level):
  ┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  connect  │────►│  create  │────►│  iptables │────►│  send to │
  │  (SVC IP) │     │  packet  │     │  DNAT to  │     │  Pod IP  │
  │           │     │  SVC IP  │     │  Pod IP   │     │          │
  └──────────┘     └──────────┘     └──────────┘     └──────────┘

  Cilium eBPF (socket level):
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │  connect  │────►│  eBPF    │────►│  create  │
  │  (SVC IP) │     │  rewrites│     │  packet  │
  │           │     │  to Pod  │     │  Pod IP  │
  │           │     │  IP      │     │  directly│
  └──────────┘     └──────────┘     └──────────┘

  No DNAT! No conntrack entry! No return-path un-DNAT!
```

**2. TC (traffic control) level load balancing:**

For traffic that doesn't originate from a local socket (e.g., traffic entering from
another node), eBPF programs at the tc layer handle the DNAT. This is still faster than
iptables because it uses eBPF maps (hash tables) instead of linear rule scanning.

#### Why eBPF Mode is Faster

- **No conntrack overhead:** Socket-level LB means no DNAT → no conntrack entries
- **O(1) lookups:** eBPF maps (hash tables) for service→endpoint mapping
- **Short-circuit:** Same-node Pod-to-Service traffic never leaves the stack
- **No rule sync:** No iptables rule rewriting when endpoints change — just update eBPF maps

#### Hands-On: Cilium Service Handling

```bash
# Install Cilium with kube-proxy replacement
cilium install --set kubeProxyReplacement=true

# Verify kube-proxy replacement
cilium status | grep KubeProxyReplacement
# KubeProxyReplacement:   True

# View Cilium's service map
cilium service list
# ID  Frontend            Service Type   Backend
# 1   10.96.0.1:443       ClusterIP      1 => 192.168.1.10:6443
# 2   10.96.0.10:53       ClusterIP      1 => 10.244.1.2:53
#                                         2 => 10.244.2.3:53
# 3   10.96.45.123:80     ClusterIP      1 => 10.244.1.5:8080
#                                         2 => 10.244.2.9:8080
#                                         3 => 10.244.3.4:8080

# View eBPF maps
cilium bpf lb list
# SERVICE ADDRESS     BACKEND ADDRESS
# 10.96.45.123:80     10.244.1.5:8080 (1)
#                     10.244.2.9:8080 (2)
#                     10.244.3.4:8080 (3)

# Monitor real-time packet flow
cilium monitor --type=l7
```

---

## 23.5 Packet Flow Trace — Complete Journey

Let's trace the complete journey of an HTTP request from Pod A on Node 1 to a Service
that resolves to Pod B on Node 2. We'll use iptables mode with VXLAN overlay as the
baseline, since that's the most common setup.

### Setup

```
Pod A:     10.244.1.20 on Node 1 (192.168.1.10)
Service:   my-service, ClusterIP 10.96.45.123:80
Pod B:     10.244.2.9:8080 on Node 2 (192.168.1.11)  (selected by kube-proxy)
Pod C:     10.244.1.5:8080 on Node 1 (not selected this time)
Pod D:     10.244.3.4:8080 on Node 3 (not selected this time)
```

### Step-by-Step Trace

```
  Node 1 (192.168.1.10)                    Node 2 (192.168.1.11)
  ┌──────────────────────────┐              ┌──────────────────────────┐
  │                          │              │                          │
  │  Pod A (10.244.1.20)     │              │  Pod B (10.244.2.9)      │
  │  ┌────────────────────┐  │              │  ┌────────────────────┐  │
  │  │ curl 10.96.45.123  │  │              │  │ nginx :8080        │  │
  │  └─────────┬──────────┘  │              │  └─────────▲──────────┘  │
  │    ① DNS? No, raw IP     │              │            │              │
  │            │              │              │    ⑫ Packet delivered    │
  │  ┌─────────▼──────────┐  │              │  ┌─────────┴──────────┐  │
  │  │ eth0 (veth inside) │  │              │  │ eth0 (veth inside) │  │
  │  └─────────┬──────────┘  │              │  └─────────▲──────────┘  │
  │    ② Exits Pod namespace │              │    ⑪ Enters Pod namespace│
  │            │              │              │            │              │
  │  ┌─════════╪══════════════╪══════════════╪════════════╪═══════════┐ │
  │  ║ Host    │ Namespace    │              │            │           ║ │
  │  ║─────────▼──────────┐   │              │  ┌─────────┴────────  ║ │
  │  ║ vethXXXX           │   │              │  │ vethYYYY          ║  │
  │  ║─────────┬──────────┘   │              │  └─────────▲────────  ║ │
  │  ║    ③    │              │              │    ⑩       │          ║ │
  │  ║─────────▼──────────┐   │              │  ┌─────────┴────────  ║ │
  │  ║ Bridge (cni0)      │   │              │  │ Bridge (cni0)     ║  │
  │  ║─────────┬──────────┘   │              │  └─────────▲────────  ║ │
  │  ║    ④    │              │              │    ⑨       │          ║ │
  │  ║─────────▼──────────┐   │              │  ┌─────────┴────────  ║ │
  │  ║ iptables NAT       │   │              │  │ iptables NAT      ║ │
  │  ║ DNAT:              │   │              │  │ (conntrack un-    ║  │
  │  ║ 10.96.45.123:80 →  │   │              │  │  DNATs response)  ║ │
  │  ║ 10.244.2.9:8080    │   │              │  │                   ║  │
  │  ║─────────┬──────────┘   │              │  └─────────▲────────  ║ │
  │  ║    ⑤    │              │              │    ⑧       │          ║ │
  │  ║─────────▼──────────┐   │              │  ┌─────────┴────────  ║ │
  │  ║ Routing table:     │   │              │  │ VTEP decapsulate  ║  │
  │  ║ 10.244.2.0/24 via  │   │              │  │ (flannel.1)       ║ │
  │  ║ flannel.1          │   │              │  │                   ║  │
  │  ║─────────┬──────────┘   │              │  └─────────▲────────  ║ │
  │  ║    ⑥    │              │              │    ⑦       │          ║ │
  │  ║─────────▼──────────┐   │              │  ┌─────────┴────────  ║ │
  │  ║ VTEP encapsulate   │   │              │  │ Physical NIC      ║  │
  │  ║ (flannel.1)        │   │              │  │ (eth0)            ║ │
  │  ║─────────┬──────────┘   │              │  └─────────▲────────  ║ │
  │  ║─────────▼──────────┐   │              │            │          ║ │
  │  ║ Physical NIC (eth0)│   │              │            │          ║ │
  │  ║─────────┬──────────┘   │              │            │          ║ │
  │  ╚═════════╪══════════════╪══════════════╪════════════╪══════════╝ │
  └────────────┼──────────────┘              └────────────┼────────────┘
               │          Physical Network                │
               └──────────────────────────────────────────┘
```

### Detailed Hop-by-Hop

**① Application sends request.**
Pod A's curl calls `connect(10.96.45.123, 80)`. The kernel creates a TCP SYN packet:
`src=10.244.1.20:54321, dst=10.96.45.123:80`

**② Packet exits Pod namespace.**
The packet goes through Pod A's eth0 (the veth inside the namespace). Since the default
route points to eth0, the packet is emitted to the veth pair.

**③ Arrives at bridge.**
The host-side veth (vethXXXX) receives the packet and delivers it to the bridge (cni0).

**④ iptables NAT (DNAT).**
Before the bridge routes the packet, netfilter processes it. The packet enters the
PREROUTING chain → KUBE-SERVICES → KUBE-SVC-XXX chain. The statistic module selects
KUBE-SEP-BBB → DNAT to 10.244.2.9:8080.

Packet is now: `src=10.244.1.20:54321, dst=10.244.2.9:8080`
A conntrack entry is created recording this translation.

**⑤ Routing decision.**
The kernel consults the routing table. `10.244.2.0/24` matches a route via `flannel.1`
(the VTEP). The packet is handed to the VXLAN device.

**⑥ VXLAN encapsulation.**
flannel.1 wraps the packet in a VXLAN header:
- Outer: `src=192.168.1.10, dst=192.168.1.11, UDP:4789`
- Inner: `src=10.244.1.20, dst=10.244.2.9`

**⑦ Physical network transit.**
The encapsulated packet is sent from Node 1's eth0 to Node 2's eth0 via the physical network.

**⑧ VXLAN decapsulation.**
Node 2's kernel receives the UDP packet on port 4789. The VTEP (flannel.1) strips the
outer headers, revealing the inner packet: `src=10.244.1.20, dst=10.244.2.9:8080`

**⑨ Local routing.**
The kernel routes the inner packet. `10.244.2.0/24` is local (on cni0), so the packet
goes to the bridge.

**⑩ Bridge forwards.**
The bridge looks up the MAC address for 10.244.2.9 and forwards to the appropriate veth.

**⑪ Enters Pod B namespace.**
The packet traverses the veth pair into Pod B's namespace.

**⑫ Application receives request.**
Pod B's nginx receives the HTTP request on port 8080.

### Return Path

Pod B responds: `src=10.244.2.9:8080, dst=10.244.1.20:54321`

The response follows the reverse path: Pod B → veth → bridge → VTEP encap → physical
network → Node 1 VTEP decap → bridge → **conntrack un-DNAT** (rewrites
`src=10.244.2.9:8080` to `src=10.96.45.123:80`) → veth → Pod A.

Pod A's curl receives the response appearing to come from 10.96.45.123:80 — it never
knows that 10.244.2.9 was involved.

### Hands-On: Tracing the Full Path

```bash
# Terminal 1: tcpdump on Node 1's bridge (shows DNAT'd packets)
docker exec kind-worker tcpdump -i cni0 -nn host 10.244.1.20

# Terminal 2: tcpdump on Node 1's physical interface (shows VXLAN)
docker exec kind-worker tcpdump -i eth0 -nn udp port 4789

# Terminal 3: tcpdump on Node 2's bridge (shows decapsulated packets)
docker exec kind-worker2 tcpdump -i cni0 -nn host 10.244.2.9

# Terminal 4: Send traffic
kubectl exec pod-a -- curl -s 10.96.45.123

# Terminal 1 output (after DNAT):
# 10.244.1.20.54321 > 10.244.2.9.8080: Flags [S], ...
# (Notice: destination is Pod IP, not Service IP — DNAT happened!)

# Terminal 2 output (VXLAN encapsulated):
# 192.168.1.10.46372 > 192.168.1.11.4789: VXLAN, ...
#   10.244.1.20.54321 > 10.244.2.9.8080: ...

# Terminal 3 output (decapsulated):
# 10.244.1.20.54321 > 10.244.2.9.8080: Flags [S], ...

# Verify conntrack entries on Node 1
docker exec kind-worker conntrack -L -d 10.96.45.123

# Verify iptables counters
docker exec kind-worker iptables -t nat -L KUBE-SVC-XXX -nv
# (packet counters show which SEP chains were hit)
```

---

## 23.6 NodePort and LoadBalancer — External Access

So far, all communication has been within the cluster. NodePort and LoadBalancer Services
expose applications to external clients.

### NodePort

A NodePort Service allocates a port (default range: 30000-32767) on every node's external
IP. Traffic arriving at `<NodeIP>:<NodePort>` is forwarded to the Service.

```
  External Client
       │
       │  dst=192.168.1.10:30080
       ▼
  ┌─────────────────────────────────────────┐
  │ Node 1 (192.168.1.10)                   │
  │                                          │
  │ iptables (KUBE-NODEPORTS chain):         │
  │ -p tcp --dport 30080                     │
  │ -j KUBE-SVC-XXXXXXXXXXXXXXXX             │
  │                                          │
  │ Same KUBE-SVC chain as ClusterIP:        │
  │ → DNAT to Pod A, B, or C                 │
  │                                          │
  └─────────────────────────────────────────┘
```

**iptables rules for NodePort:**

```
# In KUBE-NODEPORTS chain:
-p tcp --dport 30080 -j KUBE-SVC-XXXXXXXXXXXXXXXX

# KUBE-SERVICES chain has a catch-all at the end:
-m addrtype --dst-type LOCAL -j KUBE-NODEPORTS
```

The key insight: NodePort reuses the same `KUBE-SVC-xxx` chain as ClusterIP. The only
difference is the entry point — instead of matching on the ClusterIP, it matches on the
NodePort.

### LoadBalancer

A LoadBalancer Service is a NodePort Service plus an external load balancer provisioned
by the cloud provider (or MetalLB on bare metal).

```
  External Client
       │
       │  dst=34.123.45.67:80  (External LB IP)
       ▼
  ┌──────────────────────┐
  │  External Load       │
  │  Balancer             │
  │  (Cloud Provider)    │
  └─────────┬────────────┘
            │  dst=192.168.1.10:30080  (NodePort)
            ▼
  ┌─────────────────────────────────────────┐
  │  Node 1                                  │
  │  iptables DNAT → Pod backend             │
  └─────────────────────────────────────────┘
```

### externalTrafficPolicy: Cluster vs Local

This is a critical setting that affects source IP preservation and traffic distribution.

#### externalTrafficPolicy: Cluster (default)

```
  External Client (203.0.113.5)
       │
       ▼
  ┌─────────────────────────────────────┐
  │ Node 1 (no local Pod!)              │
  │                                      │
  │ iptables:                            │
  │ 1. DNAT to Pod B on Node 2          │
  │ 2. SNAT source to Node 1's IP       │
  │    (so the return path comes back    │
  │     through Node 1)                  │
  │                                      │
  │ Packet after NAT:                    │
  │ src=192.168.1.10 dst=10.244.2.9     │
  │ (original client IP 203.0.113.5 is  │
  │  lost!)                              │
  └────────────┬────────────────────────┘
               │
               ▼
  ┌─────────────────────────────────────┐
  │ Node 2                               │
  │ Pod B receives packet from           │
  │ 192.168.1.10, not from 203.0.113.5  │
  └─────────────────────────────────────┘
```

**Why SNAT?** Without SNAT, Pod B would respond directly to 203.0.113.5. But the client
sent its request to Node 1 — the response from Node 2 would have the wrong source IP and
the client's TCP stack would drop it (it's not from the IP it connected to). SNAT ensures
all traffic goes back through Node 1, which un-SNATs it.

**Drawbacks:**
- Source IP is lost — Pod B sees Node 1's IP, not the real client
- Extra network hop — traffic bounces between nodes
- Even distribution — but some nodes may have more Pods than others

#### externalTrafficPolicy: Local

```
  External Client (203.0.113.5)
       │
       ├──────────────────────────────────┐
       ▼                                  ▼
  ┌────────────────┐              ┌────────────────┐
  │ Node 1         │              │ Node 2         │
  │ (no local Pod) │              │ (has Pod B)    │
  │                │              │                │
  │ DROP! No local │              │ DNAT to Pod B  │
  │ endpoint       │              │ No SNAT needed │
  │                │              │                │
  │                │              │ Pod B sees:    │
  │                │              │ src=203.0.113.5│
  └────────────────┘              └────────────────┘
```

**How it works:** With `externalTrafficPolicy: Local`, kube-proxy only programs NodePort
rules to DNAT to *local* Pods. If no local Pod exists on a node, the traffic is dropped.
Since the Pod is local, no SNAT is needed and the source IP is preserved.

**Health checks:** The external load balancer uses health checks on each node
(`/healthz` endpoint on NodePort) to determine which nodes have local Pods. It only sends
traffic to nodes that pass the health check.

```bash
# Health check port is in the Service spec
kubectl get svc my-service -o jsonpath='{.spec.healthCheckNodePort}'
# 31234

# Nodes with a local Pod respond 200
curl http://node2:31234/healthz
# {"localEndpoints":1}

# Nodes without a local Pod respond 503
curl http://node1:31234/healthz
# {"localEndpoints":0}
```

**Drawbacks:**
- Potential traffic imbalance — nodes with more Pods get more traffic
- Nodes without local Pods drop traffic (requires smart LB)

### Hands-On: Tracing NodePort Packet Flow

```bash
# Create a NodePort Service
kubectl expose deployment web --type=NodePort --port=80 --name=web-nodeport

# Get the allocated NodePort
kubectl get svc web-nodeport
# NAME           TYPE       CLUSTER-IP    PORT(S)       
# web-nodeport   NodePort   10.96.50.10   80:30080/TCP

# Inspect NodePort iptables rules on a node
docker exec kind-worker iptables -t nat -L KUBE-NODEPORTS -n
# -p tcp --dport 30080 -j KUBE-SVC-XXX

# With externalTrafficPolicy: Cluster, also see the SNAT rule:
docker exec kind-worker iptables -t nat -L KUBE-POSTROUTING -n
# -m mark --mark 0x4000/0x4000 -j MASQUERADE

# Send traffic from outside
curl http://192.168.1.10:30080

# On the destination node, watch conntrack
docker exec kind-worker conntrack -E -p tcp --dport 30080
```

---

## 23.7 Service Topology and Traffic Routing

Kubernetes provides mechanisms to control where Service traffic goes based on topology
(e.g., preferring same-zone or same-node endpoints).

### internalTrafficPolicy

`internalTrafficPolicy` controls routing for traffic originating *inside* the cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  internalTrafficPolicy: Local   # Only route to Pods on the same node
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

| Value | Behavior |
|-------|----------|
| `Cluster` (default) | Route to any endpoint in the cluster |
| `Local` | Only route to endpoints on the same node; fail if none exist |

**Use case:** DaemonSet services (logging agents, monitoring proxies) where every node
has a local instance and you want to avoid cross-node traffic.

### Topology Aware Hints

Topology Aware Hints (replacing the deprecated `topologyKeys`) allow kube-proxy to prefer
endpoints in the same zone as the client.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
  annotations:
    service.kubernetes.io/topology-mode: Auto
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 8080
```

When `topology-mode: Auto` is set, the EndpointSlice controller adds hints to each
endpoint indicating which zones should consume it:

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: my-service-abc
endpoints:
- addresses: ["10.244.1.5"]
  zone: us-east-1a
  hints:
    forZones:
    - name: us-east-1a    # Clients in us-east-1a should use this endpoint
- addresses: ["10.244.2.9"]
  zone: us-east-1b
  hints:
    forZones:
    - name: us-east-1b    # Clients in us-east-1b should use this endpoint
```

kube-proxy on each node reads these hints and only programs rules for endpoints that
match the node's zone. This keeps traffic local to a zone, reducing latency and
cross-zone data transfer costs.

```
  Zone A                           Zone B
  ┌──────────────────────┐         ┌──────────────────────┐
  │ Node 1 (zone A)      │         │ Node 2 (zone B)      │
  │                      │         │                      │
  │ Client Pod           │         │ Client Pod           │
  │ → Service            │         │ → Service            │
  │ → Pod A (zone A) ✓   │         │ → Pod B (zone B) ✓   │
  │   Pod B (zone B) ✗   │         │   Pod A (zone A) ✗   │
  │   (filtered out)     │         │   (filtered out)     │
  └──────────────────────┘         └──────────────────────┘
```

**Safeguards:**
- If zone allocation would cause >20% traffic imbalance, hints are not generated
- If no endpoints exist in a zone, all endpoints are used (fallback)

### Hands-On: Zone-Aware Traffic Routing

```bash
# Label nodes with zones (if not already done by cloud provider)
kubectl label node kind-worker topology.kubernetes.io/zone=zone-a
kubectl label node kind-worker2 topology.kubernetes.io/zone=zone-b

# Create a service with topology-mode
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: zone-aware-svc
  annotations:
    service.kubernetes.io/topology-mode: Auto
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80
EOF

# Examine EndpointSlice for hints
kubectl get endpointslice -l kubernetes.io/service-name=zone-aware-svc -o yaml

# Verify traffic stays in-zone
# From a Pod on kind-worker (zone-a), traffic should go to local endpoints
kubectl exec -it pod-on-zone-a -- curl -s zone-aware-svc
# Check which Pod responded (should be a zone-a Pod)
```

---

## 23.8 Troubleshooting Networking

Network issues in Kubernetes can be frustrating because there are so many layers involved.
Here's a systematic approach to diagnosing common problems.

### Problem: Pod Can't Reach a Service

**Step 1: Is DNS working?**

```bash
# From inside the Pod
kubectl exec -it my-pod -- nslookup my-service
kubectl exec -it my-pod -- nslookup my-service.my-namespace.svc.cluster.local

# If DNS fails, check CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns
kubectl logs -n kube-system -l k8s-app=kube-dns
```

**Step 2: Does the Service have endpoints?**

```bash
kubectl get endpoints my-service
# NAME         ENDPOINTS                          AGE
# my-service   10.244.1.5:8080,10.244.2.9:8080   5m

# If ENDPOINTS is empty:
# - Check that Pods exist and are Ready
# - Check that the Service selector matches Pod labels
kubectl get pods -l app=my-app --show-labels
kubectl describe svc my-service
```

**Step 3: Can you reach the ClusterIP directly?**

```bash
kubectl exec -it my-pod -- curl -v 10.96.45.123:80
# If this fails but direct Pod IP works, the issue is kube-proxy/iptables
```

**Step 4: Can you reach the Pod directly?**

```bash
kubectl exec -it my-pod -- curl -v 10.244.2.9:8080
# If this works, the issue is in Service translation (kube-proxy)
# If this fails, the issue is in Pod-to-Pod networking (CNI)
```

**Step 5: Check iptables rules**

```bash
# On the node where the client Pod runs
iptables -t nat -L KUBE-SERVICES -n | grep <ClusterIP>
# Verify the KUBE-SVC chain exists and has endpoints
```

### Problem: Cross-Node Communication Fails

**Step 1: Check CNI plugin status**

```bash
kubectl get pods -n kube-system
# Look for CNI pods (flannel, calico, cilium, etc.)
# Are they Running? Any restarts?

kubectl logs -n kube-system <cni-pod>
```

**Step 2: Check node routes**

```bash
# On Node 1, can we route to Node 2's Pod CIDR?
ip route | grep 10.244.2
# 10.244.2.0/24 via 192.168.1.11 dev flannel.1

# If the route is missing, the CNI agent hasn't programmed it
```

**Step 3: Check for firewall rules**

```bash
# Check iptables FORWARD chain
iptables -L FORWARD -n
# Are there DROP rules? (common with Docker's default iptables rules)

# Check for cloud security group rules blocking inter-node traffic
# VXLAN: UDP port 4789 must be open between nodes
# BGP: TCP port 179 must be open between nodes
# Cilium: UDP port 8472 (Geneve) or 51871 (WireGuard)
```

**Step 4: Test at each layer**

```bash
# Can nodes ping each other?
ping 192.168.1.11

# Can nodes reach each other's Pod CIDRs?
ping 10.244.2.1  # Node 2's bridge IP

# For VXLAN: is the tunnel up?
ip -d link show flannel.1
# state UNKNOWN — VXLAN interface
```

### Problem: Intermittent Connection Resets

This is often a conntrack issue. Conntrack table full, or conntrack race conditions.

```bash
# Check conntrack table size
sysctl net.netfilter.nf_conntrack_count
sysctl net.netfilter.nf_conntrack_max

# If count is near max, increase:
sysctl -w net.netfilter.nf_conntrack_max=1048576

# Check for conntrack drops
conntrack -S
# cpu=0  found=0 invalid=123 insert=0 insert_failed=45 drop=45
# insert_failed and drop indicate table full or hash collisions
```

### Diagnostic Tools Reference

```
┌────────────────────────────────────────────────────────────────────────┐
│  Tool           │ Purpose                                              │
├─────────────────┼────────────────────────────────────────────────────  │
│  kubectl exec   │ Run commands inside a Pod's network namespace        │
│  nslookup/dig   │ Test DNS resolution                                  │
│  curl/wget      │ Test HTTP connectivity                               │
│  ping           │ Test ICMP connectivity                                │
│  traceroute     │ Trace packet path (may not work with overlays)       │
│  tcpdump        │ Capture and analyze packets at any interface          │
│  ip route       │ View routing table                                    │
│  ip link        │ View network interfaces and their state               │
│  ip addr        │ View IP addresses on interfaces                       │
│  bridge link    │ Show bridge-attached interfaces                       │
│  iptables -L    │ List iptables rules (add -t nat for NAT rules)       │
│  conntrack -L   │ List connection tracking entries                      │
│  conntrack -E   │ Monitor conntrack events in real time                 │
│  ipvsadm -Ln    │ List IPVS virtual servers and real servers           │
│  ss -tnp        │ List TCP sockets and their processes                  │
│  cilium monitor │ Real-time eBPF event monitoring (Cilium)             │
│  cilium bpf     │ Inspect eBPF maps (Cilium)                           │
│  calicoctl      │ Calico-specific diagnostics                           │
└────────────────────────────────────────────────────────────────────────┘
```

### Hands-On: Systematic Network Debugging Workflow

Here's a script that performs a comprehensive network diagnostic:

```bash
#!/bin/bash
# k8s-network-debug.sh — Systematic network debugging
# Usage: kubectl exec -it debug-pod -- /bin/bash -c "$(cat k8s-network-debug.sh)"

SERVICE_NAME=${1:-"my-service"}
SERVICE_PORT=${2:-80}

echo "=== Step 1: DNS Resolution ==="
nslookup $SERVICE_NAME
echo ""

echo "=== Step 2: Service ClusterIP ==="
SVC_IP=$(nslookup $SERVICE_NAME | grep Address | tail -1 | awk '{print $2}')
echo "Service IP: $SVC_IP"
echo ""

echo "=== Step 3: Connectivity to ClusterIP ==="
curl -s -o /dev/null -w "HTTP %{http_code} in %{time_total}s\n" \
     --connect-timeout 5 http://$SVC_IP:$SERVICE_PORT/
echo ""

echo "=== Step 4: Routing Table ==="
ip route
echo ""

echo "=== Step 5: Interface Info ==="
ip addr show eth0
echo ""

echo "=== Step 6: DNS Config ==="
cat /etc/resolv.conf
echo ""

echo "=== Step 7: TCP Connections ==="
ss -tnp
echo ""

echo "=== Step 8: ARP Table ==="
ip neigh show
echo ""
```

For node-level debugging:

```bash
#!/bin/bash
# node-network-debug.sh — Run on the Kubernetes node itself

echo "=== Bridge Interfaces ==="
bridge link show 2>/dev/null

echo ""
echo "=== Pod Routes ==="
ip route | grep -E "^10\."

echo ""
echo "=== iptables Service Rules ==="
iptables -t nat -L KUBE-SERVICES -n --line-numbers 2>/dev/null | head -30

echo ""
echo "=== Conntrack Stats ==="
conntrack -S 2>/dev/null

echo ""
echo "=== Conntrack Count ==="
echo "Current: $(cat /proc/sys/net/netfilter/nf_conntrack_count)"
echo "Max:     $(cat /proc/sys/net/netfilter/nf_conntrack_max)"

echo ""
echo "=== IPVS (if enabled) ==="
ipvsadm -Ln 2>/dev/null || echo "IPVS not available"

echo ""
echo "=== Network Interfaces ==="
ip -br link show

echo ""
echo "=== Network Namespaces ==="
ip netns list 2>/dev/null | head -20
```

---

## 23.9 Summary

Kubernetes networking is a layered system. Each layer solves a specific problem:

```
  ┌──────────────────────────────────────────────────────┐
  │  Layer 7: Ingress / Gateway API                       │
  │  (HTTP routing, TLS termination — Chapter 24)        │
  ├──────────────────────────────────────────────────────┤
  │  Layer 4: Services (ClusterIP, NodePort, LoadBalancer)│
  │  Implementation: iptables / IPVS / eBPF              │
  │  Purpose: Stable VIP + load balancing                │
  ├──────────────────────────────────────────────────────┤
  │  Layer 3: Pod Network (flat IP space)                 │
  │  Implementation: CNI plugin (Flannel, Calico, Cilium)│
  │  Purpose: Pod-to-Pod connectivity                    │
  ├──────────────────────────────────────────────────────┤
  │  Layer 2: Node Network (physical/virtual)             │
  │  Implementation: veth pairs, bridges, VXLAN/BGP      │
  │  Purpose: Packet delivery between namespaces/nodes   │
  └──────────────────────────────────────────────────────┘
```

**Key takeaways:**

1. **Every Pod gets a real IP.** No NAT between Pods. This simplifies application design
   enormously — applications don't need to know about port mapping or NAT.

2. **Same-node traffic flows through veth pairs and a bridge.** The bridge learns MAC
   addresses and switches frames at L2.

3. **Cross-node traffic uses overlay (VXLAN), routing (BGP), or eBPF.** Each approach
   trades off encapsulation overhead, network requirements, and performance.

4. **Services provide stable endpoints via DNAT.** kube-proxy (iptables, IPVS, or eBPF)
   translates Service VIPs to actual Pod IPs. Conntrack remembers the translation for
   the life of the connection.

5. **iptables scales poorly; IPVS and eBPF scale well.** For large clusters, IPVS or
   Cilium's eBPF-based kube-proxy replacement is essential.

6. **externalTrafficPolicy controls source IP preservation.** `Cluster` mode SNATs
   (loses source IP but ensures even distribution), `Local` mode preserves source IP
   but may cause imbalance.

7. **Topology-aware routing reduces cross-zone traffic.** Topology hints keep traffic
   within the same failure domain when possible.

8. **Debugging requires working layer by layer.** Start with DNS, check endpoints,
   try direct Pod IPs, then inspect iptables/IPVS/eBPF rules.

In the next chapter, we'll move up to Layer 7 and explore Ingress controllers, the
Gateway API, and how HTTP routing, TLS termination, and advanced traffic management
work on top of the networking foundation we've built here.
