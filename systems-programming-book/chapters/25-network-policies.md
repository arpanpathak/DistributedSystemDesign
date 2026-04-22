# Chapter 25: Network Policies — Microsegmentation

> *"A castle with no walls is just a house. A cluster with no network policies is
> just a flat network waiting for lateral movement."*

In previous chapters we explored how Kubernetes networking provides a flat,
routable network where every Pod can reach every other Pod. That design is
elegant for development, but terrifying for production. A single compromised
container can freely probe every database, cache, and internal API in the
cluster.

**Network Policies** are Kubernetes' answer to microsegmentation — fine-grained
firewall rules that restrict which Pods can talk to which other Pods, on which
ports, in which direction. They are the foundation of **zero-trust networking**
inside the cluster.

This chapter takes you from "why do I need these?" through the spec, default
policies, real-world scenarios, enforcement internals, advanced policy types,
and rigorous testing methodology.

---

## 25.1 Why Network Policies

### 25.1.1 The Flat Network Problem

By default, Kubernetes enforces a single networking rule:

> **Every Pod can communicate with every other Pod across every namespace,
> without NAT.**

This is defined in the Kubernetes networking model (see Chapter 22). The CNI
plugin — Calico, Cilium, Flannel, or others — implements this flat network.

```text
┌─────────────────────────────────────────────────────────┐
│                   Kubernetes Cluster                     │
│                                                         │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │ frontend │◄──►│ backend  │◄──►│ database │          │
│  │ Pod      │    │ Pod      │    │ Pod      │          │
│  └──────────┘    └──────────┘    └──────────┘          │
│       ▲               ▲               ▲                │
│       │               │               │                │
│       ▼               ▼               ▼                │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │ attacker │───►│ redis    │───►│ logging  │          │
│  │ Pod      │    │ Pod      │    │ Pod      │          │
│  └──────────┘    └──────────┘    └──────────┘          │
│                                                         │
│  ALL arrows are valid — every Pod can reach every Pod   │
└─────────────────────────────────────────────────────────┘
```

In this diagram, an attacker who compromises the `frontend` Pod can directly
connect to the `database` Pod. There is no network-level barrier preventing
lateral movement.

### 25.1.2 The Zero-Trust Principle

Zero-trust networking means:

1. **No implicit trust** — just because two Pods are in the same cluster does
   not mean they should communicate.
2. **Explicit allow** — every communication path must be explicitly permitted.
3. **Least privilege** — Pods should only have the network access they need.

Network Policies implement zero-trust at the Pod level:

```text
┌─────────────────────────────────────────────────────────┐
│                   With Network Policies                  │
│                                                         │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │ frontend │───►│ backend  │───►│ database │          │
│  │ Pod      │    │ Pod      │    │ Pod      │          │
│  └──────────┘    └──────────┘    └──────────┘          │
│       ▲               ▲               ▲                │
│       │               │               │                │
│       ╳               ╳               ╳                │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐          │
│  │ attacker │ ╳  │ redis    │    │ logging  │          │
│  │ Pod      │    │ Pod      │    │ Pod      │          │
│  └──────────┘    └──────────┘    └──────────┘          │
│                                                         │
│  ╳ = blocked by NetworkPolicy                           │
│  Only explicitly allowed paths work                     │
└─────────────────────────────────────────────────────────┘
```

### 25.1.3 The CNI Requirement — Policies Are Not Magic

**Critical**: A NetworkPolicy resource is just YAML stored in etcd. The
**enforcement** is done by the CNI plugin. If your CNI does not support Network
Policies, the YAML will be accepted by the API server but **will have no
effect**.

| CNI Plugin | Network Policy Support | Enforcement Mechanism |
|------------|----------------------|----------------------|
| Calico     | ✅ Full              | iptables or eBPF     |
| Cilium     | ✅ Full + L7          | eBPF                 |
| Weave Net  | ✅ Full              | iptables             |
| Flannel    | ❌ None              | N/A                  |
| AWS VPC CNI| ⚠️ Partial (with Calico add-on) | iptables  |
| Azure CNI  | ✅ With Azure NPM    | iptables             |

> **Warning**: If you are using Flannel and create NetworkPolicy objects, the
> API server will happily accept them. `kubectl get networkpolicy` will show
> them. But no traffic will ever be blocked. This is one of the most common
> mistakes in production clusters.

To check if your CNI supports Network Policies, try deploying a deny-all policy
and testing connectivity. If traffic still flows, your CNI is not enforcing.

---

## 25.2 NetworkPolicy Spec Deep Dive

The `NetworkPolicy` resource lives in the `networking.k8s.io/v1` API group.
Let's dissect every field.

### 25.2.1 The Complete Structure

```yaml
# ──────────────────────────────────────────────────────────
# NetworkPolicy — complete annotated structure
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example-policy          # Policy name — make it descriptive
  namespace: production         # Namespace scope — policies are namespaced
spec:
  # ── podSelector ──────────────────────────────────────────
  # Which Pods in THIS namespace does this policy apply to?
  # An empty selector {} means "all Pods in this namespace."
  podSelector:
    matchLabels:
      app: backend
      tier: api

  # ── policyTypes ──────────────────────────────────────────
  # Which direction(s) does this policy control?
  # - Ingress: incoming traffic to the selected Pods
  # - Egress: outgoing traffic from the selected Pods
  # If omitted, Kubernetes infers from presence of ingress/egress rules.
  policyTypes:
    - Ingress
    - Egress

  # ── ingress ──────────────────────────────────────────────
  # Rules for incoming traffic. Each rule is an OR — if ANY
  # rule matches, traffic is allowed. Within a rule, 'from'
  # and 'ports' are ANDed.
  ingress:
    - from:
        # ── podSelector ────────────────────────────────────
        # Allow traffic from Pods matching these labels
        # in the SAME namespace as the policy.
        - podSelector:
            matchLabels:
              app: frontend

        # ── namespaceSelector ──────────────────────────────
        # Allow traffic from Pods in namespaces matching
        # these labels. Combined with podSelector for
        # cross-namespace rules.
        - namespaceSelector:
            matchLabels:
              environment: production

        # ── ipBlock ────────────────────────────────────────
        # Allow traffic from specific IP CIDR ranges.
        # 'except' carves out sub-ranges to deny.
        - ipBlock:
            cidr: 10.0.0.0/8
            except:
              - 10.0.1.0/24
      ports:
        # ── ports ──────────────────────────────────────────
        # Which ports are allowed? This is ANDed with 'from'.
        - protocol: TCP
          port: 8080
        - protocol: TCP
          port: 8443

  # ── egress ───────────────────────────────────────────────
  # Rules for outgoing traffic. Same structure as ingress
  # but with 'to' instead of 'from'.
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
    # Allow DNS — critical for almost every Pod
    - to: []
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

### 25.2.2 podSelector — Targeting Pods

The `podSelector` determines which Pods in the **policy's namespace** are
affected by this policy.

```yaml
# ── Select specific Pods by label ──────────────────────────
podSelector:
  matchLabels:
    app: backend

