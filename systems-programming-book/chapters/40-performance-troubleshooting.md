# Chapter 40: Performance Tuning & Troubleshooting

> *"In production, the question is never whether something will go wrong —
> it's how quickly you can find and fix it when it does."*

This chapter is your field guide. It covers every tool, technique, and mental
model you need to debug Kubernetes clusters and Go applications in production.
We work from the bottom up — nodes, pods, networking — then shift into Go-specific
profiling with pprof, benchmarking, Kubernetes performance tuning, resource
right-sizing, common troubleshooting scenarios, and finally chaos engineering.

Keep this chapter bookmarked. You will return to it again and again.

---

## 40.1 Node-Level Debugging

When a cluster misbehaves, start at the node level. Nodes are the physical (or
virtual) machines that run your workloads. If the node is unhealthy, nothing
running on it will be healthy either.

### 40.1.1 SSH into a Node (or kubectl debug node)

**Traditional SSH:**

```bash
# Find the node's external IP
kubectl get nodes -o wide
# NAME         STATUS   ROLES           INTERNAL-IP   EXTERNAL-IP
# node-1       Ready    <none>          10.0.1.10     34.56.78.90

ssh -i ~/.ssh/cluster-key ubuntu@34.56.78.90
```

**Modern approach — kubectl debug node:**

```bash
# This creates a privileged pod on the node with host PID, network, and filesystem
kubectl debug node/node-1 -it --image=ubuntu:22.04

# Inside the debug pod, the host filesystem is mounted at /host
chroot /host

# Now you have full access to the node as if you SSH'd in
systemctl status kubelet
journalctl -u kubelet --no-pager -n 50
```

The `kubectl debug node` approach is superior because:
- No SSH keys to manage.
- Works with managed Kubernetes (EKS, GKE, AKS) where SSH may be disabled.
- Audit-logged through the Kubernetes API.
- Automatically cleaned up when you exit.

### 40.1.2 System Tools: top, htop, vmstat, iostat, sar, dmesg

**top / htop — Process-level CPU and memory:**

```bash
# top: quick overview of system load
top -bn1 | head -20
# top - 14:30:00 up 45 days,  3:22,  1 user,  load average: 4.52, 3.81, 2.15
# Tasks: 312 total,   3 running, 309 sleeping
# %Cpu(s): 45.2 us,  8.1 sy,  0.0 ni, 42.3 id,  3.1 wa,  0.0 hi,  1.3 si
# MiB Mem:  32168.0 total,   2340.5 free,  24512.8 used,   5314.7 buff/cache

# htop: interactive, color-coded, tree view
htop
# Press F5 for tree view (shows parent-child process relationships)
# Press F6 to sort by CPU%, MEM%, etc.
# Press F4 to filter by process name
```

**Key metrics to watch in top:**

| Field          | What It Means                                     | Warning Threshold    |
|----------------|---------------------------------------------------|-----------------------|
| `load average` | Number of processes waiting for CPU (1/5/15 min)  | > number of CPU cores |
| `%wa`          | CPU time spent waiting for I/O                    | > 10%                |
| `%si`          | CPU time in software interrupts (network)         | > 5%                 |
| `MiB Mem used` | Total memory in use                               | > 90% of total       |
| `MiB Swap used`| Swap usage (indicates memory pressure)            | Any significant use  |

**vmstat — Virtual memory statistics:**

```bash
# Report every 2 seconds, 5 iterations
vmstat 2 5
# procs -----------memory---------- ---swap-- -----io---- -system-- ------cpu-----
#  r  b   swpd   free   buff  cache   si   so    bi    bo   in   cs us sy id wa st
#  2  0      0 2340508 184320 5260800  0    0    12   156  3200 5400 45  8 42  3  2
```

**Key vmstat columns:**

| Column | Meaning                            | Action If High                      |
|--------|------------------------------------|--------------------------------------|
| `r`    | Processes waiting for CPU          | Add CPU or reduce workload          |
| `b`    | Processes blocked on I/O           | Investigate disk performance        |
| `si/so`| Swap in/out                        | Add memory or reduce usage          |
| `wa`   | CPU waiting for I/O                | Faster disks, reduce I/O load       |
| `cs`   | Context switches per second        | May indicate too many small processes|

**iostat — Disk I/O statistics:**

```bash
# Show extended statistics for all devices, every 2 seconds
iostat -xz 2
# Device   r/s     w/s   rMB/s   wMB/s  rrqm/s  wrqm/s  %util  await
# nvme0n1  150.00  450.00  5.86   18.75   0.00    25.00  68.5   2.34
# sda       0.50    1.20   0.01    0.05   0.00     0.30   0.8   4.50
```

**Critical iostat metrics:**

| Metric   | Meaning                              | Warning Threshold |
|----------|--------------------------------------|--------------------|
| `%util`  | How busy the device is               | > 80%             |
| `await`  | Average I/O wait time (ms)           | > 10ms (SSD)      |
| `r/s`    | Read operations per second           | Context-dependent  |
| `w/s`    | Write operations per second          | Context-dependent  |

**sar — System Activity Reporter:**

```bash
# CPU usage history (requires sysstat package)
sar -u 1 10          # CPU every 1 second, 10 samples
sar -r 1 10          # Memory every 1 second
sar -d 1 10          # Disk I/O every 1 second
sar -n DEV 1 10      # Network every 1 second

# Historical data (from /var/log/sa/)
sar -u -f /var/log/sa/sa15   # CPU stats from the 15th of the month
```

**dmesg — Kernel ring buffer:**

```bash
# Recent kernel messages (OOM kills, hardware errors, filesystem issues)
dmesg -T --level=err,warn | tail -30

# Look for OOM killer activity
dmesg -T | grep -i "oom\|out of memory\|killed process"
# [Jan 15 14:22:33] Out of memory: Killed process 12345 (java) total-vm:8192000kB

# Look for disk errors
dmesg -T | grep -i "error\|fail\|i/o"

# Look for network issues
dmesg -T | grep -i "link\|eth\|net"
```

### 40.1.3 /proc and /sys Filesystem Investigation

The `/proc` and `/sys` filesystems are windows into the kernel's view of the system:

```bash
# CPU information
cat /proc/cpuinfo | grep "model name" | head -1
cat /proc/cpuinfo | grep processor | wc -l  # Number of CPU cores

# Memory breakdown
cat /proc/meminfo | head -10
# MemTotal:       32946176 kB
# MemFree:         2340508 kB
# MemAvailable:    7605308 kB    ← This is what matters (free + reclaimable)
# Buffers:          184320 kB
# Cached:          5260800 kB

# Per-process memory maps (replace PID)
cat /proc/<PID>/status | grep -i "vmrss\|vmsize\|threads"
# VmSize:   1234567 kB   (virtual memory)
# VmRSS:     567890 kB   (resident set size — actual physical memory)
# Threads:        42

# Open file descriptors per process
ls /proc/<PID>/fd | wc -l

# System-wide file descriptor limits
cat /proc/sys/fs/file-nr
# 15232  0  1048576    (allocated, unused, max)

# Network connection counts
cat /proc/net/sockstat
# TCP: inuse 1234 orphan 5 tw 678 alloc 2345 mem 890

# Check if swap is being used
cat /proc/swaps
swapon --show

# cgroup information for a container (find cgroup path first)
cat /proc/<PID>/cgroup
# Memory limit for a container's cgroup
cat /sys/fs/cgroup/memory/<cgroup-path>/memory.limit_in_bytes
# Current memory usage
cat /sys/fs/cgroup/memory/<cgroup-path>/memory.usage_in_bytes
```

### 40.1.4 kubelet Logs, containerd Logs

```bash
# kubelet logs — the most important logs on any node
journalctl -u kubelet --no-pager -n 100
journalctl -u kubelet --since "10 minutes ago" --no-pager

# Common kubelet error patterns to look for:
# - "Failed to create pod sandbox" → CNI or container runtime issue
# - "PLEG is not healthy" → Pod Lifecycle Event Generator stuck
# - "node not ready" → kubelet losing connection to API server
# - "evicting pod" → node under resource pressure

# containerd logs
journalctl -u containerd --no-pager -n 50

# Docker logs (if using Docker runtime)
journalctl -u docker --no-pager -n 50

# Check kubelet configuration
cat /var/lib/kubelet/config.yaml

# Check kubelet certificate expiration
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates
```

### 40.1.5 Hands-on: Diagnosing a Slow Node

Scenario: Users report that pods on `node-3` are responding slowly.

```bash
# Step 1: Check node status and conditions
kubectl describe node node-3 | grep -A5 "Conditions:"
# MemoryPressure   False
# DiskPressure     True     ← Problem found!
# PIDPressure      False
# Ready            True

# Step 2: SSH/debug into the node
kubectl debug node/node-3 -it --image=ubuntu:22.04
chroot /host

# Step 3: Check disk usage
df -h
# /dev/nvme0n1p1  100G   95G  5.0G  95% /    ← 95% full!

# Step 4: Find what's consuming disk
du -sh /var/log/*         | sort -rh | head -5
du -sh /var/lib/docker/*  | sort -rh | head -5
du -sh /var/lib/containerd/* | sort -rh | head -5

# Step 5: Check for large container logs
find /var/log/pods -name "*.log" -size +100M -exec ls -lh {} \;

# Step 6: Check I/O performance
iostat -xz 2 5
# %util at 98% — disk is saturated

# Step 7: Identify I/O-heavy processes
iotop -b -n 3 | head -20
# PID    DISK READ  DISK WRITE  COMMAND
# 12345  50.00 M/s  120.00 M/s  postgres

# Step 8: Resolution
# - Clean up old container images: crictl rmi --prune
# - Rotate large log files
# - Move I/O-heavy workloads to nodes with faster storage
# - Configure kubelet image garbage collection
```

---

## 40.2 Pod-Level Debugging

Most debugging starts here. A pod is misbehaving — it's crashing, slow, or
producing wrong results. These are your tools.

### 40.2.1 kubectl logs

