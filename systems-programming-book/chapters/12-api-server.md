# Chapter 12: API Server, Admission Controllers & API Machinery

> *"The API server is the front door, the bouncer, the receptionist, and the
> filing clerk of your Kubernetes cluster — all rolled into one."*

Every object you have ever created — every Pod, every Deployment, every
ConfigMap — passed through a single component before it reached etcd. That
component is **kube-apiserver**. It authenticates you, authorises your action,
mutates your request, validates it, persists it, and notifies every interested
controller. Nothing in a Kubernetes cluster happens without the API server
knowing about it.

In this chapter we will pull the API server apart piece by piece. We will trace
a request from the moment it leaves `kubectl` to the moment the bytes land in
etcd. We will write admission webhooks in Go, explore CEL-based policies,
build resource watchers, and get comfortable with the `apimachinery` library
that underpins the entire Kubernetes type system.

---

## 12.1 kube-apiserver — The Central Hub

### The Spoke-and-Hub Architecture

Kubernetes follows a strict **hub-and-spoke** communication model. The API
server sits at the centre; every other component — kubelet, kube-scheduler,
kube-controller-manager, cloud-controller-manager, and every custom controller
you write — communicates exclusively through the API server. Components never
talk to each other directly.

```text
                          ┌─────────────────┐
                          │   kube-apiserver │
                          │   (the hub)      │
                          └──┬───┬───┬───┬───┘
                             │   │   │   │
              ┌──────────────┘   │   │   └──────────────┐
              │                  │   │                   │
              ▼                  ▼   ▼                   ▼
        ┌──────────┐    ┌────────┐  ┌──────────┐  ┌───────────┐
        │ kubelet  │    │ sched- │  │controller│  │  your     │
        │ (node 1) │    │ uler   │  │ manager  │  │ controller│
        └──────────┘    └────────┘  └──────────┘  └───────────┘
              │
              ▼
        ┌──────────┐
        │ kubelet  │
        │ (node 2) │
        └──────────┘
```

Why this model?

1. **Single source of truth** — etcd is accessed exclusively through the API
   server, giving you a consistent, serialised view of cluster state.
2. **Uniform authentication and authorisation** — every request, no matter
   which component issues it, passes through the same auth chain.
3. **Auditability** — every mutation is logged in a single place.
4. **Extensibility** — admission webhooks and API aggregation let you inject
   custom logic without modifying core components.

### RESTful API over HTTPS

The API server exposes a RESTful HTTP/2 API secured with TLS. Resources are
addressed by URL:

```text
GET /api/v1/namespaces/default/pods/nginx-abc123
│    │      │                      │
│    │      │                      └─ resource name
│    │      └─ namespace
│    └─ API group + version (core group = /api/v1)
└─ HTTP verb
```

### HTTP Verbs → Kubernetes Verbs

Every HTTP request maps to a Kubernetes verb. The API server's authorisation
layer works with K8s verbs, not raw HTTP methods:

```text
┌──────────────────────┬──────────────────────┬──────────────────────────────┐
│ HTTP Method          │ Kubernetes Verb      │ Description                  │
├──────────────────────┼──────────────────────┼──────────────────────────────┤
│ GET  (single)        │ get                  │ Retrieve one resource        │
│ GET  (collection)    │ list                 │ Retrieve all in a namespace  │
│ GET  (?watch=true)   │ watch                │ Stream change events         │
│ POST                 │ create               │ Create a new resource        │
│ PUT                  │ update               │ Replace the entire object    │
│ PATCH                │ patch                │ Partially update an object   │
│ DELETE               │ delete               │ Delete a single resource     │
│ DELETE (collection)  │ deletecollection     │ Delete many resources        │
└──────────────────────┴──────────────────────┴──────────────────────────────┘
```

Notice that `list` and `watch` are not separate HTTP methods — they are both
GET requests. The API server distinguishes them based on query parameters.

---

## 12.2 Request Lifecycle — The Full Pipeline

When you run `kubectl apply -f pod.yaml`, the resulting HTTPS request passes
through a meticulously ordered pipeline inside the API server. Every stage can
reject the request. Let's trace the entire journey.

```text
┌──────────────────────────────────────────────────────────────────────────┐
│                      kube-apiserver request pipeline                     │
│                                                                          │
│  ┌─────────┐   ┌──────────┐   ┌──────────────┐   ┌──────────────────┐  │
│  │ TLS     │──▶│ Authen-  │──▶│ Authorisa-   │──▶│ Mutating         │  │
│  │ Handshk │   │ tication │   │ tion (RBAC)  │   │ Admission        │  │
│  └─────────┘   └──────────┘   └──────────────┘   └────────┬─────────┘  │
│                                                             │            │
│                                                             ▼            │
│  ┌─────────┐   ┌──────────┐   ┌──────────────┐   ┌──────────────────┐  │
│  │ Response│◀──│  etcd    │◀──│ Validating   │◀──│ Object Schema    │  │
│  │ to      │   │  persist │   │ Admission    │   │ Validation       │  │
│  │ client  │   └──────────┘   └──────────────┘   └──────────────────┘  │
│  └─────────┘                                                             │
└──────────────────────────────────────────────────────────────────────────┘
```

### Stage-by-Stage Breakdown

**1. TLS Termination**
The API server requires HTTPS. The TLS handshake establishes encryption and,
if the client presents a certificate, begins the authentication process.

**2. Authentication** — *"Who are you?"*
Multiple authenticators run in sequence. The first one that succeeds sets the
request's user info (username, UID, groups). If all authenticators fail, the
request is rejected with HTTP 401.

**3. Authorisation** — *"Are you allowed to do this?"*
The authoriser (typically RBAC) checks whether the authenticated user has
permission to perform the requested verb on the requested resource. A denied
request gets HTTP 403.

**4. Mutating Admission** — *"Let me adjust that for you."*
Mutating admission controllers can modify the incoming object. They run in
a defined order. Webhook-based mutating admission runs here. Examples:
injecting sidecar containers, adding default labels, setting default storage
classes.

**5. Object Schema Validation**
The API server validates the object against its OpenAPI schema. Malformed
objects are rejected with HTTP 422.

**6. Validating Admission** — *"Is this acceptable?"*
Validating admission controllers check the (now mutated) object against
policy. They cannot modify the object — only accept or reject. Webhook-based
validating admission runs here. CEL-based ValidatingAdmissionPolicy also runs
at this stage.

**7. etcd Persistence**
The validated object is serialised (typically as protobuf) and written to etcd
via a transaction. The API server uses the etcd revision number as the
resource's `resourceVersion`.

**8. Response**
The API server returns the persisted object to the client, including
server-populated fields like `metadata.uid`, `metadata.creationTimestamp`,
and `metadata.resourceVersion`.

### Watch Notification (Async)

After persistence, the watch cache broadcasts the event to all active
watchers. This is how controllers learn about changes — not by polling, but
by maintaining long-lived watch connections.

---

## 12.3 Authentication

Authentication answers a simple question: **who is making this request?**

The API server supports multiple authentication strategies simultaneously.
They are tried in order; the first one that succeeds wins.

### X.509 Client Certificates

The most common authentication method for cluster components. The client
presents a TLS certificate during the handshake. The API server checks:

1. Is the certificate signed by a trusted Certificate Authority (CA)?
2. Is the certificate still valid (not expired, not revoked)?

If valid, the **Common Name (CN)** becomes the username and the
**Organisation (O)** fields become groups:

```text
Subject: CN=system:kube-scheduler, O=system:kube-scheduler
         └─ username              └─ group
```

Your kubeconfig's `client-certificate-data` and `client-key-data` fields use
this mechanism.

### Bearer Tokens

Bearer tokens are sent in the `Authorization: Bearer <token>` header.

**ServiceAccount Tokens:**
Every pod gets a projected ServiceAccount token mounted at
`/var/run/secrets/kubernetes.io/serviceaccount/token`. These are short-lived
JWTs signed by the API server's private key. The token payload contains:

```json
{
  "iss": "https://kubernetes.default.svc",
  "sub": "system:serviceaccount:default:my-sa",
  "aud": ["https://kubernetes.default.svc"],
  "exp": 1700000000,
  "iat": 1699996400
}
```

**OIDC Tokens:**
For human users, organisations often configure the API server with an OIDC
provider (e.g., Azure AD, Google, Dex). The API server validates the JWT
signature using the provider's public keys:

```
kube-apiserver \
  --oidc-issuer-url=https://accounts.google.com \
  --oidc-client-id=my-k8s-cluster \
  --oidc-username-claim=email \
  --oidc-groups-claim=groups
```

### Authentication Proxy

A front-proxy (like an ingress or identity-aware proxy) authenticates the user
and passes identity information via HTTP headers:

```
X-Remote-User: jane@example.com
X-Remote-Group: developers
X-Remote-Group: team-alpha
```

The API server trusts these headers only if the proxy's TLS certificate is
signed by the CA specified in `--requestheader-client-ca-file`.

### Webhook Token Authentication

For custom authentication, you can configure a webhook. The API server sends
a `TokenReview` object to your webhook; your service responds with the user
info:

```yaml
# webhook-config.yaml
apiVersion: v1
kind: Config
clusters:
  - name: authn-webhook
    cluster:
      server: https://auth.example.com/authenticate
      certificate-authority: /etc/kubernetes/pki/webhook-ca.pem
users:
  - name: authn-webhook
current-context: authn-webhook
contexts:
  - context:
      cluster: authn-webhook
      user: authn-webhook
    name: authn-webhook
```

### Anonymous Authentication

If all authenticators fail and anonymous authentication is enabled (it is by
default), the request proceeds as `system:anonymous` in the
`system:unauthenticated` group. RBAC typically denies most actions for this
identity.

### How Multiple Authenticators Are Chained

```
Request arrives
    │
    ├─▶ X.509 Client Cert?      ──── YES ──▶ Authenticated ✓
    │   (check during TLS handshake)
    │
    ├─▶ Bearer Token?            ──── YES ──▶ Authenticated ✓
    │   (try ServiceAccount, OIDC, Webhook)
    │
    ├─▶ Auth Proxy Headers?      ──── YES ──▶ Authenticated ✓
    │   (X-Remote-User present?)
    │
    ├─▶ Anonymous Enabled?       ──── YES ──▶ system:anonymous ✓
    │
    └─▶ All failed               ──── 401 Unauthorized
```

Each authenticator independently decides *"I can handle this"* or *"not my
problem, pass."* The first positive match is final.

### Hands-On: Examining Your Kubeconfig Authentication

