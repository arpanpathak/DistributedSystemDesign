# Chapter 29: Service Accounts, OIDC & Authentication

## Introduction

Authentication is the first gate a request must pass through before it can do anything
in a Kubernetes cluster. Every single API call — whether it originates from a human
running `kubectl`, a Pod calling the API server from inside the cluster, or an
external CI/CD pipeline — must present credentials that the API server can verify.
Get authentication wrong, and your cluster is either an open door (too permissive)
or an impenetrable wall (too restrictive for legitimate workloads).

This chapter explores every authentication mechanism Kubernetes supports, from X.509
client certificates baked in at cluster bootstrap to modern OIDC flows that integrate
with enterprise identity providers. We will examine how ServiceAccount tokens evolved
from long-lived secrets to short-lived, audience-bound projected tokens. We will
integrate with real identity providers, create certificates by hand, and write Go
programs that authenticate both from inside and outside the cluster.

**What you will learn:**

- How the Kubernetes API server authenticates requests
- Every authentication strategy: certificates, tokens, OIDC, webhooks, proxies
- The evolution of ServiceAccount tokens and why it matters
- How to integrate with identity providers (Keycloak, Dex, Azure AD, Google)
- kubeconfig internals — clusters, users, contexts
- Impersonation for debugging and multi-tenancy
- Hands-on Go programs for in-cluster and out-of-cluster authentication

---

## 29.1 The Authentication Pipeline

When a request arrives at the Kubernetes API server, it passes through a chain of
authenticators. Each authenticator examines the request and either:

1. **Authenticates it** — returns the user identity (username, UID, groups, extra fields)
2. **Passes** — cannot authenticate, hands off to the next authenticator

If **no** authenticator succeeds, the request is rejected with `401 Unauthorized`.

```text
┌─────────────────────────────────────────────────────────────────┐
│                      API Server Request                         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
              ┌────────────────────────┐
              │  X.509 Client Certs    │──── success ──▶ Authenticated
              └────────────┬───────────┘
                           │ pass
                           ▼
              ┌────────────────────────┐
              │  Bearer Token (static) │──── success ──▶ Authenticated
              └────────────┬───────────┘
                           │ pass
                           ▼
              ┌────────────────────────┐
              │  ServiceAccount Token  │──── success ──▶ Authenticated
              └────────────┬───────────┘
                           │ pass
                           ▼
              ┌────────────────────────┐
              │  OIDC Token            │──── success ──▶ Authenticated
              └────────────┬───────────┘
                           │ pass
                           ▼
              ┌────────────────────────┐
              │  Webhook Token         │──── success ──▶ Authenticated
              └────────────┬───────────┘
                           │ pass
                           ▼
              ┌────────────────────────┐
              │  Authenticating Proxy  │──── success ──▶ Authenticated
              └────────────┬───────────┘
                           │ pass
                           ▼
                    401 Unauthorized
```

**Key principle:** Authenticators are **additive**. Multiple can be enabled
simultaneously. The first one to succeed wins.

### 29.1.1 The Authenticated User Object

Every successful authentication produces a `user.Info` object:

```go
// UserInfo describes the authenticated user. Every authenticator
// must populate at least the Name field. Groups and Extra are
// optional but critical for RBAC decisions downstream.
type UserInfo struct {
	// Name is the unique identifier for this user.
	// For ServiceAccounts: "system:serviceaccount:<namespace>:<name>"
	// For certificates: the Common Name (CN) from the client cert
	// For OIDC: the claim specified by --oidc-username-claim
	Name string

	// UID is an optional unique identifier that distinguishes
	// users with the same Name. Not all authenticators set this.
	UID string

	// Groups are the logical groups this user belongs to.
	// For certificates: extracted from the Organization (O) fields
	// For OIDC: extracted from the configured groups claim
	// Built-in groups include "system:masters" and "system:authenticated"
	Groups []string

	// Extra contains additional provider-specific information.
	// For example, OIDC authenticators might include the original
	// token claims here.
	Extra map[string][]string
}
```

### 29.1.2 Anonymous Authentication

If no authenticator succeeds and anonymous authentication is enabled (the default),
the request proceeds as:
- **Username:** `system:anonymous`
- **Group:** `system:unauthenticated`

This is controlled by `--anonymous-auth=true|false` on the API server. In production,
RBAC should ensure that `system:anonymous` has almost no permissions.

**Attack vector:** If anonymous auth is enabled and RBAC is misconfigured (e.g., a
`ClusterRoleBinding` grants permissions to `system:unauthenticated`), an attacker
with network access to the API server can read or modify cluster resources without
any credentials.

---

## 29.2 X.509 Client Certificate Authentication

This is the **oldest and most fundamental** authentication method in Kubernetes.
When you bootstrap a cluster with kubeadm, every component (kubelet, controller-manager,
scheduler) authenticates using client certificates.

### 29.2.1 How It Works

1. The API server is configured with a Certificate Authority (CA) via `--client-ca-file`
2. The client presents a TLS client certificate during the TLS handshake
3. The API server verifies the certificate chain against the CA
4. If valid, user identity is extracted:
   - **Username** ← Certificate Common Name (CN)
   - **Groups** ← Certificate Organization (O) fields

```
┌──────────────┐                      ┌──────────────┐
│   kubectl    │ ─── TLS handshake ──▶│  API Server  │
│              │     + client cert     │              │
│  CN=alice    │                       │  Verifies    │
│  O=dev-team  │◀── mTLS established ─│  against CA  │
└──────────────┘                       └──────────────┘

User Identity:
  Name:   alice
  Groups: [dev-team, system:authenticated]
```

### 29.2.2 kubeadm Certificate Layout

When `kubeadm init` bootstraps a cluster, it creates an entire PKI under
`/etc/kubernetes/pki/`:

```
/etc/kubernetes/pki/
├── ca.crt                    # Cluster CA certificate
├── ca.key                    # Cluster CA private key
├── apiserver.crt             # API server's serving certificate
├── apiserver.key             # API server's private key
├── apiserver-kubelet-client.crt  # API server → kubelet client cert
├── apiserver-kubelet-client.key
├── front-proxy-ca.crt        # CA for authenticating proxy
├── front-proxy-ca.key
├── front-proxy-client.crt    # Front proxy client cert
├── front-proxy-client.key
├── sa.key                    # ServiceAccount token signing key
├── sa.pub                    # ServiceAccount token verification key
└── etcd/
    ├── ca.crt                # Separate CA for etcd
    ├── ca.key
    ├── server.crt            # etcd server cert
    ├── server.key
    ├── peer.crt              # etcd peer-to-peer cert
    ├── peer.key
    ├── healthcheck-client.crt
    └── healthcheck-client.key
```

**Security implication:** Anyone who obtains `ca.key` can mint certificates for
**any** user, including `system:masters` (full cluster admin). This file must be
protected with the highest level of security. Some production setups delete it
from the cluster after bootstrap and store it offline.

### 29.2.3 Hands-On: Creating a Certificate-Based User

Let's create a new user "alice" in group "dev-team" using OpenSSL:

```bash
# Step 1: Generate a private key for alice
openssl genrsa -out alice.key 4096

# Step 2: Create a Certificate Signing Request (CSR)
# CN = username, O = group (can have multiple O fields for multiple groups)
openssl req -new -key alice.key -out alice.csr \
  -subj "/CN=alice/O=dev-team/O=qa-team"

# Step 3: Sign the CSR with the cluster CA
# This requires access to the CA key — typically done by a cluster admin
openssl x509 -req -in alice.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out alice.crt \
  -days 365

# Step 4: Verify the certificate
openssl x509 -in alice.crt -text -noout | grep -E "Subject:|Issuer:"
# Subject: CN = alice, O = dev-team, O = qa-team
# Issuer: CN = kubernetes
```

### 29.2.4 Using Kubernetes CertificateSigningRequest API

Instead of directly accessing the CA key, you can use the Kubernetes CSR API:

```bash
# Create a CSR object
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: alice-csr
spec:
  request: $(cat alice.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
    - client auth
EOF

# Approve the CSR (requires appropriate RBAC)
kubectl certificate approve alice-csr

# Retrieve the signed certificate
kubectl get csr alice-csr -o jsonpath='{.status.certificate}' | base64 -d > alice.crt
```

### 29.2.5 Limitations of Certificate Authentication

- **No revocation:** Kubernetes does not check Certificate Revocation Lists (CRLs)
  or use OCSP. Once a certificate is issued, it's valid until expiry.
- **No fine-grained expiry:** You set the expiry at signing time. There's no way to
  revoke a single user's access without rotating the entire CA.
- **Group membership is baked in:** If alice changes teams, you need to issue a new
  certificate.

**Mitigation:** Issue short-lived certificates (hours, not years) and use an automated
certificate management system.

---

## 29.3 Bearer Token Authentication

### 29.3.1 Static Token File (Deprecated)

The API server can read a CSV file of tokens via `--token-auth-file`:

```csv
token,user,uid,"group1,group2"
abc123,admin,admin-uid,"system:masters"
def456,developer,dev-uid,"developers"
```

**This is deprecated and dangerous.** Tokens are:
- Stored in plaintext on disk
- Never expire
- Require API server restart to change
- Cannot be revoked without restart

**Attack vector:** If an attacker reads this file (e.g., via a container escape or
node compromise), they have permanent admin access.

### 29.3.2 Bootstrap Tokens

Bootstrap tokens are a special class of bearer tokens used for cluster bootstrapping
and node joining. They are stored as Secrets in the `kube-system` namespace:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-abc123
  namespace: kube-system
type: bootstrap.kubernetes.io/token
data:
  token-id: YWJjMTIz          # base64("abc123")
  token-secret: ZGVmNDU2Nzg5  # base64("def456789")
  usage-bootstrap-authentication: dHJ1ZQ==  # base64("true")
  usage-bootstrap-signing: dHJ1ZQ==
  expiration: MjAyNS0xMi0zMVQyMzo1OTo1OVo=  # base64 ISO 8601 timestamp
```

The token format is `<token-id>.<token-secret>` (e.g., `abc123.def456789`).

---

## 29.4 ServiceAccount Tokens — The Evolution

ServiceAccount tokens are the primary authentication mechanism for workloads running
**inside** the cluster. Their implementation has evolved significantly.

### 29.4.1 Legacy ServiceAccount Tokens (Pre-1.24)

Before Kubernetes 1.24, creating a ServiceAccount automatically generated a long-lived
Secret containing a JWT:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-sa-token-abc12
  namespace: default
  annotations:
    kubernetes.io/service-account.name: my-sa
type: kubernetes.io/service-account-token
data:
  token: <base64-encoded-jwt>
  ca.crt: <base64-encoded-ca-cert>
  namespace: <base64-encoded-namespace>
```

**Problems with legacy tokens:**

1. **Never expire** — the token is valid forever (or until the Secret is deleted)
2. **Not audience-bound** — the same token works for any audience
3. **Not bound to a Pod** — if the token leaks, it can be used from anywhere
4. **Auto-mounted** — every Pod got one, even if it never needed API access
5. **Stored in etcd** — adds to the attack surface of etcd compromise

**Attack vector:** A leaked legacy ServiceAccount token gives an attacker permanent
access to whatever the ServiceAccount's RBAC allows. If the SA has cluster-admin
(a common misconfiguration), the attacker owns the cluster.

### 29.4.2 Projected (Bound) ServiceAccount Tokens (1.20+)

Starting in Kubernetes 1.20 (default in 1.21+), Pods receive **projected** tokens
instead of legacy ones. These are fundamentally different:

| Property           | Legacy Token          | Projected Token              |
|--------------------|-----------------------|------------------------------|
| Lifetime           | Infinite              | Configurable (default 1h)    |
| Audience           | Any                   | Bound to specific audience   |
| Bound to Pod       | No                    | Yes (invalidated on delete)  |
| Stored in Secret   | Yes (in etcd)         | No (generated on demand)     |
| Auto-refreshed     | No                    | Yes (kubelet refreshes)      |

