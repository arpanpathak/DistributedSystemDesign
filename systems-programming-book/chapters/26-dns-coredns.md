# Chapter 26: DNS in Kubernetes — CoreDNS Deep Dive

> *"DNS is the phone book of the internet. In Kubernetes, it's the phone book
> of your cluster. When it breaks, nobody can call anyone."*

In the previous chapter we locked down Pod communication with Network Policies.
But every one of those policies assumed that Pods could resolve service names to
IP addresses. That resolution is DNS — and in Kubernetes, DNS is provided by
**CoreDNS**.

This chapter takes you deep into how DNS works inside a Kubernetes cluster:
the naming conventions, the CoreDNS architecture, every important plugin, the
query resolution path inside a Pod, performance tuning, debugging, and
extending DNS with custom entries and ExternalDNS.

DNS is consistently one of the **top three debugging topics** in Kubernetes.
If you understand this chapter, you will resolve most DNS issues in minutes
instead of hours.

---

## 26.1 DNS in Kubernetes — Why It Matters

### 26.1.1 Service Discovery Is DNS-Based

When a Pod wants to talk to a Service, it uses the Service's DNS name:

```go
// ──────────────────────────────────────────────────────────
// Example: A Go application connecting to a backend service.
// It uses the DNS name "backend-svc" — not an IP address.
// Kubernetes DNS resolves this name to the Service's ClusterIP.
// ──────────────────────────────────────────────────────────
package main

import (
	"fmt"
	"io"
	"log"
	"net/http"
	"time"
)

func main() {
	// backendURL uses the Kubernetes service DNS name.
	// The Pod's /etc/resolv.conf is configured to resolve
	// this via CoreDNS, which returns the ClusterIP.
	backendURL := "http://backend-svc:3000/api/data"

	client := &http.Client{
		Timeout: 5 * time.Second,
	}

	resp, err := client.Get(backendURL)
	if err != nil {
		log.Fatalf("Failed to reach backend: %v", err)
	}
	defer resp.Body.Close()

	body, _ := io.ReadAll(resp.Body)
	fmt.Printf("Backend response: %s\n", body)
}
```

Without DNS, you'd need to:

1. Look up the Service's ClusterIP manually.
2. Hardcode it into every Pod's configuration.
3. Update it whenever the Service is recreated (ClusterIP changes).

DNS eliminates all of this. Services get stable DNS names. Pods resolve them
automatically.

### 26.1.2 The DNS Contract

Kubernetes defines a **DNS specification** that every conformant cluster must
implement. This specification guarantees:

- Every Service gets a DNS A/AAAA record.
- Every named port gets a DNS SRV record.
- Headless Services resolve to individual Pod IPs.
- Pods themselves get DNS records.

The implementation is pluggable, but since Kubernetes 1.13, the default (and
de facto standard) implementation is **CoreDNS**.

### 26.1.3 What Happens Without DNS

```
┌─────────────────────────────────────────────────────────┐
│                Without DNS                               │
│                                                         │
│  Pod A                          Pod B (backend)         │
│  ┌─────────┐                   ┌─────────┐             │
│  │ curl    │──── 10.96.23.45 ──►│ :3000   │             │
│  │ http:// │                   │         │             │
│  │ ???     │  What IP?         │         │             │
│  └─────────┘  It changed!      └─────────┘             │
│                                                         │
│  Problems:                                              │
│  1. How does Pod A know the IP?                         │
│  2. What if the Service is recreated with a new IP?     │
│  3. What about cross-namespace services?                │
│  4. What about headless services with multiple Pods?    │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│                With DNS                                  │
│                                                         │
│  Pod A                          Pod B (backend)         │
│  ┌─────────┐    DNS query      ┌─────────┐             │
│  │ curl    │───"backend-svc"──►│ CoreDNS │             │
│  │ http:// │◄──10.96.23.45────│         │             │
│  │ backend │                   └─────────┘             │
│  │ -svc    │──── 10.96.23.45 ──►┌─────────┐            │
│  └─────────┘                   │ :3000   │             │
│                                └─────────┘             │
│                                                         │
│  The name "backend-svc" always resolves to the right IP │
└─────────────────────────────────────────────────────────┘
```

---

## 26.2 Kubernetes DNS Specification

### 26.2.1 Service DNS Records

Every Kubernetes Service gets DNS records following this pattern:

```
<service-name>.<namespace>.svc.<cluster-domain>
```

The default cluster domain is `cluster.local`.

#### ClusterIP Service — A Record

```
# ── A record for a ClusterIP Service ──────────────────────
# Service: backend-svc in namespace production
# DNS name: backend-svc.production.svc.cluster.local
# Resolves to: ClusterIP (e.g., 10.96.23.45)

$ nslookup backend-svc.production.svc.cluster.local
Server:    10.96.0.10
Address:   10.96.0.10#53

Name:   backend-svc.production.svc.cluster.local
Address: 10.96.23.45
```

#### Headless Service — A Record Returning Pod IPs

A headless Service (`clusterIP: None`) does not get a ClusterIP. Instead, its
DNS A record returns the IP addresses of **all backing Pods**.

```yaml
# ── Headless Service definition ────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: db-headless
  namespace: production
spec:
  clusterIP: None          # This makes it headless
  selector:
    app: database
  ports:
    - port: 5432
      targetPort: 5432
```

```
# ── DNS query for headless service ─────────────────────────
# Returns individual Pod IPs, not a single ClusterIP.
$ nslookup db-headless.production.svc.cluster.local
Server:    10.96.0.10
Address:   10.96.0.10#53

Name:   db-headless.production.svc.cluster.local
Address: 10.244.1.15
Address: 10.244.2.23
Address: 10.244.3.8
```

#### Individual Pod DNS for Headless Services

Each Pod behind a headless Service also gets its own DNS record:

```
<pod-name>.<service-name>.<namespace>.svc.<cluster-domain>
```

This is critical for **StatefulSets**, where each Pod has a stable identity:

```
# ── StatefulSet Pods with headless service "db-headless" ──
# Pod: db-0
db-0.db-headless.production.svc.cluster.local → 10.244.1.15

# Pod: db-1
db-1.db-headless.production.svc.cluster.local → 10.244.2.23

# Pod: db-2
db-2.db-headless.production.svc.cluster.local → 10.244.3.8
```

### 26.2.2 Pod DNS Records

Pods themselves get DNS records, though these are less commonly used:

```
# ── Pod DNS record ─────────────────────────────────────────
# The Pod IP with dots replaced by dashes:
# Pod IP: 10.244.1.15
# DNS: 10-244-1-15.production.pod.cluster.local

$ nslookup 10-244-1-15.production.pod.cluster.local
Name:   10-244-1-15.production.pod.cluster.local
Address: 10.244.1.15
```

### 26.2.3 SRV Records

SRV records encode port information for named ports:

```
_<port-name>._<protocol>.<service>.<namespace>.svc.<cluster-domain>
```

```yaml
# ── Service with named ports ───────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: production
spec:
  selector:
    app: backend
  ports:
    - name: http          # Port name — used in SRV record
      port: 3000
      protocol: TCP
    - name: metrics       # Another named port
      port: 9090
      protocol: TCP
```

```
# ── SRV record queries ────────────────────────────────────
$ dig SRV _http._tcp.backend-svc.production.svc.cluster.local

;; ANSWER SECTION:
_http._tcp.backend-svc.production.svc.cluster.local. 30 IN SRV \
    0 100 3000 backend-svc.production.svc.cluster.local.

$ dig SRV _metrics._tcp.backend-svc.production.svc.cluster.local

;; ANSWER SECTION:
_metrics._tcp.backend-svc.production.svc.cluster.local. 30 IN SRV \
    0 100 9090 backend-svc.production.svc.cluster.local.
```