```bash
# See which authentication method your current context uses
kubectl config view --minify -o yaml

# Decode the client certificate to see your identity
kubectl config view --minify --raw -o jsonpath='{.users[0].user.client-certificate-data}' \
  | base64 -d \
  | openssl x509 -text -noout \
  | grep -E 'Subject:|Issuer:'

# Output:
#   Issuer: CN = kubernetes
#   Subject: O = system:masters, CN = kubernetes-admin

# Check who the API server thinks you are
kubectl auth whoami    # Kubernetes 1.28+

# Or the older approach:
kubectl get --raw /apis/authentication.k8s.io/v1/selfsubjectreviews \
  -X POST -H "Content-Type: application/json" \
  -d '{"apiVersion":"authentication.k8s.io/v1","kind":"SelfSubjectReview","metadata":{}}'
```

---

## 12.4 Authorisation

Once the API server knows *who* you are, it must decide *what you are allowed
to do*.

### Authorisation Modes

The API server can be configured with multiple authorisers:

```
kube-apiserver \
  --authorization-mode=Node,RBAC,Webhook
```

They are evaluated in order. Each returns one of three decisions:

```
┌────────────┬──────────────────────────────────────────────────────────┐
│ Decision   │ Meaning                                                  │
├────────────┼──────────────────────────────────────────────────────────┤
│ Allow      │ The action is permitted. No further authorisers checked. │
│ Deny       │ The action is explicitly denied. Skip remaining.         │
│ NoOpinion  │ This authoriser has no opinion. Try the next one.        │
└────────────┴──────────────────────────────────────────────────────────┘
```

If all authorisers return NoOpinion, the request is denied.

### RBAC (Role-Based Access Control)

RBAC is the most common authorisation mode. It uses four resource types:

- **Role** — grants permissions within a single namespace
- **ClusterRole** — grants permissions cluster-wide
- **RoleBinding** — binds a Role to users/groups/SAs in a namespace
- **ClusterRoleBinding** — binds a ClusterRole cluster-wide

We cover RBAC in depth in Chapter 27. Here is a quick example:

```yaml
# Allow the "deployer" ServiceAccount to manage Deployments in "production"
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: production
  name: deployment-manager
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  namespace: production
  name: deployer-can-manage-deployments
subjects:
  - kind: ServiceAccount
    name: deployer
    namespace: production
roleRef:
  kind: Role
  name: deployment-manager
  apiGroup: rbac.authorization.k8s.io
```

### ABAC (Attribute-Based Access Control)

ABAC uses a static policy file. It is rarely used in production because
changes require restarting the API server:

```json
{"apiVersion": "abac.authorization.kubernetes.io/v1beta1", "kind": "Policy",
 "spec": {"user": "alice", "namespace": "default", "resource": "pods", "readonly": true}}
```

### Webhook Authorisation

For custom authorisation logic, configure a webhook. The API server sends a
`SubjectAccessReview` to your service:

```json
{
  "apiVersion": "authorization.k8s.io/v1",
  "kind": "SubjectAccessReview",
  "spec": {
    "user": "jane",
    "groups": ["developers"],
    "resourceAttributes": {
      "namespace": "production",
      "verb": "delete",
      "resource": "pods"
    }
  }
}
```

### Node Authorisation

A special-purpose authoriser that grants kubelets access only to resources
related to pods scheduled on their node. This follows the principle of least
privilege — a compromised kubelet on Node A cannot read secrets for pods on
Node B.

### Hands-On: Testing Authorisation

```bash
# Can I create deployments in the default namespace?
kubectl auth can-i create deployments --namespace=default
# yes

# Can I delete nodes? (probably not unless you're cluster-admin)
kubectl auth can-i delete nodes
# no

# Check another user's permissions (requires escalation privileges)
kubectl auth can-i list secrets --namespace=kube-system --as=system:serviceaccount:default:my-sa
# no

# List everything you CAN do in a namespace
kubectl auth can-i --list --namespace=default

# Output:
# Resources                          Non-Resource URLs   Resource Names   Verbs
# *.*                                []                  []               [*]
# ...
```

---

## 12.5 Admission Controllers — The Gatekeepers

Admission controllers are plugins that intercept requests **after
authentication and authorisation** but **before the object is persisted to
etcd**. They are the last line of defence and the most powerful extension
point in the API server.

### What Admission Controllers Do

They serve two purposes:

1. **Mutating** — modify the object (add labels, inject sidecars, set
   defaults)
2. **Validating** — reject objects that violate policy (missing labels,
   disallowed images, resource limits)

Some built-in controllers do both.

### Execution Order

```
Mutating Admission Controllers (in order)
    │
    ├─▶ Built-in mutating controllers
    ├─▶ MutatingAdmissionWebhook (your webhooks)
    │
    ▼
Object Schema Validation
    │
    ▼
Validating Admission Controllers (in order)
    │
    ├─▶ Built-in validating controllers
    ├─▶ ValidatingAdmissionWebhook (your webhooks)
    ├─▶ ValidatingAdmissionPolicy (CEL-based)
    │
    ▼
Persist to etcd
```

### Key Built-In Admission Controllers

```
┌──────────────────────┬────────┬─────────────────────────────────────────┐
│ Controller           │ Type   │ What It Does                            │
├──────────────────────┼────────┼─────────────────────────────────────────┤
│ NamespaceLifecycle   │ Valid. │ Rejects requests in terminating NS      │
│ LimitRanger          │ Mut.   │ Applies default resource limits         │
│ ServiceAccount       │ Mut.   │ Auto-mounts SA tokens into pods         │
│ DefaultStorageClass  │ Mut.   │ Adds default SC to PVCs                 │
│ ResourceQuota        │ Valid. │ Enforces namespace resource quotas       │
│ PodSecurity          │ Valid. │ Enforces Pod Security Standards          │
│ MutatingAdmission    │ Mut.   │ Calls your mutating webhooks            │
│   Webhook            │        │                                         │
│ ValidatingAdmission  │ Valid. │ Calls your validating webhooks          │
│   Webhook            │        │                                         │
└──────────────────────┴────────┴─────────────────────────────────────────┘
```

You can see which controllers are enabled:

```bash
# Default enabled controllers (Kubernetes 1.30+)
kube-apiserver --help 2>&1 | grep -A5 'enable-admission-plugins'
```

### Admission Webhooks — Extending with Custom Logic

The webhook-based admission controllers are the primary way to inject custom
admission logic. They call external HTTPS endpoints that you operate.

**MutatingAdmissionWebhook:**
Receives an `AdmissionReview` request, returns a JSON patch that modifies the
object.

**ValidatingAdmissionWebhook:**
Receives an `AdmissionReview` request, returns an allow/deny decision.

#### Webhook Configuration

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: sidecar-injector
webhooks:
  - name: sidecar-injector.example.com
    # When does this webhook fire?
    rules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE"]
        resources: ["pods"]
        scope: "Namespaced"
    # Which objects to match
    namespaceSelector:
      matchLabels:
        sidecar-injection: enabled
    objectSelector:
      matchExpressions:
        - key: app
          operator: Exists
    # Where to send the request
    clientConfig:
      service:
        name: sidecar-injector
        namespace: admission-system
        path: "/mutate"
        port: 443
      caBundle: <base64-encoded-CA-cert>
    # What happens if the webhook is unreachable?
    failurePolicy: Fail     # or Ignore
    # Which version of AdmissionReview to send
    admissionReviewVersions: ["v1"]
    # How to match — Exact or Equivalent
    matchPolicy: Equivalent
    # Reinvocation policy — IfNeeded allows re-invocation after other
    # mutating webhooks modify the object
    reinvocationPolicy: IfNeeded
    sideEffects: None
    timeoutSeconds: 10
```

Key configuration fields:

- **`rules`** — which API operations and resources trigger the webhook
- **`namespaceSelector`** — only trigger for objects in matching namespaces
- **`objectSelector`** — only trigger for objects with matching labels
- **`failurePolicy`** — `Fail` (reject if webhook unreachable) or `Ignore`
  (allow if webhook unreachable). **Always use Fail in production.**
- **`sideEffects`** — whether the webhook has side effects beyond the
  admission decision. `None` enables dry-run support.
- **`timeoutSeconds`** — maximum time to wait (default 10, max 30)

---

### Hands-On: Building a Mutating Admission Webhook in Go

Let's build a webhook that injects an Envoy sidecar container into every pod
created in namespaces labelled `sidecar-injection: enabled`.

```go
// File: cmd/sidecar-injector/main.go
//
// sidecar-injector is a Kubernetes mutating admission webhook that
// automatically injects an Envoy proxy sidecar container into pods.
// It listens for AdmissionReview requests on /mutate and returns
// a JSON patch that adds the sidecar container to the pod spec.
//
// Usage:
//
//	go run ./cmd/sidecar-injector \
//	  --tls-cert=/etc/webhook/certs/tls.crt \
//	  --tls-key=/etc/webhook/certs/tls.key \
//	  --port=8443
package main

import (
	"encoding/json"
	"flag"
	"fmt"
	"io"
	"log"
	"net/http"

	admissionv1 "k8s.io/api/admission/v1"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/resource"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/serializer"
)

// codecs is the universal deserialiser used to decode AdmissionReview
// objects from the raw bytes received in the webhook HTTP body.
var codecs = serializer.NewCodecFactory(runtime.NewScheme())

// sidecarContainer defines the Envoy proxy sidecar that will be injected
// into every matching pod. We define it once and clone it for each
// admission request to avoid accidental mutation of the template.
var sidecarContainer = corev1.Container{
	Name:  "envoy-sidecar",
	Image: "envoyproxy/envoy:v1.28-latest",
	// Expose the Envoy admin interface on port 15000 for debugging
	// and the proxy listener on port 15001.
	Ports: []corev1.ContainerPort{
		{
			Name:          "envoy-admin",
			ContainerPort: 15000,
			Protocol:      corev1.ProtocolTCP,
		},
		{
			Name:          "envoy-proxy",
			ContainerPort: 15001,
			Protocol:      corev1.ProtocolTCP,
		},
	},
	// Set conservative resource limits so the sidecar doesn't starve
	// the main application container.
	Resources: corev1.ResourceRequirements{
		Requests: corev1.ResourceList{
			corev1.ResourceCPU:    resource.MustParse("50m"),
			corev1.ResourceMemory: resource.MustParse("64Mi"),
		},
		Limits: corev1.ResourceList{
			corev1.ResourceCPU:    resource.MustParse("200m"),
			corev1.ResourceMemory: resource.MustParse("128Mi"),
		},
	},
}

// jsonPatchOp represents a single RFC 6902 JSON Patch operation.
// The API server applies these patches to the incoming object before
// persisting it to etcd.
type jsonPatchOp struct {
	Op    string      `json:"op"`              // "add", "remove", "replace"
	Path  string      `json:"path"`            // JSON pointer to the field
	Value interface{} `json:"value,omitempty"` // the new value (for add/replace)
}

