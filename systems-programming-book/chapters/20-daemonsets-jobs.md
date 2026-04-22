# Chapter 20: DaemonSets, Jobs & CronJobs

> *"Not every workload is a long-running server. Some must run everywhere,
> some must run once, and some must run on a schedule."*

Chapters 17–19 covered workload controllers for long-running processes:
Deployments for stateless services and StatefulSets for stateful ones. But
Kubernetes supports two more fundamental workload patterns that every
production cluster depends on:

1. **DaemonSets** — run exactly one Pod on every Node (or a subset of Nodes).
   Think log collectors, monitoring agents, and network plugins.

2. **Jobs and CronJobs** — run-to-completion workloads. A Job runs a task
   until it succeeds (or exhausts its retry budget). A CronJob runs Jobs on a
   schedule.

These controllers are deceptively simple in concept but rich in configuration
and behavior. This chapter covers them in depth, with production-ready YAML
manifests and Go programs for each pattern.

---

## 20.1 DaemonSets

### 20.1.1 Purpose: One Pod Per Node

A DaemonSet ensures that **every Node** (or every Node matching a selector)
runs exactly **one copy** of a Pod. When a new Node joins the cluster, the
DaemonSet controller automatically schedules a Pod on it. When a Node is
removed, the Pod is garbage-collected.

This "one per Node" guarantee makes DaemonSets the natural choice for
**infrastructure-level concerns** that must be present on every machine:

| Use Case                      | Example                                    |
|-------------------------------|--------------------------------------------|
| **Log collection**            | Fluentd, Fluent Bit, Filebeat              |
| **Node monitoring**           | Prometheus node-exporter, Datadog agent    |
| **Network plugins (CNI)**     | Calico, Cilium, Flannel                    |
| **Storage daemons**           | GlusterFS, Ceph OSD                       |
| **Security agents**           | Falco, Sysdig                              |
| **Device plugins**            | NVIDIA GPU plugin, FPGA plugin             |

### 20.1.2 How the DaemonSet Controller Works

The DaemonSet controller runs a reconciliation loop that is conceptually
simple:

```
LOOP forever:
    desiredNodes = all Nodes matching nodeSelector/nodeAffinity
    currentPods  = all Pods owned by this DaemonSet

    for each Node in desiredNodes:
        if no Pod exists on this Node:
            create a Pod scheduled to this Node
        if Pod exists but spec is outdated:
            apply updateStrategy (RollingUpdate or OnDelete)

    for each Pod in currentPods:
        if the Pod's Node is NOT in desiredNodes:
            delete the Pod (node was removed or no longer matches selector)
```

Key implementation details:

1. **The controller sets `spec.nodeName` directly.** Unlike Deployments (where
   the scheduler assigns Pods to Nodes), the DaemonSet controller bypasses
   the scheduler entirely and sets the target Node in the Pod spec. This
   ensures Pods run even on Nodes the scheduler might skip (e.g., due to
   taints).

2. **Tolerations are critical.** By default, Kubernetes taints control-plane
   Nodes with `node-role.kubernetes.io/control-plane:NoSchedule`. A DaemonSet
   Pod must have a matching toleration to run on control-plane Nodes. Many
   infrastructure DaemonSets (like CNI plugins) need this.

3. **Resource pressure is respected.** If a Node is under memory or disk
   pressure, the kubelet may evict DaemonSet Pods. However, DaemonSet Pods
   are given priority class `system-node-critical` or `system-cluster-critical`
   for critical infrastructure.

### 20.1.3 DaemonSet Spec Deep Dive

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-monitor
  namespace: monitoring
  labels:
    app: node-monitor
spec:
  # -----------------------------------------------------------
  # selector: Label selector that must match the Pod template.
  # Immutable after creation.
  # -----------------------------------------------------------
  selector:
    matchLabels:
      app: node-monitor

  # -----------------------------------------------------------
  # updateStrategy: Controls how Pods are updated when the
  # DaemonSet spec changes.
  #
  #   RollingUpdate (default): Pods are updated one Node at a
  #     time. maxUnavailable controls how many can be down.
  #     maxSurge controls how many extra Pods can exist.
  #
  #   OnDelete: Pods are only updated when manually deleted.
  # -----------------------------------------------------------
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 0

  # -----------------------------------------------------------
  # revisionHistoryLimit: Number of old DaemonSet revisions to
  # retain for rollback. Defaults to 10.
  # -----------------------------------------------------------
  revisionHistoryLimit: 10

  # -----------------------------------------------------------
  # minReadySeconds: Minimum seconds a Pod must be Ready without
  # crashing before it is considered Available. Defaults to 0.
  # -----------------------------------------------------------
  minReadySeconds: 10

  template:
    metadata:
      labels:
        app: node-monitor
    spec:
      # -------------------------------------------------------
      # nodeSelector: Simple key-value selector to restrict which
      # Nodes get DaemonSet Pods. Only Nodes with ALL specified
      # labels will run a Pod.
      # -------------------------------------------------------
      nodeSelector:
        kubernetes.io/os: linux

      # -------------------------------------------------------
      # affinity.nodeAffinity: Advanced node selection rules.
      # More expressive than nodeSelector — supports In, NotIn,
      # Exists, DoesNotExist, Gt, Lt operators.
      # -------------------------------------------------------
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: kubernetes.io/arch
                    operator: In
                    values:
                      - amd64
                      - arm64

      # -------------------------------------------------------
      # tolerations: Allow DaemonSet Pods to run on tainted Nodes.
      # Without these, Pods won't be scheduled on control-plane
      # Nodes or Nodes with custom taints.
      # -------------------------------------------------------
      tolerations:
        # Tolerate control-plane taint (run on master nodes too)
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
        # Tolerate not-ready nodes (important during cluster bootstrap)
        - key: node.kubernetes.io/not-ready
          operator: Exists
          effect: NoSchedule
        # Tolerate unreachable nodes
        - key: node.kubernetes.io/unreachable
          operator: Exists
          effect: NoExecute
        # Tolerate disk pressure
        - key: node.kubernetes.io/disk-pressure
          operator: Exists
          effect: NoSchedule

      # -------------------------------------------------------
      # hostNetwork/hostPID/hostIPC: DaemonSets often need access
      # to the host's network namespace (for port binding),
      # process namespace (for monitoring), or IPC namespace.
      # -------------------------------------------------------
      hostNetwork: false
      hostPID: false

      # -------------------------------------------------------
      # priorityClassName: Give infrastructure DaemonSets high
      # priority so they are not preempted by application workloads.
      # -------------------------------------------------------
      priorityClassName: system-node-critical

      containers:
        - name: monitor
          image: prom/node-exporter:v1.7.0
          ports:
            - containerPort: 9100
              name: metrics
              hostPort: 9100  # Bind to host port for Prometheus scraping
          args:
            - "--path.procfs=/host/proc"
            - "--path.sysfs=/host/sys"
            - "--path.rootfs=/host/root"
            - "--collector.filesystem.mount-points-exclude=^/(dev|proc|sys|var/lib/docker/.+|var/lib/kubelet/.+)($|/)"
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
            - name: sys
              mountPath: /host/sys
              readOnly: true
            - name: root
              mountPath: /host/root
              readOnly: true
              mountPropagation: HostToContainer
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
          securityContext:
            readOnlyRootFilesystem: true
            allowPrivilegeEscalation: false
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
        - name: root
          hostPath:
            path: /
```

### 20.1.4 Targeting Specific Nodes

Not all DaemonSets should run on every Node. Use `nodeSelector` and
`nodeAffinity` to target subsets:

**Example: GPU monitoring only on GPU Nodes:**

```yaml
spec:
  template:
    spec:
      nodeSelector:
        # Only run on Nodes labeled with GPU capability
        gpu: "true"
      tolerations:
        # GPU nodes often have a taint to prevent non-GPU workloads
        - key: nvidia.com/gpu
          operator: Exists
          effect: NoSchedule
```

**Example: Run on all Nodes except control-plane:**

```yaml
spec:
  template:
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
              - matchExpressions:
                  - key: node-role.kubernetes.io/control-plane
                    operator: DoesNotExist
```

### 20.1.5 Update Strategies

**RollingUpdate** (default):

The controller updates Pods one Node at a time:

1. Terminates the old Pod on a Node.
2. Creates the new Pod on the same Node.
3. Waits until the new Pod is Ready.
4. Moves to the next Node.

`maxUnavailable` controls how many Nodes can have their DaemonSet Pod down
simultaneously (default: 1). `maxSurge` controls how many extra Pods can
exist during the update (default: 0). With `maxSurge: 1`, the new Pod is
created *before* the old one is terminated — useful for zero-downtime updates
of critical infrastructure.

**OnDelete:**

The controller does not automatically update Pods. You must manually delete
each Pod to trigger recreation with the new spec. This is useful for
infrastructure components where you want full control over the rollout (e.g.,
updating CNI plugins node by node with manual validation).

### 20.1.6 Hands-on: Deploy a Node Monitoring DaemonSet

```bash
# Apply the node-exporter DaemonSet from section 20.1.3
kubectl apply -f node-monitor-daemonset.yaml

# Verify one Pod per Node
kubectl get pods -n monitoring -o wide -l app=node-monitor
# NAME                 READY   STATUS    NODE
# node-monitor-abc12   1/1     Running   node-1
# node-monitor-def34   1/1     Running   node-2
# node-monitor-ghi56   1/1     Running   node-3

# Verify Pod count matches Node count
echo "Pods: $(kubectl get pods -n monitoring -l app=node-monitor --no-headers | wc -l)"
echo "Nodes: $(kubectl get nodes --no-headers | wc -l)"

# Check metrics endpoint from any Pod
kubectl exec -n monitoring $(kubectl get pods -n monitoring -l app=node-monitor -o name | head -1) \
  -- wget -qO- http://localhost:9100/metrics | head -20
```

### 20.1.7 Hands-on: Deploy a Log Collector DaemonSet

```yaml
# ------------------------------------------------------------------
# fluentbit-daemonset.yaml
# Fluent Bit log collector — collects container logs from every Node
# and forwards them to an Elasticsearch cluster.
# ------------------------------------------------------------------
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluent-bit
  namespace: logging
  labels:
    app: fluent-bit
spec:
  selector:
    matchLabels:
      app: fluent-bit
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: fluent-bit
    spec:
      # Run on all Nodes, including control-plane
      tolerations:
        - key: node-role.kubernetes.io/control-plane
          operator: Exists
          effect: NoSchedule
      serviceAccountName: fluent-bit
      containers:
        - name: fluent-bit
          image: fluent/fluent-bit:2.2
          ports:
            - containerPort: 2020
              name: metrics
          volumeMounts:
            # -----------------------------------------------
            # Mount the Node's /var/log directory to access
            # container logs written by the container runtime.
            # -----------------------------------------------
            - name: varlog
              mountPath: /var/log
              readOnly: true
            # -----------------------------------------------
            # Mount the container runtime's log directory.
            # The exact path depends on the runtime (Docker,
            # containerd, CRI-O).
            # -----------------------------------------------
            - name: containers
              mountPath: /var/lib/docker/containers
              readOnly: true
            # -----------------------------------------------
            # Fluent Bit configuration file.
            # -----------------------------------------------
            - name: config
              mountPath: /fluent-bit/etc/
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 256Mi
      volumes:
        - name: varlog
          hostPath:
            path: /var/log
        - name: containers
          hostPath:
            path: /var/lib/docker/containers
        - name: config
          configMap:
            name: fluent-bit-config