SRV records are used by service meshes and some applications for **port
discovery** — learning which port a service is on without hardcoding it.

### 26.2.4 Hands-On: Querying Each DNS Record Type

```bash
# ── Deploy a dnsutils pod for querying ─────────────────────
kubectl run dnsutils --image=registry.k8s.io/e2e-test-images/jessie-dnsutils:1.3 \
  --restart=Never -- sleep infinity

kubectl wait --for=condition=ready pod/dnsutils

# ── A record for ClusterIP Service ────────────────────────
kubectl exec dnsutils -- nslookup backend-svc.production.svc.cluster.local

# ── A records for Headless Service (returns Pod IPs) ──────
kubectl exec dnsutils -- nslookup db-headless.production.svc.cluster.local

# ── Individual Pod DNS (StatefulSet) ──────────────────────
kubectl exec dnsutils -- nslookup db-0.db-headless.production.svc.cluster.local

# ── Pod DNS record ────────────────────────────────────────
kubectl exec dnsutils -- nslookup 10-244-1-15.production.pod.cluster.local

# ── SRV records ───────────────────────────────────────────
kubectl exec dnsutils -- dig SRV _http._tcp.backend-svc.production.svc.cluster.local

# ── Reverse lookup (PTR) ──────────────────────────────────
kubectl exec dnsutils -- nslookup 10.96.23.45

# ── External DNS resolution (via upstream forwarding) ─────
kubectl exec dnsutils -- nslookup google.com

# ── Clean up ──────────────────────────────────────────────
kubectl delete pod dnsutils
```

---

## 26.3 CoreDNS — The DNS Server

### 26.3.1 What Is CoreDNS?

CoreDNS is a **flexible, extensible DNS server** written in Go. It became the
default DNS server in Kubernetes in version 1.13, replacing kube-dns.

Key characteristics:

- **Plugin-based architecture** — every feature is a plugin.
- **Corefile** — a single configuration file defines the entire behavior.
- **Written in Go** — single static binary, no dependencies.
- **Lightweight** — typical memory usage is 30-70 MB for small clusters.
- **CNCF graduated project** — same maturity level as Kubernetes itself.

### 26.3.2 CoreDNS Deployment in Kubernetes

CoreDNS runs as a **Deployment** in the `kube-system` namespace with a
**Service** of type ClusterIP (typically `10.96.0.10`).

```
┌─────────────────────────────────────────────────────────┐
│                    kube-system namespace                  │
│                                                         │
│  ┌──────────────────────────────────────────┐           │
│  │         CoreDNS Deployment               │           │
│  │  ┌─────────────┐  ┌─────────────┐       │           │
│  │  │ coredns-    │  │ coredns-    │       │           │
│  │  │ abc123      │  │ def456      │       │           │
│  │  │ (replica 1) │  │ (replica 2) │       │           │
│  │  └─────────────┘  └─────────────┘       │           │
│  └──────────────────────────────────────────┘           │
│                       ▲                                 │
│                       │ ClusterIP: 10.96.0.10           │
│  ┌────────────────────┴───────────────────┐             │
│  │         kube-dns Service               │             │
│  │  (name: kube-dns, port 53 UDP/TCP)     │             │
│  └────────────────────────────────────────┘             │
│                       ▲                                 │
│                       │                                 │
│  Every Pod's /etc/resolv.conf points here               │
└─────────────────────────────────────────────────────────┘
```

```bash
# ── Inspect CoreDNS components ─────────────────────────────
# The Deployment
kubectl -n kube-system get deploy coredns

# The Pods
kubectl -n kube-system get pods -l k8s-app=kube-dns

# The Service (note the ClusterIP)
kubectl -n kube-system get svc kube-dns

# The ConfigMap (contains the Corefile)
kubectl -n kube-system get configmap coredns -o yaml
```

### 26.3.3 The Corefile — Configuration File

The Corefile is stored in a ConfigMap and mounted into the CoreDNS Pods. It
defines **server blocks** — each block handles a specific DNS zone.

```
┌─────────────────────────────────────────────────────────┐
│                    Corefile Structure                     │
│                                                         │
│  .:53 {                    ← Server block for all zones │
│      plugin1               ← Plugin chain              │
│      plugin2                                            │
│      plugin3                                            │
│  }                                                      │
│                                                         │
│  example.com:53 {          ← Server block for specific  │
│      plugin1                  zone (optional)           │
│      plugin2                                            │
│  }                                                      │
└─────────────────────────────────────────────────────────┘
```

Plugins execute in the **order they appear in the Corefile** (with some
exceptions for certain plugin types). Each plugin either handles the query
and returns a response, or passes it to the next plugin.

---

## 26.4 CoreDNS Plugins Deep Dive

### 26.4.1 Annotated Default Corefile

Here is the default Corefile that ships with Kubernetes, annotated with
explanations of every plugin:

```
# ──────────────────────────────────────────────────────────
# CoreDNS Default Corefile — Annotated
# ──────────────────────────────────────────────────────────
# This server block handles ALL DNS zones on port 53.
.:53 {
    # ── errors ─────────────────────────────────────────────
    # Logs errors to stdout. Essential for debugging.
    # Without this, DNS failures are silent.
    errors

    # ── health ─────────────────────────────────────────────
    # Exposes an HTTP health check endpoint at :8080/health
    # Used by Kubernetes liveness probe.
    # The "lameduck 5s" option makes CoreDNS report unhealthy
    # for 5 seconds before shutting down, allowing in-flight
    # requests to complete.
    health {
        lameduck 5s
    }

    # ── ready ──────────────────────────────────────────────
    # Exposes a readiness endpoint at :8181/ready
    # Returns 200 when all plugins that can report readiness
    # have done so. Used by Kubernetes readiness probe.
    ready

    # ── kubernetes ─────────────────────────────────────────
    # THE core plugin. Watches the Kubernetes API for Services
    # and Endpoints, and generates DNS records from them.
    #
    # "cluster.local" — the DNS zone this plugin serves.
    # "in-addr.arpa ip6.arpa" — reverse DNS zones.
    #
    # Options:
    #   pods insecure — generate Pod DNS records without
    #     verifying the Pod exists (faster but less secure).
    #   fallthrough in-addr.arpa ip6.arpa — if this plugin
    #     can't answer a reverse lookup, pass to next plugin.
    #   ttl 30 — DNS record TTL is 30 seconds.
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }

    # ── prometheus ─────────────────────────────────────────
    # Exposes Prometheus metrics at :9153/metrics
    # Key metrics:
    #   coredns_dns_requests_total — total queries
    #   coredns_dns_responses_total — total responses by rcode
    #   coredns_dns_request_duration_seconds — query latency
    #   coredns_cache_hits_total — cache hit rate
    prometheus :9153

    # ── forward ────────────────────────────────────────────
    # For any query that the kubernetes plugin can't answer
    # (i.e., not *.cluster.local), forward to upstream DNS.
    # "/etc/resolv.conf" means use the node's DNS servers.
    #
    # In cloud environments, this resolves to the cloud
    # provider's DNS (e.g., 169.254.169.253 on AWS).
    forward . /etc/resolv.conf {
        max_concurrent 1000
    }

    # ── cache ──────────────────────────────────────────────
    # Cache DNS responses for 30 seconds.
    # Dramatically reduces load on upstream DNS servers and
    # the Kubernetes API (for internal queries).
    # "30" is the max TTL for positive responses.
    cache 30

    # ── loop ───────────────────────────────────────────────
    # Detects forwarding loops. If CoreDNS detects that it
    # is forwarding to itself (which causes infinite loops),
    # it logs an error and stops.
    # This happens when /etc/resolv.conf on the node points
    # back to CoreDNS (common with systemd-resolved).
    loop

    # ── reload ─────────────────────────────────────────────
    # Watches the Corefile for changes and reloads CoreDNS
    # automatically. The ConfigMap mount triggers this.
    # Default check interval: 30 seconds.
    reload

    # ── loadbalance ────────────────────────────────────────
    # Randomizes the order of A/AAAA records in responses.
    # Provides round-robin DNS load balancing for headless
    # services (where multiple Pod IPs are returned).
    loadbalance
}
```

