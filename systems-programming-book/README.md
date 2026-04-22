# Systems Programming, Linux & Kubernetes: The Deep Dive

### A Practitioner's Book for Systems Programmers — Written in Go

---

> *"You don't truly understand a system until you can explain what happens between the syscall and the hardware."*

---

## Who This Book Is For

You write Go. You build distributed systems. You want to understand **everything** under the hood — from the Linux kernel's process scheduler to how a Kubernetes controller reconciles state. This book is your end-to-end guide.

---

## 📚 Table of Contents

### Part I: Linux — The Foundation

| # | Chapter | Description |
|---|---------|-------------|
| 1 | [Linux Process Model & System Calls](chapters/01-linux-process-model.md) | Processes, threads, fork/exec, the syscall interface, Go's runtime interaction with Linux |
| 2 | [Memory Management — Virtual Memory, Paging, mmap](chapters/02-linux-memory.md) | Page tables, TLB, mmap, brk, Go's memory allocator, unsafe.Pointer deep dive |
| 3 | [File Systems, VFS & I/O](chapters/03-linux-filesystems-io.md) | VFS layer, inodes, file descriptors, epoll, io_uring, Go I/O patterns |
| 4 | [Linux Networking — Sockets, TCP/IP Stack, Netfilter](chapters/04-linux-networking.md) | Socket syscalls, TCP state machine, netfilter hooks, iptables, Go net package internals |
| 5 | [Namespaces & Cgroups — The Container Primitives](chapters/05-namespaces-cgroups.md) | All 8 namespace types, cgroups v1/v2, building isolation from scratch in Go |
| 6 | [Signals, IPC & Synchronization](chapters/06-signals-ipc.md) | Signal handling, pipes, Unix sockets, shared memory, futex, Go channels vs OS primitives |
| 7 | [Linux Security — Capabilities, Seccomp, AppArmor, SELinux](chapters/07-linux-security.md) | Capability model, seccomp-bpf, MAC systems, writing seccomp profiles in Go |

### Part II: Containers — From Scratch

| # | Chapter | Description |
|---|---------|-------------|
| 8 | [Building a Container Runtime in Go](chapters/08-container-runtime.md) | Implement a minimal container runtime using namespaces, cgroups, pivot_root |
| 9 | [OCI, containerd & the Container Ecosystem](chapters/09-oci-containerd.md) | OCI image/runtime spec, containerd architecture, shim v2, runc internals |

### Part III: Kubernetes Architecture — Under the Hood

| # | Chapter | Description |
|---|---------|-------------|
| 10 | [Kubernetes Architecture Deep Dive](chapters/10-k8s-architecture.md) | Control plane vs data plane, component interactions, the declarative model |
| 11 | [etcd — The Brain of Kubernetes](chapters/11-etcd.md) | Raft consensus, linearizable reads, watch API, compaction, Go etcd client |
| 12 | [API Server, Admission Controllers & API Machinery](chapters/12-api-server.md) | Request lifecycle, authn/authz chain, admission webhooks, API aggregation |
| 13 | [Scheduler Deep Dive](chapters/13-scheduler.md) | Scheduling framework, filtering/scoring, plugins, gang scheduling, Go examples |
| 14 | [Kubelet, CRI & Pod Lifecycle](chapters/14-kubelet-cri.md) | Kubelet sync loop, CRI-O/containerd, pod sandbox, container states |

### Part IV: Kubernetes Core Concepts — Hands On

| # | Chapter | Description |
|---|---------|-------------|
| 15 | [Pods — The Atomic Unit](chapters/15-pods.md) | Pod spec deep dive, init containers, sidecar pattern, probes, resource management |
| 16 | [ReplicaSets & Deployments](chapters/16-replicasets-deployments.md) | Replica management, rolling updates, rollback, blue-green, canary, Go client-go |
| 17 | [Services, Endpoints & Service Discovery](chapters/17-services.md) | ClusterIP, NodePort, LoadBalancer, ExternalName, EndpointSlices, headless services |
| 18 | [ConfigMaps, Secrets & Environment Management](chapters/18-configmaps-secrets.md) | Config injection patterns, volume mounts, env vars, immutable configs |
| 19 | [StatefulSets — Stateful Workloads Done Right](chapters/19-statefulsets.md) | Ordered deployment, stable network IDs, persistent storage, headless services |
| 20 | [DaemonSets, Jobs & CronJobs](chapters/20-daemonsets-jobs.md) | Node-level workloads, batch processing, parallel jobs, completion tracking |
| 21 | [Persistent Volumes, Storage Classes & CSI](chapters/21-storage-csi.md) | PV/PVC lifecycle, dynamic provisioning, CSI driver architecture, writing a CSI driver |

### Part V: Kubernetes Networking — The Full Stack

