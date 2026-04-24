# Chapter 24: Ingress, Gateway API & Load Balancing

> *"The edge is where the outside world meets your cluster."*

In Chapter 23, we traced packets from Pod to Pod and from Pod to Service. But Services
operate at Layer 4 — they route TCP and UDP connections by IP and port. They know nothing
about HTTP paths, hostnames, headers, or TLS certificates. If you want to expose an HTTP
API at `api.example.com/v1/users` and a web frontend at `www.example.com/`, Services
alone cannot help.

This is where Ingress and the Gateway API come in. They operate at Layer 7, understanding
HTTP semantics and making routing decisions based on hostnames, URL paths, headers, and
more. They terminate TLS, manage certificates, and provide advanced traffic management
like rate limiting, authentication, and canary deployments.

This chapter covers the full L7 stack: from the classic Ingress resource to the modern
Gateway API, from TLS certificate automation with cert-manager to building your own
ingress controller in Go.

---

## 24.1 Why Ingress Exists

### The L4 Limitation

A Kubernetes Service of type `LoadBalancer` gives you an external IP and forwards TCP/UDP
traffic to backend Pods. But consider a cluster with 20 microservices:

```
  Without Ingress — one LoadBalancer per service:

  api.example.com     → LoadBalancer ($$$) → Service A → Pods
  web.example.com     → LoadBalancer ($$$) → Service B → Pods
  admin.example.com   → LoadBalancer ($$$) → Service C → Pods
  ...
  20 services = 20 LoadBalancers = 20 external IPs = $$$$$$

  Each cloud LoadBalancer costs ~$15-20/month.
  20 LoadBalancers = $300-400/month just for routing!
```

### The L7 Solution

Ingress consolidates all external HTTP(S) traffic through a single entry point:

```text
  With Ingress — one LoadBalancer, L7 routing:

                    ┌─────────────────────────┐
                    │   Single LoadBalancer    │
                    │   (one external IP)      │
                    └────────────┬─────────────┘
                                 │
                    ┌────────────▼─────────────┐
                    │   Ingress Controller     │
                    │   (NGINX, Traefik, etc.) │
                    │                          │
                    │   Rules:                 │
                    │   api.example.com/*      │
                    │     → Service A          │
                    │   web.example.com/*      │
                    │     → Service B          │
                    │   admin.example.com/*    │
                    │     → Service C          │
                    └────┬────────┬────────┬───┘
                         │        │        │
                    Service A  Service B  Service C
                    (Pods)     (Pods)     (Pods)
```

### What Ingress Provides

| Feature | Service (L4) | Ingress (L7) |
|---------|:------------:|:------------:|
| TCP/UDP routing | ✓ | ✓ |
| HTTP path routing | ✗ | ✓ |
| Host-based routing | ✗ | ✓ |
| TLS termination | ✗ | ✓ |
| SSL certificate management | ✗ | ✓ |
| URL rewriting | ✗ | ✓ |
| Rate limiting | ✗ | ✓ |
| Authentication | ✗ | ✓ |
| WebSocket upgrade | ✗ | ✓ |
| HTTP/2, gRPC | ✗ | ✓ |

---

## 24.2 The Ingress Resource

The Ingress resource is a Kubernetes API object that declares L7 routing rules.

### Basic Structure

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    # Controller-specific configuration goes here
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx   # Which controller handles this Ingress
  tls:
  - hosts:
    - api.example.com
    - web.example.com
    secretName: example-tls  # TLS certificate Secret
  defaultBackend:
    service:
      name: default-service
      port:
        number: 80
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 80
      - path: /v2
        pathType: Prefix
        backend:
          service:
            name: api-v2
            port:
              number: 80
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-frontend
            port:
              number: 80
```

### Path Types

The `pathType` field determines how the `path` is matched:

| PathType | Behavior | Example path `/api` matches |
|----------|----------|----------------------------|
| `Exact` | Matches the URL path exactly | `/api` only |
| `Prefix` | Matches based on URL path prefix split by `/` | `/api`, `/api/`, `/api/v1` |
| `ImplementationSpecific` | Matching depends on the IngressClass | Controller-defined |

**Prefix matching nuances:**

```
Path: /api (Prefix)
  /api         → matches ✓
  /api/        → matches ✓
  /api/v1      → matches ✓
  /api/v1/user → matches ✓
  /apis        → NO match ✗ (not a path segment boundary)
  /apiv2       → NO match ✗
```

Prefix matching splits on `/` boundaries. `/api` matches `/api/anything` but not
`/apis` or `/apiv2`.

### Multiple Path Rules and Precedence

When multiple paths match a request, the longest matching path wins:

```yaml
rules:
- host: api.example.com
  http:
    paths:
    - path: /                    # Catches everything else
      pathType: Prefix
      backend:
        service:
          name: default-api
          port:
            number: 80
    - path: /v1/users            # More specific — wins for /v1/users/*
      pathType: Prefix
      backend:
        service:
          name: users-service
          port:
            number: 80
    - path: /v1/users/admin      # Most specific — wins for /v1/users/admin/*
      pathType: Exact
      backend:
        service:
          name: admin-service
          port:
            number: 80
```

```
  Request: GET /v1/users/admin    → admin-service  (Exact match wins)
  Request: GET /v1/users/123      → users-service  (longest Prefix match)
  Request: GET /v1/orders/456     → default-api    (only / matches)
```

### Annotations — Controller-Specific Configuration

The Ingress spec is deliberately minimal. Advanced features are configured through
annotations, which are specific to each Ingress controller:

```yaml
metadata:
  annotations:
    # NGINX-specific
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/limit-rps: "10"
    nginx.ingress.kubernetes.io/auth-url: "https://auth.example.com/verify"

    # Traefik-specific
    traefik.ingress.kubernetes.io/router.middlewares: default-ratelimit@kubernetescrd

    # AWS ALB-specific
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
```

This is one of the major weaknesses of the Ingress API — annotations are untyped,
unvalidated strings that differ across controllers. The Gateway API (Section 24.4)
addresses this with typed, structured configuration.

### Hands-On: Complete Ingress with Host Routing, Path Routing, and TLS

```yaml
# Step 1: Deploy backend services
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-v1
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-v1
  template:
    metadata:
      labels:
        app: api-v1
    spec:
      containers:
      - name: api
        image: hashicorp/http-echo
        args: ["-text=API v1 response", "-listen=:8080"]
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: api-v1
spec:
  selector:
    app: api-v1
  ports:
  - port: 80
    targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web-frontend
  template:
    metadata:
      labels:
        app: web-frontend
    spec:
      containers:
      - name: web
        image: hashicorp/http-echo
        args: ["-text=Web Frontend", "-listen=:8080"]
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: web-frontend
spec:
  selector:
    app: web-frontend
  ports:
  - port: 80
    targetPort: 8080

---
# Step 2: Create a self-signed TLS certificate (for testing)
# In production, use cert-manager (Section 24.5)
# openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
#   -keyout tls.key -out tls.crt \
#   -subj "/CN=*.example.com"
# kubectl create secret tls example-tls --cert=tls.crt --key=tls.key

---
# Step 3: Create the Ingress
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: main-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    - web.example.com
    secretName: example-tls
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /v1
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 80
  - host: web.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-frontend
            port:
              number: 80
```

```bash
# Apply everything
kubectl apply -f ingress-setup.yaml

# Test (using /etc/hosts or curl's --resolve flag)
curl --resolve api.example.com:443:127.0.0.1 \
     -k https://api.example.com/v1
# API v1 response

curl --resolve web.example.com:443:127.0.0.1 \
     -k https://web.example.com/
# Web Frontend