```bash
# Basic log retrieval
kubectl logs <pod-name>

# Logs from a specific container in a multi-container pod
kubectl logs <pod-name> -c <container-name>

# Logs from a previous instance (after restart/crash)
kubectl logs <pod-name> --previous

# Follow logs in real time (like tail -f)
kubectl logs <pod-name> -f

# Logs from the last hour
kubectl logs <pod-name> --since=1h

# Last 100 lines
kubectl logs <pod-name> --tail=100

# Logs from all pods matching a label
kubectl logs -l app=nginx --all-containers=true

# Logs with timestamps
kubectl logs <pod-name> --timestamps=true

# Logs from an init container
kubectl logs <pod-name> -c <init-container-name>
```

**Pro tips:**

- If a container is `CrashLoopBackOff`, use `--previous` to see logs from the
  last crash.
- For high-volume logs, use `--since` and `--tail` to avoid overwhelming your terminal.
- Use `stern` (third-party tool) for multi-pod log streaming with color coding:
  ```bash
  stern "nginx-*" --namespace=production --since=5m
  ```

### 40.2.2 kubectl exec — Running Commands in Containers

```bash
# Open an interactive shell
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -- /bin/sh   # If bash is not available

# Run a single command
kubectl exec <pod-name> -- cat /etc/resolv.conf
kubectl exec <pod-name> -- env | sort
kubectl exec <pod-name> -- ps aux
kubectl exec <pod-name> -- ls -la /app/config/

# In a specific container
kubectl exec -it <pod-name> -c <container-name> -- /bin/sh

# Check DNS resolution from inside the pod
kubectl exec <pod-name> -- nslookup kubernetes.default

# Check network connectivity
kubectl exec <pod-name> -- curl -s http://other-service:8080/health

# Check mounted volumes
kubectl exec <pod-name> -- df -h
kubectl exec <pod-name> -- ls -la /data/

# Check process resource usage
kubectl exec <pod-name> -- top -bn1 | head -10
```

### 40.2.3 kubectl debug — Ephemeral Debug Containers

Ephemeral containers are injected into a running pod without restarting it.
This is invaluable when the original container lacks debugging tools.

```bash
# Add an ephemeral debug container to a running pod
kubectl debug -it <pod-name> --image=busybox:1.36 --target=<container-name>

# Debug with a full toolkit (nicolaka/netshoot has everything)
kubectl debug -it <pod-name> --image=nicolaka/netshoot --target=app

# Inside the debug container, you share the process namespace:
ps aux                    # See processes from the target container
cat /proc/1/environ       # Environment variables of the target process
ls /proc/1/root/app/      # Filesystem of the target container

# Create a copy of the pod with a different image (for deeper debugging)
kubectl debug <pod-name> -it --copy-to=debug-pod --image=ubuntu:22.04

# Create a copy with modified command (skip the entrypoint)
kubectl debug <pod-name> -it --copy-to=debug-pod \
  --container=app --image=ubuntu:22.04 -- /bin/bash
```

### 40.2.4 kubectl cp — Copying Files To/From Containers

```bash
# Copy a file from the container to your local machine
kubectl cp <pod-name>:/app/config.yaml ./config.yaml

# Copy a file from local to the container
kubectl cp ./debug-script.sh <pod-name>:/tmp/debug-script.sh

# Copy from a specific container
kubectl cp <pod-name>:/var/log/app.log ./app.log -c <container-name>

# Copy an entire directory
kubectl cp <pod-name>:/app/data/ ./local-data/
```

> **Note:** `kubectl cp` requires `tar` in the container. It won't work with
> truly minimal images. Use ephemeral containers as an alternative.

### 40.2.5 Distroless Container Debugging

Distroless containers have no shell, no package manager, no debugging tools.
Ephemeral containers are the only way to debug them:

```bash
# The pod is running a distroless Go binary
kubectl get pod distroless-app -o jsonpath='{.spec.containers[0].image}'
# gcr.io/distroless/static:nonroot

# Cannot exec — there's no shell
kubectl exec -it distroless-app -- /bin/sh
# error: unable to start container process: exec: "/bin/sh": stat /bin/sh: no such file

# Use an ephemeral debug container
kubectl debug -it distroless-app \
  --image=busybox:1.36 \
  --target=app

# Now you can inspect the process
ls /proc/1/root/          # See the app's filesystem
cat /proc/1/cmdline       # See the command line
cat /proc/1/maps          # Memory mappings
```

### 40.2.6 Hands-on: Debugging a CrashLoopBackOff

```bash
# Step 1: Identify the problematic pod
kubectl get pods -A | grep CrashLoop
# NAMESPACE   NAME              READY   STATUS             RESTARTS   AGE
# production  api-server-abc12  0/1     CrashLoopBackOff   15         30m

# Step 2: Get pod events
kubectl describe pod api-server-abc12 -n production
# Events:
#   Warning  BackOff  30s  kubelet  Back-off restarting failed container

# Step 3: Check logs from the last crash
kubectl logs api-server-abc12 -n production --previous
# panic: runtime error: invalid memory address or nil pointer dereference
# goroutine 1 [running]:
# main.initDatabase(...)
#     /app/main.go:45

# Step 4: Check environment variables and config
kubectl get pod api-server-abc12 -n production -o yaml | grep -A20 "env:"
# - name: DATABASE_URL
#   value: "postgres://db:5432/myapp"  ← Is the database reachable?

# Step 5: Check if dependent services are running
kubectl get pods -n production -l app=postgres
# NAME              READY   STATUS    RESTARTS   AGE
# postgres-0        0/1     Pending   0          35m    ← Database is not running!

# Step 6: Fix the root cause
kubectl describe pod postgres-0 -n production
# Events:
#   Warning  FailedScheduling  No nodes match pod affinity requirements
# Fix the affinity rules or add nodes, then the api-server will stop crashing.
```

### 40.2.7 Hands-on: Debugging an OOMKilled Container

```bash
# Step 1: Identify OOMKilled pod
kubectl get pods -A | grep -i oom
kubectl get pods -n production -o json | \
  jq '.items[] | select(.status.containerStatuses[]?.lastState.terminated.reason == "OOMKilled") | .metadata.name'

# Step 2: Confirm OOMKilled
kubectl describe pod <pod-name> -n production
# State:          Waiting
#   Reason:       CrashLoopBackOff
# Last State:     Terminated
#   Reason:       OOMKilled
#   Exit Code:    137

# Step 3: Check current memory limits
kubectl get pod <pod-name> -n production -o jsonpath='{.spec.containers[0].resources}'
# {"limits":{"memory":"256Mi"},"requests":{"memory":"128Mi"}}

# Step 4: Check actual memory usage before it was killed
kubectl top pod <pod-name> -n production --containers
# POD          CONTAINER   CPU    MEMORY
# api-server   app         50m    245Mi    ← Close to 256Mi limit!

# Step 5: Check node-level dmesg for OOM details
kubectl debug node/<node-name> -it --image=busybox
chroot /host
dmesg -T | grep -i oom | tail -5
# [Jan 15 14:22:33] Memory cgroup out of memory: Killed process 12345 (app)
#                   total-vm:524288kB, anon-rss:262144kB, file-rss:0kB

# Step 6: Profile memory usage (see Section 40.4 for pprof)
# Temporarily increase the memory limit and add pprof endpoint:
# Then: go tool pprof http://pod-ip:6060/debug/pprof/heap

# Step 7: Fix — either optimize memory usage or increase limits
kubectl set resources deployment/api-server -n production \
  --limits=memory=512Mi --requests=memory=256Mi
```

### 40.2.8 Hands-on: Debugging ImagePullBackOff

```bash
# Step 1: Identify the pod
kubectl get pods | grep ImagePull
# NAME           READY   STATUS             RESTARTS   AGE
# webapp-xyz12   0/1     ImagePullBackOff   0          5m

# Step 2: Get the error details
kubectl describe pod webapp-xyz12
# Events:
#   Warning  Failed   pull image "myregistry.io/app:v2.3.1": unauthorized

# Step 3: Check image name (typo?)
kubectl get pod webapp-xyz12 -o jsonpath='{.spec.containers[0].image}'
# myregistry.io/app:v2.3.1

# Step 4: Verify the image exists
# From your local machine:
docker manifest inspect myregistry.io/app:v2.3.1
# If this fails, the image doesn't exist or the tag is wrong

# Step 5: Check image pull secrets
kubectl get pod webapp-xyz12 -o jsonpath='{.spec.imagePullSecrets}'
# [{"name":"registry-cred"}]

kubectl get secret registry-cred -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d
# Verify the credentials are correct and not expired

# Step 6: Common fixes:
# a) Wrong image tag:
kubectl set image deployment/webapp app=myregistry.io/app:v2.3.0  # Use correct tag

# b) Missing pull secret:
kubectl create secret docker-registry registry-cred \
  --docker-server=myregistry.io \
  --docker-username=user \
  --docker-password=pass

# c) Expired credentials:
kubectl delete secret registry-cred
kubectl create secret docker-registry registry-cred \
  --docker-server=myregistry.io \
  --docker-username=user \
  --docker-password=NEW_TOKEN
```

---

## 40.3 Network Debugging

Network issues are the most frustrating to debug because they're often
intermittent, hard to reproduce, and span multiple layers.

### 40.3.1 DNS Resolution Testing

DNS is the #1 source of Kubernetes networking issues. Always check DNS first.

```bash
# From inside a pod (or using kubectl exec)
# Test cluster DNS
nslookup kubernetes.default
# Server:    10.96.0.10
# Address:   10.96.0.10#53
# Name:      kubernetes.default.svc.cluster.local
# Address:   10.96.0.1

# Test service DNS
nslookup my-service.my-namespace.svc.cluster.local

# More detailed with dig
dig my-service.my-namespace.svc.cluster.local +search +short
dig @10.96.0.10 my-service.my-namespace.svc.cluster.local A

# Test external DNS
nslookup google.com
dig google.com +short

# Check resolv.conf (DNS configuration)
cat /etc/resolv.conf
# nameserver 10.96.0.10
# search my-namespace.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5

# The ndots:5 setting means any name with fewer than 5 dots gets
# the search domains appended before trying as-is. This can cause
# unexpected DNS lookups. Example:
# "api.example.com" has 2 dots (< 5), so Kubernetes first tries:
#   api.example.com.my-namespace.svc.cluster.local
#   api.example.com.svc.cluster.local
#   api.example.com.cluster.local
#   api.example.com                    ← finally tries the actual name
#
# Fix: Use FQDNs with trailing dot: "api.example.com."
# Or reduce ndots in pod spec:
# spec:
#   dnsConfig:
#     options:
#       - name: ndots
#         value: "2"
```