---
# ------------------------------------------------------------------
# fluent-bit-configmap.yaml
# Configuration for Fluent Bit — reads Kubernetes container logs,
# enriches them with Kubernetes metadata, and outputs to Elasticsearch.
# ------------------------------------------------------------------
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit-config
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush         5
        Daemon        Off
        Log_Level     info
        Parsers_File  parsers.conf
        HTTP_Server   On
        HTTP_Listen   0.0.0.0
        HTTP_Port     2020

    [INPUT]
        Name              tail
        Tag               kube.*
        Path              /var/log/containers/*.log
        Parser            docker
        DB                /var/log/flb_kube.db
        Mem_Buf_Limit     5MB
        Skip_Long_Lines   On
        Refresh_Interval  10

    [FILTER]
        Name                kubernetes
        Match               kube.*
        Kube_URL            https://kubernetes.default.svc:443
        Kube_CA_File        /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
        Kube_Token_File     /var/run/secrets/kubernetes.io/serviceaccount/token
        Merge_Log           On
        K8S-Logging.Parser  On
        K8S-Logging.Exclude On

    [OUTPUT]
        Name            es
        Match           *
        Host            elasticsearch.logging.svc.cluster.local
        Port            9200
        Logstash_Format On
        Replace_Dots    On
        Retry_Limit     False

  parsers.conf: |
    [PARSER]
        Name        docker
        Format      json
        Time_Key    time
        Time_Format %Y-%m-%dT%H:%M:%S.%L
        Time_Keep   On
```

### 20.1.8 Hands-on: Go Program for Node Metrics Collection

This Go program collects basic node metrics (CPU, memory, disk, network) and
exposes them as a Prometheus-compatible `/metrics` endpoint. It is designed to
run as a DaemonSet Pod with access to the host's `/proc` and `/sys`
filesystems.

```go
// ================================================================
// main.go — Node Metrics Collector for Kubernetes DaemonSets
// ================================================================
//
// This program collects system-level metrics from the underlying
// Node by reading Linux pseudo-filesystems (/proc, /sys). It is
// designed to run as a DaemonSet Pod, with the host's /proc and
// /sys mounted into the container.
//
// Exposed endpoint:
//   GET /metrics — Prometheus-compatible metrics
//   GET /healthz — Health check
//
// Environment variables:
//   HOST_PROC — Path to the host's /proc (default: /host/proc)
//   HOST_SYS — Path to the host's /sys  (default: /host/sys)
//   NODE_NAME — Name of the Kubernetes Node (via downward API)
//   LISTEN_ADDR — Address to listen on (default: :9100)
//
// Metrics collected:
//   - node_cpu_seconds_total (per-CPU, per-mode)
//   - node_memory_total_bytes
//   - node_memory_available_bytes
//   - node_filesystem_size_bytes (per-mountpoint)
//   - node_filesystem_free_bytes (per-mountpoint)
//   - node_network_receive_bytes_total (per-interface)
//   - node_network_transmit_bytes_total (per-interface)
//   - node_load1, node_load5, node_load15
//
// Build:
//   CGO_ENABLED=0 GOOS=linux go build -o node-collector main.go
//
// ================================================================

package main

import (
	"bufio"
	"fmt"
	"log"
	"net/http"
	"os"
	"path/filepath"
	"strconv"
	"strings"
	"time"
)

// ----------------------------------------------------------------
// config holds the runtime configuration for the collector,
// populated from environment variables at startup.
// ----------------------------------------------------------------
type config struct {
	// hostProc is the path to the host's /proc filesystem,
	// typically mounted into the container at /host/proc.
	hostProc string

	// hostSys is the path to the host's /sys filesystem,
	// typically mounted into the container at /host/sys.
	hostSys string

	// nodeName is the Kubernetes Node name, injected via the
	// downward API. Used as a label in metric output.
	nodeName string

	// listenAddr is the address:port the HTTP server binds to.
	listenAddr string
}

// loadConfig reads configuration from environment variables,
// applying defaults where values are not set.
func loadConfig() config {
	c := config{
		hostProc:   getEnv("HOST_PROC", "/host/proc"),
		hostSys:    getEnv("HOST_SYS", "/host/sys"),
		nodeName:   getEnv("NODE_NAME", "unknown"),
		listenAddr: getEnv("LISTEN_ADDR", ":9100"),
	}
	return c
}

// getEnv returns the value of the named environment variable, or
// the provided default if the variable is empty or unset.
func getEnv(key, defaultVal string) string {
	if val := os.Getenv(key); val != "" {
		return val
	}
	return defaultVal
}

// ----------------------------------------------------------------
// cpuStat represents a single line from /proc/stat for one CPU.
// Each field counts time (in USER_HZ ticks) spent in a mode.
// ----------------------------------------------------------------
type cpuStat struct {
	cpu    string  // CPU identifier (e.g., "cpu0", "cpu1")
	user   float64 // Time in user mode
	nice   float64 // Time in user mode with low priority (nice)
	system float64 // Time in kernel mode
	idle   float64 // Time doing nothing
	iowait float64 // Time waiting for I/O
	irq    float64 // Time servicing hardware interrupts
	softirq float64 // Time servicing software interrupts
	steal  float64 // Time stolen by a hypervisor
}

// readCPUStats parses /proc/stat and returns per-CPU time stats.
// Each line starting with "cpuN" represents one logical CPU.
func readCPUStats(procPath string) ([]cpuStat, error) {
	path := filepath.Join(procPath, "stat")
	file, err := os.Open(path)
	if err != nil {
		return nil, fmt.Errorf("open %s: %w", path, err)
	}
	defer file.Close()

	var stats []cpuStat
	scanner := bufio.NewScanner(file)
	for scanner.Scan() {
		line := scanner.Text()
		// Skip the aggregate "cpu" line; we want per-CPU lines ("cpu0", "cpu1", ...).
		if !strings.HasPrefix(line, "cpu") || line == "cpu" {
			// The aggregate line starts with "cpu " (with a space),
			// but individual CPUs start with "cpu0", "cpu1", etc.
			if strings.HasPrefix(line, "cpu ") {
				continue
			}
			if !strings.HasPrefix(line, "cpu") {
				continue
			}
		}

		fields := strings.Fields(line)
		if len(fields) < 9 {
			continue
		}

		stat := cpuStat{cpu: fields[0]}
		values := make([]float64, 8)
		for i := 0; i < 8 && i+1 < len(fields); i++ {
			v, _ := strconv.ParseFloat(fields[i+1], 64)
			values[i] = v
		}
		stat.user = values[0]
		stat.nice = values[1]
		stat.system = values[2]
		stat.idle = values[3]
		stat.iowait = values[4]
		stat.irq = values[5]
		stat.softirq = values[6]
		stat.steal = values[7]
		stats = append(stats, stat)
	}

	return stats, scanner.Err()
}

// ----------------------------------------------------------------
// memInfo holds selected fields from /proc/meminfo. All values
// are in bytes (converted from the kB values in /proc/meminfo).
// ----------------------------------------------------------------
type memInfo struct {
	totalBytes     uint64
	availableBytes uint64
	freeBytes      uint64
	buffersBytes   uint64
	cachedBytes    uint64
}

// readMemInfo parses /proc/meminfo and returns memory statistics.
func readMemInfo(procPath string) (memInfo, error) {
	path := filepath.Join(procPath, "meminfo")
	file, err := os.Open(path)
	if err != nil {
		return memInfo{}, fmt.Errorf("open %s: %w", path, err)
	}
	defer file.Close()

	var info memInfo
	scanner := bufio.NewScanner(file)
	for scanner.Scan() {
		line := scanner.Text()
		parts := strings.Fields(line)
		if len(parts) < 2 {
			continue
		}
		// /proc/meminfo values are in kB.
		val, _ := strconv.ParseUint(parts[1], 10, 64)
		valBytes := val * 1024

		switch parts[0] {
		case "MemTotal:":
			info.totalBytes = valBytes
		case "MemAvailable:":
			info.availableBytes = valBytes
		case "MemFree:":
			info.freeBytes = valBytes
		case "Buffers:":
			info.buffersBytes = valBytes
		case "Cached:":
			info.cachedBytes = valBytes
		}
	}

	return info, scanner.Err()
}

// ----------------------------------------------------------------
// loadAvg holds the 1, 5, and 15-minute load averages read from
// /proc/loadavg.
// ----------------------------------------------------------------
type loadAvg struct {
	load1  float64
	load5  float64
	load15 float64
}

// readLoadAvg parses /proc/loadavg and returns load averages.
func readLoadAvg(procPath string) (loadAvg, error) {
	path := filepath.Join(procPath, "loadavg")
	data, err := os.ReadFile(path)
	if err != nil {
		return loadAvg{}, fmt.Errorf("read %s: %w", path, err)
	}

	fields := strings.Fields(string(data))
	if len(fields) < 3 {
		return loadAvg{}, fmt.Errorf("unexpected format in %s", path)
	}

	var avg loadAvg
	avg.load1, _ = strconv.ParseFloat(fields[0], 64)
	avg.load5, _ = strconv.ParseFloat(fields[1], 64)
	avg.load15, _ = strconv.ParseFloat(fields[2], 64)

	return avg, nil
}

// ----------------------------------------------------------------
// netStat represents per-interface network statistics read from
// /proc/net/dev.
// ----------------------------------------------------------------
type netStat struct {
	iface       string // Interface name (e.g., "eth0")
	rxBytes     uint64 // Total bytes received
	txBytes     uint64 // Total bytes transmitted
	rxPackets   uint64 // Total packets received
	txPackets   uint64 // Total packets transmitted
}

// readNetStats parses /proc/net/dev and returns per-interface
// network statistics.
func readNetStats(procPath string) ([]netStat, error) {
	path := filepath.Join(procPath, "net", "dev")
	file, err := os.Open(path)
	if err != nil {
		return nil, fmt.Errorf("open %s: %w", path, err)
	}
	defer file.Close()

	var stats []netStat
	scanner := bufio.NewScanner(file)
	lineNum := 0
	for scanner.Scan() {
		lineNum++
		// Skip the two header lines.
		if lineNum <= 2 {
			continue
		}

		line := scanner.Text()
		// Format: "  iface: rxBytes rxPackets ... txBytes txPackets ..."
		colonIdx := strings.Index(line, ":")
		if colonIdx < 0 {
			continue
		}

		iface := strings.TrimSpace(line[:colonIdx])
		fields := strings.Fields(line[colonIdx+1:])
		if len(fields) < 10 {
			continue
		}

		rxBytes, _ := strconv.ParseUint(fields[0], 10, 64)
		rxPackets, _ := strconv.ParseUint(fields[1], 10, 64)
		txBytes, _ := strconv.ParseUint(fields[8], 10, 64)
		txPackets, _ := strconv.ParseUint(fields[9], 10, 64)

		stats = append(stats, netStat{
				iface:     iface,
				rxBytes:   rxBytes,
				txBytes:   txBytes,
				rxPackets: rxPackets,
				txPackets: txPackets,
			})
	}

	return stats, scanner.Err()
}

// ----------------------------------------------------------------
// metricsHandler serves Prometheus-compatible metrics by reading
// the host's /proc and /sys filesystems and formatting the output
// in the Prometheus exposition format.
// ----------------------------------------------------------------
func metricsHandler(cfg config) http.HandlerFunc {
	return func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "text/plain; version=0.0.4; charset=utf-8")

		// --- CPU Metrics ---
		cpuStats, err := readCPUStats(cfg.hostProc)
		if err != nil {
			fmt.Fprintf(w, "# ERROR reading CPU stats: %v\n", err)
		} else {
			fmt.Fprintln(w, "# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.")
			fmt.Fprintln(w, "# TYPE node_cpu_seconds_total counter")
			// Convert ticks to seconds (USER_HZ is typically 100).
			hz := 100.0
			for _, s := range cpuStats {
				fmt.Fprintf(w, "node_cpu_seconds_total{cpu=%q,mode=\"user\",node=%q} %.2f\n", s.cpu, cfg.nodeName, s.user/hz)
				fmt.Fprintf(w, "node_cpu_seconds_total{cpu=%q,mode=\"system\",node=%q} %.2f\n", s.cpu, cfg.nodeName, s.system/hz)
				fmt.Fprintf(w, "node_cpu_seconds_total{cpu=%q,mode=\"idle\",node=%q} %.2f\n", s.cpu, cfg.nodeName, s.idle/hz)
				fmt.Fprintf(w, "node_cpu_seconds_total{cpu=%q,mode=\"iowait\",node=%q} %.2f\n", s.cpu, cfg.nodeName, s.iowait/hz)
				fmt.Fprintf(w, "node_cpu_seconds_total{cpu=%q,mode=\"irq\",node=%q} %.2f\n", s.cpu, cfg.nodeName, s.irq/hz)
				fmt.Fprintf(w, "node_cpu_seconds_total{cpu=%q,mode=\"softirq\",node=%q} %.2f\n", s.cpu, cfg.nodeName, s.softirq/hz)
				fmt.Fprintf(w, "node_cpu_seconds_total{cpu=%q,mode=\"steal\",node=%q} %.2f\n", s.cpu, cfg.nodeName, s.steal/hz)
				fmt.Fprintf(w, "node_cpu_seconds_total{cpu=%q,mode=\"nice\",node=%q} %.2f\n", s.cpu, cfg.nodeName, s.nice/hz)
			}
		}

		// --- Memory Metrics ---
		mem, err := readMemInfo(cfg.hostProc)
		if err != nil {
			fmt.Fprintf(w, "# ERROR reading memory info: %v\n", err)
		} else {
			fmt.Fprintln(w, "# HELP node_memory_total_bytes Total physical memory in bytes.")
			fmt.Fprintln(w, "# TYPE node_memory_total_bytes gauge")
			fmt.Fprintf(w, "node_memory_total_bytes{node=%q} %d\n", cfg.nodeName, mem.totalBytes)

			fmt.Fprintln(w, "# HELP node_memory_available_bytes Available memory in bytes.")
			fmt.Fprintln(w, "# TYPE node_memory_available_bytes gauge")
			fmt.Fprintf(w, "node_memory_available_bytes{node=%q} %d\n", cfg.nodeName, mem.availableBytes)

			fmt.Fprintln(w, "# HELP node_memory_free_bytes Free memory in bytes.")
			fmt.Fprintln(w, "# TYPE node_memory_free_bytes gauge")
			fmt.Fprintf(w, "node_memory_free_bytes{node=%q} %d\n", cfg.nodeName, mem.freeBytes)
		}

		// --- Load Average ---
		load, err := readLoadAvg(cfg.hostProc)
		if err != nil {
			fmt.Fprintf(w, "# ERROR reading load average: %v\n", err)
		} else {
			fmt.Fprintln(w, "# HELP node_load1 1-minute load average.")
			fmt.Fprintln(w, "# TYPE node_load1 gauge")
			fmt.Fprintf(w, "node_load1{node=%q} %.2f\n", cfg.nodeName, load.load1)

			fmt.Fprintln(w, "# HELP node_load5 5-minute load average.")
			fmt.Fprintln(w, "# TYPE node_load5 gauge")
			fmt.Fprintf(w, "node_load5{node=%q} %.2f\n", cfg.nodeName, load.load5)

			fmt.Fprintln(w, "# HELP node_load15 15-minute load average.")
			fmt.Fprintln(w, "# TYPE node_load15 gauge")
			fmt.Fprintf(w, "node_load15{node=%q} %.2f\n", cfg.nodeName, load.load15)
		}

		// --- Network Metrics ---
		netStats, err := readNetStats(cfg.hostProc)
		if err != nil {
			fmt.Fprintf(w, "# ERROR reading network stats: %v\n", err)
		} else {
			fmt.Fprintln(w, "# HELP node_network_receive_bytes_total Total bytes received per interface.")
			fmt.Fprintln(w, "# TYPE node_network_receive_bytes_total counter")
			for _, s := range netStats {
				fmt.Fprintf(w, "node_network_receive_bytes_total{device=%q,node=%q} %d\n", s.iface, cfg.nodeName, s.rxBytes)
			}

			fmt.Fprintln(w, "# HELP node_network_transmit_bytes_total Total bytes transmitted per interface.")
			fmt.Fprintln(w, "# TYPE node_network_transmit_bytes_total counter")
			for _, s := range netStats {
				fmt.Fprintf(w, "node_network_transmit_bytes_total{device=%q,node=%q} %d\n", s.iface, cfg.nodeName, s.txBytes)
			}
		}

		// --- Collector metadata ---
		fmt.Fprintln(w, "# HELP node_collector_scrape_timestamp_seconds Time of the last scrape.")
		fmt.Fprintln(w, "# TYPE node_collector_scrape_timestamp_seconds gauge")
		fmt.Fprintf(w, "node_collector_scrape_timestamp_seconds{node=%q} %d\n", cfg.nodeName, time.Now().Unix())
	}
}

// ----------------------------------------------------------------
// main configures and starts the HTTP server that serves
// Prometheus metrics collected from the host Node's procfs/sysfs.
// ----------------------------------------------------------------
func main() {
	cfg := loadConfig()

	log.Printf("Node Metrics Collector starting on %s", cfg.listenAddr)
	log.Printf("  Node:     %s", cfg.nodeName)
	log.Printf("  hostProc: %s", cfg.hostProc)
	log.Printf("  hostSys:  %s", cfg.hostSys)

	mux := http.NewServeMux()
	mux.HandleFunc("/metrics", metricsHandler(cfg))
	mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
			w.WriteHeader(http.StatusOK)
			fmt.Fprintln(w, "ok")
		})

	server := &http.Server{
		Addr:         cfg.listenAddr,
		Handler:      mux,
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 30 * time.Second,
	}

	log.Fatal(server.ListenAndServe())
}
```

**DaemonSet manifest for the Go metrics collector:**

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-collector
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: node-collector
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: node-collector
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9100"
        prometheus.io/path: "/metrics"
    spec:
      tolerations:
        - operator: Exists  # Run on ALL nodes including control-plane
      containers:
        - name: collector
          image: myregistry/node-collector:latest
          ports:
            - containerPort: 9100
              name: metrics
          env:
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: HOST_PROC
              value: /host/proc
            - name: HOST_SYS
              value: /host/sys
          volumeMounts:
            - name: proc
              mountPath: /host/proc
              readOnly: true
            - name: sys
              mountPath: /host/sys
              readOnly: true
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
            limits:
              cpu: 200m
              memory: 128Mi
          securityContext:
            readOnlyRootFilesystem: true
      volumes:
        - name: proc
          hostPath:
            path: /proc
        - name: sys
          hostPath:
            path: /sys
```

---

## 20.2 Jobs

### 20.2.1 Purpose: Run-to-Completion Workloads

A **Job** creates one or more Pods and ensures that a specified number of them
successfully terminate. Unlike Deployments and DaemonSets (which restart Pods
forever), a Job's Pods run until they **complete** (exit with code 0) or
**fail** (exhaust retry limits).

Jobs are the right controller for:

| Use Case                       | Example                                   |
|--------------------------------|-------------------------------------------|
| **Database migrations**        | Run `flyway migrate` or `alembic upgrade` |
| **Batch processing**           | Process a queue of images/videos          |
| **Data import/export**         | ETL pipelines, backup/restore             |
| **One-time setup tasks**       | Initialize a database, seed data          |
| **Computational tasks**        | Machine learning training, rendering      |
| **Testing/CI tasks**           | Run integration test suites               |

### 20.2.2 Job Spec Deep Dive

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: data-import
spec:
  # -----------------------------------------------------------
  # completions: Total number of successful Pod completions
  # required for the Job to be considered complete.
  # Default: 1 (single-completion Job).
  # -----------------------------------------------------------
  completions: 10

  # -----------------------------------------------------------
  # parallelism: Maximum number of Pods running concurrently.
  # Default: 1 (serial execution).
  # Set to N to run up to N Pods at once.
  # -----------------------------------------------------------
  parallelism: 3

  # -----------------------------------------------------------
  # completionMode: How completions are tracked.
  #   NonIndexed (default): Each successful Pod counts as one
  #     completion. Pods are interchangeable.
  #   Indexed: Each Pod gets a unique index (0 to completions-1)
  #     via the JOB_COMPLETION_INDEX env var. Each index must
  #     complete exactly once.
  # -----------------------------------------------------------
  completionMode: Indexed

  # -----------------------------------------------------------
  # backoffLimit: Maximum number of Pod failures before the Job
  # is marked as Failed. Default: 6.
  # Each failure doubles the backoff (10s, 20s, 40s, ..., 6min).
  # -----------------------------------------------------------
  backoffLimit: 4

  # -----------------------------------------------------------
  # activeDeadlineSeconds: Maximum time (in seconds) the Job
  # can be active before it is terminated. Applies to the Job
  # as a whole, not individual Pods.
  # -----------------------------------------------------------
  activeDeadlineSeconds: 3600  # 1 hour maximum

  # -----------------------------------------------------------
  # ttlSecondsAfterFinished: Automatically delete the Job (and
  # its Pods) this many seconds after completion or failure.
  # Useful for cleanup. Set to 0 for immediate deletion.
  # -----------------------------------------------------------
  ttlSecondsAfterFinished: 86400  # Clean up after 24 hours

  # -----------------------------------------------------------
  # suspend: If true, the Job is suspended — no new Pods are
  # created and running Pods are terminated. Useful for pausing
  # batch workloads.
  # -----------------------------------------------------------
  suspend: false

  # -----------------------------------------------------------
  # podReplacementPolicy: Controls when replacement Pods are
  # created for failed Pods.
  #   TerminatingOrFailed (default): Replacement is created
  #     as soon as Pod starts terminating.
  #   Failed: Replacement waits until Pod fully terminates.
  # -----------------------------------------------------------
  podReplacementPolicy: Failed

  # -----------------------------------------------------------
  # podFailurePolicy: Fine-grained control over which Pod
  # failures count against backoffLimit and which are ignored
  # or cause immediate Job failure.
  # -----------------------------------------------------------
  podFailurePolicy:
    rules:
      # Immediately fail the Job if the container exits with
      # code 42 (indicating a fatal, non-retryable error).
      - action: FailJob
        onExitCodes:
          containerName: worker
          operator: In
          values: [42]
      # Ignore (don't count) failures caused by node disruption
      # (e.g., preemption, node drain).
      - action: Ignore
        onPodConditions:
          - type: DisruptionTarget
            status: "True"
      # Count all other failures against backoffLimit.
      - action: Count
        onExitCodes:
          containerName: worker
          operator: NotIn
          values: [0]

  template:
    spec:
      restartPolicy: Never  # Jobs require Never or OnFailure
      containers:
        - name: worker
          image: myregistry/data-importer:latest
          env:
            # -----------------------------------------------
            # JOB_COMPLETION_INDEX: Automatically set by
            # Kubernetes when completionMode is Indexed.
            # Values range from 0 to (completions - 1).
            # -----------------------------------------------
            - name: BATCH_SIZE
              value: "1000"
          resources:
            requests:
              cpu: 500m
              memory: 256Mi
            limits:
              cpu: "2"
              memory: 1Gi
```

### 20.2.3 How the Job Controller Works

```
LOOP forever:
    succeeded = count Pods in Succeeded phase
    active    = count Pods in Running or Pending phase
    failed    = count Pods in Failed phase

    if succeeded >= completions:
        mark Job as Complete
        return

    if failed > backoffLimit:
        mark Job as Failed
        delete all active Pods
        return

    if activeDeadlineSeconds exceeded:
        mark Job as Failed
        delete all active Pods
        return

    # Create Pods up to parallelism limit
    needed = min(parallelism, completions - succeeded) - active
    for i in 0..needed:
        create Pod
        if completionMode == Indexed:
            assign next unfinished index via JOB_COMPLETION_INDEX
```

### 20.2.4 Completion Modes

**NonIndexed (default):**

- Pods are interchangeable.
- The Job is complete when `completions` Pods have succeeded.
- If a Pod fails, a new Pod is created (up to `backoffLimit` total failures).
- Use for: work queue patterns where any worker can handle any item.

**Indexed:**

- Each Pod gets a unique index (0 to completions-1) in the
  `JOB_COMPLETION_INDEX` environment variable.
- Each index must complete exactly once.
- If the Pod for index 3 fails, a new Pod is created specifically for index 3.
- Use for: partitioned batch processing where each Pod handles a known subset
  of data.

### 20.2.5 Hands-on: Simple One-Shot Job

```yaml
# ------------------------------------------------------------------
# simple-job.yaml
# A minimal Job that runs a single Pod to completion.
# ------------------------------------------------------------------
apiVersion: batch/v1
kind: Job
metadata:
  name: pi-calculator
spec:
  completions: 1
  parallelism: 1
  backoffLimit: 3
  ttlSecondsAfterFinished: 300
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: pi
          image: perl:5.38
          command: ["perl", "-Mbignum=bpi", "-wle", "print bpi(2000)"]
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
```

```bash
# Create the Job
kubectl apply -f simple-job.yaml

# Watch the Pod run
kubectl get pods -l job-name=pi-calculator -w

# Check Job status
kubectl get job pi-calculator
# NAME            COMPLETIONS   DURATION   AGE
# pi-calculator   1/1           8s         30s

# View the result
kubectl logs job/pi-calculator
```

### 20.2.6 Hands-on: Parallel Job (Work Queue Pattern)

```yaml
# ------------------------------------------------------------------
# parallel-job.yaml
# A Job that processes 20 work items with 5 parallel workers.
# Each worker picks a unique item using JOB_COMPLETION_INDEX.
# ------------------------------------------------------------------
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-processor
spec:
  completions: 20
  parallelism: 5
  completionMode: Indexed
  backoffLimit: 10
  ttlSecondsAfterFinished: 600
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: worker
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              # JOB_COMPLETION_INDEX is automatically set by Kubernetes.
              INDEX=${JOB_COMPLETION_INDEX}
              echo "Worker processing item ${INDEX} of 20..."
              # Simulate work (each item takes 5-15 seconds)
              SLEEP_TIME=$((5 + INDEX % 10))
              echo "Processing will take ${SLEEP_TIME} seconds"
              sleep ${SLEEP_TIME}
              echo "Item ${INDEX} processed successfully."
          resources:
            requests:
              cpu: 50m
              memory: 32Mi
```

```bash
# Create the Job
kubectl apply -f parallel-job.yaml

# Watch Pods — at most 5 running at a time
kubectl get pods -l job-name=batch-processor -w

# Check progress
kubectl get job batch-processor
# NAME              COMPLETIONS   DURATION   AGE
# batch-processor   12/20         45s        45s

# View logs for a specific index
kubectl logs -l batch-index=3
```

### 20.2.7 Hands-on: Indexed Job

```yaml
# ------------------------------------------------------------------
# indexed-job.yaml
# Each Pod processes a specific partition of a dataset, identified
# by its JOB_COMPLETION_INDEX.
# ------------------------------------------------------------------
apiVersion: batch/v1
kind: Job
metadata:
  name: data-partitioner
spec:
  completions: 10
  parallelism: 3
  completionMode: Indexed
  backoffLimit: 5
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: processor
          image: python:3.12-slim
          command:
            - python3
            - -c
            - |
              import os, time

              index = int(os.environ["JOB_COMPLETION_INDEX"])
              total = 10  # Must match spec.completions

              # Calculate this partition's data range.
              # For 1000 records split across 10 partitions:
              total_records = 1000
              partition_size = total_records // total
              start = index * partition_size
              end = start + partition_size

              print(f"Partition {index}: processing records {start}-{end-1}")

              # Simulate processing
              for record in range(start, end):
                  if record % 25 == 0:
                      print(f"  Processed record {record}")
                  time.sleep(0.01)

              print(f"Partition {index}: COMPLETE ({partition_size} records)")
          resources:
            requests:
              cpu: 100m
              memory: 64Mi
```

### 20.2.8 Hands-on: Go Batch Processing Program

```go
// ================================================================
// main.go — Batch Image Processor for Kubernetes Jobs
// ================================================================
//
// This program processes a batch of images (simulated) in a
// Kubernetes Job. It demonstrates the Indexed completion mode:
// each Pod receives a unique JOB_COMPLETION_INDEX and processes
// a specific partition of the work.
//
// The program reads a manifest file listing all items to process,
// calculates its partition based on its index and the total
// completions, and processes only its assigned items.
//
// Environment:
//   JOB_COMPLETION_INDEX — Pod's unique index (set by Kubernetes)
//   TOTAL_COMPLETIONS    — Total number of Job completions
//   DATA_DIR             — Directory containing input data
//   OUTPUT_DIR           — Directory for processed output
//
// Build:
//   CGO_ENABLED=0 GOOS=linux go build -o batch-processor main.go
//
// ================================================================

package main

import (
	"encoding/json"
	"fmt"
	"log"
	"math/rand"
	"os"
	"path/filepath"
	"strconv"
	"time"
)

// ----------------------------------------------------------------
// WorkItem represents a single unit of work to be processed.
// In a real application, this might be an image URL, a database
// record ID, or a file path.
// ----------------------------------------------------------------
type WorkItem struct {
	// ID uniquely identifies this work item.
	ID string `json:"id"`

	// InputPath is the path to the input data (e.g., an image).
	InputPath string `json:"input_path"`

	// Priority determines processing order within a partition.
	Priority int `json:"priority"`
}

// ----------------------------------------------------------------
// ProcessingResult captures the outcome of processing a single
// work item, including timing and any errors encountered.
// ----------------------------------------------------------------
type ProcessingResult struct {
	// ItemID is the ID of the processed work item.
	ItemID string `json:"item_id"`

	// Success indicates whether processing completed without error.
	Success bool `json:"success"`

	// DurationMs is the processing time in milliseconds.
	DurationMs int64 `json:"duration_ms"`

	// Error contains any error message (empty on success).
	Error string `json:"error,omitempty"`

	// OutputPath is where the processed result was written.
	OutputPath string `json:"output_path,omitempty"`
}

// ----------------------------------------------------------------
// BatchReport summarizes the results of processing a partition.
// It is written to the output directory as a JSON file.
// ----------------------------------------------------------------
type BatchReport struct {
	// WorkerIndex is this Pod's JOB_COMPLETION_INDEX.
	WorkerIndex int `json:"worker_index"`

	// TotalWorkers is the total number of Job completions.
	TotalWorkers int `json:"total_workers"`

	// PartitionStart is the first item index in this partition.
	PartitionStart int `json:"partition_start"`

	// PartitionEnd is one past the last item index.
	PartitionEnd int `json:"partition_end"`

	// TotalItems is the number of items assigned to this partition.
	TotalItems int `json:"total_items"`

	// SuccessCount is the number of items processed successfully.
	SuccessCount int `json:"success_count"`

	// FailureCount is the number of items that failed processing.
	FailureCount int `json:"failure_count"`

	// TotalDurationMs is the total processing time in milliseconds.
	TotalDurationMs int64 `json:"total_duration_ms"`

	// Results contains per-item processing results.
	Results []ProcessingResult `json:"results"`
}

// ----------------------------------------------------------------
// processItem simulates processing a single work item. In a real
// application, this would perform actual image resizing, ML
// inference, data transformation, etc.
// ----------------------------------------------------------------
func processItem(item WorkItem, outputDir string) ProcessingResult {
	start := time.Now()

	// Simulate variable processing time (50ms to 500ms).
	processingTime := time.Duration(50+rand.Intn(450)) * time.Millisecond
	time.Sleep(processingTime)

	// Simulate occasional failures (5% failure rate).
	if rand.Float64() < 0.05 {
		return ProcessingResult{
			ItemID:     item.ID,
			Success:    false,
			DurationMs: time.Since(start).Milliseconds(),
			Error:      "simulated transient processing error",
		}
	}

	// Write output (simulated).
	outputPath := filepath.Join(outputDir, fmt.Sprintf("processed_%s.json", item.ID))

	return ProcessingResult{
		ItemID:     item.ID,
		Success:    true,
		DurationMs: time.Since(start).Milliseconds(),
		OutputPath: outputPath,
	}
}

// ----------------------------------------------------------------
// generateWorkItems creates a list of simulated work items.
// In production, these would be read from a database, message
// queue, or shared filesystem.
// ----------------------------------------------------------------
func generateWorkItems(count int) []WorkItem {
	items := make([]WorkItem, count)
	for i := 0; i < count; i++ {
		items[i] = WorkItem{
			ID:        fmt.Sprintf("item-%04d", i),
			InputPath: fmt.Sprintf("/data/input/image_%04d.jpg", i),
			Priority:  rand.Intn(10),
		}
	}
	return items
}

// ----------------------------------------------------------------
// calculatePartition determines the range of items this worker
// should process, based on its index and the total number of
// workers. Items are divided as evenly as possible, with
// earlier workers taking any remainder.
// ----------------------------------------------------------------
func calculatePartition(workerIndex, totalWorkers, totalItems int) (start, end int) {
	partitionSize := totalItems / totalWorkers
	remainder := totalItems % totalWorkers

	if workerIndex < remainder {
		// This worker gets one extra item.
		start = workerIndex * (partitionSize + 1)
		end = start + partitionSize + 1
	} else {
		start = remainder*(partitionSize+1) + (workerIndex-remainder)*partitionSize
		end = start + partitionSize
	}

	return start, end
}

// ----------------------------------------------------------------
// main orchestrates the batch processing workflow:
// 1. Read environment variables to determine partition assignment.
// 2. Generate (or load) the work item list.
// 3. Calculate this worker's partition.
// 4. Process each item in the partition.
// 5. Write a summary report.
// 6. Exit with code 0 on success, 1 on failure.
// ----------------------------------------------------------------
func main() {
	log.SetFlags(log.Ldate | log.Ltime | log.Lshortfile)
	rand.New(rand.NewSource(time.Now().UnixNano()))

	// Read environment variables.
	workerIndex, err := strconv.Atoi(os.Getenv("JOB_COMPLETION_INDEX"))
	if err != nil {
		log.Fatalf("Invalid JOB_COMPLETION_INDEX: %v", err)
	}

	totalWorkers, err := strconv.Atoi(os.Getenv("TOTAL_COMPLETIONS"))
	if err != nil {
		totalWorkers = 10 // Default for local testing
	}

	outputDir := os.Getenv("OUTPUT_DIR")
	if outputDir == "" {
		outputDir = "/data/output"
	}

	// Create output directory.
	if err := os.MkdirAll(outputDir, 0755); err != nil {
		log.Fatalf("Failed to create output directory: %v", err)
	}

	log.Printf("Worker %d of %d starting", workerIndex, totalWorkers)

	// Generate work items (in production, load from shared storage).
	totalItemCount := 1000
	allItems := generateWorkItems(totalItemCount)

	// Calculate partition.
	partStart, partEnd := calculatePartition(workerIndex, totalWorkers, totalItemCount)
	myItems := allItems[partStart:partEnd]

	log.Printf("Worker %d: processing items %d-%d (%d items)",
		workerIndex, partStart, partEnd-1, len(myItems))

	// Process each item.
	batchStart := time.Now()
	report := BatchReport{
		WorkerIndex:    workerIndex,
		TotalWorkers:   totalWorkers,
		PartitionStart: partStart,
		PartitionEnd:   partEnd,
		TotalItems:     len(myItems),
		Results:        make([]ProcessingResult, 0, len(myItems)),
	}

	for i, item := range myItems {
		result := processItem(item, outputDir)
		report.Results = append(report.Results, result)

		if result.Success {
			report.SuccessCount++
		} else {
			report.FailureCount++
			log.Printf("  WARNING: Item %s failed: %s", item.ID, result.Error)
		}

		// Log progress every 10 items.
		if (i+1)%10 == 0 || i == len(myItems)-1 {
			log.Printf("  Progress: %d/%d items processed", i+1, len(myItems))
		}
	}

	report.TotalDurationMs = time.Since(batchStart).Milliseconds()

	// Write report.
	reportPath := filepath.Join(outputDir, fmt.Sprintf("report_worker_%d.json", workerIndex))
	reportFile, err := os.Create(reportPath)
	if err != nil {
		log.Fatalf("Failed to create report file: %v", err)
	}
	defer reportFile.Close()

	encoder := json.NewEncoder(reportFile)
	encoder.SetIndent("", "  ")
	if err := encoder.Encode(report); err != nil {
		log.Fatalf("Failed to write report: %v", err)
	}

	log.Printf("Worker %d COMPLETE: %d succeeded, %d failed, %dms total",
		workerIndex, report.SuccessCount, report.FailureCount, report.TotalDurationMs)

	// Exit with failure if too many items failed (>20%).
	failureRate := float64(report.FailureCount) / float64(report.TotalItems)
	if failureRate > 0.20 {
		log.Fatalf("Failure rate %.1f%% exceeds 20%% threshold", failureRate*100)
	}
}
```

**Kubernetes manifests for the batch processor:**

```yaml
# ------------------------------------------------------------------
# batch-job.yaml
# Indexed Job running the Go batch processor.
# 10 workers process 1000 items in parallel.
# ------------------------------------------------------------------
apiVersion: batch/v1
kind: Job
metadata:
  name: image-batch
spec:
  completions: 10
  parallelism: 5
  completionMode: Indexed
  backoffLimit: 20
  activeDeadlineSeconds: 3600
  ttlSecondsAfterFinished: 86400
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: processor
          image: myregistry/batch-processor:latest
          env:
            - name: TOTAL_COMPLETIONS
              value: "10"
            - name: OUTPUT_DIR
              value: /data/output
          volumeMounts:
            - name: data
              mountPath: /data
          resources:
            requests:
              cpu: 500m
              memory: 256Mi
            limits:
              cpu: "2"
              memory: 1Gi
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: batch-data
```

---

## 20.3 CronJobs

### 20.3.1 Purpose: Scheduled Jobs

A **CronJob** creates Jobs on a recurring schedule, using the familiar cron
syntax. It is the Kubernetes equivalent of the Unix `crontab`.

Common use cases:

| Use Case                    | Schedule                    |
|-----------------------------|-----------------------------|
| Database backup             | `0 2 * * *` (daily 2 AM)   |
| Log rotation/cleanup        | `0 0 * * 0` (weekly Sunday) |
| Report generation           | `0 9 * * 1-5` (weekdays 9 AM) |
| Certificate renewal         | `0 0 1 * *` (monthly)      |
| Health check / smoke test   | `*/5 * * * *` (every 5 min) |
| Data synchronization        | `0 */6 * * *` (every 6 hours) |

### 20.3.2 CronJob Spec Deep Dive

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: db-backup
spec:
  # -----------------------------------------------------------
  # schedule: Cron expression defining when Jobs are created.
  # Format: minute hour day-of-month month day-of-week
  #
  # Examples:
  #   "0 2 * * *"     — Every day at 2:00 AM
  #   "*/15 * * * *"  — Every 15 minutes
  #   "0 9 * * 1-5"   — Weekdays at 9:00 AM
  #   "0 0 1 * *"     — First day of each month at midnight
  #   "30 4 * * 0"    — Every Sunday at 4:30 AM
  # -----------------------------------------------------------
  schedule: "0 2 * * *"

  # -----------------------------------------------------------
  # timeZone: IANA time zone for interpreting the schedule.
  # If not set, the kube-controller-manager's time zone is used.
  # Examples: "America/New_York", "Europe/London", "UTC"
  # -----------------------------------------------------------
  timeZone: "America/New_York"

  # -----------------------------------------------------------
  # concurrencyPolicy: What to do if the previous Job is still
  # running when a new scheduled run is due.
  #
  #   Allow (default): Multiple Jobs can run concurrently.
  #   Forbid: Skip the new run if the previous is still active.
  #   Replace: Cancel the running Job and start a new one.
  # -----------------------------------------------------------
  concurrencyPolicy: Forbid

  # -----------------------------------------------------------
  # startingDeadlineSeconds: Maximum seconds after the scheduled
  # time that a Job can still be started. If the CronJob
  # controller misses the window (e.g., controller was down),
  # the run is skipped.
  # If not set, there is no deadline (the Job may start late).
  # -----------------------------------------------------------
  startingDeadlineSeconds: 300

  # -----------------------------------------------------------
  # successfulJobsHistoryLimit: Number of successful Jobs to
  # retain. Default: 3. Set to 0 to keep none.
  # -----------------------------------------------------------
  successfulJobsHistoryLimit: 3

  # -----------------------------------------------------------
  # failedJobsHistoryLimit: Number of failed Jobs to retain.
  # Default: 1. Keeping some failed Jobs is useful for debugging.
  # -----------------------------------------------------------
  failedJobsHistoryLimit: 5

  # -----------------------------------------------------------
  # suspend: If true, no new Jobs will be created. Existing Jobs
  # continue to run. Useful for temporarily pausing scheduled work.
  # -----------------------------------------------------------
  suspend: false

  # -----------------------------------------------------------
  # jobTemplate: The Job spec to create on each schedule trigger.
  # All Job spec fields are available here.
  # -----------------------------------------------------------
  jobTemplate:
    spec:
      backoffLimit: 3
      activeDeadlineSeconds: 7200  # 2 hours max per Job
      ttlSecondsAfterFinished: 86400
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: backup
              image: postgres:16
              command:
                - sh
                - -c
                - |
                  echo "Starting database backup at $(date)"
                  TIMESTAMP=$(date +%Y%m%d_%H%M%S)
                  BACKUP_FILE="/backups/db_backup_${TIMESTAMP}.sql.gz"

                  pg_dump \
                    -h postgres-primary.default.svc.cluster.local \
                    -U backup_user \
                    -d myapp \
                    --format=custom \
                    --compress=9 \
                    | gzip > "${BACKUP_FILE}"

                  echo "Backup written to ${BACKUP_FILE}"
                  echo "Size: $(du -sh ${BACKUP_FILE} | awk '{print $1}')"

                  # Clean up backups older than 30 days
                  find /backups -name "db_backup_*.sql.gz" -mtime +30 -delete
                  echo "Cleanup complete. Remaining backups:"
                  ls -lh /backups/

                  echo "Backup finished at $(date)"
              env:
                - name: PGPASSWORD
                  valueFrom:
                    secretKeyRef:
                      name: postgres-backup-secret
                      key: password
              volumeMounts:
                - name: backups
                  mountPath: /backups
              resources:
                requests:
                  cpu: 200m
                  memory: 256Mi
                limits:
                  cpu: "1"
                  memory: 1Gi
          volumes:
            - name: backups
              persistentVolumeClaim:
                claimName: backup-storage
```