# ── Select ALL Pods in the namespace ───────────────────────
# An empty selector matches everything.
podSelector: {}

# ── Select Pods with matchExpressions ──────────────────────
podSelector:
  matchExpressions:
    - key: tier
      operator: In
      values:
        - frontend
        - backend
    - key: environment
      operator: NotIn
      values:
        - test
```

**Key points**:

- `podSelector` only selects Pods in the **same namespace** as the policy.
- An empty `podSelector: {}` selects **all Pods** in the namespace.
- Multiple `matchLabels` are ANDed — all must match.
- `matchExpressions` provides `In`, `NotIn`, `Exists`, `DoesNotExist`.

### 25.2.3 policyTypes — Ingress, Egress, or Both

```yaml
# ── Ingress only ───────────────────────────────────────────
# Only incoming traffic rules are evaluated.
# Outgoing traffic is unaffected by this policy.
policyTypes:
  - Ingress

# ── Egress only ────────────────────────────────────────────
# Only outgoing traffic rules are evaluated.
# Incoming traffic is unaffected by this policy.
policyTypes:
  - Egress

# ── Both ───────────────────────────────────────────────────
# Both directions are controlled by this policy.
policyTypes:
  - Ingress
  - Egress
```

**Inference rules when `policyTypes` is omitted**:

| Condition | Inferred policyTypes |
|-----------|---------------------|
| `ingress` rules present, no `egress` | `[Ingress]` |
| `egress` rules present, no `ingress` | `[Egress]` |
| Both present | `[Ingress, Egress]` |
| Neither present | `[Ingress]` |

> **Best practice**: Always specify `policyTypes` explicitly. Relying on
> inference leads to subtle bugs when you modify the policy later.

### 25.2.4 The from/to Selectors

Each ingress rule has a `from` array, and each egress rule has a `to` array.
Each element in these arrays is one of three types:

#### podSelector — Same Namespace

```yaml
# ── Allow from Pods with label app=frontend ────────────────
# in the SAME namespace as this policy
from:
  - podSelector:
      matchLabels:
        app: frontend
```

#### namespaceSelector — Cross-Namespace

```yaml
# ── Allow from ALL Pods in namespaces labeled env=prod ─────
from:
  - namespaceSelector:
      matchLabels:
        environment: production
```

#### Combined podSelector + namespaceSelector — Specific Pods in Specific Namespaces

**This is one of the most common sources of confusion.** The placement of
selectors matters:

```yaml
# ── OPTION A: Two separate selectors (OR logic) ───────────
# Allow from:
#   - ANY Pod with app=frontend in THIS namespace, OR
#   - ANY Pod in ANY namespace labeled env=prod
from:
  - podSelector:
      matchLabels:
        app: frontend
  - namespaceSelector:
      matchLabels:
        environment: production

# ── OPTION B: Combined selector (AND logic) ────────────────
# Allow from:
#   - Pods with app=frontend that are ALSO in a
#     namespace labeled env=prod
from:
  - podSelector:
      matchLabels:
        app: frontend
    namespaceSelector:
      matchLabels:
        environment: production
```

The difference is **one list element vs two list elements**. In YAML, the
leading `-` determines whether they are separate items (OR) or the same item
(AND).

```
TWO items → OR:          ONE item → AND:
from:                     from:
  - podSelector:  ← item 1   - podSelector:    ← same
      ...                       ...
  - namespaceSelector: ← item 2  namespaceSelector: ← same item!
      ...                       ...
```

#### ipBlock — External Traffic

```yaml
# ── Allow from specific CIDR, excluding a sub-range ────────
from:
  - ipBlock:
      cidr: 172.16.0.0/16
      except:
        - 172.16.1.0/24
```

`ipBlock` is primarily for traffic external to the cluster. Pod-to-Pod traffic
within the cluster should use `podSelector` or `namespaceSelector`.

### 25.2.5 Ports — Protocol and Port

```yaml
ports:
  # ── TCP port by number ───────────────────────────────────
  - protocol: TCP
    port: 8080

  # ── UDP port ─────────────────────────────────────────────
  - protocol: UDP
    port: 53

  # ── Named port ───────────────────────────────────────────
  # References the port name from the Pod spec.
  # Useful when different Pods use different port numbers
  # for the same logical service.
  - protocol: TCP
    port: http

  # ── Port range (K8s 1.25+) ──────────────────────────────
  - protocol: TCP
    port: 32000
    endPort: 32768

  # ── SCTP (if supported by CNI) ───────────────────────────
  - protocol: SCTP
    port: 3868
```

### 25.2.6 Missing Field vs Empty Field — The Critical Distinction

This is where most Network Policy bugs originate. The semantics of missing
fields versus empty fields are radically different:

```yaml
# ── SCENARIO 1: ingress field is MISSING ───────────────────
# No ingress rules at all. If Ingress is in policyTypes,
# ALL ingress is DENIED. If Ingress is NOT in policyTypes,
# ingress is UNAFFECTED by this policy.
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  # No 'ingress' field at all → deny all ingress