### 40.3.2 Connectivity Testing

```bash
# curl — HTTP/HTTPS testing
curl -v http://my-service:8080/health
curl -s -o /dev/null -w "%{http_code} %{time_total}s" http://my-service:8080/health
# 200 0.023s

# wget — simpler alternative
wget -qO- http://my-service:8080/health

# nc (netcat) — TCP connectivity testing
nc -zv my-service 8080
# Connection to my-service 8080 port [tcp/*] succeeded!

nc -zv my-database 5432
# Connection to my-database 5432 port [tcp/postgresql] succeeded!

# Test UDP connectivity
nc -zuv my-dns-service 53

# telnet — interactive TCP testing
telnet my-service 8080

# Test from specific source to understand routing
curl --interface eth0 http://my-service:8080/health
```

### 40.3.3 Packet Capture with tcpdump

```bash
# Capture all traffic on the pod's network interface
tcpdump -i eth0 -nn -c 100

# Capture traffic to/from a specific host
tcpdump -i eth0 host 10.244.1.50 -nn

# Capture DNS traffic
tcpdump -i eth0 port 53 -nn -vv

# Capture HTTP traffic
tcpdump -i eth0 port 80 -A -nn

# Save capture to file for analysis in Wireshark
tcpdump -i eth0 -w /tmp/capture.pcap -c 1000

# Capture traffic to a specific service IP
tcpdump -i any dst host 10.96.100.50 -nn

# Capture SYN packets only (connection attempts)
tcpdump -i eth0 'tcp[tcpflags] & (tcp-syn) != 0' -nn

# Capture RST packets (connection resets — often indicates issues)
tcpdump -i eth0 'tcp[tcpflags] & (tcp-rst) != 0' -nn
```

### 40.3.4 Network Path: traceroute, mtr

```bash
# traceroute — show network hops
traceroute my-service.my-namespace.svc.cluster.local

# mtr — combines traceroute and ping (better for intermittent issues)
mtr --report --report-cycles=10 10.244.1.50
# HOST                      Loss%   Snt   Last   Avg  Best  Wrst StDev
# 1. 10.244.0.1              0.0%    10    0.3   0.3   0.2   0.5   0.1
# 2. 10.244.1.50             0.0%    10    0.5   0.6   0.4   1.2   0.2
```

### 40.3.5 iptables Rules Inspection

kube-proxy uses iptables (or IPVS) to implement Services. Understanding the
rules helps debug connectivity issues:

```bash
# List all NAT rules (Service routing)
iptables -t nat -L -n -v | head -50

# Find rules for a specific service IP
iptables -t nat -L -n | grep 10.96.100.50

# KUBE-SERVICES chain — entry point for all service traffic
iptables -t nat -L KUBE-SERVICES -n

# Follow a service through the iptables chain
# KUBE-SERVICES → KUBE-SVC-XXXX → KUBE-SEP-YYYY (endpoint)
iptables -t nat -L KUBE-SVC-XXXXXXXXXXXX -n
# Chain KUBE-SVC-XXXXXXXXXXXX (1 references)
# target     prot  source    destination
# KUBE-SEP-AAAAAA  all  --  0.0.0.0/0  0.0.0.0/0  statistic mode random probability 0.33
# KUBE-SEP-BBBBBB  all  --  0.0.0.0/0  0.0.0.0/0  statistic mode random probability 0.50
# KUBE-SEP-CCCCCC  all  --  0.0.0.0/0  0.0.0.0/0

# Check IPVS rules (if using IPVS mode)
ipvsadm -Ln
# TCP  10.96.100.50:80 rr
#   -> 10.244.1.10:80   Masq   1  0  0
#   -> 10.244.2.20:80   Masq   1  0  0
#   -> 10.244.3.30:80   Masq   1  0  0

# Check network policies (via iptables FILTER table)
iptables -L -n -v | grep -i "cali\|cilium\|antrea"
```

### 40.3.6 Service Endpoint Verification

```bash
# Check if the service exists and has the right selector
kubectl get svc my-service -o wide
# NAME         TYPE        CLUSTER-IP     PORT(S)   SELECTOR
# my-service   ClusterIP   10.96.100.50   80/TCP    app=myapp

# Check endpoints — does the service have backing pods?
kubectl get endpoints my-service
# NAME         ENDPOINTS                                   AGE
# my-service   10.244.1.10:80,10.244.2.20:80,10.244.3.30:80  5d

# If endpoints are empty, the selector doesn't match any pods:
kubectl get pods -l app=myapp
# No resources found → Fix the label selector

# Detailed endpoint slices (newer API)
kubectl get endpointslices -l kubernetes.io/service-name=my-service -o yaml

# Check if pods are Ready (only Ready pods get endpoints)
kubectl get pods -l app=myapp -o jsonpath='{range .items[*]}{.metadata.name} Ready={.status.conditions[?(@.type=="Ready")].status}{"\n"}{end}'
```

### 40.3.7 Hands-on: Systematic Network Debugging Flowchart

```
Service Not Accessible
│
├─ 1. Does the Service exist?
│  └─ kubectl get svc <name> -n <ns>
│     ├─ No → Create the service
│     └─ Yes → Continue
│
├─ 2. Does the Service have endpoints?
│  └─ kubectl get endpoints <name> -n <ns>
│     ├─ No endpoints →
│     │  ├─ Check selector: kubectl get svc <name> -o yaml | grep selector
│     │  ├─ Check pods match: kubectl get pods -l <selector>
│     │  └─ Check pods are Ready: kubectl describe pod <name>
│     └─ Has endpoints → Continue
│
├─ 3. Can you reach the Pod directly?
│  └─ kubectl exec debug-pod -- curl <pod-ip>:<port>
│     ├─ No →
│     │  ├─ Pod not listening: kubectl exec <pod> -- ss -tlnp
│     │  ├─ Network policy blocking: kubectl get networkpolicy -n <ns>
│     │  └─ CNI issue: check CNI logs on the node
│     └─ Yes → Continue
│
├─ 4. Can you reach the Service ClusterIP?
│  └─ kubectl exec debug-pod -- curl <cluster-ip>:<port>
│     ├─ No →
│     │  ├─ kube-proxy issue: check kube-proxy logs
│     │  ├─ iptables rules: iptables -t nat -L KUBE-SERVICES
│     │  └─ Port mismatch: service port vs targetPort
│     └─ Yes → Continue
│
├─ 5. DNS resolution working?
│  └─ kubectl exec debug-pod -- nslookup <service>.<namespace>
│     ├─ No →
│     │  ├─ CoreDNS running: kubectl get pods -n kube-system -l k8s-app=kube-dns
│     │  ├─ CoreDNS logs: kubectl logs -n kube-system -l k8s-app=kube-dns
│     │  └─ Pod DNS config: kubectl exec <pod> -- cat /etc/resolv.conf
│     └─ Yes → Continue
│
└─ 6. External access (Ingress/LoadBalancer)?
   ├─ Check Ingress controller logs
   ├─ Check external LB health checks
   ├─ Check TLS certificate validity
   └─ Check cloud provider LB configuration
```

### 40.3.8 Hands-on: Building a Debug Pod Manifest (netshoot)

The `netshoot` image contains every networking tool you'll ever need:

```yaml
# debug-pod.yaml
# A comprehensive network debugging pod that can be deployed to any namespace
# for troubleshooting connectivity, DNS, and performance issues.
apiVersion: v1
kind: Pod
metadata:
  name: netshoot-debug
  labels:
    app: debug
spec:
  # Run on a specific node if needed
  # nodeName: node-3
  containers:
    - name: netshoot
      image: nicolaka/netshoot:latest
      command: ["sleep", "infinity"]
      resources:
        requests:
          cpu: 100m
          memory: 128Mi
        limits:
          cpu: 500m
          memory: 256Mi
      securityContext:
        capabilities:
          add:
            - NET_ADMIN
            - NET_RAW
            - SYS_PTRACE
  # Match the DNS config of the pods you're debugging
  dnsPolicy: ClusterFirst
  restartPolicy: Never
  terminationGracePeriodSeconds: 0
```

```bash
# Deploy and use
kubectl apply -f debug-pod.yaml -n production
kubectl exec -it netshoot-debug -n production -- bash

# Tools available in netshoot:
# curl, wget, ping, traceroute, mtr, dig, nslookup, tcpdump,
# iperf3, netstat, ss, ip, nmap, openssl, strace, ltrace,
# bridge, ctop, drill, ethtool, grpcurl, iftop, httpie,
# termshark (TUI wireshark), and many more

# Network performance test between pods
# On server pod:
iperf3 -s
# On client pod:
iperf3 -c <server-pod-ip> -t 10

# TLS certificate testing
openssl s_client -connect my-service:443 -servername my-service.example.com

# gRPC health check
grpcurl -plaintext my-grpc-service:50051 grpc.health.v1.Health/Check

# Clean up
kubectl delete pod netshoot-debug -n production
```

---

## 40.4 Go Application Performance

Go has best-in-class built-in profiling tools. Every Go service in production
should have pprof enabled.

### 40.4.1 pprof — CPU, Memory, Goroutine, Block, Mutex Profiling

pprof provides several profile types:

| Profile     | What It Measures                              | When To Use                        |
|-------------|-----------------------------------------------|------------------------------------|
| `cpu`       | CPU time spent in each function               | High CPU usage                     |
| `heap`      | Current heap memory allocations               | High memory usage                  |
| `allocs`    | All allocations (past and present)            | Excessive garbage collection       |
| `goroutine` | Stack traces of all goroutines                | Goroutine leaks, deadlocks         |
| `block`     | Where goroutines block on synchronization     | Contention on channels/mutexes     |
| `mutex`     | Mutex contention                              | Lock contention                    |
| `threadcreate` | Stack traces that led to new OS threads    | Thread leak investigation          |

**net/http/pprof — HTTP endpoint (recommended for services):**

