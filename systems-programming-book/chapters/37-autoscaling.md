# Chapter 37: Autoscaling — HPA, VPA, Cluster Autoscaler, KEDA

## Introduction

In Chapter 36, we built a comprehensive observability stack that gives us deep
visibility into our applications. Now we put that data to work. **Autoscaling**
is the practice of automatically adjusting compute resources in response to
changing demand. Instead of a human watching dashboards and manually scaling
deployments, Kubernetes can do this automatically — reacting to CPU spikes,
memory pressure, queue depth, request rate, or any custom metric your application
exposes.

Kubernetes provides multiple autoscaling mechanisms at different levels:

| Autoscaler        | What it scales          | Based on                    |
|-------------------|-------------------------|-----------------------------|
| HPA               | Pod replicas            | Metrics (CPU, custom, etc.) |
| VPA               | Pod resource requests   | Historical usage            |
| Cluster Autoscaler| Cluster nodes           | Pending pods                |
| KEDA              | Pod replicas (+ jobs)   | Event sources (Kafka, etc.) |

This chapter covers all four in depth, with production-ready configurations,
Go application examples that expose custom metrics for scaling, and detailed
explanations of the algorithms behind each autoscaler.

---

## 37.1 Horizontal Pod Autoscaler (HPA)

### 37.1.1 How HPA Works

The HPA controller runs as part of the `kube-controller-manager`. Every 15 seconds
(configurable via `--horizontal-pod-autoscaler-sync-period`), it:

1. **Fetches metrics** from the metrics API (metrics-server for CPU/memory, or a
   custom metrics adapter for Prometheus metrics)
2. **Calculates the desired replica count** using the formula:
   ```
   desiredReplicas = ceil(currentReplicas × (currentMetricValue / desiredMetricValue))
   ```
3. **Scales the target** (Deployment, StatefulSet, ReplicaSet) to the desired replica count

**Example calculation:**

```
Current replicas: 3
Current average CPU: 80%
Target CPU: 50%

desiredReplicas = ceil(3 × (80 / 50)) = ceil(4.8) = 5
```

The HPA scales from 3 to 5 replicas to bring average CPU closer to 50%.

### 37.1.2 Scaling Algorithm Details

The HPA algorithm has several important nuances:

**Tolerance:** The HPA has a default tolerance of 10% (configurable via
`--horizontal-pod-autoscaler-tolerance`). If the metric ratio is within
[0.9, 1.1] of the target, no scaling occurs. This prevents thrashing.

**Missing metrics:** If some pods don't report metrics, the HPA uses conservative
estimates. Pods missing metrics are assumed to consume 100% when scaling down,
and 0% when scaling up.

**Multiple metrics:** When the HPA is configured with multiple metrics, it
calculates the desired replicas for each metric independently and takes the
**maximum** of all calculated values.

```
Metric 1 (CPU):    desiredReplicas = 5
Metric 2 (memory): desiredReplicas = 3
Metric 3 (RPS):    desiredReplicas = 7

Final decision: 7 replicas (maximum)
```

### 37.1.3 Metric Types

The HPA supports four metric types:

#### Resource Metrics (CPU, Memory)

The simplest and most common. Uses metrics-server data.

```yaml
# hpa-cpu.yaml
# Scales the deployment based on average CPU utilization across all pods.
# When average CPU exceeds 50%, the HPA adds replicas.
# When average CPU drops well below 50%, the HPA removes replicas.
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: default
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
    # Target 50% average CPU utilization.
    # "Utilization" is relative to the pod's CPU request.
    # If request is 100m and pod is using 50m, utilization = 50%.
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
    # Also scale on memory — the HPA takes the max of both.
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
```

#### Pods Metrics

Custom metrics that are per-pod averages. These come from the custom metrics API.

```yaml
# hpa-pods-metric.yaml
# Scales based on a per-pod average of a custom metric.
# In this case, the application exposes "http_requests_per_second"
# and we target an average of 1000 RPS per pod.
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa-rps
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 50
  metrics:
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
```

#### Object Metrics

Metrics associated with a Kubernetes object (like an Ingress or Service).

```yaml
# hpa-object-metric.yaml
# Scales based on the total requests-per-second hitting the Ingress.
# Useful when the metric is cluster-wide, not per-pod.
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa-ingress
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 30
  metrics:
    - type: Object
      object:
        describedObject:
          apiVersion: networking.k8s.io/v1
          kind: Ingress
          name: myapp-ingress
        metric:
          name: requests_per_second
        target:
          type: Value
          value: "10000"
```

#### External Metrics

Metrics from systems outside Kubernetes (cloud provider metrics, SQS queue depth,
Pub/Sub message count).

```yaml
# hpa-external-metric.yaml
# Scales based on an external metric — in this case, the depth of
# an AWS SQS queue. As messages pile up, more pods are added to
# process them.
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa-sqs
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-worker
  minReplicas: 1
  maxReplicas: 100
  metrics:
    - type: External
      external:
        metric:
          name: sqs_queue_depth
          selector:
            matchLabels:
              queue: "orders-processing"
        target:
          type: AverageValue
          averageValue: "30"   # Target 30 messages per pod
```

### 37.1.4 Behavior Configuration

The `behavior` field (introduced in `autoscaling/v2`) gives fine-grained control
over how fast the HPA scales up and down. This is critical for preventing
**thrashing** — rapid scaling up and down.

