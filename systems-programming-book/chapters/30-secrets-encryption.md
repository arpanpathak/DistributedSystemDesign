# Chapter 30: Secrets Management & Encryption at Rest

## Introduction

Kubernetes Secrets are perhaps the most misunderstood resource in the entire platform.
Developers assume that because the resource is called "Secret," it must be encrypted
and secure. The reality is far more nuanced — and far more dangerous if you don't
understand it.

By default, Kubernetes Secrets are stored as **base64-encoded plaintext** in etcd.
Base64 is **not encryption** — it's an encoding that anyone can reverse. If an
attacker gains read access to etcd (through a backup, a compromised node, or an
API server misconfiguration), they can read every secret in the cluster: database
passwords, API keys, TLS certificates, and OAuth tokens.

This chapter covers the entire spectrum of secrets management in Kubernetes:
from the built-in encryption-at-rest mechanism to external secrets management
systems like HashiCorp Vault, External Secrets Operator, Sealed Secrets, and SOPS.
We'll write Go programs that interact with Vault, implement secret rotation
patterns, and explore RBAC strategies for controlling secret access.

**What you will learn:**

- How Kubernetes stores secrets and why the default is insecure
- Encrypting secrets at rest with EncryptionConfiguration
- Integrating with external secret stores (Vault, AWS Secrets Manager, etc.)
- GitOps-safe secrets with Sealed Secrets and SOPS
- Secret rotation patterns and automation
- RBAC strategies for secret access control
- Go programs for secret management

---

## 30.1 How Kubernetes Stores Secrets

### 30.1.1 The Secret Resource

A Kubernetes Secret is a namespaced resource that holds key-value pairs:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
  namespace: production
type: Opaque
data:
  # Values are base64-encoded
  username: YWRtaW4=          # admin
  password: cDRzc3cwcmQ=      # p4ssw0rd
  connection-string: cG9zdGdyZXM6Ly9hZG1pbjpwNHNzdzByZEBkYi5leGFtcGxlLmNvbTo1NDMyL215ZGI=
```

Or using `stringData` (plaintext input, auto-encoded to base64):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
type: Opaque
stringData:
  username: admin
  password: p4ssw0rd
  connection-string: "postgres://admin:p4ssw0rd@db.example.com:5432/mydb"
```

### 30.1.2 Secret Types

| Type | Description |
|------|-------------|
| `Opaque` | Arbitrary user-defined data (default) |
| `kubernetes.io/service-account-token` | ServiceAccount token |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials |
| `kubernetes.io/tls` | TLS certificate and key |
| `kubernetes.io/basic-auth` | Basic authentication credentials |
| `kubernetes.io/ssh-auth` | SSH private key |
| `bootstrap.kubernetes.io/token` | Bootstrap token |

### 30.1.3 How Secrets Are Stored in etcd

Let's look at what's actually in etcd:

```bash
# Read a secret directly from etcd (requires etcd access)
ETCDCTL_API=3 etcdctl get /registry/secrets/default/database-credentials \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**Without encryption at rest**, the output is a Protocol Buffers-encoded object
that contains the secret values in plaintext. You can see the actual password
bytes in the raw output.

```
/registry/secrets/default/database-credentials
k8s

v1Secret...
database-credentials...default...
username...admin
password...p4ssw0rd    <---- PLAINTEXT IN ETCD
```

**Attack vectors for unencrypted secrets:**

1. **etcd data directory:** Anyone with access to `/var/lib/etcd/` on an etcd
   node can read all secrets
2. **etcd backups:** If etcd snapshots are stored without encryption, secrets
   are exposed in the backup files
3. **etcd API:** If the etcd API is accessible (misconfigured firewall or
   compromised etcd peer certificate), all secrets can be read
4. **Physical disk:** If the etcd disk is not encrypted (no LUKS/dm-crypt),
   secrets survive disk decommissioning

### 30.1.4 Secret Access Through the API

Even through the Kubernetes API, secrets have limited protection:

```bash
# Anyone with "get secrets" permission can read secrets
kubectl get secret database-credentials -o jsonpath='{.data.password}' | base64 -d
# Output: p4ssw0rd

# List all secrets in a namespace
kubectl get secrets -n production

# Watch for secret changes (useful for attackers too)
kubectl get secrets -w -n production
```

**RBAC is the only protection** for secrets accessed through the API. We'll cover
RBAC for secrets in detail in section 30.8.

---

## 30.2 Encryption at Rest — EncryptionConfiguration

Kubernetes supports encrypting secret data before writing it to etcd. This is
configured through an `EncryptionConfiguration` resource.

### 30.2.1 The EncryptionConfiguration Resource

```yaml
# /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
      - configmaps  # You can encrypt ConfigMaps too
    providers:
      # Providers are tried in order for WRITING.
      # ALL providers are tried for READING (to support rotation).

      # Primary provider — used for all new writes
      - aescbc:
          keys:
            - name: key-2024-01
              secret: <base64-encoded-32-byte-key>

      # Previous key — still used for reading during rotation
      - aescbc:
          keys:
            - name: key-2023-06
              secret: <base64-encoded-32-byte-key>

      # Identity provider — allows reading unencrypted data
      # written before encryption was enabled
      - identity: {}
```

### 30.2.2 Encryption Providers

#### identity (No Encryption)

```yaml
providers:
  - identity: {}
```

The default. Data is stored as-is (base64-encoded Protocol Buffers). Useful as a
fallback for reading pre-encryption data.

#### aescbc (AES-CBC)

```yaml
providers:
  - aescbc:
      keys:
        - name: key1
          secret: <base64-encoded-32-byte-key>
```

- **Algorithm:** AES-256-CBC with PKCS#7 padding
- **Key size:** 32 bytes (256 bits)
- **IV:** Random, prepended to ciphertext
- **Strength:** Strong, widely vetted
- **Performance:** Moderate — each encryption/decryption is a crypto operation
- **Vulnerability:** Susceptible to padding oracle attacks if the attacker can
  trigger decryption with arbitrary ciphertext (unlikely in the etcd context)

#### aesgcm (AES-GCM)

```yaml
providers:
  - aesgcm:
      keys:
        - name: key1
          secret: <base64-encoded-16-or-32-byte-key>
```

- **Algorithm:** AES-GCM (Galois/Counter Mode)
- **Key size:** 16 or 32 bytes (128 or 256 bits)
- **Strength:** Provides both encryption AND authentication (integrity check)
- **CRITICAL WARNING:** AES-GCM nonce reuse is catastrophic. If the same nonce
  is used with the same key, it reveals the XOR of two plaintexts AND the
  authentication key. Kubernetes generates random nonces, which is safe for
  moderate volumes, but **key rotation is essential** — rotate before 2^32 writes.
- **Recommendation:** Use `aescbc` or `secretbox` unless you have a specific
  reason for GCM and a strict key rotation policy.

#### secretbox (NaCl Secretbox)

```yaml
providers:
  - secretbox:
      keys:
        - name: key1
          secret: <base64-encoded-32-byte-key>
```

- **Algorithm:** XSalsa20-Poly1305
- **Key size:** 32 bytes
- **Strength:** Authenticated encryption, modern design
- **Performance:** Fast — faster than AES-CBC
- **Recommendation:** Best choice for local encryption if KMS is not available

#### kms v1 (Deprecated)

```yaml
providers:
  - kms:
      name: my-kms
      endpoint: unix:///var/run/kms/kms.sock
      cachesize: 1000
      timeout: 3s