```go
// Package main demonstrates how to enable pprof profiling in a production
// Go HTTP service. The pprof endpoints are served on a separate port (6060)
// to avoid exposing them on the public-facing port.
package main

import (
	"log"
	"net/http"
	// Importing net/http/pprof registers pprof handlers on the default
	// ServeMux. These handlers serve profiling data at /debug/pprof/.
	_ "net/http/pprof"
)

func main() {
	// Start the pprof server on a separate port.
	// NEVER expose this on your public port — profiling data can reveal
	// sensitive information about your application's internals.
	go func() {
		log.Println("pprof server listening on :6060")
		if err := http.ListenAndServe("localhost:6060", nil); err != nil {
			log.Printf("pprof server error: %v", err)
		}
	}()

	// Your main application server runs on a different port.
	mux := http.NewServeMux()
	mux.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
			w.Write([]byte("ok"))
		})
	mux.HandleFunc("/api/data", handleData)

	log.Println("Application server listening on :8080")
	log.Fatal(http.ListenAndServe(":8080", mux))
}

// handleData simulates a real API endpoint that processes data.
// In production, this would query a database, call other services, etc.
func handleData(w http.ResponseWriter, r *http.Request) {
	// ... application logic ...
	w.Write([]byte(`{"status":"ok"}`))
}
```

**runtime/pprof — Programmatic profiling (for CLI tools and tests):**

```go
// Package main demonstrates programmatic CPU and memory profiling using
// runtime/pprof. This approach is useful for CLI tools, batch jobs, and
// benchmarks where you want to profile a specific code path.
package main

import (
	"fmt"
	"os"
	"runtime"
	"runtime/pprof"
)

func main() {
	// ── CPU Profile ──
	// Create the output file for the CPU profile.
	cpuFile, err := os.Create("cpu.prof")
	if err != nil {
		fmt.Fprintf(os.Stderr, "creating cpu profile: %v\n", err)
		os.Exit(1)
	}
	defer cpuFile.Close()

	// Start CPU profiling. This samples the call stack 100 times per second.
	if err := pprof.StartCPUProfile(cpuFile); err != nil {
		fmt.Fprintf(os.Stderr, "starting cpu profile: %v\n", err)
		os.Exit(1)
	}

	// Run your workload here.
	doWork()

	// Stop CPU profiling and flush the data.
	pprof.StopCPUProfile()

	// ── Heap Profile ──
	// Force a garbage collection to get accurate heap statistics.
	runtime.GC()

	heapFile, err := os.Create("heap.prof")
	if err != nil {
		fmt.Fprintf(os.Stderr, "creating heap profile: %v\n", err)
		os.Exit(1)
	}
	defer heapFile.Close()

	// Write the heap profile. This captures all live heap allocations.
	if err := pprof.WriteHeapProfile(heapFile); err != nil {
		fmt.Fprintf(os.Stderr, "writing heap profile: %v\n", err)
		os.Exit(1)
	}

	fmt.Println("Profiles written: cpu.prof, heap.prof")
	fmt.Println("Analyze with: go tool pprof cpu.prof")
}

// doWork simulates a CPU-intensive and memory-allocating workload.
func doWork() {
	// Allocate and process data to generate interesting profile data.
	for i := 0; i < 1000000; i++ {
		data := make([]byte, 1024)
		for j := range data {
			data[j] = byte(j % 256)
		}
	}
}
```

**go tool pprof — Analysis:**

```bash
# Interactive analysis of a CPU profile
go tool pprof cpu.prof
# (pprof) top 10
# Showing top 10 nodes out of 45
#       flat  flat%   sum%        cum   cum%
#    2.50s 25.00% 25.00%      2.50s 25.00%  runtime.memmove
#    1.80s 18.00% 43.00%      1.80s 18.00%  main.processData
#    1.20s 12.00% 55.00%      3.00s 30.00%  encoding/json.Marshal

# (pprof) list processData    ← Show source code annotated with CPU time
# (pprof) web                  ← Open flame graph in browser
# (pprof) svg > profile.svg   ← Save as SVG

# Fetch profile from a running service
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=30

# Heap profile analysis
go tool pprof http://localhost:6060/debug/pprof/heap
# (pprof) top 10 -cum
# (pprof) list MyFunction

# Goroutine analysis
go tool pprof http://localhost:6060/debug/pprof/goroutine

# Compare two profiles (before and after optimization)
go tool pprof -diff_base=before.prof after.prof

# Web UI mode (recommended — interactive flame graphs)
go tool pprof -http=:8081 cpu.prof
```

### 40.4.2 Hands-on: Complete pprof Setup and Analysis

```go
// Package main provides a complete, production-ready example of a Go HTTP
// service with pprof profiling enabled. It includes a deliberately inefficient
// endpoint that we will profile and optimize.
package main

import (
	"crypto/sha256"
	"encoding/hex"
	"fmt"
	"log"
	"math/rand"
	"net/http"
	_ "net/http/pprof"
	"strings"
	"sync"
	"time"
)

// cache is an in-memory cache that demonstrates a common memory leak pattern.
// Items are added but never removed, causing unbounded growth.
var cache = struct {
	sync.RWMutex
	items map[string][]byte
}{items: make(map[string][]byte)}

// handleHash is a deliberately inefficient endpoint that computes SHA-256
// hashes. We will use pprof to identify the bottleneck and optimize it.
func handleHash(w http.ResponseWriter, r *http.Request) {
	// Inefficiency 1: Building a large string with concatenation (O(n²))
	var result string
	for i := 0; i < 1000; i++ {
		result = result + fmt.Sprintf("item-%d-", i) // String concatenation in a loop!
	}

	// Inefficiency 2: Computing hash of the entire string multiple times
	for i := 0; i < 100; i++ {
		hash := sha256.Sum256([]byte(result))
		_ = hex.EncodeToString(hash[:])
	}

	// Inefficiency 3: Unbounded cache growth (memory leak)
	key := fmt.Sprintf("request-%d", time.Now().UnixNano())
	data := make([]byte, 10240) // 10KB per request
	rand.Read(data)
	cache.Lock()
	cache.items[key] = data
	cache.Unlock()

	w.Write([]byte("ok"))
}

// handleHashOptimized is the optimized version of handleHash.
// We use strings.Builder, compute the hash once, and bound the cache.
func handleHashOptimized(w http.ResponseWriter, r *http.Request) {
	// Fix 1: Use strings.Builder for O(n) concatenation
	var builder strings.Builder
	builder.Grow(1000 * 15) // Pre-allocate approximate capacity
	for i := 0; i < 1000; i++ {
		fmt.Fprintf(&builder, "item-%d-", i)
	}
	result := builder.String()

	// Fix 2: Compute hash only once
	hash := sha256.Sum256([]byte(result))
	_ = hex.EncodeToString(hash[:])

	// Fix 3: Bound the cache size
	cache.Lock()
	if len(cache.items) > 10000 {
		// Evict oldest entries (simplified; use LRU in production)
		cache.items = make(map[string][]byte)
	}
	key := fmt.Sprintf("request-%d", time.Now().UnixNano())
	cache.items[key] = make([]byte, 1024) // Reduced allocation
	cache.Unlock()

	w.Write([]byte("ok"))
}

func main() {
	// pprof endpoints on separate port
	go func() {
		log.Println("pprof at http://localhost:6060/debug/pprof/")
		http.ListenAndServe("localhost:6060", nil)
	}()

	mux := http.NewServeMux()
	mux.HandleFunc("/hash", handleHash)
	mux.HandleFunc("/hash-optimized", handleHashOptimized)

	log.Println("Server at :8080")
	http.ListenAndServe(":8080", mux)
}
```

**Profiling workflow:**

```bash
# Step 1: Start the server
go run main.go

# Step 2: Generate load
hey -n 10000 -c 50 http://localhost:8080/hash

# Step 3: Capture CPU profile (30 seconds)
go tool pprof -http=:8081 http://localhost:6060/debug/pprof/profile?seconds=30
# Opens browser with flame graph — look for the hottest functions

# Step 4: Capture heap profile
go tool pprof -http=:8082 http://localhost:6060/debug/pprof/heap
# Look for functions with the most allocations

# Step 5: Check goroutine count
curl -s http://localhost:6060/debug/pprof/goroutine?debug=1 | head -5
# goroutine profile: total 53

# Step 6: Now test the optimized endpoint
hey -n 10000 -c 50 http://localhost:8080/hash-optimized
# Compare latency and throughput
```

### 40.4.3 Hands-on: Finding and Fixing a Memory Leak