# ── SCENARIO 2: ingress field is EMPTY ARRAY ──────────────
# Same effect as missing — deny all ingress (if Ingress
# is in policyTypes).
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress: []        # Empty array → deny all ingress

# ── SCENARIO 3: ingress has one rule with empty from ───────
# An ingress rule with no 'from' restriction allows ALL
# sources. This is "allow all ingress."
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - {}             # A rule that matches everything → allow all

# ── SCENARIO 4: ingress has a rule with empty from array ───
# Same as scenario 3 — empty 'from' means all sources.
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from: []       # Empty from → all sources allowed
      ports:
        - protocol: TCP
          port: 80
```

Summary table:

| Field State | Meaning |
|-------------|---------|
| `ingress` missing | Deny all ingress (if Ingress in policyTypes) |
| `ingress: []` | Deny all ingress |
| `ingress: [{}]` | Allow all ingress from any source |
| `ingress: [{from: []}]` | Allow all ingress from any source |
| `egress` missing | Deny all egress (if Egress in policyTypes) |
| `egress: []` | Deny all egress |
| `egress: [{}]` | Allow all egress to any destination |

### 25.2.7 How Multiple Policies Combine

Network Policies are **additive**:

- If **no** policy selects a Pod, that Pod has unrestricted traffic.
- If **any** policy selects a Pod for a given direction (ingress/egress), then
  only traffic matching **at least one** policy's rules is allowed.
- Multiple policies selecting the same Pod **union** their allow rules.
- There is **no way** to write a standard NetworkPolicy that overrides another
  policy's allow rule. You can only add more allows.

```
Policy A allows: frontend → backend:8080
Policy B allows: monitoring → backend:9090

Result: Both frontend:8080 AND monitoring:9090 are allowed.
        Everything else to backend is denied (because at least
        one policy selects backend for Ingress).
```

---

## 25.3 Default Policies

Default policies are the building blocks of a zero-trust network. Apply them
first, then layer specific allow rules on top.

### 25.3.1 Default Deny All Ingress

```yaml
# ──────────────────────────────────────────────────────────
# Default Deny All Ingress
# ──────────────────────────────────────────────────────────
# Selects ALL Pods in the namespace (podSelector: {}).
# Declares Ingress policy type but provides NO ingress rules.
# Effect: all incoming traffic to Pods in this namespace is
# denied unless another policy explicitly allows it.
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
  namespace: production
spec:
  podSelector: {}       # All Pods in namespace
  policyTypes:
    - Ingress           # Control ingress direction
  # No 'ingress' field → deny all incoming traffic
```

### 25.3.2 Default Deny All Egress

```yaml
# ──────────────────────────────────────────────────────────
# Default Deny All Egress
# ──────────────────────────────────────────────────────────
# Selects ALL Pods in the namespace.
# Declares Egress policy type but provides NO egress rules.
# Effect: all outgoing traffic from Pods in this namespace
# is denied unless another policy explicitly allows it.
#
# WARNING: This also blocks DNS! Pods will not be able to
# resolve service names. You MUST pair this with a DNS
# allow policy (see Section 25.4.3).
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  # No 'egress' field → deny all outgoing traffic
```

### 25.3.3 Default Deny All Traffic (Both Directions)

```yaml
# ──────────────────────────────────────────────────────────
# Default Deny All Traffic
# ──────────────────────────────────────────────────────────
# The combination of deny-ingress and deny-egress in one
# policy. This is the strongest default posture.
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
  # No ingress or egress rules → deny everything
```

### 25.3.4 Default Allow All Ingress

```yaml
# ──────────────────────────────────────────────────────────
# Default Allow All Ingress
# ──────────────────────────────────────────────────────────
# Useful when another policy denies ingress and you want to
# temporarily open it back up for debugging, or when you
# want to explicitly document that a namespace is open.
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-allow-ingress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - {}                # A single rule with no restrictions
```

### 25.3.5 Default Allow All Egress

```yaml
# ──────────────────────────────────────────────────────────
# Default Allow All Egress
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-allow-egress
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - {}
```

---

## 25.4 Policy Examples — Real-World Scenarios

### 25.4.1 Allow Only Frontend to Talk to Backend

```yaml
# ──────────────────────────────────────────────────────────
# Allow Frontend → Backend on port 8080
# ──────────────────────────────────────────────────────────
# The backend Pods only accept ingress from Pods labeled
# app=frontend, on TCP port 8080.
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-allow-frontend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 8080
```

### 25.4.2 Allow Backend to Talk to Database

```yaml
# ──────────────────────────────────────────────────────────
# Allow Backend → Database on port 5432
# ──────────────────────────────────────────────────────────
# Two policies needed:
# 1. Egress on backend allowing traffic TO database:5432
# 2. Ingress on database allowing traffic FROM backend
# ──────────────────────────────────────────────────────────

# Policy 1: Backend egress to database
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress-to-database
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: database
      ports:
        - protocol: TCP
          port: 5432
    # Don't forget DNS!
    - ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53

---
# Policy 2: Database ingress from backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-allow-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: database
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 5432
```

### 25.4.3 Allow DNS Egress for All Pods (Critical!)

```yaml
# ──────────────────────────────────────────────────────────
# Allow DNS Egress — ALWAYS apply this with deny-egress
# ──────────────────────────────────────────────────────────
# Without this policy, Pods cannot resolve DNS names.
# Services, external APIs, and even other Pods (via service
# names) become unreachable.
#
# CoreDNS runs in kube-system namespace. We allow egress
# to it on UDP/TCP port 53.
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns-egress
  namespace: production
spec:
  podSelector: {}       # All Pods in namespace
  policyTypes:
    - Egress
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
```

> **Note**: The combined `namespaceSelector` + `podSelector` (in the same list
> item) means "Pods labeled `k8s-app=kube-dns` in namespace `kube-system`."
> This is more precise than allowing all traffic on port 53.

### 25.4.4 Restrict Egress to Specific External IPs

```yaml
# ──────────────────────────────────────────────────────────
# Allow egress only to a specific external payment API
# ──────────────────────────────────────────────────────────
# The payment-processor Pod can only reach:
#   - DNS (for resolution)
#   - The payment gateway at 203.0.113.0/24 on port 443
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-egress-restricted
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: payment-processor
  policyTypes:
    - Egress
  egress:
    # Allow DNS
    - ports:
        - protocol: UDP
          port: 53
        - protocol: TCP
          port: 53
    # Allow HTTPS to payment gateway only
    - to:
        - ipBlock:
            cidr: 203.0.113.0/24
      ports:
        - protocol: TCP
          port: 443