```

Uses an external Key Management Service via gRPC. The KMS holds the master key
(Key Encryption Key, KEK) and Kubernetes generates Data Encryption Keys (DEKs)
that are encrypted by the KEK before storage.

**Deprecated in 1.28+.** Use KMS v2 instead.

#### kms v2 (Recommended for Production)

```yaml
providers:
  - kms:
      apiVersion: v2
      name: my-kms
      endpoint: unix:///var/run/kms/kms.sock
      timeout: 3s
```

KMS v2 improvements over v1:
- **Performance:** Generates a single DEK per API server restart, encrypted once
  by the KEK. Uses that DEK for all subsequent encryptions (with per-object nonces).
- **Key rotation detection:** The API server detects KEK rotation without restart.
- **Health checking:** Built-in health endpoint for monitoring KMS availability.
- **Reduced KMS calls:** Dramatically fewer calls to the KMS (one at startup vs.
  one per secret write in v1).

### 30.2.3 Hands-On: Setting Up Encryption at Rest

```bash
# Step 1: Generate a strong encryption key
KEY=$(head -c 32 /dev/urandom | base64)
echo "Generated key: $KEY"

# Step 2: Create the encryption configuration
cat > /etc/kubernetes/encryption-config.yaml <<EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key-$(date +%Y%m%d)
              secret: ${KEY}
      - identity: {}
EOF

# Step 3: Set restrictive permissions on the config file
chmod 600 /etc/kubernetes/encryption-config.yaml

# Step 4: Configure the API server to use it
# Edit /etc/kubernetes/manifests/kube-apiserver.yaml
# Add the flag:
#   --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
# Add a volume mount for the encryption config file

# Step 5: Wait for the API server to restart (it's a static Pod)
kubectl wait --for=condition=Ready pod/kube-apiserver-<node> \
  -n kube-system --timeout=120s

# Step 6: Verify encryption is working
# Create a test secret
kubectl create secret generic test-encryption \
  --from-literal=mykey=myvalue

# Read it directly from etcd — it should be encrypted
ETCDCTL_API=3 etcdctl get /registry/secrets/default/test-encryption \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key | hexdump -C | head

# You should see "k8s:enc:aescbc:v1:key-YYYYMMDD" prefix
# followed by encrypted binary data — NOT plaintext

# Step 7: Re-encrypt all existing secrets
# Existing secrets were written before encryption was enabled,
# so they're still stored in plaintext. This command rewrites them:
kubectl get secrets --all-namespaces -o json | \
  kubectl replace -f -
```

### 30.2.4 Key Rotation Process

Key rotation is essential and must be done without downtime:

```bash
# Step 1: Generate a new key
NEW_KEY=$(head -c 32 /dev/urandom | base64)

# Step 2: Add the new key as the FIRST provider (for writes)
# Keep the old key as a SECOND provider (for reading existing data)
cat > /etc/kubernetes/encryption-config.yaml <<EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      # New key — all new writes use this
      - aescbc:
          keys:
            - name: key-$(date +%Y%m%d)-rotated
              secret: ${NEW_KEY}
      # Old key — needed to read existing data
      - aescbc:
          keys:
            - name: key-previous
              secret: ${OLD_KEY}
      - identity: {}
EOF

# Step 3: Restart the API server to pick up the new config
# (For static Pods, touching the manifest triggers a restart)

# Step 4: Re-encrypt ALL secrets with the new key
kubectl get secrets --all-namespaces -o json | \
  kubectl replace -f -

# Step 5: Once all secrets are re-encrypted, remove the old key
# Update the config to only have the new key + identity fallback
cat > /etc/kubernetes/encryption-config.yaml <<EOF
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key-$(date +%Y%m%d)-rotated
              secret: ${NEW_KEY}
      - identity: {}
EOF

# Step 6: Restart API server again
```

**Multi-API-server considerations:** In HA clusters with multiple API servers,
you must update the encryption config on **all** API servers before re-encrypting
secrets. Otherwise, an API server with the old config won't be able to read
secrets encrypted with the new key.

### 30.2.5 KMS v2 with Cloud Providers

#### AWS KMS

```go
// Package main implements a KMS v2 plugin for AWS KMS.
// This plugin runs as a gRPC server that the Kubernetes API server
// connects to via a Unix socket. It delegates all key management
// operations to AWS KMS, ensuring that the master encryption key
// (KEK) never leaves AWS.
//
// Architecture:
//   API Server  ──unix socket──▶  KMS Plugin  ──HTTPS──▶  AWS KMS
//
// The plugin handles:
//   - EncryptRequest: Encrypts a DEK using the AWS KMS CMK
//   - DecryptRequest: Decrypts a DEK using the AWS KMS CMK
//   - StatusRequest: Reports health and key ID
package main

import (
	"context"
	"fmt"
	"log"
	"net"
	"os"
	"os/signal"
	"syscall"

	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/kms"
	"google.golang.org/grpc"
	kmsapi "k8s.io/kms/apis/v2"
)

// awsKMSPlugin implements the KMS v2 gRPC service using AWS KMS
// as the backend. It maintains a persistent connection to AWS KMS
// and caches the CMK ARN for all operations.
type awsKMSPlugin struct {
	kmsapi.UnimplementedKeyManagementServiceServer

	// kmsClient is the AWS KMS client used for encrypt/decrypt
	// operations. It handles credential refresh, retry logic,
	// and connection pooling automatically.
	kmsClient *kms.Client

	// keyID is the ARN or alias of the AWS KMS Customer Master Key
	// (CMK) used for encrypting/decrypting Data Encryption Keys.
	// Example: "arn:aws:kms:us-east-1:123456789012:key/mrk-xxxx"
	// or "alias/kubernetes-encryption"
	keyID string
}

// Status returns the current health and key information.
// The API server calls this periodically to verify the KMS
// plugin is healthy and to detect key rotation.
func (p *awsKMSPlugin) Status(
	ctx context.Context,
	req *kmsapi.StatusRequest,
) (*kmsapi.StatusResponse, error) {
	// Perform a lightweight operation to verify KMS connectivity.
	// DescribeKey is cheap and confirms the key exists and is enabled.
	output, err := p.kmsClient.DescribeKey(ctx, &kms.DescribeKeyInput{
			KeyId: &p.keyID,
		})
	if err != nil {
		return nil, fmt.Errorf("KMS health check failed: %w", err)
	}

	return &kmsapi.StatusResponse{
		Version: "v2",
		Healthz: "ok",
		KeyId:   *output.KeyMetadata.KeyId,
	}, nil
}

// Encrypt takes a plaintext DEK and encrypts it using the AWS KMS CMK.
// The API server calls this when it needs to encrypt a new DEK.
// In KMS v2, this happens once per API server restart (not per-secret).
func (p *awsKMSPlugin) Encrypt(
	ctx context.Context,
	req *kmsapi.EncryptRequest,
) (*kmsapi.EncryptResponse, error) {
	// Call AWS KMS to encrypt the plaintext DEK with the CMK.
	// AWS KMS uses envelope encryption: the CMK encrypts the DEK,
	// and the DEK encrypts the actual secret data.
	output, err := p.kmsClient.Encrypt(ctx, &kms.EncryptInput{
			KeyId:     &p.keyID,
			Plaintext: req.Plaintext,
		})
	if err != nil {
		return nil, fmt.Errorf("encryption failed: %w", err)
	}

	return &kmsapi.EncryptResponse{
		Ciphertext:  output.CiphertextBlob,
		KeyId:       *output.KeyId,
		Annotations: map[string][]byte{},
	}, nil
}

