# Chapter 18: ConfigMaps, Secrets & Environment Management

> *"Hard-coding configuration into a container image is like printing your
> phone number on your business card in ink — fine until you change offices."*

The previous chapter showed how Services provide stable networking identities
for Pods. But a running application needs more than network connectivity — it
needs **configuration**: database URLs, feature flags, API keys, TLS
certificates, tuning parameters, and operational secrets.

Kubernetes separates configuration from container images using three
primitives:

- **ConfigMaps** — for non-sensitive configuration data.
- **Secrets** — for sensitive data (passwords, tokens, keys).
- **Downward API** — for Pod-level metadata (name, namespace, labels, resource
  limits).

This chapter covers each in depth, with production-ready Go examples
demonstrating how to consume, watch, and layer configuration.

---

## 18.1 ConfigMaps

### 18.1.1 Purpose: Decouple Configuration from Container Images

A **ConfigMap** is a Kubernetes object that stores non-sensitive configuration
data as key-value pairs. It exists to solve a fundamental problem:

```
Without ConfigMaps:
  Image v1.2.3 → hard-coded DB_HOST=prod-db.internal
  ↓
  Need to change DB_HOST?
  ↓
  Rebuild image → push → redeploy
  (minutes to hours)

With ConfigMaps:
  Image v1.2.3 → reads DB_HOST from ConfigMap
  ↓
  Need to change DB_HOST?
  ↓
  Update ConfigMap → Pod picks up the change
  (seconds, no rebuild)
```

This is the **12-factor app** principle of strict separation of config from
code (Factor III). The same image runs in dev, staging, and production — only
the ConfigMap changes.

### 18.1.2 Creating ConfigMaps

There are four ways to create a ConfigMap:

#### From Literal Key-Value Pairs

```bash
$ kubectl create configmap app-config \
    --from-literal=DB_HOST=postgres.production.svc.cluster.local \
    --from-literal=DB_PORT=5432 \
    --from-literal=LOG_LEVEL=info \
    --from-literal=MAX_CONNECTIONS=100
```

Resulting YAML:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  DB_HOST: postgres.production.svc.cluster.local
  DB_PORT: "5432"
  LOG_LEVEL: info
  MAX_CONNECTIONS: "100"
```

> **Note:** All ConfigMap values are **strings**. Even though `5432` looks like
> an integer, it is stored as the string `"5432"`. Your application must parse
> numeric values.

#### From Files

```bash
# Create a config file
$ cat > app.properties <<EOF
database.host=postgres.production.svc.cluster.local
database.port=5432
database.pool.maxSize=100
cache.ttl=300
EOF

# Create ConfigMap from file — the filename becomes the key
$ kubectl create configmap app-config --from-file=app.properties

# Result:
#   data:
#     app.properties: |
#       database.host=postgres.production.svc.cluster.local
#       database.port=5432
#       ...
```

You can also specify a custom key name:

```bash
$ kubectl create configmap app-config \
    --from-file=config.ini=app.properties
# Key is "config.ini", not "app.properties"
```

#### From Directories

```bash
# Create a config directory with multiple files
$ mkdir config/
$ echo "info" > config/log-level
$ echo "5432" > config/db-port
$ echo '{"retries": 3, "timeout": 30}' > config/http-client.json

# All files in the directory become keys
$ kubectl create configmap app-config --from-dir=config/

# Result:
#   data:
#     log-level: info
#     db-port: "5432"
#     http-client.json: '{"retries": 3, "timeout": 30}'
```

#### From YAML Manifest

```yaml
# File: configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: production
  labels:
    app: backend
data:
  # Simple key-value pairs
  DB_HOST: postgres.production.svc.cluster.local
  DB_PORT: "5432"
  LOG_LEVEL: info

  # Multi-line configuration file embedded as a value
  nginx.conf: |
    worker_processes auto;
    events {
        worker_connections 1024;
    }
    http {
        upstream backend {
            server 127.0.0.1:8080;
        }
        server {
            listen 80;
            location / {
                proxy_pass http://backend;
            }
        }
    }

  # JSON configuration embedded as a value
  feature-flags.json: |
    {
      "enableNewUI": true,
      "enableMetrics": true,
      "maxBatchSize": 500,
      "maintenanceMode": false
    }
```

### 18.1.3 Consuming ConfigMaps

There are three ways for a Pod to consume a ConfigMap:

#### As Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  containers:
    - name: backend
      image: myapp:1.0
      env:
        # Individual keys from a ConfigMap
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_PORT
              optional: true    # Pod starts even if key doesn't exist
```

Or inject **all** keys at once with `envFrom`:

```yaml
spec:
  containers:
    - name: backend
      image: myapp:1.0
      envFrom:
        - configMapRef:
            name: app-config
          prefix: APP_          # optional: prepend "APP_" to all keys
```

This creates environment variables `APP_DB_HOST`, `APP_DB_PORT`,
`APP_LOG_LEVEL`, etc. Keys that are not valid environment variable names (e.g.,
containing dots or dashes) are **silently skipped**.

#### As Volume Mounts

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  volumes:
    - name: config-volume
      configMap:
        name: app-config
        # Optional: select specific keys and set permissions
        items:
          - key: nginx.conf
            path: nginx.conf        # file name inside the volume
            mode: 0644
          - key: feature-flags.json
            path: flags.json
        defaultMode: 0644
  containers:
    - name: backend
      image: myapp:1.0
      volumeMounts:
        - name: config-volume
          mountPath: /etc/app-config
          readOnly: true
```

This creates files inside the container:

```
/etc/app-config/
├── nginx.conf          ← content of the "nginx.conf" key
└── flags.json          ← content of the "feature-flags.json" key
```

If you omit `items`, *every* key in the ConfigMap becomes a file.

> **Important:** Mounting a ConfigMap volume at a directory **replaces** the
> entire directory contents. Use `subPath` to mount a single file without
> replacing the directory:
>
> ```yaml
> volumeMounts:
>   - name: config-volume
>     mountPath: /etc/nginx/nginx.conf
>     subPath: nginx.conf
> ```
>
> **However**, `subPath` mounts do **not** receive automatic updates when the
> ConfigMap changes. Only full-directory mounts support live updates.

#### As Command-Line Arguments

```yaml
spec:
  containers:
    - name: backend
      image: myapp:1.0
      command: ["myapp"]
      args:
        - "--db-host=$(DB_HOST)"
        - "--db-port=$(DB_PORT)"
        - "--log-level=$(LOG_LEVEL)"
      env:
        - name: DB_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_HOST
        - name: DB_PORT
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: DB_PORT
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: LOG_LEVEL
```

The `$(VAR_NAME)` syntax performs variable substitution at Pod creation time.

### 18.1.4 Immutable ConfigMaps

Starting with Kubernetes 1.21 (GA), you can mark a ConfigMap as **immutable**:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config-v2
immutable: true                 # ← cannot be modified after creation
data:
  DB_HOST: postgres.production.svc.cluster.local
  DB_PORT: "5432"
```

Benefits:

1. **Performance:** The kubelet stops polling the API server for updates to
   immutable ConfigMaps, significantly reducing API server load in clusters
   with thousands of Pods.
2. **Safety:** Prevents accidental changes that could break running
   applications.
3. **GitOps-friendly:** Encourages versioned ConfigMaps (`app-config-v1`,
   `app-config-v2`) where Deployments reference a specific version.

The trade-off: to update configuration, you must create a **new** ConfigMap
and update the Deployment to reference it. This triggers a rolling update,
which is often desirable for auditability.

### 18.1.5 ConfigMap Size Limit

A single ConfigMap cannot exceed **1 MiB** (1,048,576 bytes) of data. This
limit includes all keys, values, and the object metadata.

If you need to store larger configuration:

- Split across multiple ConfigMaps.
- Use a persistent volume with a configuration file.
- Store large configs in an external system (consul, S3) and reference them
  by URL.

### 18.1.6 Hands-on: Create ConfigMap from File and Mount as Volume

```bash
# Create a realistic configuration file
$ cat > app-config.yaml <<'EOF'
server:
  host: 0.0.0.0
  port: 8080
  readTimeout: 30s
  writeTimeout: 30s

database:
  host: postgres.default.svc.cluster.local
  port: 5432
  name: myapp
  maxOpenConns: 25
  maxIdleConns: 5

logging:
  level: info
  format: json

features:
  enableCache: true
  enableMetrics: true
  enableTracing: false
EOF

# Create the ConfigMap
$ kubectl create configmap app-config --from-file=config.yaml=app-config.yaml

# Deploy a Pod that mounts the ConfigMap
$ kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: config-test
spec:
  volumes:
    - name: config
      configMap:
        name: app-config
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "cat /etc/config/config.yaml && sleep 3600"]
      volumeMounts:
        - name: config
          mountPath: /etc/config
EOF

# Verify the file is mounted correctly
$ kubectl logs config-test
server:
  host: 0.0.0.0
  port: 8080
  ...
```

### 18.1.7 Hands-on: Live Config Reload — Watching Mounted ConfigMap Changes in Go

When a ConfigMap is mounted as a volume (without `subPath`), Kubernetes
updates the files automatically. The kubelet periodically checks for changes
(default: every 60 seconds + a jitter, configurable via
`--sync-frequency`).

Here is a Go application that watches for configuration file changes using
`fsnotify`:

```go
// File: cmd/config-watcher/main.go
//
// Package main demonstrates how to watch for ConfigMap changes when the
// ConfigMap is mounted as a volume in a Kubernetes Pod.
//
// Kubernetes updates mounted ConfigMaps by atomically swapping a symlink.
// The actual directory structure looks like:
//
//	/etc/config/
//	├── ..data -> ..2024_01_15_10_30_00.123456789    (symlink to current version)
//	├── ..2024_01_15_10_30_00.123456789/             (timestamped directory)
//	│   └── config.yaml                              (actual config file)
//	└── config.yaml -> ..data/config.yaml            (symlink for convenience)
//
// When the ConfigMap is updated, Kubernetes creates a new timestamped
// directory, writes the new files into it, and atomically swaps the
// ..data symlink. This means fsnotify will see a CREATE event on the
// ..data symlink.
//
// Usage:
//
//	CONFIG_PATH=/etc/config/config.yaml go run main.go
package main

import (
	"fmt"
	"os"
	"path/filepath"
	"time"

	"github.com/fsnotify/fsnotify"
	"gopkg.in/yaml.v3"
)

// AppConfig represents the application configuration structure.
// It is deserialized from the YAML ConfigMap data.
//
// Fields are tagged with `yaml` struct tags to map YAML keys to Go
// struct fields. The structure mirrors the ConfigMap YAML format:
//
//	server:
//	  host: 0.0.0.0
//	  port: 8080
//	database:
//	  host: postgres.default.svc.cluster.local
//	  ...
type AppConfig struct {
	// Server holds the HTTP server configuration including bind address,
	// port, and timeout settings.
	Server struct {
		Host         string        `yaml:"host"`
		Port         int           `yaml:"port"`
		ReadTimeout  time.Duration `yaml:"readTimeout"`
		WriteTimeout time.Duration `yaml:"writeTimeout"`
	} `yaml:"server"`

	// Database holds the PostgreSQL connection parameters including
	// hostname, port, database name, and connection pool settings.
	Database struct {
		Host         string `yaml:"host"`
		Port         int    `yaml:"port"`
		Name         string `yaml:"name"`
		MaxOpenConns int    `yaml:"maxOpenConns"`
		MaxIdleConns int    `yaml:"maxIdleConns"`
	} `yaml:"database"`

	// Logging controls the log output level and format.
	// Level should be one of: debug, info, warn, error.
	// Format should be one of: json, text.
	Logging struct {
		Level  string `yaml:"level"`
		Format string `yaml:"format"`
	} `yaml:"logging"`

	// Features contains boolean feature flags that can be toggled
	// at runtime by updating the ConfigMap.
	Features struct {
		EnableCache   bool `yaml:"enableCache"`
		EnableMetrics bool `yaml:"enableMetrics"`
		EnableTracing bool `yaml:"enableTracing"`
	} `yaml:"features"`
}

// loadConfig reads and parses the YAML configuration file at the given path.
// It returns the parsed configuration or an error if the file cannot be
// read or contains invalid YAML.
//
// Parameters:
//   - path: The absolute filesystem path to the YAML configuration file.
//     In a Kubernetes Pod, this is typically a path under the mounted
//     ConfigMap volume (e.g., /etc/config/config.yaml).
//
// Returns:
//   - *AppConfig: The parsed configuration struct, or nil on error.
//   - error: Non-nil if the file cannot be read or parsed.
func loadConfig(path string) (*AppConfig, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return nil, fmt.Errorf("reading config file %s: %w", path, err)
	}

	var config AppConfig
	if err := yaml.Unmarshal(data, &config); err != nil {
		return nil, fmt.Errorf("parsing config file %s: %w", path, err)
	}

	return &config, nil
}

// watchConfigMap sets up a filesystem watcher on the directory containing
// the configuration file. When Kubernetes updates a mounted ConfigMap, it
// atomically swaps a symlink named "..data" in the mount directory. This
// function detects that swap and reloads the configuration.
//
// Parameters:
//   - configPath: The path to the configuration file to watch
//     (e.g., /etc/config/config.yaml).
//   - onChange: A callback function invoked with the new configuration
//     whenever the file changes. The callback runs synchronously — long
//     operations should be dispatched to a goroutine.
//
// Returns:
//   - error: Non-nil if the watcher cannot be created or set up.
//
// This function blocks until the process is terminated.
func watchConfigMap(configPath string, onChange func(*AppConfig)) error {
	watcher, err := fsnotify.NewWatcher()
	if err != nil {
		return fmt.Errorf("creating filesystem watcher: %w", err)
	}
	defer watcher.Close()

	// Watch the directory, not the file itself. Kubernetes updates
	// ConfigMap volumes by atomically swapping a "..data" symlink in
	// the parent directory. Watching the file directly would miss these
	// atomic updates because the inode changes.
	configDir := filepath.Dir(configPath)
	if err := watcher.Add(configDir); err != nil {
		return fmt.Errorf("watching directory %s: %w", configDir, err)
	}

	fmt.Printf("[watcher] Watching directory %s for ConfigMap changes\n", configDir)

	for {
		select {
		case event, ok := <-watcher.Events:
			if !ok {
				return nil
			}

			// Kubernetes ConfigMap volume updates trigger a CREATE event
			// on the "..data" symlink. We detect this specific event to
			// avoid reloading on unrelated filesystem activity.
			if filepath.Base(event.Name) == "..data" && event.Has(fsnotify.Create) {
				fmt.Printf("[watcher] ConfigMap updated (event: %s)\n", event)

				// Brief delay to ensure all symlinks are updated.
				time.Sleep(100 * time.Millisecond)

				newConfig, err := loadConfig(configPath)
				if err != nil {
					fmt.Printf("[watcher] Error reloading config: %v\n", err)
					continue
				}

				fmt.Printf("[watcher] Configuration reloaded successfully\n")
				fmt.Printf("[watcher]   Log level: %s\n", newConfig.Logging.Level)
				fmt.Printf("[watcher]   Cache enabled: %v\n", newConfig.Features.EnableCache)
				fmt.Printf("[watcher]   Tracing enabled: %v\n", newConfig.Features.EnableTracing)

				onChange(newConfig)
			}

		case err, ok := <-watcher.Errors:
			if !ok {
				return nil
			}
			fmt.Printf("[watcher] Error: %v\n", err)
		}
	}
}

func main() {
	// Read the config file path from an environment variable.
	// Default to /etc/config/config.yaml — the conventional mount point.
	configPath := os.Getenv("CONFIG_PATH")
	if configPath == "" {
		configPath = "/etc/config/config.yaml"
	}

	// Load the initial configuration.
	config, err := loadConfig(configPath)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Failed to load initial config: %v\n", err)
		os.Exit(1)
	}

	fmt.Println("=== ConfigMap Watcher ===")
	fmt.Printf("Config path: %s\n", configPath)
	fmt.Printf("Server: %s:%d\n", config.Server.Host, config.Server.Port)
	fmt.Printf("Database: %s:%d/%s\n",
		config.Database.Host, config.Database.Port, config.Database.Name)
	fmt.Printf("Log level: %s\n", config.Logging.Level)
	fmt.Printf("Features: cache=%v metrics=%v tracing=%v\n",
		config.Features.EnableCache,
		config.Features.EnableMetrics,
		config.Features.EnableTracing)
	fmt.Println()

	// Watch for changes. The callback receives the new configuration
	// whenever the ConfigMap is updated. In a real application, this
	// callback would update a thread-safe config holder (e.g., using
		// sync/atomic.Value) and reconfigure runtime components.
	err = watchConfigMap(configPath, func(newConfig *AppConfig) {
			// In a production application, you would:
			// 1. Store the new config atomically (atomic.Value or sync.RWMutex).
			// 2. Reconfigure the logger.
			// 3. Update feature flag checks.
			// 4. Optionally resize database connection pools.
			fmt.Printf("[app] Applied new configuration: log=%s cache=%v\n",
				newConfig.Logging.Level, newConfig.Features.EnableCache)
		})
	if err != nil {
		fmt.Fprintf(os.Stderr, "Watcher failed: %v\n", err)
		os.Exit(1)
	}
}
```

Deploy and test live reload:

```bash
# Deploy the watcher Pod
$ kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: config-watcher
spec:
  volumes:
    - name: config
      configMap:
        name: app-config
  containers:
    - name: watcher
      image: myregistry/config-watcher:latest
      env:
        - name: CONFIG_PATH
          value: /etc/config/config.yaml
      volumeMounts:
        - name: config
          mountPath: /etc/config
EOF

# Now update the ConfigMap
$ kubectl edit configmap app-config
# Change "level: info" to "level: debug"
# Save and exit

# Watch the Pod logs — the watcher detects the change
$ kubectl logs -f config-watcher
[watcher] Watching directory /etc/config for ConfigMap changes
[watcher] ConfigMap updated (event: CREATE "/etc/config/..data")
[watcher] Configuration reloaded successfully
[watcher]   Log level: debug
[watcher]   Cache enabled: true
[watcher]   Tracing enabled: false
[app] Applied new configuration: log=debug cache=true
```

---

## 18.2 Secrets

### 18.2.1 Purpose: Store Sensitive Data

A **Secret** is similar to a ConfigMap but designed for sensitive data:
passwords, OAuth tokens, TLS certificates, SSH keys, and API keys.

Kubernetes Secrets provide:

- Separate storage from Pod specs and container images.
- Access control via RBAC (who can read which Secrets).
- Volume mounts as `tmpfs` (RAM-backed) so secrets are never written to disk
  on the node.