| # | Chapter | Description |
|---|---------|-------------|
| 22 | [CNI — Container Network Interface Deep Dive](chapters/22-cni.md) | CNI spec, plugin chain, bridge/VXLAN/BGP, writing a CNI plugin in Go |
| 23 | [Cluster Networking — Pod-to-Pod, Pod-to-Service](chapters/23-cluster-networking.md) | Flat network model, kube-proxy modes (iptables/IPVS/eBPF), packet flow tracing |
| 24 | [Ingress, Gateway API & Load Balancing](chapters/24-ingress-gateway.md) | Ingress controllers, Gateway API resources, TLS termination, path routing |
| 25 | [Network Policies — Microsegmentation](chapters/25-network-policies.md) | Policy spec, Calico/Cilium enforcement, zero-trust networking |
| 26 | [DNS in Kubernetes — CoreDNS Deep Dive](chapters/26-dns-coredns.md) | CoreDNS architecture, plugins, service discovery, DNS debugging |

### Part VI: Kubernetes Security — Production Hardening

| # | Chapter | Description |
|---|---------|-------------|
| 27 | [RBAC — Role-Based Access Control](chapters/27-rbac.md) | Roles, ClusterRoles, bindings, least privilege, audit logging |
| 28 | [Pod Security Standards & Admission](chapters/28-pod-security.md) | PSS levels, PSA controller, OPA/Gatekeeper, Kyverno |
| 29 | [Service Accounts, OIDC & Authentication](chapters/29-authentication.md) | SA token projection, OIDC integration, certificate auth, impersonation |
| 30 | [Secrets Management & Encryption at Rest](chapters/30-secrets-encryption.md) | EncryptionConfiguration, external secrets operator, HashiCorp Vault integration |
| 31 | [Supply Chain Security — Image Signing, SBOM, Policies](chapters/31-supply-chain.md) | Cosign, Sigstore, SBOM generation, admission policies for verified images |

### Part VII: Operators & Controllers — Extending Kubernetes

| # | Chapter | Description |
|---|---------|-------------|
| 32 | [The Controller Pattern — Reconciliation Loops](chapters/32-controller-pattern.md) | Level-triggered vs edge-triggered, informers, work queues, leader election |
| 33 | [Custom Resource Definitions (CRDs)](chapters/33-crds.md) | CRD spec, structural schemas, versioning, conversion webhooks |
| 34 | [Building a Kubernetes Operator in Go](chapters/34-building-operator.md) | Full operator with kubebuilder/controller-runtime, end-to-end example |
| 35 | [Advanced Operator Patterns](chapters/35-advanced-operators.md) | Finalizers, validating/mutating webhooks, status conditions, owner references |

### Part VIII: Production Systems — Putting It All Together

| # | Chapter | Description |
|---|---------|-------------|
| 36 | [Observability — Metrics, Logs, Traces](chapters/36-observability.md) | Prometheus, Grafana, OpenTelemetry, structured logging, trace propagation |
| 37 | [Autoscaling — HPA, VPA, Cluster Autoscaler, KEDA](chapters/37-autoscaling.md) | Custom metrics, scaling algorithms, event-driven scaling |
| 38 | [Helm, Kustomize & GitOps](chapters/38-helm-gitops.md) | Chart development, Kustomize overlays, ArgoCD, Flux |
| 39 | [Disaster Recovery, Backup & Multi-Cluster](chapters/39-dr-multicluster.md) | Velero, etcd backup, federation, cluster API |
| 40 | [Performance Tuning & Troubleshooting](chapters/40-performance-troubleshooting.md) | Node debugging, pod debugging, network debugging, Go profiling |

### Appendices

| # | Appendix | Description |
|---|----------|-------------|
| A | [Essential Linux Commands Cheatsheet](chapters/appendix-a-linux-cheatsheet.md) | The commands you'll use daily |
| B | [kubectl Mastery](chapters/appendix-b-kubectl.md) | Power-user kubectl, plugins, custom columns |
| C | [Interview Questions & System Design](chapters/appendix-c-interview.md) | Systems programming interview prep, K8s design questions |

---

## How to Read This Book

1. **Part I** builds your Linux foundation. The Go syscall examples are unique to this book.
2. **Part II** connects Linux primitives to containers. You'll build one from scratch.
3. **Parts III-IV** give you Kubernetes mastery — architecture first, then hands-on.
4. **Parts V-VI** cover networking and security — the two hardest interview topics.
5. **Part VII** teaches you to extend Kubernetes — the most valuable skill for systems roles.
6. **Part VIII** is production wisdom — what separates junior from senior.

Every chapter has:
- 🔬 **Deep Dive** — What's really happening under the hood
- 💻 **Hands-On** — Working code and commands you run yourself
- 🧠 **Mental Model** — How to think about the concept
- ⚡ **Go Code** — Idiomatic Go with generous doc comments

---

## Prerequisites

- Go 1.21+ installed
- A Linux machine or VM (Ubuntu 22.04+ recommended)
- `kind` or `minikube` for local Kubernetes
- Basic understanding of Go (goroutines, channels, interfaces)

```bash
# Quick setup — create a local Kubernetes cluster
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.20.0/kind-linux-amd64
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# Create a multi-node cluster for realistic testing
cat <<EOF | kind create cluster --name deep-dive --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
- role: worker
EOF

# Verify
kubectl cluster-info --context kind-deep-dive
kubectl get nodes
```

---

*Let's begin.*