# Verify the Ingress
kubectl describe ingress main-ingress
```

---

## 24.3 Ingress Controllers — The Implementation

### The Ingress Resource Does Nothing Alone

This is a critical point that trips up beginners: creating an Ingress resource without an
Ingress controller is like creating a document that nobody reads. The Ingress resource is
a *declaration of intent* — it says "I want these routing rules." The controller is what
*implements* those rules.

```
  ┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
  │  Ingress Resource │────►│ Ingress Controller│────►│  Reverse Proxy   │
  │  (declarative    │     │  (watches API,    │     │  (nginx, envoy,  │
  │   routing rules) │     │   generates       │     │   traefik, etc.) │
  │                  │     │   proxy config)   │     │                  │
  └──────────────────┘     └──────────────────┘     └──────────────────┘
       "I want this"          "I'll make it so"        "I do the work"
```

### How NGINX Ingress Controller Works

The NGINX Ingress Controller is the most widely used controller. Here's its internal
architecture:

```
  ┌──────────────────────────────────────────────────────────────┐
  │  NGINX Ingress Controller Pod                                 │
  │                                                               │
  │  ┌─────────────────────────────────────────────────────────┐ │
  │  │  Controller Process (Go binary)                          │ │
  │  │                                                          │ │
  │  │  1. Watch API Server for:                                │ │
  │  │     - Ingress resources                                  │ │
  │  │     - Services                                           │ │
  │  │     - Endpoints / EndpointSlices                         │ │
  │  │     - Secrets (TLS certificates)                         │ │
  │  │     - ConfigMaps (global config)                         │ │
  │  │                                                          │ │
  │  │  2. Build internal model of routing rules                │ │
  │  │                                                          │ │
  │  │  3. Generate nginx.conf from template                    │ │
  │  │                                                          │ │
  │  │  4. Validate generated config (nginx -t)                 │ │
  │  │                                                          │ │
  │  │  5. Reload nginx (or use dynamic config for endpoints)   │ │
  │  └──────────┬──────────────────────────────────────────────┘ │
  │             │                                                 │
  │  ┌──────────▼──────────────────────────────────────────────┐ │
  │  │  NGINX Process                                           │ │
  │  │                                                          │ │
  │  │  server {                                                │ │
  │  │    listen 443 ssl;                                       │ │
  │  │    server_name api.example.com;                          │ │
  │  │    ssl_certificate /etc/tls/api/tls.crt;                │ │
  │  │                                                          │ │
  │  │    location /v1 {                                        │ │
  │  │      proxy_pass http://upstream_api_v1;                  │ │
  │  │    }                                                     │ │
  │  │  }                                                       │ │
  │  │                                                          │ │
  │  │  upstream upstream_api_v1 {                              │ │
  │  │    server 10.244.1.5:8080;                               │ │
  │  │    server 10.244.2.9:8080;                               │ │
  │  │  }                                                       │ │
  │  └─────────────────────────────────────────────────────────┘ │
  │                                                               │
  │  Exposed via Service type LoadBalancer (or NodePort)          │
  └──────────────────────────────────────────────────────────────┘
```

**Key implementation details:**

1. **Endpoint watching:** The controller watches EndpointSlices directly and configures
   NGINX upstream blocks with actual Pod IPs (bypassing the Service ClusterIP). This means
   NGINX does its own load balancing directly to Pods.

2. **Dynamic configuration:** Endpoint changes (Pod scaling) are applied without an NGINX
   reload using Lua/OpenResty scripting. Only routing rule changes (new Ingress, new host)
   trigger a full reload.

3. **TLS:** The controller reads TLS Secrets and writes certificate files that NGINX
   references. SNI (Server Name Indication) is used to select the right certificate when
   multiple hosts share the same IP.

### Packet Flow Through NGINX Ingress Controller

```
  External Client
       │
       │ HTTPS request to api.example.com/v1/users
       ▼
  ┌──────────────────────────────────────────────┐
  │ Cloud Load Balancer                           │
  │ (forwards to NodePort or Pod directly)        │
  └──────────────────┬───────────────────────────┘
                     │
                     ▼
  ┌──────────────────────────────────────────────┐
  │ NGINX Ingress Controller Pod                  │
  │                                               │
  │ ① TLS termination                             │
  │   - Decrypt HTTPS using certificate from      │
  │     Secret "example-tls"                      │
  │   - Now has plaintext HTTP request            │
  │                                               │
  │ ② Host matching                               │
  │   - Host: api.example.com                     │
  │   - Matches server block for api.example.com  │
  │                                               │
  │ ③ Path matching                               │
  │   - Path: /v1/users                           │
  │   - Matches location /v1 (Prefix)             │
  │                                               │
  │ ④ Upstream selection                          │
  │   - upstream_api_v1: [10.244.1.5, 10.244.2.9]│
  │   - Round-robin selects 10.244.1.5            │
  │                                               │
  │ ⑤ Proxy request                               │
  │   - Forward HTTP request to 10.244.1.5:8080   │
  │   - Add X-Forwarded-For, X-Real-IP headers    │
  │                                               │
  │ ⑥ Return response to client                   │
  └──────────────────────────────────────────────┘
```

### Other Ingress Controllers

| Controller | Proxy | Key Features |
|------------|-------|-------------|
| **NGINX Ingress** | NGINX | Most mature, huge annotation set, Lua scripting |
| **Traefik** | Traefik | Auto-discovery, Let's Encrypt built-in, middleware chains |
| **HAProxy** | HAProxy | High performance, advanced LB algorithms |
| **Contour** | Envoy | HTTPProxy CRD (richer than Ingress), rate limiting |
| **Istio** | Envoy | Service mesh integration, advanced traffic management |
| **Ambassador/Emissary** | Envoy | Developer-friendly, CRD-based config |
| **AWS ALB Controller** | AWS ALB | Native AWS ALB, no in-cluster proxy |
| **GCE Ingress** | Google Cloud LB | Native GCE LB, no in-cluster proxy |

### Hands-On: Installing NGINX Ingress Controller in kind

```bash
# Create a kind cluster with port mappings for Ingress
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        node-labels: "ingress-ready=true"
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
EOF

# Install NGINX Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml

# Wait for it to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=90s

# Verify
kubectl get pods -n ingress-nginx
# NAME                                       READY   STATUS
# ingress-nginx-controller-xxxxx-yyy         1/1     Running

# The controller is exposed via NodePort on ports 80 and 443
# Thanks to kind's port mapping, localhost:80 → controller
curl http://localhost
# <html>... 404 Not Found ...</html>   (no Ingress rules yet — 404 is correct)
```

### Hands-On: Full Ingress Setup with TLS Using cert-manager

(See Section 24.5 for cert-manager details. Quick setup here:)

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml
kubectl wait --for=condition=ready pod -l app=cert-manager -n cert-manager --timeout=60s

# Create a self-signed ClusterIssuer (for testing)
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
EOF

# Create an Ingress with automatic TLS
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
  annotations:
    cert-manager.io/cluster-issuer: selfsigned-issuer
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls     # cert-manager will create this Secret
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-frontend
            port:
              number: 80
EOF

# cert-manager automatically:
# 1. Sees the annotation cert-manager.io/cluster-issuer
# 2. Creates a Certificate resource
# 3. Generates a self-signed cert
# 4. Stores it in Secret "myapp-tls"
# 5. NGINX Ingress Controller picks up the Secret and configures TLS

kubectl get certificate
# NAME        READY   SECRET       AGE
# myapp-tls   True    myapp-tls    30s

curl -k --resolve myapp.example.com:443:127.0.0.1 https://myapp.example.com/
```

---

## 24.4 Gateway API — The Future of Kubernetes Ingress

### Why Gateway API Was Created

The Ingress resource has served Kubernetes well, but it has fundamental limitations:

```
  Ingress Limitations:
  ┌──────────────────────────────────────────────────────────┐
  │ 1. Limited expressiveness                                 │
  │    - Only HTTP(S) routing                                │
  │    - No TCP/UDP/gRPC-native routing                      │
  │    - No traffic splitting (canary deployments)            │
  │    - No header-based routing                              │
  │                                                          │
  │ 2. Annotations are a mess                                 │
  │    - Untyped strings, no validation                      │
  │    - Every controller has different annotations           │
  │    - No portability between controllers                   │
  │                                                          │
  │ 3. No role separation                                     │
  │    - One resource controls both infrastructure and routing│
  │    - Platform teams and app teams edit the same object    │
  │                                                          │
  │ 4. No standard for advanced features                      │
  │    - Rate limiting, auth, retries are all annotation-based│
  │    - No common API across controllers                     │
  └──────────────────────────────────────────────────────────┘
```