### 18.2.2 Secret Types

Kubernetes defines several built-in Secret types:

| Type | Purpose | Required Keys |
|---|---|---|
| `Opaque` | Arbitrary user-defined data (default) | None |
| `kubernetes.io/tls` | TLS certificate and private key | `tls.crt`, `tls.key` |
| `kubernetes.io/dockerconfigjson` | Docker registry credentials | `.dockerconfigjson` |
| `kubernetes.io/basic-auth` | Basic authentication credentials | `username`, `password` |
| `kubernetes.io/ssh-auth` | SSH private key | `ssh-privatekey` |
| `kubernetes.io/service-account-token` | Service account token (auto-created) | `token`, `ca.crt`, `namespace` |

The type field helps Kubernetes validate that the required keys are present.

### 18.2.3 Base64 Encoding (NOT Encryption!)

Secret values in YAML manifests are **base64-encoded**, not encrypted:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=        # base64("admin")
  password: cEBzc3cwcmQh    # base64("p@ssw0rd!")
```

Anyone who can read the YAML can decode the values:

```bash
$ echo "cEBzc3cwcmQh" | base64 --decode
p@ssw0rd!
```

Base64 encoding exists to allow binary data (certificates, keys) to be
represented in YAML — it is **not a security mechanism**.

To avoid base64 encoding when creating Secrets, use `stringData`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
stringData:                  # ← plain text, encoded at creation time
  username: admin
  password: "p@ssw0rd!"
```

Kubernetes converts `stringData` to base64-encoded `data` when the object is
stored. You will never see `stringData` when you `kubectl get secret -o yaml`
— it is a write-only convenience field.

### 18.2.4 Creating Secrets

#### From Literals

```bash
$ kubectl create secret generic db-credentials \
    --from-literal=username=admin \
    --from-literal=password='p@ssw0rd!'
```

#### From Files

```bash
# Create a TLS secret from certificate and key files
$ kubectl create secret tls my-tls-secret \
    --cert=path/to/tls.crt \
    --key=path/to/tls.key

# Create a generic secret from files
$ kubectl create secret generic app-secrets \
    --from-file=api-key=./api-key.txt \
    --from-file=db-password=./db-pass.txt
```

#### From YAML

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
  namespace: production
type: Opaque
stringData:
  API_KEY: "sk-prod-abc123def456"
  DB_PASSWORD: "p@ssw0rd!"
  REDIS_URL: "redis://:secretpass@redis.production.svc.cluster.local:6379/0"
```

### 18.2.5 Consuming Secrets

#### As Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  containers:
    - name: backend
      image: myapp:1.0
      env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
```

> **Security note:** Environment variables can be leaked through process
> listings (`/proc/*/environ`), crash dumps, and logging. Prefer **volume
> mounts** for highly sensitive data.

#### As Volume Mounts

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: backend
spec:
  volumes:
    - name: secrets
      secret:
        secretName: db-credentials
        defaultMode: 0400      # read-only by owner
  containers:
    - name: backend
      image: myapp:1.0
      volumeMounts:
        - name: secrets
          mountPath: /etc/secrets
          readOnly: true
```

The files are mounted on a `tmpfs` filesystem (in-memory), so they are never
written to the node's physical disk.

```
/etc/secrets/
├── username     ← contains "admin"
└── password     ← contains "p@ssw0rd!"
```

#### As imagePullSecrets

Docker registry Secrets are referenced in the Pod spec to pull images from
private registries:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-app
spec:
  imagePullSecrets:
    - name: my-registry-secret
  containers:
    - name: app
      image: my-private-registry.com/myapp:1.0
```

### 18.2.6 Hands-on: TLS Secret with Certificate and Key

```bash
# Generate a self-signed certificate (for demonstration)
$ openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
    -keyout tls.key \
    -out tls.crt \
    -subj "/CN=myapp.example.com/O=MyOrg"

# Create the TLS secret
$ kubectl create secret tls myapp-tls \
    --cert=tls.crt \
    --key=tls.key

# Inspect the secret
$ kubectl get secret myapp-tls -o yaml
apiVersion: v1
kind: Secret
metadata:
  name: myapp-tls
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi...    # base64-encoded certificate
  tls.key: LS0tLS1CRUdJTi...    # base64-encoded private key

# Use in a Pod (e.g., for an HTTPS server)
$ kubectl apply -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: https-server
spec:
  volumes:
    - name: tls
      secret:
        secretName: myapp-tls
        defaultMode: 0400
  containers:
    - name: server
      image: myapp:1.0
      ports:
        - containerPort: 443
      volumeMounts:
        - name: tls
          mountPath: /etc/tls
          readOnly: true
      env:
        - name: TLS_CERT_FILE
          value: /etc/tls/tls.crt
        - name: TLS_KEY_FILE
          value: /etc/tls/tls.key
EOF
```

### 18.2.7 Hands-on: Docker Registry Secret

```bash
# Create a Docker registry secret
$ kubectl create secret docker-registry my-registry-secret \
    --docker-server=my-private-registry.com \
    --docker-username=myuser \
    --docker-password='MyP@ssw0rd' \
    --docker-email=myuser@example.com

# Inspect the structure
$ kubectl get secret my-registry-secret -o jsonpath='{.data.\.dockerconfigjson}' | base64 -d | jq .
{
  "auths": {
    "my-private-registry.com": {
      "username": "myuser",
      "password": "MyP@ssw0rd",
      "email": "myuser@example.com",
      "auth": "bXl1c2VyOk15UEBzc3cwcmQ="
    }
  }
}

# Reference in a Deployment
$ kubectl apply -f - <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: private-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: private-app
  template:
    metadata:
      labels:
        app: private-app
    spec:
      imagePullSecrets:
        - name: my-registry-secret
      containers:
        - name: app
          image: my-private-registry.com/myapp:1.0
EOF
```

### 18.2.8 Hands-on: Mounting Secrets as Files in a Go Application

```go
// File: cmd/secret-reader/main.go
//
// Package main demonstrates reading Kubernetes Secrets that are mounted
// as files in a Pod's filesystem. This is the recommended approach for
// consuming sensitive configuration because:
//
//  1. Files are mounted on tmpfs (RAM) — never written to disk.
//  2. File permissions can be restricted (mode 0400 = owner read-only).
//  3. Changes are reflected automatically (unlike env vars which are fixed
	//     at Pod startup).
//  4. No risk of leaking through process listings or crash dumps.
//
// The Secret is mounted as a volume in the Pod spec:
//
//	volumes:
//	  - name: db-creds
//	    secret:
//	      secretName: db-credentials
//	      defaultMode: 0400
//	containers:
//	  - volumeMounts:
//	      - name: db-creds
//	        mountPath: /etc/secrets/db
//	        readOnly: true
package main

import (
	"database/sql"
	"fmt"
	"os"
	"strings"

	_ "github.com/lib/pq"
)

// readSecretFile reads a Kubernetes Secret value from a mounted file.
// Secret volumes mount each key as a separate file. This function reads
// the file content and trims any trailing whitespace/newlines that may
// be present in the source ConfigMap or Secret.
//
// Parameters:
//   - path: The filesystem path to the Secret file (e.g., /etc/secrets/db/password).
//
// Returns:
//   - string: The secret value with whitespace trimmed.
//   - error: Non-nil if the file cannot be read (e.g., missing mount, wrong permissions).
func readSecretFile(path string) (string, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return "", fmt.Errorf("reading secret from %s: %w", path, err)
	}
	// Trim whitespace — editors and 'kubectl create secret --from-file'
	// sometimes add trailing newlines.
	return strings.TrimSpace(string(data)), nil
}

// buildDSN constructs a PostgreSQL data source name (DSN) from individual
// secret files. Each credential component (username, password) is stored
// in a separate file under the secret mount path.
//
// Parameters:
//   - secretDir: The directory where the Secret is mounted (e.g., /etc/secrets/db).
//   - host:      The database hostname (typically from a ConfigMap, not a Secret).
//   - port:      The database port.
//   - dbName:    The database name.
//
// Returns:
//   - string: A PostgreSQL connection string.
//   - error: Non-nil if any required secret file cannot be read.
//
// The resulting DSN format:
//
//	host=<host> port=<port> user=<username> password=<password> dbname=<dbname> sslmode=require
func buildDSN(secretDir, host, port, dbName string) (string, error) {
	username, err := readSecretFile(secretDir + "/username")
	if err != nil {
		return "", fmt.Errorf("reading username: %w", err)
	}

	password, err := readSecretFile(secretDir + "/password")
	if err != nil {
		return "", fmt.Errorf("reading password: %w", err)
	}

	dsn := fmt.Sprintf(
		"host=%s port=%s user=%s password=%s dbname=%s sslmode=require",
		host, port, username, password, dbName,
	)
	return dsn, nil
}

func main() {
	// Configuration from ConfigMap (non-sensitive).
	dbHost := os.Getenv("DB_HOST")
	if dbHost == "" {
		dbHost = "postgres.default.svc.cluster.local"
	}
	dbPort := os.Getenv("DB_PORT")
	if dbPort == "" {
		dbPort = "5432"
	}
	dbName := os.Getenv("DB_NAME")
	if dbName == "" {
		dbName = "myapp"
	}

	// Secret mount path (from volume mount).
	secretDir := os.Getenv("DB_SECRET_DIR")
	if secretDir == "" {
		secretDir = "/etc/secrets/db"
	}

	fmt.Println("=== Secret Reader ===")
	fmt.Printf("Database: %s:%s/%s\n", dbHost, dbPort, dbName)
	fmt.Printf("Secrets from: %s\n", secretDir)

	// Build the connection string from mounted secret files.
	dsn, err := buildDSN(secretDir, dbHost, dbPort, dbName)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error building DSN: %v\n", err)
		os.Exit(1)
	}

	// Connect to the database. In production, you would use connection
	// pooling and health checks.
	fmt.Println("Connecting to database...")
	db, err := sql.Open("postgres", dsn)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error opening database: %v\n", err)
		os.Exit(1)
	}
	defer db.Close()

	// Verify the connection.
	if err := db.Ping(); err != nil {
		fmt.Fprintf(os.Stderr, "Error pinging database: %v\n", err)
		os.Exit(1)
	}

	fmt.Println("Successfully connected to database!")
}
```