// handleMutate is the HTTP handler for the /mutate endpoint. It:
//  1. Reads and decodes the AdmissionReview from the request body.
//  2. Extracts the pod from the AdmissionReview.
//  3. Checks whether the sidecar is already injected (idempotency).
//  4. Builds a JSON patch to inject the sidecar container.
//  5. Returns an AdmissionReview response with the patch.
func handleMutate(w http.ResponseWriter, r *http.Request) {
	// ── Step 1: Read the request body ──────────────────────────────
	body, err := io.ReadAll(r.Body)
	if err != nil {
		log.Printf("ERROR: failed to read request body: %v", err)
		http.Error(w, "failed to read body", http.StatusBadRequest)
		return
	}
	defer r.Body.Close()

	// ── Step 2: Decode the AdmissionReview ─────────────────────────
	// The API server sends an AdmissionReview object. We must decode
	// it, extract the embedded pod, and return our response in the
	// same AdmissionReview wrapper.
	var admissionReview admissionv1.AdmissionReview
	if err := json.Unmarshal(body, &admissionReview); err != nil {
		log.Printf("ERROR: failed to unmarshal admission review: %v", err)
		http.Error(w, "failed to unmarshal", http.StatusBadRequest)
		return
	}

	// Safety check: the request must be present.
	if admissionReview.Request == nil {
		log.Println("ERROR: admission review has no request")
		http.Error(w, "missing request", http.StatusBadRequest)
		return
	}

	req := admissionReview.Request

	// ── Step 3: Decode the pod from the raw object ─────────────────
	var pod corev1.Pod
	if err := json.Unmarshal(req.Object.Raw, &pod); err != nil {
		log.Printf("ERROR: failed to unmarshal pod: %v", err)
		sendAdmissionResponse(w, req.UID, false, "failed to unmarshal pod", nil)
		return
	}

	log.Printf("INFO: processing pod %s/%s", pod.Namespace, pod.Name)

	// ── Step 4: Check for existing sidecar (idempotency) ───────────
	// If the sidecar container is already present, allow without
	// modification. This prevents duplicate injection when the
	// webhook is called multiple times (reinvocationPolicy: IfNeeded).
	for _, c := range pod.Spec.Containers {
		if c.Name == sidecarContainer.Name {
			log.Printf("INFO: sidecar already present in %s/%s, skipping",
				pod.Namespace, pod.Name)
			sendAdmissionResponse(w, req.UID, true, "", nil)
			return
		}
	}

	// ── Step 5: Build the JSON patch ───────────────────────────────
	// We construct a JSON Patch (RFC 6902) that appends the sidecar
	// container to the pod's container list. If the containers array
	// is empty (unlikely but possible), we initialise it.
	var patches []jsonPatchOp

	if len(pod.Spec.Containers) == 0 {
		// If there are no containers (shouldn't happen for valid pods),
		// create the array with just the sidecar.
		patches = append(patches, jsonPatchOp{
			Op:    "add",
			Path:  "/spec/containers",
			Value: []corev1.Container{sidecarContainer},
		})
	} else {
		// Append the sidecar to the existing containers list.
		// The "-" in the JSON pointer means "append to array".
		patches = append(patches, jsonPatchOp{
			Op:    "add",
			Path:  "/spec/containers/-",
			Value: sidecarContainer,
		})
	}

	// Also add an annotation indicating the sidecar was injected.
	// This helps operators understand why a pod has extra containers.
	if pod.Annotations == nil {
		patches = append(patches, jsonPatchOp{
			Op:   "add",
			Path: "/metadata/annotations",
			Value: map[string]string{
				"sidecar-injector.example.com/injected": "true",
			},
		})
	} else {
		patches = append(patches, jsonPatchOp{
			Op:    "add",
			Path:  "/metadata/annotations/sidecar-injector.example.com~1injected",
			Value: "true",
		})
	}

	patchBytes, err := json.Marshal(patches)
	if err != nil {
		log.Printf("ERROR: failed to marshal patch: %v", err)
		sendAdmissionResponse(w, req.UID, false, "failed to marshal patch", nil)
		return
	}

	log.Printf("INFO: injecting sidecar into pod %s/%s", pod.Namespace, pod.Name)
	sendAdmissionResponse(w, req.UID, true, "", patchBytes)
}

// sendAdmissionResponse constructs and sends an AdmissionReview response.
// If allowed is true and patchBytes is non-nil, the response includes
// a JSON patch. If allowed is false, the response includes a rejection
// message.
func sendAdmissionResponse(
	w http.ResponseWriter,
	uid types.UID,
	allowed bool,
	message string,
	patchBytes []byte,
) {
	// Build the response object that the API server expects.
	response := admissionv1.AdmissionReview{
		TypeMeta: metav1.TypeMeta{
			APIVersion: "admission.k8s.io/v1",
			Kind:       "AdmissionReview",
		},
		Response: &admissionv1.AdmissionResponse{
			UID:     uid,
			Allowed: allowed,
		},
	}

	// If we have a patch, attach it to the response.
	if patchBytes != nil {
		patchType := admissionv1.PatchTypeJSONPatch
		response.Response.PatchType = &patchType
		response.Response.Patch = patchBytes
	}

	// If the request was denied, include the reason.
	if !allowed {
		response.Response.Result = &metav1.Status{
			Message: message,
		}
	}

	// Serialise and send.
	respBytes, err := json.Marshal(response)
	if err != nil {
		log.Printf("ERROR: failed to marshal response: %v", err)
		http.Error(w, "internal error", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.Write(respBytes)
}

func main() {
	// ── Parse command-line flags ───────────────────────────────────
	var (
		certFile string
		keyFile  string
		port     int
	)
	flag.StringVar(&certFile, "tls-cert", "/etc/webhook/certs/tls.crt",
		"Path to the TLS certificate file")
	flag.StringVar(&keyFile, "tls-key", "/etc/webhook/certs/tls.key",
		"Path to the TLS private key file")
	flag.IntVar(&port, "port", 8443,
		"Port to listen on for HTTPS requests")
	flag.Parse()

	// ── Register HTTP handlers ─────────────────────────────────────
	mux := http.NewServeMux()
	mux.HandleFunc("/mutate", handleMutate)

	// Health check endpoint for Kubernetes readiness/liveness probes.
	mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		w.Write([]byte("ok"))
	})

	// ── Start the HTTPS server ─────────────────────────────────────
	addr := fmt.Sprintf(":%d", port)
	log.Printf("INFO: starting sidecar-injector webhook on %s", addr)

	server := &http.Server{
		Addr:    addr,
		Handler: mux,
	}

	if err := server.ListenAndServeTLS(certFile, keyFile); err != nil {
		log.Fatalf("FATAL: server failed: %v", err)
	}
}
```

> **Note:** The import for `types.UID` is `k8s.io/apimachinery/pkg/types`.
> We use the `admissionv1` package from `k8s.io/api/admission/v1`.

---

### Hands-On: Building a Validating Admission Webhook in Go

This webhook enforces that all Deployments must carry an `owner` label and a
`cost-centre` label.

```go
// File: cmd/label-validator/main.go
//
// label-validator is a Kubernetes validating admission webhook that
// rejects Deployment creation/update requests unless the Deployment
// carries the required labels: "owner" and "cost-centre".
//
// This ensures every workload in the cluster is attributable to a
// team and a cost centre for chargeback and incident response.
package main

import (
	"encoding/json"
	"flag"
	"fmt"
	"io"
	"log"
	"net/http"
	"strings"

	admissionv1 "k8s.io/api/admission/v1"
	appsv1 "k8s.io/api/apps/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
)

// requiredLabels defines the labels that must be present on every
// Deployment. If any of these labels are missing, the admission
// request will be rejected with a descriptive error message.
var requiredLabels = []string{"owner", "cost-centre"}

// handleValidate is the HTTP handler for the /validate endpoint.
// It decodes the incoming AdmissionReview, extracts the Deployment,
// checks for required labels, and returns an allow/deny decision.
func handleValidate(w http.ResponseWriter, r *http.Request) {
	// ── Read and decode the AdmissionReview ─────────────────────
	body, err := io.ReadAll(r.Body)
	if err != nil {
		log.Printf("ERROR: failed to read body: %v", err)
		http.Error(w, "failed to read body", http.StatusBadRequest)
		return
	}
	defer r.Body.Close()

	var review admissionv1.AdmissionReview
	if err := json.Unmarshal(body, &review); err != nil {
		log.Printf("ERROR: failed to unmarshal review: %v", err)
		http.Error(w, "bad request", http.StatusBadRequest)
		return
	}

	if review.Request == nil {
		http.Error(w, "missing request", http.StatusBadRequest)
		return
	}

	req := review.Request

	// ── Decode the Deployment from the raw object ──────────────
	var deployment appsv1.Deployment
	if err := json.Unmarshal(req.Object.Raw, &deployment); err != nil {
		log.Printf("ERROR: failed to unmarshal deployment: %v", err)
		sendResponse(w, req.UID, false,
			"internal error: could not decode Deployment")
		return
	}

	// ── Check for required labels ──────────────────────────────
	// Collect all missing labels so we can give the user a single,
	// comprehensive error message rather than making them fix one
	// label at a time.
	var missingLabels []string
	for _, label := range requiredLabels {
		if _, exists := deployment.Labels[label]; !exists {
			missingLabels = append(missingLabels, label)
		}
	}

	if len(missingLabels) > 0 {
		msg := fmt.Sprintf(
			"Deployment %q is missing required labels: [%s]. "+
				"All Deployments must have 'owner' and 'cost-centre' labels.",
			deployment.Name,
			strings.Join(missingLabels, ", "),
		)
		log.Printf("DENIED: %s", msg)
		sendResponse(w, req.UID, false, msg)
		return
	}

	// All required labels are present — allow the request.
	log.Printf("ALLOWED: Deployment %s/%s has all required labels",
		deployment.Namespace, deployment.Name)
	sendResponse(w, req.UID, true, "")
}

// sendResponse constructs and writes an AdmissionReview response.
// For validating webhooks, we never include patches — only the
// allow/deny decision and an optional human-readable message.
func sendResponse(w http.ResponseWriter, uid types.UID, allowed bool, message string) {
	review := admissionv1.AdmissionReview{
		TypeMeta: metav1.TypeMeta{
			APIVersion: "admission.k8s.io/v1",
			Kind:       "AdmissionReview",
		},
		Response: &admissionv1.AdmissionResponse{
			UID:     uid,
			Allowed: allowed,
		},
	}

	if !allowed {
		review.Response.Result = &metav1.Status{
			Message: message,
			Code:    403,
		}
	}

	data, err := json.Marshal(review)
	if err != nil {
		log.Printf("ERROR: failed to marshal response: %v", err)
		http.Error(w, "internal error", http.StatusInternalServerError)
		return
	}

	w.Header().Set("Content-Type", "application/json")
	w.Write(data)
}