### 26.4.2 The kubernetes Plugin — Deep Dive

The `kubernetes` plugin is the heart of Kubernetes DNS. Let's examine its
behavior in detail.

```
┌─────────────────────────────────────────────────────────┐
│              kubernetes Plugin Internals                  │
│                                                         │
│  ┌─────────────┐         ┌──────────────┐              │
│  │ API Server  │◄───────►│  kubernetes  │              │
│  │             │  Watch   │  plugin      │              │
│  │  Services   │         │              │              │
│  │  Endpoints  │         │  In-memory   │              │
│  │  Pods       │         │  index of    │              │
│  │             │         │  DNS records │              │
│  └─────────────┘         └──────┬───────┘              │
│                                 │                       │
│                          DNS query arrives               │
│                                 │                       │
│                    ┌────────────┴────────────┐          │
│                    │  Lookup in index        │          │
│                    │                         │          │
│                    │  Match found?           │          │
│                    │  ├── Yes → Return record│          │
│                    │  └── No  → NXDOMAIN or │          │
│                    │           fallthrough   │          │
│                    └─────────────────────────┘          │
└─────────────────────────────────────────────────────────┘
```

**Record generation rules:**

| Kubernetes Resource | DNS Record Type | DNS Name Pattern |
|--------------------|----------------|-----------------|
| ClusterIP Service | A / AAAA | `<svc>.<ns>.svc.cluster.local` |
| Headless Service | A (multiple) | `<svc>.<ns>.svc.cluster.local` |
| Headless Service Pod | A / AAAA | `<pod>.<svc>.<ns>.svc.cluster.local` |
| Named Port | SRV | `_<port>._<proto>.<svc>.<ns>.svc.cluster.local` |
| Pod (auto) | A / AAAA | `<pod-ip-dashed>.<ns>.pod.cluster.local` |
| ExternalName Service | CNAME | `<svc>.<ns>.svc.cluster.local` |

**Pod modes:**

```
# ── pods insecure ──────────────────────────────────────────
# Generates Pod DNS records for any IP in the Pod CIDR.
# Does NOT verify the Pod actually exists.
# Fast but allows spoofing.
kubernetes cluster.local {
    pods insecure
}

# ── pods verified ──────────────────────────────────────────
# Only generates Pod DNS records for IPs that match a
# real Pod in the API. Slower but prevents spoofing.
kubernetes cluster.local {
    pods verified
}

# ── pods disabled ──────────────────────────────────────────
# Does not generate Pod DNS records at all.
# Most secure — Pods are only reachable via Service names.
kubernetes cluster.local {
    pods disabled
}
```

### 26.4.3 The forward Plugin

```
# ── Forward to specific upstream servers ───────────────────
# Instead of /etc/resolv.conf, you can specify servers:
forward . 8.8.8.8 8.8.4.4 {
    max_concurrent 1000    # Max concurrent upstream queries
    policy round_robin     # Load balance across upstreams
    health_check 5s        # Check upstream health every 5s
    expire 10s             # Expire cached connections after 10s
}

# ── Forward specific zones to different servers ────────────
# This is useful for split-horizon DNS.
# Corporate DNS for internal domains:
example.com:53 {
    forward . 10.0.0.1 10.0.0.2
}

# Everything else goes to public DNS:
.:53 {
    forward . 8.8.8.8 8.8.4.4
}
```

### 26.4.4 The cache Plugin

```
# ── Cache configuration options ────────────────────────────
cache {
    success 9984 30        # Cache up to 9984 positive responses
                           # for max 30 seconds
    denial 9984 5          # Cache up to 9984 negative (NXDOMAIN)
                           # responses for max 5 seconds
    prefetch 10 60s 10%    # Prefetch popular entries:
                           #   - if queried 10+ times
                           #   - within 60s of TTL expiry
                           #   - for top 10% of queries
    serve_stale            # Serve stale cache entries if
                           # upstream is unreachable
}
```

### 26.4.5 Other Important Plugins

```
# ── log ────────────────────────────────────────────────────
# Logs every DNS query. Useful for debugging but very verbose
# in production. Enable temporarily when troubleshooting.
log

# ── template ───────────────────────────────────────────────
# Generate dynamic DNS responses from templates.
# Useful for custom records.
template IN A example.com {
    match .*\.example\.com
    answer "{{ .Name }} 60 IN A 10.0.0.1"
}

# ── hosts ──────────────────────────────────────────────────
# Serve records from a hosts-format file.
# Useful for manual DNS entries.
hosts /etc/coredns/custom-hosts {
    10.0.0.100 internal-api.example.com
    10.0.0.101 internal-db.example.com
    fallthrough
}

# ── rewrite ────────────────────────────────────────────────
# Rewrite DNS queries before processing.
# Useful for aliasing or migration.
rewrite name old-service.default.svc.cluster.local new-service.default.svc.cluster.local
```

---

## 26.5 How DNS Resolution Works in a Pod

### 26.5.1 /etc/resolv.conf — The Starting Point

Every Pod in Kubernetes has a `/etc/resolv.conf` file that the kubelet
generates. This file configures how DNS resolution works:

```bash
# ── Typical /etc/resolv.conf inside a Kubernetes Pod ──────
# Generated by kubelet based on the Pod's dnsPolicy.

nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

Let's break down each line:

| Line | Meaning |
|------|---------|
| `nameserver 10.96.0.10` | DNS server IP — this is the `kube-dns` Service ClusterIP |
| `search default.svc.cluster.local ...` | Search domains appended to unqualified names |
| `options ndots:5` | If a name has fewer than 5 dots, try search domains first |

### 26.5.2 Hands-On: Examining /etc/resolv.conf

```bash
# ── Check resolv.conf in different namespaces ──────────────
# In 'default' namespace:
kubectl run test --image=busybox --rm -it --restart=Never -- cat /etc/resolv.conf
# Output:
# nameserver 10.96.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5