---

## 18.3 Security Concerns with Secrets

### 18.3.1 Secrets in etcd: Base64, Not Encrypted by Default

By default, Kubernetes stores Secrets in etcd with only **base64 encoding**.
Anyone with access to the etcd data directory (or etcd API) can read all
Secrets in plain text.

```bash
# If you have etcd access, you can read any secret:
$ ETCDCTL_API=3 etcdctl get /registry/secrets/production/db-credentials | hexdump -C
# ... you'll see the base64-encoded values in the raw etcd data ...
```

This is a significant security risk in production clusters.

### 18.3.2 Encryption at Rest — EncryptionConfiguration

Kubernetes supports **encryption at rest** for Secrets (and other resources)
stored in etcd. You configure this on the API server:

```yaml
# File: /etc/kubernetes/encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      # First provider is used for encryption; all are tried for decryption
      - aescbc:
          keys:
            - name: key1
              # 32-byte base64-encoded key for AES-256
              secret: dGhpcyBpcyBhIDMyLWJ5dGUga2V5IGZvciBhZXMtY2Jj
      - identity: {}    # fallback: allows reading unencrypted secrets
```

Configure the API server to use it:

```yaml
# kube-apiserver arguments
spec:
  containers:
    - command:
        - kube-apiserver
        - --encryption-provider-config=/etc/kubernetes/encryption-config.yaml
      volumeMounts:
        - name: encryption-config
          mountPath: /etc/kubernetes/encryption-config.yaml
          readOnly: true
```

Available encryption providers:

| Provider | Algorithm | Key Management | Use Case |
|---|---|---|---|
| `identity` | None (plaintext) | N/A | Default, insecure |
| `aescbc` | AES-CBC with PKCS#7 padding | Static key in config | Simple, self-managed |
| `aesgcm` | AES-GCM | Static key in config | Faster but requires key rotation (nonce reuse risk) |
| `secretbox` | XSalsa20 + Poly1305 | Static key in config | Strong authenticated encryption |
| `kms` v1/v2 | Delegates to external KMS | AWS KMS, GCP KMS, Azure Key Vault, HashiCorp Vault | Production recommended |

### 18.3.3 RBAC for Secrets

Control who can access Secrets with fine-grained RBAC:

```yaml
# Role that allows reading only specific secrets
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: secret-reader
  namespace: production
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["db-credentials", "api-keys"]    # specific secrets only
    verbs: ["get"]
  # Note: "list" and "watch" are intentionally excluded.
  # "list" would allow reading ALL secret data in the namespace.
```

**Best practices for Secret RBAC:**

1. Never grant `list` or `watch` on Secrets unless absolutely necessary —
   these verbs return the full secret data.
2. Use `resourceNames` to restrict access to specific Secrets.
3. Avoid granting `create` or `update` to prevent privilege escalation
   (a user could create a Secret and then mount it in a Pod they control).
4. Audit Secret access using Kubernetes audit logging.

### 18.3.4 Secret Rotation Strategies

Secrets should be rotated regularly. There are several strategies:

**1. Dual-Secret Rotation (Zero-Downtime)**

```
Step 1: Create new secret (v2) alongside old secret (v1).
Step 2: Update the application to accept both v1 and v2 credentials.
Step 3: Update external systems to use v2 credentials.
Step 4: Remove v1 credentials.
```

**2. Versioned ConfigMaps/Secrets with Rolling Updates**

```yaml
# Version your secrets
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials-v3    # ← versioned name
type: Opaque
stringData:
  username: admin
  password: new-rotated-password-v3
---
# Deployment references the specific version
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  template:
    spec:
      volumes:
        - name: db-creds
          secret:
            secretName: db-credentials-v3    # ← triggers rolling update
```

**3. External Secret Operators**

Tools like **External Secrets Operator** or **Vault Agent Injector** sync
secrets from external vaults (HashiCorp Vault, AWS Secrets Manager, GCP Secret
Manager) into Kubernetes Secrets automatically, handling rotation transparently.

### 18.3.5 Hands-on: Enabling Encryption at Rest

```bash
# Generate a 32-byte encryption key
$ head -c 32 /dev/urandom | base64
dGhpcyBpcyBhIDMyLWJ5dGUga2V5IGZvciBhZXMtY2Jj

# Create the EncryptionConfiguration file on the control plane node
$ cat > /etc/kubernetes/encryption-config.yaml <<'EOF'
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: dGhpcyBpcyBhIDMyLWJ5dGUga2V5IGZvciBhZXMtY2Jj
      - identity: {}
EOF

# Add the flag to the API server manifest
# (For kubeadm clusters, edit /etc/kubernetes/manifests/kube-apiserver.yaml)
$ vi /etc/kubernetes/manifests/kube-apiserver.yaml
# Add: --encryption-provider-config=/etc/kubernetes/encryption-config.yaml

# Wait for the API server to restart
$ kubectl get pods -n kube-system | grep kube-apiserver

# Verify encryption is working — re-encrypt all existing secrets
$ kubectl get secrets --all-namespaces -o json | \
    kubectl replace -f -

# Verify a secret is encrypted in etcd
$ ETCDCTL_API=3 etcdctl get /registry/secrets/default/db-credentials \
    --cacert=/etc/kubernetes/pki/etcd/ca.crt \
    --cert=/etc/kubernetes/pki/etcd/server.crt \
    --key=/etc/kubernetes/pki/etcd/server.key | hexdump -C
# Should see encrypted blob starting with "k8s:enc:aescbc:v1:key1:"
```

> **Reference:** Chapter 30 covers comprehensive secrets management with
> external vaults, PKI infrastructure, and automated rotation.

---

## 18.4 Downward API

The **Downward API** exposes Pod and container metadata to the running
container. This lets your application discover information about itself without
calling the Kubernetes API.

### 18.4.1 Available Fields

#### Pod-level fields (fieldRef):

| Field | Description | Example Value |
|---|---|---|
| `metadata.name` | Pod name | `backend-abc123` |
| `metadata.namespace` | Pod namespace | `production` |
| `metadata.uid` | Pod UID | `f4c4a1b2-3d4e-5f6a-7b8c-9d0e1f2a3b4c` |
| `metadata.labels` | All labels (volume only) | `app=backend\ntier=api` |
| `metadata.annotations` | All annotations (volume only) | `commit=abc123` |
| `spec.nodeName` | Node the Pod runs on | `worker-3` |
| `spec.serviceAccountName` | Service account name | `backend-sa` |
| `status.podIP` | Pod IP address | `10.244.1.5` |
| `status.hostIP` | Node IP address | `192.168.1.103` |

#### Container-level fields (resourceFieldRef):

| Field | Description | Example Value |
|---|---|---|
| `requests.cpu` | CPU request | `500m` |
| `requests.memory` | Memory request | `256Mi` |
| `limits.cpu` | CPU limit | `1000m` |
| `limits.memory` | Memory limit | `512Mi` |
| `requests.ephemeral-storage` | Ephemeral storage request | `1Gi` |
| `limits.ephemeral-storage` | Ephemeral storage limit | `2Gi` |

### 18.4.2 Consuming via Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-env
  labels:
    app: backend
    version: v2
  annotations:
    commit: "abc123"
spec:
  containers:
    - name: app
      image: myapp:1.0
      resources:
        requests:
          cpu: 500m
          memory: 256Mi
        limits:
          cpu: "1"
          memory: 512Mi
      env:
        # Pod metadata
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: HOST_IP
          valueFrom:
            fieldRef:
              fieldPath: status.hostIP
        # Container resources
        - name: CPU_REQUEST
          valueFrom:
            resourceFieldRef:
              containerName: app
              resource: requests.cpu
        - name: MEM_LIMIT
          valueFrom:
            resourceFieldRef:
              containerName: app
              resource: limits.memory
```

### 18.4.3 Consuming via Volume Mounts

For labels and annotations (which can be multi-line), use volume mounts:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-vol
  labels:
    app: backend
    version: v2
  annotations:
    commit: "abc123"
    team: "platform"
spec:
  volumes:
    - name: pod-info
      downwardAPI:
        items:
          - path: "labels"
            fieldRef:
              fieldPath: metadata.labels
          - path: "annotations"
            fieldRef:
              fieldPath: metadata.annotations
          - path: "name"
            fieldRef:
              fieldPath: metadata.name
          - path: "namespace"
            fieldRef:
              fieldPath: metadata.namespace
          - path: "cpu-request"
            resourceFieldRef:
              containerName: app
              resource: requests.cpu
              divisor: "1m"          # express in millicores
          - path: "mem-limit"
            resourceFieldRef:
              containerName: app
              resource: limits.memory
              divisor: "1Mi"         # express in MiB
  containers:
    - name: app
      image: myapp:1.0
      resources:
        requests:
          cpu: 500m
          memory: 256Mi
        limits:
          cpu: "1"
          memory: 512Mi
      volumeMounts:
        - name: pod-info
          mountPath: /etc/pod-info
```