```go
// Package main demonstrates a goroutine-based memory leak and how to detect
// it using pprof. The leak occurs because goroutines are spawned for each
// request but never properly cancelled when they complete.
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	_ "net/http/pprof"
	"sync"
	"time"
)

// LeakyProcessor demonstrates a memory leak caused by goroutines that
// hold references to large buffers and never terminate. Each call to
// Process spawns a goroutine that runs forever.
type LeakyProcessor struct {
	mu      sync.Mutex
	workers int
}

// Process spawns a background goroutine that holds a 1MB buffer.
// BUG: The goroutine runs forever — it has no cancellation mechanism.
// Over time, thousands of goroutines accumulate, each holding 1MB.
func (p *LeakyProcessor) Process(data string) {
	p.mu.Lock()
	p.workers++
	p.mu.Unlock()

	go func() {
		buf := make([]byte, 1024*1024) // 1MB buffer — leaked!
		for {
			// Simulate work that never ends
			for i := range buf {
				buf[i] = byte(i % 256)
			}
			time.Sleep(time.Second)
		}
	}()
}

// FixedProcessor demonstrates the correct approach: goroutines are
// bound to a context and cancel when the context is done. A worker
// pool limits the maximum number of concurrent goroutines.
type FixedProcessor struct {
	mu      sync.Mutex
	workers int
	sem     chan struct{} // Semaphore to limit concurrency
}

// NewFixedProcessor creates a processor with a bounded worker pool.
// maxWorkers controls the maximum number of concurrent goroutines.
func NewFixedProcessor(maxWorkers int) *FixedProcessor {
	return &FixedProcessor{
		sem: make(chan struct{}, maxWorkers),
	}
}

// Process spawns a goroutine that respects the context's cancellation
// and releases its semaphore slot when done. The buffer is allocated
// only for the duration of the work.
func (p *FixedProcessor) Process(ctx context.Context, data string) error {
	select {
	case p.sem <- struct{}{}:
		// Acquired semaphore slot
	case <-ctx.Done():
		return ctx.Err()
	}

	p.mu.Lock()
	p.workers++
	p.mu.Unlock()

	go func() {
		defer func() {
			<-p.sem // Release semaphore slot
			p.mu.Lock()
			p.workers--
			p.mu.Unlock()
		}()

		buf := make([]byte, 1024*1024)
		for i := range buf {
			buf[i] = byte(i % 256)
		}
		// Work completes and goroutine exits — no leak!
	}()

	return nil
}

func main() {
	leaky := &LeakyProcessor{}
	fixed := NewFixedProcessor(10)

	go func() {
		log.Println("pprof at http://localhost:6060/debug/pprof/")
		http.ListenAndServe("localhost:6060", nil)
	}()

	mux := http.NewServeMux()

	// Leaky endpoint — each request spawns an immortal goroutine
	mux.HandleFunc("/leaky", func(w http.ResponseWriter, r *http.Request) {
			leaky.Process("data")
			leaky.mu.Lock()
			count := leaky.workers
			leaky.mu.Unlock()
			fmt.Fprintf(w, "Workers: %d\n", count)
		})

	// Fixed endpoint — bounded worker pool with proper cleanup
	mux.HandleFunc("/fixed", func(w http.ResponseWriter, r *http.Request) {
			ctx, cancel := context.WithTimeout(r.Context(), 5*time.Second)
			defer cancel()
			if err := fixed.Process(ctx, "data"); err != nil {
				http.Error(w, err.Error(), http.StatusServiceUnavailable)
				return
			}
			fixed.mu.Lock()
			count := fixed.workers
			fixed.mu.Unlock()
			fmt.Fprintf(w, "Workers: %d\n", count)
		})

	log.Println("Server at :8080")
	http.ListenAndServe(":8080", mux)
}
```

**Detection workflow:**

```bash
# Generate load on the leaky endpoint
for i in $(seq 1 500); do curl -s http://localhost:8080/leaky > /dev/null; done

# Check goroutine count
curl -s http://localhost:6060/debug/pprof/goroutine?debug=1 | head -3
# goroutine profile: total 503   ← 500+ goroutines! Leak confirmed.

# Get goroutine profile with full stack traces
go tool pprof http://localhost:6060/debug/pprof/goroutine
# (pprof) top
# 500   99.40%  99.40%   500  99.40%  main.(*LeakyProcessor).Process.func1

# Check heap profile
go tool pprof http://localhost:6060/debug/pprof/heap
# (pprof) top
# 500MB  99.00%  main.(*LeakyProcessor).Process.func1   ← 500 * 1MB buffers
```

### 40.4.4 Hands-on: Finding and Fixing a Goroutine Leak

```go
// Package main demonstrates a common goroutine leak pattern: a producer
// goroutine sends to a channel, but the consumer gives up (e.g., due to
	// timeout), leaving the producer blocked forever on a send.
package main

import (
	"context"
	"fmt"
	"log"
	"math/rand"
	"net/http"
	_ "net/http/pprof"
	"time"
)

// fetchDataLeaky demonstrates the goroutine leak. The goroutine spawned
// inside this function will be blocked on ch <- result forever if the
// caller times out and stops reading from ch.
func fetchDataLeaky() (string, error) {
	ch := make(chan string) // Unbuffered channel — sender blocks until receiver reads

	go func() {
		// Simulate slow external call (1-5 seconds)
		time.Sleep(time.Duration(1+rand.Intn(5)) * time.Second)
		ch <- "data" // Blocks forever if no one reads from ch!
	}()

	select {
	case result := <-ch:
		return result, nil
	case <-time.After(2 * time.Second):
		return "", fmt.Errorf("timeout")
		// The goroutine is still alive, blocked on ch <- "data"
		// It will NEVER be garbage collected because it holds a reference to ch
	}
}

// fetchDataFixed demonstrates the fix: use a buffered channel so the
// goroutine can always send its result (even if nobody reads it) and exit.
// Additionally, respect context cancellation.
func fetchDataFixed(ctx context.Context) (string, error) {
	ch := make(chan string, 1) // Buffered channel — sender never blocks

	go func() {
		time.Sleep(time.Duration(1+rand.Intn(5)) * time.Second)
		ch <- "data" // Never blocks because buffer size is 1
		// Goroutine exits cleanly even if the caller timed out
	}()

	select {
	case result := <-ch:
		return result, nil
	case <-ctx.Done():
		return "", ctx.Err()
		// The goroutine will complete, send to the buffered channel,
		// and exit. The buffered value will be garbage collected.
	}
}

func main() {
	go func() {
		log.Println("pprof at http://localhost:6060/debug/pprof/")
		http.ListenAndServe("localhost:6060", nil)
	}()

	mux := http.NewServeMux()

	mux.HandleFunc("/leaky", func(w http.ResponseWriter, r *http.Request) {
			result, err := fetchDataLeaky()
			if err != nil {
				http.Error(w, err.Error(), http.StatusGatewayTimeout)
				return
			}
			fmt.Fprint(w, result)
		})

	mux.HandleFunc("/fixed", func(w http.ResponseWriter, r *http.Request) {
			ctx, cancel := context.WithTimeout(r.Context(), 2*time.Second)
			defer cancel()
			result, err := fetchDataFixed(ctx)
			if err != nil {
				http.Error(w, err.Error(), http.StatusGatewayTimeout)
				return
			}
			fmt.Fprint(w, result)
		})

	log.Println("Server at :8080")
	http.ListenAndServe(":8080", mux)
}
```

### 40.4.5 runtime/trace — Execution Tracer

The execution tracer captures a detailed timeline of goroutine scheduling,
system calls, garbage collection, and more:

```go
// Package main demonstrates the runtime/trace package for capturing
// execution traces. Unlike pprof (which samples), the tracer records
// every scheduling event, giving a complete picture of concurrency behavior.
package main

import (
	"fmt"
	"os"
	"runtime/trace"
	"sync"
)

func main() {
	// Create the trace output file.
	f, err := os.Create("trace.out")
	if err != nil {
		fmt.Fprintf(os.Stderr, "creating trace: %v\n", err)
		os.Exit(1)
	}
	defer f.Close()

	// Start the tracer. All goroutine events will be recorded.
	if err := trace.Start(f); err != nil {
		fmt.Fprintf(os.Stderr, "starting trace: %v\n", err)
		os.Exit(1)
	}
	defer trace.Stop()

	// Run concurrent work to generate interesting trace data.
	var wg sync.WaitGroup
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func(id int) {
			defer wg.Done()
			// Create a trace region for this unit of work.
			// Regions show up in the trace viewer as named spans.
			trace.WithRegion(nil, fmt.Sprintf("worker-%d", id), func() {
				// Simulate CPU-bound work
				sum := 0
				for j := 0; j < 10_000_000; j++ {
					sum += j
				}
				_ = sum
			})
		}(i)
	}
	wg.Wait()

	fmt.Println("Trace written to trace.out")
	fmt.Println("View with: go tool trace trace.out")
}
```

```bash
# Generate and view the trace
go run main.go
go tool trace trace.out
# Opens a browser with an interactive timeline showing:
# - Goroutine scheduling across OS threads
# - GC pauses and their duration
# - System calls and their latency
# - Network I/O blocking
# - User-defined regions and tasks

# For a running HTTP service, capture a trace remotely:
curl -o trace.out "http://localhost:6060/debug/pprof/trace?seconds=5"
go tool trace trace.out
```

**What to look for in traces:**

| Pattern                          | Indicates                          | Fix                              |
|----------------------------------|------------------------------------|----------------------------------|
| Long GC pauses                   | Too much heap allocation           | Reduce allocations, use sync.Pool|
| Goroutines frequently blocked    | Contention on channels/mutexes     | Reduce lock scope, use sharding  |
| Few goroutines running at once   | Serialization bottleneck           | Add concurrency, pipeline stages |
| Many goroutines in syscall       | Blocking I/O or CGo calls          | Use non-blocking I/O, pool CGo   |

### 40.4.6 Benchmarking with testing.B

Go's built-in benchmarking framework is simple but powerful:

```go
// Package hashperf benchmarks different approaches to string hashing
// to demonstrate Go's benchmarking framework and common optimization
// patterns.
package hashperf

import (
	"crypto/md5"
	"crypto/sha256"
	"fmt"
	"strings"
	"testing"
)

// buildStringConcat builds a string using naive concatenation.
// This is O(n²) because each concatenation allocates a new string.
func buildStringConcat(n int) string {
	var result string
	for i := 0; i < n; i++ {
		result += fmt.Sprintf("item-%d-", i)
	}
	return result
}

// buildStringBuilder builds a string using strings.Builder.
// This is O(n) because Builder grows its internal buffer as needed.
func buildStringBuilder(n int) string {
	var b strings.Builder
	b.Grow(n * 10) // Pre-allocate estimated capacity
	for i := 0; i < n; i++ {
		fmt.Fprintf(&b, "item-%d-", i)
	}
	return b.String()
}

// BenchmarkStringConcat measures the performance of naive string
// concatenation. The b.N loop is managed by the framework; it runs
// enough iterations to get a statistically stable measurement.
func BenchmarkStringConcat(b *testing.B) {
	for i := 0; i < b.N; i++ {
		buildStringConcat(1000)
	}
}

// BenchmarkStringBuilder measures the performance of strings.Builder.
// This should be significantly faster than concatenation.
func BenchmarkStringBuilder(b *testing.B) {
	for i := 0; i < b.N; i++ {
		buildStringBuilder(1000)
	}
}

// BenchmarkHashMD5 benchmarks MD5 hashing of a 1KB payload.
func BenchmarkHashMD5(b *testing.B) {
	data := []byte(strings.Repeat("a", 1024))
	b.ResetTimer()           // Exclude setup time from the benchmark
	b.SetBytes(1024)         // Report throughput in MB/s
	for i := 0; i < b.N; i++ {
		md5.Sum(data)
	}
}

// BenchmarkHashSHA256 benchmarks SHA-256 hashing of a 1KB payload.
func BenchmarkHashSHA256(b *testing.B) {
	data := []byte(strings.Repeat("a", 1024))
	b.ResetTimer()
	b.SetBytes(1024)
	for i := 0; i < b.N; i++ {
		sha256.Sum256(data)
	}
}

// BenchmarkStringBuilder_Sizes benchmarks string building at different
// sizes using sub-benchmarks. This helps understand how performance
// scales with input size.
func BenchmarkStringBuilder_Sizes(b *testing.B) {
	sizes := []int{10, 100, 1000, 10000}
	for _, size := range sizes {
		b.Run(fmt.Sprintf("n=%d", size), func(b *testing.B) {
				for i := 0; i < b.N; i++ {
					buildStringBuilder(size)
				}
			})
	}
}

// BenchmarkWithAllocations demonstrates ReportAllocs, which reports
// the number of heap allocations per operation. Reducing allocations
// is one of the most effective Go optimization strategies.
func BenchmarkWithAllocations(b *testing.B) {
	b.ReportAllocs()
	for i := 0; i < b.N; i++ {
		buildStringBuilder(100)
	}
}
```