# In 'production' namespace:
kubectl -n production run test --image=busybox --rm -it --restart=Never -- cat /etc/resolv.conf
# Output:
# nameserver 10.96.0.10
# search production.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
#        ↑ namespace changes in search domain!
```

### 26.5.3 The DNS Query Flow

When an application inside a Pod makes a DNS query, here's the complete path:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    DNS Query Flow                                    │
│                                                                     │
│  Application calls: getaddrinfo("backend-svc")                      │
│         │                                                           │
│         ▼                                                           │
│  ┌─────────────────────────────────────────┐                       │
│  │  glibc / musl resolver                  │                       │
│  │  Reads /etc/resolv.conf                 │                       │
│  │                                         │                       │
│  │  "backend-svc" has 0 dots.              │                       │
│  │  ndots=5, so 0 < 5 → try search        │                       │
│  │  domains FIRST.                         │                       │
│  └──────────────┬──────────────────────────┘                       │
│                 │                                                   │
│    ┌────────────┴─────────────────────────────────────┐             │
│    │  Query 1: backend-svc.default.svc.cluster.local  │             │
│    │  Query 2: backend-svc.svc.cluster.local          │  (if Q1     │
│    │  Query 3: backend-svc.cluster.local              │   fails)    │
│    │  Query 4: backend-svc.                           │             │
│    └────────────┬─────────────────────────────────────┘             │
│                 │                                                   │
│                 ▼                                                   │
│  ┌─────────────────────────────────────────┐                       │
│  │  CoreDNS (10.96.0.10:53)               │                       │
│  │                                         │                       │
│  │  Query 1 hits kubernetes plugin:        │                       │
│  │  → Found! Returns ClusterIP 10.96.23.45│                       │
│  └──────────────┬──────────────────────────┘                       │
│                 │                                                   │
│                 ▼                                                   │
│  Application receives: 10.96.23.45                                 │
│  Connects to backend-svc via ClusterIP                             │
└─────────────────────────────────────────────────────────────────────┘
```

### 26.5.4 The ndots Problem — Why Simple Names Cause Multiple Queries

The `ndots:5` setting means: "If the queried name has **fewer than 5 dots**,
append search domains before trying the name as-is."

This creates a significant problem for **external DNS names**:

```
┌─────────────────────────────────────────────────────────────────────┐
│  Application queries: "api.stripe.com" (2 dots, less than 5)       │
│                                                                     │
│  Because 2 < ndots(5), search domains are tried FIRST:              │
│                                                                     │
│  Query 1: api.stripe.com.default.svc.cluster.local → NXDOMAIN      │
│  Query 2: api.stripe.com.svc.cluster.local         → NXDOMAIN      │
│  Query 3: api.stripe.com.cluster.local              → NXDOMAIN      │
│  Query 4: api.stripe.com.                           → ✅ RESOLVED!  │
│                                                                     │
│  Result: 4 DNS queries for a single external name!                  │
│  3 of them are wasted NXDOMAIN responses.                           │
│                                                                     │
│  At scale (thousands of Pods, frequent external calls),             │
│  this DOUBLES or TRIPLES DNS traffic to CoreDNS.                    │
└─────────────────────────────────────────────────────────────────────┘
```

**Why is ndots set to 5?**

Kubernetes service names can have up to 4 dots:

```
_http._tcp.backend-svc.production.svc.cluster.local
   1    2          3          4
```text

With `ndots:5`, a name like `backend-svc` (0 dots) correctly triggers the
search domain expansion, eventually resolving to
`backend-svc.default.svc.cluster.local`.

**Mitigation strategies:**

1. **Use FQDNs for external names** — append a trailing dot:
   ```go
// ──────────────────────────────────────────────────────
// Using a trailing dot prevents search domain expansion.
// "api.stripe.com." is treated as fully qualified.
// ──────────────────────────────────────────────────────
url := "https://api.stripe.com./v1/charges"

```

2. **Lower ndots** — set `ndots:2` in Pod dnsConfig (see Section 26.6).
This works if you always use short service names.

3. **Use the autopath plugin** — CoreDNS can optimize query expansion
server-side (see Section 26.7.2).

4. **Use NodeLocal DNSCache** — caches NXDOMAIN responses locally,
eliminating network round-trips (see Section 26.7.3).

### 26.5.5 Hands-On: Tracing DNS Query Expansion

```bash
# ── Deploy a dnsutils pod ──────────────────────────────────
kubectl run dnsutils --image=nicolaka/netshoot --restart=Never -- sleep 3600

# ── Enable CoreDNS query logging ──────────────────────────
# Edit the CoreDNS ConfigMap to add the 'log' plugin:
kubectl -n kube-system edit configmap coredns
# Add "log" on its own line inside the .:53 {} block.
# Then restart CoreDNS:
kubectl -n kube-system rollout restart deploy coredns

# ── Make a DNS query from the test Pod ─────────────────────
kubectl exec dnsutils -- nslookup api.stripe.com

# ── Watch CoreDNS logs ─────────────────────────────────────
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=20

# ── Expected output shows multiple queries ─────────────────
# [INFO] 10.244.1.50:42315 - 12345 "A IN api.stripe.com.default.svc.cluster.local. udp ..."
# [INFO] 10.244.1.50:42315 - 12346 "A IN api.stripe.com.svc.cluster.local. udp ..."
# [INFO] 10.244.1.50:42315 - 12347 "A IN api.stripe.com.cluster.local. udp ..."
# [INFO] 10.244.1.50:42315 - 12348 "A IN api.stripe.com. udp ..."
# ← Four queries for one name!

# ── Now try with a trailing dot (FQDN) ────────────────────
kubectl exec dnsutils -- nslookup api.stripe.com.

# ── CoreDNS logs show only ONE query ──────────────────────
# [INFO] 10.244.1.50:42315 - 12349 "A IN api.stripe.com. udp ..."

# ── Clean up ──────────────────────────────────────────────
kubectl delete pod dnsutils
# Remember to remove 'log' from Corefile if you don't want
# verbose logging in production!
```

### 26.5.6 Search Domains Explained

The search domains in `/etc/resolv.conf` allow short names to work:

```
search default.svc.cluster.local svc.cluster.local cluster.local
```

| Short Name | Expanded Attempts |
|-----------|-------------------|
| `backend-svc` | 1. `backend-svc.default.svc.cluster.local` ← hits! |
| | 2. `backend-svc.svc.cluster.local` |
| | 3. `backend-svc.cluster.local` |
| | 4. `backend-svc.` |
| `backend-svc.production` | 1. `backend-svc.production.default.svc.cluster.local` |
| | 2. `backend-svc.production.svc.cluster.local` ← hits! |
| | 3. `backend-svc.production.cluster.local` |
| | 4. `backend-svc.production.` |

This is why a Pod in the `default` namespace can reach a service in
`production` by using `backend-svc.production` — the search domain
`.svc.cluster.local` is appended, giving the correct FQDN.

---

## 26.6 Pod DNS Policies

### 26.6.1 The Four DNS Policies

The `dnsPolicy` field in the Pod spec controls how `/etc/resolv.conf` is
generated:

#### ClusterFirst (Default)

```yaml
# ── ClusterFirst — the default ─────────────────────────────
# Sends all DNS queries to CoreDNS first.
# Non-cluster names are forwarded upstream by CoreDNS.
# This is correct for 99% of Pods.
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  dnsPolicy: ClusterFirst    # This is the default
  containers:
    - name: app
      image: nginx
```

Resulting `/etc/resolv.conf`:
```
nameserver 10.96.0.10
search default.svc.cluster.local svc.cluster.local cluster.local
options ndots:5
```

#### Default (Inherit from Node)

```yaml
# ── Default — inherit the node's resolv.conf ───────────────
# The Pod gets the same DNS config as the underlying node.
# Kubernetes service names will NOT resolve!
# Only use this for Pods that need to match node DNS behavior.
apiVersion: v1
kind: Pod
metadata:
  name: node-dns-pod
spec:
  dnsPolicy: Default         # Inherit node's resolv.conf
  containers:
    - name: app
      image: nginx
```

Resulting `/etc/resolv.conf` (depends on node):
```
nameserver 169.254.169.253   # AWS VPC DNS, for example
search ec2.internal
```

> **Warning**: With `dnsPolicy: Default`, the Pod cannot resolve Kubernetes
> service names like `backend-svc.production.svc.cluster.local`.

#### ClusterFirstWithHostNet