The resulting files:

```
/etc/pod-info/
├── labels          → "app=\"backend\"\nversion=\"v2\""
├── annotations     → "commit=\"abc123\"\nteam=\"platform\""
├── name            → "downward-vol"
├── namespace       → "default"
├── cpu-request     → "500"     (millicores)
└── mem-limit       → "512"     (MiB)
```

> **Advantage of volume mounts:** Labels and annotations update automatically
> when they change (e.g., if a controller adds a label). Environment variables
> are fixed at Pod startup.

### 18.4.4 Hands-on: Go App Reading Its Own Pod Name and Namespace

```go
// File: cmd/pod-info/main.go
//
// Package main demonstrates using the Kubernetes Downward API to read
// Pod metadata from within a running container. The Downward API injects
// Pod information as environment variables or files, allowing the
// application to identify itself without calling the Kubernetes API.
//
// Common use cases for reading Pod metadata:
//   - Logging: include pod name and namespace in structured log entries
//     for debugging and correlation.
//   - Metrics: tag Prometheus metrics with pod name, namespace, and node
//     for aggregation and alerting.
//   - Registration: register the pod with an external service discovery
//     system using its IP and identity.
//   - Resource awareness: adjust concurrency or batch sizes based on the
//     CPU/memory limits allocated to this container.
//
// Deployment requires the Downward API fields to be configured in the Pod spec:
//
//	env:
//	  - name: POD_NAME
//	    valueFrom:
//	      fieldRef:
//	        fieldPath: metadata.name
//	  - name: POD_NAMESPACE
//	    valueFrom:
//	      fieldRef:
//	        fieldPath: metadata.namespace
//	  - name: POD_IP
//	    valueFrom:
//	      fieldRef:
//	        fieldPath: status.podIP
//	  - name: NODE_NAME
//	    valueFrom:
//	      fieldRef:
//	        fieldPath: spec.nodeName
//	  - name: CPU_LIMIT
//	    valueFrom:
//	      resourceFieldRef:
//	        resource: limits.cpu
//	  - name: MEM_LIMIT
//	    valueFrom:
//	      resourceFieldRef:
//	        resource: limits.memory
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"os"
	"runtime"
	"strconv"
)

// PodIdentity holds metadata about the Pod this application is running in.
// All fields are populated from Downward API environment variables.
//
// This struct is designed to be serialized to JSON for health check
// endpoints, structured logging, and service registration.
type PodIdentity struct {
	// Name is the Pod name (e.g., "backend-7d4f5b6c8-abc12").
	// Source: metadata.name
	Name string `json:"name"`

	// Namespace is the Kubernetes namespace (e.g., "production").
	// Source: metadata.namespace
	Namespace string `json:"namespace"`

	// IP is the Pod's cluster-internal IP address (e.g., "10.244.1.5").
	// Source: status.podIP
	IP string `json:"ip"`

	// NodeName is the name of the node this Pod runs on (e.g., "worker-3").
	// Source: spec.nodeName
	NodeName string `json:"nodeName"`

	// CPULimit is the CPU limit in millicores (e.g., 1000 for 1 CPU).
	// Source: limits.cpu (parsed from Downward API string)
	CPULimit int64 `json:"cpuLimitMillicores"`

	// MemoryLimit is the memory limit in bytes.
	// Source: limits.memory (parsed from Downward API string)
	MemoryLimit int64 `json:"memoryLimitBytes"`
}

// newPodIdentityFromEnv constructs a PodIdentity by reading Downward API
// environment variables. Returns an error if any required variable is missing.
//
// Required environment variables:
//   - POD_NAME:      The Pod's name (metadata.name)
//   - POD_NAMESPACE: The Pod's namespace (metadata.namespace)
//
// Optional environment variables (defaults provided):
//   - POD_IP:    The Pod's IP (status.podIP)
//   - NODE_NAME: The node name (spec.nodeName)
//   - CPU_LIMIT: CPU limit in cores (limits.cpu)
//   - MEM_LIMIT: Memory limit in bytes (limits.memory)
//
// Returns:
//   - *PodIdentity: Populated identity struct.
//   - error: Non-nil if required variables are missing.
func newPodIdentityFromEnv() (*PodIdentity, error) {
	name := os.Getenv("POD_NAME")
	if name == "" {
		return nil, fmt.Errorf(
			"POD_NAME environment variable not set; " +
			"configure the Downward API in your Pod spec")
	}

	namespace := os.Getenv("POD_NAMESPACE")
	if namespace == "" {
		return nil, fmt.Errorf(
			"POD_NAMESPACE environment variable not set; " +
			"configure the Downward API in your Pod spec")
	}

	identity := &PodIdentity{
		Name:      name,
		Namespace: namespace,
		IP:        os.Getenv("POD_IP"),
		NodeName:  os.Getenv("NODE_NAME"),
	}

	// Parse optional resource limits. The Downward API expresses CPU
	// limits in cores (e.g., "1" for 1000m) and memory in bytes.
	if cpuStr := os.Getenv("CPU_LIMIT"); cpuStr != "" {
		cpu, err := strconv.ParseInt(cpuStr, 10, 64)
		if err == nil {
			identity.CPULimit = cpu
		}
	}

	if memStr := os.Getenv("MEM_LIMIT"); memStr != "" {
		mem, err := strconv.ParseInt(memStr, 10, 64)
		if err == nil {
			identity.MemoryLimit = mem
		}
	}

	return identity, nil
}

func main() {
	identity, err := newPodIdentityFromEnv()
	if err != nil {
		log.Fatalf("Failed to read pod identity: %v", err)
	}

	fmt.Println("=== Pod Identity (Downward API) ===")
	fmt.Printf("Pod:       %s/%s\n", identity.Namespace, identity.Name)
	fmt.Printf("IP:        %s\n", identity.IP)
	fmt.Printf("Node:      %s\n", identity.NodeName)
	fmt.Printf("CPU limit: %d millicores\n", identity.CPULimit)
	fmt.Printf("Mem limit: %d bytes (%.0f MiB)\n",
		identity.MemoryLimit,
		float64(identity.MemoryLimit)/(1024*1024))
	fmt.Printf("GOMAXPROCS: %d\n", runtime.GOMAXPROCS(0))
	fmt.Println()

	// Serve a /healthz endpoint that returns the Pod identity.
	// This is useful for debugging and verifying the Downward API
	// configuration from outside the Pod.
	http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
			w.Header().Set("Content-Type", "application/json")
			json.NewEncoder(w).Encode(identity)
		})

	addr := ":8080"
	fmt.Printf("Listening on %s\n", addr)
	log.Fatal(http.ListenAndServe(addr, nil))
}
```

Deploy with Downward API configured:

```yaml
# File: pod-info-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pod-info
spec:
  replicas: 2
  selector:
    matchLabels:
      app: pod-info
  template:
    metadata:
      labels:
        app: pod-info
    spec:
      containers:
        - name: app
          image: myregistry/pod-info:latest
          ports:
            - containerPort: 8080
          resources:
            requests:
              cpu: 250m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 256Mi
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: POD_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: CPU_LIMIT
              valueFrom:
                resourceFieldRef:
                  resource: limits.cpu
            - name: MEM_LIMIT
              valueFrom:
                resourceFieldRef:
                  resource: limits.memory
```

```bash
$ kubectl apply -f pod-info-deployment.yaml

$ kubectl port-forward deployment/pod-info 8080:8080

$ curl localhost:8080/healthz | jq .
{
  "name": "pod-info-7d4f5b6c8-abc12",
  "namespace": "default",
  "ip": "10.244.1.5",
  "nodeName": "worker-3",
  "cpuLimitMillicores": 500,
  "memoryLimitBytes": 268435456
}
```

---

## 18.5 Environment Variable Patterns

### 18.5.1 Static Environment Variables

The simplest pattern — hard-coded in the Pod spec:

```yaml
env:
  - name: APP_ENV
    value: production
  - name: LOG_FORMAT
    value: json
  - name: GOMAXPROCS
    value: "4"
```

### 18.5.2 ConfigMap-Sourced Environment Variables

Reference individual keys from a ConfigMap:

```yaml
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: DB_HOST
  - name: DB_PORT
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: DB_PORT
```

### 18.5.3 Secret-Sourced Environment Variables

Reference individual keys from a Secret:

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: password
  - name: API_KEY
    valueFrom:
      secretKeyRef:
        name: api-keys
        key: external-api-key
        optional: true     # Pod starts even if the secret/key doesn't exist
```

### 18.5.4 envFrom — Inject All Keys

Load all keys from a ConfigMap and/or Secret as environment variables:

```yaml
envFrom:
  # All keys from the ConfigMap become env vars
  - configMapRef:
      name: app-config
    prefix: CONFIG_          # optional prefix

  # All keys from the Secret become env vars
  - secretRef:
      name: app-secrets
    prefix: SECRET_          # optional prefix