func main() {
	var (
		certFile string
		keyFile  string
		port     int
	)
	flag.StringVar(&certFile, "tls-cert", "/etc/webhook/certs/tls.crt",
		"Path to TLS certificate")
	flag.StringVar(&keyFile, "tls-key", "/etc/webhook/certs/tls.key",
		"Path to TLS private key")
	flag.IntVar(&port, "port", 8443,
		"HTTPS listen port")
	flag.Parse()

	mux := http.NewServeMux()
	mux.HandleFunc("/validate", handleValidate)
	mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
	})

	addr := fmt.Sprintf(":%d", port)
	log.Printf("INFO: starting label-validator webhook on %s", addr)
	if err := (&http.Server{Addr: addr, Handler: mux}).ListenAndServeTLS(certFile, keyFile); err != nil {
		log.Fatalf("FATAL: %v", err)
	}
}
```

### Hands-On: Deploying Webhooks to Kubernetes with Proper TLS

Webhook TLS is the most common source of deployment failures. Here is a
complete setup using a self-signed CA:

```bash
#!/bin/bash
# generate-certs.sh — Generate a self-signed CA and webhook server
# certificate for the sidecar-injector webhook.
#
# The certificate's Subject Alternative Name (SAN) must match the
# service DNS name that the API server uses to call the webhook:
#   <service-name>.<namespace>.svc

set -euo pipefail

SERVICE_NAME="sidecar-injector"
NAMESPACE="admission-system"
SECRET_NAME="sidecar-injector-tls"
CERT_DIR="./certs"

mkdir -p "${CERT_DIR}"

# Step 1: Generate the CA private key and self-signed certificate.
openssl genrsa -out "${CERT_DIR}/ca.key" 2048
openssl req -new -x509 -days 3650 -key "${CERT_DIR}/ca.key" \
  -out "${CERT_DIR}/ca.crt" \
  -subj "/CN=Sidecar Injector CA"

# Step 2: Generate the webhook server private key.
openssl genrsa -out "${CERT_DIR}/tls.key" 2048

# Step 3: Create a CSR config with the correct SAN.
# The API server resolves the webhook service to:
#   <service>.<namespace>.svc
# We include multiple DNS SANs for flexibility.
cat > "${CERT_DIR}/csr.conf" <<EOF
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name
prompt = no

[req_distinguished_name]
CN = ${SERVICE_NAME}.${NAMESPACE}.svc

[v3_req]
basicConstraints = CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = ${SERVICE_NAME}
DNS.2 = ${SERVICE_NAME}.${NAMESPACE}
DNS.3 = ${SERVICE_NAME}.${NAMESPACE}.svc
DNS.4 = ${SERVICE_NAME}.${NAMESPACE}.svc.cluster.local
EOF

# Step 4: Generate the CSR and sign it with our CA.
openssl req -new -key "${CERT_DIR}/tls.key" \
  -out "${CERT_DIR}/tls.csr" \
  -config "${CERT_DIR}/csr.conf"

openssl x509 -req -in "${CERT_DIR}/tls.csr" \
  -CA "${CERT_DIR}/ca.crt" -CAkey "${CERT_DIR}/ca.key" \
  -CAcreateserial \
  -out "${CERT_DIR}/tls.crt" \
  -days 365 \
  -extensions v3_req \
  -extfile "${CERT_DIR}/csr.conf"

# Step 5: Create the Kubernetes namespace and TLS secret.
kubectl create namespace "${NAMESPACE}" --dry-run=client -o yaml | kubectl apply -f -
kubectl create secret tls "${SECRET_NAME}" \
  --cert="${CERT_DIR}/tls.crt" \
  --key="${CERT_DIR}/tls.key" \
  --namespace="${NAMESPACE}" \
  --dry-run=client -o yaml | kubectl apply -f -

# Step 6: Output the base64-encoded CA certificate for the webhook config.
echo ""
echo "=== CA Bundle (paste this into your webhook configuration) ==="
base64 < "${CERT_DIR}/ca.crt" | tr -d '\n'
echo ""
```

The Deployment and Service for the webhook:

```yaml
# webhook-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sidecar-injector
  namespace: admission-system
  labels:
    app: sidecar-injector
spec:
  replicas: 2  # Run multiple replicas for high availability
  selector:
    matchLabels:
      app: sidecar-injector
  template:
    metadata:
      labels:
        app: sidecar-injector
    spec:
      containers:
        - name: webhook
          image: myregistry.example.com/sidecar-injector:v1.0.0
          ports:
            - containerPort: 8443
              name: https
          # Mount the TLS certificate from the Kubernetes secret.
          volumeMounts:
            - name: tls-certs
              mountPath: /etc/webhook/certs
              readOnly: true
          # Resource limits for the webhook server itself.
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 128Mi
          # Readiness probe ensures the pod only receives traffic
          # when it's ready to serve webhook requests.
          readinessProbe:
            httpGet:
              path: /healthz
              port: 8443
              scheme: HTTPS
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8443
              scheme: HTTPS
            initialDelaySeconds: 10
            periodSeconds: 30
      volumes:
        - name: tls-certs
          secret:
            secretName: sidecar-injector-tls
---
apiVersion: v1
kind: Service
metadata:
  name: sidecar-injector
  namespace: admission-system
spec:
  selector:
    app: sidecar-injector
  ports:
    - port: 443
      targetPort: 8443
      protocol: TCP
```

---

## 12.6 ValidatingAdmissionPolicy (CEL-Based)

Starting with Kubernetes 1.26, you can define admission policies **without
writing a webhook**. ValidatingAdmissionPolicy uses the **Common Expression
Language (CEL)** to evaluate policies directly in the API server process.

### Why CEL?

- **No infrastructure** — no webhook servers to deploy, scale, or secure
- **No latency** — evaluation happens in-process, no network round-trip
- **Deterministic** — CEL is a pure, sandboxed expression language
- **Auditable** — policies are Kubernetes resources, version-controlled

### How It Works

You create two resources:

1. **ValidatingAdmissionPolicy** — defines the validation rules
2. **ValidatingAdmissionPolicyBinding** — binds the policy to specific
   resources or namespaces

```
Policy                              Binding
┌──────────────────────────┐       ┌────────────────────────────┐
│ matchConstraints:        │       │ policyName: require-labels │
│   resourceRules:         │       │ validationActions:         │
│     - resources: [pods]  │       │   - Deny                   │
│ validations:             │◀──────│ matchResources:            │
│   - expression: "..."    │       │   namespaceSelector: ...   │
│     message: "..."       │       └────────────────────────────┘
└──────────────────────────┘
```

### Hands-On: Writing CEL-Based Admission Policies

**Example 1: Require labels on all Deployments**

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-deployment-labels
spec:
  # failurePolicy controls what happens if CEL evaluation itself fails
  # (e.g., due to a bug in the expression). Use Fail in production.
  failurePolicy: Fail
  # matchConstraints defines which requests this policy applies to.
  matchConstraints:
    resourceRules:
      - apiGroups: ["apps"]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["deployments"]
  # validations is a list of CEL expressions. ALL must evaluate to
  # true for the request to be allowed.
  validations:
    - expression: "has(object.metadata.labels) && 'owner' in object.metadata.labels"
      messageExpression: "'Deployment ' + object.metadata.name + ' must have an owner label'"
    - expression: "has(object.metadata.labels) && 'cost-centre' in object.metadata.labels"
      message: "All Deployments must have a cost-centre label"
---
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: require-deployment-labels-binding
spec:
  policyName: require-deployment-labels
  # validationActions determines the enforcement behaviour:
  #   Deny   — reject the request
  #   Warn   — allow but add a warning header
  #   Audit  — allow but log to the audit log
  validationActions:
    - Deny
  matchResources:
    namespaceSelector:
      matchExpressions:
        - key: kubernetes.io/metadata.name
          operator: NotIn
          values: ["kube-system", "kube-public"]
```

**Example 2: Restrict container images to an approved registry**

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: restrict-image-registries
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  validations:
    # CEL can iterate over arrays using the .all() macro.
    # This expression checks that EVERY container image starts
    # with the approved registry prefix.
    - expression: >
        object.spec.containers.all(c,
          c.image.startsWith('registry.example.com/')
        )
      message: "All container images must come from registry.example.com"
    # Also check init containers if present.
    - expression: >
        !has(object.spec.initContainers) ||
        object.spec.initContainers.all(c,
          c.image.startsWith('registry.example.com/')
        )
      message: "All init container images must come from registry.example.com"
```

**Example 3: Enforce resource limits**

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: require-resource-limits
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["pods"]
  validations:
    # Every container must have memory limits set.
    - expression: >
        object.spec.containers.all(c,
          has(c.resources) &&
          has(c.resources.limits) &&
          'memory' in c.resources.limits
        )
      message: "All containers must specify memory limits"
```

### Testing CEL Policies

```bash
# Try creating a Deployment without labels — should be rejected
kubectl create deployment bad-deploy --image=nginx
# Error: Deployment bad-deploy must have an owner label

# Create with required labels — should succeed
kubectl create deployment good-deploy --image=nginx \
  --dry-run=client -o yaml | \
  kubectl label --local -f - owner=team-alpha cost-centre=cc-1234 -o yaml | \
  kubectl apply -f -
```

---

## 12.7 API Groups & Versioning

### API Group Structure

Kubernetes organises its API into groups. Each group can have multiple
versions. The URL structure reflects this:

```
Core group (legacy):
  /api/v1/namespaces/default/pods

Named groups:
  /apis/apps/v1/namespaces/default/deployments
  /apis/batch/v1/namespaces/default/jobs
  /apis/networking.k8s.io/v1/namespaces/default/ingresses

  ┌─── API prefix
  │    ┌─── group name
  │    │          ┌─── version
  │    │          │
  /apis/apps     /v1/namespaces/default/deployments
```

The **core** group (pods, services, configmaps, secrets, nodes) lives under
`/api/v1` — not `/apis/core/v1`. This is a historical artefact from before
API groups existed.

### Version Lifecycle

```
alpha (v1alpha1)  ──▶  beta (v1beta1)  ──▶  stable (v1)
     │                      │                    │
     │ - Off by default     │ - On by default    │ - Always on
     │ - May change/break   │ - Well tested      │ - Backward compatible
     │ - May be removed     │ - Migration path   │ - Long-term support
     │ - No stability       │   guaranteed        │
     │   guarantees         │                    │
```