```yaml
# ── ClusterFirstWithHostNet ────────────────────────────────
# For Pods that use hostNetwork: true.
# Without this, hostNetwork Pods inherit the node's DNS
# (like dnsPolicy: Default).
# With this, they use CoreDNS despite using the host network.
apiVersion: v1
kind: Pod
metadata:
  name: host-network-pod
spec:
  hostNetwork: true
  dnsPolicy: ClusterFirstWithHostNet   # Use CoreDNS even with hostNetwork
  containers:
    - name: app
      image: nginx
```

#### None (Custom DNS)

```yaml
# ── None — fully custom DNS configuration ──────────────────
# The Pod's resolv.conf is ENTIRELY controlled by dnsConfig.
# Kubernetes does not inject any DNS settings.
# Use when you need complete control over DNS resolution.
apiVersion: v1
kind: Pod
metadata:
  name: custom-dns-pod
spec:
  dnsPolicy: None
  dnsConfig:
    # Custom nameservers
    nameservers:
      - 10.0.0.1          # Corporate DNS server
      - 8.8.8.8           # Google DNS as fallback
    # Custom search domains
    searches:
      - mycompany.internal
      - svc.cluster.local
    # Custom options
    options:
      - name: ndots
        value: "2"         # Lower ndots for fewer queries
      - name: timeout
        value: "3"         # 3 second timeout
      - name: attempts
        value: "2"         # 2 retry attempts
  containers:
    - name: app
      image: nginx
```

Resulting `/etc/resolv.conf`:
```
nameserver 10.0.0.1
nameserver 8.8.8.8
search mycompany.internal svc.cluster.local
options ndots:2 timeout:3 attempts:2
```

### 26.6.2 Hands-On: Lowering ndots for Performance

```yaml
# ──────────────────────────────────────────────────────────
# Optimized DNS config for Pods that make many external calls
# ──────────────────────────────────────────────────────────
# This Pod uses ClusterFirst but overrides ndots to 2.
# This means names with 2+ dots (like "api.stripe.com")
# are queried directly first, avoiding 3 wasted NXDOMAIN
# queries through search domains.
#
# Trade-off: SRV records (which have 4 dots) won't resolve
# with short names. You must use FQDNs for SRV queries.
# ──────────────────────────────────────────────────────────
apiVersion: v1
kind: Pod
metadata:
  name: optimized-dns-pod
spec:
  dnsPolicy: ClusterFirst
  dnsConfig:
    options:
      - name: ndots
        value: "2"
  containers:
    - name: app
      image: nginx
```

---

## 26.7 CoreDNS Performance and Tuning

### 26.7.1 Caching Configuration

The default cache of 30 seconds is conservative. For production clusters with
heavy DNS traffic:

```
# ── Aggressive caching configuration ──────────────────────
.:53 {
    errors
    health
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    prometheus :9153

    # Aggressive cache: larger size, longer TTL, prefetch
    cache {
        success 30000 60     # 30K entries, 60s max TTL
        denial 10000 10      # 10K NXDOMAIN entries, 10s TTL
        prefetch 5 30s 20%   # Prefetch if 5+ queries in 30s
        serve_stale          # Serve stale on upstream failure
    }

    forward . /etc/resolv.conf {
        max_concurrent 2000
    }
    loop
    reload
    loadbalance
}
```

### 26.7.2 Autopath Plugin — Server-Side Query Optimization

The `autopath` plugin moves search domain expansion from the client to CoreDNS
itself, reducing the number of DNS packets on the network:

```
# ── With autopath ──────────────────────────────────────────
# Client queries "backend-svc" once.
# CoreDNS tries search domains INTERNALLY:
#   backend-svc.default.svc.cluster.local → found!
# Returns the answer in a SINGLE response.
# Without autopath: 4 round-trips. With autopath: 1.
.:53 {
    errors
    health
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
        pods insecure
        fallthrough in-addr.arpa ip6.arpa
        ttl 30
    }
    autopath @kubernetes    # Use kubernetes plugin for path info
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}
```

```
Without autopath:                    With autopath:
Client        CoreDNS               Client        CoreDNS
  |──Q1─────────►|                    |──Q1─────────►|
  |◄──NXDOMAIN───|                    |              |─┐ internal:
  |──Q2─────────►|                    |              | | try search
  |◄──NXDOMAIN───|                    |              | | domains
  |──Q3─────────►|                    |              |◄┘
  |◄──NXDOMAIN───|                    |◄──ANSWER─────|
  |──Q4─────────►|
  |◄──ANSWER─────|
  4 round-trips                       1 round-trip
```