```

If a key appears in both the ConfigMap and the Secret (with the same prefix),
the **Secret value wins** (last one in the list takes precedence).

### 18.5.5 Dependent Environment Variables

You can reference other environment variables using `$(VAR_NAME)`:

```yaml
env:
  - name: DB_HOST
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: DB_HOST
  - name: DB_PORT
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: DB_PORT
  - name: DB_USERNAME
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: username
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-credentials
        key: password
  # Compose a connection string from other variables
  - name: DATABASE_URL
    value: "postgres://$(DB_USERNAME):$(DB_PASSWORD)@$(DB_HOST):$(DB_PORT)/myapp?sslmode=require"
```

> **Note:** The `$(VAR_NAME)` substitution happens at Pod creation time, not at
> runtime. The variable must be defined *earlier* in the same `env` list (or be
> defined by `envFrom`). If the referenced variable is not found, the literal
> string `$(VAR_NAME)` is used.

To use a literal `$(...)` without substitution, use `$$(...)`:

```yaml
env:
  - name: PATTERN
    value: "$$(NOT_A_VARIABLE)"    # results in literal "$(NOT_A_VARIABLE)"
```

### 18.5.6 Hands-on: Full Environment Configuration Example

```yaml
# File: full-env-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: backend-config
data:
  APP_ENV: production
  DB_HOST: postgres.production.svc.cluster.local
  DB_PORT: "5432"
  DB_NAME: myapp
  CACHE_HOST: redis.production.svc.cluster.local
  CACHE_PORT: "6379"
  LOG_LEVEL: info
  LOG_FORMAT: json
  MAX_CONNECTIONS: "100"
---
apiVersion: v1
kind: Secret
metadata:
  name: backend-secrets
type: Opaque
stringData:
  DB_USERNAME: app_user
  DB_PASSWORD: "s3cur3-p@ss!"
  CACHE_PASSWORD: "r3d1s-p@ss!"
  API_KEY: "sk-prod-abc123"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      containers:
        - name: backend
          image: myregistry/backend:1.0
          ports:
            - containerPort: 8080
          envFrom:
            # Inject all ConfigMap keys as env vars
            - configMapRef:
                name: backend-config
            # Inject all Secret keys as env vars
            - secretRef:
                name: backend-secrets
          env:
            # Downward API
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            # Composed variables (depend on envFrom values)
            - name: DATABASE_URL
              value: "postgres://$(DB_USERNAME):$(DB_PASSWORD)@$(DB_HOST):$(DB_PORT)/$(DB_NAME)?sslmode=require"
            - name: REDIS_URL
              value: "redis://:$(CACHE_PASSWORD)@$(CACHE_HOST):$(CACHE_PORT)/0"
```

```bash
$ kubectl apply -f full-env-config.yaml

# Verify the environment inside a Pod
$ kubectl exec -it deploy/backend -- env | sort
API_KEY=sk-prod-abc123
APP_ENV=production
CACHE_HOST=redis.production.svc.cluster.local
CACHE_PASSWORD=r3d1s-p@ss!
CACHE_PORT=6379
DATABASE_URL=postgres://app_user:s3cur3-p@ss!@postgres.production.svc.cluster.local:5432/myapp?sslmode=require
DB_HOST=postgres.production.svc.cluster.local
DB_NAME=myapp
DB_PASSWORD=s3cur3-p@ss!
DB_PORT=5432
DB_USERNAME=app_user
LOG_FORMAT=json
LOG_LEVEL=info
MAX_CONNECTIONS=100
POD_NAME=backend-7d4f5b6c8-abc12
POD_NAMESPACE=production
REDIS_URL=redis://:r3d1s-p@ss!@redis.production.svc.cluster.local:6379/0
```

---

## 18.6 Projected Volumes

A **projected volume** combines data from multiple sources into a single volume
mount. This is useful when your application expects all configuration in one
directory.

### 18.6.1 Supported Sources

Projected volumes can combine:

- **ConfigMaps** — non-sensitive configuration files.
- **Secrets** — sensitive data (passwords, keys).
- **Downward API** — Pod metadata.
- **Service Account Tokens** — projected (bound) service account tokens with
  audience and expiration.

### 18.6.2 Hands-on: Projected Volume Example

```yaml
# File: projected-volume.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  app.yaml: |
    server:
      port: 8080
    logging:
      level: info
---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  db-password: "s3cur3-p@ss!"
  api-key: "sk-prod-abc123"
---
apiVersion: v1
kind: Pod
metadata:
  name: projected-test
  labels:
    app: projected
    version: v3
  annotations:
    team: platform
    owner: alice
spec:
  serviceAccountName: default
  containers:
    - name: app
      image: busybox
      command:
        - sh
        - -c
        - |
          echo "=== Projected Volume Contents ==="
          echo ""
          echo "--- ConfigMap (app.yaml) ---"
          cat /etc/app/config/app.yaml
          echo ""
          echo "--- Secret (db-password) ---"
          cat /etc/app/secrets/db-password
          echo ""
          echo "--- Secret (api-key) ---"
          cat /etc/app/secrets/api-key
          echo ""
          echo "--- Downward API (labels) ---"
          cat /etc/app/pod-info/labels
          echo ""
          echo "--- Downward API (annotations) ---"
          cat /etc/app/pod-info/annotations
          echo ""
          echo "--- Service Account Token ---"
          cat /etc/app/token/token | head -c 50
          echo "..."
          echo ""
          sleep 3600
      resources:
        requests:
          cpu: 100m
          memory: 64Mi
        limits:
          cpu: 200m
          memory: 128Mi
      volumeMounts:
        - name: all-in-one
          mountPath: /etc/app
          readOnly: true
  volumes:
    - name: all-in-one
      projected:
        defaultMode: 0440
        sources:
          # ConfigMap files go under config/
          - configMap:
              name: app-config
              items:
                - key: app.yaml
                  path: config/app.yaml

          # Secret files go under secrets/
          - secret:
              name: app-secrets
              items:
                - key: db-password
                  path: secrets/db-password
                  mode: 0400
                - key: api-key
                  path: secrets/api-key
                  mode: 0400

          # Downward API files go under pod-info/
          - downwardAPI:
              items:
                - path: pod-info/labels
                  fieldRef:
                    fieldPath: metadata.labels
                - path: pod-info/annotations
                  fieldRef:
                    fieldPath: metadata.annotations
                - path: pod-info/cpu-request
                  resourceFieldRef:
                    containerName: app
                    resource: requests.cpu
                    divisor: "1m"

          # Projected service account token with custom audience
          # and expiration. This is more secure than the legacy
          # auto-mounted token because it has a bounded lifetime.
          - serviceAccountToken:
              path: token/token
              audience: "api.myapp.com"
              expirationSeconds: 3600
```

```bash
$ kubectl apply -f projected-volume.yaml
$ kubectl logs projected-test
=== Projected Volume Contents ===

--- ConfigMap (app.yaml) ---
server:
  port: 8080
logging:
  level: info

--- Secret (db-password) ---
s3cur3-p@ss!

--- Secret (api-key) ---
sk-prod-abc123

--- Downward API (labels) ---
app="projected"
version="v3"

--- Downward API (annotations) ---
owner="alice"
team="platform"

--- Service Account Token ---
eyJhbGciOiJSUzI1NiIsImtpZCI6IjEyMzQ1Njc4...
```

The resulting directory tree inside the Pod:

```
/etc/app/
├── config/
│   └── app.yaml          ← from ConfigMap
├── secrets/
│   ├── db-password       ← from Secret (mode 0400)
│   └── api-key           ← from Secret (mode 0400)
├── pod-info/
│   ├── labels            ← from Downward API
│   ├── annotations       ← from Downward API
│   └── cpu-request       ← from Downward API
└── token/
    └── token             ← projected SA token (audience: api.myapp.com)
```

---

## 18.7 Configuration Best Practices

### 18.7.1 The 12-Factor App Configuration Model

The **Twelve-Factor App** methodology (Factor III: Config) states:

> Store config in the environment. Config is everything that varies between
> deploys (staging, production, developer environments).

In Kubernetes, this translates to:

1. **Build once, deploy everywhere:** The container image should not contain
   environment-specific configuration.
2. **Externalize all configuration:** Use ConfigMaps for non-sensitive config
   and Secrets for sensitive data.
3. **Never hard-code:** No database URLs, API keys, or feature flags in code.

### 18.7.2 Configuration Hierarchy

Real applications need layered configuration with precedence:

```
Highest priority
    ▲
    │  Command-line flags     (--log-level=debug)
    │  Environment variables  (LOG_LEVEL=debug)
    │  ConfigMap files        (/etc/config/app.yaml)
    │  Defaults in code       (level = "info")
    ▼