### API Deprecation Policy

Kubernetes follows a strict deprecation policy:

- **GA APIs** — must be available for at least **12 months** or **3 releases**
  after deprecation (whichever is longer)
- **Beta APIs** — available for at least **9 months** or **3 releases**
- **Alpha APIs** — can be removed in any release without notice

### How Version Conversion Works Internally

When you submit a v1beta1 object but the API server stores v1, it must
convert between versions. This happens through an **internal version**:

```
v1beta1  ──▶  internal  ──▶  v1 (stored)
              version
                │
                │  (hub-and-spoke conversion)
                │
v1alpha1 ──▶  internal  ──▶  v1
```

Each API group has:
1. An **internal type** (the "hub")
2. **Conversion functions** from each external version to/from the internal type
3. **Default functions** that set missing fields during conversion

This is why you can submit objects in any supported version — the API server
converts them seamlessly.

### Hands-On: Exploring API Groups

```bash
# List all API groups and their preferred versions
kubectl api-versions | sort

# Output (abbreviated):
# admissionregistration.k8s.io/v1
# apiextensions.k8s.io/v1
# apps/v1
# authentication.k8s.io/v1
# authorization.k8s.io/v1
# batch/v1
# certificates.k8s.io/v1
# coordination.k8s.io/v1
# networking.k8s.io/v1
# rbac.authorization.k8s.io/v1
# storage.k8s.io/v1
# v1   ← this is the core group

# List all resource types with their API groups
kubectl api-resources -o wide | head -20

# Discover a specific resource
kubectl api-resources | grep -i deploy
# deployments    deploy   apps/v1   true   Deployment

# Explore the raw API
kubectl get --raw /apis | python -m json.tool | head -30
kubectl get --raw /apis/apps/v1 | python -m json.tool

# See which resources support which verbs
kubectl api-resources -o wide | grep pods
# pods  po  v1  true  Pod  [create delete deletecollection get list patch update watch]
```

---

## 12.8 API Aggregation Layer

### Extending the API Server

The aggregation layer lets you run **custom API servers** that appear as
native Kubernetes APIs. The main API server proxies requests to your custom
server.

```
Client ──▶ kube-apiserver ──▶ your-custom-apiserver
                │
                ├─ /api/v1/pods               → handled by kube-apiserver
                ├─ /apis/apps/v1/deployments   → handled by kube-apiserver
                └─ /apis/metrics.k8s.io/v1beta1/pods → proxied to metrics-server
```

The **metrics-server** is the most common example. When you run
`kubectl top pods`, the request goes to the aggregated
`metrics.k8s.io/v1beta1` API.

### APIService Resource

You register your custom API server by creating an `APIService` resource:

```yaml
apiVersion: apiregistration.k8s.io/v1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
spec:
  # The API group and version your server handles.
  group: metrics.k8s.io
  version: v1beta1
  # Where to route requests — a Service in the cluster.
  service:
    name: metrics-server
    namespace: kube-system
    port: 443
  # Priority controls which server "wins" when multiple servers
  # claim the same group. Lower number = higher priority.
  groupPriorityMinimum: 100
  versionPriority: 100
  # If true, the kube-apiserver does not verify the serving cert.
  insecureSkipTLSVerify: false
  # CA bundle to verify the custom server's TLS certificate.
  caBundle: <base64-ca-cert>
```

### When to Use Aggregation vs CRDs

```
┌─────────────────────┬──────────────────────────┬───────────────────────┐
│ Feature             │ CRDs                     │ API Aggregation       │
├─────────────────────┼──────────────────────────┼───────────────────────┤
│ Complexity          │ Low — just YAML          │ High — write a server │
│ Custom storage      │ No — always etcd         │ Yes — any backend     │
│ Custom validation   │ CEL or webhook           │ Full Go code          │
│ Sub-resources       │ /status, /scale only     │ Any sub-resource      │
│ Protobuf support    │ No                       │ Yes                   │
│ Custom verbs        │ No                       │ Yes                   │
│ Versioning          │ Basic conversion webhook │ Full version control  │
│ Example             │ CertManager Certificate  │ metrics-server        │
└─────────────────────┴──────────────────────────┴───────────────────────┘
```

**Rule of thumb:** Use CRDs unless you need custom storage, custom
sub-resources, protobuf encoding, or complex version conversion. CRDs cover
90%+ of use cases.

---

## 12.9 API Priority and Fairness (APF)

### The Problem: API Server Overload

Without flow control, a misbehaving controller running tight loops could
overwhelm the API server with requests, starving other components. The
scheduler can't schedule pods if it can't reach the API server.

### APF: The Solution

API Priority and Fairness (stable since 1.29) implements a queuing and
fairness system for API server requests. It replaces the old
`--max-requests-inflight` flag with a much more sophisticated approach.

```
Incoming Request
      │
      ▼
┌──────────────┐
│ FlowSchema   │──── matches request to a priority level
│ matching     │     based on user, group, verb, resource
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ PriorityLevel│──── determines the queue and concurrency
│ assignment   │     share for this request
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Fair Queuing │──── requests are queued and dispatched
│ (shuffle     │     fairly within each priority level
│  sharding)   │
└──────────────┘
```

### Key Concepts

**PriorityLevelConfiguration:**
Defines a priority level with a concurrency share. Higher priority levels
get more of the API server's capacity.

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: PriorityLevelConfiguration
metadata:
  name: custom-workload
spec:
  type: Limited
  limited:
    # nominalConcurrencyShares determines this level's share of
    # the API server's total concurrency limit.
    nominalConcurrencyShares: 30
    # lendablePercent allows unused concurrency to be borrowed
    # by other priority levels.
    lendablePercent: 50
    limitResponse:
      type: Queue
      queuing:
        queues: 64
        handSize: 6
        queueLengthLimit: 50
```

**FlowSchema:**
Maps requests to priority levels based on attributes:

```yaml
apiVersion: flowcontrol.apiserver.k8s.io/v1
kind: FlowSchema
metadata:
  name: custom-controller-flow
spec:
  # Higher matchingPrecedence = lower priority (checked later).
  matchingPrecedence: 1000
  priorityLevelConfiguration:
    name: custom-workload
  # distinguisherMethod determines how requests are distributed
  # across queues within the priority level.
  distinguisherMethod:
    type: ByUser
  rules:
    - subjects:
        - kind: ServiceAccount
          serviceAccount:
            name: my-controller
            namespace: my-system
      resourceRules:
        - apiGroups: [""]
          resources: ["pods"]
          verbs: ["list", "watch"]
          namespaces: ["*"]
```

### Built-In Priority Levels

```
┌──────────────────────┬──────────────────────────────────────────────┐
│ Priority Level       │ Purpose                                       │
├──────────────────────┼──────────────────────────────────────────────┤
│ exempt               │ system:masters — never queued                 │
│ system               │ system:nodes, kube-system SAs                 │
│ leader-election      │ leader election requests                      │
│ workload-high        │ built-in controllers                          │
│ workload-low         │ other SAs in kube-system                      │
│ global-default       │ everything else                               │
│ catch-all            │ catch-all for unmatched requests               │
└──────────────────────┴──────────────────────────────────────────────┘
```

### Hands-On: Examining APF Configuration

```bash
# List all priority levels
kubectl get prioritylevelconfigurations

# List all flow schemas
kubectl get flowschemas

# See which flow schema matches your requests
kubectl get flowschemas -o custom-columns=\
NAME:.metadata.name,\
PRIORITY:.spec.priorityLevelConfiguration.name,\
MATCH:.spec.matchingPrecedence

# Check APF metrics (if you have access to the API server's metrics)
# These metrics are invaluable for debugging API server throttling.
kubectl get --raw /metrics | grep apiserver_flowcontrol

# Key metrics to watch:
#   apiserver_flowcontrol_dispatched_requests_total
#   apiserver_flowcontrol_rejected_requests_total
#   apiserver_flowcontrol_current_inqueue_requests
#   apiserver_flowcontrol_request_wait_duration_seconds

# Debug: see the APF debug endpoint (requires exec into API server)
kubectl get --raw /debug/api_priority_and_fairness/dump_priority_levels
kubectl get --raw /debug/api_priority_and_fairness/dump_queues
```

---

## 12.10 Server-Side Apply

### The Problem with Client-Side Apply

Before server-side apply, `kubectl apply` tracked field ownership using the
`kubectl.kubernetes.io/last-applied-configuration` annotation. This was:

1. **Client-specific** — only kubectl understood the annotation
2. **Lossy** — the annotation stored the full last-applied config, making it
   hard to detect conflicts between multiple managers
3. **Fragile** — other tools (Helm, Terraform, custom controllers) didn't
   participate in conflict detection

### Server-Side Apply (SSA)

Server-side apply moves field management into the API server. Each field is
tracked with its **manager** — the identity of the client that last set it.

```
metadata:
  managedFields:
    - manager: kubectl
      operation: Apply
      fieldsV1:
        f:metadata:
          f:labels:
            f:app: {}
        f:spec:
          f:replicas: {}
    - manager: hpa-controller
      operation: Update
      fieldsV1:
        f:spec:
          f:replicas: {}    ← conflict! Two managers own this field.
```

### Conflict Detection

When two managers try to set the same field, the API server returns a conflict
error (HTTP 409). The client must either:

1. **Force** — take ownership by setting `force=true`
2. **Yield** — remove the field from their apply configuration

### Hands-On: Using Server-Side Apply from Go

```go
// File: cmd/ssa-demo/main.go
//
// ssa-demo demonstrates Server-Side Apply (SSA) using the client-go
// library. It creates a Deployment using SSA, then updates it to show
// how field management and conflict detection work.
//
// Server-Side Apply is the recommended way to manage Kubernetes
// resources programmatically because it provides proper field
// ownership tracking and conflict detection between multiple managers.
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"path/filepath"

	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"

	appsv1apply "k8s.io/client-go/applyconfigurations/apps/v1"
	corev1apply "k8s.io/client-go/applyconfigurations/core/v1"
	metav1apply "k8s.io/client-go/applyconfigurations/meta/v1"
)