```yaml
# hpa-behavior.yaml
# Configures scaling behavior with stabilization windows and rate limits.
# This prevents the HPA from overreacting to brief metric spikes.
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa-tuned
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
  behavior:
    # Scale-up behavior: be aggressive when demand increases.
    scaleUp:
      # Stabilization window: after a scale-up, wait at least 60 seconds
      # before considering another scale-up. This gives new pods time
      # to start handling traffic and affect the average metrics.
      stabilizationWindowSeconds: 60
      policies:
        # Policy 1: Add up to 4 pods per 60 seconds.
        - type: Pods
          value: 4
          periodSeconds: 60
        # Policy 2: Double the replica count per 60 seconds.
        - type: Percent
          value: 100
          periodSeconds: 60
      # Use the maximum of all policies (most aggressive).
      selectPolicy: Max

    # Scale-down behavior: be conservative when demand decreases.
    scaleDown:
      # Stabilization window: after a scale-down, wait at least 300 seconds
      # (5 minutes) before considering another scale-down. This prevents
      # the HPA from removing pods too quickly during brief traffic dips.
      stabilizationWindowSeconds: 300
      policies:
        # Policy: Remove at most 1 pod per 60 seconds.
        - type: Pods
          value: 1
          periodSeconds: 60
      selectPolicy: Min
```

**Stabilization window explained:**

```
Time  CPU%  Without stabilization    With 300s window
─────────────────────────────────────────────────────
0:00  80%   Scale up to 8            Scale up to 8
0:05  30%   Scale down to 4          (wait 300s)
0:06  75%   Scale up to 7            (still at 8)
0:07  35%   Scale down to 4          (still at 8)
0:10  70%   Scale up to 6            (still at 8)
```

Without stabilization, the HPA thrashes. With a 300-second window, it looks at
the highest recommendation over the last 5 minutes (for scale-down), preventing
premature scale-down.

### 37.1.5 Hands-On: HPA with CPU Target

```bash
# Step 1: Deploy the application with CPU requests defined
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myregistry/myapp:v1.0.0
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 100m      # REQUIRED for CPU-based HPA
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
---
apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
EOF

# Step 2: Create the HPA
cat <<'EOF' | kubectl apply -f -
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 60
EOF

# Step 3: Generate load to trigger scaling
kubectl run load-generator --rm -i --tty --image=busybox -- /bin/sh -c \
  "while true; do wget -q -O- http://myapp; done"

# Step 4: Watch the HPA in action
kubectl get hpa myapp-hpa --watch

# Expected output:
# NAME        REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS
# myapp-hpa   Deployment/myapp   23%/50%   2         10        2
# myapp-hpa   Deployment/myapp   78%/50%   2         10        2
# myapp-hpa   Deployment/myapp   78%/50%   2         10        4
# myapp-hpa   Deployment/myapp   52%/50%   2         10        4
```

### 37.1.6 Hands-On: HPA with Custom Prometheus Metrics

To use custom metrics from Prometheus, you need a **custom metrics adapter**.
The most common is the `prometheus-adapter`.

```bash
# Install prometheus-adapter
helm install prometheus-adapter \
  prometheus-community/prometheus-adapter \
  --namespace monitoring \
  --set prometheus.url=http://kube-prometheus-stack-prometheus.monitoring.svc \
  --set prometheus.port=9090
```

Configure the adapter to expose your custom metric:

```yaml
# prometheus-adapter-config.yaml
# Maps Prometheus metrics to Kubernetes custom metrics API.
apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-adapter
  namespace: monitoring
data:
  config.yaml: |
    rules:
      # Expose per-pod request rate as a custom metric.
      # This transforms the Prometheus counter into a rate that the
      # HPA can use.
      - seriesQuery: 'myapp_http_requests_total{namespace!="",pod!=""}'
        resources:
          overrides:
            namespace:
              resource: namespace
            pod:
              resource: pod
        name:
          matches: "^(.*)_total$"
          as: "${1}_per_second"
        metricsQuery: 'sum(rate(<<.Series>>{<<.LabelMatchers>>}[2m])) by (<<.GroupBy>>)'

      # Expose queue depth as a custom metric.
      - seriesQuery: 'myapp_queue_depth{namespace!="",pod!=""}'
        resources:
          overrides:
            namespace:
              resource: namespace
            pod:
              resource: pod
        name:
          as: "myapp_queue_depth"
        metricsQuery: 'avg(<<.Series>>{<<.LabelMatchers>>}) by (<<.GroupBy>>)'
```

Now create an HPA that uses the custom metric:

```yaml
# hpa-custom-metric.yaml
# Scales based on requests-per-second from Prometheus.
# Each pod should handle ~500 requests/second.
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa-custom
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 30
  metrics:
    # Custom metric from Prometheus via prometheus-adapter
    - type: Pods
      pods:
        metric:
          name: myapp_http_requests_per_second
        target:
          type: AverageValue
          averageValue: "500"
    # Also keep CPU as a safety net
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 30
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 2
          periodSeconds: 120
```

Verify the custom metric is available:

```bash
# Check that the custom metrics API is working
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1" | jq .

# Query the specific metric
kubectl get --raw "/apis/custom.metrics.k8s.io/v1beta1/namespaces/default/pods/*/myapp_http_requests_per_second" | jq .
```

### 37.1.7 Hands-On: Go App That Exposes Custom Metrics for HPA