// Decrypt takes an encrypted DEK and decrypts it using the AWS KMS CMK.
// The API server calls this when it needs to read a secret and must
// first decrypt the DEK that was used to encrypt it.
func (p *awsKMSPlugin) Decrypt(
	ctx context.Context,
	req *kmsapi.DecryptRequest,
) (*kmsapi.DecryptResponse, error) {
	output, err := p.kmsClient.Decrypt(ctx, &kms.DecryptInput{
			KeyId:          &p.keyID,
			CiphertextBlob: req.Ciphertext,
		})
	if err != nil {
		return nil, fmt.Errorf("decryption failed: %w", err)
	}

	return &kmsapi.DecryptResponse{
		Plaintext: output.Plaintext,
	}, nil
}

func main() {
	// Load the KMS key ARN from environment or configuration.
	keyID := os.Getenv("AWS_KMS_KEY_ARN")
	if keyID == "" {
		log.Fatal("AWS_KMS_KEY_ARN environment variable is required")
	}

	// Create the AWS KMS client. The SDK automatically handles
	// credentials from environment variables, IAM roles, or
	// the EC2 instance metadata service.
	cfg, err := config.LoadDefaultConfig(context.Background())
	if err != nil {
		log.Fatalf("Unable to load AWS config: %v", err)
	}

	socketPath := "/var/run/kms/kms.sock"

	// Remove any stale socket file from a previous run.
	os.Remove(socketPath)

	// Create the Unix socket listener. The API server connects
	// to this socket to communicate with the KMS plugin.
	listener, err := net.Listen("unix", socketPath)
	if err != nil {
		log.Fatalf("Failed to listen on %s: %v", socketPath, err)
	}
	defer listener.Close()

	// Set socket permissions so the API server can connect.
	os.Chmod(socketPath, 0600)

	plugin := &awsKMSPlugin{
		kmsClient: kms.NewFromConfig(cfg),
		keyID:     keyID,
	}

	server := grpc.NewServer()
	kmsapi.RegisterKeyManagementServiceServer(server, plugin)

	// Handle graceful shutdown on SIGTERM/SIGINT.
	go func() {
		sigCh := make(chan os.Signal, 1)
		signal.Notify(sigCh, syscall.SIGTERM, syscall.SIGINT)
		<-sigCh
		log.Println("Shutting down KMS plugin...")
		server.GracefulStop()
	}()

	log.Printf("KMS v2 plugin listening on %s (key: %s)", socketPath, keyID)
	if err := server.Serve(listener); err != nil {
		log.Fatalf("gRPC server failed: %v", err)
	}
}
```

---

## 30.3 External Secrets Operator

The External Secrets Operator (ESO) synchronizes secrets from external secret
management systems into Kubernetes Secrets. This keeps the source of truth outside
the cluster while making secrets available to Pods.

### 30.3.1 Architecture

```
┌─────────────────────┐     ┌──────────────────────┐
│ External Secret      │     │ External Secret Store │
│ Store                │     │ (AWS SM, Vault, etc.) │
│                      │     │                       │
│  ┌─────────────┐    │     │  secret/db/password   │
│  │ SecretStore  │────┼─────│  secret/api/key       │
│  │ (connection) │    │     │  secret/tls/cert      │
│  └─────────────┘    │     └──────────────────────┘
│                      │
│  ┌──────────────┐   │     ┌──────────────────────┐
│  │ExternalSecret│───┼────▶│  Kubernetes Secret    │
│  │(what to sync)│   │     │  (auto-created)       │
│  └──────────────┘   │     └──────────────────────┘
└─────────────────────┘
```

### 30.3.2 Installation

```bash
# Install External Secrets Operator
helm repo add external-secrets https://charts.external-secrets.io
helm install external-secrets external-secrets/external-secrets \
  -n external-secrets --create-namespace \
  --set installCRDs=true
```

### 30.3.3 SecretStore and ClusterSecretStore

```yaml
# Namespace-scoped SecretStore — for team-specific secrets
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa
---
# Cluster-scoped — available to all namespaces
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "https://vault.example.com:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
```

### 30.3.4 ExternalSecret — Defining What to Sync

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-creds
  namespace: production
spec:
  refreshInterval: 1h  # How often to sync from the external store
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore

  target:
    name: database-credentials  # Name of the K8s Secret to create
    creationPolicy: Owner       # ESO owns and manages this Secret
    deletionPolicy: Retain      # Keep Secret if ExternalSecret is deleted
    template:
      type: Opaque
      metadata:
        labels:
          app: myapp
        annotations:
          reloader.stakater.com/match: "true"

  data:
    # Map external secret paths to Kubernetes Secret keys
    - secretKey: username
      remoteRef:
        key: production/database
        property: username

    - secretKey: password
      remoteRef:
        key: production/database
        property: password

    - secretKey: connection-string
      remoteRef:
        key: production/database
        property: connection_string

  # Or use dataFrom to sync all keys from a single external secret
  dataFrom:
    - extract:
        key: production/api-keys
```

### 30.3.5 Hands-On: ESO with AWS Secrets Manager

```bash
# Step 1: Create a secret in AWS Secrets Manager
aws secretsmanager create-secret \
  --name production/database \
  --secret-string '{"username":"admin","password":"s3cure-p@ss","host":"db.example.com"}'

# Step 2: Create an IAM role for the External Secrets SA
# (Using IRSA — IAM Roles for Service Accounts)
cat > trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "Federated": "arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE"
    },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "oidc.eks.us-east-1.amazonaws.com/id/EXAMPLE:sub": "system:serviceaccount:external-secrets:external-secrets-sa"
      }
    }
  }]
}
EOF

aws iam create-role \
  --role-name ExternalSecretsRole \
  --assume-role-policy-document file://trust-policy.json

aws iam put-role-policy \
  --role-name ExternalSecretsRole \
  --policy-name SecretsManagerRead \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": [
        "secretsmanager:GetSecretValue",
        "secretsmanager:DescribeSecret"
      ],
      "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:production/*"
    }]
  }'

# Step 3: Create annotated ServiceAccount
kubectl create sa external-secrets-sa -n external-secrets
kubectl annotate sa external-secrets-sa -n external-secrets \
  eks.amazonaws.com/role-arn=arn:aws:iam::123456789012:role/ExternalSecretsRole

# Step 4: Apply SecretStore and ExternalSecret
kubectl apply -f secret-store.yaml
kubectl apply -f external-secret.yaml

# Step 5: Verify the secret was created
kubectl get secret database-credentials -n production -o yaml
kubectl get externalsecret database-creds -n production
# STATUS should show "SecretSynced"
```

### 30.3.6 Hands-On: ESO with HashiCorp Vault

```bash
# Step 1: Enable Kubernetes auth in Vault
vault auth enable kubernetes

vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc:443"

# Step 2: Create a Vault policy
vault policy write external-secrets - <<EOF
path "secret/data/production/*" {
  capabilities = ["read"]
}
EOF

# Step 3: Create a Vault role linked to the ESO ServiceAccount
vault write auth/kubernetes/role/external-secrets \
  bound_service_account_names=external-secrets-sa \
  bound_service_account_namespaces=external-secrets \
  policies=external-secrets \
  ttl=1h

# Step 4: Store a secret in Vault
vault kv put secret/production/database \
  username=admin \
  password=vault-managed-password \
  host=db.internal

# Step 5: Create the ClusterSecretStore for Vault
cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: vault
spec:
  provider:
    vault:
      server: "http://vault.vault-system.svc:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "external-secrets"
          serviceAccountRef:
            name: external-secrets-sa
            namespace: external-secrets
EOF

# Step 6: Create ExternalSecret pointing to Vault
cat <<EOF | kubectl apply -f -
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: vault-db-creds
  namespace: production
spec:
  refreshInterval: 30m
  secretStoreRef:
    name: vault
    kind: ClusterSecretStore
  target:
    name: db-credentials
  data:
    - secretKey: username
      remoteRef:
        key: production/database
        property: username
    - secretKey: password
      remoteRef:
        key: production/database
        property: password
EOF
```