func main() {
	// ── Build the Kubernetes client from kubeconfig ─────────────
	kubeconfig := filepath.Join(homedir.HomeDir(), ".kube", "config")
	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		log.Fatalf("failed to build config: %v", err)
	}

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		log.Fatalf("failed to create clientset: %v", err)
	}

	ctx := context.Background()

	// ── Define the Deployment using apply configurations ────────
	// The applyconfigurations package provides typed builders for
	// Server-Side Apply. Unlike the regular types, these use
	// pointer fields so the API server can distinguish "field not
	// specified" from "field set to zero value".
	deployApplyConfig := appsv1apply.Deployment("nginx-ssa-demo", "default").
		WithLabels(map[string]string{
			"app":   "nginx",
			"owner": "ssa-demo",
		}).
		WithSpec(appsv1apply.DeploymentSpec().
			WithReplicas(3).
			WithSelector(metav1apply.LabelSelector().
				WithMatchLabels(map[string]string{"app": "nginx"}),
			).
			WithTemplate(corev1apply.PodTemplateSpec().
				WithLabels(map[string]string{"app": "nginx"}).
				WithSpec(corev1apply.PodSpec().
					WithContainers(
						corev1apply.Container().
							WithName("nginx").
							WithImage("nginx:1.25").
							WithPorts(
								corev1apply.ContainerPort().
									WithContainerPort(80).
									WithProtocol(corev1.ProtocolTCP),
							),
					),
				),
			),
		)

	// ── Apply the Deployment using Server-Side Apply ────────────
	// The fieldManager identifies this client. The API server uses
	// it to track which fields this manager owns. Choose a unique
	// name — typically your controller or tool name.
	result, err := clientset.AppsV1().Deployments("default").Apply(
		ctx,
		deployApplyConfig,
		metav1.ApplyOptions{
			// FieldManager identifies who is managing these fields.
			// When another manager tries to change a field we own,
			// the API server will return a conflict.
			FieldManager: "ssa-demo-controller",

			// Force: if true, takes ownership of conflicting fields.
			// Use with caution — this overrides other managers.
			Force: true,
		},
	)
	if err != nil {
		log.Fatalf("failed to apply deployment: %v", err)
	}

	fmt.Printf("Applied Deployment %s/%s (resourceVersion: %s)\n",
		result.Namespace, result.Name, result.ResourceVersion)

	// ── Inspect managed fields ──────────────────────────────────
	// After applying, we can see the field management metadata that
	// the API server tracks. This shows which manager owns which
	// fields.
	fmt.Println("\nManaged Fields:")
	for _, mf := range result.ManagedFields {
		fieldsJSON, _ := json.MarshalIndent(mf.FieldsV1, "  ", "  ")
		fmt.Printf("  Manager: %s, Operation: %s\n", mf.Manager, mf.Operation)
		fmt.Printf("  Fields: %s\n\n", string(fieldsJSON))
	}

	// ── Update: change the image using SSA ──────────────────────
	// This time we only specify the fields we want to change.
	// Server-Side Apply will merge this with the existing object,
	// only taking ownership of the fields we specify.
	updateConfig := appsv1apply.Deployment("nginx-ssa-demo", "default").
		WithSpec(appsv1apply.DeploymentSpec().
			WithTemplate(corev1apply.PodTemplateSpec().
				WithSpec(corev1apply.PodSpec().
					WithContainers(
						corev1apply.Container().
							WithName("nginx").
							WithImage("nginx:1.26"), // updated image
					),
				),
			),
		)

	updated, err := clientset.AppsV1().Deployments("default").Apply(
		ctx,
		updateConfig,
		metav1.ApplyOptions{
			FieldManager: "ssa-demo-controller",
			Force:        true,
		},
	)
	if err != nil {
		log.Fatalf("failed to update deployment: %v", err)
	}

	fmt.Printf("Updated Deployment image to nginx:1.26 (rv: %s)\n",
		updated.ResourceVersion)

	// ── Patch with a different manager ──────────────────────────
	// Demonstrate what happens when a different manager tries to
	// modify a field we own. We'll use a raw JSON patch with a
	// different field manager to simulate an HPA scaling the replicas.
	patchData := []byte(`{"spec":{"replicas":5}}`)
	patched, err := clientset.AppsV1().Deployments("default").Patch(
		ctx,
		"nginx-ssa-demo",
		types.MergePatchType,
		patchData,
		metav1.PatchOptions{
			FieldManager: "hpa-controller",
		},
	)
	if err != nil {
		log.Fatalf("failed to patch: %v", err)
	}

	fmt.Printf("HPA scaled to %d replicas (rv: %s)\n",
		*patched.Spec.Replicas, patched.ResourceVersion)

	// Now check managed fields — both managers should appear.
	fmt.Println("\nAfter HPA patch — Managed Fields:")
	for _, mf := range patched.ManagedFields {
		fmt.Printf("  Manager: %s, Operation: %s, Time: %s\n",
			mf.Manager, mf.Operation, mf.Time)
	}
}
```

---

## 12.11 API Machinery in Go

The `k8s.io/apimachinery` module is the foundation library that every
Kubernetes Go project depends on. It defines the core concepts: how objects
are represented, how they are serialised, and how types are registered.

### The runtime.Object Interface

Every Kubernetes type implements `runtime.Object`:

```go
// runtime.Object is the interface that all Kubernetes API objects
// must implement. It provides two capabilities:
//  1. GetObjectKind() — returns the GVK (Group/Version/Kind) of the object
//  2. DeepCopyObject() — returns a deep copy of the object
//
// This is how the API machinery can handle any Kubernetes object
// generically, without knowing its concrete type at compile time.
type Object interface {
	GetObjectKind() schema.ObjectKind
	DeepCopyObject() Object
}
```

Any struct that embeds `metav1.TypeMeta` and `metav1.ObjectMeta` and
implements `DeepCopyObject()` satisfies this interface.

### GVK and GVR

**GVK (Group, Version, Kind)** — identifies a type:
```
Group: apps    Version: v1    Kind: Deployment
Group: ""      Version: v1    Kind: Pod          (core group = "")
```

**GVR (Group, Version, Resource)** — identifies a REST endpoint:
```
Group: apps    Version: v1    Resource: deployments
Group: ""      Version: v1    Resource: pods
```

The mapping between GVK and GVR is handled by the **RESTMapper**.

```go
import (
	"k8s.io/apimachinery/pkg/runtime/schema"
)

// GVK — identifies a type in the Kubernetes type system.
// Used for serialisation/deserialisation and scheme lookups.
gvk := schema.GroupVersionKind{
	Group:   "apps",
	Version: "v1",
	Kind:    "Deployment",
}

// GVR — identifies a REST endpoint.
// Used for dynamic client operations and discovery.
gvr := schema.GroupVersionResource{
	Group:    "apps",
	Version:  "v1",
	Resource: "deployments",
}
```

### Scheme — The Type Registry

The Scheme is the central registry that maps between Go types and GVKs:

```go
// File: pkg/scheme-demo/main.go
//
// scheme-demo demonstrates how the Kubernetes Scheme works. The Scheme
// is the type registry that the API machinery uses to:
//  1. Map Go types to GVKs (and vice versa)
//  2. Create new instances of types by GVK
//  3. Convert between versions of the same type
//  4. Apply default values to objects
//
// Every Kubernetes client library and controller depends on a Scheme
// that knows about the types it works with.
package main

import (
	"fmt"
	"log"

	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	clientgoscheme "k8s.io/client-go/kubernetes/scheme"
)

func main() {
	// ── Create a new Scheme and register types ──────────────────
	// A Scheme starts empty. We register type groups by calling
	// their AddToScheme functions. The client-go scheme includes
	// all built-in Kubernetes types.
	scheme := runtime.NewScheme()

	// Register all built-in Kubernetes types (pods, deployments, etc.)
	// utilruntime.Must panics on error — appropriate during init.
	utilruntime.Must(clientgoscheme.AddToScheme(scheme))

	// ── Look up a GVK from a Go type ───────────────────────────
	// Given a Go struct, find its GVK(s) in the scheme.
	gvks, _, err := scheme.ObjectKinds(&appsv1.Deployment{})
	if err != nil {
		log.Fatalf("failed to get GVKs: %v", err)
	}
	fmt.Printf("Deployment GVKs: %v\n", gvks)
	// Output: [{apps v1 Deployment}]

	// ── Create an object by GVK ────────────────────────────────
	// Given a GVK, the Scheme can instantiate the correct Go struct.
	// This is how the API server deserialises objects from JSON/YAML
	// — it reads the apiVersion and kind, looks up the Go type in
	// the Scheme, and unmarshals into that type.
	gvk := schema.GroupVersionKind{
		Group:   "",
		Version: "v1",
		Kind:    "Pod",
	}
	obj, err := scheme.New(gvk)
	if err != nil {
		log.Fatalf("failed to create object: %v", err)
	}
	pod, ok := obj.(*corev1.Pod)
	if !ok {
		log.Fatal("expected *corev1.Pod")
	}
	fmt.Printf("Created Pod: %T\n", pod)
	// Output: Created Pod: *v1.Pod

	// ── Check if a type is registered ──────────────────────────
	registered := scheme.Recognizes(schema.GroupVersionKind{
		Group:   "apps",
		Version: "v1",
		Kind:    "Deployment",
	})
	fmt.Printf("Deployment registered: %v\n", registered)
	// Output: true

	unregistered := scheme.Recognizes(schema.GroupVersionKind{
		Group:   "custom.example.com",
		Version: "v1",
		Kind:    "Widget",
	})
	fmt.Printf("Widget registered: %v\n", unregistered)
	// Output: false

	// ── List all registered types ──────────────────────────────
	allTypes := scheme.AllKnownTypes()
	fmt.Printf("\nTotal registered types: %d\n", len(allTypes))
	fmt.Println("First 10 types:")
	count := 0
	for gvk := range allTypes {
		if count >= 10 {
			break
		}
		fmt.Printf("  %s\n", gvk)
		count++
	}
}
```

### Hands-On: Working with API Machinery Types in Go

```go
// File: pkg/apimachinery-demo/main.go
//
// apimachinery-demo shows common patterns for working with
// Kubernetes objects using the apimachinery library:
//   - Constructing objects programmatically
//   - Working with unstructured objects (dynamic typing)
//   - Using labels and selectors
//   - Parsing and comparing resource quantities
package main

import (
	"encoding/json"
	"fmt"
	"log"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/resource"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/apimachinery/pkg/runtime/schema"
)