The Gateway API addresses all of these issues with a rich, typed, role-oriented resource
model.

### Resource Model

```
  ┌─────────────────────────────────────────────────────────────┐
  │                        Gateway API Resources                 │
  │                                                              │
  │  ┌──────────────┐   Managed by: Infrastructure Provider     │
  │  │ GatewayClass │   (cloud provider, platform team)         │
  │  │              │   "What kind of gateway is available?"     │
  │  └──────┬───────┘                                           │
  │         │                                                    │
  │  ┌──────▼───────┐   Managed by: Cluster Operator             │
  │  │   Gateway    │   (platform/SRE team)                     │
  │  │              │   "Deploy a gateway with these listeners"  │
  │  │  - listeners │                                            │
  │  │  - addresses │                                            │
  │  └──────┬───────┘                                            │
  │         │                                                    │
  │  ┌──────▼───────┐   Managed by: Application Developer       │
  │  │  HTTPRoute   │   (dev team)                              │
  │  │  TCPRoute    │   "Route my traffic like this"            │
  │  │  TLSRoute    │                                            │
  │  │  GRPCRoute   │                                            │
  │  └──────────────┘                                            │
  └─────────────────────────────────────────────────────────────┘
```

**Role-oriented design:** Each resource is owned by a different team:

| Resource | Owner | Responsibility |
|----------|-------|---------------|
| `GatewayClass` | Infrastructure provider | Define what gateway implementations exist |
| `Gateway` | Cluster operator / platform team | Deploy gateways, configure listeners, TLS |
| `HTTPRoute` | Application developer | Define routing rules for their service |

This separation means app developers can create HTTPRoutes without needing permissions
to modify the Gateway itself.

### GatewayClass

A GatewayClass defines a "class" of Gateway, similar to StorageClass for volumes or
IngressClass for Ingress. It references a controller that implements the Gateway.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: istio
spec:
  controllerName: istio.io/gateway-controller
  description: "Istio-based gateway implementation"
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: cilium
spec:
  controllerName: io.cilium/gateway-controller
  description: "Cilium-based gateway implementation"
```

### Gateway

A Gateway is a deployment of a gateway (typically a reverse proxy like Envoy). It
specifies which GatewayClass to use and what listeners to configure.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: gateway-system
spec:
  gatewayClassName: istio
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    hostname: "*.example.com"       # Accept traffic for all subdomains
    allowedRoutes:
      namespaces:
        from: All                   # Routes from any namespace can attach
  - name: https
    port: 443
    protocol: HTTPS
    hostname: "*.example.com"
    tls:
      mode: Terminate
      certificateRefs:
      - kind: Secret
        name: wildcard-example-tls
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            gateway-access: "true"  # Only namespaces with this label
```

**Key features:**
- **Multiple listeners:** A single Gateway can listen on multiple ports/protocols
- **Hostname scoping:** Each listener can be scoped to specific hostnames
- **Namespace isolation:** `allowedRoutes` controls which namespaces can attach routes
- **TLS per-listener:** Each listener can have its own TLS configuration

### HTTPRoute

HTTPRoutes define the actual routing rules. They "attach" to a Gateway.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-routes
  namespace: api-team     # App team's namespace
spec:
  parentRefs:
  - name: main-gateway
    namespace: gateway-system   # Attach to the shared Gateway
    sectionName: https          # Attach to the HTTPS listener
  hostnames:
  - "api.example.com"
  rules:
  # Rule 1: /v1/users → users-service
  - matches:
    - path:
        type: PathPrefix
        value: /v1/users
    backendRefs:
    - name: users-service
      port: 80

  # Rule 2: /v1/orders → orders-service
  - matches:
    - path:
        type: PathPrefix
        value: /v1/orders
    backendRefs:
    - name: orders-service
      port: 80

  # Rule 3: Header-based routing (canary)
  - matches:
    - headers:
      - name: X-Canary
        value: "true"
    backendRefs:
    - name: users-service-canary
      port: 80
```

### HTTPRoute Advanced Features

**Weight-based traffic splitting (canary deployments):**

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: canary-route
spec:
  parentRefs:
  - name: main-gateway
    namespace: gateway-system
  hostnames:
  - "api.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1
    backendRefs:
    - name: api-v1-stable
      port: 80
      weight: 90         # 90% of traffic → stable
    - name: api-v1-canary
      port: 80
      weight: 10         # 10% of traffic → canary
```

**Request header modification:**

```yaml
rules:
- matches:
  - path:
      type: PathPrefix
      value: /api
  filters:
  - type: RequestHeaderModifier
    requestHeaderModifier:
      add:
      - name: X-Request-Source
        value: gateway
      set:
      - name: Host
        value: internal-api.default.svc
      remove:
      - X-Internal-Debug
  backendRefs:
  - name: api-service
    port: 80
```

**URL rewriting:**

```yaml
rules:
- matches:
  - path:
      type: PathPrefix
      value: /api/v1
  filters:
  - type: URLRewrite
    urlRewrite:
      path:
        type: ReplacePrefixMatch
        replacePrefixMatch: /v1
  backendRefs:
  - name: api-service
    port: 80
# /api/v1/users → /v1/users (prefix replaced)
```

**Redirects:**

```yaml
rules:
- matches:
  - path:
      type: PathPrefix
      value: /old-api
  filters:
  - type: RequestRedirect
    requestRedirect:
      scheme: https
      hostname: api.example.com
      path:
        type: ReplacePrefixMatch
        replacePrefixMatch: /v2
      statusCode: 301
```

### TCPRoute and TLSRoute

The Gateway API isn't limited to HTTP. TCPRoute and TLSRoute handle L4 traffic:

```yaml
# TCPRoute — raw TCP forwarding
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
metadata:
  name: postgres-route
spec:
  parentRefs:
  - name: tcp-gateway
    sectionName: postgres
  rules:
  - backendRefs:
    - name: postgres-service
      port: 5432
---
# TLSRoute — TLS passthrough (no termination)
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TLSRoute
metadata:
  name: tls-passthrough
spec:
  parentRefs:
  - name: main-gateway
    sectionName: tls-passthrough
  hostnames:
  - "secure.example.com"
  rules:
  - backendRefs:
    - name: secure-backend
      port: 8443
```

### GRPCRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: grpc-route
spec:
  parentRefs:
  - name: main-gateway
  hostnames:
  - "grpc.example.com"
  rules:
  - matches:
    - method:
        service: mypackage.MyService
        method: GetUser
    backendRefs:
    - name: grpc-service
      port: 50051
```

### Hands-On: Full Gateway API Setup

```bash
# Step 1: Install Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml

# Step 2: Install a Gateway API controller (using Cilium as example)
# If using Cilium:
cilium install --set gatewayAPI.enabled=true

# Or using Istio:
# istioctl install --set profile=minimal
# kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml

# Step 3: Create GatewayClass (usually done by the controller install)
kubectl get gatewayclass
# NAME     CONTROLLER                    ACCEPTED
# cilium   io.cilium/gateway-controller  True

# Step 4: Create a Gateway
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: cilium
  listeners:
  - name: http
    port: 80
    protocol: HTTP
    allowedRoutes:
      namespaces:
        from: All
EOF

# Step 5: Create HTTPRoutes
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-routes
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - "app.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-v1
      port: 80
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-frontend
      port: 80
EOF

# Step 6: Create a canary route with traffic splitting
kubectl apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: canary-route
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - "canary.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: web-frontend
      port: 80
      weight: 90
    - name: web-frontend-v2
      port: 80
      weight: 10
EOF

# Verify
kubectl get gateway
kubectl get httproute
kubectl describe httproute app-routes