```go
// hpa_metrics.go
//
// A Go application that exposes custom Prometheus metrics specifically
// designed for HPA consumption. This demonstrates:
//
//   - Exposing a "requests per second" gauge that the prometheus-adapter
//     can serve to the HPA via the custom metrics API.
//   - Exposing a "queue depth" gauge for scaling based on pending work.
//   - Using a separate goroutine to calculate rate metrics, since
//     Prometheus counters need rate() which requires the adapter.
//
// The key insight: HPA works best with instantaneous gauge values
// (like queue depth) rather than counters (which require rate()
// computation). While the prometheus-adapter can compute rate(),
// exposing a pre-computed gauge is simpler and more reliable.
package main

import (
	"fmt"
	"log"
	"math/rand"
	"net/http"
	"sync"
	"sync/atomic"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// queueDepth is a gauge that reports the current number of items
// waiting to be processed. This is an ideal HPA metric because:
//   - It directly reflects pending work (demand signal)
//   - It naturally goes to zero when the queue is drained
//   - The HPA can target "averageValue: 10" meaning each pod
//     should have ~10 items in its queue
var queueDepth = promauto.NewGauge(
	prometheus.GaugeOpts{
		Namespace: "myapp",
		Name:      "queue_depth",
		Help:      "Current number of items in the processing queue.",
	},
)

// processingDuration records how long each queue item takes to process.
// This helps set appropriate HPA targets: if items take 100ms each and
// the target is 10 items per pod, each pod processes ~100 items/sec.
var processingDuration = promauto.NewHistogram(
	prometheus.HistogramOpts{
		Namespace: "myapp",
		Name:      "queue_processing_duration_seconds",
		Help:      "Time to process a single queue item.",
		Buckets:   []float64{0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1},
	},
)

// activeWorkers tracks the number of goroutines currently processing
// queue items. Useful for understanding if the application is CPU-bound
// or I/O-bound.
var activeWorkers = promauto.NewGauge(
	prometheus.GaugeOpts{
		Namespace: "myapp",
		Name:      "active_workers",
		Help:      "Number of goroutines currently processing queue items.",
	},
)

// requestCounter is an atomic counter used to compute the request rate
// gauge below. We use atomic operations for thread-safe counting.
var requestCounter int64

// requestRate is a pre-computed gauge that reports the current request
// rate in requests/second. This is updated every 10 seconds by a
// background goroutine.
//
// Why a gauge instead of using rate() on a counter?
// The prometheus-adapter can compute rate(), but a pre-computed gauge is:
//   - Simpler to configure in the adapter
//   - More predictable (no rate() windowing issues)
//   - Easier to test
var requestRate = promauto.NewGauge(
	prometheus.GaugeOpts{
		Namespace: "myapp",
		Name:      "http_requests_per_second",
		Help:      "Current HTTP request rate (computed every 10s).",
	},
)

// WorkQueue is a simple in-memory work queue that simulates a message
// queue or task queue. In production, this would be backed by Kafka,
// RabbitMQ, Redis, or similar.
type WorkQueue struct {
	mu    sync.Mutex
	items []string
}

// Enqueue adds an item to the queue and updates the depth metric.
func (q *WorkQueue) Enqueue(item string) {
	q.mu.Lock()
	defer q.mu.Unlock()
	q.items = append(q.items, item)
	queueDepth.Set(float64(len(q.items)))
}

// Dequeue removes and returns the next item from the queue.
// Returns empty string and false if the queue is empty.
func (q *WorkQueue) Dequeue() (string, bool) {
	q.mu.Lock()
	defer q.mu.Unlock()
	if len(q.items) == 0 {
		return "", false
	}
	item := q.items[0]
	q.items = q.items[1:]
	queueDepth.Set(float64(len(q.items)))
	return item, true
}

// Len returns the current queue depth.
func (q *WorkQueue) Len() int {
	q.mu.Lock()
	defer q.mu.Unlock()
	return len(q.items)
}

var queue = &WorkQueue{}

// startRateCalculator runs a background goroutine that computes the
// request rate every 10 seconds and updates the requestRate gauge.
// This provides a smoothed rate that's ideal for HPA consumption.
func startRateCalculator() {
	ticker := time.NewTicker(10 * time.Second)
	var lastCount int64

	go func() {
		for range ticker.C {
			current := atomic.LoadInt64(&requestCounter)
			rate := float64(current-lastCount) / 10.0 // requests per second
			requestRate.Set(rate)
			lastCount = current
		}
	}()
}

// startQueueProducer simulates external work arriving in the queue.
// In production, this would be a Kafka consumer, HTTP webhook handler, etc.
func startQueueProducer() {
	go func() {
		for {
			// Simulate bursty traffic: sometimes many items arrive at once.
			batchSize := rand.Intn(20) + 1
			for i := 0; i < batchSize; i++ {
				queue.Enqueue(fmt.Sprintf("item-%d", time.Now().UnixNano()))
			}
			time.Sleep(time.Duration(100+rand.Intn(900)) * time.Millisecond)
		}
	}()
}

// startQueueWorkers starts N goroutines that process items from the queue.
// Each worker picks an item, processes it (simulated), and loops.
func startQueueWorkers(numWorkers int) {
	for i := 0; i < numWorkers; i++ {
		go func(workerID int) {
			for {
				item, ok := queue.Dequeue()
				if !ok {
					// Queue is empty — back off to avoid busy-waiting.
					time.Sleep(100 * time.Millisecond)
					continue
				}

				activeWorkers.Inc()

				// Simulate processing time (50ms to 200ms).
				start := time.Now()
				processItem(item)
				processingDuration.Observe(time.Since(start).Seconds())

				activeWorkers.Dec()
			}
		}(i)
	}
}

// processItem simulates processing a queue item. The variable duration
// simulates real-world processing where some items are more expensive
// than others.
func processItem(item string) {
	duration := time.Duration(50+rand.Intn(150)) * time.Millisecond
	time.Sleep(duration)
	// In production: actual business logic here
	_ = item
}

// handleEnqueue is an HTTP handler that accepts work items via POST.
// Each call adds an item to the queue and increments the request counter.
func handleEnqueue(w http.ResponseWriter, r *http.Request) {
	if r.Method != http.MethodPost {
		http.Error(w, "method not allowed", http.StatusMethodNotAllowed)
		return
	}

	atomic.AddInt64(&requestCounter, 1)

	item := fmt.Sprintf("http-item-%d", time.Now().UnixNano())
	queue.Enqueue(item)

	w.WriteHeader(http.StatusAccepted)
	fmt.Fprintf(w, `{"queued": "%s", "queue_depth": %d}`, item, queue.Len())
}

func main() {
	// Start background systems.
	startRateCalculator()
	startQueueProducer()
	startQueueWorkers(5)

	// HTTP endpoints.
	http.HandleFunc("/enqueue", handleEnqueue)
	http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		fmt.Fprint(w, `{"status": "ok"}`)
	})
	http.Handle("/metrics", promhttp.Handler())

	log.Println("Starting server on :8080")
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## 37.2 Vertical Pod Autoscaler (VPA)

### 37.2.1 What VPA Does

While HPA scales **horizontally** (more pods), VPA scales **vertically** (bigger
pods). It automatically adjusts the CPU and memory **requests** (and optionally
limits) of your containers based on actual usage patterns.

**Why VPA matters:**
- Most teams guess at resource requests and get them wrong
- Over-provisioning wastes cluster resources (and money)
- Under-provisioning causes OOM kills and CPU throttling
- VPA learns from actual usage and recommends the right values

### 37.2.2 Architecture

```
┌─────────────────────────────────────────────────────┐
│                        VPA                           │
│                                                      │
│  ┌──────────────┐  ┌─────────────┐  ┌─────────────┐│
│  │  Recommender  │  │   Updater   │  │  Admission  ││
│  │              │  │             │  │  Controller  ││
│  │ Watches pod  │  │ Evicts pods │  │ Mutates pod  ││
│  │ metrics,     │  │ that need   │  │ specs on     ││
│  │ calculates   │──│ resizing    │──│ creation to  ││
│  │ optimal      │  │ (triggers   │  │ apply VPA    ││
│  │ requests     │  │  restart)   │  │ recommendations││
│  └──────────────┘  └─────────────┘  └─────────────┘│
└─────────────────────────────────────────────────────┘
```

**Recommender:** Watches pod resource usage over time (from metrics-server) and
calculates optimal CPU/memory requests. It uses a decaying histogram to weight
recent usage more heavily.

**Updater:** Compares current pod resource requests with VPA recommendations.
If the difference exceeds a threshold, it evicts the pod so it can be recreated
with updated requests (via the Admission Controller).

**Admission Controller:** A mutating webhook that intercepts pod creation and
modifies the resource requests to match VPA recommendations.

### 37.2.3 Update Modes

| Mode     | Recommender | Updater | Admission Controller | Description                          |
|----------|-------------|---------|----------------------|--------------------------------------|
| Off      | ✓           | ✗       | ✗                    | Only generates recommendations       |
| Initial  | ✓           | ✗       | ✓                    | Sets requests only at pod creation   |
| Recreate | ✓           | ✓       | ✓                    | Evicts and recreates pods to resize  |
| Auto     | ✓           | ✓       | ✓                    | Same as Recreate (may use in-place in future) |

**Best practice:** Start with `Off` mode to see recommendations without any
automatic changes. Once you trust the recommendations, move to `Auto`.

### 37.2.4 VPA Resource

```yaml
# vpa.yaml
# Configures VPA for the myapp deployment.
# The VPA will analyze resource usage and set appropriate requests.
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
  namespace: default