The projected token volume configuration:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-sa
  containers:
    - name: app
      image: my-app:latest
      volumeMounts:
        - name: token
          mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          readOnly: true
  volumes:
    - name: token
      projected:
        sources:
          - serviceAccountToken:
              # The audience field restricts who can accept this token.
              # The API server only accepts tokens with its own audience.
              audience: "https://kubernetes.default.svc"
              # expirationSeconds controls token lifetime.
              # The kubelet refreshes the token before it expires.
              # Minimum is 600 seconds (10 minutes).
              expirationSeconds: 3600
              path: token
          - configMap:
              name: kube-root-ca.crt
              items:
                - key: ca.crt
                  path: ca.crt
          - downwardAPI:
              items:
                - path: namespace
                  fieldRef:
                    fieldPath: metadata.namespace
```

### 29.4.3 The TokenRequest API

The `TokenRequest` API (stable since 1.22) allows programmatic creation of bound
ServiceAccount tokens:

```bash
# Request a token with a specific audience and expiration
kubectl create token my-sa \
  --audience=https://my-service.example.com \
  --duration=30m

# The returned JWT contains:
# - iss: https://kubernetes.default.svc
# - sub: system:serviceaccount:default:my-sa
# - aud: ["https://my-service.example.com"]
# - exp: <30 minutes from now>
# - kubernetes.io/serviceaccount/namespace: default
# - kubernetes.io/serviceaccount/service-account.name: my-sa
```

### 29.4.4 Anatomy of a ServiceAccount JWT

Let's decode a projected token:

```json
{
  "header": {
    "alg": "RS256",
    "kid": "abc123..."
  },
  "payload": {
    "aud": ["https://kubernetes.default.svc"],
    "exp": 1700000000,
    "iat": 1699996400,
    "iss": "https://kubernetes.default.svc",
    "jti": "unique-token-id",
    "kubernetes.io": {
      "namespace": "default",
      "pod": {
        "name": "my-app-xyz",
        "uid": "pod-uid-here"
      },
      "serviceaccount": {
        "name": "my-sa",
        "uid": "sa-uid-here"
      }
    },
    "nbf": 1699996400,
    "sub": "system:serviceaccount:default:my-sa"
  }
}
```

**Bound claims:** The token includes the Pod name and UID. If the Pod is deleted,
the token is immediately invalidated by the TokenReview API — even if it hasn't
expired yet.

### 29.4.5 Hands-On: Creating a ServiceAccount with Bound Token

```bash
# Create a ServiceAccount
kubectl create serviceaccount ci-deployer -n staging

# Create an RBAC role that allows deploying
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: deployer
  namespace: staging
rules:
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "create", "update", "patch"]
  - apiGroups: [""]
    resources: ["services", "configmaps"]
    verbs: ["get", "list", "create", "update"]
EOF

# Bind the role to the ServiceAccount
kubectl create rolebinding ci-deployer-binding \
  --role=deployer \
  --serviceaccount=staging:ci-deployer \
  -n staging

# Request a short-lived token for CI/CD usage
TOKEN=$(kubectl create token ci-deployer -n staging --duration=1h)

# Use the token to authenticate
kubectl --token="$TOKEN" get deployments -n staging
```

---

## 29.5 OIDC Authentication

OpenID Connect (OIDC) is the **recommended** authentication method for human users
in production clusters. It delegates authentication to an external identity provider
(IdP) while Kubernetes only verifies the resulting JWT.

### 29.5.1 How OIDC Authentication Works

```
┌─────────────┐     1. Login      ┌──────────────────┐
│   User      │ ─────────────────▶│  Identity Provider│
│  (Browser)  │                    │  (Keycloak, AAD,  │
│             │◀─── 2. ID Token ──│   Dex, Google)    │
└──────┬──────┘                    └──────────────────┘
       │
       │ 3. kubectl --token=<id_token>
       ▼
┌──────────────┐     4. Verify JWT     ┌──────────────────┐
│   kubectl    │ ─────────────────────▶│   API Server     │
│              │                        │                  │
│              │◀── 5. Authenticated ──│  Checks:         │
└──────────────┘                        │  - Signature     │
                                        │  - Issuer        │
                                        │  - Audience      │
                                        │  - Expiry        │
                                        │  - Claims mapping│
                                        └──────────────────┘
```

**Key property:** The API server **never** contacts the IdP at request time.
It only needs the IdP's public keys (fetched from the OIDC discovery endpoint
at startup and periodically refreshed). This means authentication works even if
the IdP is temporarily unavailable (as long as tokens haven't expired).

### 29.5.2 API Server OIDC Configuration

```bash
# API server flags for OIDC authentication
kube-apiserver \
  --oidc-issuer-url=https://keycloak.example.com/realms/kubernetes \
  --oidc-client-id=kubernetes \
  --oidc-username-claim=email \
  --oidc-username-prefix="oidc:" \
  --oidc-groups-claim=groups \
  --oidc-groups-prefix="oidc:" \
  --oidc-ca-file=/etc/kubernetes/pki/oidc-ca.crt \
  --oidc-required-claim="kubernetes_access=true"
```

| Flag                    | Purpose                                               |
|-------------------------|-------------------------------------------------------|
| `--oidc-issuer-url`     | The IdP's issuer URL (must use HTTPS)                 |
| `--oidc-client-id`      | The OAuth2 client ID — tokens must have this audience |
| `--oidc-username-claim`  | JWT claim to use as username (default: `sub`)         |
| `--oidc-username-prefix` | Prefix to prevent collision with other auth methods   |
| `--oidc-groups-claim`    | JWT claim containing group memberships                |
| `--oidc-groups-prefix`   | Prefix for group names                                |
| `--oidc-ca-file`         | CA to verify the IdP's TLS certificate                |
| `--oidc-required-claim`  | Additional claim that must be present in the token    |

### 29.5.3 Structured Authentication Configuration (1.30+)

Starting in Kubernetes 1.30, you can configure multiple OIDC providers using
a structured configuration file instead of command-line flags:

```yaml
# /etc/kubernetes/auth-config.yaml
apiVersion: apiserver.config.k8s.io/v1beta1
kind: AuthenticationConfiguration
jwt:
  - issuer:
      url: https://keycloak.example.com/realms/kubernetes
      audiences:
        - kubernetes
      certificateAuthority: |
        -----BEGIN CERTIFICATE-----
        ...
        -----END CERTIFICATE-----
    claimMappings:
      username:
        claim: email
        prefix: "keycloak:"
      groups:
        claim: groups
        prefix: "keycloak:"
    claimValidationRules:
      - claim: email_verified
        requiredValue: "true"
  - issuer:
      url: https://accounts.google.com
      audiences:
        - kubernetes-google
    claimMappings:
      username:
        claim: email
        prefix: "google:"