### 20.3.3 Cron Syntax Deep Dive

The cron schedule uses five fields:

```
┌───────── minute (0–59)
│ ┌─────── hour (0–23)
│ │ ┌───── day of month (1–31)
│ │ │ ┌─── month (1–12 or JAN–DEC)
│ │ │ │ ┌─ day of week (0–7, where 0 and 7 are Sunday, or SUN–SAT)
│ │ │ │ │
* * * * *
```

| Symbol  | Meaning                           | Example                        |
|---------|-----------------------------------|--------------------------------|
| `*`     | Every value                       | `* * * * *` = every minute     |
| `,`     | List                              | `1,15 * * * *` = minute 1 & 15 |
| `-`     | Range                             | `1-5 * * * *` = minutes 1–5   |
| `/`     | Step                              | `*/10 * * * *` = every 10 min  |

Common patterns:

```
"0 * * * *"       — Every hour, on the hour
"0 */2 * * *"     — Every 2 hours
"0 9-17 * * 1-5"  — Hourly during business hours, weekdays
"0 0 * * *"       — Midnight daily
"0 0 * * 0"       — Midnight Sunday
"0 0 1 * *"       — Midnight on the 1st of each month
"0 0 1 1 *"       — Midnight on January 1st
"*/5 * * * *"     — Every 5 minutes
"30 4 * * 2,4"    — 4:30 AM on Tuesdays and Thursdays
```