```

### 25.4.5 Allow Monitoring Namespace to Scrape All Pods

```yaml
# ──────────────────────────────────────────────────────────
# Allow Prometheus (in monitoring namespace) to scrape
# metrics from all Pods in production namespace.
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-monitoring-scrape
  namespace: production
spec:
  podSelector: {}        # All Pods in production
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9090      # Prometheus metrics port
        - protocol: TCP
          port: 9091      # Pushgateway port
```

### 25.4.6 Namespace Isolation — Deny Cross-Namespace Traffic

```yaml
# ──────────────────────────────────────────────────────────
# Namespace Isolation — only allow traffic within namespace
# ──────────────────────────────────────────────────────────
# Denies all ingress EXCEPT from Pods in the same namespace.
# This effectively creates a namespace boundary.
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: namespace-isolation
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        # podSelector with no namespaceSelector means
        # "same namespace only". Empty podSelector means
        # "all Pods in that namespace."
        - podSelector: {}
```

### 25.4.7 Complete 3-Tier Application Policy Set

Here is a complete set of policies for a production 3-tier application:

```
┌─────────────────────────────────────────────────────────┐
│                    production namespace                   │
│                                                         │
│  Internet ──► Ingress Controller                        │
│                     │                                   │
│                     ▼ (port 8080)                       │
│              ┌──────────┐                               │
│              │ frontend │──────► DNS (kube-system)      │
│              │ app=fe   │                               │
│              └────┬─────┘                               │
│                   │ (port 3000)                         │
│                   ▼                                     │
│              ┌──────────┐                               │
│              │ backend  │──────► DNS (kube-system)      │
│              │ app=be   │                               │
│              └────┬─────┘                               │
│                   │ (port 5432)                         │
│                   ▼                                     │
│              ┌──────────┐                               │
│              │ postgres │──────► DNS (kube-system)      │
│              │ app=db   │                               │
│              └──────────┘                               │
│                                                         │
│  monitoring ──► All Pods (port 9090) for Prometheus     │
└─────────────────────────────────────────────────────────┘
```

```yaml
# ──────────────────────────────────────────────────────────
# Policy 1: Default deny all traffic in production
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
---
# ──────────────────────────────────────────────────────────
# Policy 2: Allow DNS for all Pods
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
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
---
# ──────────────────────────────────────────────────────────
# Policy 3: Frontend — allow ingress from ingress controller
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: fe
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
---
# ──────────────────────────────────────────────────────────
# Policy 4: Frontend — allow egress to backend
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-egress-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: fe
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: be
      ports:
        - protocol: TCP
          port: 3000
---
# ──────────────────────────────────────────────────────────
# Policy 5: Backend — allow ingress from frontend
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: be
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: fe
      ports:
        - protocol: TCP
          port: 3000
---
# ──────────────────────────────────────────────────────────
# Policy 6: Backend — allow egress to database
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-egress-to-db
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: be
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: db
      ports:
        - protocol: TCP
          port: 5432
---
# ──────────────────────────────────────────────────────────
# Policy 7: Database — allow ingress from backend only
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: database-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: db
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: be
      ports:
        - protocol: TCP
          port: 5432
---
# ──────────────────────────────────────────────────────────
# Policy 8: Allow Prometheus scraping from monitoring ns
# ──────────────────────────────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scrape
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app: prometheus
      ports:
        - protocol: TCP
          port: 9090
```

---

## 25.5 How Network Policies Are Enforced

### 25.5.1 The Enforcement Architecture

Network Policies are not enforced by the Kubernetes control plane. The flow is:

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  kubectl     │────►│  API Server  │────►│    etcd       │
│  apply NP    │     │  (validates) │     │  (stores NP)  │
└──────────────┘     └──────────────┘     └──────────────┘
                                                 │
                                                 │ Watch
                                                 ▼
                                          ┌──────────────┐
                                          │  CNI Plugin   │
                                          │  Controller   │
                                          │  (e.g. Calico │
                                          │   felix)      │
                                          └──────┬───────┘
                                                 │
                                    Translates NP to:
                                                 │
                          ┌──────────────────────┼──────────────────────┐
                          │                      │                      │
                          ▼                      ▼                      ▼
                    ┌───────────┐          ┌───────────┐          ┌───────────┐
                    │ iptables  │          │  eBPF     │          │  ipsets   │
                    │ rules     │          │ programs  │          │           │
                    │ (Calico)  │          │ (Cilium)  │          │ (Calico)  │
                    └───────────┘          └───────────┘          └───────────┘
```

### 25.5.2 Calico Enforcement — iptables

Calico's Felix agent watches NetworkPolicy resources and translates them into
iptables rules on each node. Let's examine what this looks like.

**Hands-on: Examining iptables rules created by Calico**