```

```bash
kube-apiserver --authentication-config=/etc/kubernetes/auth-config.yaml
```

### 29.5.4 Identity Providers

#### Keycloak

Keycloak is a popular open-source identity provider:

```bash
# Deploy Keycloak
helm repo add codecentric https://codecentric.github.io/helm-charts
helm install keycloak codecentric/keycloak \
  --set ingress.enabled=true \
  --set ingress.rules[0].host=keycloak.example.com

# After setup, create:
# 1. A realm called "kubernetes"
# 2. A client called "kubernetes" (confidential, with valid redirect URIs)
# 3. A "groups" client scope with a groups mapper
# 4. Users with group memberships
```

#### Dex

Dex is a lightweight OIDC provider that federates to upstream IdPs:

```yaml
# Dex configuration
issuer: https://dex.example.com
storage:
  type: kubernetes
  config:
    inCluster: true
connectors:
  - type: github
    id: github
    name: GitHub
    config:
      clientID: $GITHUB_CLIENT_ID
      clientSecret: $GITHUB_CLIENT_SECRET
      redirectURI: https://dex.example.com/callback
      orgs:
        - name: my-org
          teams:
            - platform-team
            - dev-team
  - type: ldap
    id: ldap
    name: Corporate LDAP
    config:
      host: ldap.corp.example.com:636
      rootCAData: <base64-ca>
      bindDN: cn=serviceaccount,dc=example,dc=com
      bindPW: $LDAP_PASSWORD
      userSearch:
        baseDN: ou=people,dc=example,dc=com
        username: uid
        emailAttr: mail
      groupSearch:
        baseDN: ou=groups,dc=example,dc=com
        userMatchers:
          - userAttr: DN
            groupAttr: member
```

#### Azure AD (Microsoft Entra ID)

```bash
# API server configuration for Azure AD
kube-apiserver \
  --oidc-issuer-url=https://login.microsoftonline.com/<tenant-id>/v2.0 \
  --oidc-client-id=<application-id> \
  --oidc-username-claim=email \
  --oidc-groups-claim=groups
```

For AKS clusters, Azure AD integration is built-in:

```bash
# Enable Azure AD integration on AKS
az aks update -g myRG -n myCluster --enable-aad \
  --aad-admin-group-object-ids <group-object-id>
```

#### Google

```bash
kube-apiserver \
  --oidc-issuer-url=https://accounts.google.com \
  --oidc-client-id=<google-client-id>.apps.googleusercontent.com \
  --oidc-username-claim=email \
  --oidc-groups-claim=groups
```

### 29.5.5 kubectl OIDC Integration

Users authenticate via an OIDC helper or credential plugin:

```yaml
# kubeconfig with OIDC credentials
apiVersion: v1
kind: Config
clusters:
  - cluster:
      server: https://k8s.example.com:6443
      certificate-authority: /path/to/ca.crt
    name: production
users:
  - name: alice
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1beta1
        command: kubectl
        args:
          - oidc-login
          - get-token
          - --oidc-issuer-url=https://keycloak.example.com/realms/kubernetes
          - --oidc-client-id=kubernetes
          - --oidc-client-secret=<secret>
contexts:
  - context:
      cluster: production
      user: alice
    name: production-alice
current-context: production-alice
```

### 29.5.6 Hands-On: Setting Up OIDC with Dex and kubelogin

```bash
# 1. Install Dex
helm repo add dex https://charts.dexidp.io
helm install dex dex/dex -f dex-values.yaml -n dex-system

# 2. Install kubelogin (kubectl oidc-login plugin)
kubectl krew install oidc-login

# 3. Configure kubeconfig
kubectl config set-credentials oidc-user \
  --exec-api-version=client.authentication.k8s.io/v1beta1 \
  --exec-command=kubectl \
  --exec-arg=oidc-login \
  --exec-arg=get-token \
  --exec-arg=--oidc-issuer-url=https://dex.example.com \
  --exec-arg=--oidc-client-id=kubernetes \
  --exec-arg=--oidc-client-secret=kubernetes-secret

# 4. Set context
kubectl config set-context oidc-context \
  --cluster=my-cluster \
  --user=oidc-user

# 5. Test authentication
kubectl --context=oidc-context get pods
# This opens a browser for OIDC login flow
```

---

## 29.6 Webhook Token Authentication

Webhook authentication delegates token verification to an external service:

```bash
kube-apiserver \
  --authentication-token-webhook-config-file=/etc/kubernetes/webhook-config.yaml \
  --authentication-token-webhook-cache-ttl=2m
```

```yaml
# /etc/kubernetes/webhook-config.yaml
apiVersion: v1
kind: Config
clusters:
  - name: auth-webhook
    cluster:
      server: https://auth.example.com/authenticate
      certificate-authority: /etc/kubernetes/pki/webhook-ca.crt
users:
  - name: apiserver
    user:
      client-certificate: /etc/kubernetes/pki/webhook-client.crt
      client-key: /etc/kubernetes/pki/webhook-client.key
current-context: webhook
contexts:
  - context:
      cluster: auth-webhook
      user: apiserver
    name: webhook
```

The API server sends a `TokenReview` to the webhook:

```json
{
  "apiVersion": "authentication.k8s.io/v1",
  "kind": "TokenReview",
  "spec": {
    "token": "the-bearer-token-from-the-request"
  }
}
```

The webhook responds with the authenticated user:

```json
{
  "apiVersion": "authentication.k8s.io/v1",
  "kind": "TokenReview",
  "status": {
    "authenticated": true,
    "user": {
      "username": "alice@example.com",
      "uid": "alice-uid",
      "groups": ["developers", "system:authenticated"],
      "extra": {
        "auth-method": ["oauth2"]
      }
    }
  }
}
```

---

## 29.7 Authenticating Proxy

An authenticating proxy sits in front of the API server and performs authentication
itself, passing the user identity in HTTP headers:

```bash
kube-apiserver \
  --requestheader-client-ca-file=/etc/kubernetes/pki/front-proxy-ca.crt \
  --requestheader-allowed-names=front-proxy-client \
  --requestheader-username-headers=X-Remote-User \
  --requestheader-group-headers=X-Remote-Group \
  --requestheader-extra-headers-prefix=X-Remote-Extra-
