# Chapter 36: Observability — Metrics, Logs, Traces

## Introduction

Observability is the ability to understand the internal state of a system by examining
its external outputs. In modern distributed systems — especially those running on
Kubernetes — observability is not a luxury but a necessity. When a user reports that
"the checkout page is slow," you need to answer: *Which service is slow? Is it the
database? A downstream API? A resource contention issue?* Observability gives you
the tools to answer these questions without deploying new code or adding ad-hoc
debug statements.

This chapter covers the **three pillars of observability** — metrics, logs, and
traces — along with the tools and practices that make them actionable in production
Kubernetes environments. We will instrument Go applications from scratch, deploy
a full monitoring stack, write queries to surface issues, and correlate signals
across all three pillars using OpenTelemetry.

---

## 36.1 The Three Pillars of Observability

### 36.1.1 Metrics

Metrics are **numeric measurements** collected over time. They are the most
efficient observability signal because they are highly compressible and cheap to
store. A single metric time series — say, `http_requests_total` — might consume
only a few bytes per sample, yet it tells you the request rate for your entire
service.

**Types of metrics:**

| Type      | Description                              | Example                          |
|-----------|------------------------------------------|----------------------------------|
| Counter   | Monotonically increasing value           | Total HTTP requests served       |
| Gauge     | Value that goes up and down              | Current number of active connections |
| Histogram | Distribution of values in buckets        | Request latency distribution     |
| Summary   | Similar to histogram, with quantiles     | 95th-percentile latency          |

Metrics answer questions like:
- What is the request rate?
- What is the error rate?
- What is the 99th-percentile latency?
- How much memory is the process using?

### 36.1.2 Logs

Logs are **discrete events** recorded by a system. They are the most familiar
observability signal — every programmer has used `fmt.Println` for debugging.
In production systems, logs are structured (typically JSON), timestamped, and
enriched with contextual metadata (trace IDs, request IDs, user IDs).

**Log levels:**

| Level | Usage                                                      |
|-------|------------------------------------------------------------|
| DEBUG | Verbose diagnostic information, typically disabled in prod |
| INFO  | Normal operational events (server started, request handled)|
| WARN  | Unexpected but recoverable conditions                      |
| ERROR | Failures that need attention                               |
| FATAL | Unrecoverable errors that terminate the process            |

Logs answer questions like:
- What happened right before the crash?
- What was the exact error message?
- What input caused the failure?

### 36.1.3 Traces

Traces are **causal chains** that follow a request as it flows through distributed
services. A trace consists of **spans** — each span represents a unit of work
(an HTTP call, a database query, a function execution). Spans are connected by
parent-child relationships, forming a tree that shows the entire lifecycle of a
request.

Traces answer questions like:
- Which service in the chain is slow?
- How many downstream calls does a single request make?
- Where does the request spend most of its time?

### 36.1.4 How the Pillars Work Together

No single pillar is sufficient:

```
Metrics: "Error rate spiked to 5% at 14:03"
    ↓
Logs:    "Error: connection refused to database at 14:03:12"
    ↓
Traces:  "Request abc-123 spent 30s waiting for DB connection pool"
```

The ideal workflow:
1. **Metrics** alert you that something is wrong (error rate spike).
2. **Logs** tell you what went wrong (specific error messages).
3. **Traces** tell you where and why (which service, which call).

---

## 36.2 Prometheus

### 36.2.1 Architecture

Prometheus is the de facto standard for metrics in the Kubernetes ecosystem. It
was the second project to graduate from the CNCF (after Kubernetes itself).

**Core components:**

```text
┌─────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  Prometheus      │────▶│  Alertmanager    │────▶│  PagerDuty/Slack │
│  Server          │     │                  │     │  Email/Webhook   │
│                  │     └──────────────────┘     └──────────────────┘
│  - Scrapes       │
│    targets       │     ┌──────────────────┐
│  - Stores TSDB   │◀────│  Pushgateway     │◀─── Short-lived jobs
│  - Evaluates     │     └──────────────────┘
│    rules         │
│  - Serves PromQL │     ┌──────────────────┐
│    queries       │◀────│  Service         │
└────────┬────────┘     │  Discovery       │
         │               │  (K8s, DNS,      │
         │               │   file, consul)  │
         ▼               └──────────────────┘
┌─────────────────┐
│  TSDB (local)   │
│  or Remote Write │──▶  Thanos / Cortex / Mimir
└─────────────────┘
```

**Prometheus Server:** The heart of the system. It scrapes metrics from targets
at configured intervals (typically 15–30 seconds), stores them in a local
time-series database (TSDB), evaluates alerting rules, and serves PromQL queries.

**Exporters:** Agents that expose metrics in Prometheus format from systems that
don't natively support it. Examples:
- `node-exporter` — Linux host metrics (CPU, memory, disk, network)
- `blackbox-exporter` — Probes endpoints (HTTP, TCP, ICMP)
- `mysqld-exporter` — MySQL database metrics
- `redis-exporter` — Redis metrics

**Alertmanager:** Handles alerts from Prometheus. It deduplicates, groups, routes,
and sends notifications to receivers (Slack, PagerDuty, email, webhooks). It also
supports silencing and inhibition rules.

**Pushgateway:** Allows short-lived batch jobs to push metrics. These jobs may
terminate before Prometheus can scrape them. The Pushgateway acts as a metrics
cache that Prometheus scrapes.

### 36.2.2 Pull vs. Push Model

Prometheus uses a **pull model**: the Prometheus server scrapes metrics from targets.
This is fundamentally different from push-based systems (like StatsD or Datadog Agent).

**Advantages of pull:**
- Prometheus controls the scrape interval
- Easy to detect if a target is down (scrape fails)
- No need for targets to know where Prometheus is
- Targets can be discovered dynamically

**When to use push (Pushgateway):**
- Batch jobs that run to completion before scrape
- Jobs behind firewalls where Prometheus can't reach

### 36.2.3 PromQL Basics

PromQL (Prometheus Query Language) is a powerful functional query language for
selecting, aggregating, and transforming time series data.

#### Instant Vectors

An instant vector selects a set of time series, each with a single sample at the
current time:

```promql
# All HTTP request counters
http_requests_total

# Filter by label
http_requests_total{method="GET", status="200"}

# Regex match
http_requests_total{method=~"GET|POST"}

# Negative match
http_requests_total{status!~"2.."}
```

#### Range Vectors

A range vector selects a set of time series over a time range. Required by
functions like `rate()` and `increase()`:

```promql
# All samples in the last 5 minutes
http_requests_total[5m]

# Rate of requests per second over the last 5 minutes
rate(http_requests_total[5m])

# Increase in the last hour
increase(http_requests_total[1h])
```

#### Aggregations

```promql
# Total requests across all instances
sum(rate(http_requests_total[5m]))

# Requests per method
sum by (method) (rate(http_requests_total[5m]))

# Average latency (from histogram)
histogram_quantile(0.99, sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# Top 5 endpoints by request rate
topk(5, sum by (handler) (rate(http_requests_total[5m])))

# Percentage of 5xx errors
sum(rate(http_requests_total{status=~"5.."}[5m]))
/
sum(rate(http_requests_total[5m]))
* 100
```

#### Common PromQL Patterns

```promql
# --- RED Method (Rate, Errors, Duration) ---

# Request rate
sum(rate(http_requests_total[5m]))

# Error rate (as percentage)
sum(rate(http_requests_total{status=~"5.."}[5m]))
/ sum(rate(http_requests_total[5m])) * 100

# Duration (p99)
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m]))
)

# --- USE Method (Utilization, Saturation, Errors) ---

# CPU utilization
1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m]))

# Memory utilization
1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# Disk saturation (IO utilization)
rate(node_disk_io_time_seconds_total[5m])
```

### 36.2.4 ServiceMonitor and PodMonitor CRDs

The **prometheus-operator** extends Kubernetes with CRDs that declaratively
configure Prometheus scrape targets.

**ServiceMonitor** — Discovers targets via Kubernetes Services:

```yaml
# servicemonitor.yaml
# Tells Prometheus to scrape all pods behind the "myapp" Service
# on port "metrics" at the /metrics endpoint.
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  namespace: monitoring
  labels:
    release: kube-prometheus-stack   # Must match Prometheus selector
spec:
  # Namespace selector — which namespaces to look in
  namespaceSelector:
    matchNames:
      - default
      - production
  # Service selector — which Services to discover
  selector:
    matchLabels:
      app: myapp
  # Endpoint configuration — how to scrape
  endpoints:
    - port: metrics          # Name of the Service port
      path: /metrics         # HTTP path to scrape
      interval: 15s          # Scrape interval
      scrapeTimeout: 10s     # Timeout for each scrape
      # Optional: relabeling to add/modify labels
      metricRelabelings:
        - sourceLabels: [__name__]
          regex: "go_.*"
          action: drop        # Drop Go runtime metrics to save storage
```

**PodMonitor** — Discovers targets directly from Pods (no Service needed):

```yaml
# podmonitor.yaml
# Useful for sidecar containers or pods without a Service.
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: myapp-pods
  namespace: monitoring
spec:
  namespaceSelector:
    matchNames:
      - default
  selector:
    matchLabels:
      app: myapp
  podMetricsEndpoints:
    - port: metrics
      path: /metrics
      interval: 30s
```

### 36.2.5 Hands-On: Deploying Prometheus with kube-prometheus-stack

The `kube-prometheus-stack` Helm chart deploys a complete monitoring stack:
Prometheus, Alertmanager, Grafana, node-exporter, kube-state-metrics, and the
prometheus-operator.