spec:
  # Target the deployment whose pods should be right-sized.
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp

  # UpdatePolicy controls how recommendations are applied.
  updatePolicy:
    # "Auto" means the VPA will evict pods and recreate them
    # with updated resource requests when recommendations change
    # significantly.
    # Use "Off" to only generate recommendations without acting.
    updateMode: "Auto"

  # ResourcePolicy sets constraints on what the VPA can recommend.
  # Without constraints, VPA might recommend very small or very large
  # values that could cause problems.
  resourcePolicy:
    containerPolicies:
      - containerName: myapp
        # Minimum resources — VPA will never recommend less than this.
        # Set this to the minimum your app needs to start and run.
        minAllowed:
          cpu: 50m
          memory: 64Mi
        # Maximum resources — VPA will never recommend more than this.
        # Set this based on your node size and budget.
        maxAllowed:
          cpu: "2"
          memory: 2Gi
        # Which resources VPA should manage.
        controlledResources: ["cpu", "memory"]
        # ControlledValues determines whether VPA sets just "requests"
        # or both "requests" and "limits".
        # "RequestsOnly" is recommended — it lets CPU burst above
        # the request while preventing OOM (memory limit is kept).
        controlledValues: RequestsOnly

      # You can disable VPA for specific containers (e.g., sidecars).
      - containerName: istio-proxy
        mode: "Off"
```

### 37.2.5 Hands-On: VPA for Right-Sizing Resource Requests

```bash
# Step 1: Install VPA
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler
./hack/vpa-up.sh

# Verify VPA components
kubectl -n kube-system get pods | grep vpa

# Step 2: Deploy an application with intentionally wrong resource requests
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: myapp
          image: myregistry/myapp:v1.0.0
          resources:
            # Intentionally wrong: app actually uses ~200m CPU and ~300Mi memory
            requests:
              cpu: "1"           # Way too high
              memory: 1Gi        # Way too high
            limits:
              cpu: "2"
              memory: 2Gi
EOF

# Step 3: Create a VPA in "Off" mode first to see recommendations
cat <<'EOF' | kubectl apply -f -
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Off"     # Just recommend, don't act
  resourcePolicy:
    containerPolicies:
      - containerName: myapp
        minAllowed:
          cpu: 50m
          memory: 64Mi
        maxAllowed:
          cpu: "4"
          memory: 4Gi
EOF

# Step 4: Wait for recommendations (takes ~5 minutes of metric collection)
kubectl get vpa myapp-vpa -o yaml

# Expected output (after collecting metrics):
# status:
#   recommendation:
#     containerRecommendations:
#       - containerName: myapp
#         lowerBound:
#           cpu: 100m
#           memory: 200Mi
#         target:              # ← This is what VPA recommends
#           cpu: 200m
#           memory: 300Mi
#         uncappedTarget:
#           cpu: 200m
#           memory: 300Mi
#         upperBound:
#           cpu: 500m
#           memory: 600Mi