# Test
GATEWAY_IP=$(kubectl get gateway my-gateway -o jsonpath='{.status.addresses[0].value}')
curl --resolve app.example.com:80:$GATEWAY_IP http://app.example.com/api
curl --resolve app.example.com:80:$GATEWAY_IP http://app.example.com/
```

---

## 24.5 TLS and Certificate Management

### TLS Termination at the Ingress

TLS termination means the Ingress controller decrypts HTTPS traffic and forwards
plaintext HTTP to backend Pods. This simplifies backend services — they don't need to
manage certificates.

```
  Client                  Ingress Controller           Backend Pod
    │                          │                           │
    │──── HTTPS (encrypted) ──►│                           │
    │                          │── HTTP (plaintext) ──────►│
    │                          │                           │
    │◄── HTTPS (encrypted) ───│                           │
    │                          │◄── HTTP (plaintext) ──────│
```

**TLS modes:**

| Mode | Description | Use Case |
|------|-------------|----------|
| Terminate | Decrypt at ingress, plaintext to backend | Most common |
| Passthrough | Forward encrypted traffic directly | Backend manages its own TLS |
| Re-encrypt | Decrypt at ingress, re-encrypt to backend | End-to-end encryption required |

### cert-manager — Automated Certificate Management

cert-manager is the standard tool for automating TLS certificate lifecycle in Kubernetes.
It watches for Certificate resources (or Ingress annotations) and automatically:

1. Requests certificates from a CA (Certificate Authority)
2. Validates domain ownership (ACME challenges)
3. Stores certificates as Kubernetes Secrets
4. Renews certificates before expiry

```
  ┌──────────────────────────────────────────────────────────────┐
  │  cert-manager Architecture                                    │
  │                                                               │
  │  ┌────────────┐    ┌────────────────┐    ┌────────────────┐  │
  │  │  Issuer/    │    │  Certificate   │    │  Secret        │  │
  │  │  Cluster    │◄───│  Resource      │───►│  (tls.crt +    │  │
  │  │  Issuer     │    │                │    │   tls.key)     │  │
  │  └──────┬─────┘    └────────────────┘    └────────────────┘  │
  │         │                                                     │
  │         ▼                                                     │
  │  ┌─────────────────────────────────────────────────────────┐ │
  │  │  cert-manager controller                                 │ │
  │  │                                                          │ │
  │  │  1. Watches Certificate resources                        │ │
  │  │  2. Creates Order (ACME) or signs directly               │ │
  │  │  3. Solves challenges (HTTP-01 or DNS-01)                │ │
  │  │  4. Stores issued cert in Secret                         │ │
  │  │  5. Renews before expiry (default: 30 days before)       │ │
  │  └─────────────────────────────────────────────────────────┘ │
  └──────────────────────────────────────────────────────────────┘
```

### Issuers and ClusterIssuers

An Issuer defines how certificates are obtained. `Issuer` is namespace-scoped,
`ClusterIssuer` is cluster-wide.

**Let's Encrypt (ACME) — Production:**

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-account-key
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
```

**Let's Encrypt — Staging (for testing):**

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-staging-account-key
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
```

**Self-signed (for development):**

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned
spec:
  selfSigned: {}
```

**Internal CA:**

```yaml
# First create a self-signed CA certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: ca-cert
  namespace: cert-manager
spec:
  isCA: true
  commonName: my-internal-ca
  secretName: ca-secret
  issuerRef:
    name: selfsigned
    kind: ClusterIssuer
---
# Then create a ClusterIssuer using the CA
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: internal-ca
spec:
  ca:
    secretName: ca-secret
```

### ACME Challenge Types

When using Let's Encrypt (ACME protocol), cert-manager must prove you control the domain.

**HTTP-01 Challenge:**

```
  Let's Encrypt                  cert-manager              Ingress
       │                              │                      │
  ①    │ "Prove you own              │                      │
       │  api.example.com"            │                      │
       │──────────────────────────►  │                      │
       │                              │                      │
  ②    │                              │ Create temporary     │
       │                              │ Ingress rule:        │
       │                              │ /.well-known/acme-   │
       │                              │ challenge/xxx → token│
       │                              │─────────────────────►│
       │                              │                      │
  ③    │ GET http://api.example.com/.well-known/acme-challenge/xxx
       │──────────────────────────────────────────────────►  │
       │                              │                      │
  ④    │◄──────────────── token ──────────────────────────── │
       │                              │                      │
  ⑤    │ "Domain verified!           │                      │
       │  Here's your certificate"    │                      │
       │──────────────────────────►  │                      │
```

**HTTP-01 requirements:**
- Port 80 must be reachable from the internet
- DNS must resolve to your Ingress controller

**DNS-01 Challenge:**

```
  Let's Encrypt                  cert-manager              DNS Provider
       │                              │                      │
  ①    │ "Prove you own              │                      │
       │  api.example.com"            │                      │
       │──────────────────────────►  │                      │
       │                              │                      │
  ②    │                              │ Create TXT record:   │
       │                              │ _acme-challenge.     │
       │                              │ api.example.com      │
       │                              │ = "token123"         │
       │                              │─────────────────────►│
       │                              │                      │
  ③    │ DNS lookup:                  │                      │
       │ TXT _acme-challenge.api.example.com                 │
       │──────────────────────────────────────────────────►  │
       │◄──────────── "token123" ──────────────────────────  │
       │                              │                      │
  ④    │ "Domain verified! Cert:"    │                      │
       │──────────────────────────►  │                      │
```

**DNS-01 requirements:**
- cert-manager needs API access to your DNS provider
- Supports wildcard certificates (HTTP-01 does not)
- Works even if port 80 is blocked

**DNS-01 solver configuration (examples):**

```yaml
solvers:
- dns01:
    cloudflare:
      email: admin@example.com
      apiTokenSecretRef:
        name: cloudflare-token
        key: api-token
- dns01:
    route53:
      region: us-east-1
      accessKeyIDSecretRef:
        name: aws-credentials
        key: access-key-id
      secretAccessKeySecretRef:
        name: aws-credentials
        key: secret-access-key
```

### Certificate Resource

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: api-cert
  namespace: api-team
spec:
  secretName: api-tls                    # Where the cert will be stored
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  commonName: api.example.com
  dnsNames:
  - api.example.com
  - api-staging.example.com
  duration: 2160h      # 90 days
  renewBefore: 720h    # Renew 30 days before expiry
```

### Hands-On: cert-manager Installation and Let's Encrypt Setup

```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

# Verify
kubectl get pods -n cert-manager
# NAME                                      READY   STATUS
# cert-manager-xxxxxxx                      1/1     Running
# cert-manager-cainjector-xxxxxxx           1/1     Running
# cert-manager-webhook-xxxxxxx              1/1     Running

# Create Let's Encrypt staging issuer (use staging first to avoid rate limits!)
kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-staging-key
    solvers:
    - http01:
        ingress:
          ingressClassName: nginx
EOF

# Verify issuer is ready
kubectl get clusterissuer
# NAME                  READY   AGE
# letsencrypt-staging   True    30s
```

### Hands-On: Automatic TLS Certificate for an Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auto-tls-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls-auto     # cert-manager creates this!
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 80
```

```bash
kubectl apply -f auto-tls-ingress.yaml

# Watch cert-manager work
kubectl get certificate -w
# NAME           READY   SECRET         AGE
# api-tls-auto   False   api-tls-auto   5s    ← Pending
# api-tls-auto   True    api-tls-auto   30s   ← Issued!

# View the Order (ACME workflow)
kubectl get order
kubectl describe order api-tls-auto-xxxxx

# View the Challenge
kubectl get challenge
kubectl describe challenge api-tls-auto-xxxxx

# Verify the Secret was created
kubectl get secret api-tls-auto
# NAME           TYPE                DATA   AGE
# api-tls-auto   kubernetes.io/tls   2      30s

# Test
curl -k --resolve api.example.com:443:$INGRESS_IP https://api.example.com/
```

---

## 24.6 Advanced L7 Features

### Rate Limiting

Rate limiting protects backends from abuse and ensures fair resource usage.

**NGINX Ingress annotations:**