```bash
# Add the Prometheus community Helm repo
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts
helm repo update

# Create a values file for customization
cat > prom-values.yaml <<'EOF'
# Prometheus configuration
prometheus:
  prometheusSpec:
    # Retain metrics for 15 days
    retention: 15d
    # Resource requests/limits for the Prometheus server
    resources:
      requests:
        memory: 2Gi
        cpu: 500m
      limits:
        memory: 4Gi
        cpu: "2"
    # Storage configuration — use a PersistentVolumeClaim
    storageSpec:
      volumeClaimTemplate:
        spec:
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
    # Select ALL ServiceMonitors across all namespaces
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false

# Grafana configuration
grafana:
  adminPassword: "changeme-in-production"
  # Provision dashboards from ConfigMaps
  sidecar:
    dashboards:
      enabled: true
      searchNamespace: ALL

# Alertmanager configuration
alertmanager:
  alertmanagerSpec:
    resources:
      requests:
        memory: 256Mi
        cpu: 100m

# Node exporter for host-level metrics
nodeExporter:
  enabled: true

# kube-state-metrics for Kubernetes object metrics
kubeStateMetrics:
  enabled: true
EOF

# Install the stack
helm install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --create-namespace \
  -f prom-values.yaml

# Verify all pods are running
kubectl -n monitoring get pods

# Access Prometheus UI (port-forward for local dev)
kubectl -n monitoring port-forward svc/kube-prometheus-stack-prometheus 9090:9090 &

# Access Grafana UI
kubectl -n monitoring port-forward svc/kube-prometheus-stack-grafana 3000:80 &
```

### 36.2.6 Hands-On: Instrumenting a Go HTTP Server

This is a complete, production-ready Go application instrumented with Prometheus
metrics. We define custom counters, histograms, and gauges, then expose them on
a `/metrics` endpoint.

```go
// main.go
//
// A fully instrumented Go HTTP server demonstrating all major Prometheus
// metric types: counters, gauges, and histograms. This server exposes
// custom application metrics alongside Go runtime metrics on the
// /metrics endpoint.
//
// Key design decisions:
//   - Metrics are registered in an init() function for clarity, but
//     production code may use a custom registry to avoid polluting the
//     global default registry.
//   - Histogram buckets are chosen to match expected latency ranges.
//     The default buckets (DefBuckets) work for many HTTP services.
//   - Labels are kept low-cardinality to prevent metric explosion.
//     Never use user IDs, request IDs, or other high-cardinality
//     values as label values.
package main

import (
	"fmt"
	"log"
	"math/rand"
	"net/http"
	"time"

	"github.com/prometheus/client_golang/prometheus"
	"github.com/prometheus/client_golang/prometheus/promauto"
	"github.com/prometheus/client_golang/prometheus/promhttp"
)

// httpRequestsTotal counts the total number of HTTP requests processed,
// partitioned by HTTP method, handler path, and response status code.
// This is the primary metric for calculating request rate and error rate.
//
// Example PromQL:
//
//	rate(http_requests_total{handler="/api/orders"}[5m])
var httpRequestsTotal = promauto.NewCounterVec(
	prometheus.CounterOpts{
		Namespace: "myapp",
		Subsystem: "http",
		Name:      "requests_total",
		Help:      "Total number of HTTP requests processed, partitioned by method, handler, and status.",
	},
	[]string{"method", "handler", "status"},
)

// httpRequestDuration records the latency distribution of HTTP requests
// in seconds. The buckets are chosen to give good resolution for a
// typical web application where most requests complete in under 1 second.
//
// Example PromQL (99th-percentile latency):
//
//	histogram_quantile(0.99,
	//	  sum by (le, handler) (rate(myapp_http_request_duration_seconds_bucket[5m]))
	//	)
var httpRequestDuration = promauto.NewHistogramVec(
	prometheus.HistogramOpts{
		Namespace: "myapp",
		Subsystem: "http",
		Name:      "request_duration_seconds",
		Help:      "Histogram of HTTP request latencies in seconds.",
		// Custom buckets: 5ms, 10ms, 25ms, 50ms, 100ms, 250ms, 500ms, 1s, 2.5s, 5s, 10s
		Buckets: []float64{0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10},
	},
	[]string{"method", "handler"},
)

// httpRequestsInFlight tracks the number of HTTP requests currently being
// processed. This is a gauge because the value goes up and down.
// A sustained high value indicates saturation.
//
// Example PromQL:
//
//	myapp_http_requests_in_flight
var httpRequestsInFlight = promauto.NewGauge(
	prometheus.GaugeOpts{
		Namespace: "myapp",
		Subsystem: "http",
		Name:      "requests_in_flight",
		Help:      "Current number of HTTP requests being processed.",
	},
)

// httpResponseSizeBytes records the distribution of HTTP response sizes
// in bytes. Useful for capacity planning and detecting anomalous responses.
var httpResponseSizeBytes = promauto.NewHistogramVec(
	prometheus.HistogramOpts{
		Namespace: "myapp",
		Subsystem: "http",
		Name:      "response_size_bytes",
		Help:      "Histogram of HTTP response sizes in bytes.",
		Buckets:   prometheus.ExponentialBuckets(100, 10, 6), // 100, 1K, 10K, 100K, 1M, 10M
	},
	[]string{"method", "handler"},
)

// businessOrdersTotal tracks the total number of orders created,
// partitioned by order type. This is a business-level metric that
// helps correlate technical issues with business impact.
var businessOrdersTotal = promauto.NewCounterVec(
	prometheus.CounterOpts{
		Namespace: "myapp",
		Subsystem: "business",
		Name:      "orders_total",
		Help:      "Total number of orders created, partitioned by order type.",
	},
	[]string{"type"},
)

// instrumentHandler wraps an HTTP handler with Prometheus instrumentation.
// It records request count, duration, in-flight requests, and response size
// for every request.
//
// Parameters:
//   - handlerName: A short, low-cardinality label for this handler (e.g., "/api/orders").
//     Do NOT use the full URL path if it contains variable segments (e.g., user IDs).
//   - next: The actual HTTP handler to wrap.
//
// Returns an http.Handler that records metrics before delegating to next.
func instrumentHandler(handlerName string, next http.HandlerFunc) http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
			// Track in-flight requests with a gauge.
			httpRequestsInFlight.Inc()
			defer httpRequestsInFlight.Dec()

			// Start the timer for request duration.
			start := time.Now()

			// Wrap the ResponseWriter to capture the status code and response size.
			wrapped := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

			// Delegate to the actual handler.
			next.ServeHTTP(wrapped, r)

			// Record metrics after the handler completes.
			duration := time.Since(start).Seconds()
			status := fmt.Sprintf("%d", wrapped.statusCode)

			httpRequestsTotal.WithLabelValues(r.Method, handlerName, status).Inc()
			httpRequestDuration.WithLabelValues(r.Method, handlerName).Observe(duration)
			httpResponseSizeBytes.WithLabelValues(r.Method, handlerName).Observe(float64(wrapped.bytesWritten))
		})
}

// responseWriter wraps http.ResponseWriter to capture the status code
// and total bytes written. This is necessary because the standard
// http.ResponseWriter does not expose these values after WriteHeader/Write.
type responseWriter struct {
	http.ResponseWriter
	statusCode   int
	bytesWritten int
}

// WriteHeader captures the status code before delegating to the
// underlying ResponseWriter.
func (rw *responseWriter) WriteHeader(code int) {
	rw.statusCode = code
	rw.ResponseWriter.WriteHeader(code)
}

// Write captures the number of bytes written before delegating to the
// underlying ResponseWriter.
func (rw *responseWriter) Write(b []byte) (int, error) {
	n, err := rw.ResponseWriter.Write(b)
	rw.bytesWritten += n
	return n, err
}

// handleOrders simulates an order creation endpoint. It introduces
// random latency and occasional errors to demonstrate how metrics
// capture real-world behavior patterns.
func handleOrders(w http.ResponseWriter, r *http.Request) {
	// Simulate variable processing time (10ms to 500ms).
	delay := time.Duration(10+rand.Intn(490)) * time.Millisecond
	time.Sleep(delay)

	// Simulate occasional errors (roughly 5% of requests).
	if rand.Float64() < 0.05 {
		http.Error(w, `{"error": "internal server error"}`, http.StatusInternalServerError)
		return
	}

	// Record a business metric for successful orders.
	orderType := "standard"
	if rand.Float64() < 0.2 {
		orderType = "express"
	}
	businessOrdersTotal.WithLabelValues(orderType).Inc()

	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(http.StatusCreated)
	fmt.Fprintf(w, `{"order_id": "%d", "type": "%s"}`, rand.Int63(), orderType)
}

// handleHealth returns a simple health check response. This endpoint
// is typically NOT instrumented with detailed metrics to avoid noise,
// but we include it here for completeness.
func handleHealth(w http.ResponseWriter, r *http.Request) {
	w.Header().Set("Content-Type", "application/json")
	fmt.Fprint(w, `{"status": "healthy"}`)
}

func main() {
	mux := http.NewServeMux()

	// Register application endpoints with instrumentation.
	mux.Handle("/api/orders", instrumentHandler("/api/orders", handleOrders))
	mux.Handle("/healthz", instrumentHandler("/healthz", handleHealth))

	// Register the Prometheus metrics endpoint.
	// This exposes all registered metrics in Prometheus exposition format.
	mux.Handle("/metrics", promhttp.Handler())

	server := &http.Server{
		Addr:         ":8080",
		Handler:      mux,
		ReadTimeout:  10 * time.Second,
		WriteTimeout: 10 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	log.Printf("Starting server on :8080")
	log.Printf("Metrics available at http://localhost:8080/metrics")
	if err := server.ListenAndServe(); err != nil {
		log.Fatalf("Server failed: %v", err)
	}
}
```

**Kubernetes manifests for the instrumented app:**

```yaml
# deployment.yaml
# Deploys the instrumented Go application with proper resource
# limits, health checks, and a named port for ServiceMonitor discovery.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
      # Optional: Prometheus annotations for simple scraping
      # (use ServiceMonitor instead for production)
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/metrics"
    spec:
      containers:
        - name: myapp
          image: myregistry/myapp:v1.0.0
          ports:
            - name: http
              containerPort: 8080
            - name: metrics
              containerPort: 8080   # Same port; separate name for clarity
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          livenessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /healthz
              port: http
            initialDelaySeconds: 5
            periodSeconds: 5
---
# service.yaml
# Exposes the application. The named port "metrics" is referenced
# by the ServiceMonitor.
apiVersion: v1
kind: Service
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  selector:
    app: myapp
  ports:
    - name: http
      port: 80
      targetPort: http
    - name: metrics
      port: 8080
      targetPort: metrics
---
# servicemonitor.yaml
# Tells Prometheus (via the operator) to scrape our app's /metrics endpoint.
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: myapp
  labels:
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      app: myapp
  endpoints:
    - port: metrics
      path: /metrics
      interval: 15s
```