---

## 30.4 HashiCorp Vault Integration

Vault is the most feature-rich secrets management system commonly used with
Kubernetes. There are three primary integration patterns.

### 30.4.1 Vault Agent Injector (Sidecar Pattern)

The Vault Agent Injector uses a mutating webhook to inject a Vault Agent sidecar
into Pods:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    metadata:
      annotations:
        # These annotations trigger the Vault Agent Injector
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "myapp"
        vault.hashicorp.com/agent-inject-secret-db-creds: "secret/data/production/database"
        vault.hashicorp.com/agent-inject-template-db-creds: |
          {{- with secret "secret/data/production/database" -}}
          export DB_USER="{{ .Data.data.username }}"
          export DB_PASS="{{ .Data.data.password }}"
          export DB_HOST="{{ .Data.data.host }}"
          {{- end }}
    spec:
      serviceAccountName: myapp-sa
      containers:
        - name: myapp
          image: myapp:latest
          command: ["/bin/sh", "-c"]
          args:
            - source /vault/secrets/db-creds && exec /app/myapp
```

**How it works:**

1. Pod is created with Vault annotations
2. Mutating webhook intercepts the Pod creation
3. Vault Agent init container authenticates to Vault using the SA token
4. Vault Agent sidecar continuously renders secrets to shared volumes
5. The application reads secrets from `/vault/secrets/`

**Pros:**
- Application doesn't need to know about Vault
- Automatic token renewal and secret refresh
- Template support for complex secret formats

**Cons:**
- Extra sidecar container per Pod (resource overhead)
- Init container delays Pod startup
- Secrets are written to files (tmpfs, but still on the filesystem)

### 30.4.2 CSI Secret Store Driver

The Secrets Store CSI Driver mounts secrets as volumes without sidecars:

```bash
# Install CSI Driver
helm repo add secrets-store-csi-driver \
  https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
helm install csi-secrets-store \
  secrets-store-csi-driver/secrets-store-csi-driver \
  -n kube-system \
  --set syncSecret.enabled=true

# Install Vault CSI provider
helm install vault hashicorp/vault \
  --set "csi.enabled=true" \
  --set "injector.enabled=false" \
  --set "server.enabled=false"
```

```yaml
# SecretProviderClass defines what secrets to mount
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: vault-db-creds
spec:
  provider: vault
  parameters:
    vaultAddress: "http://vault.vault-system:8200"
    roleName: "myapp"
    objects: |
      - objectName: "db-username"
        secretPath: "secret/data/production/database"
        secretKey: "username"
      - objectName: "db-password"
        secretPath: "secret/data/production/database"
        secretKey: "password"
  # Optionally sync to a Kubernetes Secret
  secretObjects:
    - secretName: db-credentials-synced
      type: Opaque
      data:
        - objectName: db-username
          key: username
        - objectName: db-password
          key: password
---
apiVersion: v1
kind: Pod
metadata:
  name: myapp
spec:
  serviceAccountName: myapp-sa
  containers:
    - name: myapp
      image: myapp:latest
      volumeMounts:
        - name: secrets
          mountPath: "/mnt/secrets"
          readOnly: true
  volumes:
    - name: secrets
      csi:
        driver: secrets-store.csi.k8s.io
        readOnly: true
        volumeAttributes:
          secretProviderClass: vault-db-creds
```

### 30.4.3 Vault Secrets Operator

The newest approach — a Kubernetes-native operator that syncs Vault secrets:

```yaml
# VaultAuth — how to authenticate with Vault
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultAuth
metadata:
  name: vault-auth
  namespace: production
spec:
  method: kubernetes
  mount: kubernetes
  kubernetes:
    role: myapp
    serviceAccount: default
    audiences:
      - vault
---
# VaultStaticSecret — sync a static secret
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultStaticSecret
metadata:
  name: db-creds
  namespace: production
spec:
  type: kv-v2
  mount: secret
  path: production/database
  destination:
    name: db-credentials
    create: true
  refreshAfter: 30s
  vaultAuthRef: vault-auth
---
# VaultDynamicSecret — generate dynamic credentials
apiVersion: secrets.hashicorp.com/v1beta1
kind: VaultDynamicSecret
metadata:
  name: postgres-creds
  namespace: production
spec:
  mount: database
  path: creds/myapp-role
  destination:
    name: dynamic-db-credentials
    create: true
  renewalPercent: 67
  vaultAuthRef: vault-auth
```

### 30.4.4 Hands-On: Go Program That Reads Secrets from Vault

```go
// Package main demonstrates multiple patterns for reading secrets
// from HashiCorp Vault in a Kubernetes environment. It shows:
//
//   1. Kubernetes auth method — using a projected ServiceAccount token
//   2. Reading static KV secrets
//   3. Reading dynamic database credentials
//   4. Automatic token renewal
//
// This program is designed to run inside a Kubernetes Pod where
// a projected ServiceAccount token is available at the well-known
// path. The token is used to authenticate with Vault's Kubernetes
// auth method, which verifies the token with the Kubernetes
// TokenReview API.
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"time"

	vault "github.com/hashicorp/vault/api"
	auth "github.com/hashicorp/vault/api/auth/kubernetes"
)

// VaultSecretReader encapsulates Vault client operations and
// provides methods for common secret reading patterns. It
// manages authentication, token lifecycle, and retries.
type VaultSecretReader struct {
	// client is the Vault API client. It handles HTTP
	// communication, TLS, and token management.
	client *vault.Client

	// role is the Vault Kubernetes auth role that this
	// application authenticates as. The role defines which
	// Vault policies (and thus which secrets) are accessible.
	role string

	// mountPath is the Vault auth mount path for the Kubernetes
	// auth method. The default is "kubernetes" but can be
	// customized for multi-cluster setups.
	mountPath string
}

// NewVaultSecretReader creates a new reader configured for the
// given Vault address and Kubernetes auth role. It does NOT
// authenticate — call Authenticate() after creation.
//
// The vaultAddr should be the full URL including scheme and port,
// e.g., "http://vault.vault-system.svc:8200" for in-cluster access
// or "https://vault.example.com:8200" for external access.
func NewVaultSecretReader(vaultAddr, role, mountPath string) (*VaultSecretReader, error) {
	// Configure the Vault client. The default configuration
	// reads VAULT_ADDR, VAULT_TOKEN, and other environment
	// variables, but we override the address explicitly.
	config := vault.DefaultConfig()
	config.Address = vaultAddr

	// Set a reasonable timeout. Vault operations should be fast;
	// if they take more than 30 seconds, something is wrong.
	config.Timeout = 30 * time.Second

	client, err := vault.NewClient(config)
	if err != nil {
		return nil, fmt.Errorf("failed to create Vault client: %w", err)
	}

	if mountPath == "" {
		mountPath = "kubernetes"
	}

	return &VaultSecretReader{
		client:    client,
		role:      role,
		mountPath: mountPath,
	}, nil
}

// Authenticate performs Kubernetes auth with Vault using the
// projected ServiceAccount token. The token is read from the
// well-known path where Kubernetes mounts it.
//
// On success, the Vault client is configured with a Vault token
// that can be used for subsequent operations. The Vault token
// has a TTL defined by the Vault role configuration.
//
// Returns the auth secret (containing the Vault token, TTL, and
	// policies) for inspection, and the token renewal function.