func main() {
	// ── Constructing a typed object ────────────────────────────
	// Using the strongly-typed API types from k8s.io/api.
	pod := &corev1.Pod{
		TypeMeta: metav1.TypeMeta{
			APIVersion: "v1",
			Kind:       "Pod",
		},
		ObjectMeta: metav1.ObjectMeta{
			Name:      "demo-pod",
			Namespace: "default",
			Labels: map[string]string{
				"app":     "demo",
				"version": "v1",
				"env":     "production",
			},
		},
		Spec: corev1.PodSpec{
			Containers: []corev1.Container{
				{
					Name:  "app",
					Image: "myapp:v1.0",
					Resources: corev1.ResourceRequirements{
						Requests: corev1.ResourceList{
							corev1.ResourceCPU:    resource.MustParse("100m"),
							corev1.ResourceMemory: resource.MustParse("128Mi"),
						},
						Limits: corev1.ResourceList{
							corev1.ResourceCPU:    resource.MustParse("500m"),
							corev1.ResourceMemory: resource.MustParse("256Mi"),
						},
					},
				},
			},
		},
	}

	// Verify it implements runtime.Object.
	fmt.Printf("Pod GVK: %v\n", pod.GetObjectKind().GroupVersionKind())

	// ── Working with Unstructured objects ───────────────────────
	// Unstructured objects are the "any" type of Kubernetes. They
	// represent objects as map[string]interface{}, useful when you
	// don't know the type at compile time (e.g., CRDs).
	u := &unstructured.Unstructured{
		Object: map[string]interface{}{
			"apiVersion": "example.com/v1",
			"kind":       "Widget",
			"metadata": map[string]interface{}{
				"name":      "my-widget",
				"namespace": "default",
			},
			"spec": map[string]interface{}{
				"color":    "blue",
				"replicas": int64(3),
			},
		},
	}

	// Access fields using helper methods.
	fmt.Printf("Unstructured name: %s\n", u.GetName())
	fmt.Printf("Unstructured GVK: %v\n", u.GroupVersionKind())

	// Access nested fields using the nested helpers.
	color, found, err := unstructured.NestedString(u.Object, "spec", "color")
	if err == nil && found {
		fmt.Printf("Widget color: %s\n", color)
	}

	// ── Label Selectors ────────────────────────────────────────
	// Label selectors are used throughout Kubernetes — Services
	// select Pods, Deployments select ReplicaSets, etc.

	// Parse a label selector string.
	selector, err := labels.Parse("app=demo,version in (v1,v2),env!=staging")
	if err != nil {
		log.Fatalf("failed to parse selector: %v", err)
	}

	// Check if our pod matches the selector.
	podLabels := labels.Set(pod.Labels)
	matches := selector.Matches(podLabels)
	fmt.Printf("Pod matches selector: %v\n", matches)
	// Output: true

	// ── Resource Quantities ────────────────────────────────────
	// Resource quantities handle CPU (millicores) and memory (bytes)
	// with proper unit parsing and arithmetic.
	cpu := resource.MustParse("500m")   // 500 millicores = 0.5 CPU
	memory := resource.MustParse("1Gi") // 1 gibibyte

	fmt.Printf("CPU: %s (millivalue: %d)\n", cpu.String(), cpu.MilliValue())
	fmt.Printf("Memory: %s (bytes: %d)\n", memory.String(), memory.Value())

	// Arithmetic with quantities.
	totalCPU := cpu.DeepCopy()
	additionalCPU := resource.MustParse("250m")
	totalCPU.Add(additionalCPU)
	fmt.Printf("Total CPU: %s\n", totalCPU.String()) // 750m

	// Comparison.
	limit := resource.MustParse("1") // 1 CPU = 1000m
	if totalCPU.Cmp(limit) < 0 {
		fmt.Printf("%s is under the %s limit\n", totalCPU.String(), limit.String())
	}

	// ── GVK/GVR utilities ──────────────────────────────────────
	gvr := schema.GroupVersionResource{
		Group:    "apps",
		Version:  "v1",
		Resource: "deployments",
	}
	fmt.Printf("GVR: %s\n", gvr.String()) // apps/v1, Resource=deployments

	// Parse a GVR from a string representation.
	gv := schema.GroupVersion{Group: "batch", Version: "v1"}
	fmt.Printf("GroupVersion: %s\n", gv.String()) // batch/v1

	// Serialise the pod to JSON.
	podJSON, _ := json.MarshalIndent(pod, "", "  ")
	fmt.Printf("\nPod JSON:\n%s\n", string(podJSON))
}
```

---

## 12.12 Watching Resources Efficiently

Controllers need to react to changes in cluster state. Polling is wasteful.
Kubernetes provides a **watch** mechanism that streams change events over
a long-lived HTTP connection.

### Watch Semantics

When you watch a resource, the API server sends events:

```
Type: ADDED    — a new object was created
Type: MODIFIED — an existing object was updated
Type: DELETED  — an object was removed
Type: BOOKMARK — a checkpoint (no object change)
Type: ERROR    — something went wrong
```

Each event includes the **resourceVersion** — a monotonically increasing
token that represents the state of etcd at the time of the change. Watches
use resourceVersion for:

1. **Resumption** — if the connection drops, you reconnect with the last
   seen resourceVersion to avoid missing events
2. **Consistency** — you know exactly which version of the object you're
   looking at

### Watch Cache

The API server maintains an in-memory **watch cache** so it doesn't need to
read from etcd for every watch. The cache stores recent events and serves
them to watchers. This is why watches are so efficient — the API server
handles thousands of watchers without proportional etcd load.

```
etcd ──▶ Watch Cache ──▶ Watcher 1 (kubelet)
                    ├──▶ Watcher 2 (scheduler)
                    ├──▶ Watcher 3 (controller-manager)
                    └──▶ Watcher 4 (your controller)
```

### The Reflector → Informer → Lister Pattern

Raw watches are low-level and error-prone. The client-go library provides
a layered abstraction:

```
┌───────────────────────────────────────────────────────────────┐
│                        Your Controller                        │
│                                                               │
│  1. Register event handlers (AddFunc, UpdateFunc, DeleteFunc) │
│  2. Read from the Lister (in-memory, no API calls)            │
│                                                               │
├───────────────────────────────────────────────────────────────┤
│                         Informer                              │
│                                                               │
│  - Manages the Reflector                                      │
│  - Maintains the in-memory Store (cache)                      │
│  - Dispatches events to registered handlers                   │
│  - Provides the Lister interface for read access              │
│                                                               │
├───────────────────────────────────────────────────────────────┤
│                         Reflector                             │
│                                                               │
│  - Performs List (initial sync) then Watch (incremental)      │
│  - Handles reconnection and resourceVersion bookkeeping       │
│  - Feeds events into the Informer's Store                     │
│                                                               │
├───────────────────────────────────────────────────────────────┤
│                      API Server Watch                         │
└───────────────────────────────────────────────────────────────┘
```

**Reflector:**
- Performs an initial `List` to get the current state of all objects
- Then opens a `Watch` to receive incremental updates
- If the watch connection drops, it reconnects using the last resourceVersion
- Handles the `410 Gone` error (resourceVersion too old) by re-listing

**Informer:**
- Wraps the Reflector
- Maintains an in-memory cache (Store) of all objects
- Dispatches events (add/update/delete) to registered handlers
- Never makes redundant API calls — reads come from the cache

**Lister:**
- A read-only interface to the Informer's cache
- `List()` and `Get()` read from memory — not from the API server

### Hands-On: Building an Efficient Resource Watcher in Go

```go
// File: cmd/pod-watcher/main.go
//
// pod-watcher demonstrates the Informer pattern for efficiently watching
// Kubernetes resources. Instead of polling or using raw watches, we use
// the SharedInformerFactory from client-go, which:
//
//  1. Performs an initial List to populate an in-memory cache
//  2. Opens a Watch to receive incremental updates
//  3. Dispatches events to our registered handlers
//  4. Provides a Lister for fast, in-memory reads
//
// This is the same pattern used by every built-in Kubernetes controller.
// It is the recommended way to watch resources in production code.
package main

import (
	"context"
	"fmt"
	"log"
	"os"
	"os/signal"
	"path/filepath"
	"syscall"
	"time"

	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
)