### 36.2.7 Hands-On: Common PromQL Queries for the Go App

Once the app is deployed and scraped, use these queries in the Prometheus UI or
Grafana:

```promql
# --- Request Rate ---
# Total requests per second across all instances
sum(rate(myapp_http_requests_total[5m]))

# Requests per second per handler
sum by (handler) (rate(myapp_http_requests_total[5m]))

# --- Error Rate ---
# Percentage of 5xx errors
sum(rate(myapp_http_requests_total{status=~"5.."}[5m]))
/ sum(rate(myapp_http_requests_total[5m])) * 100

# Error rate per handler
sum by (handler) (rate(myapp_http_requests_total{status=~"5.."}[5m]))
/ sum by (handler) (rate(myapp_http_requests_total[5m])) * 100

# --- Latency ---
# 50th percentile (median) latency
histogram_quantile(0.50,
  sum by (le) (rate(myapp_http_request_duration_seconds_bucket[5m]))
)

# 95th percentile latency per handler
histogram_quantile(0.95,
  sum by (le, handler) (rate(myapp_http_request_duration_seconds_bucket[5m]))
)

# 99th percentile latency
histogram_quantile(0.99,
  sum by (le) (rate(myapp_http_request_duration_seconds_bucket[5m]))
)

# --- Saturation ---
# Current in-flight requests
myapp_http_requests_in_flight

# --- Business Metrics ---
# Orders per second by type
sum by (type) (rate(myapp_business_orders_total[5m]))

# Total orders in the last 24 hours
sum(increase(myapp_business_orders_total[24h]))

# --- Go Runtime Metrics (auto-exposed by client_golang) ---
# Goroutine count
go_goroutines

# Memory usage
go_memstats_alloc_bytes

# GC pause duration (99th percentile)
histogram_quantile(0.99, sum by (le) (rate(go_gc_duration_seconds_bucket[5m])))
```

---

## 36.3 Grafana

### 36.3.1 Dashboard Design Principles

Grafana is the visualization layer for your observability stack. A well-designed
dashboard tells a story and guides the operator from high-level health to
detailed diagnostics.

**Dashboard hierarchy:**

```
Level 1: Service Overview
  ├── Request rate, error rate, latency (RED method)
  ├── Resource utilization (CPU, memory)
  └── Business metrics (orders/sec, revenue)

Level 2: Service Detail
  ├── Per-endpoint breakdown
  ├── Per-instance breakdown
  └── Dependency health (database, cache, downstream services)

Level 3: Infrastructure
  ├── Node-level metrics
  ├── Pod-level metrics
  └── Network, disk, I/O
```

**Best practices:**
- Use **template variables** to make dashboards reusable across environments
- Place the most important panels (error rate, latency) at the top
- Use **consistent color coding**: green = healthy, yellow = warning, red = critical
- Set **meaningful thresholds** on panels (e.g., latency > 500ms = yellow)
- Include **links** between dashboards for drill-down navigation

### 36.3.2 Panels and Visualization Types

| Panel Type     | Best For                                      |
|----------------|-----------------------------------------------|
| Time series    | Metrics over time (request rate, latency)     |
| Stat           | Single current value (uptime, error count)    |
| Gauge          | Value within a range (CPU utilization %)      |
| Bar gauge      | Comparing multiple values                     |
| Table          | Detailed multi-column data                    |
| Heatmap        | Distribution over time (latency heatmap)      |
| Logs           | Log entries (with Loki data source)           |
| Node graph     | Service topology and dependencies             |

### 36.3.3 Template Variables

Template variables make dashboards dynamic. Users can select values from
dropdowns to filter data across all panels.

```
# Variable: namespace
# Type: Query
# Data source: Prometheus
# Query: label_values(kube_pod_info, namespace)

# Variable: pod
# Type: Query
# Depends on: namespace
# Query: label_values(kube_pod_info{namespace="$namespace"}, pod)

# Variable: handler
# Type: Query
# Query: label_values(myapp_http_requests_total, handler)
```

### 36.3.4 Hands-On: Creating a Dashboard for the Go Application

This dashboard JSON can be imported directly into Grafana. It provides a
complete RED-method overview of the instrumented Go application.

```json
{
  "dashboard": {
    "title": "MyApp - Service Overview",
    "tags": ["myapp", "service"],
    "timezone": "browser",
    "templating": {
      "list": [
        {
          "name": "namespace",
          "type": "query",
          "datasource": "Prometheus",
          "query": "label_values(myapp_http_requests_total, namespace)",
          "refresh": 2
        },
        {
          "name": "handler",
          "type": "query",
          "datasource": "Prometheus",
          "query": "label_values(myapp_http_requests_total{namespace=\"$namespace\"}, handler)",
          "refresh": 2,
          "includeAll": true,
          "allValue": ".*"
        }
      ]
    },
    "panels": [
      {
        "title": "Request Rate",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 8, "x": 0, "y": 0 },
        "targets": [
          {
            "expr": "sum by (handler) (rate(myapp_http_requests_total{namespace=\"$namespace\", handler=~\"$handler\"}[5m]))",
            "legendFormat": "{{handler}}"
          }
        ]
      },
      {
        "title": "Error Rate (%)",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 8, "x": 8, "y": 0 },
        "targets": [
          {
            "expr": "sum by (handler) (rate(myapp_http_requests_total{namespace=\"$namespace\", handler=~\"$handler\", status=~\"5..\"}[5m])) / sum by (handler) (rate(myapp_http_requests_total{namespace=\"$namespace\", handler=~\"$handler\"}[5m])) * 100",
            "legendFormat": "{{handler}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "thresholds": {
              "steps": [
                { "value": 0, "color": "green" },
                { "value": 1, "color": "yellow" },
                { "value": 5, "color": "red" }
              ]
            }
          }
        }
      },
      {
        "title": "Latency (p95)",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 8, "x": 16, "y": 0 },
        "targets": [
          {
            "expr": "histogram_quantile(0.95, sum by (le, handler) (rate(myapp_http_request_duration_seconds_bucket{namespace=\"$namespace\", handler=~\"$handler\"}[5m])))",
            "legendFormat": "p95 {{handler}}"
          },
          {
            "expr": "histogram_quantile(0.50, sum by (le, handler) (rate(myapp_http_request_duration_seconds_bucket{namespace=\"$namespace\", handler=~\"$handler\"}[5m])))",
            "legendFormat": "p50 {{handler}}"
          }
        ],
        "fieldConfig": {
          "defaults": {
            "unit": "s"
          }
        }
      },
      {
        "title": "In-Flight Requests",
        "type": "stat",
        "gridPos": { "h": 4, "w": 6, "x": 0, "y": 8 },
        "targets": [
          {
            "expr": "sum(myapp_http_requests_in_flight{namespace=\"$namespace\"})"
          }
        ]
      },
      {
        "title": "Orders per Second",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 0, "y": 12 },
        "targets": [
          {
            "expr": "sum by (type) (rate(myapp_business_orders_total{namespace=\"$namespace\"}[5m]))",
            "legendFormat": "{{type}}"
          }
        ]
      },
      {
        "title": "Go Runtime - Goroutines",
        "type": "timeseries",
        "gridPos": { "h": 8, "w": 12, "x": 12, "y": 12 },
        "targets": [
          {
            "expr": "go_goroutines{namespace=\"$namespace\", job=\"myapp\"}",
            "legendFormat": "{{pod}}"
          }
        ]
      }
    ]
  }
}
```

---

## 36.4 OpenTelemetry

### 36.4.1 What is OpenTelemetry?

OpenTelemetry (OTel) is a **vendor-neutral** observability framework that provides
APIs, SDKs, and tools for generating, collecting, and exporting telemetry data
(traces, metrics, and logs). It is the merger of OpenTracing and OpenCensus, and
is now the second most active CNCF project after Kubernetes.

**Why OpenTelemetry?**
- **Vendor-neutral**: Instrument once, export to any backend (Jaeger, Zipkin,
  Datadog, New Relic, Prometheus, etc.)
- **Unified**: Single SDK for traces, metrics, and logs
- **Context propagation**: Automatically correlates traces across services
- **Auto-instrumentation**: Automatic instrumentation for common libraries

### 36.4.2 Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Your Application                     │
│                                                          │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐ │
│  │ OTel API    │  │ OTel SDK     │  │ Exporters      │ │
│  │ (tracing,   │──│ (processing, │──│ (OTLP, Jaeger, │ │
│  │  metrics,   │  │  sampling,   │  │  Prometheus,   │ │
│  │  logging)   │  │  batching)   │  │  stdout)       │ │
│  └─────────────┘  └──────────────┘  └───────┬────────┘ │
└──────────────────────────────────────────────┼──────────┘
                                               │ OTLP (gRPC/HTTP)
                                               ▼
                              ┌─────────────────────────┐
                              │   OTel Collector        │
                              │                         │
                              │  Receivers → Processors │
                              │         → Exporters     │
                              └────────┬────────────────┘
                                       │
                          ┌────────────┼────────────┐
                          ▼            ▼            ▼
                    ┌──────────┐ ┌──────────┐ ┌──────────┐
                    │  Jaeger  │ │Prometheus│ │  Loki    │
                    │ (traces) │ │(metrics) │ │ (logs)   │
                    └──────────┘ └──────────┘ └──────────┘