func (r *VaultSecretReader) Authenticate(ctx context.Context) (*vault.Secret, error) {
	// The default path where Kubernetes mounts the projected
	// ServiceAccount token. This can be overridden if you're
	// using a custom projected volume configuration.
	saTokenPath := "/var/run/secrets/kubernetes.io/serviceaccount/token"

	// Allow override via environment variable for testing
	// outside Kubernetes (e.g., in integration tests).
	if path := os.Getenv("SA_TOKEN_PATH"); path != "" {
		saTokenPath = path
	}

	// Create the Kubernetes auth method. This reads the SA token
	// from the specified path and sends it to Vault's
	// auth/kubernetes/login endpoint.
	k8sAuth, err := auth.NewKubernetesAuth(
		r.role,
		auth.WithServiceAccountTokenPath(saTokenPath),
		auth.WithMountPath(r.mountPath),
	)
	if err != nil {
		return nil, fmt.Errorf("failed to create kubernetes auth: %w", err)
	}

	// Login to Vault. This sends the SA token to Vault, which
	// verifies it by calling the Kubernetes TokenReview API.
	// If the token is valid and the SA matches the Vault role's
	// bound_service_account_names/namespaces, Vault returns a
	// Vault token with the policies from the role.
	authInfo, err := r.client.Auth().Login(ctx, k8sAuth)
	if err != nil {
		return nil, fmt.Errorf("vault login failed: %w", err)
	}

	if authInfo == nil {
		return nil, fmt.Errorf("vault login returned nil auth info")
	}

	log.Printf("Authenticated with Vault (policies: %v, ttl: %v)",
		authInfo.Auth.Policies, authInfo.Auth.LeaseDuration)

	return authInfo, nil
}

// ReadKVSecret reads a static secret from Vault's KV v2 secrets
// engine. KV v2 provides versioning, metadata, and soft-delete
// capabilities.
//
// The path should NOT include the "data/" prefix — this is added
// automatically by the KV v2 client. For example, to read
// "secret/data/production/database", pass path="production/database"
// and mount="secret".
func (r *VaultSecretReader) ReadKVSecret(
	ctx context.Context,
	mount string,
	path string,
) (map[string]interface{}, error) {
	// Use the KVv2 helper which handles the "data/" prefix and
	// extracts the nested data from the response automatically.
	secret, err := r.client.KVv2(mount).Get(ctx, path)
	if err != nil {
		return nil, fmt.Errorf("failed to read KV secret at %s/%s: %w",
			mount, path, err)
	}

	if secret == nil || secret.Data == nil {
		return nil, fmt.Errorf("secret at %s/%s is empty", mount, path)
	}

	log.Printf("Read KV secret: %s/%s (version: %d, created: %v)",
		mount, path, secret.VersionMetadata.Version,
		secret.VersionMetadata.CreatedTime)

	return secret.Data, nil
}

// ReadDynamicSecret reads a dynamic secret from Vault. Dynamic
// secrets are generated on-demand and have a lease that must be
// renewed or the credentials will expire.
//
// Common dynamic secret backends:
//   - database/creds/<role> — generates database credentials
//   - aws/creds/<role> — generates AWS IAM credentials
//   - pki/issue/<role> — generates TLS certificates
//
// The returned Secret includes lease information that the caller
// should use to manage credential lifecycle.
func (r *VaultSecretReader) ReadDynamicSecret(
	ctx context.Context,
	path string,
) (*vault.Secret, error) {
	secret, err := r.client.Logical().ReadWithContext(ctx, path)
	if err != nil {
		return nil, fmt.Errorf("failed to read dynamic secret at %s: %w",
			path, err)
	}

	if secret == nil {
		return nil, fmt.Errorf("no secret at path %s", path)
	}

	log.Printf("Read dynamic secret: %s (lease_id: %s, ttl: %ds)",
		path, secret.LeaseID, secret.LeaseDuration)

	return secret, nil
}

// StartTokenRenewal begins a background goroutine that
// automatically renews the Vault token before it expires.
// This is essential for long-running applications — without
// renewal, the token expires and all Vault operations fail.
//
// The renewal process uses Vault's token renewal API, which
// extends the token's TTL without generating a new token.
// If the token's max TTL is reached, renewal fails and the
// application should re-authenticate.
func (r *VaultSecretReader) StartTokenRenewal(
	ctx context.Context,
	authSecret *vault.Secret,
) {
	go func() {
		// NewLifetimeWatcher handles the renewal loop, including:
		//   - Calculating when to renew (before 2/3 of TTL expires)
		//   - Handling renewal failures with backoff
		//   - Signaling when the token is no longer renewable
		watcher, err := r.client.NewLifetimeWatcher(&vault.LifetimeWatcherInput{
				Secret:    authSecret,
				Increment: 3600, // Request 1-hour renewal
			})
		if err != nil {
			log.Printf("Failed to create token watcher: %v", err)
			return
		}

		go watcher.Start()
		defer watcher.Stop()

		for {
			select {
			case <-ctx.Done():
				log.Println("Token renewal stopped: context cancelled")
				return

			case err := <-watcher.DoneCh():
				// Token is no longer renewable. The application
				// should re-authenticate with Vault.
				log.Printf("Token renewal ended: %v — re-authentication needed", err)
				return

			case renewal := <-watcher.RenewCh():
				// Token was successfully renewed.
				log.Printf("Token renewed (ttl: %ds)",
					renewal.Secret.Auth.LeaseDuration)
			}
		}
	}()
}

func main() {
	// Read configuration from environment variables.
	// In production, these would come from a ConfigMap or
	// the Pod spec's environment section.
	vaultAddr := os.Getenv("VAULT_ADDR")
	if vaultAddr == "" {
		vaultAddr = "http://vault.vault-system.svc:8200"
	}

	vaultRole := os.Getenv("VAULT_ROLE")
	if vaultRole == "" {
		vaultRole = "myapp"
	}

	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Minute)
	defer cancel()

	// Create the secret reader.
	reader, err := NewVaultSecretReader(vaultAddr, vaultRole, "")
	if err != nil {
		log.Fatalf("Failed to create Vault reader: %v", err)
	}

	// Authenticate with Vault using the Kubernetes SA token.
	authSecret, err := reader.Authenticate(ctx)
	if err != nil {
		log.Fatalf("Failed to authenticate: %v", err)
	}

	// Start automatic token renewal in the background.
	reader.StartTokenRenewal(ctx, authSecret)

	// Read a static KV secret.
	dbCreds, err := reader.ReadKVSecret(ctx, "secret", "production/database")
	if err != nil {
		log.Fatalf("Failed to read database credentials: %v", err)
	}

	fmt.Println("Database credentials:")
	for key, value := range dbCreds {
		// In production, NEVER log secret values. This is for
		// demonstration only. Use the values directly in your
		// database connection code.
		fmt.Printf("  %s = %v\n", key, value)
	}

	// Read dynamic database credentials.
	// These are freshly generated for this request and will
	// be automatically revoked after the lease expires.
	dynamicCreds, err := reader.ReadDynamicSecret(ctx, "database/creds/myapp-role")
	if err != nil {
		log.Printf("Dynamic secret read failed (expected if DB engine not configured): %v", err)
	} else {
		fmt.Println("\nDynamic database credentials:")
		credJSON, _ := json.MarshalIndent(dynamicCreds.Data, "  ", "  ")
		fmt.Printf("  %s\n", credJSON)
		fmt.Printf("  Lease ID: %s\n", dynamicCreds.LeaseID)
		fmt.Printf("  Lease Duration: %d seconds\n", dynamicCreds.LeaseDuration)
	}

	// In a real application, you would now use these credentials
	// to connect to your database and start serving requests.
	// The token renewal goroutine keeps the Vault session alive.
	fmt.Println("\nApplication ready. Press Ctrl+C to exit.")
	select {}
}
```

---

## 30.5 Sealed Secrets — Encrypted Secrets in Git

Sealed Secrets, created by Bitnami, solve the "secrets in Git" problem. They use
asymmetric encryption: the cluster holds the private key, and anyone can encrypt
with the public key.

### 30.5.1 Architecture

```
Developer                  Git Repository              Kubernetes Cluster
┌──────────┐  kubeseal   ┌──────────────┐  GitOps    ┌──────────────────┐
│ Secret   │────────────▶│ SealedSecret │───────────▶│ SealedSecret CR  │
│ (plain)  │  (encrypt)  │ (encrypted)  │  (sync)    │                  │
└──────────┘              └──────────────┘             │ ┌──────────────┐│
                                                       │ │Sealed Secrets││
                                                       │ │ Controller   ││
                                                       │ │  (decrypt)   ││
                                                       │ └──────┬───────┘│
                                                       │        │        │
                                                       │ ┌──────▼───────┐│
                                                       │ │  K8s Secret  ││
                                                       │ │  (plaintext) ││
                                                       │ └──────────────┘│
                                                       └──────────────────┘