```

**How it works:**

1. The proxy authenticates the user (e.g., via SSO, SAML, etc.)
2. The proxy connects to the API server using a client certificate that the API
   server trusts (verified against `--requestheader-client-ca-file`)
3. The proxy sets headers: `X-Remote-User`, `X-Remote-Group`
4. The API server trusts these headers because the proxy's cert is valid

**Attack vector:** If an attacker can send requests directly to the API server
(bypassing the proxy) with these headers, and the API server accepts them, the
attacker can impersonate any user. The `--requestheader-client-ca-file` and
`--requestheader-allowed-names` flags prevent this by requiring the proxy to
present a valid certificate.

---

## 29.8 kubeconfig Structure

The kubeconfig file (`~/.kube/config`) defines how kubectl connects to clusters:

```yaml
apiVersion: v1
kind: Config

# Clusters define API server endpoints and their CA certificates
clusters:
  - name: production
    cluster:
      server: https://k8s-prod.example.com:6443
      certificate-authority-data: <base64-ca-cert>
      # Optional: disable TLS verification (NEVER in production)
      # insecure-skip-tls-verify: true
  - name: staging
    cluster:
      server: https://k8s-staging.example.com:6443
      certificate-authority-data: <base64-ca-cert>

# Users define credentials for authentication
users:
  - name: admin
    user:
      client-certificate-data: <base64-cert>
      client-key-data: <base64-key>
  - name: developer
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1beta1
        command: aws
        args:
          - eks
          - get-token
          - --cluster-name
          - my-cluster
  - name: ci-bot
    user:
      token: <bearer-token>

# Contexts combine a cluster + user + optional namespace
contexts:
  - name: prod-admin
    context:
      cluster: production
      user: admin
      namespace: default
  - name: staging-dev
    context:
      cluster: staging
      user: developer
      namespace: development

# The currently active context
current-context: staging-dev
```

### 29.8.1 Credential Plugins (ExecCredential)

Modern kubeconfigs use credential plugins that dynamically fetch tokens:

```yaml
users:
  - name: oidc-user
    user:
      exec:
        apiVersion: client.authentication.k8s.io/v1
        command: /usr/local/bin/my-auth-plugin
        args:
          - --provider=oidc
          - --issuer=https://auth.example.com
        env:
          - name: PLUGIN_CACHE_DIR
            value: /home/user/.kube/cache
        # installHint is shown if the command is not found
        installHint: |
          Install the auth plugin:
            brew install my-auth-plugin
        # provideClusterInfo sends cluster details to the plugin
        provideClusterInfo: true
```

The plugin returns an `ExecCredential`:

```json
{
  "apiVersion": "client.authentication.k8s.io/v1",
  "kind": "ExecCredential",
  "status": {
    "token": "my-bearer-token",
    "expirationTimestamp": "2025-01-01T00:00:00Z"
  }
}
```

---

## 29.9 Impersonation

Impersonation allows an authenticated user to act as a different user. This is
powerful for debugging, multi-tenancy, and building proxy systems.

### 29.9.1 How Impersonation Works

```bash
# Act as user "alice"
kubectl get pods --as=alice

# Act as user "alice" in group "dev-team"
kubectl get pods --as=alice --as-group=dev-team

# Act as a ServiceAccount
kubectl get pods --as=system:serviceaccount:default:my-sa
```

The impersonation headers:

| Header                          | Description                   |
|---------------------------------|-------------------------------|
| `Impersonate-User`              | The user to impersonate       |
| `Impersonate-Group`             | The group (can be repeated)   |
| `Impersonate-Uid`               | The UID of the user           |
| `Impersonate-Extra-<key>`       | Extra fields                  |

### 29.9.2 RBAC for Impersonation

Impersonation requires explicit RBAC permission:

```yaml
# Allow "ci-system" SA to impersonate users in "dev-team"
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: impersonator
rules:
  - apiGroups: [""]
    resources: ["users"]
    verbs: ["impersonate"]
    resourceNames: ["alice", "bob"]  # Limit who can be impersonated
  - apiGroups: [""]
    resources: ["groups"]
    verbs: ["impersonate"]
    resourceNames: ["dev-team"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: ci-impersonator
subjects:
  - kind: ServiceAccount
    name: ci-system
    namespace: ci
roleRef:
  kind: ClusterRole
  name: impersonator
  apiGroup: rbac.authorization.k8s.io
```

**Attack vector:** Over-permissive impersonation rules can allow privilege escalation.
An SA that can impersonate any user effectively has the permissions of all users combined.
Always restrict `resourceNames` in impersonation rules.

---

## 29.10 Hands-On: Go Programs for Authentication

### 29.10.1 In-Cluster Authentication

This program runs inside a Pod and uses the projected ServiceAccount token:

```go
// Package main demonstrates in-cluster authentication using
// the projected ServiceAccount token automatically mounted
// into every Pod. This is the standard way for applications
// running inside Kubernetes to talk to the API server.
//
// The in-cluster config reads:
//   - Token from /var/run/secrets/kubernetes.io/serviceaccount/token
//   - CA cert from /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
//   - API server URL from KUBERNETES_SERVICE_HOST and KUBERNETES_SERVICE_PORT
package main

import (
	"context"
	"fmt"
	"os"
	"time"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
)

func main() {
	// BuildConfigFromFlags with empty strings triggers in-cluster config
	// detection. This reads the projected ServiceAccount token and CA
	// certificate from well-known file paths mounted by the kubelet.
	//
	// The token is automatically refreshed by the kubelet before it
	// expires, so long-running applications never need to worry about
	// token rotation — the client-go library re-reads the file on
	// each request.
	config, err := rest.InClusterConfig()
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error building in-cluster config: %v\n", err)
		os.Exit(1)
	}

	// Create a typed Kubernetes clientset. This provides access to
	// every built-in API group (core/v1, apps/v1, batch/v1, etc.)
	// through strongly-typed Go methods.
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error creating clientset: %v\n", err)
		os.Exit(1)
	}

	// Use a timeout context to ensure we don't hang forever if the
	// API server is unreachable or slow to respond.
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	// List Pods in the current namespace. The namespace is read from
	// the ServiceAccount token's namespace claim, which is also
	// available at /var/run/secrets/kubernetes.io/serviceaccount/namespace.
	namespace, err := os.ReadFile(
		"/var/run/secrets/kubernetes.io/serviceaccount/namespace",
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error reading namespace: %v\n", err)
		os.Exit(1)
	}

	// ListOptions can filter and paginate results. An empty ListOptions
	// returns all Pods in the namespace (subject to RBAC permissions).
	pods, err := clientset.CoreV1().Pods(string(namespace)).List(
		ctx, metav1.ListOptions{},
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error listing pods: %v\n", err)
		os.Exit(1)
	}

	fmt.Printf("Found %d pods in namespace %s:\n", len(pods.Items), namespace)
	for _, pod := range pods.Items {
		// Display each Pod's name, phase (Running, Pending, etc.),
		// and the node it's scheduled on.
		fmt.Printf("  - %s (phase: %s, node: %s)\n",
			pod.Name, pod.Status.Phase, pod.Spec.NodeName)
	}
}
```

### 29.10.2 Out-of-Cluster Authentication

This program runs outside the cluster and uses a kubeconfig file:

```go
// Package main demonstrates out-of-cluster authentication by
// loading credentials from a kubeconfig file. This is the pattern
// used by CLI tools, CI/CD pipelines, and local development
// programs that need to interact with a Kubernetes cluster.
//
// The program uses the following precedence for finding the
// kubeconfig:
//   1. --kubeconfig flag (if provided)
//   2. KUBECONFIG environment variable
//   3. ~/.kube/config (default path)
package main