### 20.3.4 Concurrency Policies

The `concurrencyPolicy` field controls behavior when a new scheduled run
triggers while a previous Job is still active:

**Allow** (default):
- Multiple Jobs can run concurrently.
- Risk: Resource contention, data races if Jobs access shared resources.
- Use for: Independent tasks with no shared state.

**Forbid**:
- The new run is skipped if the previous Job is still active.
- The skipped run is not rescheduled.
- Use for: Tasks that must not overlap (e.g., database backups).

**Replace**:
- The currently running Job is cancelled and replaced with a new one.
- Use for: Tasks where only the latest run matters (e.g., cache warm-up).

### 20.3.5 Time Zones

The `timeZone` field (stable since Kubernetes 1.27) specifies the IANA time
zone for interpreting the schedule:

```yaml
spec:
  schedule: "0 9 * * 1-5"
  timeZone: "America/New_York"  # 9 AM Eastern, every weekday
```

Without `timeZone`, the schedule is interpreted in the kube-controller-manager's
local time zone (which is often UTC). This matters for schedules like "daily
at 2 AM" — 2 AM UTC is very different from 2 AM Pacific.

### 20.3.6 Hands-on: CronJob for Database Backup

The full YAML was shown in section 20.3.2. Here is how to deploy and monitor it:

```bash
# Deploy the CronJob
kubectl apply -f db-backup-cronjob.yaml

# Check the CronJob status
kubectl get cronjob db-backup
# NAME        SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# db-backup   0 2 * * *   False     0        <none>          5s

# Manually trigger a Job for testing
kubectl create job --from=cronjob/db-backup db-backup-manual-test

# Watch the Job run
kubectl get jobs -w

# Check the Job's Pod logs
kubectl logs job/db-backup-manual-test

# View Job history
kubectl get jobs -l job-name=db-backup
```

### 20.3.7 Hands-on: CronJob for Cleanup Tasks

```yaml
# ------------------------------------------------------------------
# cleanup-cronjob.yaml
# Runs daily to clean up old resources in the cluster.
# ------------------------------------------------------------------
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cluster-cleanup
spec:
  schedule: "0 3 * * *"  # Daily at 3 AM
  timeZone: "UTC"
  concurrencyPolicy: Forbid
  successfulJobsHistoryLimit: 3
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      ttlSecondsAfterFinished: 86400
      template:
        spec:
          restartPolicy: OnFailure
          serviceAccountName: cluster-cleanup
          containers:
            - name: cleanup
              image: bitnami/kubectl:1.29
              command:
                - sh
                - -c
                - |
                  echo "=== Cluster Cleanup Starting at $(date) ==="

                  # Delete completed Jobs older than 7 days
                  echo "--- Cleaning up old completed Jobs ---"
                  kubectl get jobs --all-namespaces \
                    -o json | \
                    jq -r '.items[] |
                      select(.status.completionTime != null) |
                      select((now - (.status.completionTime | fromdateiso8601)) > 604800) |
                      "\(.metadata.namespace)/\(.metadata.name)"' | \
                    while read job; do
                      NS=$(echo $job | cut -d/ -f1)
                      NAME=$(echo $job | cut -d/ -f2)
                      echo "Deleting completed Job: $NS/$NAME"
                      kubectl delete job "$NAME" -n "$NS"
                    done

                  # Delete failed Pods older than 3 days
                  echo "--- Cleaning up old failed Pods ---"
                  kubectl get pods --all-namespaces --field-selector=status.phase=Failed \
                    -o json | \
                    jq -r '.items[] |
                      select((now - (.metadata.creationTimestamp | fromdateiso8601)) > 259200) |
                      "\(.metadata.namespace)/\(.metadata.name)"' | \
                    while read pod; do
                      NS=$(echo $pod | cut -d/ -f1)
                      NAME=$(echo $pod | cut -d/ -f2)
                      echo "Deleting failed Pod: $NS/$NAME"
                      kubectl delete pod "$NAME" -n "$NS"
                    done

                  # Delete evicted Pods
                  echo "--- Cleaning up evicted Pods ---"
                  kubectl get pods --all-namespaces \
                    --field-selector=status.phase=Failed \
                    -o json | \
                    jq -r '.items[] |
                      select(.status.reason == "Evicted") |
                      "\(.metadata.namespace)/\(.metadata.name)"' | \
                    while read pod; do
                      NS=$(echo $pod | cut -d/ -f1)
                      NAME=$(echo $pod | cut -d/ -f2)
                      echo "Deleting evicted Pod: $NS/$NAME"
                      kubectl delete pod "$NAME" -n "$NS"
                    done

                  echo "=== Cluster Cleanup Complete at $(date) ==="
              resources:
                requests:
                  cpu: 50m
                  memory: 64Mi
                limits:
                  cpu: 200m
                  memory: 128Mi
```

### 20.3.8 Hands-on: Go Report Generator (Deployed as CronJob)