```

### 30.5.2 Installation

```bash
# Install the controller
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm install sealed-secrets sealed-secrets/sealed-secrets \
  -n kube-system

# Install the CLI tool
brew install kubeseal
# Or download from GitHub releases
```

### 30.5.3 Hands-On: Sealed Secrets Workflow

```bash
# Step 1: Create a regular Secret manifest (DO NOT apply it!)
cat > my-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: database-credentials
  namespace: production
type: Opaque
stringData:
  username: admin
  password: super-secret-password
  api-key: sk-1234567890abcdef
EOF

# Step 2: Seal the secret using the cluster's public key
kubeseal --format yaml < my-secret.yaml > sealed-secret.yaml

# Step 3: Inspect the sealed secret
cat sealed-secret.yaml
```

The sealed secret looks like:

```yaml
apiVersion: bitnami.com/v1alpha1
kind: SealedSecret
metadata:
  name: database-credentials
  namespace: production
spec:
  encryptedData:
    username: AgBy3i4OJSWK+PiTySYZZA9rO43cGDEq...
    password: AgCtrSEP/3R9Ihx6YHbS4u6K2Ud+ZXHks...
    api-key: AgBzY2VgLi0W3H8OlSrY7a+9Pz2p1kBLR...
  template:
    metadata:
      name: database-credentials
      namespace: production
    type: Opaque
```

```bash
# Step 4: Commit the sealed secret to Git (safe!)
git add sealed-secret.yaml
git commit -m "Add sealed database credentials"
git push

# Step 5: Apply the sealed secret to the cluster
kubectl apply -f sealed-secret.yaml

# Step 6: Verify the controller decrypted it
kubectl get secret database-credentials -n production
kubectl get secret database-credentials -n production -o jsonpath='{.data.password}' | base64 -d
# Output: super-secret-password

# Step 7: Delete the plaintext file!
rm my-secret.yaml
```

### 30.5.4 Scoping Modes

```bash
# strict (default) — bound to name AND namespace
kubeseal --scope strict < secret.yaml > sealed.yaml

# namespace-wide — bound to namespace only (can change name)
kubeseal --scope namespace-wide < secret.yaml > sealed.yaml

# cluster-wide — can be used in any namespace
kubeseal --scope cluster-wide < secret.yaml > sealed.yaml
```

**Security consideration:** `cluster-wide` scope means anyone who can read the
sealed secret from Git can apply it to any namespace. Use `strict` scope in
production.

### 30.5.5 Key Rotation

The Sealed Secrets controller automatically generates new signing keys every
30 days. Old keys are retained for decryption. To manually rotate:

```bash
# Fetch the current public key
kubeseal --fetch-cert > sealed-secrets-cert.pem

# Re-seal all secrets with the new key
for f in sealed-*.yaml; do
  kubeseal --re-encrypt < "$f" > "$f.tmp" && mv "$f.tmp" "$f"
done
```

---

## 30.6 SOPS — Mozilla Secrets Management

SOPS (Secrets OPerationS) encrypts files while preserving their structure.
It supports YAML, JSON, ENV, and INI formats, encrypting only the **values**
while leaving keys visible.

### 30.6.1 How SOPS Works

```yaml
# Original (plaintext)
database:
  host: db.example.com
  username: admin
  password: super-secret

# After SOPS encryption — keys are visible, values are encrypted
database:
  host: ENC[AES256_GCM,data:F5mPOkYwWQ==,iv:abc...,tag:def...,type:str]
  username: ENC[AES256_GCM,data:8kGp7g==,iv:ghi...,tag:jkl...,type:str]
  password: ENC[AES256_GCM,data:W8eRPSmLKQ==,iv:mno...,tag:pqr...,type:str]
sops:
  kms:
    - arn: arn:aws:kms:us-east-1:123456789012:key/mrk-xxxx
  gcp_kms:
    - resource_id: projects/my-project/locations/global/keyRings/sops/cryptoKeys/sops-key
  age:
    - recipient: age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p
  lastmodified: "2024-01-15T10:30:00Z"
  version: 3.8.1
```

### 30.6.2 Installation and Setup

```bash
# Install SOPS
brew install sops
# Or download from GitHub

# Option 1: Use age (recommended for simplicity)
age-keygen -o ~/.sops/key.txt

# Create .sops.yaml configuration
cat > .sops.yaml <<EOF
creation_rules:
  # Encrypt production secrets with AWS KMS
  - path_regex: production/.*\.yaml$
    kms: 'arn:aws:kms:us-east-1:123456789012:key/mrk-xxxx'

  # Encrypt staging secrets with age
  - path_regex: staging/.*\.yaml$
    age: 'age1ql3z7hjy54pw3hyww5ayyfg7zqgvc7w3j2elw8zmrj2kg5sfn9aqmcac8p'

  # Encrypt everything else with PGP
  - pgp: '1234567890ABCDEF1234567890ABCDEF12345678'
EOF
```

### 30.6.3 Usage

```bash
# Encrypt a file
sops -e secrets.yaml > secrets.enc.yaml

# Decrypt a file
sops -d secrets.enc.yaml

# Edit in-place (decrypts, opens editor, re-encrypts)
sops secrets.enc.yaml

# Rotate the data key (re-encrypts with new data key)
sops -r secrets.enc.yaml
```

### 30.6.4 SOPS with Flux CD

Flux CD has native SOPS integration for GitOps:

```yaml
# Kustomization that decrypts SOPS-encrypted secrets
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: app-secrets
  namespace: flux-system
spec:
  interval: 10m
  path: ./secrets/production
  prune: true
  sourceRef:
    kind: GitRepository
    name: flux-system
  decryption:
    provider: sops
    secretRef:
      name: sops-age  # Secret containing the age private key
```

```bash
# Create the decryption key secret in the cluster
cat ~/.sops/key.txt | kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin
```

---

## 30.7 Secret Rotation Patterns

### 30.7.1 Why Rotate Secrets?

- **Breach containment:** If a secret is compromised, rotation limits the window
  of exposure
- **Compliance:** Many standards (PCI DSS, SOC 2, HIPAA) require regular rotation
- **Least privilege over time:** Credentials issued for a specific purpose should
  not persist indefinitely

### 30.7.2 Rotation Strategies

#### Strategy 1: Blue-Green Rotation

```
Time ──────────────────────────────────────────────────▶

Key A:  ████████████████████████████
                    ▲ Key B created