```

**OTel API:** The interface your code uses. It is safe to use even without an
SDK installed (all calls become no-ops). This allows library authors to instrument
their code without forcing a dependency on any specific backend.

**OTel SDK:** The implementation of the API. It handles sampling, batching,
resource detection, and export. You configure the SDK once in your application's
`main()` function.

**OTel Collector:** A standalone process that receives, processes, and exports
telemetry data. It decouples your application from the backend — if you switch
from Jaeger to Zipkin, you only reconfigure the Collector, not your code.

### 36.4.3 Traces: Spans and Context Propagation

A **trace** represents the full journey of a request through a distributed system.
It is composed of **spans**:

```
Trace: abc-123
│
├── Span: API Gateway (100ms)
│   ├── Span: Auth Service (20ms)
│   └── Span: Order Service (75ms)
│       ├── Span: DB Query (30ms)
│       └── Span: Payment Service (40ms)
│           └── Span: Stripe API Call (35ms)
```

**Span attributes:**
- `span.name`: Short description of the operation ("GET /api/orders")
- `span.kind`: CLIENT, SERVER, PRODUCER, CONSUMER, INTERNAL
- `span.status`: OK, ERROR, UNSET
- `span.attributes`: Key-value pairs with additional context
- `span.events`: Timestamped events within the span (e.g., "cache miss")

**Context propagation** is how trace context (trace ID, span ID, trace flags)
flows between services. The most common propagation format is **W3C Trace Context**,
which uses HTTP headers:

```
traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01
tracestate: vendor1=value1,vendor2=value2
```

### 36.4.4 Sampling Strategies

Not every trace needs to be recorded — especially in high-traffic systems.
Sampling reduces the volume of trace data while preserving statistical significance.

| Strategy          | Description                                    | Use Case                |
|-------------------|------------------------------------------------|-------------------------|
| AlwaysOn          | Record every trace                             | Development, low-traffic|
| AlwayOff          | Record nothing                                 | Disabled tracing        |
| TraceIDRatio      | Sample a percentage (e.g., 10%)                | High-traffic production |
| ParentBased       | Respect the parent span's sampling decision    | Cross-service consistency|
| Tail-based        | Decide at Collector after seeing full trace     | Keep interesting traces |

### 36.4.5 Hands-On: Full OpenTelemetry Setup in Go

This example demonstrates a complete OpenTelemetry setup with both tracing and
metrics in a Go application.

```go
// otel.go
//
// Configures OpenTelemetry tracing and metrics for a Go application.
// This file provides InitTelemetry(), which sets up:
//   - A TracerProvider with OTLP gRPC export and configurable sampling.
//   - A MeterProvider with OTLP gRPC export for metrics.
//   - W3C Trace Context propagation for cross-service correlation.
//   - Resource attributes that identify this service in the backend.
//
// Call InitTelemetry() early in main() and defer the returned shutdown
// function to ensure all telemetry is flushed before the process exits.
package main

import (
	"context"
	"fmt"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetricgrpc"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracegrpc"
	"go.opentelemetry.io/otel/propagation"
	"go.opentelemetry.io/otel/sdk/metric"
	"go.opentelemetry.io/otel/sdk/resource"
	"go.opentelemetry.io/otel/sdk/trace"
	semconv "go.opentelemetry.io/otel/semconv/v1.21.0"
)

// InitTelemetry initializes OpenTelemetry tracing and metrics.
//
// Parameters:
//   - ctx: Context for initialization (used for establishing gRPC connections).
//   - serviceName: The logical name of this service (e.g., "order-service").
//     This appears in trace backends as the service identity.
//   - serviceVersion: The version of this service (e.g., "v1.2.3").
//   - collectorEndpoint: The address of the OTel Collector (e.g., "otel-collector:4317").
//
// Returns a shutdown function that flushes all pending telemetry and releases
// resources. Always defer this in main():
//
//	shutdown, err := InitTelemetry(ctx, "my-service", "v1.0.0", "localhost:4317")
//	if err != nil { log.Fatal(err) }
//	defer shutdown(ctx)
func InitTelemetry(ctx context.Context, serviceName, serviceVersion, collectorEndpoint string) (func(context.Context) error, error) {
	// Build a Resource that describes this service. Resources are attached
	// to all telemetry (traces, metrics, logs) and identify the source.
	res, err := resource.Merge(
		resource.Default(),
		resource.NewWithAttributes(
			semconv.SchemaURL,
			semconv.ServiceName(serviceName),
			semconv.ServiceVersion(serviceVersion),
			attribute.String("environment", "production"),
			attribute.String("team", "platform"),
		),
	)
	if err != nil {
		return nil, fmt.Errorf("creating resource: %w", err)
	}

	// --- Tracing Setup ---

	// Create an OTLP trace exporter that sends spans to the Collector via gRPC.
	traceExporter, err := otlptracegrpc.New(ctx,
		otlptracegrpc.WithEndpoint(collectorEndpoint),
		otlptracegrpc.WithInsecure(), // Use TLS in production
	)
	if err != nil {
		return nil, fmt.Errorf("creating trace exporter: %w", err)
	}

	// Create a TracerProvider with:
	//   - Batch span processor (buffers spans and sends them in batches)
	//   - Parent-based sampler that samples 10% of root spans
	//   - Resource identifying this service
	tracerProvider := trace.NewTracerProvider(
		trace.WithBatcher(traceExporter,
			trace.WithBatchTimeout(5*time.Second),
			trace.WithMaxExportBatchSize(512),
		),
		trace.WithSampler(
			trace.ParentBased(trace.TraceIDRatioBased(0.1)),
		),
		trace.WithResource(res),
	)

	// Register the TracerProvider globally so that otel.Tracer() works
	// from any package without passing the provider explicitly.
	otel.SetTracerProvider(tracerProvider)

	// Set up W3C Trace Context propagation. This ensures that trace IDs
	// are passed between services via HTTP headers (traceparent, tracestate).
	otel.SetTextMapPropagator(
		propagation.NewCompositeTextMapPropagator(
			propagation.TraceContext{},
			propagation.Baggage{},
		),
	)

	// --- Metrics Setup ---

	// Create an OTLP metric exporter that sends metrics to the Collector.
	metricExporter, err := otlpmetricgrpc.New(ctx,
		otlpmetricgrpc.WithEndpoint(collectorEndpoint),
		otlpmetricgrpc.WithInsecure(),
	)
	if err != nil {
		return nil, fmt.Errorf("creating metric exporter: %w", err)
	}

	// Create a MeterProvider with a periodic reader that exports metrics
	// every 30 seconds.
	meterProvider := metric.NewMeterProvider(
		metric.WithReader(
			metric.NewPeriodicReader(metricExporter,
				metric.WithInterval(30*time.Second),
			),
		),
		metric.WithResource(res),
	)

	// Register the MeterProvider globally.
	otel.SetMeterProvider(meterProvider)

	// Return a shutdown function that flushes and closes both providers.
	shutdown := func(ctx context.Context) error {
		var errs []error
		if err := tracerProvider.Shutdown(ctx); err != nil {
			errs = append(errs, fmt.Errorf("shutting down tracer provider: %w", err))
		}
		if err := meterProvider.Shutdown(ctx); err != nil {
			errs = append(errs, fmt.Errorf("shutting down meter provider: %w", err))
		}
		if len(errs) > 0 {
			return fmt.Errorf("shutdown errors: %v", errs)
		}
		return nil
	}

	return shutdown, nil
}
```

```go
// server_otel.go
//
// HTTP server instrumented with OpenTelemetry tracing and metrics.
// Each request creates a span, records metrics, and propagates trace
// context to downstream services.
//
// This demonstrates:
//   - Creating spans with attributes and events
//   - Recording span errors
//   - Using OTel metrics (counters, histograms)
//   - Propagating trace context in outbound HTTP calls
//   - Extracting trace/span IDs for log correlation
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"io"
	"log/slog"
	"net/http"
	"time"

	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/codes"
	"go.opentelemetry.io/otel/metric"
	otelTrace "go.opentelemetry.io/otel/trace"
)

// tracer is the application-level tracer. The name identifies this
// instrumentation library in the trace backend, making it easy to
// filter spans by origin.
var tracer = otel.Tracer("myapp/server")

// meter is the application-level meter for recording custom metrics
// via OpenTelemetry (as opposed to Prometheus client_golang).
var meter = otel.Meter("myapp/server")

// otelRequestCount tracks the total number of requests processed,
// recorded via OpenTelemetry metrics (which can be exported to
	// Prometheus via the OTLP-to-Prometheus pipeline).
var otelRequestCount metric.Int64Counter

// otelRequestDuration records request latency in milliseconds via
// OpenTelemetry metrics.
var otelRequestDuration metric.Float64Histogram

// initMetrics creates the OTel metric instruments. This must be called
// after InitTelemetry() has set the global MeterProvider.
func initMetrics() error {
	var err error

	otelRequestCount, err = meter.Int64Counter("myapp.http.requests",
		metric.WithDescription("Total HTTP requests"),
		metric.WithUnit("{request}"),
	)
	if err != nil {
		return fmt.Errorf("creating request counter: %w", err)
	}

	otelRequestDuration, err = meter.Float64Histogram("myapp.http.duration",
		metric.WithDescription("HTTP request duration"),
		metric.WithUnit("ms"),
	)
	if err != nil {
		return fmt.Errorf("creating duration histogram: %w", err)
	}

	return nil
}