# Step 5: Switch to "Auto" mode to apply recommendations
kubectl patch vpa myapp-vpa --type='json' \
  -p='[{"op": "replace", "path": "/spec/updatePolicy/updateMode", "value": "Auto"}]'

# The VPA will now evict pods one at a time and recreate them with
# the recommended resource requests (200m CPU, 300Mi memory).
```

### 37.2.6 VPA + HPA: Can They Coexist?

**VPA and HPA should NOT both manage the same metric** (e.g., CPU). They will
fight — HPA adds pods when CPU is high, VPA increases CPU requests which makes
utilization appear lower, HPA removes pods, etc.

**Safe combinations:**
- HPA on custom metrics (requests per second) + VPA on CPU/memory
- HPA on CPU + VPA in "Off" mode (recommendation only)
- Use Multidimensional Pod Autoscaler (MPA) which coordinates both

```yaml
# Safe: HPA scales on custom metric, VPA right-sizes resources
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Pods
      pods:
        metric:
          name: myapp_http_requests_per_second
        target:
          type: AverageValue
          averageValue: "500"
---
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: myapp-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
      - containerName: myapp
        controlledResources: ["cpu", "memory"]
        controlledValues: RequestsOnly
```

---

## 37.3 Cluster Autoscaler

### 37.3.1 How Cluster Autoscaler Works

The Cluster Autoscaler (CA) operates at the **node level**. While HPA/VPA scale
pods, the CA scales the underlying infrastructure.

**Scale-up trigger:** A pod is Pending because no node has enough resources.

```
Pod (requests: 2 CPU) → Pending (Unschedulable)
    ↓
Cluster Autoscaler detects pending pod
    ↓
CA simulates adding a node from each node group
    ↓
CA selects the best node group and requests a new node
    ↓
Cloud provider provisions the node
    ↓
Pod is scheduled on the new node
```

**Scale-down trigger:** A node is underutilized (< 50% utilization) for 10 minutes.

```
Node utilization < 50% for 10 minutes
    ↓
CA checks if all pods can be moved to other nodes
    ↓
CA checks for PodDisruptionBudgets
    ↓
CA drains the node (evicts pods)
    ↓
CA requests node deletion from cloud provider
```

### 37.3.2 Expanders

When multiple node groups can accommodate a pending pod, the CA uses an
**expander** strategy to choose:

| Expander    | Strategy                                              |
|-------------|-------------------------------------------------------|
| random      | Random node group                                     |
| most-pods   | Node group that can schedule the most pending pods    |
| least-waste | Node group that wastes the least resources after scheduling |
| priority    | User-defined priority order                           |
| price       | Cheapest node group (cloud-specific)                  |

```yaml
# Cluster Autoscaler deployment with priority expander
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
        - name: cluster-autoscaler
          image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.28.0
          command:
            - ./cluster-autoscaler
            - --v=4
            - --cloud-provider=aws
            - --expander=priority    # Use priority expander
            - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/my-cluster
            - --balance-similar-node-groups
            - --skip-nodes-with-system-pods=false
            - --scale-down-enabled=true
            - --scale-down-delay-after-add=10m
            - --scale-down-delay-after-delete=1m
            - --scale-down-unneeded-time=10m
            - --scale-down-utilization-threshold=0.5
            - --max-graceful-termination-sec=600
```

### 37.3.3 Annotations for Controlling Behavior

```yaml
# Prevent scale-down of specific nodes
# (e.g., nodes with local persistent storage)
apiVersion: v1
kind: Node
metadata:
  annotations:
    cluster-autoscaler.kubernetes.io/scale-down-disabled: "true"

---
# Prevent a pod from blocking scale-down
# (tells CA this pod is safe to evict)
apiVersion: v1
kind: Pod
metadata:
  annotations:
    cluster-autoscaler.kubernetes.io/safe-to-evict: "true"
```

### 37.3.4 Hands-On: Cluster Autoscaler (Conceptual with kind)

While the Cluster Autoscaler requires a cloud provider to actually add/remove
nodes, we can understand its behavior conceptually:

```bash
# Simulate scale-up scenario
# Deploy pods that exceed current cluster capacity
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: resource-hungry
spec:
  replicas: 50    # Many replicas requesting significant resources
  selector:
    matchLabels:
      app: resource-hungry
  template:
    metadata:
      labels:
        app: resource-hungry
    spec:
      containers:
        - name: app
          image: nginx
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
EOF

# Check for pending pods
kubectl get pods --field-selector=status.phase=Pending

# In a real cluster with CA, these pending pods would trigger
# node provisioning. Check CA logs:
kubectl -n kube-system logs -l app=cluster-autoscaler --tail=50

# Common CA log messages:
# "Scale-up: setting group ... size to ..."
# "Pod ... is unschedulable"
# "ScaleDown: node ... is unneeded since ..."
```

### 37.3.5 Node Provisioning Strategies

For cloud environments, combine CA with **node pools** (or Auto Scaling Groups)
optimized for different workloads:

```yaml
# AWS EKS: Multiple managed node groups
# General purpose: m5.xlarge (4 CPU, 16GB)
# CPU-optimized: c5.2xlarge (8 CPU, 16GB)
# Memory-optimized: r5.xlarge (4 CPU, 32GB)
# GPU: p3.2xlarge (8 CPU, 61GB, 1 V100)

# Priority expander configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-priority-expander
  namespace: kube-system
data:
  priorities: |-
    10:
      - .*spot.*           # Prefer spot instances (cheapest)
    20:
      - .*general.*        # Then general purpose
    30:
      - .*cpu-optimized.*  # Then CPU-optimized
    40:
      - .*memory-optimized.*  # Then memory-optimized (most expensive)