Lowest priority
```

This allows:

- **Sensible defaults** in code (works out of the box).
- **ConfigMaps** for per-environment settings (staging vs. production).
- **Environment variables** for quick overrides (debugging in production).
- **Flags** for one-off runs (local development, testing).

### 18.7.3 Hands-on: Building a Go App with Layered Configuration Using Viper

[Viper](https://github.com/spf13/viper) is the most popular Go library for
layered configuration. It supports:

- Defaults
- Configuration files (YAML, JSON, TOML, HCL, envfile, properties)
- Environment variables
- Command-line flags (via pflag)
- Remote config (etcd, Consul)
- Live watching and re-reading of config files

```go
// File: cmd/layered-config/main.go
//
// Package main demonstrates a production-ready layered configuration
// pattern for Kubernetes applications using Viper. Configuration is
// loaded from multiple sources with clear precedence:
//
//  1. Defaults (hard-coded in the application)
//  2. Configuration file (from a mounted ConfigMap)
//  3. Environment variables (from Pod spec, ConfigMap, or Secret)
//  4. Command-line flags (for local development and testing)
//
// This approach follows the 12-factor app methodology while providing
// a great developer experience:
//
//   - In production: ConfigMap provides the base config, Secrets provide
//     credentials via env vars, and flags are not used.
//   - In development: Defaults work out of the box, a local config file
//     can override them, and flags provide quick overrides.
//
// Usage:
//
//	# Production (in a Pod with ConfigMap mounted at /etc/config/):
//	CONFIG_FILE=/etc/config/app.yaml ./layered-config
//
//	# Development (local):
//	./layered-config --log-level=debug --db-host=localhost
//
//	# Override via env var:
//	APP_LOG_LEVEL=debug ./layered-config
package main

import (
	"encoding/json"
	"fmt"
	"log"
	"net/http"
	"os"
	"strings"
	"sync"

	"github.com/fsnotify/fsnotify"
	"github.com/spf13/pflag"
	"github.com/spf13/viper"
)

// Config holds the complete application configuration. All fields are
// populated by Viper from the configuration hierarchy.
//
// Struct tags:
//   - `mapstructure`: Used by Viper to map configuration keys to struct fields.
//   - `json`: Used when serializing the config to JSON (e.g., for /config endpoint).
//
// The configuration hierarchy (highest priority first):
//  1. Command-line flags:   --server.port=9090
//  2. Environment variables: APP_SERVER_PORT=9090
//  3. Config file:          server.port: 9090 (in app.yaml)
//  4. Defaults:             server.port: 8080 (hard-coded)
type Config struct {
	// Server contains HTTP server settings.
	Server ServerConfig `mapstructure:"server" json:"server"`

	// Database contains PostgreSQL connection settings.
	// Sensitive fields (username, password) are typically provided
	// via environment variables sourced from Kubernetes Secrets.
	Database DatabaseConfig `mapstructure:"database" json:"database"`

	// Logging controls log output behavior.
	Logging LoggingConfig `mapstructure:"logging" json:"logging"`

	// Features contains boolean feature flags that can be toggled
	// at runtime by updating the ConfigMap (if file watching is enabled).
	Features FeatureConfig `mapstructure:"features" json:"features"`
}

// ServerConfig holds HTTP server configuration.
type ServerConfig struct {
	// Host is the address to bind to (e.g., "0.0.0.0" or "127.0.0.1").
	Host string `mapstructure:"host" json:"host"`

	// Port is the TCP port to listen on.
	Port int `mapstructure:"port" json:"port"`

	// ReadTimeoutSec is the maximum duration in seconds for reading
	// the entire request, including the body.
	ReadTimeoutSec int `mapstructure:"readTimeoutSec" json:"readTimeoutSec"`

	// WriteTimeoutSec is the maximum duration in seconds before
	// timing out writes of the response.
	WriteTimeoutSec int `mapstructure:"writeTimeoutSec" json:"writeTimeoutSec"`
}

// DatabaseConfig holds PostgreSQL connection settings.
type DatabaseConfig struct {
	// Host is the database server hostname.
	Host string `mapstructure:"host" json:"host"`

	// Port is the database server port.
	Port int `mapstructure:"port" json:"port"`

	// Name is the database name to connect to.
	Name string `mapstructure:"name" json:"name"`

	// Username for database authentication.
	// In Kubernetes, typically sourced from a Secret via env var.
	Username string `mapstructure:"username" json:"username"`

	// Password for database authentication.
	// SECURITY: This field is omitted from JSON serialization to prevent
	// accidental exposure in health check endpoints or logs.
	Password string `mapstructure:"password" json:"-"`

	// MaxOpenConns is the maximum number of open connections in the pool.
	MaxOpenConns int `mapstructure:"maxOpenConns" json:"maxOpenConns"`

	// MaxIdleConns is the maximum number of idle connections in the pool.
	MaxIdleConns int `mapstructure:"maxIdleConns" json:"maxIdleConns"`
}

// LoggingConfig controls log output.
type LoggingConfig struct {
	// Level is the minimum log level (debug, info, warn, error).
	Level string `mapstructure:"level" json:"level"`

	// Format is the log output format (json or text).
	Format string `mapstructure:"format" json:"format"`
}

// FeatureConfig holds feature flags that can be toggled at runtime
// by updating the ConfigMap.
type FeatureConfig struct {
	// EnableCache enables the in-memory response cache.
	EnableCache bool `mapstructure:"enableCache" json:"enableCache"`

	// EnableMetrics enables the Prometheus /metrics endpoint.
	EnableMetrics bool `mapstructure:"enableMetrics" json:"enableMetrics"`

	// EnableTracing enables distributed tracing (OpenTelemetry).
	EnableTracing bool `mapstructure:"enableTracing" json:"enableTracing"`
}

// configHolder provides thread-safe access to the current configuration.
// When the config file changes (ConfigMap update), the config is reloaded
// and swapped atomically using a RWMutex.
//
// Readers use Get() which acquires a read lock (allows concurrent reads).
// The reload callback uses Set() which acquires a write lock (exclusive).
type configHolder struct {
	mu     sync.RWMutex
	config Config
}

// Get returns a copy of the current configuration. It is safe to call
// from multiple goroutines concurrently.
//
// Returns a copy (not a pointer) so callers cannot accidentally mutate
// the shared configuration.
func (h *configHolder) Get() Config {
	h.mu.RLock()
	defer h.mu.RUnlock()
	return h.config
}

// Set atomically replaces the current configuration. This is called
// when the ConfigMap file changes and Viper reloads the config.
//
// Parameters:
//   - cfg: The new configuration to store.
func (h *configHolder) Set(cfg Config) {
	h.mu.Lock()
	defer h.mu.Unlock()
	h.config = cfg
}

// setDefaults configures sensible default values for all configuration
// keys. These defaults are used when no other source provides a value.
//
// Defaults should make the application work out of the box for local
// development. Production values should come from ConfigMaps.
func setDefaults() {
	// Server defaults — suitable for local development.
	viper.SetDefault("server.host", "0.0.0.0")
	viper.SetDefault("server.port", 8080)
	viper.SetDefault("server.readTimeoutSec", 30)
	viper.SetDefault("server.writeTimeoutSec", 30)

	// Database defaults — localhost for local development.
	viper.SetDefault("database.host", "localhost")
	viper.SetDefault("database.port", 5432)
	viper.SetDefault("database.name", "myapp")
	viper.SetDefault("database.username", "postgres")
	viper.SetDefault("database.password", "postgres")
	viper.SetDefault("database.maxOpenConns", 25)
	viper.SetDefault("database.maxIdleConns", 5)

	// Logging defaults.
	viper.SetDefault("logging.level", "info")
	viper.SetDefault("logging.format", "json")

	// Feature flag defaults — conservative (most features off).
	viper.SetDefault("features.enableCache", false)
	viper.SetDefault("features.enableMetrics", true)
	viper.SetDefault("features.enableTracing", false)
}

// bindFlags registers command-line flags and binds them to Viper keys.
// Flags have the highest priority in the configuration hierarchy.
//
// Flag names use dot notation to match Viper's nested key structure:
//
//	--server.port=9090     → viper.Get("server.port") = 9090
//	--database.host=mydb   → viper.Get("database.host") = "mydb"
//	--logging.level=debug  → viper.Get("logging.level") = "debug"
func bindFlags() {
	pflag.String("server.host", "", "Server bind address")
	pflag.Int("server.port", 0, "Server port")
	pflag.String("database.host", "", "Database host")
	pflag.Int("database.port", 0, "Database port")
	pflag.String("database.name", "", "Database name")
	pflag.String("logging.level", "", "Log level (debug, info, warn, error)")
	pflag.String("logging.format", "", "Log format (json, text)")
	pflag.Bool("features.enableCache", false, "Enable response caching")
	pflag.Bool("features.enableMetrics", true, "Enable Prometheus metrics")
	pflag.Bool("features.enableTracing", false, "Enable distributed tracing")

	pflag.Parse()
	_ = viper.BindPFlags(pflag.CommandLine)
}

// bindEnvVars configures Viper to read environment variables with the
// prefix "APP_". Dots in key names are replaced with underscores:
//
//	server.port      → APP_SERVER_PORT
//	database.host    → APP_DATABASE_HOST
//	logging.level    → APP_LOGGING_LEVEL
//
// Additionally, some keys are bound to specific environment variable
// names that match Kubernetes conventions (e.g., DB_HOST from a ConfigMap).
func bindEnvVars() {
	// Set the prefix for automatic env binding.
	viper.SetEnvPrefix("APP")

	// Replace dots with underscores in env var names.
	// "server.port" → "APP_SERVER_PORT"
	viper.SetEnvKeyReplacer(strings.NewReplacer(".", "_"))

	// Enable automatic env binding for all keys.
	viper.AutomaticEnv()

	// Bind specific env vars from Kubernetes ConfigMaps/Secrets.
	// These follow Kubernetes naming conventions (uppercase, underscores).
	_ = viper.BindEnv("database.host", "DB_HOST")
	_ = viper.BindEnv("database.port", "DB_PORT")
	_ = viper.BindEnv("database.name", "DB_NAME")
	_ = viper.BindEnv("database.username", "DB_USERNAME")
	_ = viper.BindEnv("database.password", "DB_PASSWORD")
}