// handleGetOrder handles GET /api/orders/:id. It demonstrates:
//   - Creating a child span for the handler
//   - Adding span attributes for request context
//   - Adding span events for significant milestones
//   - Calling a downstream service with trace propagation
//   - Recording errors on the span
func handleGetOrder(w http.ResponseWriter, r *http.Request) {
	// Start a new span for this handler. The span is automatically
	// linked to any parent span extracted from the incoming request headers.
	ctx, span := tracer.Start(r.Context(), "handleGetOrder",
		otelTrace.WithAttributes(
			attribute.String("http.method", r.Method),
			attribute.String("http.url", r.URL.String()),
			attribute.String("http.user_agent", r.UserAgent()),
		),
	)
	defer span.End()

	start := time.Now()

	// Simulate a database lookup with its own child span.
	order, err := fetchOrderFromDB(ctx, "order-123")
	if err != nil {
		// Record the error on the span. This marks the span as failed
		// in the trace backend, making it easy to find error traces.
		span.RecordError(err)
		span.SetStatus(codes.Error, "failed to fetch order")

		http.Error(w, `{"error": "order not found"}`, http.StatusNotFound)

		// Record metrics for the failed request.
		otelRequestCount.Add(ctx, 1,
			metric.WithAttributes(
				attribute.String("handler", "/api/orders"),
				attribute.String("status", "error"),
			),
		)
		return
	}

	// Add a span event to mark a milestone in the request processing.
	span.AddEvent("order fetched", otelTrace.WithAttributes(
			attribute.String("order.id", order.ID),
			attribute.String("order.status", order.Status),
		))

	// Call a downstream service with trace context propagation.
	enrichErr := enrichOrderFromInventory(ctx, order)
	if enrichErr != nil {
		// Log the error but don't fail the request — enrichment is optional.
		span.AddEvent("inventory enrichment failed",
			otelTrace.WithAttributes(attribute.String("error", enrichErr.Error())),
		)
	}

	// Record success metrics.
	duration := float64(time.Since(start).Milliseconds())
	otelRequestCount.Add(ctx, 1,
		metric.WithAttributes(
			attribute.String("handler", "/api/orders"),
			attribute.String("status", "success"),
		),
	)
	otelRequestDuration.Record(ctx, duration,
		metric.WithAttributes(
			attribute.String("handler", "/api/orders"),
		),
	)

	// Set the span status to OK.
	span.SetStatus(codes.Ok, "")

	// Return the response.
	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(order)
}

// Order represents a customer order. Exported for JSON serialization.
type Order struct {
	ID       string  `json:"id"`
	Status   string  `json:"status"`
	Total    float64 `json:"total"`
	Enriched bool    `json:"enriched"`
}

// fetchOrderFromDB simulates a database query with its own traced span.
// In production, this would use a real database client with OTel
// instrumentation (e.g., otelsql for database/sql).
func fetchOrderFromDB(ctx context.Context, orderID string) (*Order, error) {
	// Create a child span for the DB operation.
	ctx, span := tracer.Start(ctx, "db.query",
		otelTrace.WithAttributes(
			attribute.String("db.system", "postgresql"),
			attribute.String("db.statement", "SELECT * FROM orders WHERE id = $1"),
			attribute.String("db.operation", "SELECT"),
			attribute.String("db.table", "orders"),
		),
		otelTrace.WithSpanKind(otelTrace.SpanKindClient),
	)
	defer span.End()

	// Simulate DB latency.
	time.Sleep(25 * time.Millisecond)

	return &Order{
		ID:     orderID,
		Status: "confirmed",
		Total:  99.99,
	}, nil
}

// enrichOrderFromInventory calls the inventory service to add stock info.
// This demonstrates outbound HTTP calls with automatic trace propagation.
//
// The otelhttp.NewTransport() wraps the default HTTP transport to:
//   - Create a client span for the outbound call
//   - Inject trace context (W3C traceparent header) into the request
//   - Record the response status on the span
func enrichOrderFromInventory(ctx context.Context, order *Order) error {
	// Create an HTTP client with OTel instrumentation.
	// The transport automatically injects traceparent headers.
	client := &http.Client{
		Transport: otelhttp.NewTransport(http.DefaultTransport),
		Timeout:   5 * time.Second,
	}

	req, err := http.NewRequestWithContext(ctx, "GET",
		"http://inventory-service:8080/api/stock/"+order.ID, nil)
	if err != nil {
		return fmt.Errorf("creating request: %w", err)
	}

	resp, err := client.Do(req)
	if err != nil {
		return fmt.Errorf("calling inventory service: %w", err)
	}
	defer resp.Body.Close()
	io.ReadAll(resp.Body) // Drain the body

	if resp.StatusCode != http.StatusOK {
		return fmt.Errorf("inventory service returned %d", resp.StatusCode)
	}

	order.Enriched = true
	return nil
}

// main initializes OpenTelemetry and starts the HTTP server.
func main() {
	ctx := context.Background()

	// Initialize OpenTelemetry (traces + metrics).
	shutdown, err := InitTelemetry(ctx,
		"order-service", "v1.0.0", "otel-collector:4317")
	if err != nil {
		slog.Error("failed to initialize telemetry", "error", err)
		return
	}
	defer shutdown(ctx)

	// Initialize OTel metric instruments.
	if err := initMetrics(); err != nil {
		slog.Error("failed to initialize metrics", "error", err)
		return
	}

	// Create an HTTP mux with OTel HTTP middleware.
	// otelhttp.NewHandler automatically:
	//   - Extracts trace context from incoming requests
	//   - Creates a server span for each request
	//   - Records HTTP metrics (request count, duration, size)
	mux := http.NewServeMux()
	mux.HandleFunc("/api/orders", handleGetOrder)

	handler := otelhttp.NewHandler(mux, "myapp-server")

	server := &http.Server{
		Addr:    ":8080",
		Handler: handler,
	}

	slog.Info("starting server", "addr", ":8080")
	if err := server.ListenAndServe(); err != nil {
		slog.Error("server failed", "error", err)
	}
}
```

### 36.4.6 OTel Collector Configuration

The OpenTelemetry Collector receives telemetry from your applications and routes
it to the appropriate backends.

```yaml
# otel-collector-config.yaml
#
# This configuration:
#   - Receives traces and metrics via OTLP (gRPC on 4317, HTTP on 4318)
#   - Processes telemetry with batching and memory limiting
#   - Exports traces to Jaeger and metrics to Prometheus
#   - Adds Kubernetes metadata to all telemetry

receivers:
  # OTLP receiver — the standard protocol for receiving OTel data
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  # Batch processor — groups telemetry into batches for efficient export
  batch:
    timeout: 5s
    send_batch_size: 1024
    send_batch_max_size: 2048

  # Memory limiter — prevents the Collector from running out of memory
  memory_limiter:
    check_interval: 5s
    limit_mib: 512
    spike_limit_mib: 128

  # Resource processor — adds/modifies resource attributes
  resource:
    attributes:
      - key: environment
        value: production
        action: upsert

  # Kubernetes attributes processor — enriches telemetry with pod metadata
  k8sattributes:
    extract:
      metadata:
        - k8s.pod.name
        - k8s.namespace.name
        - k8s.deployment.name
        - k8s.node.name
    pod_association:
      - sources:
          - from: resource_attribute
            name: k8s.pod.ip

exporters:
  # Jaeger exporter for traces
  otlp/jaeger:
    endpoint: jaeger-collector:4317
    tls:
      insecure: true

  # Prometheus exporter — serves metrics on :8889 for Prometheus to scrape
  prometheus:
    endpoint: 0.0.0.0:8889
    namespace: otel
    resource_to_telemetry_conversion:
      enabled: true

  # Debug exporter — logs telemetry to stdout (useful for debugging)
  debug:
    verbosity: basic

extensions:
  # Health check extension — liveness/readiness probes
  health_check:
    endpoint: 0.0.0.0:13133
  # zPages for debugging the Collector itself
  zpages:
    endpoint: 0.0.0.0:55679

service:
  extensions: [health_check, zpages]
  pipelines:
    traces:
      receivers: [otlp]
      processors: [memory_limiter, k8sattributes, resource, batch]
      exporters: [otlp/jaeger, debug]
    metrics:
      receivers: [otlp]
      processors: [memory_limiter, k8sattributes, resource, batch]
      exporters: [prometheus, debug]
```

**Deploying the Collector on Kubernetes:**

```yaml
# otel-collector-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  replicas: 2
  selector:
    matchLabels:
      app: otel-collector
  template:
    metadata:
      labels:
        app: otel-collector
    spec:
      serviceAccountName: otel-collector
      containers:
        - name: collector
          image: otel/opentelemetry-collector-contrib:0.90.0
          args: ["--config=/conf/otel-collector-config.yaml"]
          ports:
            - name: otlp-grpc
              containerPort: 4317
            - name: otlp-http
              containerPort: 4318
            - name: prometheus
              containerPort: 8889
            - name: health
              containerPort: 13133
          volumeMounts:
            - name: config
              mountPath: /conf
          resources:
            requests:
              cpu: 200m
              memory: 256Mi
            limits:
              cpu: "1"
              memory: 512Mi
          livenessProbe:
            httpGet:
              path: /
              port: health
          readinessProbe:
            httpGet:
              path: /
              port: health
      volumes:
        - name: config
          configMap:
            name: otel-collector-config
---
apiVersion: v1
kind: Service
metadata:
  name: otel-collector
  namespace: monitoring
spec:
  selector:
    app: otel-collector
  ports:
    - name: otlp-grpc
      port: 4317
      targetPort: otlp-grpc
    - name: otlp-http
      port: 4318
      targetPort: otlp-http
    - name: prometheus
      port: 8889
      targetPort: prometheus
```

### 36.4.7 Hands-On: Trace Propagation Across Microservices

Here's how the **inventory service** (called by the order service above) receives
and continues the trace:

```go
// inventory_service.go
//
// The inventory microservice demonstrates the receiving side of trace
// propagation. When the order service calls this service, the incoming
// HTTP headers contain the W3C traceparent header. The OTel HTTP
// middleware automatically extracts the trace context, so any spans
// created in this service become children of the order service's span.
//
// This creates a complete distributed trace:
//   Order Service → Inventory Service → DB Lookup
package main

import (
	"context"
	"encoding/json"
	"log/slog"
	"net/http"
	"time"

	"go.opentelemetry.io/contrib/instrumentation/net/http/otelhttp"
	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	otelTrace "go.opentelemetry.io/otel/trace"
)

// inventoryTracer is this service's tracer instance.
var inventoryTracer = otel.Tracer("inventory-service")

// StockInfo represents inventory stock information for a product.
type StockInfo struct {
	OrderID   string `json:"order_id"`
	InStock   bool   `json:"in_stock"`
	Quantity  int    `json:"quantity"`
	Warehouse string `json:"warehouse"`
}