```bash
# ── Step 1: Deploy a simple deny-all policy ────────────────
kubectl create namespace netpol-demo
kubectl -n netpol-demo apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
EOF

# ── Step 2: Deploy a test Pod ──────────────────────────────
kubectl -n netpol-demo run test --image=nginx --labels="app=test"
kubectl -n netpol-demo wait --for=condition=ready pod/test

# ── Step 3: SSH into the node where the Pod is scheduled ───
NODE=$(kubectl -n netpol-demo get pod test -o jsonpath='{.spec.nodeName}')
ssh $NODE

# ── Step 4: List Calico-specific iptables chains ──────────
# Calico creates chains prefixed with "cali-" for each
# workload endpoint.
sudo iptables-save | grep -c "cali-"
# Typically hundreds of rules

# ── Step 5: Find the chain for our Pod's interface ─────────
# Each Pod gets a veth pair; Calico names them "cali*"
VETH=$(ip link | grep cali | head -1 | awk '{print $2}' | tr -d ':@')
sudo iptables -L cali-tw-${VETH} -n --line-numbers

# ── Example output ─────────────────────────────────────────
# Chain cali-tw-cali1234abcd (1 references)
# num  target     prot opt source    destination
# 1    MARK       all  --  0.0.0.0/0  0.0.0.0/0  MARK and 0x0
# 2    cali-pri-  all  --  0.0.0.0/0  0.0.0.0/0  /* Policy */
# 3    RETURN     all  --  0.0.0.0/0  0.0.0.0/0  MARK match 0x10000/0x10000
# 4    DROP       all  --  0.0.0.0/0  0.0.0.0/0  /* Drop if no policy match */
```

Key observations:
- Calico creates **per-interface chains** for each Pod workload.
- Packets traverse policy chains; if no rule marks the packet as allowed, it
  hits a final `DROP`.
- IP sets are used for efficient matching against large numbers of Pod IPs.

### 25.5.3 Cilium Enforcement — eBPF

Cilium takes a fundamentally different approach: instead of iptables, it
attaches eBPF programs to the network interfaces.

```
┌────────────────────────────────────────────────────┐
│                     Node                            │
│                                                    │
│  ┌──────────┐     ┌──────────────┐     ┌────────┐ │
│  │   Pod    │────►│   eBPF prog  │────►│  veth  │ │
│  │          │     │  (tc ingress │     │  pair  │ │
│  │          │     │   classifier)│     │        │ │
│  └──────────┘     └──────────────┘     └────────┘ │
│                          │                         │
│                   Policy decision:                 │
│                   ALLOW / DROP                     │
│                                                    │
│  Advantages:                                       │
│  - No iptables rule traversal overhead             │
│  - O(1) map lookups vs O(n) rule chains            │
│  - L7 visibility without proxying all traffic      │
│  - Identity-based (not IP-based) enforcement       │
└────────────────────────────────────────────────────┘
```

Cilium assigns each Pod a **security identity** (a numeric ID derived from its
labels). eBPF programs enforce policies by checking the identity of the source
or destination, not the IP address. This avoids the race condition where a Pod
gets a new IP before policies update.

```bash
# ── Inspect Cilium endpoint policies ──────────────────────
# Each Pod is a "Cilium endpoint" with its own policy map.
kubectl -n kube-system exec -it ds/cilium -- cilium endpoint list

# ── View the BPF policy map for a specific endpoint ───────
kubectl -n kube-system exec -it ds/cilium -- \
  cilium bpf policy get <endpoint-id>
```

---

## 25.6 AdminNetworkPolicy and BaselineAdminNetworkPolicy

### 25.6.1 The Limitation of Standard NetworkPolicies

Standard NetworkPolicies are **namespaced**. This creates problems:

1. **Namespace admins can override cluster security** — a namespace admin can
   create a policy that allows all traffic, undermining cluster-wide security.
2. **No priority** — policies are purely additive; you cannot express "deny this
   even if a namespace policy allows it."
3. **Cross-cutting concerns** — policies like "all namespaces must deny traffic
   from namespace X" require a policy in every namespace.

### 25.6.2 AdminNetworkPolicy (ANP)

`AdminNetworkPolicy` is a **cluster-scoped** resource with explicit **priority**
ordering. It is part of the `policy.networking.k8s.io` API group (currently
in alpha/beta, check your cluster version).

```yaml
# ──────────────────────────────────────────────────────────
# AdminNetworkPolicy — cluster-level mandatory policy
# ──────────────────────────────────────────────────────────
# This policy is evaluated BEFORE any namespace-level
# NetworkPolicy. Lower priority numbers = higher precedence.
# ──────────────────────────────────────────────────────────
apiVersion: policy.networking.k8s.io/v1alpha1
kind: AdminNetworkPolicy
metadata:
  name: deny-from-untrusted
spec:
  priority: 10           # Lower = higher precedence
  subject:
    # Which Pods does this policy apply to?
    namespaces:
      matchLabels:
        environment: production
  ingress:
    - name: "deny-from-untrusted-namespaces"
      action: Deny       # ANP supports Deny, Allow, and Pass
      from:
        - namespaces:
            matchLabels:
              trust-level: untrusted
  egress:
    - name: "allow-dns"
      action: Allow
      to:
        - namespaces:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - portNumber:
            protocol: UDP
            port: 53
```

Key differences from standard NetworkPolicy:

| Feature | NetworkPolicy | AdminNetworkPolicy |
|---------|--------------|-------------------|
| Scope | Namespace | Cluster |
| Actions | Allow only (additive) | Allow, Deny, Pass |
| Priority | No priority | Explicit priority number |
| Override | Cannot override others | Higher priority overrides |
| Creator | Namespace admin | Cluster admin |

The three actions:

- **Allow**: Permit the traffic immediately. No further evaluation needed.
- **Deny**: Drop the traffic immediately. No further evaluation needed.
- **Pass**: Delegate to the next lower-priority ANP, then BaselineANP, then
  standard NetworkPolicies.

### 25.6.3 BaselineAdminNetworkPolicy (BANP)

`BaselineAdminNetworkPolicy` provides **default cluster-wide rules** that are
evaluated **after** all AdminNetworkPolicies and standard NetworkPolicies. There
can be at most one BANP in a cluster, and it must be named `default`.

```yaml
# ──────────────────────────────────────────────────────────
# BaselineAdminNetworkPolicy — cluster-wide defaults
# ──────────────────────────────────────────────────────────
# Evaluated LAST — after all ANPs and standard NPs.
# Acts as a safety net for traffic that no other policy
# matched. Only Allow and Deny actions (no Pass).
# ──────────────────────────────────────────────────────────
apiVersion: policy.networking.k8s.io/v1alpha1
kind: BaselineAdminNetworkPolicy
metadata:
  name: default           # Must be named "default"
spec:
  subject:
    namespaces: {}        # All namespaces
  ingress:
    - name: "default-deny-all-ingress"
      action: Deny
      from:
        - namespaces: {}
  egress:
    - name: "default-deny-all-egress"
      action: Deny
      to:
        - namespaces: {}
```