// loadConfigFile loads the configuration file specified by the CONFIG_FILE
// environment variable (or the default path). Viper supports YAML, JSON,
// TOML, HCL, and envfile formats.
//
// In Kubernetes, the config file is typically a ConfigMap mounted as a volume:
//
//	volumes:
//	  - name: config
//	    configMap:
//	      name: app-config
//	containers:
//	  - volumeMounts:
//	      - name: config
//	        mountPath: /etc/config
//	    env:
//	      - name: CONFIG_FILE
//	        value: /etc/config/app.yaml
//
// If the config file does not exist, Viper silently falls back to defaults
// and environment variables — this is intentional for local development
// where a config file may not be present.
func loadConfigFile() {
	configFile := os.Getenv("CONFIG_FILE")
	if configFile != "" {
		viper.SetConfigFile(configFile)
	} else {
		// Default locations for local development.
		viper.SetConfigName("app")
		viper.SetConfigType("yaml")
		viper.AddConfigPath("/etc/config")
		viper.AddConfigPath(".")
	}

	if err := viper.ReadInConfig(); err != nil {
		if _, ok := err.(viper.ConfigFileNotFoundError); ok {
			fmt.Println("[config] No config file found — using defaults and env vars")
		} else {
			fmt.Printf("[config] Warning: error reading config file: %v\n", err)
		}
	} else {
		fmt.Printf("[config] Loaded config from: %s\n", viper.ConfigFileUsed())
	}
}

func main() {
	fmt.Println("=== Layered Configuration Demo ===")
	fmt.Println()

	// Initialize configuration in priority order (lowest first).
	// Each layer can override values from the previous layer.

	// Layer 1: Defaults (lowest priority).
	setDefaults()

	// Layer 2: Config file (from mounted ConfigMap).
	loadConfigFile()

	// Layer 3: Environment variables.
	bindEnvVars()

	// Layer 4: Command-line flags (highest priority).
	bindFlags()

	// Unmarshal the merged configuration into a typed struct.
	var config Config
	if err := viper.Unmarshal(&config); err != nil {
		log.Fatalf("Error unmarshaling config: %v", err)
	}

	// Store in a thread-safe holder for live reloading.
	holder := &configHolder{}
	holder.Set(config)

	// Print the resolved configuration.
	fmt.Println("Resolved configuration:")
	fmt.Printf("  Server:   %s:%d\n", config.Server.Host, config.Server.Port)
	fmt.Printf("  Database: %s:%d/%s (user: %s)\n",
		config.Database.Host, config.Database.Port,
		config.Database.Name, config.Database.Username)
	fmt.Printf("  Logging:  level=%s format=%s\n",
		config.Logging.Level, config.Logging.Format)
	fmt.Printf("  Features: cache=%v metrics=%v tracing=%v\n",
		config.Features.EnableCache,
		config.Features.EnableMetrics,
		config.Features.EnableTracing)
	fmt.Println()

	// Set up live config reloading. When the ConfigMap is updated,
	// Kubernetes atomically swaps the mounted files, and Viper detects
	// the change via fsnotify.
	viper.OnConfigChange(func(e fsnotify.Event) {
		fmt.Printf("[config] Config file changed: %s\n", e.Name)

		var newConfig Config
		if err := viper.Unmarshal(&newConfig); err != nil {
			fmt.Printf("[config] Error reloading: %v\n", err)
			return
		}

		holder.Set(newConfig)
		fmt.Printf("[config] Reloaded: level=%s cache=%v tracing=%v\n",
			newConfig.Logging.Level,
			newConfig.Features.EnableCache,
			newConfig.Features.EnableTracing)
	})
	viper.WatchConfig()

	// Start the HTTP server with endpoints that use the live config.
	mux := http.NewServeMux()

	// /config — returns the current (possibly reloaded) configuration.
	// Useful for debugging and verifying ConfigMap updates.
	mux.HandleFunc("/config", func(w http.ResponseWriter, r *http.Request) {
		w.Header().Set("Content-Type", "application/json")
		currentConfig := holder.Get()
		json.NewEncoder(w).Encode(currentConfig)
	})

	// /healthz — a simple health check endpoint.
	mux.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		fmt.Fprintln(w, "ok")
	})

	addr := fmt.Sprintf("%s:%d", config.Server.Host, config.Server.Port)
	fmt.Printf("Listening on %s\n", addr)
	fmt.Println("Try: curl http://localhost:8080/config | jq .")
	log.Fatal(http.ListenAndServe(addr, mux))
}
```

Deploy to Kubernetes:

```yaml
# File: layered-config-deploy.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: layered-app-config
data:
  app.yaml: |
    server:
      host: 0.0.0.0
      port: 8080
      readTimeoutSec: 30
      writeTimeoutSec: 30
    database:
      host: postgres.production.svc.cluster.local
      port: 5432
      name: myapp
      maxOpenConns: 50
      maxIdleConns: 10
    logging:
      level: info
      format: json
    features:
      enableCache: true
      enableMetrics: true
      enableTracing: false
---
apiVersion: v1
kind: Secret
metadata:
  name: layered-app-secrets
type: Opaque
stringData:
  DB_USERNAME: app_user
  DB_PASSWORD: "pr0d-s3cur3-p@ss!"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: layered-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: layered-app
  template:
    metadata:
      labels:
        app: layered-app
    spec:
      volumes:
        - name: config
          configMap:
            name: layered-app-config
      containers:
        - name: app
          image: myregistry/layered-config:latest
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: config
              mountPath: /etc/config
          env:
            # Point the app to the mounted config file
            - name: CONFIG_FILE
              value: /etc/config/app.yaml
            # Database credentials from Secret
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: layered-app-secrets
                  key: DB_USERNAME
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: layered-app-secrets
                  key: DB_PASSWORD
            # Downward API
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
```

Test the live reload:

```bash
$ kubectl apply -f layered-config-deploy.yaml

# Check current config
$ kubectl port-forward deploy/layered-app 8080:8080 &
$ curl localhost:8080/config | jq .
{
  "server": {"host": "0.0.0.0", "port": 8080, ...},
  "database": {"host": "postgres.production.svc.cluster.local", ...},
  "logging": {"level": "info", "format": "json"},
  "features": {"enableCache": true, "enableMetrics": true, "enableTracing": false}
}

# Update the ConfigMap — change log level and enable tracing
$ kubectl edit configmap layered-app-config
# Change: level: info → level: debug
# Change: enableTracing: false → enableTracing: true

# Wait ~60 seconds for kubelet sync, then check again
$ curl localhost:8080/config | jq .logging
{
  "level": "debug",
  "format": "json"
}
# It updated without a Pod restart!
```

---

## 18.8 Summary

| Concept | Key Takeaway |
|---|---|
| **ConfigMap** | Key-value store for non-sensitive configuration; consumed as env vars, volumes, or args |
| **Immutable ConfigMap** | `immutable: true` — reduces API server load and prevents accidental changes |
| **Secret** | Like ConfigMap but for sensitive data; mounted on tmpfs, base64-encoded (not encrypted) |
| **Secret types** | `Opaque`, `tls`, `dockerconfigjson`, `basic-auth`, `ssh-auth`, `service-account-token` |
| **Encryption at rest** | `EncryptionConfiguration` + KMS provider for production Secret security |
| **Downward API** | Expose Pod metadata (name, namespace, labels, IP, resource limits) to containers |
| **envFrom** | Inject all keys from a ConfigMap/Secret as environment variables at once |
| **$(VAR_NAME)** | Compose environment variables from other variables at Pod creation time |
| **Projected volumes** | Combine ConfigMap + Secret + Downward API + SA token in a single mount |
| **Live reload** | Volume-mounted ConfigMaps auto-update; use `fsnotify` to detect and apply changes |
| **Layered config** | Defaults → file → env vars → flags; Viper implements this pattern elegantly |

---

## 18.9 Exercises

1. **ConfigMap lifecycle:** Create a ConfigMap with three keys. Mount it as a
   volume. Update one key using `kubectl edit`. Verify that the file in the
   Pod updates. Time how long it takes.

2. **Immutable vs. mutable:** Create an immutable ConfigMap. Try to update it
   with `kubectl edit` and observe the error. Create a new version and update
   your Deployment to reference it.

3. **Secret security audit:** Create a Secret. Read it from the etcd data
   store (if you have access). Then enable encryption at rest and verify the
   Secret is encrypted in etcd.

4. **Downward API labels:** Create a Pod with the Downward API volume mount
   for labels. Add a new label to the running Pod using `kubectl label`.
   Verify the label file inside the Pod updates.

5. **Projected volume:** Create a projected volume that combines a ConfigMap,
   a Secret, Downward API metadata, and a projected service account token.
   Verify all files are present and correctly populated.

6. **Layered config app:** Build the Viper-based Go application from §18.7.3.
   Deploy it with a ConfigMap. Override a value with an environment variable.
   Override another with a command-line flag. Verify the priority order.

7. **Live reload race condition:** Deploy the config watcher from §18.1.7
   with 10 replicas. Update the ConfigMap rapidly (5 times in 10 seconds).
   Verify that all replicas converge to the same configuration.

---

*Next chapter: Ingress, Gateway API & Traffic Routing — exposing Services to
the outside world with HTTP-aware routing, TLS termination, and the new
Gateway API.*