// handleGetStock returns stock information for an order.
// The trace context is automatically extracted from the incoming
// request by the otelhttp middleware, linking this span to the
// caller's trace.
func handleGetStock(w http.ResponseWriter, r *http.Request) {
	ctx := r.Context()

	// This span becomes a child of the order service's
	// "enrichOrderFromInventory" span.
	ctx, span := inventoryTracer.Start(ctx, "handleGetStock",
		otelTrace.WithAttributes(
			attribute.String("http.method", r.Method),
			attribute.String("http.path", r.URL.Path),
		),
	)
	defer span.End()

	// Simulate a database lookup with its own span.
	stock, err := lookupStock(ctx, "order-123")
	if err != nil {
		span.RecordError(err)
		http.Error(w, `{"error": "stock lookup failed"}`, http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	json.NewEncoder(w).Encode(stock)
}

// lookupStock performs the actual inventory database query.
// It creates a child span to track DB latency independently
// from the handler span.
func lookupStock(ctx context.Context, orderID string) (*StockInfo, error) {
	_, span := inventoryTracer.Start(ctx, "db.inventory_lookup",
		otelTrace.WithAttributes(
			attribute.String("db.system", "postgresql"),
			attribute.String("db.operation", "SELECT"),
			attribute.String("order.id", orderID),
		),
		otelTrace.WithSpanKind(otelTrace.SpanKindClient),
	)
	defer span.End()

	// Simulate DB latency.
	time.Sleep(15 * time.Millisecond)

	return &StockInfo{
		OrderID:   orderID,
		InStock:   true,
		Quantity:  42,
		Warehouse: "us-east-1",
	}, nil
}

func main() {
	ctx := context.Background()

	// Initialize OTel (same pattern as the order service).
	shutdown, err := InitTelemetry(ctx,
		"inventory-service", "v1.0.0", "otel-collector:4317")
	if err != nil {
		slog.Error("failed to init telemetry", "error", err)
		return
	}
	defer shutdown(ctx)

	mux := http.NewServeMux()
	mux.HandleFunc("/api/stock/", handleGetStock)

	// Wrap with OTel HTTP handler — this extracts trace context from
	// incoming requests and creates server spans automatically.
	handler := otelhttp.NewHandler(mux, "inventory-service")

	slog.Info("starting inventory service", "addr", ":8080")
	http.ListenAndServe(":8080", handler)
}
```

The resulting trace in Jaeger looks like:

```
Trace: abc-123-def-456
│
├── [order-service] GET /api/orders (105ms)
│   ├── [order-service] db.query (25ms)
│   │   └── db.system: postgresql
│   │       db.operation: SELECT
│   └── [order-service] HTTP GET inventory-service (45ms)
│       └── [inventory-service] GET /api/stock/order-123 (40ms)
│           └── [inventory-service] db.inventory_lookup (15ms)
│               └── db.system: postgresql
│                   db.operation: SELECT
```

---

## 36.5 Structured Logging

### 36.5.1 Why Structured Logging?

Unstructured logs like `log.Printf("user %s created order %s", user, orderID)`
are human-readable but machine-hostile. You can't efficiently search, filter, or
aggregate them. Structured logging produces machine-parseable output (typically
JSON) with consistent fields.

**Unstructured:**
```
2024-01-15 14:03:22 INFO: user john created order abc-123
```

**Structured:**
```json
{
  "time": "2024-01-15T14:03:22Z",
  "level": "INFO",
  "msg": "order created",
  "user_id": "john",
  "order_id": "abc-123",
  "trace_id": "0af7651916cd43dd8448eb211c80319c",
  "span_id": "b7ad6b7169203331"
}
```

With structured logs, you can:
- Filter: `jq 'select(.level == "ERROR")'`
- Search: `jq 'select(.order_id == "abc-123")'`
- Aggregate: Count errors per user, per endpoint
- Correlate: Join logs with traces via `trace_id`

### 36.5.2 Go Logging Libraries Comparison

| Library    | Stdlib? | Structured? | Performance | Notes                          |
|------------|---------|-------------|-------------|--------------------------------|
| `log`      | Yes     | No          | Low         | Basic, not for production      |
| `log/slog` | Yes (1.21+) | Yes     | High        | Recommended for new projects   |
| `zerolog`  | No      | Yes         | Very High   | Zero-allocation, fastest       |
| `zap`      | No      | Yes         | Very High   | Uber, great ecosystem          |
| `logr`     | No      | Yes (iface) | Varies      | Interface, works with any backend|

### 36.5.3 Hands-On: Structured Logging with slog

```go
// logging.go
//
// Demonstrates production-ready structured logging using Go's built-in
// log/slog package (Go 1.21+). This file covers:
//
//   - Configuring slog with a JSON handler for production
//   - Adding default attributes to every log entry
//   - Log levels and dynamic level switching
//   - Correlation with OpenTelemetry trace context
//   - Using slog.LogValuer for lazy/expensive attribute computation
//   - Grouping related attributes
//   - Middleware that adds request context to all downstream logs
//
// slog is the recommended logging library for new Go projects because:
//   - It's in the standard library (no external dependency)
//   - It's designed for structured output (JSON, logfmt, etc.)
//   - It supports log levels natively
//   - It integrates well with OpenTelemetry via context
package main

import (
	"context"
	"log/slog"
	"net/http"
	"os"
	"time"

	"go.opentelemetry.io/otel/trace"
)

// NewLogger creates a production-ready slog.Logger configured with:
//   - JSON output (for machine parsing by log aggregators)
//   - Configurable minimum log level
//   - Source code location in log entries
//   - Default attributes attached to every log entry
//
// Parameters:
//   - level: The minimum log level to emit. Messages below this level
//     are silently discarded. Use slog.LevelDebug in development,
//     slog.LevelInfo in production.
//   - serviceName: Identifies this service in aggregated logs.
//   - version: The application version for tracking deployments.
//
// Returns a configured *slog.Logger ready for use.
func NewLogger(level slog.Level, serviceName, version string) *slog.Logger {
	// LevelVar allows dynamic level changes at runtime
	// (e.g., via an admin HTTP endpoint).
	levelVar := &slog.LevelVar{}
	levelVar.Set(level)

	handler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
			// AddSource includes the source file and line number in each
			// log entry. Useful for debugging, but adds ~5% overhead.
			AddSource: true,
			// Level controls the minimum level. Using a LevelVar allows
			// runtime adjustment without restarting the process.
			Level: levelVar,
			// ReplaceAttr customizes how attributes are formatted.
			ReplaceAttr: func(groups []string, a slog.Attr) slog.Attr {
				// Rename the built-in "time" key to "@timestamp" for
				// compatibility with Elasticsearch/ELK stack.
				if a.Key == slog.TimeKey {
					a.Key = "@timestamp"
				}
				// Format time in RFC3339 with nanosecond precision.
				if a.Key == "@timestamp" {
					a.Value = slog.StringValue(a.Value.Time().Format(time.RFC3339Nano))
				}
				return a
			},
		})

	// Wrap the handler to add default attributes to every log entry.
	// These attributes identify the service and are invaluable when
	// logs from multiple services are aggregated in a single system.
	logger := slog.New(handler).With(
		slog.String("service", serviceName),
		slog.String("version", version),
	)

	return logger
}

// traceContextHandler is an slog.Handler wrapper that automatically
// extracts OpenTelemetry trace context from the context.Context and
// adds trace_id and span_id to every log entry. This enables
// correlation between logs and traces in your observability backend.
type traceContextHandler struct {
	inner slog.Handler
}

// Enabled delegates to the inner handler's level check.
func (h *traceContextHandler) Enabled(ctx context.Context, level slog.Level) bool {
	return h.inner.Enabled(ctx, level)
}

// Handle extracts trace context from ctx and adds it to the log record
// before passing it to the inner handler.
func (h *traceContextHandler) Handle(ctx context.Context, record slog.Record) error {
	// Extract the current span from the context.
	spanCtx := trace.SpanContextFromContext(ctx)
	if spanCtx.IsValid() {
		// Add trace_id and span_id as top-level log attributes.
		// These are the same IDs used by OTel, enabling direct
		// correlation in backends like Grafana (Loki + Tempo).
		record.AddAttrs(
			slog.String("trace_id", spanCtx.TraceID().String()),
			slog.String("span_id", spanCtx.SpanID().String()),
		)
		// Add trace flags (sampled, etc.)
		if spanCtx.IsSampled() {
			record.AddAttrs(slog.Bool("trace_sampled", true))
		}
	}
	return h.inner.Handle(ctx, record)
}

// WithAttrs returns a new handler with the given attributes pre-added.
func (h *traceContextHandler) WithAttrs(attrs []slog.Attr) slog.Handler {
	return &traceContextHandler{inner: h.inner.WithAttrs(attrs)}
}

// WithGroup returns a new handler that groups subsequent attributes
// under the given name.
func (h *traceContextHandler) WithGroup(name string) slog.Handler {
	return &traceContextHandler{inner: h.inner.WithGroup(name)}
}

// NewLoggerWithTraceContext creates a logger that automatically includes
// OpenTelemetry trace context in every log entry. Use this as the default
// logger when your application uses OpenTelemetry tracing.
//
// Example output:
//
//	{
	//	  "@timestamp": "2024-01-15T14:03:22.123456789Z",
	//	  "level": "INFO",
	//	  "msg": "order created",
	//	  "service": "order-service",
	//	  "trace_id": "0af7651916cd43dd8448eb211c80319c",
	//	  "span_id": "b7ad6b7169203331",
	//	  "trace_sampled": true,
	//	  "order_id": "abc-123"
	//	}
func NewLoggerWithTraceContext(level slog.Level, serviceName, version string) *slog.Logger {
	jsonHandler := slog.NewJSONHandler(os.Stdout, &slog.HandlerOptions{
			AddSource: true,
			Level:     level,
		})

	traceHandler := &traceContextHandler{inner: jsonHandler}

	return slog.New(traceHandler).With(
		slog.String("service", serviceName),
		slog.String("version", version),
	)
}