import (
	"context"
	"flag"
	"fmt"
	"os"
	"path/filepath"
	"time"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
)

func main() {
	// Define the kubeconfig flag. If the user has a home directory,
	// default to ~/.kube/config. Otherwise, require the flag.
	var kubeconfig *string
	if home := homedir.HomeDir(); home != "" {
		kubeconfig = flag.String("kubeconfig",
			filepath.Join(home, ".kube", "config"),
			"(optional) absolute path to the kubeconfig file")
	} else {
		kubeconfig = flag.String("kubeconfig", "",
			"absolute path to the kubeconfig file")
	}

	// contextName allows switching between kubeconfig contexts
	// without modifying the current-context in the file.
	contextName := flag.String("context", "",
		"kubeconfig context to use (default: current-context)")
	flag.Parse()

	// Build the config using the specified kubeconfig and context.
	// clientcmd handles all the complexity of:
	//   - Loading and merging multiple kubeconfig files (KUBECONFIG can
		//     contain multiple paths separated by the OS path delimiter)
	//   - Resolving the current context
	//   - Executing credential plugins (exec-based authentication)
	//   - Loading certificates from files or inline data
	loadingRules := &clientcmd.ClientConfigLoadingRules{
		ExplicitPath: *kubeconfig,
	}
	configOverrides := &clientcmd.ConfigOverrides{}
	if *contextName != "" {
		configOverrides.CurrentContext = *contextName
	}

	config, err := clientcmd.NewNonInteractiveDeferredLoadingClientConfig(
		loadingRules, configOverrides,
	).ClientConfig()
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error building config: %v\n", err)
		os.Exit(1)
	}

	// Create the clientset with the loaded configuration.
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error creating clientset: %v\n", err)
		os.Exit(1)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	// List all namespaces — a common smoke test that requires only
	// the "list" verb on "namespaces" (a cluster-scoped resource).
	namespaces, err := clientset.CoreV1().Namespaces().List(
		ctx, metav1.ListOptions{},
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error listing namespaces: %v\n", err)
		os.Exit(1)
	}

	fmt.Printf("Connected to cluster. Found %d namespaces:\n",
		len(namespaces.Items))
	for _, ns := range namespaces.Items {
		fmt.Printf("  - %s (status: %s)\n", ns.Name, ns.Status.Phase)
	}

	// Also demonstrate that we can read our own identity by
	// performing a self-subject access review. This tells us
	// what the API server thinks our identity is.
	selfReview, err := clientset.AuthenticationV1().
	SelfSubjectReviews().
	Create(ctx, &authv1.SelfSubjectReview{}, metav1.CreateOptions{})
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error checking identity: %v\n", err)
	} else {
		fmt.Printf("\nAuthenticated as: %s\n",
			selfReview.Status.UserInfo.Username)
		fmt.Printf("Groups: %v\n",
			selfReview.Status.UserInfo.Groups)
	}
}
```

### 29.10.3 Creating and Using ServiceAccount Tokens Programmatically

```go
// Package main demonstrates programmatic creation and usage of
// ServiceAccount tokens using the TokenRequest API. This is
// essential for building controllers, operators, and automation
// tools that need to create scoped credentials for other
// components.
//
// The TokenRequest API creates short-lived, audience-bound tokens
// that are superior to legacy long-lived ServiceAccount secrets.
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"os"
	"time"

	authenticationv1 "k8s.io/api/authentication/v1"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/homedir"
	"k8s.io/utils/ptr"
)