```go
// ================================================================
// main.go — Daily Report Generator for Kubernetes CronJobs
// ================================================================
//
// This program generates a daily summary report by querying
// various data sources and producing a formatted report. It is
// designed to run as a CronJob Pod, executing once per day and
// writing the report to a shared PVC or uploading it to cloud
// storage.
//
// The program demonstrates:
// - How to structure a CronJob-compatible Go application
// - Graceful handling of CronJob lifecycle (short-lived process)
// - Writing output to persistent storage for later retrieval
// - Proper exit codes for Job success/failure tracking
//
// Environment:
//   REPORT_DATE    — Date to generate report for (YYYY-MM-DD).
//                    Defaults to yesterday.
//   OUTPUT_DIR     — Directory to write reports to.
//   DATA_SOURCE    — URL of the data source API.
//   REPORT_FORMAT  — Output format: "json", "csv", or "text".
//
// Build:
//   CGO_ENABLED=0 GOOS=linux go build -o report-generator main.go
//
// ================================================================

package main

import (
	"encoding/csv"
	"encoding/json"
	"fmt"
	"log"
	"math/rand"
	"os"
	"path/filepath"
	"time"
)

// ----------------------------------------------------------------
// MetricSummary contains aggregated metrics for a single service
// over the reporting period. Each field represents a key
// operational metric that stakeholders care about.
// ----------------------------------------------------------------
type MetricSummary struct {
	// ServiceName identifies the microservice or component.
	ServiceName string `json:"service_name"`

	// TotalRequests is the total number of requests served.
	TotalRequests int64 `json:"total_requests"`

	// ErrorCount is the number of requests that resulted in errors.
	ErrorCount int64 `json:"error_count"`

	// ErrorRate is the percentage of requests that failed.
	ErrorRate float64 `json:"error_rate"`

	// AvgLatencyMs is the average request latency in milliseconds.
	AvgLatencyMs float64 `json:"avg_latency_ms"`

	// P99LatencyMs is the 99th percentile latency in milliseconds.
	P99LatencyMs float64 `json:"p99_latency_ms"`

	// PodCount is the average number of running Pods.
	PodCount int `json:"pod_count"`

	// CPUUsagePercent is the average CPU utilization percentage.
	CPUUsagePercent float64 `json:"cpu_usage_percent"`

	// MemoryUsageMB is the average memory usage in megabytes.
	MemoryUsageMB float64 `json:"memory_usage_mb"`
}

// ----------------------------------------------------------------
// DailyReport is the top-level report structure containing
// metadata and per-service metric summaries.
// ----------------------------------------------------------------
type DailyReport struct {
	// ReportDate is the date this report covers.
	ReportDate string `json:"report_date"`

	// GeneratedAt is the timestamp when the report was generated.
	GeneratedAt string `json:"generated_at"`

	// TotalServices is the number of services included.
	TotalServices int `json:"total_services"`

	// OverallHealthScore is a 0-100 score summarizing cluster health.
	OverallHealthScore float64 `json:"overall_health_score"`

	// Services contains per-service metric summaries.
	Services []MetricSummary `json:"services"`

	// Alerts contains any notable conditions detected.
	Alerts []string `json:"alerts,omitempty"`
}

// ----------------------------------------------------------------
// collectMetrics gathers metrics for all services. In production,
// this would query Prometheus, Datadog, or another metrics backend.
// Here we simulate the data for demonstration purposes.
// ----------------------------------------------------------------
func collectMetrics(reportDate string) []MetricSummary {
	services := []string{
		"api-gateway", "user-service", "order-service",
		"payment-service", "notification-service", "search-service",
		"inventory-service", "analytics-service",
	}

	metrics := make([]MetricSummary, len(services))
	for i, svc := range services {
		totalReqs := int64(100000 + rand.Intn(900000))
		errorCount := int64(float64(totalReqs) * (0.001 + rand.Float64()*0.01))

		metrics[i] = MetricSummary{
			ServiceName:     svc,
			TotalRequests:   totalReqs,
			ErrorCount:      errorCount,
			ErrorRate:       float64(errorCount) / float64(totalReqs) * 100,
			AvgLatencyMs:    10 + rand.Float64()*90,
			P99LatencyMs:    100 + rand.Float64()*400,
			PodCount:        2 + rand.Intn(8),
			CPUUsagePercent: 10 + rand.Float64()*60,
			MemoryUsageMB:   128 + rand.Float64()*896,
		}
	}

	return metrics
}

// ----------------------------------------------------------------
// generateAlerts analyzes metrics and produces human-readable
// alerts for any concerning conditions.
// ----------------------------------------------------------------
func generateAlerts(metrics []MetricSummary) []string {
	var alerts []string

	for _, m := range metrics {
		if m.ErrorRate > 1.0 {
			alerts = append(alerts,
				fmt.Sprintf("HIGH ERROR RATE: %s has %.2f%% error rate (%d errors out of %d requests)",
					m.ServiceName, m.ErrorRate, m.ErrorCount, m.TotalRequests))
		}
		if m.P99LatencyMs > 400 {
			alerts = append(alerts,
				fmt.Sprintf("HIGH LATENCY: %s p99 latency is %.0fms (threshold: 400ms)",
					m.ServiceName, m.P99LatencyMs))
		}
		if m.CPUUsagePercent > 70 {
			alerts = append(alerts,
				fmt.Sprintf("HIGH CPU: %s at %.1f%% CPU utilization",
					m.ServiceName, m.CPUUsagePercent))
		}
		if m.MemoryUsageMB > 800 {
			alerts = append(alerts,
				fmt.Sprintf("HIGH MEMORY: %s using %.0fMB memory",
					m.ServiceName, m.MemoryUsageMB))
		}
	}

	return alerts
}

// ----------------------------------------------------------------
// calculateHealthScore computes an overall 0-100 health score
// based on error rates, latencies, and resource utilization.
// ----------------------------------------------------------------
func calculateHealthScore(metrics []MetricSummary) float64 {
	if len(metrics) == 0 {
		return 0
	}

	totalScore := 0.0
	for _, m := range metrics {
		svcScore := 100.0

		// Penalize high error rates.
		if m.ErrorRate > 0.1 {
			svcScore -= m.ErrorRate * 10
		}

		// Penalize high latency.
		if m.P99LatencyMs > 200 {
			svcScore -= (m.P99LatencyMs - 200) / 10
		}

		// Penalize high resource usage.
		if m.CPUUsagePercent > 60 {
			svcScore -= (m.CPUUsagePercent - 60) / 5
		}

		if svcScore < 0 {
			svcScore = 0
		}
		totalScore += svcScore
	}

	return totalScore / float64(len(metrics))
}

// ----------------------------------------------------------------
// writeJSONReport writes the report as a JSON file.
// ----------------------------------------------------------------
func writeJSONReport(report DailyReport, outputDir string) (string, error) {
	filename := fmt.Sprintf("daily_report_%s.json", report.ReportDate)
	path := filepath.Join(outputDir, filename)

	file, err := os.Create(path)
	if err != nil {
		return "", fmt.Errorf("create %s: %w", path, err)
	}
	defer file.Close()

	encoder := json.NewEncoder(file)
	encoder.SetIndent("", "  ")
	if err := encoder.Encode(report); err != nil {
		return "", fmt.Errorf("encode JSON: %w", err)
	}

	return path, nil
}

// ----------------------------------------------------------------
// writeCSVReport writes the per-service metrics as a CSV file.
// ----------------------------------------------------------------
func writeCSVReport(report DailyReport, outputDir string) (string, error) {
	filename := fmt.Sprintf("daily_report_%s.csv", report.ReportDate)
	path := filepath.Join(outputDir, filename)

	file, err := os.Create(path)
	if err != nil {
		return "", fmt.Errorf("create %s: %w", path, err)
	}
	defer file.Close()

	writer := csv.NewWriter(file)
	defer writer.Flush()

	// Write header.
	header := []string{
		"Service", "Total Requests", "Errors", "Error Rate %",
		"Avg Latency (ms)", "P99 Latency (ms)", "Pods",
		"CPU %", "Memory (MB)",
	}
	if err := writer.Write(header); err != nil {
		return "", fmt.Errorf("write header: %w", err)
	}

	// Write rows.
	for _, m := range report.Services {
		row := []string{
			m.ServiceName,
			fmt.Sprintf("%d", m.TotalRequests),
			fmt.Sprintf("%d", m.ErrorCount),
			fmt.Sprintf("%.2f", m.ErrorRate),
			fmt.Sprintf("%.1f", m.AvgLatencyMs),
			fmt.Sprintf("%.1f", m.P99LatencyMs),
			fmt.Sprintf("%d", m.PodCount),
			fmt.Sprintf("%.1f", m.CPUUsagePercent),
			fmt.Sprintf("%.0f", m.MemoryUsageMB),
		}
		if err := writer.Write(row); err != nil {
			return "", fmt.Errorf("write row: %w", err)
		}
	}

	return path, nil
}

// ----------------------------------------------------------------
// writeTextReport writes a human-readable text report.
// ----------------------------------------------------------------
func writeTextReport(report DailyReport, outputDir string) (string, error) {
	filename := fmt.Sprintf("daily_report_%s.txt", report.ReportDate)
	path := filepath.Join(outputDir, filename)

	file, err := os.Create(path)
	if err != nil {
		return "", fmt.Errorf("create %s: %w", path, err)
	}
	defer file.Close()

	fmt.Fprintf(file, "╔══════════════════════════════════════════════╗\n")
	fmt.Fprintf(file, "║         DAILY OPERATIONS REPORT              ║\n")
	fmt.Fprintf(file, "║         Date: %-30s ║\n", report.ReportDate)
	fmt.Fprintf(file, "╠══════════════════════════════════════════════╣\n")
	fmt.Fprintf(file, "║  Generated: %-33s║\n", report.GeneratedAt)
	fmt.Fprintf(file, "║  Services:  %-33d║\n", report.TotalServices)
	fmt.Fprintf(file, "║  Health:    %-33.1f║\n", report.OverallHealthScore)
	fmt.Fprintf(file, "╚══════════════════════════════════════════════╝\n\n")

	if len(report.Alerts) > 0 {
		fmt.Fprintf(file, "⚠️  ALERTS (%d):\n", len(report.Alerts))
		for _, alert := range report.Alerts {
			fmt.Fprintf(file, "  • %s\n", alert)
		}
		fmt.Fprintln(file)
	}

	fmt.Fprintf(file, "%-22s %10s %8s %8s %8s %8s\n",
		"SERVICE", "REQUESTS", "ERRORS", "ERR%", "AVG(ms)", "P99(ms)")
	fmt.Fprintf(file, "%s\n", "────────────────────── ────────── ──────── ──────── ──────── ────────")

	for _, m := range report.Services {
		fmt.Fprintf(file, "%-22s %10d %8d %7.2f%% %8.1f %8.1f\n",
			m.ServiceName, m.TotalRequests, m.ErrorCount,
			m.ErrorRate, m.AvgLatencyMs, m.P99LatencyMs)
	}

	return path, nil
}

// ----------------------------------------------------------------
// main generates the daily report, writes it to the configured
// output directory, and exits. In a CronJob context, Kubernetes
// tracks whether the Job Pod exits with code 0 (success) or
// non-zero (failure).
// ----------------------------------------------------------------
func main() {
	log.SetFlags(log.Ldate | log.Ltime | log.Lshortfile)
	rand.New(rand.NewSource(time.Now().UnixNano()))

	// Determine report date.
	reportDate := os.Getenv("REPORT_DATE")
	if reportDate == "" {
		// Default to yesterday.
		reportDate = time.Now().AddDate(0, 0, -1).Format("2006-01-02")
	}

	outputDir := os.Getenv("OUTPUT_DIR")
	if outputDir == "" {
		outputDir = "/reports"
	}

	reportFormat := os.Getenv("REPORT_FORMAT")
	if reportFormat == "" {
		reportFormat = "json"
	}

	log.Printf("Generating report for %s (format: %s)", reportDate, reportFormat)

	// Create output directory.
	if err := os.MkdirAll(outputDir, 0755); err != nil {
		log.Fatalf("Failed to create output directory: %v", err)
	}

	// Collect metrics.
	metrics := collectMetrics(reportDate)
	alerts := generateAlerts(metrics)
	healthScore := calculateHealthScore(metrics)

	report := DailyReport{
		ReportDate:         reportDate,
		GeneratedAt:        time.Now().Format(time.RFC3339),
		TotalServices:      len(metrics),
		OverallHealthScore: healthScore,
		Services:           metrics,
		Alerts:             alerts,
	}

	// Write report in requested format(s).
	var outputPath string
	var err error

	switch reportFormat {
	case "json":
		outputPath, err = writeJSONReport(report, outputDir)
	case "csv":
		outputPath, err = writeCSVReport(report, outputDir)
	case "text":
		outputPath, err = writeTextReport(report, outputDir)
	case "all":
		if p, e := writeJSONReport(report, outputDir); e != nil {
			err = e
		} else {
			log.Printf("JSON report: %s", p)
		}
		if p, e := writeCSVReport(report, outputDir); e != nil {
			err = e
		} else {
			log.Printf("CSV report: %s", p)
		}
		if p, e := writeTextReport(report, outputDir); e != nil {
			err = e
		} else {
			log.Printf("Text report: %s", p)
			outputPath = p
		}
	default:
		log.Fatalf("Unknown report format: %s", reportFormat)
	}

	if err != nil {
		log.Fatalf("Failed to write report: %v", err)
	}

	log.Printf("Report generated: %s", outputPath)
	log.Printf("Health score: %.1f/100", healthScore)
	if len(alerts) > 0 {
		log.Printf("⚠️  %d alerts generated", len(alerts))
	}

	log.Println("Report generation complete.")
}
```

**CronJob manifest for the report generator:**

```yaml
# ------------------------------------------------------------------
# report-cronjob.yaml
# Generates a daily operations report every morning at 6 AM.
# ------------------------------------------------------------------
apiVersion: batch/v1
kind: CronJob
metadata:
  name: daily-report
spec:
  schedule: "0 6 * * *"
  timeZone: "America/New_York"
  concurrencyPolicy: Forbid
  startingDeadlineSeconds: 600
  successfulJobsHistoryLimit: 7
  failedJobsHistoryLimit: 3
  jobTemplate:
    spec:
      backoffLimit: 2
      activeDeadlineSeconds: 1800
      ttlSecondsAfterFinished: 604800
      template:
        spec:
          restartPolicy: OnFailure
          containers:
            - name: reporter
              image: myregistry/report-generator:latest
              env:
                - name: OUTPUT_DIR
                  value: /reports
                - name: REPORT_FORMAT
                  value: all
              volumeMounts:
                - name: reports
                  mountPath: /reports
              resources:
                requests:
                  cpu: 200m
                  memory: 128Mi
                limits:
                  cpu: "1"
                  memory: 512Mi
          volumes:
            - name: reports
              persistentVolumeClaim:
                claimName: report-storage
```

---

## 20.4 Advanced Job Patterns

### 20.4.1 Work Queue with Shared Storage

In this pattern, multiple Job Pods consume work items from a shared queue
(e.g., a Redis list, an SQS queue, or a file in shared storage). Each Pod
runs indefinitely until the queue is empty, then exits successfully.

```yaml
# ------------------------------------------------------------------
# work-queue-job.yaml
# Multiple workers consume items from a Redis queue until it's empty.
# completions is unset — the Job is complete when ANY Pod exits
# successfully (indicating the queue is empty).
# ------------------------------------------------------------------
apiVersion: batch/v1
kind: Job
metadata:
  name: work-queue
spec:
  # No completions field — this is a "work queue" Job.
  # The Job finishes when parallelism workers all exit 0.
  parallelism: 5
  backoffLimit: 10
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: worker
          image: myregistry/queue-worker:latest
          env:
            - name: REDIS_URL
              value: "redis://redis.default.svc.cluster.local:6379"
            - name: QUEUE_NAME
              value: "work-items"
```

### 20.4.2 Fan-Out / Fan-In Processing

This pattern uses two Jobs in sequence:

1. **Fan-out Job:** Splits input data into N partitions and writes them to
   shared storage.
2. **Fan-in Job:** Reads all N partition results and produces a final summary.