// LoggingMiddleware returns an HTTP middleware that:
//   1. Creates a child logger with request-specific attributes
//   2. Stores the logger in the request context for downstream handlers
//   3. Logs the request completion with duration and status code
//
// This ensures that all log entries within a request handler automatically
// include the request method, path, and request ID.
func LoggingMiddleware(logger *slog.Logger) func(http.Handler) http.Handler {
	return func(next http.Handler) http.Handler {
		return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
				start := time.Now()

				// Create a child logger with request-specific attributes.
				requestLogger := logger.With(
					slog.String("method", r.Method),
					slog.String("path", r.URL.Path),
					slog.String("remote_addr", r.RemoteAddr),
					slog.String("user_agent", r.UserAgent()),
				)

				// Extract or generate a request/correlation ID.
				requestID := r.Header.Get("X-Request-ID")
				if requestID == "" {
					requestID = r.Header.Get("X-Correlation-ID")
				}
				if requestID != "" {
					requestLogger = requestLogger.With(slog.String("request_id", requestID))
				}

				// Wrap the ResponseWriter to capture the status code.
				wrapped := &responseWriter{ResponseWriter: w, statusCode: http.StatusOK}

				// Log the request start at Debug level (to avoid noise in production).
				requestLogger.DebugContext(r.Context(), "request started")

				// Serve the request.
				next.ServeHTTP(wrapped, r)

				// Log the request completion with duration and status.
				duration := time.Since(start)
				requestLogger.InfoContext(r.Context(), "request completed",
					slog.Int("status", wrapped.statusCode),
					slog.Duration("duration", duration),
					slog.Int("bytes", wrapped.bytesWritten),
				)
			})
	}
}

// SensitiveValue implements slog.LogValuer to redact sensitive data
// in log output. When slog encounters a LogValuer, it calls LogValue()
// to get the actual value to log.
//
// Usage:
//
//	logger.Info("user authenticated",
	//	    slog.Any("email", SensitiveValue{Value: user.Email}),
	//	)
//
// Output: {"msg": "user authenticated", "email": "j***@example.com"}
type SensitiveValue struct {
	Value string
}

// LogValue returns a redacted version of the sensitive value.
// The first character and domain (for emails) are preserved
// for debugging, while the rest is masked.
func (s SensitiveValue) LogValue() slog.Value {
	if len(s.Value) <= 1 {
		return slog.StringValue("***")
	}
	return slog.StringValue(s.Value[:1] + "***" + s.Value[len(s.Value)-1:])
}
```

### 36.5.4 Log Aggregation in Kubernetes

In Kubernetes, logs from containers are written to stdout/stderr and collected
by the container runtime. A log aggregator (Loki, Fluentd, Fluent Bit, or the
ELK stack) collects these logs and makes them searchable.

**Grafana Loki** is the recommended solution for Kubernetes because:
- It indexes only metadata (labels), not log content — making it cheap to operate
- It uses the same label model as Prometheus
- It integrates natively with Grafana for unified metrics + logs

```yaml
# Loki stack with Helm
helm repo add grafana https://grafana.github.io/helm-charts
helm install loki grafana/loki-stack \
  --namespace monitoring \
  --set promtail.enabled=true \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=10Gi
```

**LogQL** (Loki's query language) examples:

```logql
# All logs from the order-service
{app="order-service"}

# Error logs only
{app="order-service"} |= "ERROR"

# JSON parsing with field filtering
{app="order-service"} | json | level="ERROR" | order_id != ""

# Logs for a specific trace
{app=~"order-service|inventory-service"} | json | trace_id="0af7651916cd43dd"

# Log volume (requests per second)
sum(rate({app="order-service"} | json | level="INFO" | msg="request completed" [5m]))
```

---

## 36.6 Kubernetes-Specific Observability

### 36.6.1 kube-state-metrics

kube-state-metrics listens to the Kubernetes API server and generates metrics
about the state of Kubernetes objects. It does NOT measure resource utilization —
it measures the declared and actual state of objects.

**Key metrics:**

```promql
# Pods not in Running state
kube_pod_status_phase{phase!="Running"} == 1

# Pods in CrashLoopBackOff
kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"} == 1

# Deployments with unavailable replicas
kube_deployment_status_replicas_unavailable > 0

# PVCs with Pending status
kube_persistentvolumeclaim_status_phase{phase="Pending"} == 1

# Jobs that have failed
kube_job_status_failed > 0

# HPA at max replicas (may need more capacity)
kube_horizontalpodautoscaler_status_current_replicas
== kube_horizontalpodautoscaler_spec_max_replicas
```

### 36.6.2 node-exporter

node-exporter runs as a DaemonSet and exposes host-level metrics from every node.

**Key metrics:**

```promql
# CPU utilization per node
1 - avg by (instance) (rate(node_cpu_seconds_total{mode="idle"}[5m]))

# Memory utilization per node
1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# Disk utilization
1 - (node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})

# Network throughput (bytes/sec)
rate(node_network_receive_bytes_total[5m])
rate(node_network_transmit_bytes_total[5m])

# Disk I/O utilization
rate(node_disk_io_time_seconds_total[5m])
```

### 36.6.3 cAdvisor

cAdvisor (Container Advisor) is built into the kubelet and exposes container-level
metrics. These metrics are automatically available in Prometheus when using
kube-prometheus-stack.

**Key metrics:**

```promql
# Container CPU usage
sum by (pod) (rate(container_cpu_usage_seconds_total{container!=""}[5m]))

# Container memory usage
sum by (pod) (container_memory_working_set_bytes{container!=""})

# Container memory vs. limit (OOM risk)
container_memory_working_set_bytes{container!=""}
/ container_spec_memory_limit_bytes{container!=""} * 100

# Container restarts
sum by (pod) (kube_pod_container_status_restarts_total)

# Container network I/O
sum by (pod) (rate(container_network_receive_bytes_total[5m]))
```

### 36.6.4 kubectl top and metrics-server

The metrics-server is a cluster-level aggregator of resource usage data. It
provides the data for `kubectl top` and for the Horizontal Pod Autoscaler.

```bash
# Install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# View node resource usage
kubectl top nodes

# View pod resource usage
kubectl top pods -n default

# View pod resource usage sorted by CPU
kubectl top pods --sort-by=cpu

# View container-level usage
kubectl top pods --containers
```

### 36.6.5 Kubernetes Events as Observability Signal

Kubernetes events are often overlooked but contain valuable information about
cluster operations: pod scheduling, image pulls, OOM kills, scaling events, etc.

```bash
# View recent events
kubectl get events --sort-by='.lastTimestamp' -n default

# Watch events in real time
kubectl get events -w

# Filter events by type (Warning events indicate problems)
kubectl get events --field-selector type=Warning
```

**Exporting events as metrics:**

The `kube-events-exporter` or `kubernetes-event-exporter` can forward events to
external systems (Elasticsearch, Loki, Slack).

```yaml
# kubernetes-event-exporter config
apiVersion: v1
kind: ConfigMap
metadata:
  name: event-exporter-cfg
  namespace: monitoring
data:
  config.yaml: |
    logLevel: error
    logFormat: json
    route:
      routes:
        - match:
            - receiver: "loki"
          drop:
            - namespace: "kube-system"
              reason: "SuccessfulCreate"
    receivers:
      - name: "loki"
        loki:
          url: "http://loki:3100/loki/api/v1/push"
          streamLabels:
            source: "kubernetes-events"
```

### 36.6.6 Hands-On: Setting Up a Complete Monitoring Stack

This section brings everything together: Prometheus, Grafana, Loki, Jaeger, and
the OTel Collector — all deployed on Kubernetes.

```bash
#!/bin/bash
# deploy-monitoring-stack.sh
#
# Deploys a complete observability stack on Kubernetes:
#   - Prometheus + Alertmanager (metrics)
#   - Grafana (visualization)
#   - Loki + Promtail (logs)
#   - Jaeger (traces)
#   - OpenTelemetry Collector (telemetry pipeline)

set -euo pipefail

NAMESPACE="monitoring"

echo "=== Creating monitoring namespace ==="
kubectl create namespace "$NAMESPACE" --dry-run=client -o yaml | kubectl apply -f -

echo "=== Adding Helm repos ==="
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

echo "=== Installing kube-prometheus-stack (Prometheus + Grafana) ==="
helm upgrade --install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  --namespace "$NAMESPACE" \
  --set prometheus.prometheusSpec.retention=7d \
  --set prometheus.prometheusSpec.serviceMonitorSelectorNilUsesHelmValues=false \
  --set prometheus.prometheusSpec.podMonitorSelectorNilUsesHelmValues=false \
  --set grafana.adminPassword=admin \
  --wait

echo "=== Installing Loki + Promtail (logs) ==="
helm upgrade --install loki grafana/loki-stack \
  --namespace "$NAMESPACE" \
  --set promtail.enabled=true \
  --set loki.persistence.enabled=true \
  --set loki.persistence.size=10Gi \
  --wait

echo "=== Installing Jaeger (traces) ==="
helm upgrade --install jaeger jaegertracing/jaeger \
  --namespace "$NAMESPACE" \
  --set provisionDataStore.cassandra=false \
  --set allInOne.enabled=true \
  --set storage.type=memory \
  --set query.enabled=false \
  --wait

echo "=== Installing OTel Collector ==="
helm upgrade --install otel-collector open-telemetry/opentelemetry-collector \
  --namespace "$NAMESPACE" \
  --set mode=deployment \
  --set config.receivers.otlp.protocols.grpc.endpoint="0.0.0.0:4317" \
  --set config.receivers.otlp.protocols.http.endpoint="0.0.0.0:4318" \
  --wait