> **Caveat**: The `autopath` plugin requires the `pods verified` or
> `pods insecure` mode to identify which Pod sent the query (and thus which
> namespace's search domains to use).

### 26.7.3 NodeLocal DNSCache

NodeLocal DNSCache runs a DNS caching agent on **every node** as a DaemonSet.
Pods query the local agent instead of CoreDNS directly.

```
┌──────────────────────────────────────────────────────────┐
│                      Without NodeLocal DNS                │
│                                                          │
│  ┌────────┐   ┌────────┐   ┌────────┐                  │
│  │ Pod A  │   │ Pod B  │   │ Pod C  │   ← Node 1       │
│  └───┬────┘   └───┬────┘   └───┬────┘                  │
│      │            │            │                        │
│      └────────────┼────────────┘                        │
│                   │  All queries go over the network    │
│                   ▼  to CoreDNS ClusterIP               │
│           ┌──────────────┐                              │
│           │   CoreDNS    │   ← Could be on another node│
│           └──────────────┘                              │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                      With NodeLocal DNS                   │
│                                                          │
│  ┌────────┐   ┌────────┐   ┌────────┐                  │
│  │ Pod A  │   │ Pod B  │   │ Pod C  │   ← Node 1       │
│  └───┬────┘   └───┬────┘   └───┬────┘                  │
│      │            │            │                        │
│      └────────────┼────────────┘                        │
│                   │  Queries go to LOCAL cache           │
│                   ▼  (link-local IP 169.254.20.10)      │
│           ┌──────────────┐                              │
│           │  node-local  │   ← Same node! No network    │
│           │  dns-cache   │     hop. Sub-millisecond.    │
│           └──────┬───────┘                              │
│                  │  Cache miss only                     │
│                  ▼                                      │
│           ┌──────────────┐                              │
│           │   CoreDNS    │                              │
│           └──────────────┘                              │
└──────────────────────────────────────────────────────────┘
```

Benefits:
- **Latency**: Local cache avoids network hop — sub-millisecond responses.
- **Reliability**: If CoreDNS Pods are temporarily unavailable, cached entries
  still resolve.
- **Scalability**: Reduces load on CoreDNS by ~80% in typical clusters.
- **Conntrack**: Avoids conntrack table issues with UDP DNS on busy nodes.

### 26.7.4 Hands-On: Installing NodeLocal DNSCache

```bash
# ── Step 1: Get the CoreDNS ClusterIP ─────────────────────
COREDNS_IP=$(kubectl -n kube-system get svc kube-dns -o jsonpath='{.spec.clusterIP}')
echo "CoreDNS ClusterIP: $COREDNS_IP"

# ── Step 2: Download and customize the manifest ───────────
# The official manifest is templated with __PILLAR__ variables.
curl -sL https://raw.githubusercontent.com/kubernetes/kubernetes/master/cluster/addons/dns/nodelocaldns/nodelocaldns.yaml \
  | sed "s/__PILLAR__DNS__SERVER__/${COREDNS_IP}/g" \
  | sed "s/__PILLAR__LOCAL__DNS__/169.254.20.10/g" \
  | sed "s/__PILLAR__DNS__DOMAIN__/cluster.local/g" \
  | sed "s/__PILLAR__CLUSTER__DNS__/${COREDNS_IP}/g" \
  | sed "s/__PILLAR__UPSTREAM__SERVERS__/\/etc\/resolv.conf/g" \
  > nodelocaldns.yaml

# ── Step 3: Apply the manifest ─────────────────────────────
kubectl apply -f nodelocaldns.yaml

# ── Step 4: Verify DaemonSet is running ────────────────────
kubectl -n kube-system get ds node-local-dns
kubectl -n kube-system get pods -l k8s-app=node-local-dns

# ── Step 5: Verify it's working ───────────────────────────
# From a Pod, the local cache responds:
kubectl run test --image=nicolaka/netshoot --rm -it --restart=Never -- \
  dig @169.254.20.10 backend-svc.default.svc.cluster.local
```

### 26.7.5 Hands-On: Measuring DNS Latency

```bash
# ── Before NodeLocal DNSCache ──────────────────────────────
# Measure baseline DNS latency by querying CoreDNS directly.
kubectl run bench --image=nicolaka/netshoot --rm -it --restart=Never -- \
  sh -c "
    echo '=== DNS Latency to CoreDNS ==='
    for i in \$(seq 1 10); do
      dig @10.96.0.10 backend-svc.default.svc.cluster.local \
        +noall +stats | grep 'Query time'
    done
  "

# ── After NodeLocal DNSCache ──────────────────────────────
# Measure latency to the local cache.
kubectl run bench --image=nicolaka/netshoot --rm -it --restart=Never -- \
  sh -c "
    echo '=== DNS Latency to NodeLocal Cache ==='
    for i in \$(seq 1 10); do
      dig @169.254.20.10 backend-svc.default.svc.cluster.local \
        +noall +stats | grep 'Query time'
    done
  "

# ── Typical results ───────────────────────────────────────
# CoreDNS (remote):     Query time: 1-5 msec
# NodeLocal (cached):   Query time: 0 msec (sub-millisecond)
```

---

## 26.8 DNS Debugging

### 26.8.1 Common DNS Issues

| Issue | Symptom | Root Cause |
|-------|---------|-----------|
| DNS timeout | Connection hangs, then fails after 5s | CoreDNS down, network policy blocks port 53, conntrack table full |
| NXDOMAIN | "server can't find..." | Wrong name, wrong namespace, service doesn't exist |
| Slow resolution | First request takes 5-10s | ndots:5 causing extra queries, no caching |
| Intermittent failures | Works sometimes, fails other times | CoreDNS OOM, UDP packet loss, iptables conntrack race |
| External names fail | Internal names work, external don't | Upstream DNS misconfigured, forward plugin error |
| Resolution after pod restart | Old Pod IPs returned briefly | DNS cache TTL, stale endpoints |

### 26.8.2 Debugging Tools

```bash
# ── Deploy a comprehensive debug pod ──────────────────────
kubectl run debug-dns --image=nicolaka/netshoot --restart=Never -- sleep 3600
kubectl wait --for=condition=ready pod/debug-dns
```

```bash
# ── Tool 1: nslookup — quick name resolution check ────────
kubectl exec debug-dns -- nslookup backend-svc
kubectl exec debug-dns -- nslookup backend-svc.production.svc.cluster.local
kubectl exec debug-dns -- nslookup google.com

# ── Tool 2: dig — detailed DNS query info ─────────────────
# Shows query time, server used, full answer section.
kubectl exec debug-dns -- dig backend-svc.default.svc.cluster.local

# With specific server:
kubectl exec debug-dns -- dig @10.96.0.10 backend-svc.default.svc.cluster.local

# SRV record:
kubectl exec debug-dns -- dig SRV _http._tcp.backend-svc.default.svc.cluster.local

# Trace (shows full resolution path):
kubectl exec debug-dns -- dig +trace google.com

# ── Tool 3: Check resolv.conf ─────────────────────────────
kubectl exec debug-dns -- cat /etc/resolv.conf

# ── Tool 4: Test DNS response time ────────────────────────
kubectl exec debug-dns -- sh -c "
  time nslookup backend-svc > /dev/null 2>&1
"
```

### 26.8.3 CoreDNS Logs

```bash
# ── Check CoreDNS Pod status ──────────────────────────────
kubectl -n kube-system get pods -l k8s-app=kube-dns
# Look for: Running, Restarts (OOMKills cause restarts)

# ── Check CoreDNS logs for errors ──────────────────────────
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=50

# ── Common error patterns ──────────────────────────────────
# "i/o timeout" — upstream DNS is unreachable
# "REFUSED" — query rejected by upstream
# "SERVFAIL" — upstream server failure
# "plugin/loop: Loop ... detected" — forwarding loop!

# ── Enable query logging temporarily ──────────────────────
# Edit the CoreDNS ConfigMap:
kubectl -n kube-system edit configmap coredns
# Add "log" inside the .:53 {} block, then:
kubectl -n kube-system rollout restart deploy coredns

# ── Watch logs in real-time ────────────────────────────────
kubectl -n kube-system logs -f -l k8s-app=kube-dns
```

### 26.8.4 Hands-On: Systematic DNS Debugging Workflow

```bash
#!/bin/bash
# ──────────────────────────────────────────────────────────
# DNS Debugging Workflow
# ──────────────────────────────────────────────────────────
# Usage: ./debug-dns.sh <service-name> <namespace>
# Runs through a systematic checklist to identify DNS issues.
# ──────────────────────────────────────────────────────────

SVC=${1:-backend-svc}
NS=${2:-default}
DNS_POD="dns-debug-$$"

echo "=== DNS Debug Workflow for ${SVC} in ${NS} ==="
echo ""

# Step 1: Is CoreDNS running?
echo "--- Step 1: CoreDNS Status ---"
kubectl -n kube-system get pods -l k8s-app=kube-dns -o wide
echo ""

# Step 2: Is the kube-dns Service reachable?
echo "--- Step 2: kube-dns Service ---"
kubectl -n kube-system get svc kube-dns
DNS_IP=$(kubectl -n kube-system get svc kube-dns -o jsonpath='{.spec.clusterIP}')
echo "CoreDNS IP: $DNS_IP"
echo ""

# Step 3: Does the target Service exist?
echo "--- Step 3: Target Service ---"
kubectl -n $NS get svc $SVC 2>&1
echo ""

# Step 4: Does the Service have endpoints?
echo "--- Step 4: Service Endpoints ---"
kubectl -n $NS get endpoints $SVC 2>&1
echo ""

# Step 5: DNS resolution from a debug Pod
echo "--- Step 5: DNS Resolution Test ---"
kubectl run $DNS_POD --image=nicolaka/netshoot --restart=Never \
  --rm -it -- sh -c "
    echo 'resolv.conf:'
    cat /etc/resolv.conf
    echo ''
    echo 'nslookup $SVC:'
    nslookup $SVC
    echo ''
    echo 'nslookup $SVC.$NS.svc.cluster.local:'
    nslookup $SVC.$NS.svc.cluster.local
    echo ''
    echo 'dig @$DNS_IP $SVC.$NS.svc.cluster.local:'
    dig @$DNS_IP $SVC.$NS.svc.cluster.local +short
    echo ''
    echo 'External DNS test:'
    nslookup google.com
" 2>&1

echo ""
echo "--- Step 6: CoreDNS Logs (last 20 lines) ---"
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=20

echo ""
echo "--- Step 7: Network Policies affecting DNS ---"
kubectl -n kube-system get networkpolicies 2>&1
kubectl -n $NS get networkpolicies 2>&1

echo ""
echo "=== Debug Complete ==="
```

### 26.8.5 The Forwarding Loop Problem

One of the most insidious DNS issues is a **forwarding loop**:

```
CoreDNS (forward . /etc/resolv.conf)
    │
    ▼
/etc/resolv.conf → nameserver 10.96.0.10 (kube-dns!)
    │
    ▼
CoreDNS (receives its own query again)
    │
    ▼
Infinite loop → CoreDNS CPU spikes → OOMKill
```

This happens when:
1. The node's `/etc/resolv.conf` points to `10.96.0.10` (the kube-dns ClusterIP).
2. CoreDNS forwards non-cluster queries to `/etc/resolv.conf`.
3. The query comes right back to CoreDNS.

**Fix**: The `loop` plugin detects this and halts CoreDNS with an error.
To resolve, change the `forward` plugin to use explicit upstream servers:

```
# ── Fix: Use explicit upstream DNS servers ─────────────────
forward . 8.8.8.8 8.8.4.4 {
    max_concurrent 1000
}
```

---

## 26.9 ExternalDNS

### 26.9.1 What Is ExternalDNS?

ExternalDNS synchronizes Kubernetes resources (Services, Ingresses) with
**external DNS providers** (Route 53, Cloud DNS, Azure DNS, Cloudflare, etc.).

```
┌─────────────────────────────────────────────────────────┐
│                    ExternalDNS Flow                       │
│                                                         │
│  ┌──────────┐     ┌──────────────┐     ┌─────────────┐ │
│  │ Ingress  │     │ ExternalDNS  │     │ Route 53    │ │
│  │ resource │────►│ controller   │────►│ / Cloud DNS │ │
│  │          │     │ (watches K8s │     │ / Azure DNS │ │
│  │ host:    │     │  resources)  │     │             │ │
│  │ app.co.io│     │              │     │ A record:   │ │
│  │          │     │              │     │ app.co.io → │ │
│  └──────────┘     └──────────────┘     │ 34.56.78.90 │ │
│                                        └─────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### 26.9.2 How ExternalDNS Works

1. You create a Service or Ingress with a hostname annotation.
2. ExternalDNS watches for these resources.
3. When it finds one, it creates/updates DNS records in your provider.
4. When the resource is deleted, ExternalDNS removes the DNS record.

### 26.9.3 Hands-On: ExternalDNS Configuration

```yaml
# ──────────────────────────────────────────────────────────
# ExternalDNS Deployment — AWS Route 53 example
# ──────────────────────────────────────────────────────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: kube-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
        - name: external-dns
          image: registry.k8s.io/external-dns/external-dns:v0.14.0
          args:
            # DNS provider
            - --provider=aws
            # Only manage records in this hosted zone
            - --domain-filter=example.com
            # Which resources to watch
            - --source=ingress
            - --source=service
            # Don't delete records (safety)
            - --policy=upsert-only
            # Use TXT records for ownership tracking
            - --registry=txt
            - --txt-owner-id=my-cluster
          env:
            - name: AWS_DEFAULT_REGION
              value: us-west-2
---
# ── Service Account with IAM role ─────────────────────────
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
  namespace: kube-system
  annotations:
    # For EKS with IRSA (IAM Roles for Service Accounts)
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789:role/external-dns
```

```yaml
# ──────────────────────────────────────────────────────────
# Ingress with ExternalDNS annotation
# ──────────────────────────────────────────────────────────
# ExternalDNS will create an A record in Route 53:
#   app.example.com → <Ingress external IP>
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    # ExternalDNS picks up this annotation
    external-dns.alpha.kubernetes.io/hostname: app.example.com
    # TTL for the DNS record
    external-dns.alpha.kubernetes.io/ttl: "300"
spec:
  ingressClassName: nginx
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
```

```yaml
# ──────────────────────────────────────────────────────────
# LoadBalancer Service with ExternalDNS
# ──────────────────────────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: api-service
  annotations:
    external-dns.alpha.kubernetes.io/hostname: api.example.com
spec:
  type: LoadBalancer
  selector:
    app: api
  ports:
    - port: 443
      targetPort: 8443
```

---

## 26.10 Custom DNS Entries

### 26.10.1 Stub Domains and Upstream Forwarding

For split-horizon DNS where internal domain names resolve differently inside
vs. outside the cluster:

```yaml
# ──────────────────────────────────────────────────────────
# CoreDNS ConfigMap with stub domains
# ──────────────────────────────────────────────────────────
# Forward *.corp.example.com to the corporate DNS server.
# Forward everything else to Google DNS.
# ──────────────────────────────────────────────────────────
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready
        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . 8.8.8.8 8.8.4.4 {
            max_concurrent 1000
        }
        cache 30
        loop
        reload
        loadbalance
    }

    # Stub domain: forward corp.example.com to corporate DNS
    corp.example.com:53 {
        errors
        cache 30
        forward . 10.0.0.1 10.0.0.2
    }

    # Stub domain: forward consul queries to Consul DNS
    consul.local:53 {
        errors
        cache 30
        forward . 10.0.0.10:8600
    }