```bash
# Run all benchmarks
go test -bench=. -benchmem ./...
# BenchmarkStringConcat-8         100     12345678 ns/op    52428800 B/op   2000 allocs/op
# BenchmarkStringBuilder-8      10000       123456 ns/op       32768 B/op      3 allocs/op
# BenchmarkHashMD5-8          5000000          234 ns/op  4376.07 MB/s
# BenchmarkHashSHA256-8       3000000          345 ns/op  2968.12 MB/s

# Run a specific benchmark
go test -bench=BenchmarkStringBuilder -benchmem -count=5 ./...

# Compare benchmarks with benchstat
# Before optimization:
go test -bench=. -benchmem -count=10 ./... > old.txt
# After optimization:
go test -bench=. -benchmem -count=10 ./... > new.txt

# Install and run benchstat
go install golang.org/x/perf/cmd/benchstat@latest
benchstat old.txt new.txt
# name            old time/op    new time/op    delta
# StringConcat-8  12.3ms ± 2%    0.12ms ± 1%   -99.03%  (p=0.000 n=10+10)
#
# name            old alloc/op   new alloc/op   delta
# StringConcat-8  52.4MB ± 0%    0.03MB ± 0%   -99.94%  (p=0.000 n=10+10)
```

---

## 40.5 Kubernetes Performance Tuning

For large clusters (hundreds of nodes, thousands of pods), default settings
are often insufficient. Here are the key tuning knobs.

### 40.5.1 API Server Performance

```yaml
# kube-apiserver configuration tuning

# API Priority and Fairness (APF) — prevents any single client from
# overwhelming the API server. Replaces the older --max-requests-inflight.
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: FlowSchema
metadata:
  name: high-priority-controllers
spec:
  priorityLevelConfiguration:
    name: workload-high
  matchingPrecedence: 100
  rules:
    - subjects:
        - kind: ServiceAccount
          serviceAccount:
            name: critical-controller
            namespace: kube-system
      resourceRules:
        - apiGroups: ["*"]
          resources: ["*"]
          verbs: ["*"]
---
apiVersion: flowcontrol.apiserver.k8s.io/v1beta3
kind: PriorityLevelConfiguration
metadata:
  name: workload-high
spec:
  type: Limited
  limited:
    nominalConcurrencyShares: 40    # Higher share = more capacity
    lendablePercent: 0
    limitResponse:
      type: Queue
      queuing:
        queues: 64
        handSize: 6
        queueLengthLimit: 50
```

**Key API server flags:**

| Flag                           | Default | Recommendation for Large Clusters |
|-------------------------------|---------|-----------------------------------|
| `--watch-cache-size`          | Auto    | Increase for heavy watch workloads|
| `--default-watch-cache-size`  | 100     | 500-1000 for large clusters       |
| `--max-requests-inflight`     | 400     | 800+ (or use APF)                 |
| `--max-mutating-requests`     | 200     | 400+                              |
| `--audit-log-maxage`          | 30      | Reduce if disk is constrained     |
| `--event-ttl`                 | 1h      | Increase if events are needed     |

### 40.5.2 etcd Performance

etcd performance is critical because every API server operation touches etcd:

```bash
# Check etcd performance metrics
etcdctl endpoint status --cluster --write-table
# +-------------------+------------------+---------+---------+-----------+
# |     ENDPOINT      |        ID        | VERSION | DB SIZE | IS LEADER |
# +-------------------+------------------+---------+---------+-----------+
# | https://10.0.0.10 | 8e9e05c52164694d | 3.5.10  |  256 MB |      true |
# +-------------------+------------------+---------+---------+-----------+

# Check for slow apply warnings
journalctl -u etcd | grep "apply request took too long"

# Disk I/O benchmark (etcd needs < 10ms for fsync)
fio --name=etcd-test --filename=/var/lib/etcd/fio-test \
    --rw=write --ioengine=sync --fdatasync=1 \
    --size=100m --bs=2300 --runtime=60
# Target: 99th percentile fdatasync < 10ms
```

**etcd tuning parameters:**

| Parameter                     | Default    | Recommendation                     |
|------------------------------|------------|-------------------------------------|
| `--quota-backend-bytes`      | 2 GB       | 8 GB for large clusters            |
| `--auto-compaction-retention`| 0 (off)    | 5m (minutes) or 1 (hourly)        |
| `--snapshot-count`           | 100000     | Reduce to 10000 for faster recovery|
| `--heartbeat-interval`       | 100ms      | Keep default or match network RTT  |
| `--election-timeout`         | 1000ms     | 5-10x heartbeat interval           |
| Disk type                    | Any        | SSD/NVMe required for production   |
| Dedicated disk               | No         | Yes — never share etcd's disk      |

### 40.5.3 Scheduler Performance

```yaml
# kube-scheduler configuration for large clusters
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
# percentageOfNodesToScore controls how many nodes the scheduler evaluates
# before making a decision. For 5000-node clusters, evaluating all nodes
# is too slow. Setting this to 10% means the scheduler evaluates ~500 nodes
# and picks the best among those.
percentageOfNodesToScore: 10

profiles:
  - schedulerName: default-scheduler
    plugins:
      score:
        disabled:
          # Disable expensive scoring plugins you don't need
          - name: ImageLocality     # Skip if you use a registry mirror
          - name: InterPodAffinity  # Skip if not using pod affinity
```

### 40.5.4 kubelet Performance

```yaml
# kubelet configuration (/var/lib/kubelet/config.yaml)
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration

# Maximum number of pods on this node (default: 110)
maxPods: 250

# Reduce event recording to decrease API server load
eventRecordQPS: 5            # Default: 50
eventBurst: 10               # Default: 100

# Image garbage collection
imageMinimumGCAge: "2m"      # Minimum age before GC (default: 2m)
imageGCHighThresholdPercent: 85  # Start GC when disk usage exceeds this
imageGCLowThresholdPercent: 80   # Stop GC when disk usage drops below this

# Container log rotation
containerLogMaxSize: "50Mi"   # Max size per container log file
containerLogMaxFiles: 5       # Max number of log files per container

# CPU and memory manager (for latency-sensitive workloads)
cpuManagerPolicy: "static"    # Pin exclusive CPUs to guaranteed QoS pods
memoryManagerPolicy: "Static" # Reserve memory for guaranteed QoS pods
topologyManagerPolicy: "best-effort"  # NUMA-aware placement

# Eviction thresholds
evictionHard:
  memory.available: "500Mi"
  nodefs.available: "10%"
  imagefs.available: "15%"
evictionSoft:
  memory.available: "1Gi"
  nodefs.available: "15%"
evictionSoftGracePeriod:
  memory.available: "1m"
  nodefs.available: "2m"

# Node status update frequency (reduce for large clusters)
nodeStatusUpdateFrequency: "20s"    # Default: 10s
nodeStatusReportFrequency: "5m"     # Default: 5m
```

### 40.5.5 Network Performance

```bash
# Check current MTU
ip link show eth0 | grep mtu
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500

# For overlay networks, the encapsulation overhead reduces effective MTU:
# VXLAN: 50 bytes overhead → effective MTU = 1450
# Geneve: 50 bytes overhead → effective MTU = 1450
# WireGuard: 60 bytes overhead → effective MTU = 1440

# If your cloud provider supports jumbo frames (MTU 9001):
# Set node MTU to 9001 and pod MTU to 8951 (for VXLAN)

# kube-proxy modes
# iptables (default): Good for < 5000 services
# IPVS: Better for > 5000 services (O(1) lookup vs O(n) iptables chains)
# Change mode:
kubectl edit configmap kube-proxy -n kube-system
# mode: "ipvs"
# ipvs:
#   scheduler: "rr"       # Round-robin (or lc, sh, dh, wrr)
#   strictARP: true       # Required for MetalLB
kubectl rollout restart daemonset kube-proxy -n kube-system
```

### 40.5.6 Hands-on: Tuning Cluster Parameters

```bash
# Step 1: Baseline measurement
# Check API server response times
kubectl get --raw /readyz?verbose
time kubectl get pods -A > /dev/null

# Step 2: Check API server metrics
kubectl get --raw /metrics | grep apiserver_request_duration_seconds

# Step 3: Check etcd latency
kubectl -n kube-system exec etcd-controlplane -- etcdctl \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  endpoint status --write-table

# Step 4: Check scheduler latency
kubectl get --raw /metrics | grep scheduler_scheduling_attempt_duration_seconds

# Step 5: Apply tuning (example: increase watch cache)
# Edit API server manifest on control plane node:
# /etc/kubernetes/manifests/kube-apiserver.yaml
# Add: --default-watch-cache-size=500

# Step 6: Verify improvement
time kubectl get pods -A > /dev/null
# Should be faster with larger watch cache
```

---

## 40.6 Resource Right-Sizing

Over-provisioned resources waste money. Under-provisioned resources cause
OOMKills and throttling. Right-sizing means finding the sweet spot.

### 40.6.1 Using VPA Recommendations

The Vertical Pod Autoscaler (VPA) observes actual resource usage and
recommends request/limit values:

```yaml
# vpa.yaml — Create a VPA to monitor a deployment
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: api-server-vpa
  namespace: production
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  updatePolicy:
    updateMode: "Off"  # "Off" = recommend only; "Auto" = auto-apply
  resourcePolicy:
    containerPolicies:
      - containerName: "*"
        minAllowed:
          cpu: 100m
          memory: 128Mi
        maxAllowed:
          cpu: 4
          memory: 8Gi
        controlledResources: ["cpu", "memory"]
```

```bash
# Check VPA recommendations after it has collected data (wait ~24 hours)
kubectl describe vpa api-server-vpa -n production
# Recommendation:
#   Container Recommendations:
#     Container Name:  app
#     Lower Bound:
#       Cpu:     250m
#       Memory:  384Mi
#     Target:
#       Cpu:     500m
#       Memory:  768Mi       ← Use this for requests
#     Uncapped Target:
#       Cpu:     500m
#       Memory:  768Mi
#     Upper Bound:
#       Cpu:     2
#       Memory:  2Gi         ← Use this for limits
```

### 40.6.2 Goldilocks — VPA Dashboard

Goldilocks runs VPA in recommendation mode for every deployment and provides
a web dashboard:

```bash
# Install Goldilocks
helm repo add fairwinds-stable https://charts.fairwinds.com/stable
helm install goldilocks fairwinds-stable/goldilocks \
  --namespace goldilocks --create-namespace

# Enable Goldilocks for a namespace
kubectl label namespace production goldilocks.fairwinds.com/enabled=true

# Access the dashboard
kubectl port-forward -n goldilocks svc/goldilocks-dashboard 8080:80
# Open http://localhost:8080
# The dashboard shows recommended CPU/memory for every container
```

### 40.6.3 Resource Analysis with Kubecost

```bash
# Install Kubecost
helm install kubecost cost-analyzer \
  --repo https://kubecost.github.io/cost-analyzer/ \
  --namespace kubecost --create-namespace

# Access the dashboard
kubectl port-forward -n kubecost svc/kubecost-cost-analyzer 9090:9090
# Open http://localhost:9090

# Kubecost shows:
# - Cost per namespace, deployment, pod
# - Right-sizing recommendations
# - Idle resource identification
# - Cost trends over time
```

### 40.6.4 Hands-on: Right-Sizing Workflow

```bash
# Step 1: Identify over-provisioned deployments
kubectl top pods -n production --sort-by=memory
# NAME              CPU(cores)  MEMORY(bytes)
# api-server-abc    50m         256Mi          ← Request: 2Gi → Huge waste!
# worker-xyz        1500m       1024Mi         ← Request: 1Gi → Under-provisioned CPU

# Step 2: Get current resource settings
kubectl get deployment api-server -n production -o jsonpath='{.spec.template.spec.containers[0].resources}'
# {"limits":{"cpu":"4","memory":"4Gi"},"requests":{"cpu":"2","memory":"2Gi"}}

# Step 3: Check VPA recommendations
kubectl describe vpa api-server-vpa -n production | grep -A10 "Target:"
# Target: cpu=500m, memory=768Mi

# Step 4: Check historical usage (Prometheus query)
# rate(container_cpu_usage_seconds_total{pod=~"api-server.*"}[5m])
# Max over 7 days: 400m CPU, 650Mi memory

# Step 5: Set right-sized values
# Requests = VPA target (covers normal load)
# Limits = VPA upper bound or 2x target (covers spikes)
kubectl set resources deployment/api-server -n production \
  --requests=cpu=500m,memory=768Mi \
  --limits=cpu=2,memory=2Gi

# Step 6: Monitor after change
# Watch for OOMKills or throttling over the next week
kubectl get events -n production --field-selector reason=OOMKilling
kubectl top pods -n production -l app=api-server
```

---

## 40.7 Common Troubleshooting Scenarios

This section is a decision tree for the most common Kubernetes issues. For
each scenario, we list the symptoms, root causes, and step-by-step resolution.

### 40.7.1 Pod Stuck in Pending

**Symptoms:** Pod shows `Pending` status and never transitions to `Running`.

```bash
# Diagnosis
kubectl describe pod <pod-name>
# Look at Events section for clues

# Common causes and fixes:

# 1. Insufficient resources
# Event: "0/5 nodes are available: 5 Insufficient cpu"
# Fix: Add nodes, reduce requests, or delete unused pods
kubectl get nodes -o custom-columns="NAME:.metadata.name,CPU:.status.allocatable.cpu,MEM:.status.allocatable.memory"

# 2. Node selector / affinity not satisfied
# Event: "0/5 nodes are available: 5 node(s) didn't match node selector"
# Fix: Add labels to nodes or adjust selector
kubectl label node node-3 gpu=true

# 3. Taints and tolerations
# Event: "0/5 nodes are available: 5 node(s) had taint {key=value:NoSchedule}"
# Fix: Add toleration to pod spec or remove taint from node
kubectl taint node node-3 key=value:NoSchedule-

# 4. PVC not bound
# Event: "pod has unbound immediate PersistentVolumeClaims"
# Fix: Check PVC status, storage class, and available PVs
kubectl get pvc -n <namespace>
```

### 40.7.2 Pod Stuck in ContainerCreating

**Symptoms:** Pod shows `ContainerCreating` for more than a few seconds.

```bash
# Diagnosis
kubectl describe pod <pod-name>

# Common causes:

# 1. Image pull taking too long (large image)
# Event: "Pulling image 'myregistry.io/huge-image:latest'"
# Fix: Use smaller images, pre-pull on nodes, use image cache

# 2. Volume mount failure
# Event: "Unable to attach or mount volumes"
# Fix: Check PV/PVC status, storage driver logs
kubectl get pv,pvc -n <namespace>
kubectl describe pv <pv-name>

# 3. CNI plugin failure
# Event: "Failed to create pod sandbox: rpc error: ... network not ready"
# Fix: Check CNI plugin pods, node network configuration
kubectl get pods -n kube-system -l k8s-app=calico-node
kubectl logs -n kube-system <cni-pod>

# 4. Secret/ConfigMap not found
# Event: "Error: secret 'my-secret' not found"
# Fix: Create the missing secret/configmap
kubectl get secrets,configmaps -n <namespace>
```

### 40.7.3 CrashLoopBackOff

**Symptoms:** Pod repeatedly starts and crashes.

```bash
# Check exit code and reason
kubectl describe pod <pod-name> | grep -A5 "Last State"
# Last State:     Terminated
#   Reason:       Error
#   Exit Code:    1         ← Application error
#   Exit Code:    137       ← OOMKilled (128 + 9 = SIGKILL)
#   Exit Code:    143       ← SIGTERM (graceful shutdown failed)

# Check logs
kubectl logs <pod-name> --previous

# Common fixes:
# - Exit code 1: Application bug → fix the code
# - Exit code 137: OOMKilled → increase memory limits
# - Exit code 143: Slow shutdown → increase terminationGracePeriodSeconds
# - No logs at all: Entrypoint/command misconfigured → check image CMD/ENTRYPOINT
```

### 40.7.4 OOMKilled

```bash
# Confirm OOMKill
kubectl describe pod <pod-name> | grep -B2 "OOMKilled"
# Reason: OOMKilled

# Check memory limit vs usage
kubectl top pod <pod-name> --containers
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].resources.limits.memory}'

# Node-level investigation
kubectl debug node/<node> -it --image=busybox
chroot /host
dmesg -T | grep -i oom

# Fix: Either optimize memory usage (pprof heap profile) or increase limits
```

### 40.7.5 Evicted

```bash
# List evicted pods
kubectl get pods -A --field-selector=status.phase=Failed | grep Evicted

# Check why
kubectl describe pod <evicted-pod> | grep -A2 "Status\|Message"
# Status: Failed
# Reason: Evicted
# Message: The node was low on resource: memory

# Check node conditions
kubectl describe node <node> | grep -A10 "Conditions"
# MemoryPressure: True    ← Node ran out of memory
# DiskPressure: True      ← Node ran out of disk

# Fix: Add resources to node, reduce workload, tune eviction thresholds
# Clean up evicted pods:
kubectl get pods -A --field-selector=status.phase=Failed -o json | \
  jq -r '.items[] | select(.status.reason=="Evicted") | "\(.metadata.namespace) \(.metadata.name)"' | \
  xargs -L1 bash -c 'kubectl delete pod $1 -n $0'
```

### 40.7.6 Node NotReady

```bash
# Check node status
kubectl get nodes
kubectl describe node <node> | grep -A20 "Conditions"

# Common causes:

# 1. kubelet not running
ssh node-3
systemctl status kubelet
journalctl -u kubelet --no-pager -n 50

# 2. Certificate expired
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -noout -dates
# If expired: kubeadm certs renew all

# 3. Network partition
# From another node: ping <node-ip>
# Check network interfaces: ip addr show

# 4. Disk pressure
df -h
# If disk full: clean up logs, old images

# 5. Container runtime crash
systemctl status containerd
journalctl -u containerd --no-pager -n 50
```

### 40.7.7 Service Not Accessible

```bash
# Follow the network debugging flowchart (Section 40.3.7)
# Quick checks:
kubectl get svc <name> -n <ns>        # Service exists?
kubectl get endpoints <name> -n <ns>   # Has endpoints?
kubectl get pods -l <selector> -n <ns> # Pods running?
kubectl get networkpolicy -n <ns>      # Policies blocking?

# Test from inside cluster
kubectl run test-curl --rm -it --image=curlimages/curl -- \
  curl -v http://<service-name>.<namespace>.svc.cluster.local:<port>/
```

### 40.7.8 PVC Stuck in Pending

```bash
# Check PVC status
kubectl describe pvc <name> -n <ns>

# Common causes:
# 1. No matching PV (for static provisioning)
# Event: "no persistent volumes available for this claim"
kubectl get pv

# 2. Storage class doesn't exist
kubectl get sc
kubectl describe pvc <name> -n <ns> | grep StorageClass

# 3. Volume plugin issue
kubectl get pods -n kube-system | grep csi
kubectl logs -n kube-system <csi-pod>

# 4. Capacity exhausted (cloud provider)
# Check cloud provider quotas and limits
```

### 40.7.9 Hands-on: Decision Tree for Each Scenario