Key B:              ████████████████████████████████████
                              ▲ Pods rolled to use Key B
Key A:              ██████████████                      
                                   ▲ Key A revoked
```

1. Create new credentials (Key B) alongside existing ones (Key A)
2. Update the Kubernetes Secret with Key B
3. Roll Pods to pick up the new secret
4. Verify all Pods are using Key B
5. Revoke Key A

#### Strategy 2: Vault Dynamic Secrets

Vault generates short-lived credentials on demand. No manual rotation needed:

```bash
# Configure the database secrets engine
vault write database/config/mydb \
  plugin_name=postgresql-database-plugin \
  allowed_roles="myapp" \
  connection_url="postgresql://{{username}}:{{password}}@db:5432/mydb?sslmode=require" \
  username="vault-admin" \
  password="admin-password"

# Create a role that generates credentials with 1-hour TTL
vault write database/roles/myapp \
  db_name=mydb \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; GRANT SELECT ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Each read generates fresh credentials
vault read database/creds/myapp
# username: v-kubernetes-myapp-abc123
# password: randomly-generated-password
# lease_id: database/creds/myapp/abcd-1234
# lease_duration: 3600
```

#### Strategy 3: Stakater Reloader

Automatically restart Pods when Secrets or ConfigMaps change:

```bash
# Install Reloader
helm repo add stakater https://stakater.github.io/stakater-charts
helm install reloader stakater/reloader -n kube-system
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  annotations:
    # Reloader watches this specific secret and triggers a rolling
    # restart when it changes
    reloader.stakater.com/auto: "true"
spec:
  template:
    spec:
      containers:
        - name: myapp
          image: myapp:latest
          envFrom:
            - secretRef:
                name: database-credentials
```

### 30.7.3 Rotation Automation with Go

```go
// Package main demonstrates an automated secret rotation controller
// that periodically rotates database passwords and updates both the
// database and the Kubernetes Secret. This pattern is useful when
// you cannot use Vault dynamic secrets and need to manage rotation
// yourself.
//
// The controller:
//   1. Generates a new random password
//   2. Updates the password in the database (ALTER ROLE)
//   3. Updates the Kubernetes Secret
//   4. Verifies the new password works
//   5. Logs the rotation event for audit
package main

import (
	"context"
	"crypto/rand"
	"database/sql"
	"encoding/base64"
	"fmt"
	"log"
	"time"

	_ "github.com/lib/pq"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/rest"
)

// SecretRotator handles the lifecycle of rotating database
// credentials stored in Kubernetes Secrets. It coordinates
// the password change between the database and Kubernetes
// to ensure atomicity and rollback capability.
type SecretRotator struct {
	// k8sClient communicates with the Kubernetes API server
	// to read and update Secret resources.
	k8sClient kubernetes.Interface

	// rotationInterval defines how frequently passwords are
	// rotated. A shorter interval reduces the window of
	// exposure if a credential is compromised.
	rotationInterval time.Duration
}

// generatePassword creates a cryptographically secure random
// password of the specified length. It uses crypto/rand to
// ensure the password is suitable for production use.
//
// The password is base64url-encoded (no padding) which ensures
// it contains only alphanumeric characters plus '-' and '_',
// making it safe for use in connection strings without escaping.
func generatePassword(length int) (string, error) {
	// We generate more random bytes than needed because base64
	// encoding expands the data by ~33%. We then truncate to
	// the desired length.
	bytes := make([]byte, length)
	if _, err := rand.Read(bytes); err != nil {
		return "", fmt.Errorf("failed to generate random bytes: %w", err)
	}
	return base64.RawURLEncoding.EncodeToString(bytes)[:length], nil
}

// RotateDBPassword performs a single password rotation cycle.
// It follows a careful order of operations to minimize the risk
// of leaving the system in an inconsistent state:
//
//   1. Read the current credentials from Kubernetes
//   2. Generate a new password
//   3. Update the database password (using current credentials)
//   4. Verify the new password works
//   5. Update the Kubernetes Secret with the new password
//
// If step 4 fails, we attempt to rollback the database password.
// If step 5 fails, the database has the new password but Pods
// still have the old one — they will fail until the Secret is
// manually updated.
func (r *SecretRotator) RotateDBPassword(
	ctx context.Context,
	namespace, secretName string,
) error {
	log.Printf("Starting password rotation for %s/%s", namespace, secretName)

	// Step 1: Read current credentials from the Kubernetes Secret.
	secret, err := r.k8sClient.CoreV1().Secrets(namespace).Get(
		ctx, secretName, metav1.GetOptions{},
	)
	if err != nil {
		return fmt.Errorf("failed to read secret %s/%s: %w",
			namespace, secretName, err)
	}

	currentUser := string(secret.Data["username"])
	currentPass := string(secret.Data["password"])
	dbHost := string(secret.Data["host"])
	dbName := string(secret.Data["database"])

	// Step 2: Generate a new secure password.
	newPassword, err := generatePassword(32)
	if err != nil {
		return fmt.Errorf("failed to generate password: %w", err)
	}

	// Step 3: Connect to the database with current credentials
	// and change the password.
	connStr := fmt.Sprintf(
		"host=%s user=%s password=%s dbname=%s sslmode=require",
		dbHost, currentUser, currentPass, dbName,
	)

	db, err := sql.Open("postgres", connStr)
	if err != nil {
		return fmt.Errorf("failed to connect to database: %w", err)
	}
	defer db.Close()

	// ALTER ROLE changes the password. We use a parameterized query
	// to prevent SQL injection (though the password is generated by
		// us, defense in depth is important).
	_, err = db.ExecContext(ctx,
		fmt.Sprintf("ALTER ROLE %s WITH PASSWORD $1", currentUser),
		newPassword,
	)
	if err != nil {
		return fmt.Errorf("failed to update database password: %w", err)
	}

	log.Printf("Database password updated for user %s", currentUser)

	// Step 4: Verify the new password works by connecting with it.
	verifyConnStr := fmt.Sprintf(
		"host=%s user=%s password=%s dbname=%s sslmode=require",
		dbHost, currentUser, newPassword, dbName,
	)

	verifyDB, err := sql.Open("postgres", verifyConnStr)
	if err != nil {
		// Rollback: try to restore the old password
		log.Printf("WARNING: New password verification failed, attempting rollback: %v", err)
		db.ExecContext(ctx,
			fmt.Sprintf("ALTER ROLE %s WITH PASSWORD $1", currentUser),
			currentPass,
		)
		return fmt.Errorf("new password verification failed: %w", err)
	}

	if err := verifyDB.PingContext(ctx); err != nil {
		verifyDB.Close()
		log.Printf("WARNING: New password ping failed, attempting rollback: %v", err)
		db.ExecContext(ctx,
			fmt.Sprintf("ALTER ROLE %s WITH PASSWORD $1", currentUser),
			currentPass,
		)
		return fmt.Errorf("new password ping failed: %w", err)
	}
	verifyDB.Close()

	// Step 5: Update the Kubernetes Secret with the new password.
	secret.Data["password"] = []byte(newPassword)
	secret.Data["connection-string"] = []byte(fmt.Sprintf(
			"postgresql://%s:%s@%s:5432/%s?sslmode=require",
			currentUser, newPassword, dbHost, dbName,
		))

	// Add rotation metadata as annotations for audit purposes.
	if secret.Annotations == nil {
		secret.Annotations = make(map[string]string)
	}
	secret.Annotations["rotation/last-rotated"] = time.Now().UTC().Format(time.RFC3339)
	secret.Annotations["rotation/rotated-by"] = "secret-rotator"

	_, err = r.k8sClient.CoreV1().Secrets(namespace).Update(
		ctx, secret, metav1.UpdateOptions{},
	)
	if err != nil {
		// CRITICAL: The database has the new password, but the
		// Secret still has the old one. Manual intervention needed.
		return fmt.Errorf(
			"CRITICAL: database updated but K8s secret update failed "+
			"(new password: %s...): %w",
			newPassword[:8], err)
	}

	log.Printf("Successfully rotated password for %s/%s", namespace, secretName)
	return nil
}