```

### 26.10.2 Hosts Plugin — Manual DNS Entries

```yaml
# ──────────────────────────────────────────────────────────
# CoreDNS ConfigMap with custom hosts entries
# ──────────────────────────────────────────────────────────
# Use the 'hosts' plugin to serve manual A records for
# internal resources that don't have Kubernetes Services.
# ──────────────────────────────────────────────────────────
apiVersion: v1
kind: ConfigMap
metadata:
  name: coredns
  namespace: kube-system
data:
  Corefile: |
    .:53 {
        errors
        health {
            lameduck 5s
        }
        ready

        # Custom hosts entries — placed BEFORE kubernetes plugin
        # so they take priority for these specific names.
        hosts {
            10.0.0.100 legacy-api.internal
            10.0.0.101 legacy-db.internal
            10.0.0.102 jenkins.internal
            fallthrough    # IMPORTANT: pass unmatched queries
                           # to the next plugin (kubernetes)
        }

        kubernetes cluster.local in-addr.arpa ip6.arpa {
            pods insecure
            fallthrough in-addr.arpa ip6.arpa
            ttl 30
        }
        prometheus :9153
        forward . /etc/resolv.conf
        cache 30
        loop
        reload
        loadbalance
    }
```

### 26.10.3 Hands-On: Adding Custom DNS Entries

```bash
# ── Step 1: Edit the CoreDNS ConfigMap ─────────────────────
kubectl -n kube-system edit configmap coredns

# Add the hosts block inside .:53 {} BEFORE the kubernetes line:
#
#   hosts {
#       10.0.0.100 legacy-api.internal
#       fallthrough
#   }

# ── Step 2: Wait for CoreDNS to reload ────────────────────
# The 'reload' plugin detects ConfigMap changes automatically.
# Wait ~30 seconds, then verify:
kubectl -n kube-system logs -l k8s-app=kube-dns --tail=5
# Look for: "Reloading complete"

# ── Step 3: Test the custom entry ──────────────────────────
kubectl run test --image=busybox --rm -it --restart=Never -- \
  nslookup legacy-api.internal

# Expected output:
# Server:    10.96.0.10
# Address:   10.96.0.10:53
#
# Name:   legacy-api.internal
# Address: 10.0.0.100

# ── Step 4: Verify Kubernetes DNS still works ──────────────
kubectl run test --image=busybox --rm -it --restart=Never -- \
  nslookup kubernetes.default.svc.cluster.local