```
Pod Issue?
│
├─ Status: Pending
│  ├─ Insufficient resources → Add nodes or reduce requests
│  ├─ Node selector/affinity → Fix labels or selectors
│  ├─ Taints → Add tolerations or remove taints
│  └─ Unbound PVC → Fix PVC (see PVC section)
│
├─ Status: ContainerCreating
│  ├─ Image pull → Fix image name, credentials, or network
│  ├─ Volume mount → Fix PV/PVC, check CSI driver
│  ├─ CNI error → Check CNI pods and logs
│  └─ Missing secret/configmap → Create the resource
│
├─ Status: Running but unhealthy
│  ├─ Liveness probe failing → Fix health endpoint or probe config
│  ├─ Readiness probe failing → Fix readiness check
│  └─ Startup probe failing → Increase startup timeout
│
├─ Status: CrashLoopBackOff
│  ├─ Exit code 1 → Check logs, fix application bug
│  ├─ Exit code 137 → OOMKilled — increase memory or optimize
│  ├─ Exit code 143 → Slow shutdown — increase grace period
│  └─ No logs → Check command/entrypoint configuration
│
├─ Status: Evicted
│  ├─ Memory pressure → Add memory to node or reduce usage
│  └─ Disk pressure → Clean up disk, increase storage
│
└─ Status: Completed (shouldn't be running)
   └─ Check if this is a Job/CronJob pod (expected behavior)
```

---

## 40.8 Chaos Engineering

Chaos engineering is the practice of deliberately injecting failures into a
system to build confidence in its ability to withstand turbulent conditions
in production.

### 40.8.1 Principles of Chaos Engineering

From the Netflix team that pioneered the discipline:

1. **Start with a hypothesis.** "If we kill one replica, the remaining replicas
   will handle the load and users won't notice."
2. **Vary real-world events.** Kill pods, inject network latency, fill disks,
   exhaust CPU.
3. **Run experiments in production** (when safe). Staging environments often
   have different characteristics.
4. **Automate experiments** to run continuously. Manual chaos isn't sustainable.
5. **Minimize blast radius.** Start small, expand gradually.

### 40.8.2 Tools: Chaos Mesh, LitmusChaos

**Chaos Mesh** (CNCF project):

```bash
# Install Chaos Mesh
helm repo add chaos-mesh https://charts.chaos-mesh.org
helm install chaos-mesh chaos-mesh/chaos-mesh \
  --namespace chaos-mesh --create-namespace \
  --set chaosDaemon.runtime=containerd \
  --set chaosDaemon.socketPath=/run/containerd/containerd.sock
```

**LitmusChaos** (CNCF project):

```bash
# Install LitmusChaos
helm repo add litmuschaos https://litmuschaos.github.io/litmus-helm/
helm install litmus litmuschaos/litmus \
  --namespace litmus --create-namespace
```

### 40.8.3 Pod Failure Injection, Network Partition, I/O Latency

**Pod kill experiment (Chaos Mesh):**

```yaml
# pod-kill.yaml — Randomly kills one pod matching the selector every 60 seconds.
# This tests whether the deployment's self-healing (ReplicaSet) works correctly
# and whether the service remains available during pod restarts.
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-test
  namespace: chaos-mesh
spec:
  action: pod-kill
  mode: one                    # Kill one random pod
  selector:
    namespaces:
      - production
    labelSelectors:
      app: api-server
  scheduler:
    cron: "@every 60s"         # Repeat every 60 seconds
  duration: "5m"               # Run for 5 minutes total
```

**Network partition experiment:**

```yaml
# network-partition.yaml — Simulates a network partition between the API
# server and the database. Tests whether the application handles database
# unreachability gracefully (circuit breaker, retry, degraded mode).
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-partition-test
  namespace: chaos-mesh
spec:
  action: partition
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: api-server
  direction: both
  target:
    selector:
      namespaces:
        - production
      labelSelectors:
        app: postgres
    mode: all
  duration: "2m"
```

**Network latency injection:**

```yaml
# network-delay.yaml — Injects 200ms of latency (±50ms jitter) on all
# traffic from the API server pods. Tests whether the application handles
# increased latency without cascading failures.
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: network-delay-test
  namespace: chaos-mesh
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: api-server
  delay:
    latency: "200ms"
    jitter: "50ms"
    correlation: "75"
  duration: "5m"
```

**Disk I/O latency injection:**

```yaml
# io-delay.yaml — Injects 100ms of latency on all read operations for
# the database pods. Tests how the application performs under slow disk I/O.
apiVersion: chaos-mesh.org/v1alpha1
kind: IOChaos
metadata:
  name: io-delay-test
  namespace: chaos-mesh
spec:
  action: latency
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: postgres
  volumePath: /var/lib/postgresql/data
  path: "*"
  delay: "100ms"
  percent: 100
  duration: "5m"
```

**CPU stress experiment:**

```yaml
# cpu-stress.yaml — Consumes CPU on selected pods to test behavior
# under CPU pressure. Validates that CPU limits and throttling work
# as expected and that the application degrades gracefully.
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  name: cpu-stress-test
  namespace: chaos-mesh
spec:
  mode: all
  selector:
    namespaces:
      - production
    labelSelectors:
      app: api-server
  stressors:
    cpu:
      workers: 2           # Number of CPU stress workers
      load: 80             # Target 80% CPU load
  duration: "5m"
```

### 40.8.4 Hands-on: Basic Chaos Experiments

```bash
# ── Experiment 1: Pod Kill ──

# Step 1: Deploy a sample application
kubectl create namespace chaos-test
kubectl -n chaos-test apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  template:
    metadata:
      labels:
        app: web-app
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: web-app
spec:
  selector:
    app: web-app
  ports:
    - port: 80
EOF

# Step 2: Verify baseline
kubectl -n chaos-test get pods
# 3/3 Running

# Step 3: Start continuous monitoring in another terminal
watch -n 1 'kubectl -n chaos-test get pods && echo "---" && \
  kubectl -n chaos-test get endpoints web-app'

# Step 4: Apply pod kill chaos
kubectl apply -f - <<EOF
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  name: pod-kill-demo
  namespace: chaos-test
spec:
  action: pod-kill
  mode: one
  selector:
    namespaces:
      - chaos-test
    labelSelectors:
      app: web-app
  duration: "30s"
EOF

# Step 5: Observe
# One pod gets killed → ReplicaSet creates a new one → Service stays available
# The key question: Did any requests fail during the pod kill?

# Step 6: Test with curl during chaos
for i in $(seq 1 100); do
  curl -s -o /dev/null -w "%{http_code}\n" http://web-app.chaos-test.svc:80/
  sleep 0.1
done | sort | uniq -c
# Expected: 100 200 (all successful if readiness probe is configured)

# Step 7: Clean up
kubectl -n chaos-test delete podchaos pod-kill-demo

# ── Experiment 2: Network Delay ──

# Step 1: Apply network delay
kubectl apply -f - <<EOF
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  name: delay-demo
  namespace: chaos-test
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - chaos-test
    labelSelectors:
      app: web-app
  delay:
    latency: "500ms"
    jitter: "100ms"
  duration: "2m"
EOF

# Step 2: Measure latency
for i in $(seq 1 10); do
  curl -s -o /dev/null -w "HTTP %{http_code} in %{time_total}s\n" \
    http://web-app.chaos-test.svc:80/
done
# HTTP 200 in 0.523s    ← ~500ms added latency
# HTTP 200 in 0.612s
# HTTP 200 in 0.489s

# Step 3: Clean up
kubectl -n chaos-test delete networkchaos delay-demo
kubectl delete namespace chaos-test
```

---

## 40.9 Summary

| Area                    | Key Tools / Techniques                                         |
|------------------------|---------------------------------------------------------------|
| Node debugging         | top, htop, vmstat, iostat, dmesg, /proc, kubectl debug node   |
| Pod debugging          | kubectl logs, exec, debug, cp; ephemeral containers           |
| Network debugging      | nslookup, dig, curl, tcpdump, iptables, netshoot              |
| Go profiling           | pprof (CPU, heap, goroutine), runtime/trace, testing.B        |
| K8s performance        | APF, etcd tuning, scheduler tuning, kubelet config            |
| Resource right-sizing  | VPA, Goldilocks, Kubecost                                     |
| Troubleshooting        | Decision trees for Pending, CrashLoopBackOff, OOMKilled, etc. |
| Chaos engineering      | Chaos Mesh, LitmusChaos — pod kill, network, I/O experiments  |

**The debugging mindset:**

1. **Observe** — Gather data before forming hypotheses.
2. **Hypothesize** — Form a theory based on the data.
3. **Test** — Validate the theory with targeted investigation.
4. **Fix** — Apply the smallest change that resolves the issue.
5. **Verify** — Confirm the fix works and doesn't break anything else.
6. **Document** — Update runbooks so the next person solves it faster.

---

## 40.10 Exercises

1. **Node Debugging**: Deploy a memory-intensive workload that triggers the
   OOM killer on a node. Use dmesg and kubectl to trace the entire event chain
   from kernel kill signal to pod restart.

2. **pprof Mastery**: Build a Go HTTP service with a deliberately leaky goroutine
   pool. Use pprof to identify the leak, fix it, and verify the fix with before/after
   goroutine profiles.

3. **Network Debugging**: Deploy two services in different namespaces with a
   NetworkPolicy that blocks traffic between them. Use the debugging flowchart
   to identify and fix the issue.

4. **Benchmarking**: Write benchmarks for three different JSON serialization
   approaches (encoding/json, json-iterator, easyjson). Use benchstat to
   compare them and explain the results.

5. **Chaos Experiment**: Design and run a chaos experiment that tests your
   application's behavior when 50% of database connections are dropped. Document
   the hypothesis, observations, and improvements made.

6. **Right-Sizing**: Deploy a sample application with deliberately oversized
   resource requests (4 CPU, 8Gi memory for a lightweight service). Use VPA
   and kubectl top to determine the right-sized values and calculate the cost
   savings.

7. **Complete Troubleshooting**: Given a cluster where `kubectl` commands are
   extremely slow (>10 seconds), systematically debug from the API server
   through etcd to identify the bottleneck. Document each step.