```yaml
# ------------------------------------------------------------------
# Step 1: Fan-out — split work into partitions
# ------------------------------------------------------------------
apiVersion: batch/v1
kind: Job
metadata:
  name: fanout-split
spec:
  completions: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: splitter
          image: myregistry/data-splitter:latest
          command: ["./splitter", "--partitions=10", "--output=/data/partitions/"]
          volumeMounts:
            - name: shared
              mountPath: /data
      volumes:
        - name: shared
          persistentVolumeClaim:
            claimName: pipeline-data

---
# ------------------------------------------------------------------
# Step 2: Fan-out — process each partition in parallel
# ------------------------------------------------------------------
apiVersion: batch/v1
kind: Job
metadata:
  name: fanout-process
spec:
  completions: 10
  parallelism: 5
  completionMode: Indexed
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: processor
          image: myregistry/data-processor:latest
          command:
            - sh
            - -c
            - |
              INDEX=${JOB_COMPLETION_INDEX}
              ./processor \
                --input=/data/partitions/part_${INDEX}.json \
                --output=/data/results/result_${INDEX}.json
          volumeMounts:
            - name: shared
              mountPath: /data
      volumes:
        - name: shared
          persistentVolumeClaim:
            claimName: pipeline-data

---
# ------------------------------------------------------------------
# Step 3: Fan-in — merge all results
# ------------------------------------------------------------------
apiVersion: batch/v1
kind: Job
metadata:
  name: fanin-merge
spec:
  completions: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: merger
          image: myregistry/data-merger:latest
          command: ["./merger", "--input=/data/results/", "--output=/data/final_report.json"]
          volumeMounts:
            - name: shared
              mountPath: /data
      volumes:
        - name: shared
          persistentVolumeClaim:
            claimName: pipeline-data
```

### 20.4.3 Job Chaining with Init Containers

For simple sequential dependencies, you can use init containers within a
single Job to run prerequisite steps before the main container:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: migration-job
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: Never
      initContainers:
        # Step 1: Wait for database to be ready
        - name: wait-for-db
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              until nc -z postgres-primary 5432; do
                echo "Waiting for PostgreSQL..."
                sleep 2
              done
              echo "Database is ready."

        # Step 2: Take a backup before migration
        - name: pre-migration-backup
          image: postgres:16
          command:
            - sh
            - -c
            - |
              pg_dump -h postgres-primary -U postgres -d myapp \
                --format=custom > /data/pre_migration_backup.dump
              echo "Backup complete."
          volumeMounts:
            - name: data
              mountPath: /data

      # Step 3: Run the actual migration
      containers:
        - name: migrate
          image: myregistry/migrator:latest
          command: ["./migrate", "--database-url=postgres://postgres-primary/myapp"]
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          emptyDir: {}
```

### 20.4.4 Hands-on: Building a Distributed Work Queue Processor in Go

This Go program implements a distributed work queue consumer designed to run
as a Kubernetes Job. Multiple Pods consume items from a shared work queue
(simulated via filesystem) until the queue is empty.

```go
// ================================================================
// main.go — Distributed Work Queue Processor for Kubernetes Jobs
// ================================================================
//
// This program implements a work queue consumer that runs as part
// of a Kubernetes Job. Multiple instances (Pods) run in parallel,
// each consuming work items from a shared queue until the queue
// is exhausted.
//
// The work queue is implemented as a directory of files on a
// shared PersistentVolumeClaim. Each file represents a work item.
// Workers atomically "claim" items by renaming them to a
// processing directory, preventing double-processing.
//
// This pattern is useful for:
// - Image/video processing pipelines
// - Data transformation jobs
// - Any batch workload that can be parallelized
//
// The program handles:
// - Atomic work item claiming (rename-based locking)
// - Graceful shutdown on SIGTERM (Kubernetes Pod termination)
// - Progress reporting to stdout (visible via kubectl logs)
// - Proper exit codes (0 = queue empty, 1 = fatal error)
//
// Environment:
//   QUEUE_DIR      — Directory containing work items (default: /queue/pending)
//   PROCESSING_DIR — Directory for items being processed (default: /queue/processing)
//   DONE_DIR       — Directory for completed items (default: /queue/done)
//   FAILED_DIR     — Directory for failed items (default: /queue/failed)
//   WORKER_ID      — Unique worker identifier (default: hostname)
//   MAX_RETRIES    — Max retries per item (default: 3)
//
// Build:
//   CGO_ENABLED=0 GOOS=linux go build -o queue-processor main.go
//
// ================================================================

package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"math/rand"
	"os"
	"os/signal"
	"path/filepath"
	"strconv"
	"strings"
	"syscall"
	"time"
)

// ----------------------------------------------------------------
// WorkItem represents a single unit of work read from the queue.
// The file on disk contains a JSON-serialized WorkItem.
// ----------------------------------------------------------------
type WorkItem struct {
	// ID is a unique identifier for this work item.
	ID string `json:"id"`

	// Payload contains the work-specific data (e.g., an image URL,
		// a database query, a transformation spec).
	Payload string `json:"payload"`

	// Priority indicates processing urgency (higher = more urgent).
	Priority int `json:"priority"`

	// RetryCount tracks how many times this item has been attempted.
	RetryCount int `json:"retry_count"`

	// CreatedAt records when the item was enqueued.
	CreatedAt string `json:"created_at"`
}

// ----------------------------------------------------------------
// WorkerStats tracks cumulative statistics for this worker's
// lifetime. Logged periodically and at shutdown for observability.
// ----------------------------------------------------------------
type WorkerStats struct {
	// WorkerID identifies this specific worker Pod.
	WorkerID string `json:"worker_id"`

	// ItemsProcessed is the count of successfully processed items.
	ItemsProcessed int `json:"items_processed"`

	// ItemsFailed is the count of items that failed after max retries.
	ItemsFailed int `json:"items_failed"`

	// TotalProcessingMs is cumulative processing time in milliseconds.
	TotalProcessingMs int64 `json:"total_processing_ms"`

	// StartTime is when this worker started.
	StartTime time.Time `json:"start_time"`
}

// ----------------------------------------------------------------
// queueProcessor encapsulates the queue directories, worker
// configuration, and processing state.
// ----------------------------------------------------------------
type queueProcessor struct {
	// queueDir is where pending work items are stored.
	queueDir string

	// processingDir is where items are moved during processing.
	processingDir string

	// doneDir is where successfully processed items are archived.
	doneDir string

	// failedDir is where permanently failed items are stored.
	failedDir string

	// workerID uniquely identifies this worker instance.
	workerID string

	// maxRetries is the maximum number of processing attempts per item.
	maxRetries int

	// stats tracks this worker's cumulative statistics.
	stats WorkerStats
}

// newQueueProcessor creates a queueProcessor from environment
// variables and ensures all required directories exist.
func newQueueProcessor() (*queueProcessor, error) {
	qp := &queueProcessor{
		queueDir:      getEnvDefault("QUEUE_DIR", "/queue/pending"),
		processingDir: getEnvDefault("PROCESSING_DIR", "/queue/processing"),
		doneDir:       getEnvDefault("DONE_DIR", "/queue/done"),
		failedDir:     getEnvDefault("FAILED_DIR", "/queue/failed"),
		workerID:      getEnvDefault("WORKER_ID", mustHostname()),
		maxRetries:    getEnvInt("MAX_RETRIES", 3),
	}

	// Create directories if they don't exist.
	for _, dir := range []string{qp.queueDir, qp.processingDir, qp.doneDir, qp.failedDir} {
		if err := os.MkdirAll(dir, 0755); err != nil {
			return nil, fmt.Errorf("create directory %s: %w", dir, err)
		}
	}

	qp.stats = WorkerStats{
		WorkerID:  qp.workerID,
		StartTime: time.Now(),
	}

	return qp, nil
}

// claimItem attempts to atomically claim the next available work
// item by renaming it from the queue directory to the processing
// directory. The rename is atomic on most filesystems, preventing
// two workers from claiming the same item.
//
// Returns the claimed WorkItem and its file path in the processing
// directory. Returns nil if the queue is empty.
func (qp *queueProcessor) claimItem() (*WorkItem, string, error) {
	entries, err := os.ReadDir(qp.queueDir)
	if err != nil {
		return nil, "", fmt.Errorf("read queue dir: %w", err)
	}

	for _, entry := range entries {
		if entry.IsDir() || !strings.HasSuffix(entry.Name(), ".json") {
			continue
		}

		srcPath := filepath.Join(qp.queueDir, entry.Name())
		dstPath := filepath.Join(qp.processingDir, fmt.Sprintf("%s_%s", qp.workerID, entry.Name()))

		// Attempt atomic rename. If another worker already claimed
		// this file, the rename will fail (source doesn't exist).
		if err := os.Rename(srcPath, dstPath); err != nil {
			// Race condition — another worker claimed it. Try next.
			continue
		}

		// Read the work item.
		data, err := os.ReadFile(dstPath)
		if err != nil {
			return nil, dstPath, fmt.Errorf("read item %s: %w", dstPath, err)
		}

		var item WorkItem
		if err := json.Unmarshal(data, &item); err != nil {
			return nil, dstPath, fmt.Errorf("parse item %s: %w", dstPath, err)
		}

		return &item, dstPath, nil
	}

	// Queue is empty.
	return nil, "", nil
}

// processItem simulates processing a single work item. In a real
// application, this would perform the actual work (image
	// processing, data transformation, API calls, etc.).
//
// Returns an error if processing fails.
func (qp *queueProcessor) processItem(item *WorkItem) error {
	log.Printf("[%s] Processing item %s (priority: %d, attempt: %d/%d)",
		qp.workerID, item.ID, item.Priority, item.RetryCount+1, qp.maxRetries)

	// Simulate variable processing time.
	processingTime := time.Duration(100+rand.Intn(900)) * time.Millisecond
	time.Sleep(processingTime)

	// Simulate occasional transient failures (10% rate).
	if rand.Float64() < 0.10 {
		return fmt.Errorf("simulated transient error processing %s", item.ID)
	}

	log.Printf("[%s] Item %s processed successfully in %v",
		qp.workerID, item.ID, processingTime)

	return nil
}

// completeItem moves a processed item from the processing directory
// to the done directory.
func (qp *queueProcessor) completeItem(processingPath string) error {
	filename := filepath.Base(processingPath)
	donePath := filepath.Join(qp.doneDir, filename)
	return os.Rename(processingPath, donePath)
}

// failItem handles a failed work item. If retries remain, the item
// is returned to the queue with an incremented retry count.
// If retries are exhausted, it is moved to the failed directory.
func (qp *queueProcessor) failItem(item *WorkItem, processingPath string) error {
	item.RetryCount++

	if item.RetryCount < qp.maxRetries {
		// Return to queue for retry.
		log.Printf("[%s] Item %s failed (attempt %d/%d) — returning to queue",
			qp.workerID, item.ID, item.RetryCount, qp.maxRetries)

		data, err := json.Marshal(item)
		if err != nil {
			return fmt.Errorf("marshal item: %w", err)
		}

		retryPath := filepath.Join(qp.queueDir, fmt.Sprintf("%s.json", item.ID))
		if err := os.WriteFile(retryPath, data, 0644); err != nil {
			return fmt.Errorf("write retry item: %w", err)
		}

		return os.Remove(processingPath)
	}

	// Max retries exhausted — move to failed directory.
	log.Printf("[%s] Item %s permanently failed after %d attempts",
		qp.workerID, item.ID, qp.maxRetries)

	failedPath := filepath.Join(qp.failedDir, filepath.Base(processingPath))
	return os.Rename(processingPath, failedPath)
}

// run is the main processing loop. It continuously claims and
// processes work items until the queue is empty or the context
// is cancelled (SIGTERM).
func (qp *queueProcessor) run(ctx context.Context) error {
	log.Printf("[%s] Worker starting. Queue: %s", qp.workerID, qp.queueDir)

	consecutiveEmpty := 0
	const maxEmptyChecks = 3 // Exit after 3 consecutive empty checks

	for {
		// Check for shutdown signal.
		select {
		case <-ctx.Done():
			log.Printf("[%s] Shutdown signal received. Stopping gracefully.", qp.workerID)
			return nil
		default:
		}

		// Claim next item.
		item, processingPath, err := qp.claimItem()
		if err != nil {
			log.Printf("[%s] Error claiming item: %v", qp.workerID, err)
			time.Sleep(time.Second)
			continue
		}

		if item == nil {
			consecutiveEmpty++
			if consecutiveEmpty >= maxEmptyChecks {
				log.Printf("[%s] Queue empty after %d checks. Exiting.",
					qp.workerID, maxEmptyChecks)
				return nil
			}
			log.Printf("[%s] Queue appears empty (check %d/%d). Waiting...",
				qp.workerID, consecutiveEmpty, maxEmptyChecks)
			time.Sleep(2 * time.Second)
			continue
		}

		consecutiveEmpty = 0

		// Process the item.
		start := time.Now()
		if err := qp.processItem(item); err != nil {
			// Processing failed.
			if failErr := qp.failItem(item, processingPath); failErr != nil {
				log.Printf("[%s] Error handling failed item: %v", qp.workerID, failErr)
			}
			qp.stats.ItemsFailed++
		} else {
			// Processing succeeded.
			if err := qp.completeItem(processingPath); err != nil {
				log.Printf("[%s] Error completing item: %v", qp.workerID, err)
			}
			qp.stats.ItemsProcessed++
		}

		qp.stats.TotalProcessingMs += time.Since(start).Milliseconds()

		// Log periodic stats.
		total := qp.stats.ItemsProcessed + qp.stats.ItemsFailed
		if total%10 == 0 {
			log.Printf("[%s] Stats: %d processed, %d failed, %dms total",
				qp.workerID, qp.stats.ItemsProcessed, qp.stats.ItemsFailed,
				qp.stats.TotalProcessingMs)
		}
	}
}