```

---

## 37.4 KEDA (Kubernetes Event-Driven Autoscaling)

### 37.4.1 What KEDA Does

KEDA extends Kubernetes autoscaling beyond what HPA supports natively. While HPA
requires metrics to go through the Kubernetes metrics APIs, KEDA can scale based
on **event sources** directly:

- Message queue depth (Kafka, RabbitMQ, SQS, Azure Service Bus)
- Database query results
- Prometheus metrics
- Cron schedules
- HTTP request rate
- Custom sources

**Key capability:** KEDA can scale **to and from zero**. HPA's minimum is 1 replica,
but KEDA can scale to 0 when there's no work, saving resources completely.

### 37.4.2 Architecture

```
┌──────────────────────────────────────────────┐
│                    KEDA                       │
│                                               │
│  ┌─────────────────┐  ┌────────────────────┐ │
│  │   KEDA Operator  │  │  Metrics Server    │ │
│  │                  │  │  (keda-metrics-    │ │
│  │  Watches         │  │   apiserver)       │ │
│  │  ScaledObjects   │  │                    │ │
│  │  and ScaledJobs  │  │  Serves metrics    │ │
│  │                  │  │  to HPA via        │ │
│  │  Creates/manages │  │  external metrics  │ │
│  │  HPA resources   │  │  API               │ │
│  └────────┬─────────┘  └─────────┬──────────┘ │
│           │                      │             │
│           ▼                      ▼             │
│  ┌─────────────────────────────────────────┐  │
│  │              Scalers                     │  │
│  │  kafka | rabbitmq | prometheus | cron    │  │
│  │  http  | redis    | aws-sqs   | custom  │  │
│  └─────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

**How it works:**
1. You create a `ScaledObject` that defines the scaling rules
2. KEDA Operator creates and manages an HPA behind the scenes
3. KEDA Metrics Server queries the event source (Kafka, Prometheus, etc.)
4. The HPA uses KEDA's metrics to scale the target Deployment
5. When there are zero events, KEDA scales to 0 (bypasses HPA minimum)

### 37.4.3 Installing KEDA

```bash
# Install KEDA with Helm
helm repo add kedacore https://kedacore.github.io/charts
helm repo update

helm install keda kedacore/keda \
  --namespace keda \
  --create-namespace \
  --set metricsServer.enabled=true

# Verify
kubectl -n keda get pods
```

### 37.4.4 ScaledObject

```yaml
# scaled-object.yaml
# The ScaledObject is KEDA's primary CRD. It defines:
#   - What to scale (targetRef)
#   - When to scale (triggers)
#   - How to scale (scaling parameters)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: myapp-scaledobject
  namespace: default
spec:
  # The Deployment to scale.
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-worker

  # Scaling parameters.
  minReplicaCount: 0       # KEDA's killer feature: scale to zero
  maxReplicaCount: 100
  pollingInterval: 15      # How often KEDA checks the event source (seconds)
  cooldownPeriod: 300      # Seconds to wait before scaling to zero
  idleReplicaCount: 0      # Replicas when no events (0 = scale to zero)

  # Fallback configuration — what to do if the scaler can't reach the
  # event source (e.g., Kafka is down).
  fallback:
    failureThreshold: 3
    replicas: 5            # Maintain 5 replicas if scaler fails

  # Advanced HPA configuration passed through to the generated HPA.
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleUp:
          stabilizationWindowSeconds: 30
          policies:
            - type: Percent
              value: 100
              periodSeconds: 15
        scaleDown:
          stabilizationWindowSeconds: 300
          policies:
            - type: Pods
              value: 5
              periodSeconds: 60

  # Triggers define the event sources that drive scaling.
  triggers:
    # Scale based on Kafka consumer lag.
    - type: kafka
      metadata:
        bootstrapServers: kafka-bootstrap.kafka:9092
        consumerGroup: myapp-group
        topic: orders
        lagThreshold: "50"       # Scale up when lag > 50 messages per partition
        activationLagThreshold: "10"  # Start from zero when lag > 10
```

### 37.4.5 ScaledJob

For batch workloads, KEDA can create Jobs instead of scaling a Deployment:

```yaml
# scaled-job.yaml
# ScaledJob creates Kubernetes Jobs based on event source activity.
# Each job processes a batch of events and terminates.
# This is ideal for:
#   - Image processing pipelines
#   - Report generation
#   - Data transformation tasks
apiVersion: keda.sh/v1alpha1
kind: ScaledJob
metadata:
  name: image-processor
spec:
  jobTargetRef:
    template:
      spec:
        containers:
          - name: processor
            image: myregistry/image-processor:v1.0.0
            env:
              - name: QUEUE_URL
                value: "amqp://rabbitmq:5672"
        restartPolicy: Never
    backoffLimit: 3
  pollingInterval: 10
  minReplicaCount: 0
  maxReplicaCount: 50
  successfulJobsHistoryLimit: 5
  failedJobsHistoryLimit: 3
  scalingStrategy:
    strategy: "accurate"    # Create exactly as many jobs as needed
  triggers:
    - type: rabbitmq
      metadata:
        host: "amqp://guest:guest@rabbitmq:5672"
        queueName: images
        queueLength: "5"     # Each job handles 5 messages
```

### 37.4.6 Hands-On: KEDA with Prometheus Scaler

This is particularly powerful because it lets you scale based on **any Prometheus
metric** without the complexity of prometheus-adapter.

```yaml
# keda-prometheus.yaml
# Scales based on the HTTP request rate from Prometheus.
# KEDA queries Prometheus directly — no adapter needed.
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: myapp-prometheus-scaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicaCount: 1
  maxReplicaCount: 50
  pollingInterval: 15
  cooldownPeriod: 300
  triggers:
    - type: prometheus
      metadata:
        # Prometheus server URL
        serverAddress: http://kube-prometheus-stack-prometheus.monitoring:9090
        # PromQL query that returns a single scalar value.
        # This query calculates the total request rate.
        query: |
          sum(rate(myapp_http_requests_total{namespace="default"}[2m]))
        # Each pod should handle 500 RPS.
        # KEDA divides the query result by this threshold to get desired replicas.
        threshold: "500"
        # Activation threshold: minimum value to scale from zero.
        activationThreshold: "10"
        # How KEDA interprets the query result.
        metricType: AverageValue
```