func main() {
	// Load kubeconfig — the caller must have permission to create
	// ServiceAccounts and TokenRequests.
	config, err := clientcmd.BuildConfigFromFlags("",
		filepath.Join(homedir.HomeDir(), ".kube", "config"))
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error loading config: %v\n", err)
		os.Exit(1)
	}

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error creating clientset: %v\n", err)
		os.Exit(1)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
	defer cancel()

	namespace := "default"

	// Step 1: Create a ServiceAccount for our workload.
	// In production, you would also create appropriate RBAC rules
	// to limit what this ServiceAccount can do.
	sa := &corev1.ServiceAccount{
		ObjectMeta: metav1.ObjectMeta{
			Name:      "token-demo-sa",
			Namespace: namespace,
			Labels: map[string]string{
				"app":        "token-demo",
				"managed-by": "go-program",
			},
		},
		// AutomountServiceAccountToken controls whether Pods using
		// this SA automatically get a token mounted. Setting to false
		// is a security best practice — only mount tokens when needed.
		AutomountServiceAccountToken: ptr.To(false),
	}

	createdSA, err := clientset.CoreV1().ServiceAccounts(namespace).Create(
		ctx, sa, metav1.CreateOptions{},
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error creating ServiceAccount: %v\n", err)
		os.Exit(1)
	}
	fmt.Printf("Created ServiceAccount: %s/%s\n",
		createdSA.Namespace, createdSA.Name)

	// Step 2: Request a bound token using the TokenRequest API.
	// This token is:
	//   - Audience-bound: only valid for the specified audiences
	//   - Time-bound: expires after expirationSeconds
	//   - Not stored: generated on-the-fly, not persisted in etcd
	tokenRequest := &authenticationv1.TokenRequest{
		Spec: authenticationv1.TokenRequestSpec{
			// Audiences specifies the intended recipients of the token.
			// When this token is presented to a service, that service
			// should verify its audience matches before accepting it.
			Audiences: []string{
				"https://my-service.example.com",
				"https://kubernetes.default.svc",
			},
			// ExpirationSeconds sets the token lifetime. The minimum
			// allowed by the API server is 600 seconds (10 minutes).
			// The maximum depends on --service-account-max-token-expiration.
			ExpirationSeconds: ptr.To(int64(3600)), // 1 hour
		},
	}

	// CreateToken issues the token through the TokenRequest API.
	// This requires the "create" verb on "serviceaccounts/token".
	tokenResponse, err := clientset.CoreV1().ServiceAccounts(namespace).
	CreateToken(ctx, createdSA.Name, tokenRequest, metav1.CreateOptions{})
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error creating token: %v\n", err)
		os.Exit(1)
	}

	token := tokenResponse.Status.Token
	expiresAt := tokenResponse.Status.ExpirationTimestamp.Time

	fmt.Printf("Token created successfully\n")
	fmt.Printf("  Expires at: %s\n", expiresAt.Format(time.RFC3339))
	fmt.Printf("  Token (first 20 chars): %s...\n", token[:20])

	// Step 3: Decode and inspect the JWT claims (without verification).
	// In production, always verify the signature. Here we just decode
	// for inspection purposes.
	inspectJWT(token)

	// Step 4: Use the token to create a new client that authenticates
	// as the ServiceAccount. This simulates what an external system
	// would do with the token.
	saConfig := *config // Copy the config
	saConfig.BearerToken = token
	// Clear any certificate-based auth from the original config
	// so we authenticate purely with the token.
	saConfig.CertData = nil
	saConfig.CertFile = ""
	saConfig.KeyData = nil
	saConfig.KeyFile = ""

	saClientset, err := kubernetes.NewForConfig(&saConfig)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error creating SA clientset: %v\n", err)
		os.Exit(1)
	}

	// Attempt to list pods — this will fail unless we've granted
	// the SA appropriate RBAC. The error message is informative.
	_, err = saClientset.CoreV1().Pods(namespace).List(
		ctx, metav1.ListOptions{},
	)
	if err != nil {
		fmt.Printf("\nExpected RBAC error (SA has no permissions): %v\n", err)
	} else {
		fmt.Printf("\nSuccessfully listed pods as the ServiceAccount\n")
	}

	// Step 5: Clean up — delete the ServiceAccount.
	// This also invalidates any tokens issued for it.
	err = clientset.CoreV1().ServiceAccounts(namespace).Delete(
		ctx, createdSA.Name, metav1.DeleteOptions{},
	)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error deleting ServiceAccount: %v\n", err)
	} else {
		fmt.Println("\nCleaned up ServiceAccount")
	}
}

// inspectJWT decodes a JWT token without verifying its signature
// and prints the claims for inspection. This is useful for debugging
// but MUST NOT be used for authentication decisions — always verify
// the signature in production code.
func inspectJWT(token string) {
	// JWTs have three parts separated by dots: header.payload.signature
	// We decode the payload (second part) to see the claims.
	parts := strings.Split(token, ".")
	if len(parts) != 3 {
		fmt.Println("  Invalid JWT format")
		return
	}

	// The payload is base64url-encoded (no padding). We need to add
	// padding back before decoding.
	payload := parts[1]
	switch len(payload) % 4 {
	case 2:
		payload += "=="
	case 3:
		payload += "="
	}

	decoded, err := base64.URLEncoding.DecodeString(payload)
	if err != nil {
		fmt.Printf("  Error decoding JWT payload: %v\n", err)
		return
	}

	// Pretty-print the claims
	var claims map[string]interface{}
	if err := json.Unmarshal(decoded, &claims); err != nil {
		fmt.Printf("  Error parsing JWT claims: %v\n", err)
		return
	}

	fmt.Println("\nJWT Claims:")
	prettyJSON, _ := json.MarshalIndent(claims, "  ", "  ")
	fmt.Printf("  %s\n", prettyJSON)
}
```

---

## 29.11 Advanced Topics

### 29.11.1 Token Projection for Non-Kubernetes Services

Projected tokens can target non-Kubernetes audiences, enabling workload identity
federation:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: vault-client
spec:
  serviceAccountName: vault-sa
  containers:
    - name: app
      image: my-app:latest
      volumeMounts:
        - name: vault-token
          mountPath: /var/run/secrets/vault
          readOnly: true
        - name: aws-token
          mountPath: /var/run/secrets/aws
          readOnly: true
  volumes:
    # Token for authenticating to HashiCorp Vault
    - name: vault-token
      projected:
        sources:
          - serviceAccountToken:
              audience: "vault"
              expirationSeconds: 600
              path: token
    # Token for AWS IAM Roles for Service Accounts (IRSA)
    - name: aws-token
      projected:
        sources:
          - serviceAccountToken:
              audience: "sts.amazonaws.com"
              expirationSeconds: 86400
              path: token
```

### 29.11.2 Workload Identity Federation

Cloud providers use Kubernetes SA tokens for cloud IAM:

**AWS EKS (IRSA):**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/s3-reader-role
```

**GKE Workload Identity:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gcs-reader
  annotations:
    iam.gke.io/gcp-service-account: gcs-reader@project.iam.gserviceaccount.com
```