func main() {
	// Build in-cluster config — this controller runs inside the cluster.
	config, err := rest.InClusterConfig()
	if err != nil {
		log.Fatalf("Failed to build in-cluster config: %v", err)
	}

	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		log.Fatalf("Failed to create Kubernetes client: %v", err)
	}

	rotator := &SecretRotator{
		k8sClient:        clientset,
		rotationInterval: 24 * time.Hour,
	}

	// Run the rotation loop. In production, you'd use a proper
	// controller framework (controller-runtime) with leader election
	// to ensure only one instance rotates at a time.
	ticker := time.NewTicker(rotator.rotationInterval)
	defer ticker.Stop()

	ctx := context.Background()

	// Rotate immediately on startup, then on each tick.
	for {
		if err := rotator.RotateDBPassword(ctx, "production", "database-credentials"); err != nil {
			log.Printf("Rotation failed: %v", err)
		}

		select {
		case <-ticker.C:
			continue
		case <-ctx.Done():
			return
		}
	}
}
```

---

## 30.8 RBAC for Secrets

### 30.8.1 Default RBAC Problems

By default, many Kubernetes distributions grant overly broad access to secrets.
Common issues:

```yaml
# DANGEROUS: Wildcard access to all resources includes secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: overly-permissive
rules:
  - apiGroups: [""]
    resources: ["*"]  # This includes secrets!
    verbs: ["*"]
```

### 30.8.2 Principle of Least Privilege for Secrets

```yaml
# Good: Separate roles for secret access
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
  # Only allow reading specific secrets by name
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["database-credentials", "api-keys"]
    verbs: ["get"]

---
# Good: Separate role for secret creation (CI/CD)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-manager
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["create", "update", "patch"]
    # Note: resourceNames doesn't work with "create" verb
    # because the resource doesn't exist yet

---
# Good: Deny list/watch to prevent enumeration
# (Don't grant "list" or "watch" unless absolutely needed)
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: production
rules:
  # Application can read its own config
  - apiGroups: [""]
    resources: ["configmaps"]
    resourceNames: ["app-config"]
    verbs: ["get", "watch"]
  # Application can read its own secret (but not list all secrets)
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["app-credentials"]
    verbs: ["get"]
```

### 30.8.3 Auditing Secret Access

```yaml
# Audit policy for secret operations
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
  # Log ALL secret access at the Request level
  # (includes who accessed what, but not the secret values)
  - level: Request
    resources:
      - group: ""
        resources: ["secrets"]
    # Exclude system components to reduce noise
    users:
      - "system:kube-controller-manager"
      - "system:kube-scheduler"
    omitStages:
      - "RequestReceived"
  
  # Log secret reads by non-system users at Metadata level
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets"]
    verbs: ["get", "list", "watch"]
```

**Attack detection patterns to look for in audit logs:**

1. **Secret enumeration:** A user listing secrets across multiple namespaces
2. **Unusual access:** A ServiceAccount reading secrets it doesn't normally access
3. **Watch on secrets:** Someone setting up a watch to capture secret changes
4. **High-frequency access:** Rapid repeated reads of the same secret

---

## 30.9 Security Best Practices Summary

### 30.9.1 Storage

| Practice | Priority | Description |
|----------|----------|-------------|
| Enable encryption at rest | **Critical** | Use EncryptionConfiguration with aescbc or KMS v2 |
| Encrypt etcd disk | High | Use LUKS/dm-crypt for the etcd data directory |
| Encrypt etcd backups | High | Use encryption when storing etcd snapshots |
| Use external secret stores | High | Vault, AWS Secrets Manager, etc. |

### 30.9.2 Access Control

| Practice | Priority | Description |
|----------|----------|-------------|
| Restrict secret RBAC | **Critical** | Use resourceNames, avoid wildcards |
| Don't grant list/watch on secrets | High | Prevents enumeration |
| Use separate namespaces | High | Namespace isolation for secret access |
| Audit secret access | High | Enable audit logging for secrets |
| Use Projected SA tokens | High | Not legacy long-lived tokens |

### 30.9.3 Operations

| Practice | Priority | Description |
|----------|----------|-------------|
| Rotate secrets regularly | High | At least every 90 days, ideally more often |
| Use short-lived credentials | High | Vault dynamic secrets, projected tokens |
| Don't commit secrets to Git | **Critical** | Use Sealed Secrets or SOPS |
| Scan for secret leaks | High | Use tools like truffleHog, gitleaks |
| Use environment variables or volumes | Medium | Not command-line arguments (visible in /proc) |

### 30.9.4 What NOT to Do

1. **Never** put secrets in ConfigMaps — they're not intended for sensitive data
2. **Never** embed secrets in container images
3. **Never** pass secrets as command-line arguments (visible in `ps`)
4. **Never** log secret values (even in debug mode)
5. **Never** use base64 as "encryption" — it's encoding, not encryption
6. **Never** store secrets in annotations or labels
7. **Never** commit plaintext secrets to Git (even in private repos)

---

## 30.10 Summary

Kubernetes secrets management is a defense-in-depth challenge that requires
multiple layers:

1. **Encryption at rest** protects secrets stored in etcd from direct disk access
   and backup theft
2. **External secret stores** (Vault, AWS Secrets Manager) keep the source of truth
   outside the cluster, with their own access controls and audit logs
3. **Sealed Secrets and SOPS** enable GitOps workflows without exposing secrets in
   repositories
4. **RBAC** controls who can read which secrets through the API
5. **Short-lived credentials** (Vault dynamic secrets, projected SA tokens) minimize
   the window of exposure
6. **Rotation automation** ensures credentials don't become stale attack vectors

No single tool solves the secrets problem. A production-grade setup combines
encryption at rest, an external secret store, GitOps-safe secret management, strict
RBAC, and automated rotation. The Go programs in this chapter demonstrate the
building blocks for implementing these patterns.

In the next chapter, we'll explore supply chain security — how to ensure the
container images running in your cluster are genuine, unmodified, and free of
known vulnerabilities.

---

## Exercises

1. **Encryption Verification:** Enable encryption at rest with `secretbox`. Create
   a secret, then read it directly from etcd. Verify the data is encrypted. Rotate
   the key and verify all secrets are re-encrypted.

2. **Vault Integration:** Deploy Vault in dev mode. Configure the Kubernetes auth
   method. Write a Go program that authenticates, reads a secret, and handles
   token renewal gracefully.

3. **Sealed Secrets Pipeline:** Set up a complete GitOps pipeline: create a secret,
   seal it, commit to Git, have a GitOps tool (Flux or ArgoCD) sync it, and verify
   the decrypted secret is available in the cluster.

4. **RBAC Audit:** In a test cluster, create roles with different levels of secret
   access. Use `kubectl auth can-i` to verify the permission boundaries. Enable
   audit logging and verify that secret access is logged.

5. **Rotation Controller:** Extend the rotation controller from section 30.7.3 to
   handle multiple secrets, implement leader election for HA, and add Prometheus
   metrics for rotation success/failure.

6. **Breach Simulation:** Simulate an etcd backup theft. Extract secrets from the
   backup with and without encryption at rest enabled. Document the difference in
   effort required.