```yaml
# keda-prometheus-error-rate.yaml
# A more advanced example: scale based on error rate.
# When error rate exceeds 5%, add more replicas to reduce per-pod load.
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: myapp-error-scaler
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicaCount: 3     # Always maintain at least 3 replicas
  maxReplicaCount: 30
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://kube-prometheus-stack-prometheus.monitoring:9090
        query: |
          sum(rate(myapp_http_requests_total{status=~"5..", namespace="default"}[5m]))
          / sum(rate(myapp_http_requests_total{namespace="default"}[5m]))
          * 100
        threshold: "5"     # Scale when error rate > 5%
        activationThreshold: "1"
```

### 37.4.7 Hands-On: KEDA with HTTP Trigger

KEDA's HTTP add-on scales based on the number of concurrent HTTP requests:

```bash
# Install KEDA HTTP Add-on
helm install http-add-on kedacore/keda-add-ons-http \
  --namespace keda
```

```yaml
# keda-http.yaml
# HTTPScaledObject provides true scale-to-zero for HTTP workloads.
# KEDA's HTTP proxy sits in front of your app and counts pending requests.
apiVersion: http.keda.sh/v1alpha1
kind: HTTPScaledObject
metadata:
  name: myapp-http
spec:
  hosts:
    - myapp.example.com
  scaleTargetRef:
    deployment: myapp
    service: myapp
    port: 8080
  replicas:
    min: 0
    max: 50
  scalingMetric:
    requestRate:
      targetValue: 100          # Target 100 RPS per pod
      granularity: 1s
      window: 1m
  scaledownPeriod: 300           # Wait 5 minutes before scaling to zero
```

### 37.4.8 Common KEDA Scalers

```yaml
# --- Kafka Scaler ---
# Scales based on consumer group lag.
triggers:
  - type: kafka
    metadata:
      bootstrapServers: kafka:9092
      consumerGroup: my-group
      topic: events
      lagThreshold: "100"

# --- RabbitMQ Scaler ---
# Scales based on queue depth.
triggers:
  - type: rabbitmq
    metadata:
      host: "amqp://guest:guest@rabbitmq:5672"
      queueName: tasks
      queueLength: "20"
      mode: QueueLength

# --- Cron Scaler ---
# Pre-scales based on a schedule (e.g., business hours).
triggers:
  - type: cron
    metadata:
      timezone: America/New_York
      start: "0 8 * * 1-5"      # 8 AM weekdays
      end: "0 18 * * 1-5"       # 6 PM weekdays
      desiredReplicas: "10"

# --- AWS SQS Scaler ---
triggers:
  - type: aws-sqs-queue
    metadata:
      queueURL: https://sqs.us-east-1.amazonaws.com/123456789/my-queue
      queueLength: "5"
      awsRegion: us-east-1
    authenticationRef:
      name: aws-credentials

# --- Redis Scaler ---
triggers:
  - type: redis
    metadata:
      address: redis:6379
      listName: work-items
      listLength: "10"

# --- MySQL Scaler ---
triggers:
  - type: mysql
    metadata:
      connectionStringFromEnv: MYSQL_CONN
      query: "SELECT COUNT(*) FROM jobs WHERE status='pending'"
      queryValue: "50"
```

---

## 37.5 Autoscaling Best Practices

### 37.5.1 Right-Size First, Then Autoscale

Before configuring autoscaling, ensure your pods have correct resource requests:

```
1. Deploy with VPA in "Off" mode
2. Run your application under realistic load for 24-48 hours
3. Review VPA recommendations
4. Set resource requests to VPA's "target" values
5. NOW configure HPA
```

If resource requests are wrong, the HPA will make wrong decisions:
- Requests too low → CPU utilization appears high → HPA scales up too aggressively
- Requests too high → CPU utilization appears low → HPA never scales up

### 37.5.2 Set Appropriate Resource Requests

```yaml
# GOOD: Requests reflect actual baseline usage
resources:
  requests:
    cpu: 200m      # Based on VPA recommendation
    memory: 256Mi
  limits:
    cpu: "1"       # Allow burst up to 1 CPU
    memory: 512Mi  # Hard limit to prevent OOM on node

# BAD: Requests are a guess
resources:
  requests:
    cpu: "1"       # Way too high — HPA thinks pod is underutilized
    memory: 1Gi
```

### 37.5.3 Avoid Thrashing with Stabilization Windows

```yaml
# Anti-thrashing configuration
behavior:
  scaleUp:
    stabilizationWindowSeconds: 60    # Wait 1 min after scale-up
    policies:
      - type: Percent
        value: 100
        periodSeconds: 60
  scaleDown:
    stabilizationWindowSeconds: 300   # Wait 5 min after scale-down
    policies:
      - type: Pods
        value: 1
        periodSeconds: 120            # Remove at most 1 pod every 2 min
```

### 37.5.4 PodDisruptionBudgets with Autoscaling

**Always** configure PDBs for autoscaled workloads to prevent service disruptions
during scale-down or VPA-triggered evictions:

```yaml
# pdb.yaml
# Ensures at least 2 pods are always available during scaling events.
# This prevents the autoscaler from removing all pods at once.
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: myapp-pdb
spec:
  # At least 2 pods must always be available.
  # The autoscaler/VPA/cluster-autoscaler must respect this.
  minAvailable: 2
  # Alternative: allow at most 1 pod to be unavailable
  # maxUnavailable: 1
  selector:
    matchLabels:
      app: myapp
```