```yaml
metadata:
  annotations:
    # Limit to 10 requests per second per client IP
    nginx.ingress.kubernetes.io/limit-rps: "10"

    # Limit to 100 connections per client IP
    nginx.ingress.kubernetes.io/limit-connections: "100"

    # Custom rate limiting with burst
    nginx.ingress.kubernetes.io/limit-rpm: "600"       # 600 req/min = 10 req/sec
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"  # Allow burst of 50
    
    # Rate limit response code (default 503)
    nginx.ingress.kubernetes.io/limit-rate-after: "10"  # Rate limit after 10MB
    nginx.ingress.kubernetes.io/limit-rate: "1"         # Then 1 byte/sec
```

**Gateway API (via policy attachment):**

```yaml
# With Cilium's Gateway API implementation
apiVersion: cilium.io/v2
kind: CiliumHTTPRateLimit
metadata:
  name: api-rate-limit
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: api-routes
  default:
    requestsPerSecond: 10
    burst: 50
```

### Hands-On: Rate Limiting with NGINX Annotations

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rate-limited-ingress
  annotations:
    nginx.ingress.kubernetes.io/limit-rps: "5"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "3"
spec:
  ingressClassName: nginx
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-v1
            port:
              number: 80
```

```bash
kubectl apply -f rate-limited-ingress.yaml