echo "=== Verification ==="
kubectl -n "$NAMESPACE" get pods
echo ""
echo "Access Prometheus: kubectl -n $NAMESPACE port-forward svc/kube-prometheus-stack-prometheus 9090:9090"
echo "Access Grafana:    kubectl -n $NAMESPACE port-forward svc/kube-prometheus-stack-grafana 3000:80"
echo "Access Jaeger:     kubectl -n $NAMESPACE port-forward svc/jaeger-query 16686:16686"
```

---

## 36.7 Alerting

### 36.7.1 PrometheusRule CRD

The prometheus-operator uses PrometheusRule CRDs to define alerting rules
declaratively:

```yaml
# alerting-rules.yaml
#
# Production alerting rules following the RED method.
# These rules generate alerts when service-level indicators (SLIs)
# breach their thresholds.
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: myapp-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: myapp.rules
      rules:
        # --- High Error Rate ---
        # Alert when >5% of requests are failing.
        - alert: HighErrorRate
          expr: |
            sum(rate(myapp_http_requests_total{status=~"5.."}[5m]))
            / sum(rate(myapp_http_requests_total[5m]))
            > 0.05
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "High error rate on {{ $labels.namespace }}/myapp"
            description: >
              Error rate is {{ $value | humanizePercentage }} over the last 5 minutes.
              This exceeds the 5% threshold.
            runbook_url: "https://wiki.example.com/runbooks/high-error-rate"

        # --- High Latency ---
        # Alert when p95 latency exceeds 1 second.
        - alert: HighLatency
          expr: |
            histogram_quantile(0.95,
              sum by (le) (rate(myapp_http_request_duration_seconds_bucket[5m]))
            ) > 1
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "High latency on {{ $labels.namespace }}/myapp"
            description: "P95 latency is {{ $value | humanizeDuration }}."

        # --- Pod Crash Looping ---
        - alert: PodCrashLooping
          expr: |
            rate(kube_pod_container_status_restarts_total{
              namespace="default", container="myapp"
            }[15m]) * 60 * 15 > 0
          for: 5m
          labels:
            severity: critical
          annotations:
            summary: "Pod {{ $labels.pod }} is crash looping"
            description: "Pod has restarted {{ $value }} times in the last 15 minutes."

        # --- High Memory Usage ---
        - alert: HighMemoryUsage
          expr: |
            container_memory_working_set_bytes{container="myapp"}
            / container_spec_memory_limit_bytes{container="myapp"}
            > 0.9
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High memory usage for {{ $labels.pod }}"
            description: "Memory usage is at {{ $value | humanizePercentage }} of limit."

        # --- SLO: Availability < 99.9% ---
        - alert: SLOAvailabilityBreach
          expr: |
            (
              1 - (
                sum(rate(myapp_http_requests_total{status=~"5.."}[1h]))
                / sum(rate(myapp_http_requests_total[1h]))
              )
            ) < 0.999
          for: 10m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "SLO breach: availability below 99.9%"
            description: "Current availability: {{ $value | humanizePercentage }}."
```

### 36.7.2 Alertmanager Configuration

```yaml
# alertmanager-config.yaml
#
# Routes alerts to appropriate channels based on severity and team labels.
apiVersion: monitoring.coreos.com/v1alpha1
kind: AlertmanagerConfig
metadata:
  name: myapp
  namespace: monitoring
spec:
  route:
    receiver: default
    groupBy: ['alertname', 'namespace']
    groupWait: 30s
    groupInterval: 5m
    repeatInterval: 4h
    routes:
      # Critical alerts → PagerDuty
      - matchers:
          - name: severity
            value: critical
        receiver: pagerduty-critical
        repeatInterval: 1h

      # Warning alerts → Slack
      - matchers:
          - name: severity
            value: warning
        receiver: slack-warnings
        repeatInterval: 4h

  receivers:
    - name: default
      slackConfigs:
        - apiURL:
            name: slack-webhook-url
            key: url
          channel: '#alerts-default'
          title: '{{ .GroupLabels.alertname }}'
          text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'

    - name: pagerduty-critical
      pagerdutyConfigs:
        - routingKey:
            name: pagerduty-routing-key
            key: key
          severity: critical
          description: '{{ .GroupLabels.alertname }}: {{ .CommonAnnotations.summary }}'

    - name: slack-warnings
      slackConfigs:
        - apiURL:
            name: slack-webhook-url
            key: url
          channel: '#alerts-warnings'
          title: '⚠️ {{ .GroupLabels.alertname }}'
          text: '{{ range .Alerts }}{{ .Annotations.description }}{{ end }}'
```

---

## 36.8 Putting It All Together: The Observability Stack

Here is the final architecture showing how all components integrate:

```
┌─────────────────────────────────────────────────────────────────┐
│                        Kubernetes Cluster                        │
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐ │
│  │ order-svc   │  │ inventory-  │  │ payment-svc             │ │
│  │ (Go + OTel) │──│ svc (Go +   │──│ (Go + OTel)             │ │
│  │             │  │  OTel)      │  │                         │ │
│  └─────┬───────┘  └──────┬──────┘  └──────────┬──────────────┘ │
│        │                 │                     │                 │
│        └─────────────────┼─────────────────────┘                │
│                          │ OTLP (gRPC)                          │
│                          ▼                                      │
│               ┌──────────────────────┐                          │
│               │ OTel Collector       │                          │
│               │ - Receives traces    │                          │
│               │ - Receives metrics   │                          │
│               │ - Enriches with K8s  │                          │
│               │   metadata           │                          │
│               └──┬─────────┬─────┬──┘                          │
│                  │         │     │                              │
│         ┌────────┘    ┌────┘     └────────┐                    │
│         ▼             ▼                   ▼                    │
│  ┌────────────┐ ┌──────────┐     ┌────────────┐               │
│  │ Jaeger     │ │Prometheus│     │  Loki      │               │
│  │ (Traces)   │ │(Metrics) │     │  (Logs)    │               │
│  └─────┬──────┘ └────┬─────┘     └─────┬──────┘               │
│        │             │                  │                      │
│        └─────────────┼──────────────────┘                      │
│                      ▼                                         │
│               ┌──────────────┐                                 │
│               │   Grafana    │                                 │
│               │              │                                 │
│               │ - Dashboards │                                 │
│               │ - Alerts     │                                 │
│               │ - Explore    │                                 │
│               └──────────────┘                                 │
└─────────────────────────────────────────────────────────────────┘
```

**The workflow:**
1. Applications send traces and metrics to the OTel Collector via OTLP
2. Applications write structured JSON logs to stdout
3. Promtail collects logs from pod stdout and ships them to Loki
4. OTel Collector exports traces to Jaeger and metrics to Prometheus
5. Prometheus also scrapes application `/metrics` endpoints directly
6. Grafana visualizes all three signals with cross-linking:
   - Click a spike in a metric panel → jump to related logs in Loki
   - Click a trace ID in logs → jump to the trace in Jaeger
   - Click an error span in Jaeger → see related metrics

---

## 36.9 Best Practices

### 36.9.1 Metric Naming Conventions

Follow the Prometheus naming conventions:

```
# Format: <namespace>_<subsystem>_<name>_<unit>
# Example: myapp_http_request_duration_seconds

# DO:
myapp_http_requests_total        # Counter with _total suffix
myapp_http_request_duration_seconds  # Histogram with unit suffix
myapp_http_response_size_bytes   # Histogram with unit suffix

# DON'T:
myapp_http_request_count         # Use _total for counters
myapp_http_latency               # Missing unit suffix
myapp_http_latency_ms            # Use base units (seconds, not milliseconds)
```

### 36.9.2 Label Cardinality

**The #1 cause of Prometheus performance problems is high-cardinality labels.**

```
# BAD: User ID as a label (millions of unique values)
http_requests_total{user_id="abc-123"}   # ← Creates a new time series per user

# GOOD: Use low-cardinality labels
http_requests_total{method="GET", status="200", handler="/api/orders"}

# Rule of thumb: If a label has more than ~100 unique values,
# it probably shouldn't be a label. Track high-cardinality data
# in logs or traces instead.
```

### 36.9.3 Dashboard Design

- **USE Method** dashboards for infrastructure: Utilization, Saturation, Errors
- **RED Method** dashboards for services: Rate, Errors, Duration
- **Four Golden Signals** for SRE: Latency, Traffic, Errors, Saturation
- Always include links between related dashboards
- Use template variables for namespace, service, and instance filtering

### 36.9.4 Trace Instrumentation Guidelines

- Instrument **boundaries**: HTTP handlers, gRPC methods, database calls, cache lookups
- Add **meaningful attributes**: request IDs, user IDs, order IDs (as span attributes, not labels)
- Record **errors** on spans with `span.RecordError()` and `span.SetStatus(codes.Error)`
- Use **span events** for milestones within a span (cache hit/miss, retry attempt)
- Keep span names **low-cardinality**: `GET /api/orders/{id}`, not `GET /api/orders/abc-123`

### 36.9.5 Log Guidelines

- Always use **structured logging** (JSON)
- Include **trace_id** and **span_id** in every log entry for correlation
- Log at the **right level**: DEBUG for detailed diagnostics, INFO for normal operations, ERROR for failures
- **Don't log sensitive data** (passwords, tokens, PII) — use the `LogValuer` pattern for redaction
- Include enough **context** to understand the log without reading code

---

## 36.10 Summary

| Signal  | Tool              | Storage          | Query Language | Best For                |
|---------|-------------------|------------------|----------------|-------------------------|
| Metrics | Prometheus        | Prometheus TSDB  | PromQL         | Alerting, dashboards    |
| Logs    | Loki              | Object storage   | LogQL          | Debugging, audit trails |
| Traces  | Jaeger/Tempo      | Object storage   | TraceQL        | Performance analysis    |
| All     | OpenTelemetry     | (collector only) | N/A            | Instrumentation SDK     |
| Viz     | Grafana           | N/A              | N/A            | Unified dashboards      |

**Key takeaways:**

1. **Instrument with OpenTelemetry** — it's vendor-neutral and covers all three pillars.
2. **Use Prometheus for metrics** — it's the Kubernetes-native standard.
3. **Use structured JSON logging** with trace correlation — `log/slog` is excellent for Go.
4. **Design dashboards with RED/USE methods** — they guide you from detection to diagnosis.
5. **Alert on symptoms, not causes** — alert on "error rate > 5%" not "disk usage > 80%".
6. **Keep label cardinality low** — this is the most common Prometheus pitfall.
7. **Correlate across pillars** — the real power comes from linking metrics → logs → traces.

In the next chapter, we will use the observability stack built here to drive
**autoscaling decisions** — using Prometheus custom metrics to scale applications
beyond simple CPU-based autoscaling.