### 37.5.5 Graceful Shutdown for Autoscaled Pods

When pods are removed by autoscaling, they receive a SIGTERM. Your application
must handle this gracefully:

```go
// graceful_shutdown.go
//
// Demonstrates graceful shutdown handling for autoscaled workloads.
// When the HPA scales down or VPA evicts a pod, Kubernetes sends
// SIGTERM to the process. This code ensures:
//
//   - In-flight HTTP requests are completed (not dropped)
//   - Background workers finish their current task
//   - Database connections are closed cleanly
//   - The process exits within the terminationGracePeriodSeconds
//
// This is critical for autoscaled workloads because pods are
// frequently created and destroyed.
package main

import (
	"context"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"
)

// runGracefulServer starts an HTTP server that shuts down cleanly
// when SIGTERM is received. This is the pattern every Kubernetes
// application should use.
//
// The key is http.Server.Shutdown(), which:
//  1. Stops accepting new connections
//  2. Waits for in-flight requests to complete
//  3. Returns when all connections are closed or the context expires
//
// The shutdownTimeout should be LESS than the pod's
// terminationGracePeriodSeconds (default 30s) to ensure cleanup
// completes before Kubernetes sends SIGKILL.
func runGracefulServer() {
	logger := slog.Default()

	mux := http.NewServeMux()
	mux.HandleFunc("/api/orders", handleOrders)
	mux.HandleFunc("/healthz", handleHealth)

	server := &http.Server{
		Addr:         ":8080",
		Handler:      mux,
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 30 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	// Start the server in a goroutine.
	go func() {
		logger.Info("server starting", "addr", server.Addr)
		if err := server.ListenAndServe(); err != http.ErrServerClosed {
			logger.Error("server error", "error", err)
			os.Exit(1)
		}
	}()

	// Wait for SIGTERM (from Kubernetes) or SIGINT (from Ctrl+C).
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)
	sig := <-sigCh
	logger.Info("received signal, starting graceful shutdown", "signal", sig)

	// Create a context with timeout for the shutdown.
	// This should be less than terminationGracePeriodSeconds (default 30s).
	ctx, cancel := context.WithTimeout(context.Background(), 25*time.Second)
	defer cancel()

	// Shut down the HTTP server — waits for in-flight requests.
	if err := server.Shutdown(ctx); err != nil {
		logger.Error("server shutdown error", "error", err)
	}

	// Perform additional cleanup (close DB connections, flush buffers, etc.)
	logger.Info("cleaning up resources")
	// db.Close()
	// cache.Close()
	// telemetryShutdown(ctx)

	logger.Info("shutdown complete")
}
```

### 37.5.6 Monitoring Autoscaler Behavior

Use these PromQL queries to monitor your autoscalers:

```promql
# --- HPA Metrics ---

# Current vs desired replicas
kube_horizontalpodautoscaler_status_current_replicas{horizontalpodautoscaler="myapp-hpa"}
kube_horizontalpodautoscaler_status_desired_replicas{horizontalpodautoscaler="myapp-hpa"}

# HPA at maximum capacity (may need to increase maxReplicas)
kube_horizontalpodautoscaler_status_current_replicas
== kube_horizontalpodautoscaler_spec_max_replicas

# HPA condition (ScalingActive, AbleToScale, ScalingLimited)
kube_horizontalpodautoscaler_status_condition{condition="ScalingLimited",status="true"}

# --- VPA Metrics ---

# VPA recommendations vs actual usage
vpa_recommender_vpa_recommendation_cpu_target
vpa_recommender_vpa_recommendation_memory_target

# --- Cluster Autoscaler Metrics ---

# Unschedulable pods (triggers scale-up)
cluster_autoscaler_unschedulable_pods_count

# Nodes added/removed
cluster_autoscaler_scaled_up_nodes_total
cluster_autoscaler_scaled_down_nodes_total

# --- KEDA Metrics ---

# Scaler activity
keda_scaler_metrics_value
keda_scaler_active
keda_scaled_object_paused
```

### 37.5.7 Autoscaling Decision Matrix

| Scenario                           | Solution                    |
|------------------------------------|-----------------------------|
| Web app with variable HTTP traffic | HPA (CPU) + PDB            |
| Queue worker processing messages   | KEDA (queue scaler)         |
| Batch job triggered by events      | KEDA ScaledJob              |
| Unknown resource requirements      | VPA (Off mode) → tune → HPA|
| Predictable daily traffic pattern  | KEDA (cron) + HPA (CPU)    |
| Scale to zero when idle            | KEDA                        |
| Right-size pod resources           | VPA                         |
| Not enough cluster capacity        | Cluster Autoscaler          |

---

## 37.6 Summary

| Autoscaler         | Scales What   | Scale to Zero | Based On                      |
|--------------------|---------------|---------------|-------------------------------|
| HPA                | Pod replicas  | No (min=1)    | CPU, memory, custom metrics   |
| VPA                | Pod resources | No            | Historical usage              |
| Cluster Autoscaler | Nodes         | Yes           | Pending pods, utilization     |
| KEDA               | Pods + Jobs   | Yes           | Event sources, any metric     |

**Key takeaways:**

1. **Start with right-sizing** — use VPA in Off mode to understand your app's needs.
2. **HPA for most workloads** — CPU-based is simplest; add custom metrics for precision.
3. **Use stabilization windows** — prevent thrashing with conservative scale-down.
4. **KEDA for event-driven workloads** — especially when scale-to-zero matters.
5. **Always set PDBs** — protect availability during scaling events.
6. **Handle graceful shutdown** — your pods will be frequently terminated.
7. **Monitor the autoscalers** — they can misbehave; watch their metrics.

In the next chapter, we'll package our applications with **Helm and Kustomize**,
and deploy them with **GitOps** using ArgoCD — where the autoscaling
configurations we built here become version-controlled, declarative resources
deployed from Git.