# Test — send 20 rapid requests
for i in $(seq 1 20); do
  CODE=$(curl -s -o /dev/null -w "%{http_code}" \
    --resolve api.example.com:80:127.0.0.1 \
    http://api.example.com/)
  echo "Request $i: HTTP $CODE"
done

# Expected output:
# Request 1: HTTP 200
# ...
# Request 15: HTTP 200
# Request 16: HTTP 503     ← Rate limited!
# Request 17: HTTP 503
# ...
```

### Authentication (OAuth2 Proxy)

The OAuth2 Proxy pattern delegates authentication to an external OAuth2/OIDC provider
(Google, GitHub, Azure AD, etc.).

```
  Client          NGINX Ingress       OAuth2 Proxy        Backend
    │                  │                   │                  │
    │ GET /api ────►   │                   │                  │
    │                  │ auth-url check ──►│                  │
    │                  │                   │ No valid cookie  │
    │                  │◄── 401 ───────── │                  │
    │◄── 302 redirect to OAuth provider   │                  │
    │                                                        │
    │── Login at OAuth provider ──►                          │
    │◄── Redirect back with code ──                          │
    │                                                        │
    │ GET /oauth2/callback ────────────►  │                  │
    │                                     │ Exchange code    │
    │                                     │ for token        │
    │◄── Set cookie, redirect to /api ── │                  │
    │                                                        │
    │ GET /api (with cookie) ──►│         │                  │
    │                           │ auth ──►│                  │
    │                           │◄── 200 ─│                  │
    │                           │ forward ──────────────────►│
    │◄──────────────────────── response ─────────────────── │
```

**Setup:**

```yaml
# OAuth2 Proxy Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: oauth2-proxy
spec:
  replicas: 1
  selector:
    matchLabels:
      app: oauth2-proxy
  template:
    metadata:
      labels:
        app: oauth2-proxy
    spec:
      containers:
      - name: oauth2-proxy
        image: quay.io/oauth2-proxy/oauth2-proxy:latest
        args:
        - --provider=github
        - --email-domain=*
        - --upstream=static://200
        - --http-address=0.0.0.0:4180
        - --cookie-secure=false      # Set true in production with HTTPS
        env:
        - name: OAUTH2_PROXY_CLIENT_ID
          valueFrom:
            secretKeyRef:
              name: oauth2-proxy-secret
              key: client-id
        - name: OAUTH2_PROXY_CLIENT_SECRET
          valueFrom:
            secretKeyRef:
              name: oauth2-proxy-secret
              key: client-secret
        - name: OAUTH2_PROXY_COOKIE_SECRET
          valueFrom:
            secretKeyRef:
              name: oauth2-proxy-secret
              key: cookie-secret
        ports:
        - containerPort: 4180
---
apiVersion: v1
kind: Service
metadata:
  name: oauth2-proxy
spec:
  selector:
    app: oauth2-proxy
  ports:
  - port: 4180
    targetPort: 4180
---
# Ingress with OAuth2 Proxy authentication
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: protected-ingress
  annotations:
    nginx.ingress.kubernetes.io/auth-url: "http://oauth2-proxy.default.svc.cluster.local:4180/oauth2/auth"
    nginx.ingress.kubernetes.io/auth-signin: "http://oauth2-proxy.example.com/oauth2/start?rd=$scheme://$host$request_uri"
spec:
  ingressClassName: nginx
  rules:
  - host: protected.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-protected-app
            port:
              number: 80
```

### CORS Configuration

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://frontend.example.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE, OPTIONS"
    nginx.ingress.kubernetes.io/cors-allow-headers: "Authorization, Content-Type"
    nginx.ingress.kubernetes.io/cors-allow-credentials: "true"
    nginx.ingress.kubernetes.io/cors-max-age: "3600"
```

### Rewrites and Redirects

```yaml
# Rewrite: /api/v1/users → /v1/users (strip /api prefix)
metadata:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 80
---
# Redirect: HTTP → HTTPS
metadata:
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
---
# Permanent redirect: old domain → new domain
metadata:
  annotations:
    nginx.ingress.kubernetes.io/permanent-redirect: "https://new.example.com"
    nginx.ingress.kubernetes.io/permanent-redirect-code: "301"
```

### WebSocket Support

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/proxy-read-timeout: "3600"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "3600"
    # WebSocket support is enabled by default in NGINX Ingress
    # The Connection: Upgrade header is forwarded automatically
```

### gRPC Routing

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
spec:
  rules:
  - host: grpc.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grpc-service
            port:
              number: 50051
```

---

## 24.7 Load Balancing Algorithms

### Algorithms Available in NGINX Ingress

| Algorithm | Annotation | Description |
|-----------|-----------|-------------|
| Round Robin | (default) | Equal distribution across backends |
| Least Connections | `nginx.ingress.kubernetes.io/upstream-hash-by: ""` with `least_conn` | Sends to backend with fewest active connections |
| IP Hash | `nginx.ingress.kubernetes.io/upstream-hash-by: "$remote_addr"` | Same client IP → same backend |
| Consistent Hash | `nginx.ingress.kubernetes.io/upstream-hash-by: "$request_uri"` | Hash of URI determines backend |
| Random with Two Choices | Custom NGINX config | Pick 2 random backends, choose the one with fewer connections |
| EWMA | `nginx.ingress.kubernetes.io/load-balance: "ewma"` | Exponentially weighted moving average of response times |

### Session Persistence (Sticky Sessions)

Some applications require that a client always reaches the same backend (e.g., shopping
cart stored in memory, WebSocket connections).

**Cookie-based affinity:**

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/affinity-mode: "persistent"
    nginx.ingress.kubernetes.io/session-cookie-name: "SERVERID"
    nginx.ingress.kubernetes.io/session-cookie-expires: "172800"  # 2 days
    nginx.ingress.kubernetes.io/session-cookie-max-age: "172800"
    nginx.ingress.kubernetes.io/session-cookie-change-on-failure: "true"
```

```
  First request:
  Client → Ingress → selects Pod A → response + Set-Cookie: SERVERID=pod-a-hash

  Subsequent requests:
  Client (Cookie: SERVERID=pod-a-hash) → Ingress → routes to Pod A
```

### Health Checking

NGINX Ingress supports passive health checking by default and active health checking
with NGINX Plus.

**Passive health checking (open source):**

```yaml
metadata:
  annotations:
    # Mark backend as unhealthy after 3 failures in 5 seconds
    nginx.ingress.kubernetes.io/proxy-next-upstream: "error timeout http_503"
    nginx.ingress.kubernetes.io/proxy-next-upstream-tries: "3"
    nginx.ingress.kubernetes.io/proxy-next-upstream-timeout: "5"
```

**Kubernetes-native health checking:**
NGINX Ingress watches EndpointSlices. Pods that fail readiness probes are removed from
endpoints, and NGINX removes them from its upstream pool. This is the primary health
checking mechanism.

---

## 24.8 External DNS

ExternalDNS automatically creates DNS records based on Kubernetes resources (Services,
Ingresses, Gateway API resources).

### How It Works

```
  ┌────────────────────┐    watches     ┌────────────────────┐
  │  Ingress/Service/  │───────────────►│   ExternalDNS      │
  │  Gateway resources │                │   Controller        │
  └────────────────────┘                └─────────┬──────────┘
                                                  │
                                                  │ Creates/updates
                                                  │ DNS records
                                                  ▼
                                        ┌────────────────────┐
                                        │   DNS Provider     │
                                        │   (Route53, CF,    │
                                        │    Google DNS...)   │
                                        └────────────────────┘

  Example:
  1. You create an Ingress for "api.example.com"
  2. Ingress gets an external IP: 34.123.45.67
  3. ExternalDNS sees this and creates:
     api.example.com. A 34.123.45.67
```

### Hands-On: ExternalDNS Concept and Configuration

```yaml
# ExternalDNS Deployment (AWS Route53 example)
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
        - --source=ingress           # Watch Ingress resources
        - --source=service           # Watch Service resources
        - --provider=aws             # Use AWS Route53
        - --aws-zone-type=public
        - --registry=txt             # Use TXT records for ownership
        - --txt-owner-id=my-cluster  # Identify records owned by this cluster
        - --domain-filter=example.com # Only manage records under example.com
        - --policy=upsert-only       # Never delete records (safe mode)
```

```bash
# Create a Service with the ExternalDNS annotation
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: my-web
  annotations:
    external-dns.alpha.kubernetes.io/hostname: web.example.com
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
EOF

# ExternalDNS will create:
# web.example.com. A <LoadBalancer-IP>
# web.example.com. TXT "heritage=external-dns,external-dns/owner=my-cluster"

# For Ingress, no annotation needed — ExternalDNS reads the host from the rules:
# spec.rules[*].host → DNS record
```

---

## 24.9 Go: Building a Simple Ingress Controller

To truly understand how Ingress controllers work, let's build a minimal one in Go. This
controller watches Ingress resources and configures a reverse proxy to route traffic.

```go
// main.go
//
// MinimalIngressController is an educational ingress controller that demonstrates
// the core mechanics of how a Kubernetes ingress controller works. It watches
// for Ingress resources via the Kubernetes API, builds an in-memory routing
// table, and runs an HTTP reverse proxy that routes requests based on the
// Host header and URL path.
//
// Architecture:
//
//	┌─────────────────┐       ┌──────────────────────┐       ┌───────────────┐
//	│ Kubernetes API   │──watch──►│ Controller (informer) │──update──►│ Router (mux)  │
//	│ (Ingress objects)│       │ builds routing table  │       │ reverse proxy │
//	└─────────────────┘       └──────────────────────┘       └───────────────┘
//
// This is NOT production-ready. It is intentionally simplified to demonstrate:
//   - How informers watch the Kubernetes API for Ingress changes
//   - How routing tables are built from Ingress specs
//   - How reverse proxying works with httputil.ReverseProxy
//
// A production controller would add: TLS termination, health checks,
// connection pooling, metrics, leader election, and proper error handling.
package main

import (
	"context"
	"fmt"
	"log"
	"net/http"
	"net/http/httputil"
	"net/url"
	"os"
	"os/signal"
	"strings"
	"sync"
	"syscall"
	"time"

	networkingv1 "k8s.io/api/networking/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
)

// routeKey uniquely identifies a routing rule by combining the
// hostname and URL path. For example, the key for
// "api.example.com" with path "/v1" would be:
//
//	routeKey{host: "api.example.com", path: "/v1"}
//
// The router uses this as a map key to find the correct backend
// for an incoming request.
type routeKey struct {
	host string // HTTP Host header value (e.g., "api.example.com")
	path string // URL path prefix (e.g., "/v1")
}

// backend represents a Kubernetes Service that should receive
// proxied traffic. It stores the internal cluster URL that the
// reverse proxy will forward requests to.
//
// In a real ingress controller, this would include:
//   - Multiple endpoints (Pod IPs) for load balancing
//   - Health check state per endpoint
//   - Connection pool configuration
//   - TLS settings for backend connections
type backend struct {
	// serviceURL is the internal Kubernetes Service URL.
	// Format: "http://<service-name>.<namespace>.svc.cluster.local:<port>"
	// The reverse proxy uses this as the target for forwarded requests.
	serviceURL *url.URL
}

// routeTable is a thread-safe mapping from (host, path) pairs to backend
// services. It is rebuilt whenever Ingress resources are created, updated,
// or deleted in the Kubernetes API.
//
// The routing algorithm is:
//  1. Extract the Host header and URL path from the incoming request
//  2. Try progressively shorter path prefixes until a match is found
//  3. Fall back to "/" if no more specific path matches
//  4. Return 404 if no route matches at all
//
// Thread safety: The route table is protected by a sync.RWMutex because
// it is read by HTTP handler goroutines (concurrent) and written by the
// informer event handler (single-threaded, but concurrent with reads).
type routeTable struct {
	mu     sync.RWMutex
	routes map[routeKey]*backend
}

// newRouteTable creates an empty route table ready to receive routes
// from the Ingress informer's event handlers.
func newRouteTable() *routeTable {
	return &routeTable{
		routes: make(map[routeKey]*backend),
	}
}

// update replaces the entire route table atomically. This is called
// whenever an Ingress resource changes. We rebuild the full table
// rather than applying incremental updates because:
//   - It's simpler and less error-prone
//   - Ingress changes are infrequent (seconds/minutes, not milliseconds)
//   - The table is small (hundreds of entries at most)
//
// Parameters:
//   - newRoutes: the complete set of routes derived from all current Ingress resources
func (rt *routeTable) update(newRoutes map[routeKey]*backend) {
	rt.mu.Lock()
	defer rt.mu.Unlock()
	rt.routes = newRoutes
}

// lookup finds the backend for a given host and path. It implements
// longest-prefix matching: for a request to "api.example.com/v1/users/123",
// it tries these keys in order:
//
//	("api.example.com", "/v1/users/123") → miss
//	("api.example.com", "/v1/users")     → miss
//	("api.example.com", "/v1")           → HIT → return backend
//
// If no path-specific match is found, it falls back to the root path "/".
// Returns nil if no route matches at all.
//
// Parameters:
//   - host: the HTTP Host header value (e.g., "api.example.com")
//   - path: the request URL path (e.g., "/v1/users/123")
//
// Returns:
//   - The matching backend, or nil if no route matches
func (rt *routeTable) lookup(host, path string) *backend {
	rt.mu.RLock()
	defer rt.mu.RUnlock()

	// Try increasingly shorter path prefixes.
	// This implements the Kubernetes Ingress "Prefix" path type semantics:
	// the longest matching prefix wins.
	for path != "" {
		if b, ok := rt.routes[routeKey{host: host, path: path}]; ok {
			return b
		}
		// Remove the last path segment to try a shorter prefix.
		// "/v1/users/123" → "/v1/users" → "/v1" → "/"
		idx := strings.LastIndex(path, "/")
		if idx <= 0 {
			// We've reached the root — try "/" and stop.
			break
		}
		path = path[:idx]
	}

	// Final fallback: try the root path for this host.
	if b, ok := rt.routes[routeKey{host: host, path: "/"}]; ok {
		return b
	}

	return nil
}

// IngressController watches Kubernetes Ingress resources and runs
// an HTTP reverse proxy that routes traffic according to the Ingress
// rules. It is the main orchestrator of the application.
//
// Lifecycle:
//  1. NewIngressController() creates the controller with a Kubernetes client
//  2. Run() starts the informer (watches API) and the HTTP server
//  3. The informer calls onAdd/onUpdate/onDelete when Ingresses change
//  4. These handlers call rebuildRoutes() to update the route table
//  5. The HTTP server uses the route table to proxy requests
//  6. Run() blocks until the context is cancelled (SIGINT/SIGTERM)
type IngressController struct {
	// clientset is the Kubernetes API client used to list and watch
	// Ingress resources and resolve Service endpoints.
	clientset kubernetes.Interface

	// routes holds the current routing configuration derived from
	// all Ingress resources in the cluster.
	routes *routeTable

	// informerFactory creates shared informers that efficiently watch
	// the Kubernetes API with a single connection per resource type.
	informerFactory informers.SharedInformerFactory

	// ingressClassName is the IngressClass this controller handles.
	// Ingress resources with a different ingressClassName are ignored.
	// This allows multiple controllers to coexist in the same cluster.
	ingressClassName string
}

// NewIngressController creates a new IngressController. It initializes
// the Kubernetes client, shared informer factory, and route table.
//
// Parameters:
//   - clientset: a Kubernetes API client (created from kubeconfig or in-cluster config)
//   - className: the IngressClass name this controller handles (e.g., "minimal")
//
// Returns:
//   - A fully initialized IngressController ready to be started with Run()
func NewIngressController(clientset kubernetes.Interface, className string) *IngressController {
	return &IngressController{
		clientset: clientset,
		routes:    newRouteTable(),
		// Resync every 30 seconds: the informer will re-list all Ingress
		// resources and call update handlers. This catches any missed events.
		informerFactory:  informers.NewSharedInformerFactory(clientset, 30*time.Second),
		ingressClassName: className,
	}
}

// Run starts the controller's informer and HTTP server. It blocks until
// the provided context is cancelled.
//
// The startup sequence:
//  1. Register event handlers on the Ingress informer
//  2. Start the informer factory (begins watching the API)
//  3. Wait for the informer cache to sync (initial list completed)
//  4. Start the HTTP reverse proxy server
//  5. Block until context cancellation, then gracefully shut down
//
// Parameters:
//   - ctx: controls the lifecycle; cancel to trigger graceful shutdown
//   - listenAddr: the address to listen on (e.g., ":8080")
func (ic *IngressController) Run(ctx context.Context, listenAddr string) error {
	// Get the Ingress informer from the shared factory.
	// This informer watches all Ingress resources across all namespaces.
	ingressInformer := ic.informerFactory.Networking().V1().Ingresses().Informer()

	// Register event handlers. These are called by the informer whenever
	// Ingress resources are created, updated, or deleted.
	ingressInformer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		// AddFunc is called when a new Ingress is created.
		AddFunc: func(obj interface{}) {
			log.Printf("[INFO] Ingress added: %s/%s",
				obj.(*networkingv1.Ingress).Namespace,
				obj.(*networkingv1.Ingress).Name)
			ic.rebuildRoutes(ingressInformer.GetStore())
		},
		// UpdateFunc is called when an existing Ingress is modified.
		UpdateFunc: func(oldObj, newObj interface{}) {
			log.Printf("[INFO] Ingress updated: %s/%s",
				newObj.(*networkingv1.Ingress).Namespace,
				newObj.(*networkingv1.Ingress).Name)
			ic.rebuildRoutes(ingressInformer.GetStore())
		},
		// DeleteFunc is called when an Ingress is deleted.
		DeleteFunc: func(obj interface{}) {
			log.Printf("[INFO] Ingress deleted")
			ic.rebuildRoutes(ingressInformer.GetStore())
		},
	})

	// Start all registered informers. This begins the watch connection
	// to the Kubernetes API server.
	ic.informerFactory.Start(ctx.Done())

	// Wait for the informer cache to sync. This means the initial LIST
	// of all Ingress resources has completed, and our route table reflects
	// the current state of the cluster.
	log.Println("[INFO] Waiting for informer cache sync...")
	if !cache.WaitForCacheSync(ctx.Done(), ingressInformer.HasSynced) {
		return fmt.Errorf("failed to sync informer cache")
	}
	log.Println("[INFO] Informer cache synced")

	// Create the HTTP server with our routing handler.
	server := &http.Server{
		Addr:    listenAddr,
		Handler: ic.httpHandler(),
	}

	// Start the HTTP server in a goroutine so we can handle shutdown.
	go func() {
		log.Printf("[INFO] Listening on %s", listenAddr)
		if err := server.ListenAndServe(); err != http.ErrServerClosed {
			log.Fatalf("[FATAL] HTTP server error: %v", err)
		}
	}()

	// Block until context is cancelled (SIGINT/SIGTERM).
	<-ctx.Done()
	log.Println("[INFO] Shutting down...")

	// Graceful shutdown: wait up to 10 seconds for in-flight requests.
	shutdownCtx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()
	return server.Shutdown(shutdownCtx)
}