### 25.6.4 Policy Evaluation Order

```
Incoming packet
      │
      ▼
┌─────────────────────────────────┐
│  AdminNetworkPolicy (by priority)│
│  Priority 1 → 2 → 3 → ...      │
│                                  │
│  Action: Allow → ALLOW           │
│  Action: Deny  → DROP            │
│  Action: Pass  → continue ↓      │
└─────────────────┬───────────────┘
                  │ (no match or Pass)
                  ▼
┌─────────────────────────────────┐
│  Standard NetworkPolicy          │
│  (namespace-scoped, additive)    │
│                                  │
│  If any NP selects Pod:          │
│    Match found → ALLOW           │
│    No match    → DROP            │
│  If no NP selects Pod:           │
│    → continue ↓                  │
└─────────────────┬───────────────┘
                  │ (no NP selects Pod)
                  ▼
┌─────────────────────────────────┐
│  BaselineAdminNetworkPolicy      │
│  (single cluster-wide default)   │
│                                  │
│  Action: Allow → ALLOW           │
│  Action: Deny  → DROP            │
│  No match      → ALLOW (default) │
└─────────────────────────────────┘
```

---

## 25.7 Cilium Network Policies — Beyond K8s Standard

### 25.7.1 Why Cilium Extends the Standard

Standard Kubernetes NetworkPolicies operate at **Layer 3/4** — IP addresses and
ports. But many real-world security requirements operate at **Layer 7**:

- "Allow HTTP GET to `/api/v1/health` but deny POST to `/api/v1/admin`"
- "Allow gRPC calls to `UserService.GetUser` but deny `UserService.DeleteUser`"
- "Allow Kafka produce to topic `events` but deny consume from `secrets`"

Cilium implements `CiliumNetworkPolicy` (CNP) and `CiliumClusterwideNetworkPolicy`
(CCNP) to support these use cases.

### 25.7.2 CiliumNetworkPolicy — L7 HTTP Policies

```yaml
# ──────────────────────────────────────────────────────────
# CiliumNetworkPolicy — L7 HTTP path filtering
# ──────────────────────────────────────────────────────────
# Allow only GET requests to /api/v1/public/* paths.
# Block all POST, PUT, DELETE, and other paths.
# ──────────────────────────────────────────────────────────
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: api-l7-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: api-server
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: frontend
      toPorts:
        - ports:
            - port: "8080"
              protocol: TCP
          rules:
            http:
              # Allow GET to public API paths only
              - method: GET
                path: "/api/v1/public/.*"
              # Allow health checks
              - method: GET
                path: "/healthz"
```

### 25.7.3 CiliumNetworkPolicy — gRPC Filtering

```yaml
# ──────────────────────────────────────────────────────────
# CiliumNetworkPolicy — gRPC method filtering
# ──────────────────────────────────────────────────────────
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: grpc-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: user-service
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: api-gateway
      toPorts:
        - ports:
            - port: "50051"
              protocol: TCP
          rules:
            http:
              # gRPC uses HTTP/2 — Cilium treats gRPC paths
              # as HTTP paths in the form /package.Service/Method
              - method: POST
                path: "/user.UserService/GetUser"
              - method: POST
                path: "/user.UserService/ListUsers"
              # DeleteUser is NOT listed — it is denied
```

### 25.7.4 FQDN-Based Egress Policies

```yaml
# ──────────────────────────────────────────────────────────
# CiliumNetworkPolicy — FQDN-based egress
# ──────────────────────────────────────────────────────────
# Allow egress only to specific external domains.
# Cilium intercepts DNS responses to learn the IPs
# associated with each FQDN.
# ──────────────────────────────────────────────────────────
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: egress-fqdn-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: payment-service
  egress:
    # Allow DNS first (required for FQDN resolution)
    - toEndpoints:
        - matchLabels:
            k8s:io.kubernetes.pod.namespace: kube-system
            k8s-app: kube-dns
      toPorts:
        - ports:
            - port: "53"
              protocol: ANY
          rules:
            dns:
              - matchPattern: "*.stripe.com"
              - matchPattern: "*.paypal.com"
    # Allow HTTPS to the resolved FQDNs
    - toFQDNs:
        - matchPattern: "*.stripe.com"
        - matchPattern: "*.paypal.com"
      toPorts:
        - ports:
            - port: "443"
              protocol: TCP
```

### 25.7.5 Kafka-Aware Policies

```yaml
# ──────────────────────────────────────────────────────────
# CiliumNetworkPolicy — Kafka topic filtering
# ──────────────────────────────────────────────────────────
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: kafka-policy
  namespace: production
spec:
  endpointSelector:
    matchLabels:
      app: kafka-broker
  ingress:
    - fromEndpoints:
        - matchLabels:
            app: event-producer
      toPorts:
        - ports:
            - port: "9092"
              protocol: TCP
          rules:
            kafka:
              - role: produce
                topic: "events"
              - role: produce
                topic: "audit-log"
              # Consumer role is NOT listed — denied
```

---

## 25.8 Testing Network Policies

### 25.8.1 The Testing Methodology

Network Policies are notoriously easy to get wrong. A systematic testing
approach is essential.

```
┌──────────────────────────────────────────────────────────┐
│           Network Policy Testing Workflow                 │
│                                                          │
│  1. Deploy default deny policies                         │
│  2. Deploy allow policies for your application           │
│  3. For EACH expected communication path:                │
│     a. Verify it WORKS (positive test)                   │
│  4. For EACH path that should be BLOCKED:                │
│     b. Verify it FAILS (negative test)                   │
│  5. Specifically test DNS resolution                     │
│  6. Test from multiple source Pods/namespaces            │
└──────────────────────────────────────────────────────────┘
```

### 25.8.2 The Netshoot Pod — Your Network Testing Swiss Army Knife