func main() {
	// ── Build the Kubernetes client ────────────────────────────
	kubeconfig := filepath.Join(homedir.HomeDir(), ".kube", "config")
	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		log.Fatalf("failed to build kubeconfig: %v", err)
	}

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		log.Fatalf("failed to create clientset: %v", err)
	}

	// ── Create a SharedInformerFactory ─────────────────────────
	// The factory creates informers for different resource types.
	// It ensures that multiple controllers watching the same resource
	// type share a single underlying watch connection (hence "Shared").
	//
	// The resync period (30s here) controls how often the informer
	// re-dispatches the full list of objects to your event handlers.
	// This is a safety net — if your handler missed an event, the
	// periodic resync will fire an Update event for every object.
	factory := informers.NewSharedInformerFactoryWithOptions(
		clientset,
		30*time.Second,
		// Optionally filter by namespace:
		// informers.WithNamespace("default"),
		//
		// Optionally filter by labels:
		// informers.WithTweakListOptions(func(opts *metav1.ListOptions) {
		//     opts.LabelSelector = "app=myapp"
		// }),
	)

	// ── Get the Pod informer ───────────────────────────────────
	// This creates (or retrieves if already created) a shared
	// informer for Pods. The informer starts with a List call
	// to populate its cache, then opens a Watch.
	podInformer := factory.Core().V1().Pods()

	// ── Register event handlers ────────────────────────────────
	// Event handlers are called by the informer when objects are
	// added, updated, or deleted. These run in the informer's
	// goroutine, so keep them fast. For heavy work, enqueue
	// items to a workqueue (covered in the Controllers chapter).
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		// AddFunc is called when a new Pod appears in the cache.
		// This fires for every existing pod during initial sync,
		// and for genuinely new pods after that.
		AddFunc: func(obj interface{}) {
			pod, ok := obj.(*corev1.Pod)
			if !ok {
				return
			}
			log.Printf("POD ADDED: %s/%s (phase: %s)",
				pod.Namespace, pod.Name, pod.Status.Phase)
		},

		// UpdateFunc is called when an existing Pod changes.
		// Both the old and new versions of the object are provided.
		// Compare them to determine what actually changed.
		UpdateFunc: func(oldObj, newObj interface{}) {
			oldPod, ok1 := oldObj.(*corev1.Pod)
			newPod, ok2 := newObj.(*corev1.Pod)
			if !ok1 || !ok2 {
				return
			}

			// Only log if the phase changed — pods are updated
			// frequently (status, conditions, etc.) and logging
			// every update would be extremely noisy.
			if oldPod.Status.Phase != newPod.Status.Phase {
				log.Printf("POD PHASE CHANGED: %s/%s: %s → %s",
					newPod.Namespace, newPod.Name,
					oldPod.Status.Phase, newPod.Status.Phase)
			}
		},

		// DeleteFunc is called when a Pod is removed from the cache.
		// The object may be a *corev1.Pod or a cache.DeletedFinalStateUnknown
		// wrapper (when the delete event was missed and discovered during resync).
		DeleteFunc: func(obj interface{}) {
			// Handle the DeletedFinalStateUnknown wrapper.
			// This occurs when the informer missed the delete event
			// and discovered the deletion during a re-list.
			pod, ok := obj.(*corev1.Pod)
			if !ok {
				tombstone, ok := obj.(cache.DeletedFinalStateUnknown)
				if !ok {
					log.Printf("ERROR: unexpected object type: %T", obj)
					return
				}
				pod, ok = tombstone.Obj.(*corev1.Pod)
				if !ok {
					log.Printf("ERROR: unexpected tombstone object type: %T",
						tombstone.Obj)
					return
				}
			}
			log.Printf("POD DELETED: %s/%s", pod.Namespace, pod.Name)
		},
	})

	// ── Start the informer factory ─────────────────────────────
	// This starts all registered informers in background goroutines.
	// Each informer will:
	//   1. Perform an initial List to populate its cache
	//   2. Open a Watch to receive incremental updates
	//   3. Begin dispatching events to registered handlers
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	factory.Start(ctx.Done())

	// ── Wait for the cache to sync ─────────────────────────────
	// Before using the Lister, we must wait for the initial List
	// to complete. cache.WaitForCacheSync blocks until all started
	// informers have completed their initial list.
	log.Println("Waiting for informer cache to sync...")
	factory.WaitForCacheSync(ctx.Done())
	log.Println("Cache synced! Now watching for changes...")

	// ── Use the Lister for in-memory reads ─────────────────────
	// The Lister reads from the informer's in-memory cache.
	// These calls never hit the API server. This is why informers
	// are so efficient — reads are O(1) memory lookups.
	lister := podInformer.Lister()

	// List all pods in the "kube-system" namespace.
	systemPods, err := lister.Pods("kube-system").List(labels.Everything())
	if err != nil {
		log.Printf("ERROR: failed to list system pods: %v", err)
	} else {
		log.Printf("Found %d pods in kube-system:", len(systemPods))
		for _, p := range systemPods {
			log.Printf("  - %s (phase: %s)", p.Name, p.Status.Phase)
		}
	}

	// ── Demonstrate direct watch (lower-level) ─────────────────
	// For educational purposes, here's how you would use the raw
	// Watch API directly. In production, always prefer informers.
	go func() {
		time.Sleep(5 * time.Second) // Let the informer demo run first

		log.Println("\n=== Raw Watch Demo (educational only) ===")
		watcher, err := clientset.CoreV1().Pods("default").Watch(
			ctx,
			metav1.ListOptions{
				// Start watching from now (empty resourceVersion = latest).
				// In production, you'd track the last seen resourceVersion.
				ResourceVersion: "",

				// AllowWatchBookmarks: when true, the API server sends
				// periodic BOOKMARK events that update our resourceVersion
				// without any actual object change. This helps us resume
				// after a disconnect without re-listing.
				AllowWatchBookmarks: true,

				// TimeoutSeconds: how long the server keeps the watch
				// connection open. After this, we need to reconnect.
				// The informer handles this automatically.
				TimeoutSeconds: int64Ptr(300),
			},
		)
		if err != nil {
			log.Printf("ERROR: raw watch failed: %v", err)
			return
		}
		defer watcher.Stop()

		for event := range watcher.ResultChan() {
			switch event.Type {
			case "ADDED", "MODIFIED", "DELETED":
				pod, ok := event.Object.(*corev1.Pod)
				if ok {
					log.Printf("[RAW WATCH] %s: %s/%s",
						event.Type, pod.Namespace, pod.Name)
				}
			case "BOOKMARK":
				log.Println("[RAW WATCH] BOOKMARK received")
			case "ERROR":
				log.Printf("[RAW WATCH] ERROR: %v", event.Object)
			}
		}
	}()

	// ── Wait for shutdown signal ───────────────────────────────
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	<-sigCh
	log.Println("Shutting down...")
	cancel()
}

// int64Ptr is a helper that returns a pointer to an int64 value.
// Go doesn't allow taking the address of a literal, so this helper
// is commonly used when constructing Kubernetes API objects.
func int64Ptr(i int64) *int64 {
	return &i
}
```

---

## 12.13 Bringing It All Together

Let's recap the journey of a `kubectl apply` request through the API server:

```
 kubectl apply -f deployment.yaml
       │
       │  HTTPS POST /apis/apps/v1/namespaces/default/deployments
       │  Authorization: Bearer <token>
       │  Content-Type: application/yaml
       │
       ▼
 ┌─ kube-apiserver ─────────────────────────────────────────────────────┐
 │                                                                       │
 │  1. TLS handshake ✓                                                  │
 │                                                                       │
 │  2. Authentication                                                    │
 │     Bearer token → ServiceAccountAuth → "system:serviceaccount:      │
 │     default:deployer" ✓                                              │
 │                                                                       │
 │  3. Authorisation (RBAC)                                             │
 │     Can deployer SA create deployments in default? → Check           │
 │     RoleBindings → Found matching Role → ALLOW ✓                     │
 │                                                                       │
 │  4. Mutating Admission                                               │
 │     a) DefaultStorageClass → N/A (not a PVC)                         │
 │     b) ServiceAccount → N/A (not a pod)                              │
 │     c) MutatingAdmissionWebhook:                                     │
 │        → sidecar-injector: namespace has sidecar-injection=enabled   │
 │          but resource is Deployment, not Pod → SKIP                  │
 │        → default-labels-injector: adds "managed-by: kubectl" ✓      │
 │                                                                       │
 │  5. Schema Validation                                                 │
 │     Validate against apps/v1 Deployment OpenAPI schema ✓             │
 │                                                                       │
 │  6. Validating Admission                                             │
 │     a) ResourceQuota → check namespace quota → within limits ✓       │
 │     b) ValidatingAdmissionWebhook:                                   │
 │        → label-validator: check owner + cost-centre labels → ✓       │
 │     c) ValidatingAdmissionPolicy (CEL):                              │
 │        → require-resource-limits: deployment template has limits → ✓ │
 │                                                                       │
 │  7. Persist to etcd                                                   │
 │     Serialise as protobuf → etcd Put with lease → rv=12345 ✓        │
 │                                                                       │
 │  8. Response                                                          │
 │     HTTP 201 Created                                                  │
 │     Body: the complete Deployment object with server-set fields      │
 │                                                                       │
 │  9. Watch notification (async)                                        │
 │     Watch cache broadcasts ADDED event to all watchers               │
 │     → deployment-controller sees the event                           │
 │     → creates a ReplicaSet (another trip through this pipeline)      │
 │                                                                       │
 └───────────────────────────────────────────────────────────────────────┘
```

### Key Takeaways

1. **The API server is the single entry point** — no component bypasses it.
   This gives you one place for authentication, authorisation, admission,
   and auditing.

2. **Admission controllers are incredibly powerful** — they can modify or
   reject any request. Use mutating webhooks for automatic injection
   (sidecars, labels, defaults) and validating webhooks for policy
   enforcement.

3. **CEL-based policies are the future** — ValidatingAdmissionPolicy
   eliminates the need for webhook infrastructure for simple policies.
   Prefer CEL over webhooks when your validation logic is expressible as
   a CEL expression.

4. **Server-Side Apply is the correct way to manage resources
   programmatically** — it provides proper field ownership and conflict
   detection. Always use it in controllers and automation tools.

5. **Informers are essential** — never poll the API server. The
   Reflector → Informer → Lister pattern is the foundation of every
   Kubernetes controller. It gives you real-time reactivity with
   minimal API server load.

6. **API Priority and Fairness protects the API server** — understand the
   built-in priority levels and create custom FlowSchemas for your
   controllers if they generate significant API traffic.

7. **Understand GVK and GVR** — these are the coordinates that identify
   every type and endpoint in Kubernetes. The Scheme maps between Go
   types and GVKs. The RESTMapper maps between GVKs and GVRs.

### Debugging Checklist

When things go wrong with the API server, check these in order:

```bash
# 1. Can you reach the API server?
kubectl cluster-info

# 2. Are you authenticated?
kubectl auth whoami

# 3. Are you authorised?
kubectl auth can-i <verb> <resource> --namespace=<ns>

# 4. Is admission blocking you?
kubectl apply -f resource.yaml --dry-run=server -v=6

# 5. Check audit logs for the full request lifecycle
kubectl logs -n kube-system kube-apiserver-<node> | grep <resource-name>

# 6. Check webhook configurations
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations
kubectl get validatingadmissionpolicies

# 7. Check APF for throttling
kubectl get flowschemas
kubectl get prioritylevelconfigurations

# 8. Check API server metrics for errors
kubectl get --raw /metrics | grep -E 'apiserver_request_total.*code="(4|5)"'
```

---

## Summary

```
┌────────────────────────────────────────────────────────────────────────┐
│                     Chapter 12 Concept Map                             │
│                                                                        │
│  Request Pipeline:                                                     │
│    TLS → AuthN → AuthZ → Mutating Admission → Validation              │
│    → Validating Admission → etcd → Response → Watch Notify             │
│                                                                        │
│  Authentication:                                                       │
│    X.509 certs | Bearer tokens (SA, OIDC) | Proxy | Webhook           │
│                                                                        │
│  Authorisation:                                                        │
│    RBAC (primary) | ABAC | Webhook | Node                             │
│    Decisions: Allow | Deny | NoOpinion                                 │
│                                                                        │
│  Admission:                                                            │
│    Built-in controllers | Mutating webhooks | Validating webhooks      │
│    | CEL-based ValidatingAdmissionPolicy                               │
│                                                                        │
│  API Structure:                                                        │
│    Core (/api/v1) | Named (/apis/<group>/<version>)                    │
│    Version lifecycle: alpha → beta → stable                            │
│    GVK (type identity) | GVR (REST endpoint)                           │
│                                                                        │
│  Extension Points:                                                     │
│    Admission webhooks | API aggregation | CRDs                         │
│                                                                        │
│  Efficiency:                                                           │
│    Watch cache | Informers | API Priority & Fairness                   │
│    Server-Side Apply for field management                              │
│                                                                        │
│  Go Libraries:                                                         │
│    k8s.io/apimachinery (types, scheme, GVK)                            │
│    k8s.io/client-go (clientset, informers, SSA)                        │
│    k8s.io/api (typed API objects)                                      │
│                                                                        │
│  Next: Chapter 13 — etcd & the Consistency Model                       │
└────────────────────────────────────────────────────────────────────────┘
```

---

*In the next chapter, we descend one layer deeper — into etcd, the
distributed key-value store that holds the entire state of your cluster.
We will explore its consistency model, its watch mechanism, how the API
server uses it, and what happens when etcd fails.*