// rebuildRoutes iterates over all Ingress resources in the informer's cache
// and constructs a new route table. This is called whenever any Ingress
// resource changes.
//
// For each Ingress rule, it constructs a Service URL in the format:
//
//	http://<service-name>.<namespace>.svc.cluster.local:<port>
//
// This URL is used by the reverse proxy to forward traffic. Kubernetes
// DNS (CoreDNS) resolves this to the Service's ClusterIP or directly
// to Pod IPs (headless service).
//
// Parameters:
//   - store: the informer's local cache containing all known Ingress resources
func (ic *IngressController) rebuildRoutes(store cache.Store) {
	newRoutes := make(map[routeKey]*backend)

	// Iterate over every Ingress resource in the cache.
	for _, obj := range store.List() {
		ingress, ok := obj.(*networkingv1.Ingress)
		if !ok {
			continue
		}

		// Skip Ingresses that don't match our IngressClass.
		// This allows multiple controllers to coexist.
		if ingress.Spec.IngressClassName != nil &&
			*ingress.Spec.IngressClassName != ic.ingressClassName {
			continue
		}

		// Process each routing rule in the Ingress spec.
		for _, rule := range ingress.Spec.Rules {
			if rule.HTTP == nil {
				continue
			}
			for _, p := range rule.HTTP.Paths {
				// Construct the internal Service URL.
				// The Service DNS name follows the pattern:
				// <service>.<namespace>.svc.cluster.local
				svcURL := fmt.Sprintf("http://%s.%s.svc.cluster.local:%d",
					p.Backend.Service.Name,
					ingress.Namespace,
					p.Backend.Service.Port.Number,
				)
				u, err := url.Parse(svcURL)
				if err != nil {
					log.Printf("[ERROR] Failed to parse URL %s: %v", svcURL, err)
					continue
				}
				key := routeKey{
					host: rule.Host,
					path: p.Path,
				}
				newRoutes[key] = &backend{serviceURL: u}
				log.Printf("[INFO] Route: %s%s → %s", rule.Host, p.Path, svcURL)
			}
		}
	}

	// Atomically replace the entire route table.
	ic.routes.update(newRoutes)
	log.Printf("[INFO] Route table updated: %d routes", len(newRoutes))
}