```bash
# ── Deploy a netshoot pod for testing ──────────────────────
# nicolaka/netshoot contains: curl, wget, nslookup, dig,
# ping, traceroute, tcpdump, iperf3, nmap, and more.
kubectl -n production run netshoot \
  --image=nicolaka/netshoot \
  --labels="app=netshoot,role=tester" \
  -it --rm -- /bin/bash

# ── Inside the netshoot pod ────────────────────────────────

# Test 1: DNS resolution
nslookup backend-svc.production.svc.cluster.local

# Test 2: TCP connectivity to backend
curl -v --connect-timeout 5 http://backend-svc:3000/health

# Test 3: TCP connectivity to database (should be blocked)
nc -zv postgres-svc 5432 -w 5
# Expected: Connection timed out (if policy denies)

# Test 4: Egress to external
curl -v --connect-timeout 5 https://example.com
# Expected: Timeout if egress is denied
```

### 25.8.3 Systematic Testing Script

```bash
#!/bin/bash
# ──────────────────────────────────────────────────────────
# Network Policy Test Suite
# ──────────────────────────────────────────────────────────
# Usage: ./test-netpol.sh <namespace>
# Deploys test pods and verifies connectivity matrix.
# ──────────────────────────────────────────────────────────

NAMESPACE=${1:-production}
TIMEOUT=5

# Colors for output
GREEN='\033[0;32m'
RED='\033[0;31m'
NC='\033[0m'

pass() { echo -e "${GREEN}[PASS]${NC} $1"; }
fail() { echo -e "${RED}[FAIL]${NC} $1"; }

# ── Test function ──────────────────────────────────────────
# Runs a connectivity test from a source pod to a target.
# $1: source pod label, $2: target host, $3: port,
# $4: expected result (pass/fail)
test_connectivity() {
  local src_label=$1 target=$2 port=$3 expected=$4
  local desc="$src_label → $target:$port (expect: $expected)"

  result=$(kubectl -n $NAMESPACE exec \
    $(kubectl -n $NAMESPACE get pod -l $src_label -o name | head -1) \
    -- timeout $TIMEOUT sh -c "nc -zv $target $port 2>&1" 2>&1)

  if echo "$result" | grep -q "succeeded\|open"; then
    if [ "$expected" = "pass" ]; then
      pass "$desc"
    else
      fail "$desc — connection succeeded but should have been blocked"
    fi
  else
    if [ "$expected" = "fail" ]; then
      pass "$desc — correctly blocked"
    else
      fail "$desc — connection failed but should have succeeded"
    fi
  fi
}

echo "=== Network Policy Test Suite ==="
echo "Namespace: $NAMESPACE"
echo ""

# ── Positive tests (should succeed) ───────────────────────
echo "--- Positive Tests (should connect) ---"
test_connectivity "app=fe" "backend-svc" "3000" "pass"
test_connectivity "app=be" "postgres-svc" "5432" "pass"

# ── Negative tests (should fail) ──────────────────────────
echo ""
echo "--- Negative Tests (should be blocked) ---"
test_connectivity "app=fe" "postgres-svc" "5432" "fail"
test_connectivity "app=netshoot" "backend-svc" "3000" "fail"
test_connectivity "app=netshoot" "postgres-svc" "5432" "fail"

# ── DNS tests ──────────────────────────────────────────────
echo ""
echo "--- DNS Tests ---"
dns_result=$(kubectl -n $NAMESPACE exec \
  $(kubectl -n $NAMESPACE get pod -l app=fe -o name | head -1) \
  -- timeout $TIMEOUT nslookup backend-svc 2>&1)

if echo "$dns_result" | grep -q "Address:"; then
  pass "DNS resolution from frontend"
else
  fail "DNS resolution from frontend"
fi
```

### 25.8.4 Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Forgot DNS egress rule | All service names fail to resolve | Add egress allow for port 53 |
| Wrong label selector | Policy has no effect | Check `kubectl get pods --show-labels` |
| OR vs AND confusion in from/to | Too much or too little traffic allowed | Check YAML indentation carefully |
| Policy in wrong namespace | No effect on target Pods | Policies are namespace-scoped |
| CNI doesn't support policies | All traffic still flows | Switch to Calico or Cilium |
| Forgot egress policy | Pods can't reach external services | Add explicit egress allow rules |
| Port name mismatch | Traffic blocked despite "correct" policy | Use numeric ports for clarity |

---

## 25.9 Network Policy Best Practices

### 25.9.1 The Golden Rules

1. **Start with default deny**. Apply a deny-all policy to every namespace
   before deploying any workloads. This ensures that any new Pod is locked down
   by default.

2. **Allow DNS first**. After default deny, the very next policy should allow
   DNS egress. Without DNS, nothing works — service discovery, external API
   calls, and even `nslookup` for debugging.

3. **Be explicit about every communication path**. If frontend talks to backend,
   you need:
   - Egress from frontend to backend (direction: out from frontend)
   - Ingress to backend from frontend (direction: in to backend)

4. **Use descriptive policy names**. `backend-allow-frontend-ingress` is far
   better than `policy-1`.

5. **Label everything**. Network Policies select Pods by labels. If your Pods
   are not labeled consistently, policies become impossible to write correctly.

6. **Document policies alongside application manifests**. Keep network policies
   in the same Git repository as the application they protect. Review them in
   the same PR.

