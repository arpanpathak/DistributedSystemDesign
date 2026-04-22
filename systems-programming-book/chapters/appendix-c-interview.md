# Appendix C: Interview Questions & System Design

> Deep answers to the questions you'll face in systems programming, Kubernetes,
> and infrastructure interviews. Every answer is thorough enough to stand on its own.

---

## Table of Contents

1. [Linux Systems Programming Questions](#1-linux-systems-programming-questions)
2. [Kubernetes Conceptual Questions](#2-kubernetes-conceptual-questions)
3. [System Design Questions](#3-system-design-questions)
4. [Coding Challenges (Go)](#4-coding-challenges-go)
5. [Behavioral / Systems Thinking](#5-behavioral--systems-thinking)

---

## 1. Linux Systems Programming Questions

### Q: Explain what happens when you type `ls -l` and press Enter

**Answer — the full kernel path:**

1. **Shell reads input.** The terminal driver delivers keystrokes to the shell process
   (e.g., bash). The shell's `readline` library buffers characters until Enter (`\n`).

2. **Parsing and expansion.** The shell tokenizes `ls -l`, performs alias expansion,
   glob expansion (none here), and variable substitution. It searches `$PATH` for the
   `ls` binary (e.g., `/usr/bin/ls`).

3. **fork().** The shell calls `fork()`, which creates a child process. The kernel:
   - Allocates a new `task_struct` (process descriptor).
   - Copies the parent's page tables with Copy-on-Write (COW) semantics — physical pages
     are shared until either process writes, triggering a page fault and actual copy.
   - Assigns a new PID. The child inherits open file descriptors, environment, etc.

4. **execve().** The child calls `execve("/usr/bin/ls", ["ls", "-l"], environ)`.
   The kernel:
   - Opens the ELF binary and checks the magic number.
   - Tears down the child's old address space.
   - Maps the ELF segments (text, data, bss) into the new address space.
   - Sets up the stack with `argc`, `argv`, `envp`, and auxiliary vectors.
   - If dynamically linked, maps `ld-linux.so` (the dynamic linker) into memory.
   - Sets the instruction pointer to the entry point (dynamic linker or `_start`).

5. **Dynamic linking.** `ld-linux.so` loads shared libraries (`libc.so`, etc.),
   resolves symbols, performs relocations, then jumps to `main()`.

6. **ls execution.** `ls` calls `opendir()` → `getdents64()` syscall to read directory
   entries. For `-l`, it calls `stat()` or `lstat()` on each entry to get metadata
   (permissions, size, timestamps, link count). It formats output and calls `write()`
   to fd 1 (stdout).

7. **write() syscall.** The kernel copies data from userspace to the kernel buffer.
   The terminal driver and pseudo-terminal subsystem deliver the bytes to the terminal
   emulator, which renders them on screen.

8. **exit().** `ls` calls `exit()`. The kernel releases the process's resources (memory
   mappings, file descriptors, signal handlers), sets the state to `EXIT_ZOMBIE`, and
   sends `SIGCHLD` to the parent (the shell).

9. **wait().** The shell calls `waitpid()` to collect the child's exit status. The
   kernel removes the zombie `task_struct`. The shell prints the next prompt.

---

### Q: What happens when a process calls malloc()?

**Answer:**

`malloc()` is a userspace function in libc (glibc's implementation is ptmalloc2). It does
NOT directly invoke a syscall for every allocation.

**Small allocations (< ~128KB by default):**

1. `malloc()` checks its internal free lists (bins) for a suitable previously-freed chunk.
2. If none available, it calls `brk()` / `sbrk()` to extend the program break — the end
   of the data segment. The kernel updates the `mm_struct.brk` field but does NOT
   allocate physical memory yet.
3. When the program first accesses the new memory, a **page fault** occurs:
   - The MMU can't find a valid page table entry → triggers a hardware exception.
   - The kernel's page fault handler checks if the address is within a valid VMA
     (Virtual Memory Area). It is (because `brk()` extended the VMA).
   - The kernel allocates a physical page frame, zeroes it, creates a page table entry
     mapping the virtual page to the physical frame, and returns to userspace.
   - This is a **minor page fault** (no disk I/O needed).

**Large allocations (>= ~128KB):**

1. `malloc()` calls `mmap()` with `MAP_ANONYMOUS | MAP_PRIVATE`.
2. The kernel creates a new VMA in the process's address space but again does NOT
   allocate physical pages immediately (demand paging).
3. First access triggers the same minor page fault mechanism.

**Key details:**
- `free()` returns memory to malloc's internal free lists. For `mmap()`-allocated
  regions, `free()` calls `munmap()` to return memory to the kernel immediately.
- The program break only shrinks if there's a large free region at the top of the heap.
- This means a process's RSS (Resident Set Size) can remain high even after `free()`
  because the freed heap pages are still mapped — they're just reusable by malloc.

---

### Q: Explain fork() — what's copied, what's shared, COW

**Answer:**

`fork()` creates a new process that is a near-exact copy of the parent.

**What's copied (logically):**
- Address space (text, data, heap, stack) — via COW, see below
- File descriptor table (but the underlying file descriptions / `struct file` are shared)
- Signal dispositions and pending signals
- Process credentials (UID, GID)
- Current working directory, root directory, umask
- Resource limits
- Process group and session membership

**What's NOT copied / is different:**
- PID (child gets a new one)
- PPID (child's parent is the calling process)
- Pending signals are cleared in the child
- File locks are NOT inherited
- timers are NOT inherited
- The child's resource usage counters are reset to zero

**Copy-on-Write (COW):**

After `fork()`, parent and child share the SAME physical pages. The kernel marks all
writable pages as read-only in both processes' page tables. When either process tries
to write:

1. The CPU raises a page fault (write to read-only page).
2. The kernel's fault handler sees this is a COW page (reference count > 1).
3. The kernel allocates a new physical page, copies the contents, and updates the
   writing process's page table to point to the new page (now writable).
4. The original page's reference count is decremented.

This makes `fork()` extremely fast — the only real cost is duplicating the page tables
and a few kernel structures. If the child immediately calls `exec()`, almost no pages
are actually copied.

---

### Q: What's the difference between a process and a thread in Linux?

**Answer:**

In Linux, both processes and threads are represented by `task_struct` — the kernel
doesn't fundamentally distinguish between them. The difference is what they share.

| Aspect | Process (fork) | Thread (clone with CLONE_VM, etc.) |
|--------|---------------|-------------------------------------|
| Address space | Separate (COW) | Shared |
| File descriptors | Copied | Shared |
| PID | Different | Same TGID, different TID |
| Signal handlers | Copied | Shared |
| Stack | Separate | Separate (each thread has its own) |
| Memory mappings | Independent after fork | Shared (mmap in one thread visible to all) |
| Scheduling | Independent | Independent (each thread is a schedulable entity) |

**Implementation detail:** `clone()` is the underlying syscall for both `fork()` and
thread creation. The flags determine what's shared:
- `fork()` = `clone()` with no sharing flags
- `pthread_create()` = `clone(CLONE_VM | CLONE_FS | CLONE_FILES | CLONE_SIGHAND | CLONE_THREAD | ...)`

**TGID vs TID:** Every thread has a unique TID (Thread ID). Threads in the same process
share a TGID (Thread Group ID), which is the PID of the first thread. `getpid()` returns
the TGID, `gettid()` returns the TID.

---

### Q: Explain the TCP three-way handshake and TIME_WAIT

**Answer:**

**Three-way handshake (connection establishment):**

```
Client                          Server
  |--- SYN (seq=x) ------------>|    Client sends SYN, enters SYN_SENT
  |<-- SYN-ACK (seq=y, ack=x+1)|    Server sends SYN-ACK, enters SYN_RCVD
  |--- ACK (ack=y+1) ---------->|    Client sends ACK, both enter ESTABLISHED
```

1. **SYN:** Client picks an initial sequence number (ISN) `x`, sends SYN segment.
2. **SYN-ACK:** Server picks its ISN `y`, acknowledges client's ISN (`ack=x+1`), sends
   SYN-ACK.
3. **ACK:** Client acknowledges server's ISN (`ack=y+1`). Connection is established.

The ISNs are randomized to prevent sequence prediction attacks.

**Connection termination (four-way):**

```
Client                          Server
  |--- FIN ---------------------->|    Client initiates close
  |<-- ACK ----------------------|    Server acknowledges
  |<-- FIN ----------------------|    Server finishes sending and closes
  |--- ACK ---------------------->|    Client acknowledges, enters TIME_WAIT
  |   (waits 2*MSL)              |
```

**TIME_WAIT:** The side that initiates the close enters TIME_WAIT for 2×MSL (Maximum
Segment Lifetime, typically 60 seconds on Linux). This serves two purposes:

1. **Reliable termination:** If the final ACK is lost, the remote side will retransmit
   its FIN, and the TIME_WAIT side can re-send the ACK.
2. **Prevent stale segments:** Ensures old duplicate packets from this connection expire
   before the same port pair can be reused.

**Practical implications:**
- High-traffic servers can accumulate thousands of TIME_WAIT sockets.
- `SO_REUSEADDR` allows binding to a port in TIME_WAIT.
- `tcp_tw_reuse` kernel parameter allows reusing TIME_WAIT connections for outbound.
- `ss -o state time-wait | wc -l` to count TIME_WAIT connections.

---

### Q: What is epoll and how does it differ from select/poll?

**Answer:**

All three are I/O multiplexing mechanisms — they let a single thread monitor multiple
file descriptors for readiness.

**select():**
- Pass a bitmask of fds to the kernel. Kernel scans ALL fds on every call.
- Limited to `FD_SETSIZE` (typically 1024) file descriptors.
- O(n) per call where n = highest fd number.
- The fd sets are destroyed on return, so you must rebuild them every time.

**poll():**
- Pass an array of `pollfd` structs. No fd limit.
- Still O(n) per call — kernel scans the entire array.
- Doesn't destroy the input array, slightly more convenient than select.

**epoll():**
- Two-phase API: `epoll_create()` + `epoll_ctl()` (register fds) + `epoll_wait()` (block).
- The kernel maintains a data structure (red-black tree + ready list).
- When an fd becomes ready, the kernel adds it to the ready list via a callback.
- `epoll_wait()` returns ONLY ready fds — O(1) for the wait, O(k) where k = ready fds.
- Supports edge-triggered (EPOLLET) and level-triggered modes.

**Why epoll wins at scale:**

| fds | select/poll per-call cost | epoll per-call cost |
|-----|---------------------------|---------------------|
| 100 | O(100) | O(ready) |
| 10,000 | O(10,000) | O(ready) |
| 100,000 | O(100,000) | O(ready) |

**Edge-triggered vs Level-triggered:**
- **Level-triggered (default):** `epoll_wait()` returns an fd as long as it's ready.
  If you don't read all data, it will be returned again next call.
- **Edge-triggered (EPOLLET):** `epoll_wait()` returns an fd only when its state
  changes (new data arrives). You MUST read until `EAGAIN` or you'll miss events.

---

### Q: How does virtual memory work? Page tables, TLB, page faults

**Answer:**

**Virtual memory** gives each process its own private, contiguous address space that
is independent of physical memory layout.

**Page tables:**
- Virtual addresses are split into: page number + offset within page (typically 4KB pages).
- The page table maps virtual page numbers to physical frame numbers.
- x86-64 uses 4-level page tables: PGD → PUD → PMD → PTE (Page Table Entry).
  Each level indexes 9 bits of the virtual address, plus 12 bits of offset = 48-bit
  virtual addresses (256 TB).
- Each PTE contains: physical frame number, present bit, read/write bit, user/supervisor
  bit, accessed bit, dirty bit, NX (no-execute) bit.

**TLB (Translation Lookaside Buffer):**
- The TLB is a CPU cache of recent virtual→physical translations.
- On every memory access, the CPU first checks the TLB. TLB hit = no page table walk.
- TLB miss = hardware page table walker traverses the 4-level structure (expensive: up to
  4 memory accesses).
- TLB is flushed on context switch (or uses ASID/PCID tags to avoid full flushes).

**Page faults:**

When a PTE's present bit is 0, the CPU raises a page fault exception:

1. **Minor fault:** The page exists (e.g., COW, demand-zero, already in page cache) but
   isn't mapped. The kernel maps it and returns. No disk I/O.
2. **Major fault:** The page must be read from disk (e.g., memory-mapped file, swapped-out
   page). The kernel initiates disk I/O, blocks the process, and reschedules.
3. **Invalid fault:** The address isn't in any VMA. The kernel sends SIGSEGV
   (segmentation fault) to the process.

**Huge pages:**
- Instead of 4KB pages, use 2MB or 1GB pages.
- Fewer TLB entries needed, dramatically better TLB hit rate for large working sets.
- Configured via `hugetlbfs` or Transparent Huge Pages (THP).

---

### Q: What are namespaces and cgroups? How do they create containers?

**Answer:**

**Namespaces** provide isolation — they make a process think it has its own instance of
a global resource:

| Namespace | Isolates | Flag |
|-----------|----------|------|
| PID | Process IDs (init is PID 1 inside) | CLONE_NEWPID |
| Network | Network stack, interfaces, iptables, routes | CLONE_NEWNET |
| Mount | Filesystem mount points | CLONE_NEWNS |
| UTS | Hostname and domain name | CLONE_NEWUTS |
| IPC | System V IPC, POSIX message queues | CLONE_NEWIPC |
| User | UID/GID mappings (rootless containers) | CLONE_NEWUSER |
| Cgroup | Cgroup root directory | CLONE_NEWCGROUP |
| Time | System clocks (Linux 5.6+) | CLONE_NEWTIME |

**Cgroups** provide resource limiting and accounting:

- **cpu:** CPU time allocation (shares, quotas, periods)
- **memory:** Memory limits (hard limit, soft limit, swap limit, OOM control)
- **blkio/io:** Block I/O bandwidth and IOPS limits
- **pids:** Maximum number of processes
- **cpuset:** Pin processes to specific CPUs and memory nodes
- **devices:** Control access to device files

**How containers are built from these primitives:**

1. Create new namespaces (`clone()` or `unshare()`) for PID, network, mount, UTS, IPC, user.
2. Set up a root filesystem (container image) and `pivot_root()` into it.
3. Configure the network namespace (create veth pair, assign IP, set up routing).
4. Create a cgroup and move the container's processes into it; set resource limits.
5. Set the hostname (UTS namespace).
6. Drop capabilities, apply seccomp filters, set up AppArmor/SELinux profiles.
7. `exec()` the container's entrypoint process (becomes PID 1 inside the PID namespace).

A "container" is just a regular Linux process with namespaces and cgroups applied.
There is no kernel object called "container."

---

### Q: Explain the Linux I/O stack from userspace to disk

**Answer:**

```text
Application
    │  write(fd, buf, len)
    ▼
VFS (Virtual File System)
    │  Dispatches to specific filesystem
    ▼
Filesystem (ext4, XFS, btrfs, etc.)
    │  Maps file offsets to logical block numbers
    │  Manages metadata (inodes, directories, journals)
    ▼
Page Cache
    │  Buffers data in memory. write() returns here (for buffered I/O).
    │  Dirty pages are flushed by pdflush/writeback threads.
    ▼
Block Layer
    │  Merges, sorts, and schedules I/O requests (I/O scheduler: mq-deadline, bfq, none)
    │  Splits/merges BIOs, respects device queue limits
    ▼
Device Driver (SCSI, NVMe, virtio-blk, etc.)
    │  Translates block requests to device-specific commands
    ▼
Hardware (HDD, SSD, NVMe)
```

**Key details:**
- **Buffered I/O (default):** `write()` copies data to the page cache and returns
  immediately. Data is written to disk later by the kernel's writeback mechanism.
  `fsync()` forces an immediate flush.
- **Direct I/O (`O_DIRECT`):** Bypasses the page cache. Data goes directly from
  userspace buffer to the block layer. Used by databases for their own caching.
- **I/O schedulers:** `mq-deadline` (good for spinning disks), `none`/`noop` (best
  for NVMe/SSDs that have their own internal scheduling), `bfq` (fair queuing for
  desktop responsiveness).
- **Read-ahead:** The kernel pre-reads sequential data into the page cache to improve
  read throughput. Controlled by `blockdev --setra`.

---

## 2. Kubernetes Conceptual Questions

### Q: Explain the Kubernetes architecture — every component

**Answer:**

**Control Plane components:**

| Component | Role |
|-----------|------|
| **kube-apiserver** | The front door. All communication goes through it. RESTful API server that validates and persists resource state to etcd. Supports admission control, authentication, authorization. Horizontally scalable. |
| **etcd** | Distributed key-value store. Single source of truth for all cluster state. Uses Raft consensus for leader election and replication. All data is stored under `/registry/` prefix. |
| **kube-scheduler** | Watches for unscheduled Pods (`.spec.nodeName` is empty). Runs a filtering + scoring pipeline to select the best node. Considers: resource requests, affinity/anti-affinity, taints/tolerations, topology spread constraints, priority. |
| **kube-controller-manager** | Runs ~30+ controllers as goroutines in a single process: Deployment, ReplicaSet, Node, Job, EndpointSlice, ServiceAccount, Namespace, etc. Each controller implements a reconciliation loop: observe current state → compare to desired state → take action. |
| **cloud-controller-manager** | Cloud-specific controllers: Node (detects deleted VMs), Route (configures cloud routes), Service (provisions load balancers). Decouples cloud logic from core Kubernetes. |

**Node components:**

| Component | Role |
|-----------|------|
| **kubelet** | Agent on every node. Watches the API server for Pods assigned to its node. Manages Pod lifecycle via the Container Runtime Interface (CRI). Reports node status and pod status back. Runs liveness, readiness, and startup probes. |
| **kube-proxy** | Implements Service networking. Programs iptables/IPVS rules to load-balance traffic to Pod endpoints. Watches Services and EndpointSlices. |
| **Container runtime** | Actually runs containers. containerd and CRI-O are the most common. Interfaces with the kernel (namespaces, cgroups) via runc or similar OCI runtime. |

**Add-ons:**

| Add-on | Role |
|--------|------|
| **CoreDNS** | Cluster DNS server. Resolves `service.namespace.svc.cluster.local`. |
| **CNI plugin** | Container Network Interface. Provides pod-to-pod networking (Calico, Cilium, Flannel, etc.). |
| **Metrics Server** | Collects resource metrics from kubelets for `kubectl top` and HPA. |

---

### Q: What happens when you run `kubectl apply -f deployment.yaml`?

**Answer — the full path:**

1. **kubectl** reads `deployment.yaml`, validates it client-side, and sends an HTTP
   PATCH request (or POST for create) to the API server:
   `PATCH /apis/apps/v1/namespaces/default/deployments/my-deploy`

2. **Authentication:** API server authenticates the request (client cert, bearer token,
   OIDC, etc.).

3. **Authorization:** RBAC checks if this user/SA can perform this verb on this resource.

4. **Admission control:**
   - **Mutating admission webhooks** may modify the request (e.g., inject sidecar,
     set defaults, add labels).
   - **Object schema validation** and defaulting.
   - **Validating admission webhooks** may reject the request (e.g., policy enforcement).

5. **Persistence:** The API server writes the Deployment object to **etcd** via its
   storage layer.

6. **Deployment controller** (in kube-controller-manager) watches for Deployment changes.
   It sees the new/updated Deployment and creates or updates a **ReplicaSet** with the
   desired Pod template hash.

7. **ReplicaSet controller** watches ReplicaSets. It sees the new ReplicaSet needs N
   replicas and creates N **Pod** objects (without `.spec.nodeName`).

8. **Scheduler** watches for unscheduled Pods. For each Pod, it:
   - Filters nodes (resource fit, nodeSelector, affinity, taints).
   - Scores remaining nodes (spreading, resource balance, affinity preference).
   - Binds the Pod to the best node (sets `.spec.nodeName`).

9. **kubelet** on the selected node watches for Pods bound to it. It:
   - Pulls the container image (if not cached) via CRI.
   - Creates a Pod sandbox (pause container for shared network namespace).
   - Starts containers with resource limits (cgroups), namespaces, volume mounts.
   - Sets up networking via the CNI plugin.
   - Starts running probes.

10. **kube-proxy** watches Services and EndpointSlices. If the Pod matches a Service
    selector, the EndpointSlice controller adds the Pod's IP to the endpoint list.
    kube-proxy updates iptables/IPVS rules so traffic to the Service VIP reaches this Pod.

---

### Q: How does a Service work? Trace the packet

**Answer — Pod A → Service → Pod B (different nodes):**

Assume: ClusterIP Service `my-svc` (10.96.0.100:80) backed by Pod B (10.244.2.5:8080)
on Node 2. Pod A (10.244.1.3) is on Node 1.

1. **DNS resolution:** Pod A resolves `my-svc.default.svc.cluster.local` → 10.96.0.100
   (CoreDNS answers from its watch of Service objects).

2. **Pod A sends packet:** Source: 10.244.1.3:random → Destination: 10.96.0.100:80.

3. **iptables/IPVS on Node 1 (kube-proxy rules):** The packet hits the PREROUTING chain.
   kube-proxy's rules match destination 10.96.0.100:80 and DNAT the packet to one of the
   backend Pods (e.g., 10.244.2.5:8080). Load balancing is random (iptables) or
   round-robin/least-conn (IPVS). Now: Source: 10.244.1.3:random → Destination:
   10.244.2.5:8080.

4. **CNI routing:** The CNI plugin determines that 10.244.2.5 is on Node 2.
   - **Overlay (e.g., VXLAN/Flannel):** Encapsulates the packet in a UDP/VXLAN header,
     sends it to Node 2's host IP.
   - **Non-overlay (e.g., Calico BGP):** Uses BGP routes — the host routes the packet
     directly to Node 2 via the physical network.

5. **Node 2 receives the packet.** The CNI plugin decapsulates (if overlay) and routes
   to Pod B's veth interface.

6. **Pod B processes the request** and sends a reply. The reply follows the reverse path.
   conntrack on Node 1 ensures the SNAT/DNAT is reversed so Pod A sees the reply from
   10.96.0.100:80.

---

### Q: Deployment vs StatefulSet vs DaemonSet

**Answer:**

| Feature | Deployment | StatefulSet | DaemonSet |
|---------|-----------|-------------|-----------|
| **Use case** | Stateless apps (web servers, APIs) | Stateful apps (databases, Kafka, ZooKeeper) | Node-level agents (log collectors, monitoring, CNI) |
| **Pod identity** | Random names (deploy-abc-xyz) | Stable ordinal names (sts-0, sts-1, sts-2) | One pod per node |
| **Scaling** | Random order | Ordered (0→1→2 up, 2→1→0 down) | Automatic (one per node) |
| **Storage** | Shared or ephemeral | volumeClaimTemplates — each Pod gets its own PVC | Typically uses hostPath or local volumes |
| **Network identity** | None (use Service) | Headless Service provides stable DNS per pod (sts-0.svc) | Typically hostNetwork or hostPort |
| **Update strategy** | RollingUpdate, Recreate | RollingUpdate (ordered), OnDelete | RollingUpdate, OnDelete |
| **Pod replacement** | New random name | Same name, same PVC reattached | Rescheduled on same node |

---

### Q: How does DNS work in Kubernetes?

**Answer:**

**CoreDNS** runs as a Deployment in `kube-system`. It watches the API server for
Services and Endpoints.

**DNS records created automatically:**

| Record | Example | Resolves to |
|--------|---------|-------------|
| Service A record | `my-svc.default.svc.cluster.local` | ClusterIP (10.96.0.100) |
| Service SRV record | `_http._tcp.my-svc.default.svc.cluster.local` | Port + target |
| Headless Service | `my-svc.default.svc.cluster.local` | Set of Pod IPs |
| StatefulSet Pod | `sts-0.my-svc.default.svc.cluster.local` | Specific Pod IP |
| Pod A record | `10-244-1-3.default.pod.cluster.local` | Pod IP |

**Pod DNS configuration:**

Each Pod's `/etc/resolv.conf` is set by kubelet:
```
nameserver 10.96.0.10          # CoreDNS ClusterIP
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

The `ndots:5` setting means any name with fewer than 5 dots is treated as relative and
searched through the search domains. This is why `curl my-svc` works but generates
multiple DNS queries.

---

### Q: Explain RBAC in Kubernetes

**Answer:**

RBAC (Role-Based Access Control) has four key objects:

1. **Role** — namespaced set of permissions (verbs + resources):
   ```yaml
   kind: Role
   metadata:
     namespace: production
     name: pod-reader
   rules:
   - apiGroups: [""]
     resources: ["pods"]
     verbs: ["get", "list", "watch"]
   ```

2. **ClusterRole** — cluster-wide set of permissions (non-namespaced resources like
   nodes, or used as a template for multiple namespaces).

3. **RoleBinding** — binds a Role to a subject (User, Group, ServiceAccount) in a
   specific namespace.

4. **ClusterRoleBinding** — binds a ClusterRole to a subject cluster-wide.

**Subjects can be:**
- Users (external identity, no User object in K8s)
- Groups (from authentication provider)
- ServiceAccounts (namespaced K8s objects, used by Pods)

**Verbs:** get, list, watch, create, update, patch, delete, deletecollection

**Checking permissions:**
```bash
kubectl auth can-i create pods --namespace=production
kubectl auth can-i '*' '*'  # Am I cluster-admin?
kubectl auth can-i create pods --as=system:serviceaccount:default:my-sa
```

---

### Q: How does the scheduler decide where to place a Pod?

**Answer:**

The scheduler runs a two-phase pipeline:

**Phase 1 — Filtering (Predicates):**
Remove nodes that cannot run the Pod:
- `PodFitsResources`: Node has enough CPU/memory for the Pod's requests.
- `PodFitsHostPorts`: Requested host ports are available.
- `NodeSelector/NodeAffinity`: Pod's nodeSelector matches node labels.
- `TaintToleration`: Pod tolerates the node's taints.
- `PodTopologySpread`: Pod doesn't violate maxSkew constraints.
- `VolumeZone`: Required volumes are in the node's zone.
- Various others (disk pressure, PID pressure, unschedulable flag).

**Phase 2 — Scoring (Priorities):**
Score remaining nodes 0-100:
- `LeastRequestedPriority`: Prefer nodes with the most available resources.
- `BalancedResourceAllocation`: Prefer nodes with balanced CPU/memory usage.
- `InterPodAffinity`: Prefer nodes that satisfy affinity preferences.
- `NodeAffinity`: Prefer nodes matching preferred affinity terms.
- `ImageLocality`: Prefer nodes that already have the container image cached.
- `TaintToleration`: Prefer nodes with fewer matching taints.

The node with the highest total score wins. Ties are broken randomly.

---

### Q: What are admission webhooks and why do they matter?

**Answer:**

Admission webhooks intercept API requests AFTER authentication and authorization but
BEFORE persistence to etcd. There are two types:

1. **MutatingAdmissionWebhook:** Can modify the incoming object. Runs FIRST.
   - Use cases: inject sidecar containers (Istio), set default resource limits,
     add labels/annotations, inject environment variables.

2. **ValidatingAdmissionWebhook:** Can only accept or reject. Runs AFTER mutating.
   - Use cases: enforce security policies (no privileged containers), require labels,
     validate image registries (only allow from trusted registries), enforce resource
     quotas.

**How they work:**
- The API server sends an `AdmissionReview` request (JSON) to the webhook's HTTPS
  endpoint.
- The webhook responds with an `AdmissionReview` response (allowed: true/false,
  optional patch for mutating).
- `failurePolicy: Ignore` means if the webhook is unreachable, the request proceeds.
  `failurePolicy: Fail` means the request is rejected if the webhook is unreachable.

**Why they matter:** They're the primary extension mechanism for policy enforcement in
Kubernetes. Tools like OPA/Gatekeeper, Kyverno, and Istio rely heavily on them.

---

### Q: Explain the controller pattern and reconciliation loops

**Answer:**

The controller pattern is the core design pattern in Kubernetes:

```
                    ┌──────────┐
                    │  Desired  │
                    │   State   │ (stored in etcd via API server)
                    └─────┬─────┘
                          │ compare
                    ┌─────▼─────┐
                    │ Controller │ ◄── watch events (informers)
                    └─────┬─────┘
                          │ reconcile (take action)
                    ┌─────▼─────┐
                    │  Actual   │
                    │   State   │ (the real world)
                    └───────────┘
```

**Reconciliation loop:**

1. **Observe:** The controller watches the API server for changes to its resources
   (using informers and a work queue).
2. **Diff:** Compare the desired state (spec) with the actual state (status).
3. **Act:** Take the minimum action to move actual state toward desired state.
4. **Update status:** Report the current state back to the API server.

**Key properties:**
- **Level-triggered, not edge-triggered:** The controller reacts to the current state,
  not to events. If it misses an event, the next reconciliation still converges.
- **Idempotent:** Running the reconciliation loop twice produces the same result.
- **Eventually consistent:** The system converges to the desired state over time.

**Example — ReplicaSet controller:**
- Desired: 3 replicas. Actual: 2 running Pods.
- Action: Create 1 more Pod.
- If a Pod is manually deleted, the controller detects 2 < 3 and creates another.

---

## 3. System Design Questions

### Design a Container Orchestration Platform (Mini-Kubernetes)

**Requirements:**
- Schedule containers across a cluster of machines.
- Handle container failures (restart, reschedule).
- Service discovery and load balancing.
- Support declarative desired-state management.

**Components:**

```
┌─────────────────────────────────────────────────┐
│                 Control Plane                    │
│  ┌─────────┐  ┌───────────┐  ┌──────────────┐  │
│  │API Server│  │ Scheduler │  │  Controller  │  │
│  │  (HTTP)  │  │           │  │   Manager    │  │
│  └────┬─────┘  └─────┬─────┘  └──────┬───────┘  │
│       │              │               │           │
│       └──────────────┼───────────────┘           │
│                      │                           │
│              ┌───────▼────────┐                  │
│              │   State Store  │                  │
│              │  (etcd / BoltDB)                  │
│              └────────────────┘                  │
└─────────────────────────────────────────────────┘
         │                       │
    ┌────▼────┐            ┌────▼────┐
    │  Agent  │            │  Agent  │
    │ (Node1) │            │ (Node2) │
    │ Runtime │            │ Runtime │
    └─────────┘            └─────────┘
```

**Data flow:**
1. User submits desired state (e.g., "run 3 copies of nginx") via API.
2. API server validates and stores in the state store.
3. Scheduler watches for unassigned tasks, selects a node (bin-packing), assigns.
4. Agent on the selected node watches for tasks assigned to it, starts containers.
5. Controller watches for divergence (container died) and creates replacement tasks.

**Trade-offs:**
- Single etcd vs embedded KV (BoltDB): etcd is distributed but complex; BoltDB is
  simpler but single-node.
- Pull-based (agents poll) vs push-based (server pushes): Pull is simpler and more
  resilient to network partitions; push is lower latency.

---

### Design a Distributed Key-Value Store (etcd-like)

**Requirements:**
- Strong consistency (linearizable reads and writes).
- Fault tolerance (survive minority node failures).
- Watch support (clients notified of changes).
- Key-range queries and prefix listing.

**Components:**
- **Raft consensus module** — leader election, log replication.
- **WAL (Write-Ahead Log)** — durability before applying to state machine.
- **State machine** — in-memory B-tree or skiplist.
- **Snapshot module** — periodic snapshots to compact the WAL.
- **Watch hub** — event streaming to clients.
- **gRPC API** — client-facing interface.

**Data flow (write):**
1. Client sends Put(key, value) to the leader.
2. Leader appends entry to its WAL.
3. Leader replicates the entry to followers via AppendEntries RPC.
4. Once a majority acknowledges, the entry is committed.
5. Leader applies the entry to its state machine and responds to the client.
6. Followers apply the entry when they learn it's committed.

**Trade-offs:**
- Raft is easier to understand than Paxos but has a single leader bottleneck.
- In-memory state enables fast reads but limits data size to available RAM.
- Linearizable reads require either a leader lease or a read index; the former is
  faster, the latter is safer.

---

### Design a Log Aggregation System for Kubernetes

**Requirements:**
- Collect logs from all pods across all nodes.
- Support filtering, searching, and alerting.
- Handle high throughput with reasonable latency.
- Retain logs for configurable periods.

**Architecture:**

```
Pods → DaemonSet (log collector per node) → Message Queue → Indexer → Storage → Query API
        (Fluent Bit / Vector)               (Kafka)        (custom)  (S3+index) (HTTP)
```

**Components:**
1. **Collection:** DaemonSet on each node reads container logs from
   `/var/log/containers/*.log`. Adds metadata (pod name, namespace, labels).
2. **Transport:** Kafka (or similar) buffers and distributes log streams. Decouples
   producers from consumers. Provides backpressure handling.
3. **Indexing:** Consumer processes parse, transform, and index logs.
   Store indices in a search-optimized store (inverted index).
4. **Storage:** Raw logs in object storage (S3) for cost efficiency.
   Indices in faster storage (SSD/memory) for query performance.
5. **Query:** API server that queries indices and fetches from storage.
6. **Retention:** Background process that deletes expired data.

**Trade-offs:**
- Push vs pull collection: Push (Fluent Bit → Kafka) is simpler. Pull (Prometheus-style)
  is less common for logs.
- Structured vs unstructured: Structured (JSON) enables rich queries but requires
  applications to cooperate.
- Full-text search (Elasticsearch-like) vs log-specific (Loki label-based): Full-text
  is more flexible but 10-100x more expensive to index.

---

### Design a CI/CD Pipeline for Kubernetes Deployments

**Requirements:**
- Git-triggered builds and tests.
- Container image building and registry push.
- Progressive deployment (canary, blue-green).
- Rollback on failure.

**Components:**
1. **Source:** Git repository with webhook triggers.
2. **Build:** Build server runs tests, compiles, and builds container image.
3. **Registry:** Container image stored in OCI-compliant registry.
4. **Deploy:** GitOps controller (Argo CD-like) watches a Git repo for manifests.
5. **Progressive delivery:** Canary controller gradually shifts traffic.
6. **Monitoring:** Metrics-based analysis to decide promote/rollback.

**Data flow:**
1. Developer pushes code → webhook triggers build pipeline.
2. Pipeline runs lint, unit tests, integration tests.
3. Pipeline builds container image, tags with git SHA, pushes to registry.
4. Pipeline updates manifests in the GitOps repo (image tag update).
5. GitOps controller detects manifest change, applies to cluster.
6. Canary controller deploys new version to 5% of traffic.
7. Monitoring evaluates error rate and latency. If OK, promotes to 100%.
   If not, rolls back automatically.

**Trade-offs:**
- Push-based deploy (pipeline applies) vs pull-based (GitOps): GitOps is more secure
  (cluster pulls, no external write access) but adds complexity.
- Canary vs blue-green: Canary is gradual and catches issues early; blue-green is
  simpler but all-or-nothing.

---

### Design a Multi-Tenant Kubernetes Platform

**Requirements:**
- Multiple teams share a cluster with isolation.
- Cost attribution per tenant.
- Security boundaries between tenants.
- Self-service for tenants within guardrails.

**Architecture:**

Namespace-per-tenant model (most common):
1. **Namespace isolation:** Each tenant gets a namespace (or set of namespaces).
2. **RBAC:** Each tenant's ServiceAccount/Group has Role/RoleBinding scoped to their namespace.
3. **Network Policies:** Default-deny ingress/egress per namespace. Tenants can only
   communicate with explicitly allowed namespaces.
4. **Resource Quotas:** CPU, memory, storage, and object count limits per namespace.
5. **Limit Ranges:** Default and max resource requests/limits per container.
6. **Pod Security Standards:** Enforce restricted profile (no privileged, no host
   namespaces, no root).
7. **Admission webhooks:** Enforce image registry allowlist, require labels, prevent
   dangerous configurations.

**Trade-offs:**
- Namespace isolation vs virtual clusters vs separate clusters: Namespaces are cheapest
  but weakest isolation. Separate clusters are strongest but most expensive. Virtual
  clusters (vcluster) are a middle ground.
- Shared control plane is a single point of failure for all tenants.
- "Noisy neighbor" is mitigated by quotas but not eliminated.

---

### Design a Service Mesh from Scratch

**Requirements:**
- Transparent service-to-service communication.
- mTLS between all services.
- Observability (metrics, tracing, logging).
- Traffic management (retries, timeouts, circuit breaking, traffic splitting).

**Architecture:**

```
┌─────────────────────────┐
│      Control Plane      │
│  ┌─────┐ ┌──────┐      │
│  │ API │ │Config│      │
│  │     │ │ Push │      │
│  └──┬──┘ └──┬───┘      │
│     └────┬───┘          │
│     ┌────▼──────┐       │
│     │ CA / PKI  │       │
│     └───────────┘       │
└─────────┬───────────────┘
          │ xDS (config push)
    ┌─────▼─────┐    ┌───────────┐
    │  Sidecar  │───▶│  Sidecar  │
    │  Proxy    │    │  Proxy    │
    │ (Pod A)   │◄───│ (Pod B)   │
    └───────────┘    └───────────┘
```

**Components:**
1. **Sidecar proxy:** Injected into every Pod (via mutating webhook). Intercepts all
   inbound and outbound traffic via iptables rules. Handles mTLS, retries, timeouts,
   circuit breaking, load balancing.
2. **Control plane:** Pushes configuration to sidecars via xDS protocol. Manages
   certificates (issues short-lived certs, handles rotation). Provides policy API.
3. **Certificate Authority:** Issues mTLS certificates to sidecar proxies. Short-lived
   (hours) for security.

**Trade-offs:**
- Sidecar per pod adds latency (extra hop) and resource overhead.
- Sidecar vs per-node proxy: Per-node is more efficient but less isolated.
- xDS push vs pull: Push is lower latency but requires the control plane to track all
  connections.

---

### Design a Custom Autoscaler

**Requirements:**
- Scale based on custom metrics (queue depth, request latency).
- React faster than the default HPA (custom polling interval).
- Support scale-to-zero and scale-from-zero.
- Configurable per workload.

**Components:**
1. **Metrics collector:** Scrapes custom metrics from applications or message queues.
2. **Decision engine:** Applies scaling algorithms (target-based, rate-of-change).
3. **Actuator:** Patches the Deployment/StatefulSet replica count via the K8s API.
4. **Configuration:** CRD (CustomResourceDefinition) per workload specifying scaling rules.
5. **Cooldown manager:** Prevents flapping by enforcing cooldown periods between scale
   events.

**Scaling algorithms:**
- **Target-based:** desired = ceil(current_metric / target_metric × current_replicas).
- **Rate-of-change:** If metric is rising faster than threshold, scale proactively.
- **Predictive:** Use historical data to anticipate load spikes.

**Scale-to-zero:** Mark the workload as inactive after cooldown. An activator component
(like Knative's) holds incoming requests, triggers scale-from-zero, and forwards when
ready.

---

### Design a Cluster-Aware Job Scheduler

**Requirements:**
- Run batch jobs across a cluster with dependency management.
- Support priority, preemption, and fair sharing between teams.
- Handle job failures with retry and backoff.
- Gang scheduling (all pods of a job start together or none do).

**Components:**
1. **Job queue:** Priority queue of pending jobs, stored in etcd or a database.
2. **Scheduler:** Dequeues jobs, checks cluster capacity, assigns nodes.
3. **Worker agent:** Runs on each node, executes assigned tasks, reports status.
4. **Dependency resolver:** Builds a DAG of job dependencies, only enqueues jobs
   whose dependencies are complete.
5. **Preemption manager:** If a high-priority job can't fit, evicts lower-priority jobs.
6. **Fair-share calculator:** Allocates cluster capacity proportionally between teams.

**Trade-offs:**
- Gang scheduling requires either resource reservation or preemption, both complex.
- Preemption can cause cascading failures if not carefully managed.
- Centralized scheduler is simpler but can become a bottleneck; distributed scheduling
  (multi-scheduler) is harder to coordinate.

---

## 4. Coding Challenges (Go)

### Rate Limiter — Token Bucket

```go
// Package ratelimit provides a thread-safe token bucket rate limiter.
//
// A token bucket starts full and refills at a constant rate. Each request
// consumes one token. If the bucket is empty, the request is rejected.
//
// This is the most common rate-limiting algorithm used in production systems
// (e.g., API gateways, network traffic shaping).
package ratelimit

import (
	"sync"
	"time"
)

// TokenBucket implements the token bucket rate limiting algorithm.
// It is safe for concurrent use.
type TokenBucket struct {
	mu         sync.Mutex
	tokens     float64   // current number of tokens in the bucket
	maxTokens  float64   // maximum capacity of the bucket
	refillRate float64   // tokens added per second
	lastRefill time.Time // last time tokens were refilled
}

// NewTokenBucket creates a rate limiter that allows `rate` requests per second
// with a burst capacity of `burst`.
//
// The burst parameter controls how many requests can be made simultaneously
// when the bucket is full. The rate parameter controls the sustained request rate.
func NewTokenBucket(rate float64, burst int) *TokenBucket {
	return &TokenBucket{
		tokens:     float64(burst),
		maxTokens:  float64(burst),
		refillRate: rate,
		lastRefill: time.Now(),
	}
}

// Allow reports whether a single request is permitted at this instant.
// It consumes one token if available and returns true, or returns false
// if the bucket is empty. This method never blocks.
func (tb *TokenBucket) Allow() bool {
	tb.mu.Lock()
	defer tb.mu.Unlock()

	// Refill tokens based on elapsed time since last refill.
	now := time.Now()
	elapsed := now.Sub(tb.lastRefill).Seconds()
	tb.tokens += elapsed * tb.refillRate
	if tb.tokens > tb.maxTokens {
		tb.tokens = tb.maxTokens
	}
	tb.lastRefill = now

	// Try to consume one token.
	if tb.tokens >= 1.0 {
		tb.tokens--
		return true
	}
	return false
}

// SlidingWindowCounter implements the sliding window rate limiting algorithm.
// It divides time into fixed windows and interpolates between the previous
// and current window to provide smoother rate limiting than fixed windows.
type SlidingWindowCounter struct {
	mu          sync.Mutex
	limit       int           // max requests per window
	windowSize  time.Duration // size of each window
	prevCount   int           // request count in previous window
	currCount   int           // request count in current window
	windowStart time.Time     // start of current window
}

// NewSlidingWindowCounter creates a rate limiter that allows `limit` requests
// per `window` duration.
func NewSlidingWindowCounter(limit int, window time.Duration) *SlidingWindowCounter {
	return &SlidingWindowCounter{
		limit:       limit,
		windowSize:  window,
		windowStart: time.Now().Truncate(window),
	}
}

// Allow reports whether a request is permitted under the sliding window limit.
// The effective count is: prevCount * overlapFraction + currCount.
func (sw *SlidingWindowCounter) Allow() bool {
	sw.mu.Lock()
	defer sw.mu.Unlock()

	now := time.Now()
	windowStart := now.Truncate(sw.windowSize)

	// Advance windows if needed.
	if windowStart != sw.windowStart {
		elapsed := windowStart.Sub(sw.windowStart)
		if elapsed >= 2*sw.windowSize {
			// More than two windows have passed — reset everything.
			sw.prevCount = 0
			sw.currCount = 0
		} else {
			// Slide forward by one window.
			sw.prevCount = sw.currCount
			sw.currCount = 0
		}
		sw.windowStart = windowStart
	}

	// Calculate the weighted count using the overlap of the previous window.
	elapsedInWindow := now.Sub(sw.windowStart).Seconds()
	windowSecs := sw.windowSize.Seconds()
	overlapFraction := 1.0 - (elapsedInWindow / windowSecs)

	effectiveCount := float64(sw.prevCount)*overlapFraction + float64(sw.currCount)

	if effectiveCount < float64(sw.limit) {
		sw.currCount++
		return true
	}
	return false
}
```

---

### Connection Pool

```go
// Package pool provides a generic, thread-safe connection pool.
//
// Connections are lazily created up to a maximum size. Idle connections
// are reused. If the pool is exhausted, Get blocks until a connection
// is returned or the context is cancelled.
package pool

import (
	"context"
	"errors"
	"sync"
)

// Conn represents a poolable connection. Implementations must be safe
// to reuse after being returned to the pool.
type Conn interface {
	// Close permanently closes the underlying connection.
	Close() error

	// IsAlive checks if the connection is still usable.
	// The pool calls this before handing out a connection.
	IsAlive() bool
}

// Factory is a function that creates a new connection.
type Factory func(ctx context.Context) (Conn, error)

// Pool manages a pool of reusable connections.
type Pool struct {
	factory  Factory
	maxSize  int
	mu       sync.Mutex
	idle     []Conn     // idle connections available for reuse
	active   int        // total connections currently created (idle + in-use)
	waiters  []chan Conn // goroutines waiting for a connection
}

// New creates a connection pool with the given factory function and maximum size.
// The pool starts empty; connections are created lazily as needed.
func New(factory Factory, maxSize int) *Pool {
	return &Pool{
		factory: factory,
		maxSize: maxSize,
		idle:    make([]Conn, 0, maxSize),
	}
}

// Get retrieves a connection from the pool. It first tries to reuse an idle
// connection, then creates a new one if under the limit, and finally blocks
// until a connection is returned by another goroutine or the context is cancelled.
func (p *Pool) Get(ctx context.Context) (Conn, error) {
	p.mu.Lock()

	// Try to get an idle connection, verifying it's still alive.
	for len(p.idle) > 0 {
		conn := p.idle[len(p.idle)-1]
		p.idle = p.idle[:len(p.idle)-1]
		if conn.IsAlive() {
			p.mu.Unlock()
			return conn, nil
		}
		// Connection is dead — discard it and decrement count.
		p.active--
		conn.Close()
	}

	// If under the limit, create a new connection.
	if p.active < p.maxSize {
		p.active++
		p.mu.Unlock()

		conn, err := p.factory(ctx)
		if err != nil {
			p.mu.Lock()
			p.active--
			p.mu.Unlock()
			return nil, err
		}
		return conn, nil
	}

	// Pool is exhausted — wait for a connection to be returned.
	waiter := make(chan Conn, 1)
	p.waiters = append(p.waiters, waiter)
	p.mu.Unlock()

	select {
	case conn := <-waiter:
		return conn, nil
	case <-ctx.Done():
		// Remove ourselves from the waiters list.
		p.mu.Lock()
		for i, w := range p.waiters {
			if w == waiter {
				p.waiters = append(p.waiters[:i], p.waiters[i+1:]...)
				break
			}
		}
		p.mu.Unlock()
		// Check if a connection arrived concurrently.
		select {
		case conn := <-waiter:
			p.Put(conn) // Return it so someone else can use it.
		default:
		}
		return nil, ctx.Err()
	}
}

// Put returns a connection to the pool. If there are goroutines waiting,
// the connection is handed directly to the longest-waiting goroutine.
func (p *Pool) Put(conn Conn) {
	if conn == nil {
		return
	}

	p.mu.Lock()
	defer p.mu.Unlock()

	// If anyone is waiting, hand the connection directly to them.
	if len(p.waiters) > 0 {
		waiter := p.waiters[0]
		p.waiters = p.waiters[1:]
		waiter <- conn
		return
	}

	// Otherwise, return to idle pool.
	p.idle = append(p.idle, conn)
}

// Close drains the pool, closing all idle connections.
func (p *Pool) Close() error {
	p.mu.Lock()
	defer p.mu.Unlock()

	var firstErr error
	for _, conn := range p.idle {
		if err := conn.Close(); err != nil && firstErr == nil {
			firstErr = err
		}
	}
	p.idle = nil

	// Signal all waiters that the pool is closed.
	for _, waiter := range p.waiters {
		close(waiter)
	}
	p.waiters = nil

	return firstErr
}

// Stats returns current pool statistics.
func (p *Pool) Stats() (active, idle, waiting int) {
	p.mu.Lock()
	defer p.mu.Unlock()
	return p.active, len(p.idle), len(p.waiters)
}

var ErrPoolClosed = errors.New("pool: closed")
```

---

### Simple Load Balancer

```go
// Package lb provides a configurable HTTP load balancer with multiple strategies.
//
// It supports round-robin, least-connections, and random backend selection.
// Backends are health-checked periodically and removed from rotation when unhealthy.
package lb

import (
	"context"
	"math/rand"
	"net/http"
	"net/http/httputil"
	"net/url"
	"sync"
	"sync/atomic"
	"time"
)

// Backend represents a single upstream server.
type Backend struct {
	URL         *url.URL
	Alive       atomic.Bool
	ActiveConns atomic.Int64
	Proxy       *httputil.ReverseProxy
}

// Strategy defines the load balancing algorithm.
type Strategy int

const (
	RoundRobin Strategy = iota
	LeastConnections
	Random
)

// LoadBalancer distributes HTTP requests across a set of backends.
type LoadBalancer struct {
	backends []*Backend
	strategy Strategy
	current  atomic.Uint64 // round-robin counter
	mu       sync.RWMutex
}

// NewLoadBalancer creates a load balancer with the given backend URLs and strategy.
// It starts background health checks for each backend.
func NewLoadBalancer(urls []string, strategy Strategy) (*LoadBalancer, error) {
	var backends []*Backend

	for _, rawURL := range urls {
		u, err := url.Parse(rawURL)
		if err != nil {
			return nil, err
		}

		proxy := httputil.NewSingleHostReverseProxy(u)
		b := &Backend{
			URL:   u,
			Proxy: proxy,
		}
		b.Alive.Store(true)
		backends = append(backends, b)
	}

	lb := &LoadBalancer{
		backends: backends,
		strategy: strategy,
	}

	// Start health checking for all backends.
	for _, b := range backends {
		go lb.healthCheck(b)
	}

	return lb, nil
}

// ServeHTTP handles incoming HTTP requests by forwarding them to a selected backend.
func (lb *LoadBalancer) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	backend := lb.selectBackend()
	if backend == nil {
		http.Error(w, "Service Unavailable: no healthy backends", http.StatusServiceUnavailable)
		return
	}

	backend.ActiveConns.Add(1)
	defer backend.ActiveConns.Add(-1)

	backend.Proxy.ServeHTTP(w, r)
}

// selectBackend picks a healthy backend using the configured strategy.
func (lb *LoadBalancer) selectBackend() *Backend {
	alive := lb.aliveBackends()
	if len(alive) == 0 {
		return nil
	}

	switch lb.strategy {
	case RoundRobin:
		idx := lb.current.Add(1) - 1
		return alive[idx%uint64(len(alive))]

	case LeastConnections:
		var best *Backend
		var minConns int64 = 1<<63 - 1
		for _, b := range alive {
			conns := b.ActiveConns.Load()
			if conns < minConns {
				minConns = conns
				best = b
			}
		}
		return best

	case Random:
		return alive[rand.Intn(len(alive))]

	default:
		return alive[0]
	}
}

// aliveBackends returns all backends currently marked as healthy.
func (lb *LoadBalancer) aliveBackends() []*Backend {
	var result []*Backend
	for _, b := range lb.backends {
		if b.Alive.Load() {
			result = append(result, b)
		}
	}
	return result
}

// healthCheck periodically checks if a backend is reachable by sending
// an HTTP GET to its root path. Backends are marked alive or dead based
// on the response.
func (lb *LoadBalancer) healthCheck(b *Backend) {
	ticker := time.NewTicker(10 * time.Second)
	defer ticker.Stop()

	client := &http.Client{Timeout: 5 * time.Second}

	for range ticker.C {
		ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
		req, _ := http.NewRequestWithContext(ctx, http.MethodGet, b.URL.String()+"/healthz", nil)

		resp, err := client.Do(req)
		cancel()

		if err != nil || resp.StatusCode >= 500 {
			b.Alive.Store(false)
		} else {
			b.Alive.Store(true)
		}
		if resp != nil {
			resp.Body.Close()
		}
	}
}
```

---

### Leader Election

```go
// Package election implements a simple leader election algorithm using
// a distributed lock (simulated here with a shared store interface).
//
// In production, this would use etcd leases, Kubernetes Lease objects,
// or a similar distributed coordination primitive.
package election

import (
	"context"
	"fmt"
	"log"
	"sync"
	"time"
)

// Store is the interface for a distributed key-value store that supports
// atomic compare-and-swap operations. In production, this would be backed
// by etcd, ZooKeeper, or a Kubernetes Lease object.
type Store interface {
	// TryAcquire attempts to set the key to value if it is currently unset
	// or expired. Returns true if the lock was acquired.
	TryAcquire(ctx context.Context, key, value string, ttl time.Duration) (bool, error)

	// Renew extends the TTL of a lock held by the given value.
	// Returns true if the renewal was successful (the caller still holds the lock).
	Renew(ctx context.Context, key, value string, ttl time.Duration) (bool, error)

	// Release deletes the lock if it is held by the given value.
	Release(ctx context.Context, key, value string) error

	// Get returns the current holder of the lock, or empty string if no one holds it.
	Get(ctx context.Context, key string) (string, error)
}

// LeaderElector manages leader election for a single participant.
type LeaderElector struct {
	store      Store
	lockKey    string
	identity   string        // unique identity of this participant
	leaseTTL   time.Duration // how long the lease lasts
	renewEvery time.Duration // how often to renew (must be < leaseTTL)

	mu       sync.Mutex
	isLeader bool
	onStart  func(ctx context.Context) // called when we become leader
	onStop   func()                    // called when we lose leadership
}

// Config holds configuration for the leader elector.
type Config struct {
	Store      Store
	LockKey    string
	Identity   string
	LeaseTTL   time.Duration
	RenewEvery time.Duration
	OnStarted  func(ctx context.Context) // called when this instance becomes leader
	OnStopped  func()                    // called when this instance loses leadership
}

// NewLeaderElector creates a new leader elector with the given configuration.
func NewLeaderElector(cfg Config) *LeaderElector {
	if cfg.RenewEvery >= cfg.LeaseTTL {
		cfg.RenewEvery = cfg.LeaseTTL / 3
	}
	return &LeaderElector{
		store:      cfg.Store,
		lockKey:    cfg.LockKey,
		identity:   cfg.Identity,
		leaseTTL:   cfg.LeaseTTL,
		renewEvery: cfg.RenewEvery,
		onStart:    cfg.OnStarted,
		onStop:     cfg.OnStopped,
	}
}

// Run starts the leader election loop. It blocks until the context is cancelled.
// When this instance becomes the leader, it calls OnStarted in a goroutine.
// When it loses leadership, it calls OnStopped.
func (le *LeaderElector) Run(ctx context.Context) error {
	ticker := time.NewTicker(le.renewEvery)
	defer ticker.Stop()

	var leaderCtx context.Context
	var leaderCancel context.CancelFunc

	for {
		select {
		case <-ctx.Done():
			le.mu.Lock()
			wasLeader := le.isLeader
			le.isLeader = false
			le.mu.Unlock()

			if wasLeader {
				if leaderCancel != nil {
					leaderCancel()
				}
				le.store.Release(context.Background(), le.lockKey, le.identity)
				if le.onStop != nil {
					le.onStop()
				}
			}
			return ctx.Err()

		case <-ticker.C:
			le.mu.Lock()
			currentlyLeader := le.isLeader
			le.mu.Unlock()

			if currentlyLeader {
				// Try to renew the lease.
				renewed, err := le.store.Renew(ctx, le.lockKey, le.identity, le.leaseTTL)
				if err != nil || !renewed {
					log.Printf("[%s] lost leadership: err=%v renewed=%v", le.identity, err, renewed)
					le.mu.Lock()
					le.isLeader = false
					le.mu.Unlock()
					if leaderCancel != nil {
						leaderCancel()
					}
					if le.onStop != nil {
						le.onStop()
					}
				}
			} else {
				// Try to acquire the lock.
				acquired, err := le.store.TryAcquire(ctx, le.lockKey, le.identity, le.leaseTTL)
				if err != nil {
					log.Printf("[%s] failed to acquire lock: %v", le.identity, err)
					continue
				}
				if acquired {
					log.Printf("[%s] became leader", le.identity)
					le.mu.Lock()
					le.isLeader = true
					le.mu.Unlock()

					leaderCtx, leaderCancel = context.WithCancel(ctx)
					if le.onStart != nil {
						go le.onStart(leaderCtx)
					}
				}
			}
		}
	}
}

// IsLeader returns whether this instance currently holds the leader lock.
func (le *LeaderElector) IsLeader() bool {
	le.mu.Lock()
	defer le.mu.Unlock()
	return le.isLeader
}

// String returns a human-readable description of this elector.
func (le *LeaderElector) String() string {
	return fmt.Sprintf("LeaderElector{identity=%s, key=%s, leader=%v}", le.identity, le.lockKey, le.IsLeader())
}
```

---

### Pub/Sub System

```go
// Package pubsub provides an in-process publish/subscribe message broker.
//
// Topics are created on demand. Subscribers receive messages published after
// they subscribe. Each subscriber gets its own buffered channel so a slow
// subscriber doesn't block others.
package pubsub

import (
	"context"
	"sync"
)

// Message represents a published message.
type Message struct {
	Topic   string
	Payload interface{}
}

// Subscriber receives messages from subscribed topics.
type Subscriber struct {
	id     string
	ch     chan Message
	topics map[string]bool
	mu     sync.RWMutex
	closed bool
}

// Ch returns the channel on which messages are delivered to this subscriber.
// Read from this channel in a loop to receive messages.
func (s *Subscriber) Ch() <-chan Message {
	return s.ch
}

// Broker is the central message broker that manages topics and subscribers.
type Broker struct {
	mu          sync.RWMutex
	subscribers map[string][]*Subscriber // topic -> subscribers
	bufferSize  int
	nextID      int
}

// NewBroker creates a new message broker. The bufferSize parameter controls
// the channel buffer size for each subscriber; if a subscriber falls behind
// by more than bufferSize messages, new messages to that subscriber are dropped.
func NewBroker(bufferSize int) *Broker {
	return &Broker{
		subscribers: make(map[string][]*Subscriber),
		bufferSize:  bufferSize,
	}
}

// Subscribe creates a new subscriber for the given topics.
// The subscriber will receive all messages published to any of these topics
// after this call returns.
func (b *Broker) Subscribe(topics ...string) *Subscriber {
	b.mu.Lock()
	defer b.mu.Unlock()

	b.nextID++
	sub := &Subscriber{
		id:     fmt.Sprintf("sub-%d", b.nextID),
		ch:     make(chan Message, b.bufferSize),
		topics: make(map[string]bool),
	}

	for _, topic := range topics {
		sub.topics[topic] = true
		b.subscribers[topic] = append(b.subscribers[topic], sub)
	}

	return sub
}

// Unsubscribe removes a subscriber from all its topics and closes its channel.
func (b *Broker) Unsubscribe(sub *Subscriber) {
	b.mu.Lock()
	defer b.mu.Unlock()

	sub.mu.Lock()
	defer sub.mu.Unlock()

	if sub.closed {
		return
	}

	for topic := range sub.topics {
		subs := b.subscribers[topic]
		for i, s := range subs {
			if s == sub {
				b.subscribers[topic] = append(subs[:i], subs[i+1:]...)
				break
			}
		}
		// Clean up empty topic entries.
		if len(b.subscribers[topic]) == 0 {
			delete(b.subscribers, topic)
		}
	}

	sub.closed = true
	close(sub.ch)
}

// Publish sends a message to all subscribers of the given topic.
// If a subscriber's buffer is full, the message is dropped for that subscriber
// (non-blocking send). Returns the number of subscribers that received the message.
func (b *Broker) Publish(topic string, payload interface{}) int {
	b.mu.RLock()
	subs := b.subscribers[topic]
	b.mu.RUnlock()

	msg := Message{Topic: topic, Payload: payload}
	delivered := 0

	for _, sub := range subs {
		sub.mu.RLock()
		if sub.closed {
			sub.mu.RUnlock()
			continue
		}
		sub.mu.RUnlock()

		// Non-blocking send — drop message if subscriber is too slow.
		select {
		case sub.ch <- msg:
			delivered++
		default:
			// Subscriber's buffer is full; message dropped.
		}
	}

	return delivered
}

// PublishWithContext is like Publish but respects context cancellation.
// It blocks until the message is delivered to all subscribers or the context
// is cancelled.
func (b *Broker) PublishWithContext(ctx context.Context, topic string, payload interface{}) int {
	b.mu.RLock()
	subs := b.subscribers[topic]
	b.mu.RUnlock()

	msg := Message{Topic: topic, Payload: payload}
	delivered := 0

	for _, sub := range subs {
		sub.mu.RLock()
		if sub.closed {
			sub.mu.RUnlock()
			continue
		}
		sub.mu.RUnlock()

		select {
		case sub.ch <- msg:
			delivered++
		case <-ctx.Done():
			return delivered
		}
	}

	return delivered
}

// Topics returns the list of topics that have at least one subscriber.
func (b *Broker) Topics() []string {
	b.mu.RLock()
	defer b.mu.RUnlock()

	topics := make([]string, 0, len(b.subscribers))
	for topic := range b.subscribers {
		topics = append(topics, topic)
	}
	return topics
}

// Note: This package requires "fmt" in its import block for subscriber ID generation.
```

---

### Circuit Breaker

```go
// Package circuitbreaker implements the circuit breaker pattern for resilient
// service-to-service communication.
//
// The circuit breaker has three states:
//   - Closed: Requests flow normally. Failures are counted.
//   - Open: Requests are immediately rejected. After a timeout, moves to HalfOpen.
//   - HalfOpen: A limited number of probe requests are allowed. If they succeed,
//     the circuit closes. If they fail, it opens again.
//
// This prevents cascading failures when a downstream service is unhealthy.
package circuitbreaker

import (
	"context"
	"errors"
	"sync"
	"time"
)

// State represents the current state of the circuit breaker.
type State int

const (
	StateClosed   State = iota // normal operation
	StateOpen                  // failing fast, rejecting requests
	StateHalfOpen              // testing if downstream has recovered
)

// String returns the human-readable state name.
func (s State) String() string {
	switch s {
	case StateClosed:
		return "closed"
	case StateOpen:
		return "open"
	case StateHalfOpen:
		return "half-open"
	default:
		return "unknown"
	}
}

var (
	// ErrCircuitOpen is returned when the circuit breaker is in the Open state
	// and requests are being rejected without being attempted.
	ErrCircuitOpen = errors.New("circuit breaker is open")
)

// Settings configures the circuit breaker behavior.
type Settings struct {
	// MaxFailures is the number of consecutive failures in the Closed state
	// before the circuit opens.
	MaxFailures int

	// Timeout is how long the circuit stays open before transitioning to HalfOpen.
	Timeout time.Duration

	// HalfOpenMaxRequests is the number of requests allowed in the HalfOpen state
	// to probe whether the downstream has recovered.
	HalfOpenMaxRequests int
}

// CircuitBreaker wraps a function call with circuit breaker logic.
type CircuitBreaker struct {
	settings Settings

	mu              sync.Mutex
	state           State
	failures        int       // consecutive failure count in Closed state
	lastFailureTime time.Time // when the circuit was opened
	halfOpenCount   int       // requests attempted in HalfOpen state
}

// New creates a new circuit breaker with the given settings.
func New(settings Settings) *CircuitBreaker {
	if settings.MaxFailures <= 0 {
		settings.MaxFailures = 5
	}
	if settings.Timeout <= 0 {
		settings.Timeout = 30 * time.Second
	}
	if settings.HalfOpenMaxRequests <= 0 {
		settings.HalfOpenMaxRequests = 1
	}
	return &CircuitBreaker{
		settings: settings,
		state:    StateClosed,
	}
}

// Execute runs the given function through the circuit breaker.
//
// In the Closed state, all requests are attempted. If failures exceed
// MaxFailures, the circuit opens.
//
// In the Open state, requests are immediately rejected with ErrCircuitOpen.
// After the Timeout period, the state transitions to HalfOpen.
//
// In the HalfOpen state, a limited number of probe requests are allowed.
// If a probe succeeds, the circuit closes. If it fails, the circuit opens again.
func (cb *CircuitBreaker) Execute(ctx context.Context, fn func(ctx context.Context) error) error {
	if err := cb.beforeRequest(); err != nil {
		return err
	}

	// Execute the actual function.
	err := fn(ctx)

	cb.afterRequest(err)
	return err
}

// beforeRequest checks whether the request should be allowed through.
func (cb *CircuitBreaker) beforeRequest() error {
	cb.mu.Lock()
	defer cb.mu.Unlock()

	switch cb.state {
	case StateClosed:
		return nil

	case StateOpen:
		// Check if timeout has elapsed — if so, transition to HalfOpen.
		if time.Since(cb.lastFailureTime) >= cb.settings.Timeout {
			cb.state = StateHalfOpen
			cb.halfOpenCount = 0
			return nil
		}
		return ErrCircuitOpen

	case StateHalfOpen:
		// Allow a limited number of probe requests.
		if cb.halfOpenCount >= cb.settings.HalfOpenMaxRequests {
			return ErrCircuitOpen
		}
		cb.halfOpenCount++
		return nil
	}

	return nil
}

// afterRequest updates the circuit breaker state based on the result.
func (cb *CircuitBreaker) afterRequest(err error) {
	cb.mu.Lock()
	defer cb.mu.Unlock()

	if err != nil {
		// Request failed.
		switch cb.state {
		case StateClosed:
			cb.failures++
			if cb.failures >= cb.settings.MaxFailures {
				cb.state = StateOpen
				cb.lastFailureTime = time.Now()
			}
		case StateHalfOpen:
			// Probe failed — reopen the circuit.
			cb.state = StateOpen
			cb.lastFailureTime = time.Now()
		}
	} else {
		// Request succeeded.
		switch cb.state {
		case StateClosed:
			cb.failures = 0 // reset failure count on success
		case StateHalfOpen:
			// Probe succeeded — close the circuit.
			cb.state = StateClosed
			cb.failures = 0
			cb.halfOpenCount = 0
		}
	}
}

// State returns the current state of the circuit breaker.
func (cb *CircuitBreaker) State() State {
	cb.mu.Lock()
	defer cb.mu.Unlock()
	return cb.state
}

// Failures returns the current consecutive failure count.
func (cb *CircuitBreaker) Failures() int {
	cb.mu.Lock()
	defer cb.mu.Unlock()
	return cb.failures
}

// Reset manually resets the circuit breaker to the Closed state.
// Use this with caution — typically the circuit breaker should manage its own state.
func (cb *CircuitBreaker) Reset() {
	cb.mu.Lock()
	defer cb.mu.Unlock()
	cb.state = StateClosed
	cb.failures = 0
	cb.halfOpenCount = 0
}
```

---

## 5. Behavioral / Systems Thinking

### Q: How would you troubleshoot a Pod stuck in CrashLoopBackOff?

**Answer — systematic approach:**

1. **Understand the symptom:** CrashLoopBackOff means the container starts, crashes,
   kubelet restarts it, it crashes again, and the backoff delay grows exponentially.

2. **Check Pod events:**
   ```bash
   kubectl describe pod <name>
   ```
   Look at the Events section: OOMKilled? ImagePullBackOff? FailedMount?

3. **Check container logs:**
   ```bash
   kubectl logs <pod> --previous   # Logs from the crashed container
   ```
   Look for application-level errors, stack traces, missing config.

4. **Check resource limits:**
   ```bash
   kubectl describe pod <name> | grep -A5 Limits
   ```
   If OOMKilled: container is exceeding its memory limit. Increase limits or fix the
   memory leak.

5. **Check the image:**
   Is the image correct? Does the entrypoint exist? Try running it locally:
   ```bash
   docker run --rm -it <image> /bin/sh
   ```

6. **Check ConfigMaps/Secrets:** Are all referenced ConfigMaps and Secrets present?
   Missing ones cause mount failures.

7. **Check liveness probes:** An aggressive liveness probe can kill a slow-starting
   container. Check `initialDelaySeconds` and `failureThreshold`.

8. **Check volume mounts:** Permissions issues, missing PVCs, wrong paths.

9. **Debug with ephemeral container:**
   ```bash
   kubectl debug <pod> -it --image=busybox --target=<container>
   ```

---

### Q: A service is intermittently timing out — how do you debug?

**Answer — structured debugging:**

1. **Scope the problem:** Which endpoints? Which clients? What percentage of requests?
   Check metrics (latency percentiles: p50 vs p99).

2. **Client side:**
   - Check DNS resolution time: `dig +stats my-svc.namespace.svc.cluster.local`
   - Check if the timeout is hitting a specific Pod or random:
     ```bash
     kubectl logs -l app=my-svc --all-containers | grep -i timeout
     ```

3. **Network path:**
   - Test connectivity: `kubectl exec debug -- curl -w '%{time_total}' http://my-svc/health`
   - Check for packet loss: `kubectl exec debug -- ping -c 100 <pod-ip>`
   - Check kube-proxy / IPVS rules: Are endpoints up to date?
     ```bash
     kubectl get endpoints my-svc
     ```
   - Check for network policy blocking traffic.

4. **Service side:**
   - Check Pod resource usage: `kubectl top pods -l app=my-svc`
   - CPU throttling? Check cgroup stats:
     ```bash
     kubectl exec <pod> -- cat /sys/fs/cgroup/cpu/cpu.stat
     ```
   - Thread pool exhaustion? Connection pool exhaustion? Check application metrics.
   - Is garbage collection causing pauses? Check GC logs.

5. **Downstream dependencies:**
   - The service might be waiting on a database or external API.
   - Check if the downstream is healthy.
   - Add timeout and circuit breaker to downstream calls.

6. **Common culprits:**
   - DNS resolution latency (ndots:5 causes multiple DNS queries).
   - CPU throttling (request too low, limit OK).
   - Connection pool exhaustion.
   - Kernel conntrack table full (`dmesg | grep conntrack`).
   - NAT/SNAT port exhaustion for external calls.

---

### Q: How do you perform a zero-downtime migration of a database?

**Answer — the expand/contract pattern:**

**Phase 1: Dual-write setup**
1. Deploy a new database alongside the old one.
2. Modify the application to write to BOTH databases (old and new).
3. The old database remains the source of truth for reads.

**Phase 2: Backfill**
1. Copy all existing data from old database to new database.
2. Verify data consistency between old and new.
3. Continue dual-writing during the backfill.

**Phase 3: Shadow reads**
1. Read from both databases and compare results (log discrepancies).
2. Fix any inconsistencies found.
3. Build confidence that the new database returns correct results.

**Phase 4: Cut over reads**
1. Switch reads to the new database.
2. Continue dual-writing as a safety net.
3. Monitor error rates and latency closely.

**Phase 5: Stop dual-write**
1. Once confident, stop writing to the old database.
2. Keep the old database running for a rollback window.
3. After the rollback window, decommission the old database.

**Key considerations:**
- Each phase should be controlled by a feature flag so you can quickly rollback.
- Schema changes should use the expand/contract pattern: add new columns first, migrate
  data, then remove old columns.
- For large databases, the backfill should be incremental and throttled.
- Test the full procedure in staging first.

---

### Q: How do you handle secrets rotation without downtime?

**Answer:**

**The key principle:** At any given moment, both the old and new secret must be valid.

**Approach 1: Overlapping validity windows**

1. Generate the new secret while keeping the old one valid.
2. Update the application configuration to accept BOTH old and new secrets.
   For API keys: accept both keys for authentication.
   For database passwords: add the new password to the DB, then update the app.
   For TLS certificates: the new cert is valid before the old one expires.
3. Roll out the application update (rolling deployment — some pods use old, some new).
4. Once all pods are using the new secret, revoke the old one.

**Approach 2: Kubernetes native**

```bash
# Update the Secret
kubectl create secret generic db-creds \
  --from-literal=password=new_password \
  --dry-run=client -o yaml | kubectl apply -f -

# If using env vars: requires pod restart (rolling)
kubectl rollout restart deployment/my-app

# If using volume mounts: kubelet auto-updates the projected volume
# (within kubelet sync period, typically ~1 minute)
# Application must watch the file and reload.
```

**Approach 3: External secrets manager**

Use HashiCorp Vault, AWS Secrets Manager, or similar:
1. Application fetches secrets at runtime with a short TTL.
2. Rotate the secret in the secrets manager.
3. Applications automatically pick up the new secret on next refresh.
4. The secrets manager handles the overlapping validity window.

**Key considerations:**
- Database credential rotation: Create a new user, update app, delete old user.
  Or: Use Vault's dynamic secrets (creates temporary credentials on demand).
- TLS certificate rotation: Use cert-manager in Kubernetes for automatic renewal.
- Always have monitoring/alerting on authentication failures during rotation.

---

*These questions and answers reflect the depth expected in senior systems programming
and infrastructure engineering interviews. Practice explaining them out loud — the
ability to walk through complex systems clearly is as important as knowing the answer.*