// httpHandler returns an http.Handler that implements the reverse proxy.
// For each incoming request, it:
//  1. Extracts the Host header and URL path
//  2. Looks up the matching backend in the route table
//  3. Proxies the request to the backend using httputil.ReverseProxy
//  4. Returns 502 Bad Gateway if the backend is unreachable
//  5. Returns 404 Not Found if no route matches
//
// The handler adds standard proxy headers:
//   - X-Forwarded-For: the client's IP address
//   - X-Forwarded-Host: the original Host header
//   - X-Forwarded-Proto: the original protocol (http/https)
func (ic *IngressController) httpHandler() http.Handler {
	return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
		// Extract the hostname (strip port if present).
		host := r.Host
		if idx := strings.Index(host, ":"); idx != -1 {
			host = host[:idx]
		}

		// Look up the backend for this host and path.
		b := ic.routes.lookup(host, r.URL.Path)
		if b == nil {
			log.Printf("[WARN] No route for %s%s", host, r.URL.Path)
			http.Error(w, "No route found", http.StatusNotFound)
			return
		}

		// Create a reverse proxy targeting the backend Service.
		// httputil.NewSingleHostReverseProxy handles:
		//   - Rewriting the request URL to the backend
		//   - Copying request headers
		//   - Streaming the response back to the client
		proxy := httputil.NewSingleHostReverseProxy(b.serviceURL)

		// Customize the Director to add proxy headers.
		// The Director function modifies the outgoing request before it's sent.
		originalDirector := proxy.Director
		proxy.Director = func(req *http.Request) {
			originalDirector(req)
			// Set standard proxy headers so the backend knows about the
			// original client request.
			req.Header.Set("X-Forwarded-For", r.RemoteAddr)
			req.Header.Set("X-Forwarded-Host", r.Host)
			req.Header.Set("X-Forwarded-Proto", "http")
		}

		// Custom error handler: return 502 if the backend is unreachable.
		proxy.ErrorHandler = func(w http.ResponseWriter, r *http.Request, err error) {
			log.Printf("[ERROR] Proxy error for %s%s → %s: %v",
				host, r.URL.Path, b.serviceURL, err)
			http.Error(w, "Bad Gateway", http.StatusBadGateway)
		}

		log.Printf("[INFO] %s %s%s → %s", r.Method, host, r.URL.Path, b.serviceURL)
		proxy.ServeHTTP(w, r)
	})
}

// buildKubeClient creates a Kubernetes client. It first attempts to use
// in-cluster configuration (when running as a Pod), then falls back to
// the kubeconfig file (for local development).
//
// Returns:
//   - A Kubernetes clientset for API access
//   - An error if no valid configuration is found
func buildKubeClient() (kubernetes.Interface, error) {
	// Try in-cluster config first (when running as a Pod).
	// This uses the ServiceAccount token mounted at
	// /var/run/secrets/kubernetes.io/serviceaccount/token
	config, err := rest.InClusterConfig()
	if err != nil {
		// Fall back to kubeconfig (for local development).
		// Uses the KUBECONFIG env var or ~/.kube/config
		kubeconfig := os.Getenv("KUBECONFIG")
		if kubeconfig == "" {
			kubeconfig = os.Getenv("HOME") + "/.kube/config"
		}
		config, err = clientcmd.BuildConfigFromFlags("", kubeconfig)
		if err != nil {
			return nil, fmt.Errorf("failed to build kube config: %w", err)
		}
	}
	return kubernetes.NewForConfig(config)
}

// main is the entry point. It:
//  1. Creates a Kubernetes client
//  2. Creates the IngressController
//  3. Sets up signal handling for graceful shutdown
//  4. Runs the controller until SIGINT or SIGTERM
func main() {
	clientset, err := buildKubeClient()
	if err != nil {
		log.Fatalf("[FATAL] Failed to create Kubernetes client: %v", err)
	}

	// Create the controller. "minimal" is our IngressClass name.
	// Only Ingress resources with ingressClassName: "minimal" will be handled.
	controller := NewIngressController(clientset, "minimal")

	// Set up signal handling for graceful shutdown.
	ctx, cancel := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer cancel()

	// Run blocks until the context is cancelled.
	if err := controller.Run(ctx, ":8080"); err != nil {
		log.Fatalf("[FATAL] Controller error: %v", err)
	}
	log.Println("[INFO] Controller stopped")
}
```

### Deploying the Minimal Ingress Controller

```yaml
# Deploy our controller as a Pod in the cluster
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: minimal
spec:
  controller: example.com/minimal-ingress-controller
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: minimal-ingress-controller
  namespace: ingress-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: minimal-ingress-controller
rules:
- apiGroups: ["networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["services", "endpoints"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: minimal-ingress-controller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: minimal-ingress-controller
subjects:
- kind: ServiceAccount
  name: minimal-ingress-controller
  namespace: ingress-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minimal-ingress-controller
  namespace: ingress-system
spec:
  replicas: 1
  selector:
    matchLabels:
      app: minimal-ingress-controller
  template:
    metadata:
      labels:
        app: minimal-ingress-controller
    spec:
      serviceAccountName: minimal-ingress-controller
      containers:
      - name: controller
        image: minimal-ingress-controller:latest
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: minimal-ingress-controller
  namespace: ingress-system
spec:
  type: LoadBalancer
  selector:
    app: minimal-ingress-controller
  ports:
  - port: 80
    targetPort: 8080
```

### Testing the Controller

```bash
# Create an Ingress that targets our controller
kubectl apply -f - <<EOF
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-ingress
spec:
  ingressClassName: minimal     # Must match our controller's class
  rules:
  - host: test.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-frontend
            port:
              number: 80
EOF

# Check controller logs
kubectl logs -n ingress-system -l app=minimal-ingress-controller
# [INFO] Ingress added: default/test-ingress
# [INFO] Route: test.example.com/ → http://web-frontend.default.svc.cluster.local:80
# [INFO] Route table updated: 1 routes

# Send a test request
curl --resolve test.example.com:80:$CONTROLLER_IP http://test.example.com/
# [Controller log]: GET test.example.com/ → http://web-frontend.default.svc.cluster.local:80
```

---

## 24.10 Summary

```
  L7 Traffic Flow — Complete Picture:

  Internet
     │
     │  HTTPS request to api.example.com/v1/users
     ▼
  ┌─────────────────────────────────────────────────────────┐
  │  DNS Resolution                                          │
  │  api.example.com → 34.123.45.67 (ExternalDNS created)   │
  └────────────────────────────┬────────────────────────────┘
                               ▼
  ┌─────────────────────────────────────────────────────────┐
  │  Cloud Load Balancer (L4)                                │
  │  34.123.45.67:443 → NodePort 30443 on worker nodes      │
  └────────────────────────────┬────────────────────────────┘
                               ▼
  ┌─────────────────────────────────────────────────────────┐
  │  Ingress Controller / Gateway (L7)                       │
  │  1. TLS termination (cert from cert-manager)             │
  │  2. Host match: api.example.com                          │
  │  3. Path match: /v1/users (Prefix)                       │
  │  4. Rate limit check: OK                                 │
  │  5. Auth check: OAuth2 proxy → OK                        │
  │  6. Select backend: users-service (round-robin)          │
  │  7. Proxy to Pod IP: 10.244.2.9:8080                     │
  └────────────────────────────┬────────────────────────────┘
                               ▼
  ┌─────────────────────────────────────────────────────────┐
  │  Kubernetes Networking (L3/L4) — Chapter 23              │
  │  Pod-to-Pod delivery via CNI plugin                      │
  └────────────────────────────┬────────────────────────────┘
                               ▼
  ┌─────────────────────────────────────────────────────────┐
  │  Backend Pod (users-service)                             │
  │  Receives HTTP request, processes, returns response      │
  └─────────────────────────────────────────────────────────┘
```

**Key takeaways:**

1. **Ingress provides L7 routing** — host-based, path-based, with TLS termination.
   A single LoadBalancer can serve hundreds of services.

2. **The Ingress resource is declarative; the controller is imperative.** Without a
   controller, Ingress resources do nothing.

3. **Gateway API is the successor to Ingress.** It adds typed configuration, role
   separation (GatewayClass → Gateway → Routes), traffic splitting, header matching,
   and support for TCP/TLS/gRPC.

4. **cert-manager automates TLS.** ACME (Let's Encrypt) challenges, certificate issuance,
   and renewal are fully automated.

5. **Advanced L7 features** (rate limiting, auth, CORS, rewrites) are available through
   controller-specific annotations (Ingress) or policy attachments (Gateway API).

6. **ExternalDNS closes the loop** by automatically creating DNS records pointing to
   your Ingress or Gateway's external IP.

7. **Building an ingress controller is surprisingly straightforward** — it's a Kubernetes
   informer + a reverse proxy. The complexity in production controllers comes from edge
   cases: TLS, WebSockets, gRPC, health checks, connection draining, and configuration
   hot-reloading.

In the next chapter, we'll explore service meshes — what happens when you extend the
proxy concept from the edge (Ingress) to every Pod in the cluster (sidecar proxies),
gaining mutual TLS, observability, and fine-grained traffic control across all
service-to-service communication.