7. **Audit regularly**. Use tools like:
   - `kubectl get networkpolicy -A` — list all policies across namespaces
   - Cilium's Hubble UI — visualize actual traffic flows
   - Calico's `calicoctl` — inspect computed policy for a specific endpoint
   - Network Policy Editor (https://editor.networkpolicy.io) — visualize YAML

### 25.9.2 Policy Organization Pattern

```
app-repo/
├── manifests/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── network-policies/
│       ├── 00-default-deny.yaml      # Applied first
│       ├── 01-allow-dns.yaml         # DNS access
│       ├── 02-frontend-ingress.yaml  # Frontend rules
│       ├── 03-backend-ingress.yaml   # Backend rules
│       ├── 04-backend-egress.yaml    # Backend → DB
│       ├── 05-database-ingress.yaml  # Database rules
│       └── 06-monitoring.yaml        # Prometheus access
```

### 25.9.3 Namespace Labels for Cross-Namespace Policies

Kubernetes 1.21+ automatically labels namespaces with
`kubernetes.io/metadata.name: <name>`. Use this for cross-namespace policies:

```yaml
# ── Target Pods in a specific namespace by name ────────────
from:
  - namespaceSelector:
      matchLabels:
        kubernetes.io/metadata.name: monitoring
```

For older clusters, manually label namespaces:

```bash
kubectl label namespace monitoring purpose=monitoring
kubectl label namespace production environment=production
```

### 25.9.4 GitOps and Network Policies

Network Policies are perfect for GitOps because:

- They are declarative YAML.
- They are idempotent — applying the same policy twice has no effect.
- Changes are visible in Git diffs.
- Rollback is `git revert` + `kubectl apply`.

```yaml
# In your Argo CD Application or Flux Kustomization:
# Include network policies in the sync path.
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: production-app
spec:
  source:
    path: manifests/       # Contains both app and network-policies/
    repoURL: https://github.com/org/app-repo
  destination:
    namespace: production
    server: https://kubernetes.default.svc
```

---

## 25.10 Hands-On Lab: Building Zero-Trust from Scratch

This lab walks through building a zero-trust network for a complete application.

### Step 1: Create the namespace and deploy the application

```bash
# ── Create namespace ───────────────────────────────────────
kubectl create namespace zero-trust-lab

# ── Deploy a 3-tier application ────────────────────────────
kubectl -n zero-trust-lab apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
        tier: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.25
          ports:
            - containerPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
        tier: api
    spec:
      containers:
        - name: backend
          image: hashicorp/http-echo
          args: ["-text=hello from backend", "-listen=:3000"]
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
spec:
  selector:
    app: backend
  ports:
    - port: 3000
      targetPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
spec:
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
EOF
```

### Step 2: Verify open connectivity (before policies)

```bash
# ── From a test pod, verify everything is reachable ────────
kubectl -n zero-trust-lab run tester --image=nicolaka/netshoot \
  --rm -it -- sh -c "
    echo '--- Testing DNS ---'
    nslookup backend-svc.zero-trust-lab.svc.cluster.local
    echo '--- Testing backend ---'
    curl -s --connect-timeout 3 http://backend-svc:3000
    echo '--- Testing frontend ---'
    curl -s --connect-timeout 3 http://frontend-svc:80 | head -5
    echo '--- Testing external ---'
    curl -s --connect-timeout 3 -o /dev/null -w '%{http_code}' https://example.com
  "
```

### Step 3: Apply default deny

```bash
kubectl -n zero-trust-lab apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
EOF
```

### Step 4: Verify everything is now blocked

```bash
kubectl -n zero-trust-lab run tester --image=nicolaka/netshoot \
  --rm -it -- sh -c "
    echo '--- Testing DNS (should fail) ---'
    timeout 3 nslookup backend-svc.zero-trust-lab.svc.cluster.local || echo 'DNS BLOCKED'
    echo '--- Testing backend (should fail) ---'
    timeout 3 curl -s http://backend-svc:3000 || echo 'BACKEND BLOCKED'
  "
```

### Step 5: Add policies incrementally

```bash
# ── Allow DNS first ────────────────────────────────────────
kubectl -n zero-trust-lab apply -f - <<'EOF'
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
EOF

# ── Allow frontend → backend ──────────────────────────────
kubectl -n zero-trust-lab apply -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend-egress
spec:
  podSelector:
    matchLabels:
      app: frontend
  policyTypes:
    - Egress
  egress:
    - to:
        - podSelector:
            matchLabels:
              app: backend
      ports:
        - protocol: TCP
          port: 3000
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: backend-from-frontend-ingress
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: frontend
      ports:
        - protocol: TCP
          port: 3000
EOF
```

### Step 6: Verify the final state

```bash
# ── From frontend: backend should be reachable ─────────────
kubectl -n zero-trust-lab exec deploy/frontend -- \
  sh -c "curl -s --connect-timeout 3 http://backend-svc:3000"
# Expected: "hello from backend"

# ── From tester: backend should be blocked ─────────────────
kubectl -n zero-trust-lab run tester --image=nicolaka/netshoot \
  --labels="app=tester" --rm -it -- sh -c "
    timeout 3 curl -s http://backend-svc:3000 || echo 'CORRECTLY BLOCKED'
  "
```

---

## 25.11 Summary

| Concept | Key Takeaway |
|---------|-------------|
| Default behavior | All Pods can talk to all Pods — flat network |
| NetworkPolicy | Namespace-scoped, additive-only, L3/L4 |
| CNI requirement | No enforcement without supporting CNI |
| podSelector | Targets Pods in the policy's namespace |
| policyTypes | Always specify explicitly |
| Missing vs empty | Missing rules = deny; empty rule `{}` = allow all |
| Default deny | First policy to apply in every namespace |
| DNS egress | Second policy — without it, nothing works |
| AdminNetworkPolicy | Cluster-scoped, priority-based, can deny |
| CiliumNetworkPolicy | L7 policies: HTTP paths, gRPC methods, Kafka topics |
| Testing | Systematic positive + negative tests with netshoot |

The next chapter covers DNS in Kubernetes — CoreDNS Deep Dive, which is
intimately connected to network policies (since DNS egress is the first thing
you must allow after a default deny).

---

## Further Reading

- [Kubernetes Network Policy Documentation](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Network Policy Editor — Visual Tool](https://editor.networkpolicy.io/)
- [Calico Network Policy Documentation](https://docs.tigera.io/calico/latest/network-policy/)
- [Cilium Network Policy Documentation](https://docs.cilium.io/en/stable/security/)
- [AdminNetworkPolicy KEP](https://github.com/kubernetes/enhancements/tree/master/keps/sig-network/2091-admin-network-policy)
- [Ahmet's Kubernetes Network Policy Recipes](https://github.com/ahmetb/kubernetes-network-policy-recipes)