// ----------------------------------------------------------------
// Utility functions
// ----------------------------------------------------------------

// getEnvDefault returns the value of the named environment variable
// or the default if it is empty or unset.
func getEnvDefault(key, defaultVal string) string {
	if val := os.Getenv(key); val != "" {
		return val
	}
	return defaultVal
}

// getEnvInt returns the integer value of the named environment
// variable, or the default on parse failure or if unset.
func getEnvInt(key string, defaultVal int) int {
	val := os.Getenv(key)
	if val == "" {
		return defaultVal
	}
	n, err := strconv.Atoi(val)
	if err != nil {
		return defaultVal
	}
	return n
}

// mustHostname returns the system hostname or "unknown" if it
// cannot be determined.
func mustHostname() string {
	h, err := os.Hostname()
	if err != nil {
		return "unknown"
	}
	return h
}

// ----------------------------------------------------------------
// main sets up signal handling for graceful shutdown and runs the
// work queue processor until the queue is empty or the Pod is
// terminated.
// ----------------------------------------------------------------
func main() {
	log.SetFlags(log.Ldate | log.Ltime | log.Lshortfile)
	rand.New(rand.NewSource(time.Now().UnixNano()))

	qp, err := newQueueProcessor()
	if err != nil {
		log.Fatalf("Failed to initialize queue processor: %v", err)
	}

	// Set up graceful shutdown on SIGTERM (sent by Kubernetes during
		// Pod termination) and SIGINT (for local development).
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)
	go func() {
		sig := <-sigCh
		log.Printf("Received signal %v", sig)
		cancel()
	}()

	// Run the processing loop.
	if err := qp.run(ctx); err != nil {
		log.Fatalf("Worker failed: %v", err)
	}

	// Print final stats.
	elapsed := time.Since(qp.stats.StartTime)
	log.Printf("=== FINAL STATS ===")
	log.Printf("Worker:     %s", qp.stats.WorkerID)
	log.Printf("Processed:  %d items", qp.stats.ItemsProcessed)
	log.Printf("Failed:     %d items", qp.stats.ItemsFailed)
	log.Printf("Duration:   %v", elapsed.Round(time.Second))
	if qp.stats.ItemsProcessed > 0 {
		avgMs := qp.stats.TotalProcessingMs / int64(qp.stats.ItemsProcessed+qp.stats.ItemsFailed)
		log.Printf("Avg time:   %dms per item", avgMs)
	}
}
```

**Kubernetes manifests for the work queue processor:**

```yaml
# ------------------------------------------------------------------
# queue-setup-job.yaml
# Pre-populate the work queue with items. Run this before the
# processing Job.
# ------------------------------------------------------------------
apiVersion: batch/v1
kind: Job
metadata:
  name: queue-setup
spec:
  completions: 1
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: setup
          image: busybox:1.36
          command:
            - sh
            - -c
            - |
              mkdir -p /queue/pending /queue/processing /queue/done /queue/failed

              # Generate 100 work items
              for i in $(seq 1 100); do
                ID=$(printf "item-%04d" $i)
                cat > "/queue/pending/${ID}.json" <<EOF
              {
                "id": "${ID}",
                "payload": "Process data batch ${i}",
                "priority": $((RANDOM % 10)),
                "retry_count": 0,
                "created_at": "$(date -u +%Y-%m-%dT%H:%M:%SZ)"
              }
              EOF
              done

              echo "Created 100 work items in /queue/pending/"
              ls /queue/pending/ | wc -l
          volumeMounts:
            - name: queue
              mountPath: /queue
      volumes:
        - name: queue
          persistentVolumeClaim:
            claimName: work-queue-pvc

---
# ------------------------------------------------------------------
# queue-processor-job.yaml
# Process all items in the queue using 5 parallel workers.
# No "completions" field — this is a work-queue-style Job.
# Workers exit when the queue is empty.
# ------------------------------------------------------------------
apiVersion: batch/v1
kind: Job
metadata:
  name: queue-processor
spec:
  parallelism: 5
  backoffLimit: 15
  activeDeadlineSeconds: 3600
  ttlSecondsAfterFinished: 86400
  template:
    spec:
      restartPolicy: Never
      containers:
        - name: processor
          image: myregistry/queue-processor:latest
          env:
            - name: QUEUE_DIR
              value: /queue/pending
            - name: PROCESSING_DIR
              value: /queue/processing
            - name: DONE_DIR
              value: /queue/done
            - name: FAILED_DIR
              value: /queue/failed
            - name: MAX_RETRIES
              value: "3"
            - name: WORKER_ID
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
          volumeMounts:
            - name: queue
              mountPath: /queue
          resources:
            requests:
              cpu: 200m
              memory: 128Mi
            limits:
              cpu: "1"
              memory: 512Mi
      volumes:
        - name: queue
          persistentVolumeClaim:
            claimName: work-queue-pvc

---
# ------------------------------------------------------------------
# work-queue-pvc.yaml
# Shared storage for the work queue. Uses ReadWriteMany so
# multiple Pods can access it concurrently.
# ------------------------------------------------------------------
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: work-queue-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: nfs  # Must support ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```

---

## 20.5 DaemonSet vs Deployment — When to Use What

| Criterion                      | Deployment                  | DaemonSet                    |
|--------------------------------|-----------------------------|------------------------------|
| **Scheduling model**           | Scheduler assigns Nodes     | One Pod per Node (or subset) |
| **Replica count**              | You set it explicitly       | Equals number of matching Nodes |
| **Scaling**                    | `kubectl scale`             | Automatic as Nodes join/leave |
| **Use case**                   | Application workloads       | Infrastructure/node-level agents |
| **Host access needed**         | Rarely                      | Often (hostPath, hostNetwork) |
| **Update strategy**            | RollingUpdate, Recreate     | RollingUpdate, OnDelete       |

---

## 20.6 Job vs CronJob — When to Use What

| Criterion                     | Job                          | CronJob                      |
|-------------------------------|------------------------------|------------------------------|
| **Trigger**                   | Manual (kubectl/API/CI)      | Scheduled (cron expression)  |
| **Recurrence**                | One-time                     | Recurring                    |
| **Creates**                   | Pods directly                | Jobs (which create Pods)     |
| **History**                   | Single Job object            | Multiple Job objects (configurable limit) |
| **Concurrency control**       | Via parallelism/completions  | Via concurrencyPolicy        |

---

## 20.7 Production Checklist

### DaemonSets

- [ ] **Tolerations** configured for control-plane and other tainted Nodes (if needed).
- [ ] **Resource limits** set — DaemonSet Pods compete with application Pods on every Node.
- [ ] **Priority class** set for critical infrastructure (e.g., `system-node-critical`).
- [ ] **Update strategy** chosen — `RollingUpdate` with `maxUnavailable: 1` for safety.
- [ ] **hostPath volumes** are read-only where possible.
- [ ] **Security context** is restrictive (`readOnlyRootFilesystem`, no privilege escalation).

### Jobs

- [ ] **backoffLimit** set appropriately — too low wastes failed attempts, too high delays failure detection.
- [ ] **activeDeadlineSeconds** set to prevent runaway Jobs.
- [ ] **ttlSecondsAfterFinished** set to auto-cleanup completed Jobs.
- [ ] **restartPolicy** is `Never` or `OnFailure` (required for Jobs).
- [ ] **podFailurePolicy** configured to distinguish retryable from fatal errors.
- [ ] **Resource requests** set — Jobs can burst resource usage unexpectedly.

### CronJobs

- [ ] **concurrencyPolicy** set — usually `Forbid` for database operations.
- [ ] **startingDeadlineSeconds** set to handle controller downtime.
- [ ] **timeZone** explicitly set (don't rely on controller default).
- [ ] **History limits** configured to retain enough failed Jobs for debugging.
- [ ] **Manual trigger** tested with `kubectl create job --from=cronjob/<name>`.

---

## 20.8 Debugging DaemonSets, Jobs, and CronJobs

### DaemonSets

```bash
# Check DaemonSet status — desired vs current vs ready
kubectl get daemonset -n monitoring
# NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR
# node-monitor   5         5         5       5            5           <none>

# Find Nodes missing a DaemonSet Pod
kubectl get nodes -o name | while read node; do
  POD=$(kubectl get pods -n monitoring -l app=node-monitor \
    --field-selector "spec.nodeName=${node##node/}" --no-headers 2>/dev/null)
  if [ -z "$POD" ]; then
    echo "MISSING: ${node}"
  fi
done

# Check why a Pod wasn't scheduled on a specific Node
kubectl describe pod -n monitoring <pod-name>
```

### Jobs

```bash
# Check Job status
kubectl get job data-import
# NAME          COMPLETIONS   DURATION   AGE
# data-import   7/10          3m         3m

# View failed Pods
kubectl get pods -l job-name=data-import --field-selector=status.phase=Failed

# Check failure details
kubectl describe job data-import
kubectl logs <failed-pod-name>

# Check indexed Job progress
kubectl get pods -l job-name=data-import \
  -o custom-columns=NAME:.metadata.name,INDEX:.metadata.annotations."batch\.kubernetes\.io/job-completion-index",STATUS:.status.phase
```

### CronJobs

```bash
# Check CronJob status and next schedule
kubectl get cronjob
# NAME        SCHEDULE    SUSPEND   ACTIVE   LAST SCHEDULE   AGE
# db-backup   0 2 * * *   False     0        26h             30d

# List Jobs created by the CronJob
kubectl get jobs -l job-name=db-backup

# Manually trigger for testing
kubectl create job --from=cronjob/db-backup db-backup-test-$(date +%s)

# Suspend a CronJob (prevent new runs)
kubectl patch cronjob db-backup -p '{"spec":{"suspend":true}}'

# Resume
kubectl patch cronjob db-backup -p '{"spec":{"suspend":false}}'
```

---

## 20.9 Summary

This chapter covered the three remaining workload controllers in Kubernetes:

1. **DaemonSets** ensure exactly one Pod runs on every Node (or a subset).
   They are the backbone of cluster infrastructure — log collection,
   monitoring, networking, and storage all depend on DaemonSets. The
   controller bypasses the scheduler and directly assigns Pods to Nodes,
   making tolerations essential for running on tainted Nodes.

2. **Jobs** provide run-to-completion semantics. They support single-shot
   tasks, parallel batch processing (with `parallelism`), and indexed
   partitioning (with `completionMode: Indexed`). The `podFailurePolicy`
   gives fine-grained control over retry behavior, and
   `ttlSecondsAfterFinished` handles automatic cleanup.

3. **CronJobs** create Jobs on a cron schedule. The `concurrencyPolicy` field
   prevents overlapping runs, `timeZone` ensures schedule accuracy, and
   `startingDeadlineSeconds` handles missed schedules gracefully.

Together with Deployments (Chapter 17) and StatefulSets (Chapter 19), these
five controllers — Deployment, StatefulSet, DaemonSet, Job, and CronJob —
cover the vast majority of workload patterns in Kubernetes. In the next
chapter, we move to networking and Services, exploring how these workloads
communicate with each other and the outside world.

---

## 20.10 Exercises

1. **DaemonSet targeting:** Deploy a DaemonSet that runs only on Nodes with
   the label `disk=ssd`. Add the label to one Node and verify the DaemonSet
   creates a Pod on it. Remove the label and verify the Pod is deleted.

2. **DaemonSet update:** Deploy a DaemonSet with `updateStrategy: OnDelete`.
   Update the image version and verify that existing Pods are NOT updated.
   Manually delete one Pod and verify the replacement uses the new image.

3. **Indexed Job:** Create an Indexed Job with 5 completions and 2
   parallelism. Have each Pod print its `JOB_COMPLETION_INDEX` and sleep for
   a random duration. Observe how Kubernetes maintains at most 2 running Pods.

4. **Pod failure policy:** Create a Job with a `podFailurePolicy` that
   immediately fails the Job on exit code 42 but retries on exit code 1.
   Test both paths.

5. **CronJob concurrency:** Create a CronJob that runs every minute with
   `concurrencyPolicy: Forbid`. Make each Job sleep for 90 seconds. Observe
   that every other scheduled run is skipped.

6. **Work queue:** Implement the work queue pattern from Section 20.4.4.
   Pre-populate a queue with 50 items, then run 3 parallel workers. Verify
   that all items are processed exactly once (no duplicates, no misses).

7. **Fan-out/fan-in:** Implement the fan-out/fan-in pattern from Section
   20.4.2. Use one Job to split a CSV file into 5 partitions, an Indexed Job
   to process each partition, and a final Job to merge the results.