# The 'fallthrough' in the hosts plugin ensures that
# queries NOT matching custom entries still reach the
# kubernetes plugin.
```

### 26.10.4 Rewrite Plugin — DNS Aliases

```
# ──────────────────────────────────────────────────────────
# Rewrite: Create a DNS alias for a service
# ──────────────────────────────────────────────────────────
# "database" resolves to "postgres-primary.database.svc.cluster.local"
# Useful during migrations or for shorter names.
.:53 {
    rewrite name database.default.svc.cluster.local postgres-primary.database.svc.cluster.local
    # ... other plugins ...
}
```

---

## 26.11 DNS and Network Policies

DNS and Network Policies are deeply intertwined, as covered in Chapter 25. Here
is a summary of the critical interactions:

### 26.11.1 The DNS Egress Requirement

When you apply a default deny egress policy, **DNS breaks immediately**.
Every Pod needs egress access to CoreDNS on port 53.

```yaml
# ── ALWAYS apply this with default deny egress ─────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### 26.11.2 DNS and NodeLocal DNSCache

If you're using NodeLocal DNSCache, the DNS traffic pattern changes:

```
Without NodeLocal:  Pod → CoreDNS Service (10.96.0.10) → CoreDNS Pod
With NodeLocal:     Pod → Local agent (169.254.20.10) → CoreDNS Pod
```

Your Network Policy for DNS egress may need to account for the local cache IP:

```yaml
# ── DNS egress with NodeLocal DNSCache ─────────────────────
egress:
  - ports:
      - protocol: UDP
        port: 53
      - protocol: TCP
        port: 53
    to:
      # CoreDNS Service
      - namespaceSelector:
          matchLabels:
            kubernetes.io/metadata.name: kube-system
        podSelector:
          matchLabels:
            k8s-app: kube-dns
      # NodeLocal DNS (link-local IP)
      - ipBlock:
          cidr: 169.254.20.10/32
```

---

## 26.12 CoreDNS Scaling and High Availability

### 26.12.1 Horizontal Scaling

CoreDNS handles approximately **30,000-50,000 queries per second per core**.
For large clusters:

```bash
# ── Scale CoreDNS replicas ─────────────────────────────────
kubectl -n kube-system scale deploy coredns --replicas=5

# ── Or use HPA for auto-scaling ───────────────────────────
kubectl -n kube-system autoscale deploy coredns \
  --min=2 --max=10 --cpu-percent=70
```

### 26.12.2 Resource Tuning

```yaml
# ── CoreDNS resource requests and limits ───────────────────
# Default is usually very conservative. For large clusters
# (1000+ nodes, 10K+ services), increase:
resources:
  requests:
    cpu: 200m
    memory: 128Mi
  limits:
    cpu: 1000m      # 1 full core per replica
    memory: 256Mi   # Increase for large clusters
```

### 26.12.3 Monitoring CoreDNS

Key Prometheus metrics to monitor:

```
# ── Critical CoreDNS metrics ──────────────────────────────

# Total queries — watch for sudden drops or spikes
rate(coredns_dns_requests_total[5m])

# Error rate — should be near zero
rate(coredns_dns_responses_total{rcode="SERVFAIL"}[5m])

# Query latency — P99 should be < 10ms for internal queries
histogram_quantile(0.99, rate(coredns_dns_request_duration_seconds_bucket[5m]))

# Cache hit rate — should be > 80% for most workloads
rate(coredns_cache_hits_total[5m]) /
(rate(coredns_cache_hits_total[5m]) + rate(coredns_cache_misses_total[5m]))

# Panics — should ALWAYS be zero
coredns_panics_total

# Forward request failures — upstream DNS issues
rate(coredns_forward_responses_total{rcode="SERVFAIL"}[5m])
```

```yaml
# ── Grafana dashboard alert rules ──────────────────────────
# Alert if CoreDNS error rate exceeds 1%
- alert: CoreDNSHighErrorRate
  expr: >
    sum(rate(coredns_dns_responses_total{rcode=~"SERVFAIL|REFUSED"}[5m]))
    / sum(rate(coredns_dns_responses_total[5m])) > 0.01
  for: 5m
  labels:
    severity: warning

# Alert if CoreDNS latency P99 exceeds 50ms
- alert: CoreDNSHighLatency
  expr: >
    histogram_quantile(0.99,
      sum(rate(coredns_dns_request_duration_seconds_bucket[5m])) by (le)
    ) > 0.05
  for: 5m
  labels:
    severity: warning
```

---

## 26.13 ExternalName Services

A special type of Service that returns a CNAME record instead of an A record:

```yaml
# ──────────────────────────────────────────────────────────
# ExternalName Service — DNS CNAME alias
# ──────────────────────────────────────────────────────────
# Querying "my-database.production.svc.cluster.local"
# returns a CNAME to "prod-db.c9y3h2.us-east-1.rds.amazonaws.com"
#
# Use case: Point a Kubernetes service name at an external
# resource (RDS, external API, etc.) without proxying.
# ──────────────────────────────────────────────────────────
apiVersion: v1
kind: Service
metadata:
  name: my-database
  namespace: production
spec:
  type: ExternalName
  externalName: prod-db.c9y3h2.us-east-1.rds.amazonaws.com
```

```bash
# ── DNS resolution of ExternalName ─────────────────────────
$ nslookup my-database.production.svc.cluster.local
# Returns CNAME: prod-db.c9y3h2.us-east-1.rds.amazonaws.com
# Then resolves the CNAME to the actual IP.
```

> **Caveats**: ExternalName services don't support ports (the Port field is
> ignored). They also don't work with kube-proxy — there is no ClusterIP.
> HTTPS certificates may not match the original service name.

---

## 26.14 Summary

| Concept | Key Takeaway |
|---------|-------------|
| Service DNS | `<svc>.<ns>.svc.cluster.local` → ClusterIP |
| Headless DNS | Returns individual Pod IPs |
| StatefulSet DNS | `<pod>.<svc>.<ns>.svc.cluster.local` |
| SRV Records | Encodes port information for named ports |
| CoreDNS | Plugin-based DNS server, default since K8s 1.13 |
| Corefile | Configuration file defining plugin chains |
| kubernetes plugin | Watches API, generates DNS records |
| forward plugin | Sends non-cluster queries upstream |
| cache plugin | Caches responses, reduces load |
| /etc/resolv.conf | `nameserver`, `search`, `ndots:5` |
| ndots problem | External names trigger 4x queries |
| autopath plugin | Server-side search domain optimization |
| NodeLocal DNSCache | Per-node cache, sub-ms latency |
| dnsPolicy | ClusterFirst, Default, ClusterFirstWithHostNet, None |
| ExternalDNS | Syncs K8s resources to external DNS providers |
| Custom entries | hosts plugin, stub domains, rewrite |
| DNS + Network Policies | Always allow port 53 egress |

DNS is the invisible backbone of Kubernetes. When it works, nobody notices.
When it breaks, **everything** breaks. Master this chapter, and you'll be the
person your team calls when DNS goes wrong.

---

## Further Reading

- [Kubernetes DNS Specification](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)
- [CoreDNS Documentation](https://coredns.io/manual/toc/)
- [CoreDNS Plugin Reference](https://coredns.io/plugins/)
- [NodeLocal DNSCache](https://kubernetes.io/docs/tasks/administer-cluster/nodelocaldns/)
- [ExternalDNS GitHub](https://github.com/kubernetes-sigs/external-dns)
- [DNS Performance Tuning](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/#dns-config)
- [CoreDNS Grafana Dashboard](https://grafana.com/grafana/dashboards/14981-coredns/)