**Azure Workload Identity:**
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: blob-reader
  annotations:
    azure.workload.identity/client-id: <azure-ad-app-client-id>
  labels:
    azure.workload.identity/use: "true"
```

### 29.11.3 Service Account Token Volume Projection — Disabling Auto-Mount

A security best practice is to disable automatic token mounting for ServiceAccounts
that don't need API access:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: no-api-access
automountServiceAccountToken: false
---
apiVersion: v1
kind: Pod
metadata:
  name: web-server
spec:
  serviceAccountName: no-api-access
  # Pod-level override (takes precedence over SA-level setting)
  automountServiceAccountToken: false
  containers:
    - name: nginx
      image: nginx:latest
```

**Why this matters:** If a Pod is compromised and a token is mounted, the attacker
can use it to probe the API server. Even with minimal RBAC, this reveals information
about the cluster (API server version, available APIs, etc.).

---

## 29.12 Authentication Security Best Practices

### 29.12.1 Do's

1. **Use OIDC for humans** — centralized identity, MFA support, token expiry
2. **Use projected SA tokens for workloads** — bound, expiring, audience-scoped
3. **Disable automountServiceAccountToken** where not needed
4. **Use short-lived certificates** (hours, not years) with automated rotation
5. **Prefix OIDC usernames/groups** to avoid collisions with system identities
6. **Audit authentication events** — enable audit logging for authentication
7. **Protect the CA key** — consider offline storage after bootstrap
8. **Use credential plugins** (exec-based) instead of static tokens in kubeconfig

### 29.12.2 Don'ts

1. **Don't use static token files** — they're deprecated and insecure
2. **Don't create legacy SA token secrets** — use TokenRequest API instead
3. **Don't share kubeconfigs** with embedded credentials
4. **Don't grant impersonation rights broadly** — always use `resourceNames`
5. **Don't skip TLS verification** — `insecure-skip-tls-verify` disables all security
6. **Don't use `system:masters` group** for regular users — it bypasses all RBAC
7. **Don't expose the API server publicly** without additional authentication layers

### 29.12.3 Monitoring Authentication

```bash
# Check who is authenticating and how
kubectl get events --field-selector reason=Forbidden -A

# Audit logs show authentication details
# Enable in the API server:
# --audit-policy-file=/etc/kubernetes/audit-policy.yaml
# --audit-log-path=/var/log/kubernetes/audit.log
```

Example audit policy for authentication events:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log all authentication failures at RequestResponse level
  - level: RequestResponse
    users: ["system:anonymous"]
    verbs: ["*"]
  # Log TokenReview requests (SA token validation)
  - level: Metadata
    resources:
      - group: "authentication.k8s.io"
        resources: ["tokenreviews"]
  # Log certificate signing requests
  - level: Metadata
    resources:
      - group: "certificates.k8s.io"
        resources: ["certificatesigningrequests"]
```

---

## 29.13 Troubleshooting Authentication

### Common Issues

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| `error: You must be logged in to the server (Unauthorized)` | Invalid/expired token or certificate | Check token expiry, regenerate credentials |
| `error: x509: certificate signed by unknown authority` | kubeconfig CA doesn't match cluster CA | Update CA in kubeconfig |
| `error: x509: certificate has expired` | Client certificate expired | Reissue certificate |
| OIDC: `error: oidc: verify token: oidc: id token issued by different provider` | Mismatched issuer URL | Verify `--oidc-issuer-url` matches token's `iss` claim |
| SA token: `error: unauthorized` | Token audience mismatch | Check `--api-audiences` flag on API server |

### Debugging Commands

```bash
# Check the current user identity
kubectl auth whoami  # Kubernetes 1.27+

# Check if a user can perform an action
kubectl auth can-i create pods --as=alice

# Decode a JWT token
kubectl get secret <sa-secret> -o jsonpath='{.data.token}' | \
  base64 -d | jwt decode -

# Check API server authentication configuration
kubectl -n kube-system get cm kubeadm-config -o yaml

# Test ServiceAccount token
TOKEN=$(kubectl create token my-sa)
curl -H "Authorization: Bearer $TOKEN" \
  https://kubernetes.default.svc/api/v1/namespaces/default/pods \
  --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
```

---

## 29.14 Summary

Authentication in Kubernetes is a layered, pluggable system:

- **X.509 certificates** are the foundation — used by all control plane components
  and available for user authentication, but inflexible for large organizations
- **ServiceAccount tokens** have evolved from insecure long-lived secrets to
  short-lived, bound, audience-scoped projected tokens
- **OIDC** is the recommended method for human authentication, integrating with
  any standards-compliant identity provider
- **Webhook and proxy authentication** enable custom integrations
- **Impersonation** allows privileged users to act as others, but must be
  tightly controlled

The key insight is that authentication only answers "Who are you?" — it says nothing
about what you're allowed to do. That's the domain of authorization (RBAC), which
we covered in the previous chapter. Authentication and authorization work together:
a strong authentication system is meaningless if RBAC is misconfigured, and
tight RBAC is useless if an attacker can forge identities.

In the next chapter, we'll explore secrets management — how to securely store and
distribute the credentials that your workloads need.

---

## Exercises

1. **Certificate Deep Dive:** Create a user certificate with multiple Organization
   fields. Verify that the API server correctly maps them to groups. Write RBAC
   rules that distinguish between the groups.

2. **Token Comparison:** Create a legacy SA token (Secret-based) and a projected
   token (TokenRequest API). Decode both JWTs and compare their claims. Which has
   more security metadata?

3. **OIDC Integration:** Set up Dex with a GitHub connector. Configure the API
   server to use it. Verify that GitHub organization memberships map to Kubernetes
   groups.

4. **Token Audience Scoping:** Create two projected token volumes in the same Pod
   with different audiences. Write a Go program that reads both tokens and verifies
   they have different audience claims.

5. **Impersonation Audit:** Enable audit logging, then perform impersonation.
   Examine the audit logs to see how both the original user and the impersonated
   user are recorded.

6. **Security Hardening:** Take a default kubeadm cluster and harden its
   authentication: disable anonymous auth, disable static token files, enable
   OIDC, restrict ServiceAccount token audiences.
