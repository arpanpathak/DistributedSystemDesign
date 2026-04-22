# Chapter 34: Building a Kubernetes Operator in Go

> *"An operator is a software extension to Kubernetes that makes use of custom resources to manage applications and their components. Operators follow Kubernetes principles, notably the control loop."*
> — Kubernetes Documentation

In previous chapters we explored the Kubernetes API, client-go, custom resources, and controller
fundamentals. Now we bring it all together. In this chapter we build a **complete, production-grade
Kubernetes Operator** from scratch — a `PostgresCluster` operator that manages PostgreSQL database
instances declaratively on Kubernetes.

This is not a toy example. By the end of this chapter you will have a working operator that creates
StatefulSets, headless Services, ConfigMaps, handles scaling, manages version upgrades, reports
status conditions, and cleans up after itself. The patterns you learn here apply to any operator
you will ever build — whether for databases, message queues, monitoring stacks, or internal
platform tooling.

---

## 34.1 What Is an Operator?

### 34.1.1 The Operator Pattern

A Kubernetes **Operator** is the combination of three things:

1. **A Custom Resource Definition (CRD)** — extends the Kubernetes API with a new resource type
   (e.g., `PostgresCluster`).
2. **A Controller** — watches that custom resource and reconciles the cluster toward the desired
   state declared in the resource's `spec`.
3. **Domain Knowledge** — the controller encodes operational expertise that a human SRE would
   otherwise perform manually: provisioning, scaling, upgrading, backing up, failover, and
   decommissioning.

The core insight of the Operator pattern is:

```
Desired State (spec) + Current State (status + cluster) → Actions (reconcile)
```

The reconciler is called repeatedly. Each time, it reads the desired state from the custom resource,
observes the current state of the world, and takes the minimal set of actions to bring reality
closer to the desired state. This loop is **level-triggered** (reacts to the current state, not
edge-triggered events), which makes it naturally idempotent and self-healing.

### 34.1.2 Why Operators Exist

Before operators, running stateful workloads on Kubernetes required:

- **Helm charts** that could install resources but couldn't react to failures
- **Manual runbooks** that operators (humans) followed for day-2 operations
- **Custom scripts** glued together with shell and cron jobs

Operators encode all of this knowledge in software that runs inside Kubernetes itself. They turn
imperative runbooks into declarative intent:

```yaml
# Instead of: "SSH into the server, download PostgreSQL 16.2, edit the config..."
# You declare:
apiVersion: postgres.example.com/v1alpha1
kind: PostgresCluster
metadata:
  name: my-database
spec:
  version: "16.2"
  replicas: 3
  storage:
    size: 100Gi
```

The operator handles everything else.

### 34.1.3 Real-World Operator Examples

| Operator | What It Manages | Key Operations |
|----------|----------------|----------------|
| **CrunchyData PGO** | PostgreSQL clusters | Provisioning, HA, backups, restores |
| **cert-manager** | TLS certificates | Issuance, renewal, integration with ACME/Vault |
| **Prometheus Operator** | Monitoring stack | ServiceMonitor discovery, Prometheus/Alertmanager lifecycle |
| **Strimzi** | Apache Kafka | Broker management, topic management, user management |
| **ArgoCD** | GitOps deployments | Application sync, health checks, rollback |

All of these follow the same pattern we will implement in this chapter.

### 34.1.4 Operator Maturity Model

The Operator Framework defines five levels of maturity:

| Level | Capability | Example |
|-------|-----------|---------|
| **1** | Basic install | Helm-like: create resources from a template |
| **2** | Seamless upgrades | Handle version upgrades, schema migrations |
| **3** | Full lifecycle | Backup, restore, failure recovery |
| **4** | Deep insights | Metrics, alerts, log aggregation |
| **5** | Auto-pilot | Auto-scaling, auto-tuning, anomaly detection |

Our PostgresCluster operator will reach **Level 2–3** by the end of this chapter, with hooks for
Level 4 in Chapter 35.

---

## 34.2 Project Setup with Kubebuilder

### 34.2.1 What Is Kubebuilder?

**Kubebuilder** is the standard scaffolding tool for building Kubernetes operators in Go. It
generates:

- CRD type definitions with marker comments for code generation
- Controller scaffolding with the reconcile loop
- RBAC manifests
- Webhook scaffolding
- A `Makefile` with targets for generating code, building, testing, and deploying
- Integration test setup with `envtest`

Kubebuilder builds on top of **controller-runtime**, the Go library that provides the `Manager`,
`Controller`, `Reconciler`, and `Client` abstractions we explored in earlier chapters.

### 34.2.2 Installing Kubebuilder

```bash
# Download the latest release (Linux/macOS)
curl -L -o kubebuilder \
  "https://go.kubebuilder.io/dl/latest/$(go env GOOS)/$(go env GOARCH)"
chmod +x kubebuilder
sudo mv kubebuilder /usr/local/bin/

# Verify installation
kubebuilder version
```

### 34.2.3 Initializing the Project

```bash
# Create and enter the project directory
mkdir postgres-operator && cd postgres-operator

# Initialize a new kubebuilder project
# --domain: your organization's domain (used in API group names)
# --repo: the Go module path
kubebuilder init \
  --domain example.com \
  --repo github.com/example/postgres-operator

# Create the API (CRD + Controller)
# --group: API group (e.g., "postgres")
# --version: API version (start with v1alpha1)
# --kind: the resource kind (e.g., "PostgresCluster")
kubebuilder create api \
  --group postgres \
  --version v1alpha1 \
  --kind PostgresCluster \
  --resource true \
  --controller true
```

When you run `kubebuilder create api` with `--resource true --controller true`, it generates both
the type definitions and the controller scaffolding and asks you to confirm.

### 34.2.4 Project Structure

After scaffolding, your project looks like this:

```
postgres-operator/
├── api/
│   └── v1alpha1/
│       ├── groupversion_info.go    # GV registration
│       ├── postgrescluster_types.go # CRD type definitions ← we edit this
│       └── zz_generated.deepcopy.go # auto-generated DeepCopy methods
├── cmd/
│   └── main.go                     # Entrypoint: sets up Manager, registers controllers
├── config/
│   ├── crd/                        # Generated CRD YAML manifests
│   ├── default/                    # Kustomize base
│   ├── manager/                    # Deployment for the operator
│   ├── rbac/                       # RBAC manifests
│   └── samples/                    # Example CR YAML
├── internal/
│   └── controller/
│       ├── postgrescluster_controller.go  # Reconciler ← we edit this
│       └── suite_test.go                  # envtest setup
├── Dockerfile                      # Multi-stage build
├── Makefile                        # Build targets
├── go.mod
└── go.sum
```

### 34.2.5 Key Makefile Targets

```makefile
# Generate DeepCopy methods and CRD manifests
make generate    # runs controller-gen object paths=./...
make manifests   # runs controller-gen rbac:... crd:... paths=./...

# Build and run
make build       # go build -o bin/manager cmd/main.go
make run         # go run cmd/main.go (runs against current kubeconfig)

# Test
make test        # go test ./... (uses envtest)

# Docker
make docker-build IMG=myregistry/postgres-operator:v0.1.0
make docker-push  IMG=myregistry/postgres-operator:v0.1.0

# Deploy to cluster
make install     # Install CRDs into the cluster
make deploy      IMG=myregistry/postgres-operator:v0.1.0
```

### 34.2.6 The Generated main.go

The entrypoint sets up the controller-runtime `Manager` and registers our controller:

```go
// Package main is the entrypoint for the postgres-operator.
// It initializes the controller-runtime Manager, registers the
// PostgresCluster reconciler, and starts the manager's control loop.
package main

import (
	"crypto/tls"
	"flag"
	"os"

	// Import all Kubernetes client auth plugins (e.g., GCP, Azure, OIDC)
	_ "k8s.io/client-go/plugin/pkg/client/auth"

	"k8s.io/apimachinery/pkg/runtime"
	utilruntime "k8s.io/apimachinery/pkg/util/runtime"
	clientgoscheme "k8s.io/client-go/kubernetes/scheme"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/healthz"
	"sigs.k8s.io/controller-runtime/pkg/log/zap"
	metricsserver "sigs.k8s.io/controller-runtime/pkg/metrics/server"
	ctrlwebhook "sigs.k8s.io/controller-runtime/pkg/webhook"

	postgresv1alpha1 "github.com/example/postgres-operator/api/v1alpha1"
	"github.com/example/postgres-operator/internal/controller"
)

var (
	// scheme is the runtime scheme that maps Go types to GVKs.
	// Both built-in Kubernetes types and our custom types are registered here.
	scheme   = runtime.NewScheme()
	setupLog = ctrl.Log.WithName("setup")
)

func init() {
	// Register the core Kubernetes types (Pod, Service, etc.)
	utilruntime.Must(clientgoscheme.AddToScheme(scheme))

	// Register our custom PostgresCluster types
	utilruntime.Must(postgresv1alpha1.AddToScheme(scheme))
}

func main() {
	// Command-line flags for operator configuration
	var metricsAddr string
	var enableLeaderElection bool
	var probeAddr string
	var secureMetrics bool
	var enableHTTP2 bool

	flag.StringVar(&metricsAddr, "metrics-bind-address", "0",
		"The address the metrics endpoint binds to. "+
		"Use :8443 for HTTPS with /metrics on all interfaces.")
	flag.StringVar(&probeAddr, "health-probe-bind-address", ":8081",
		"The address the health probe endpoint binds to.")
	flag.BoolVar(&enableLeaderElection, "leader-elect", false,
		"Enable leader election for controller manager. "+
		"Enabling this will ensure there is only one active controller manager.")
	flag.BoolVar(&secureMetrics, "metrics-secure", true,
		"If set, the metrics endpoint is served securely via HTTPS.")
	flag.BoolVar(&enableHTTP2, "enable-http2", false,
		"If set, HTTP/2 will be enabled for the metrics and webhook servers.")
	opts := zap.Options{Development: true}
	opts.BindFlags(flag.CommandLine)
	flag.Parse()

	ctrl.SetLogger(zap.New(zap.UseFlagOptions(&opts)))

	// TLS configuration — disable HTTP/2 by default to mitigate CVEs
	tlsOpts := []func(*tls.Config){}
	if !enableHTTP2 {
		tlsOpts = append(tlsOpts, func(c *tls.Config) {
				c.NextProtos = []string{"http/1.1"}
			})
	}

	// Create the Manager. The Manager:
	// - Creates a shared cache (informers) for all watched resources
	// - Provides a client for reading/writing Kubernetes resources
	// - Manages controller lifecycle (start, stop, health)
	// - Handles leader election
	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
			Scheme: scheme,
			Metrics: metricsserver.Options{
				BindAddress:   metricsAddr,
				SecureServing: secureMetrics,
				TLSOpts:       tlsOpts,
			},
			WebhookServer: ctrlwebhook.NewServer(ctrlwebhook.Options{
					TLSOpts: tlsOpts,
				}),
			HealthProbeBindAddress: probeAddr,
			LeaderElection:         enableLeaderElection,
			LeaderElectionID:       "postgres-operator.example.com",
		})
	if err != nil {
		setupLog.Error(err, "unable to start manager")
		os.Exit(1)
	}

	// Register the PostgresCluster reconciler with the Manager.
	// The reconciler is the core of our operator — it watches PostgresCluster
	// resources and reconciles the cluster toward the desired state.
	if err = (&controller.PostgresClusterReconciler{
			Client:   mgr.GetClient(),
			Scheme:   mgr.GetScheme(),
			Recorder: mgr.GetEventRecorderFor("postgrescluster-controller"),
		}).SetupWithManager(mgr); err != nil {
		setupLog.Error(err, "unable to create controller", "controller", "PostgresCluster")
		os.Exit(1)
	}

	// Health and readiness probes
	if err := mgr.AddHealthzCheck("healthz", healthz.Ping); err != nil {
		setupLog.Error(err, "unable to set up health check")
		os.Exit(1)
	}
	if err := mgr.AddReadyzCheck("readyz", healthz.Ping); err != nil {
		setupLog.Error(err, "unable to set up ready check")
		os.Exit(1)
	}

	setupLog.Info("starting manager")
	if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
		setupLog.Error(err, "problem running manager")
		os.Exit(1)
	}
}
```

---

## 34.3 Defining the API (CRD Types)

The API types define the contract between your users and your operator. Users write YAML that
conforms to your spec; your operator reads it and acts. Getting the API right is the single most
important design decision in an operator.

### 34.3.1 Design Principles for CRD APIs

1. **Declarative, not imperative** — the spec describes *what* the user wants, not *how* to get
   there.
2. **Sensible defaults** — users should only need to specify what they care about.
3. **Status reflects reality** — the status subresource reports what the operator observes, not
   what was requested.
4. **Conditions over phases** — use an array of conditions (each independently true/false) rather
   than a single phase string.
5. **Versioning** — start with `v1alpha1`, graduate to `v1beta1`, then `v1` as the API stabilizes.

### 34.3.2 The PostgresCluster Types

Edit `api/v1alpha1/postgrescluster_types.go`:

```go
// Package v1alpha1 contains API Schema definitions for the postgres v1alpha1 API group.
// This package defines the PostgresCluster custom resource, which declaratively
// manages PostgreSQL database clusters on Kubernetes. The types here are used by
// controller-gen to produce CRD manifests, DeepCopy methods, and RBAC rules.
//
// +kubebuilder:object:generate=true
// +groupName=postgres.example.com
package v1alpha1

import (
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/resource"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// PostgresClusterSpec defines the desired state of a PostgreSQL cluster.
// Users create a PostgresCluster resource with this spec, and the operator
// reconciles the cluster to match. All fields have sensible defaults so
// that a minimal spec (just a name) produces a working single-node cluster.
type PostgresClusterSpec struct {
	// Version specifies the PostgreSQL major.minor version to deploy.
	// The operator maps this to a container image tag.
	// Examples: "16.2", "15.6", "14.11"
	//
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:Pattern=`^\d+\.\d+$`
	// +kubebuilder:default="16.2"
	Version string `json:"version"`

	// Replicas is the desired number of PostgreSQL instances in the cluster.
	// The first replica (ordinal 0) is the primary; all others are streaming
	// replicas. Must be at least 1. For high availability, use 3 or more.
	//
	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=10
	// +kubebuilder:default=1
	Replicas int32 `json:"replicas"`

	// Storage configures the persistent storage for each PostgreSQL instance.
	// Each instance gets its own PersistentVolumeClaim via the StatefulSet's
	// volumeClaimTemplates.
	//
	// +kubebuilder:validation:Required
	Storage StorageSpec `json:"storage"`

	// Resources defines the CPU and memory requests/limits for each
	// PostgreSQL container. If not specified, no resource constraints
	// are applied (not recommended for production).
	//
	// +optional
	Resources corev1.ResourceRequirements `json:"resources,omitempty"`

	// Backup configures automated backup settings. If not specified,
	// no automated backups are performed.
	//
	// +optional
	Backup *BackupSpec `json:"backup,omitempty"`

	// Parameters is a map of PostgreSQL configuration parameters to set
	// in postgresql.conf. These override the operator's defaults.
	// Example: {"max_connections": "200", "shared_buffers": "256MB"}
	//
	// +optional
	Parameters map[string]string `json:"parameters,omitempty"`

	// ImagePullSecrets is a list of references to secrets in the same
	// namespace to use for pulling the PostgreSQL container image.
	//
	// +optional
	ImagePullSecrets []corev1.LocalObjectReference `json:"imagePullSecrets,omitempty"`

	// ServiceAccountName is the name of the ServiceAccount to use for
	// PostgreSQL Pods. If not set, the default ServiceAccount is used.
	//
	// +optional
	ServiceAccountName string `json:"serviceAccountName,omitempty"`
}

// StorageSpec defines the persistent storage configuration for PostgreSQL data.
// The operator creates a PersistentVolumeClaim for each replica using these settings.
type StorageSpec struct {
	// Size is the requested storage size for each PostgreSQL instance.
	// Uses Kubernetes resource quantity format (e.g., "10Gi", "500Mi").
	//
	// +kubebuilder:validation:Required
	// +kubebuilder:default="10Gi"
	Size resource.Quantity `json:"size"`

	// StorageClassName is the name of the StorageClass to use for dynamic
	// provisioning. If empty, the cluster's default StorageClass is used.
	//
	// +optional
	StorageClassName *string `json:"storageClassName,omitempty"`
}

// BackupSpec configures automated PostgreSQL backups.
// The operator uses pg_basebackup for physical backups and can integrate
// with external backup solutions via backup hooks.
type BackupSpec struct {
	// Schedule is a cron expression defining when backups run.
	// Uses standard 5-field cron syntax (minute hour day-of-month month day-of-week).
	// Example: "0 2 * * *" for daily at 2:00 AM UTC.
	//
	// +kubebuilder:validation:Required
	// +kubebuilder:validation:Pattern=`^(\S+\s){4}\S+$`
	Schedule string `json:"schedule"`

	// RetentionDays is the number of days to keep backups before they are
	// automatically deleted. Minimum 1, maximum 365.
	//
	// +kubebuilder:validation:Minimum=1
	// +kubebuilder:validation:Maximum=365
	// +kubebuilder:default=7
	RetentionDays int32 `json:"retentionDays"`
}

// PostgresClusterPhase represents the high-level lifecycle phase of a
// PostgresCluster. While conditions provide detailed boolean signals,
// the phase provides a quick human-readable summary.
// +kubebuilder:validation:Enum=Pending;Creating;Running;Scaling;Upgrading;Failed;Deleting
type PostgresClusterPhase string

const (
	// PhasePending indicates the cluster has been created but the operator
	// has not yet started reconciling it.
	PhasePending PostgresClusterPhase = "Pending"

	// PhaseCreating indicates the operator is creating the initial set of
	// resources (StatefulSet, Services, ConfigMaps).
	PhaseCreating PostgresClusterPhase = "Creating"

	// PhaseRunning indicates the cluster is healthy and all replicas are ready.
	PhaseRunning PostgresClusterPhase = "Running"

	// PhaseScaling indicates the cluster is adding or removing replicas.
	PhaseScaling PostgresClusterPhase = "Scaling"

	// PhaseUpgrading indicates the cluster is performing a rolling version upgrade.
	PhaseUpgrading PostgresClusterPhase = "Upgrading"

	// PhaseFailed indicates the cluster is in an error state that may require
	// manual intervention.
	PhaseFailed PostgresClusterPhase = "Failed"

	// PhaseDeleting indicates the cluster is being torn down (finalizer running).
	PhaseDeleting PostgresClusterPhase = "Deleting"
)

// PostgresClusterStatus defines the observed state of a PostgreSQL cluster.
// The status is updated by the operator during each reconcile loop and reflects
// the actual state of the cluster, not the desired state. Consumers (users,
	// monitoring systems, other controllers) read the status to understand what
// the cluster is actually doing.
type PostgresClusterStatus struct {
	// Phase is the high-level lifecycle phase of the cluster.
	// Use Conditions for detailed status; Phase is for human readability.
	//
	// +optional
	Phase PostgresClusterPhase `json:"phase,omitempty"`

	// Conditions represent the latest available observations of the cluster's
	// state. Each condition has a type (e.g., "Ready", "Available") and a
	// boolean status. Conditions are the primary mechanism for communicating
	// detailed cluster health.
	//
	// +optional
	// +listType=map
	// +listMapKey=type
	Conditions []metav1.Condition `json:"conditions,omitempty"`

	// ReadyReplicas is the number of PostgreSQL instances that are currently
	// ready to accept connections. This is observed from the StatefulSet status.
	//
	// +optional
	ReadyReplicas int32 `json:"readyReplicas,omitempty"`

	// CurrentReplicas is the total number of PostgreSQL instances that currently
	// exist, whether or not they are ready.
	//
	// +optional
	CurrentReplicas int32 `json:"currentReplicas,omitempty"`

	// CurrentVersion is the PostgreSQL version currently running in the cluster.
	// During an upgrade, this may differ from spec.version.
	//
	// +optional
	CurrentVersion string `json:"currentVersion,omitempty"`

	// ObservedGeneration is the most recent generation (metadata.generation)
	// observed by the operator. If this does not match metadata.generation,
	// the status is stale and a reconcile is in progress.
	//
	// +optional
	ObservedGeneration int64 `json:"observedGeneration,omitempty"`
}

// Condition type constants for PostgresCluster. These are the standard
// condition types that the operator sets on every PostgresCluster resource.
const (
	// ConditionTypeReady indicates that the cluster is fully reconciled and
	// all replicas are available. When Ready is True, the cluster is healthy.
	ConditionTypeReady = "Ready"

	// ConditionTypeAvailable indicates that at least one replica is accepting
	// connections. The cluster may not be fully healthy, but it is usable.
	ConditionTypeAvailable = "Available"

	// ConditionTypeInitialized indicates that the initial cluster setup
	// (StatefulSet, Services, ConfigMap) has been created successfully.
	ConditionTypeInitialized = "Initialized"

	// ConditionTypeProgressing indicates that the operator is actively
	// making changes (scaling, upgrading, etc.). When Progressing is False
	// and Ready is True, the cluster is stable.
	ConditionTypeProgressing = "Progressing"
)

// PostgresCluster is the Schema for the postgresclusters API.
// It represents a PostgreSQL database cluster managed by the operator.
// Users create PostgresCluster resources to declaratively manage PostgreSQL
// instances — the operator handles provisioning, scaling, upgrading,
// and health monitoring.
//
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:shortName=pg;pgcluster
// +kubebuilder:printcolumn:name="Version",type=string,JSONPath=`.spec.version`
// +kubebuilder:printcolumn:name="Replicas",type=integer,JSONPath=`.spec.replicas`
// +kubebuilder:printcolumn:name="Ready",type=integer,JSONPath=`.status.readyReplicas`
// +kubebuilder:printcolumn:name="Phase",type=string,JSONPath=`.status.phase`
// +kubebuilder:printcolumn:name="Age",type=date,JSONPath=`.metadata.creationTimestamp`
type PostgresCluster struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	// Spec is the desired state of the PostgreSQL cluster.
	Spec PostgresClusterSpec `json:"spec,omitempty"`

	// Status is the observed state of the PostgreSQL cluster,
	// updated by the operator during each reconciliation.
	Status PostgresClusterStatus `json:"status,omitempty"`
}

// PostgresClusterList contains a list of PostgresCluster resources.
// This type is required by the Kubernetes API machinery for list operations.
//
// +kubebuilder:object:root=true
type PostgresClusterList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`

	// Items is the list of PostgresCluster resources.
	Items []PostgresCluster `json:"items"`
}

func init() {
	// Register the PostgresCluster and PostgresClusterList types with the
	// scheme builder so they can be serialized/deserialized by the API machinery.
	SchemeBuilder.Register(&PostgresCluster{}, &PostgresClusterList{})
}
```

### 34.3.3 Understanding the Kubebuilder Markers

The comments beginning with `+kubebuilder:` are **marker comments** processed by `controller-gen`.
They drive code generation:

| Marker | Purpose |
|--------|---------|
| `+kubebuilder:object:root=true` | Marks this as a root API object (gets its own CRD) |
| `+kubebuilder:subresource:status` | Enables the `/status` subresource (separate RBAC for spec vs status) |
| `+kubebuilder:resource:shortName=pg` | Allows `kubectl get pg` as a shortcut |
| `+kubebuilder:printcolumn:...` | Adds columns to `kubectl get` output |
| `+kubebuilder:validation:Required` | Field is required in the CRD schema |
| `+kubebuilder:validation:Minimum=1` | Numeric minimum validation |
| `+kubebuilder:validation:Pattern=...` | Regex validation |
| `+kubebuilder:default=...` | Default value if not specified by the user |

After editing the types, run:

```bash
# Regenerate DeepCopy methods
make generate

# Regenerate CRD manifests, RBAC rules
make manifests
```

### 34.3.4 The Generated CRD YAML

Running `make manifests` produces `config/crd/bases/postgres.example.com_postgresclusters.yaml`.
This is the CRD that you install into the cluster with `kubectl apply -f` or `make install`. It
contains the full OpenAPI v3 schema derived from your Go struct tags and kubebuilder markers.

### 34.3.5 Sample Custom Resource

The scaffolding also produces a sample CR in `config/samples/`. Here is what a full example looks
like:

```yaml
# config/samples/postgres_v1alpha1_postgrescluster.yaml
apiVersion: postgres.example.com/v1alpha1
kind: PostgresCluster
metadata:
  name: my-database
  namespace: default
spec:
  version: "16.2"
  replicas: 3
  storage:
    size: 50Gi
    storageClassName: fast-ssd
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2"
      memory: "4Gi"
  backup:
    schedule: "0 2 * * *"
    retentionDays: 14
  parameters:
    max_connections: "200"
    shared_buffers: "1GB"
    effective_cache_size: "3GB"
    work_mem: "16MB"
```

---

## 34.4 Implementing the Reconciler — Step by Step

The reconciler is the heart of the operator. It implements the `Reconcile` method of the
`reconcile.Reconciler` interface. Each time a `PostgresCluster` resource is created, updated, or
deleted — or any resource it owns changes — the reconciler is invoked.

We build the reconciler incrementally, one sub-resource at a time.

### 34.4.1 The Reconciler Struct

```go
// Package controller implements the reconciliation logic for PostgresCluster
// resources. The PostgresClusterReconciler watches for changes to PostgresCluster
// objects and reconciles the cluster toward the desired state by creating and
// updating Kubernetes resources (StatefulSets, Services, ConfigMaps).
package controller

import (
	"context"
	"fmt"
	"time"

	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/equality"
	apierrors "k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/api/meta"
	"k8s.io/apimachinery/pkg/api/resource"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/apimachinery/pkg/util/intstr"
	"k8s.io/client-go/tools/record"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/log"

	postgresv1alpha1 "github.com/example/postgres-operator/api/v1alpha1"
)

// finalizerName is the finalizer added to PostgresCluster resources.
// It prevents the resource from being deleted until the operator has
// performed cleanup (e.g., deleting external resources, revoking DNS).
const finalizerName = "postgres.example.com/finalizer"

// postgresImage returns the container image for a given PostgreSQL version.
// In production, this would be configurable via operator flags or a ConfigMap.
func postgresImage(version string) string {
	return fmt.Sprintf("postgres:%s", version)
}

// PostgresClusterReconciler reconciles a PostgresCluster object.
// It watches for changes to PostgresCluster custom resources and ensures
// that the actual state of the Kubernetes cluster matches the desired state
// declared in the resource's spec.
//
// The reconciler creates and manages:
//   - A StatefulSet for PostgreSQL Pods (primary + replicas)
//   - A headless Service for StatefulSet DNS (peer discovery)
//   - A ClusterIP Service for client connections
//   - A ConfigMap for PostgreSQL configuration files
//
// Each reconcile invocation is idempotent: calling it multiple times with
// the same inputs produces the same outputs with no side effects.
type PostgresClusterReconciler struct {
	// Client is the controller-runtime client for reading and writing
	// Kubernetes resources. It uses the Manager's shared cache for reads
	// and direct API calls for writes.
	client.Client

	// Scheme maps Go types to GroupVersionKinds. Required for setting
	// owner references (the scheme tells the API machinery what GVK
		// corresponds to our PostgresCluster Go type).
	Scheme *runtime.Scheme

	// Recorder is used to emit Kubernetes Events on the PostgresCluster
	// resource. Events are visible via `kubectl describe` and provide
	// an audit trail of operator actions.
	Recorder record.EventRecorder
}
```

### 34.4.2 Naming Conventions

All child resources use consistent naming derived from the PostgresCluster name:

```go
// resourceName returns the base name used for all child resources
// (StatefulSet, Services, ConfigMap) owned by this PostgresCluster.
// Using a consistent naming scheme makes it easy to correlate resources.
func resourceName(pg *postgresv1alpha1.PostgresCluster) string {
	return pg.Name
}

// headlessServiceName returns the name of the headless Service used for
// StatefulSet Pod DNS. The headless Service must exist before the StatefulSet
// so that Pods get stable DNS names like: <pod-name>.<service-name>.<namespace>.svc.cluster.local
func headlessServiceName(pg *postgresv1alpha1.PostgresCluster) string {
	return fmt.Sprintf("%s-headless", pg.Name)
}

// configMapName returns the name of the ConfigMap that holds the PostgreSQL
// configuration files (postgresql.conf, pg_hba.conf).
func configMapName(pg *postgresv1alpha1.PostgresCluster) string {
	return fmt.Sprintf("%s-config", pg.Name)
}

// labelsForCluster returns the standard set of labels applied to all resources
// owned by this PostgresCluster. These labels enable label-based selection
// and are required for the StatefulSet's pod selector.
func labelsForCluster(pg *postgresv1alpha1.PostgresCluster) map[string]string {
	return map[string]string{
		"app.kubernetes.io/name":       "postgresql",
		"app.kubernetes.io/instance":   pg.Name,
		"app.kubernetes.io/managed-by": "postgres-operator",
		"app.kubernetes.io/part-of":    pg.Name,
	}
}
```

### 34.4.3 Step 1 — Create the StatefulSet

The StatefulSet is the primary workload resource. It provides:

- **Stable Pod identities** — each Pod gets a deterministic name (e.g., `my-database-0`,
  `my-database-1`).
- **Stable persistent storage** — each Pod gets its own PVC that survives Pod rescheduling.
- **Ordered deployment** — Pods are created sequentially, which is important for primary/replica
  setup.

```go
// reconcileStatefulSet ensures that a StatefulSet exists for the PostgresCluster
// and that its spec matches the desired state. If the StatefulSet does not exist,
// it is created. If it exists but differs from the desired state, it is updated.
//
// The StatefulSet runs PostgreSQL containers with:
//   - A persistent volume for /var/lib/postgresql/data
//   - Configuration mounted from the ConfigMap
//   - Liveness and readiness probes
//   - The PostgreSQL container running as the postgres user (UID 999)
//
// Returns an error if the StatefulSet cannot be created or updated.
func (r *PostgresClusterReconciler) reconcileStatefulSet(
	ctx context.Context,
	pg *postgresv1alpha1.PostgresCluster,
) error {
	log := log.FromContext(ctx)

	// Build the desired StatefulSet from the PostgresCluster spec.
	desired := r.desiredStatefulSet(pg)

	// Set the owner reference so the StatefulSet is garbage collected
	// when the PostgresCluster is deleted.
	if err := controllerutil.SetControllerReference(pg, desired, r.Scheme); err != nil {
		return fmt.Errorf("setting owner reference on StatefulSet: %w", err)
	}

	// Check if the StatefulSet already exists.
	existing := &appsv1.StatefulSet{}
	err := r.Get(ctx, types.NamespacedName{
			Name:      desired.Name,
			Namespace: desired.Namespace,
		}, existing)

	if apierrors.IsNotFound(err) {
		// StatefulSet does not exist — create it.
		log.Info("creating StatefulSet",
			"name", desired.Name,
			"replicas", *desired.Spec.Replicas,
		)
		if err := r.Create(ctx, desired); err != nil {
			return fmt.Errorf("creating StatefulSet: %w", err)
		}
		r.Recorder.Eventf(pg, corev1.EventTypeNormal, "StatefulSetCreated",
			"Created StatefulSet %s with %d replicas", desired.Name, *desired.Spec.Replicas)
		return nil
	}
	if err != nil {
		return fmt.Errorf("getting StatefulSet: %w", err)
	}

	// StatefulSet exists — check if it needs updating.
	// We compare the fields we manage; we do not overwrite fields managed
	// by the StatefulSet controller itself (e.g., status, currentRevision).
	if r.statefulSetNeedsUpdate(existing, desired) {
		log.Info("updating StatefulSet",
			"name", existing.Name,
			"oldReplicas", *existing.Spec.Replicas,
			"newReplicas", *desired.Spec.Replicas,
		)
		existing.Spec.Replicas = desired.Spec.Replicas
		existing.Spec.Template = desired.Spec.Template
		if err := r.Update(ctx, existing); err != nil {
			return fmt.Errorf("updating StatefulSet: %w", err)
		}
		r.Recorder.Eventf(pg, corev1.EventTypeNormal, "StatefulSetUpdated",
			"Updated StatefulSet %s", existing.Name)
	}

	return nil
}

// statefulSetNeedsUpdate returns true if the existing StatefulSet's managed
// fields differ from the desired state. This avoids unnecessary API calls
// (updates are expensive — they trigger Pod rolling restarts if the template changes).
func (r *PostgresClusterReconciler) statefulSetNeedsUpdate(
	existing, desired *appsv1.StatefulSet,
) bool {
	// Check replica count
	if *existing.Spec.Replicas != *desired.Spec.Replicas {
		return true
	}

	// Check container image (version upgrade)
	if len(existing.Spec.Template.Spec.Containers) > 0 &&
	len(desired.Spec.Template.Spec.Containers) > 0 {
		if existing.Spec.Template.Spec.Containers[0].Image !=
		desired.Spec.Template.Spec.Containers[0].Image {
			return true
		}
	}

	// Check resource requirements
	if !equality.Semantic.DeepEqual(
		existing.Spec.Template.Spec.Containers[0].Resources,
		desired.Spec.Template.Spec.Containers[0].Resources,
	) {
		return true
	}

	return false
}

// desiredStatefulSet constructs the StatefulSet that should exist for the
// given PostgresCluster. This is the "desired state" — the reconciler
// compares this against what actually exists and takes corrective action.
//
// The StatefulSet template includes:
//   - A PostgreSQL container with configurable image, resources, and probes
//   - A volume mount for persistent data at /var/lib/postgresql/data
//   - A volume mount for configuration from the ConfigMap
//   - Pod anti-affinity to spread replicas across nodes
func (r *PostgresClusterReconciler) desiredStatefulSet(
	pg *postgresv1alpha1.PostgresCluster,
) *appsv1.StatefulSet {
	labels := labelsForCluster(pg)
	replicas := pg.Spec.Replicas

	// PostgreSQL data directory. Using a subdirectory of the mount point
	// because PostgreSQL requires the data directory to be empty on init.
	const pgDataDir = "/var/lib/postgresql/data/pgdata"

	sts := &appsv1.StatefulSet{
		ObjectMeta: metav1.ObjectMeta{
			Name:      resourceName(pg),
			Namespace: pg.Namespace,
			Labels:    labels,
		},
		Spec: appsv1.StatefulSetSpec{
			// ServiceName must match the headless Service for DNS.
			ServiceName: headlessServiceName(pg),
			Replicas:    &replicas,

			// The selector is immutable after creation. It must match
			// the Pod template labels exactly.
			Selector: &metav1.LabelSelector{
				MatchLabels: labels,
			},

			// RollingUpdate strategy ensures zero-downtime upgrades.
			// Pods are updated one at a time, starting from the highest
			// ordinal (replicas first, primary last).
			UpdateStrategy: appsv1.StatefulSetUpdateStrategy{
				Type: appsv1.RollingUpdateStatefulSetStrategyType,
			},

			// PodManagementPolicy controls Pod creation order.
			// OrderedReady: Pods are created sequentially (0, then 1, then 2...).
			// This is important for primary/replica setup.
			PodManagementPolicy: appsv1.OrderedReadyPodManagement,

			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: labels,
				},
				Spec: corev1.PodSpec{
					// TerminationGracePeriodSeconds gives PostgreSQL time to
					// shut down cleanly (checkpoint, close connections).
					TerminationGracePeriodSeconds: int64Ptr(30),

					// ServiceAccountName for Pod identity.
					ServiceAccountName: pg.Spec.ServiceAccountName,

					// ImagePullSecrets for private registries.
					ImagePullSecrets: pg.Spec.ImagePullSecrets,

					// SecurityContext at the Pod level: run as the postgres
					// user/group (UID/GID 999) and use a read-only root filesystem
					// where possible.
					SecurityContext: &corev1.PodSecurityContext{
						RunAsUser:  int64Ptr(999),
						RunAsGroup: int64Ptr(999),
						FSGroup:    int64Ptr(999),
					},

					// Pod anti-affinity ensures PostgreSQL replicas are
					// scheduled on different nodes for high availability.
					Affinity: &corev1.Affinity{
						PodAntiAffinity: &corev1.PodAntiAffinity{
							PreferredDuringSchedulingIgnoredDuringExecution: []corev1.WeightedPodAffinityTerm{
								{
									Weight: 100,
									PodAffinityTerm: corev1.PodAffinityTerm{
										LabelSelector: &metav1.LabelSelector{
											MatchLabels: labels,
										},
										TopologyKey: "kubernetes.io/hostname",
									},
								},
							},
						},
					},

					Containers: []corev1.Container{
						{
							Name:  "postgresql",
							Image: postgresImage(pg.Spec.Version),

							// PGDATA must point to a subdirectory of the volume mount.
							Env: []corev1.EnvVar{
								{
									Name:  "PGDATA",
									Value: pgDataDir,
								},
								{
									// POSTGRES_PASSWORD from a generated Secret.
									// For simplicity we use a fixed value here;
									// a production operator would generate a Secret.
									Name:  "POSTGRES_PASSWORD",
									Value: "changeme",
								},
							},

							// Ports exposed by the container.
							Ports: []corev1.ContainerPort{
								{
									Name:          "postgresql",
									ContainerPort: 5432,
									Protocol:      corev1.ProtocolTCP,
								},
							},

							// Resources from the PostgresCluster spec.
							Resources: pg.Spec.Resources,

							// Volume mounts for data and configuration.
							VolumeMounts: []corev1.VolumeMount{
								{
									Name:      "data",
									MountPath: "/var/lib/postgresql/data",
								},
								{
									Name:      "config",
									MountPath: "/etc/postgresql",
									ReadOnly:  true,
								},
							},

							// LivenessProbe checks if PostgreSQL is running.
							// If it fails, Kubernetes restarts the container.
							LivenessProbe: &corev1.Probe{
								ProbeHandler: corev1.ProbeHandler{
									Exec: &corev1.ExecAction{
										Command: []string{
											"pg_isready",
											"-U", "postgres",
											"-h", "localhost",
										},
									},
								},
								InitialDelaySeconds: 30,
								PeriodSeconds:       10,
								TimeoutSeconds:      5,
								FailureThreshold:    3,
							},

							// ReadinessProbe checks if PostgreSQL is ready to
							// accept connections. The Pod is not added to the
							// Service endpoints until this probe passes.
							ReadinessProbe: &corev1.Probe{
								ProbeHandler: corev1.ProbeHandler{
									Exec: &corev1.ExecAction{
										Command: []string{
											"pg_isready",
											"-U", "postgres",
											"-h", "localhost",
										},
									},
								},
								InitialDelaySeconds: 5,
								PeriodSeconds:       5,
								TimeoutSeconds:      3,
								FailureThreshold:    3,
							},
						},
					},

					// Config volume from the ConfigMap.
					Volumes: []corev1.Volume{
						{
							Name: "config",
							VolumeSource: corev1.VolumeSource{
								ConfigMap: &corev1.ConfigMapVolumeSource{
									LocalObjectReference: corev1.LocalObjectReference{
										Name: configMapName(pg),
									},
								},
							},
						},
					},
				},
			},

			// VolumeClaimTemplates create a PVC per replica. These PVCs
			// are NOT deleted when the StatefulSet scales down — data is preserved.
			VolumeClaimTemplates: []corev1.PersistentVolumeClaim{
				{
					ObjectMeta: metav1.ObjectMeta{
						Name:   "data",
						Labels: labels,
					},
					Spec: corev1.PersistentVolumeClaimSpec{
						AccessModes: []corev1.PersistentVolumeAccessMode{
							corev1.ReadWriteOnce,
						},
						StorageClassName: pg.Spec.Storage.StorageClassName,
						Resources: corev1.VolumeResourceRequirements{
							Requests: corev1.ResourceList{
								corev1.ResourceStorage: pg.Spec.Storage.Size,
							},
						},
					},
				},
			},
		},
	}

	return sts
}

// int64Ptr is a helper that returns a pointer to an int64 value.
// Go does not allow taking the address of a literal, so helpers like this
// are standard practice in Kubernetes operator code.
func int64Ptr(i int64) *int64 {
	return &i
}
```

### 34.4.4 Step 2 — Create the Headless Service

StatefulSets require a headless Service (one with `clusterIP: None`) to provide DNS entries for
each Pod. Without it, Pods do not get stable network identities.

```go
// reconcileHeadlessService ensures that a headless Service exists for the
// PostgresCluster's StatefulSet. The headless Service provides DNS records
// for each Pod in the StatefulSet, enabling peer discovery.
//
// DNS format: <pod-name>.<headless-svc>.<namespace>.svc.cluster.local
// Example:    my-database-0.my-database-headless.default.svc.cluster.local
//
// This is critical for PostgreSQL streaming replication, where replicas
// need to connect to the primary by a stable hostname.
func (r *PostgresClusterReconciler) reconcileHeadlessService(
	ctx context.Context,
	pg *postgresv1alpha1.PostgresCluster,
) error {
	log := log.FromContext(ctx)
	labels := labelsForCluster(pg)

	desired := &corev1.Service{
		ObjectMeta: metav1.ObjectMeta{
			Name:      headlessServiceName(pg),
			Namespace: pg.Namespace,
			Labels:    labels,
		},
		Spec: corev1.ServiceSpec{
			// ClusterIP: None makes this a headless Service.
			// No virtual IP is allocated; DNS resolves directly to Pod IPs.
			ClusterIP: corev1.ClusterIPNone,

			// Selector matches the StatefulSet Pods.
			Selector: labels,

			// PublishNotReadyAddresses ensures that DNS records exist for Pods
			// even before they pass readiness checks. This is important during
			// initial cluster setup when the primary needs to be reachable
			// by replicas before it's "ready" from Kubernetes' perspective.
			PublishNotReadyAddresses: true,

			Ports: []corev1.ServicePort{
				{
					Name:       "postgresql",
					Port:       5432,
					TargetPort: intstr.FromString("postgresql"),
					Protocol:   corev1.ProtocolTCP,
				},
			},
		},
	}

	// Set owner reference for garbage collection.
	if err := controllerutil.SetControllerReference(pg, desired, r.Scheme); err != nil {
		return fmt.Errorf("setting owner reference on headless Service: %w", err)
	}

	// Create or update the headless Service.
	existing := &corev1.Service{}
	err := r.Get(ctx, types.NamespacedName{
			Name:      desired.Name,
			Namespace: desired.Namespace,
		}, existing)

	if apierrors.IsNotFound(err) {
		log.Info("creating headless Service", "name", desired.Name)
		if err := r.Create(ctx, desired); err != nil {
			return fmt.Errorf("creating headless Service: %w", err)
		}
		r.Recorder.Eventf(pg, corev1.EventTypeNormal, "ServiceCreated",
			"Created headless Service %s", desired.Name)
		return nil
	}
	if err != nil {
		return fmt.Errorf("getting headless Service: %w", err)
	}

	// Headless Services rarely need updates, but we reconcile the ports
	// in case they change.
	if !equality.Semantic.DeepEqual(existing.Spec.Ports, desired.Spec.Ports) {
		existing.Spec.Ports = desired.Spec.Ports
		if err := r.Update(ctx, existing); err != nil {
			return fmt.Errorf("updating headless Service: %w", err)
		}
	}

	return nil
}
```

### 34.4.5 Step 3 — Create the ConfigMap

PostgreSQL configuration is managed through a ConfigMap mounted into the container.

```go
// reconcileConfigMap ensures that a ConfigMap exists with the PostgreSQL
// configuration files. The ConfigMap contains:
//
//   - postgresql.conf: Main PostgreSQL configuration, including parameters
//     from the PostgresCluster spec and operator defaults.
//   - pg_hba.conf: Host-Based Authentication file, controlling which clients
//     can connect and how they authenticate.
//
// When the ConfigMap changes, the StatefulSet Pods detect the volume change
// and PostgreSQL can be signaled to reload its configuration (via SIGHUP
	// or pg_reload_conf()).
func (r *PostgresClusterReconciler) reconcileConfigMap(
	ctx context.Context,
	pg *postgresv1alpha1.PostgresCluster,
) error {
	log := log.FromContext(ctx)
	labels := labelsForCluster(pg)

	// Build postgresql.conf content from operator defaults merged with
	// user-specified parameters. User parameters take precedence.
	postgresConf := buildPostgresConf(pg)
	pgHBAConf := buildPgHBAConf()

	desired := &corev1.ConfigMap{
		ObjectMeta: metav1.ObjectMeta{
			Name:      configMapName(pg),
			Namespace: pg.Namespace,
			Labels:    labels,
		},
		Data: map[string]string{
			"postgresql.conf": postgresConf,
			"pg_hba.conf":     pgHBAConf,
		},
	}

	if err := controllerutil.SetControllerReference(pg, desired, r.Scheme); err != nil {
		return fmt.Errorf("setting owner reference on ConfigMap: %w", err)
	}

	existing := &corev1.ConfigMap{}
	err := r.Get(ctx, types.NamespacedName{
			Name:      desired.Name,
			Namespace: desired.Namespace,
		}, existing)

	if apierrors.IsNotFound(err) {
		log.Info("creating ConfigMap", "name", desired.Name)
		if err := r.Create(ctx, desired); err != nil {
			return fmt.Errorf("creating ConfigMap: %w", err)
		}
		r.Recorder.Eventf(pg, corev1.EventTypeNormal, "ConfigMapCreated",
			"Created ConfigMap %s", desired.Name)
		return nil
	}
	if err != nil {
		return fmt.Errorf("getting ConfigMap: %w", err)
	}

	// Update if the configuration has changed.
	if !equality.Semantic.DeepEqual(existing.Data, desired.Data) {
		log.Info("updating ConfigMap", "name", existing.Name)
		existing.Data = desired.Data
		if err := r.Update(ctx, existing); err != nil {
			return fmt.Errorf("updating ConfigMap: %w", err)
		}
		r.Recorder.Eventf(pg, corev1.EventTypeNormal, "ConfigMapUpdated",
			"Updated ConfigMap %s with new configuration", existing.Name)
	}

	return nil
}

// buildPostgresConf generates the postgresql.conf content by merging operator
// defaults with user-specified parameters from the PostgresCluster spec.
// User parameters always override operator defaults.
//
// The defaults are conservative and suitable for a general-purpose database.
// Production deployments should tune shared_buffers, effective_cache_size,
// and work_mem based on the container's resource limits.
func buildPostgresConf(pg *postgresv1alpha1.PostgresCluster) string {
	// Operator defaults — safe for most workloads.
	defaults := map[string]string{
		"listen_addresses":    "'*'",
		"port":                "5432",
		"max_connections":     "100",
		"shared_buffers":      "128MB",
		"effective_cache_size": "512MB",
		"work_mem":            "4MB",
		"maintenance_work_mem": "64MB",
		"wal_level":           "replica",
		"max_wal_senders":     "10",
		"max_replication_slots": "10",
		"hot_standby":         "on",
		"logging_collector":   "on",
		"log_directory":       "'pg_log'",
		"log_filename":        "'postgresql-%a.log'",
		"log_truncate_on_rotation": "on",
		"log_rotation_age":    "1d",
		"log_rotation_size":   "0",
		"log_min_messages":    "warning",
		"log_min_error_statement": "error",
	}

	// Override defaults with user-specified parameters.
	for k, v := range pg.Spec.Parameters {
		defaults[k] = v
	}

	// Build the config file content.
	conf := "# PostgreSQL configuration managed by postgres-operator\n"
	conf += "# Do not edit manually — changes will be overwritten.\n\n"
	for k, v := range defaults {
		conf += fmt.Sprintf("%s = %s\n", k, v)
	}
	return conf
}

// buildPgHBAConf generates the pg_hba.conf content. This file controls
// host-based authentication — which clients can connect, to which databases,
// as which users, and with what authentication method.
//
// The default configuration allows:
//   - Local connections (Unix socket) without password (trust) for the postgres superuser
//   - All TCP connections with MD5 password authentication
//   - Replication connections from any host with MD5 authentication
func buildPgHBAConf() string {
	return `# PostgreSQL Host-Based Authentication (pg_hba.conf)
	# Managed by postgres-operator — do not edit manually.
	#
	# TYPE  DATABASE        USER            ADDRESS                 METHOD

	# Local connections (Unix domain socket)
	local   all             postgres                                trust
	local   all             all                                     md5

	# IPv4 connections
	host    all             all             0.0.0.0/0               md5

	# IPv6 connections
	host    all             all             ::/0                    md5

	# Replication connections
	host    replication     all             0.0.0.0/0               md5
	host    replication     all             ::/0                    md5
	`
}
```

### 34.4.6 Step 4 — Create the Client-Facing Service

While the headless Service is for internal StatefulSet DNS, we also need a **ClusterIP Service**
for clients to connect to. This Service load-balances across all PostgreSQL instances (or just the
primary, depending on your strategy).

```go
// reconcileService ensures that a ClusterIP Service exists for client
// connections to the PostgreSQL cluster. Unlike the headless Service (used
	// for StatefulSet DNS), this Service provides a single stable endpoint
// that clients use to connect.
//
// For a simple setup, this Service targets all Pods (primary + replicas).
// A production operator might create separate Services for read-write
// (primary only) and read-only (replicas only) traffic.
func (r *PostgresClusterReconciler) reconcileService(
	ctx context.Context,
	pg *postgresv1alpha1.PostgresCluster,
) error {
	log := log.FromContext(ctx)
	labels := labelsForCluster(pg)

	desired := &corev1.Service{
		ObjectMeta: metav1.ObjectMeta{
			Name:      resourceName(pg),
			Namespace: pg.Namespace,
			Labels:    labels,
		},
		Spec: corev1.ServiceSpec{
			// Type ClusterIP is the default — a stable virtual IP accessible
			// only from within the cluster.
			Type:     corev1.ServiceTypeClusterIP,
			Selector: labels,
			Ports: []corev1.ServicePort{
				{
					Name:       "postgresql",
					Port:       5432,
					TargetPort: intstr.FromString("postgresql"),
					Protocol:   corev1.ProtocolTCP,
				},
			},
		},
	}

	if err := controllerutil.SetControllerReference(pg, desired, r.Scheme); err != nil {
		return fmt.Errorf("setting owner reference on Service: %w", err)
	}

	existing := &corev1.Service{}
	err := r.Get(ctx, types.NamespacedName{
			Name:      desired.Name,
			Namespace: desired.Namespace,
		}, existing)

	if apierrors.IsNotFound(err) {
		log.Info("creating Service", "name", desired.Name)
		if err := r.Create(ctx, desired); err != nil {
			return fmt.Errorf("creating Service: %w", err)
		}
		r.Recorder.Eventf(pg, corev1.EventTypeNormal, "ServiceCreated",
			"Created Service %s", desired.Name)
		return nil
	}
	if err != nil {
		return fmt.Errorf("getting Service: %w", err)
	}

	// Update ports if they've changed (leave ClusterIP intact — it's immutable).
	if !equality.Semantic.DeepEqual(existing.Spec.Ports, desired.Spec.Ports) {
		existing.Spec.Ports = desired.Spec.Ports
		if err := r.Update(ctx, existing); err != nil {
			return fmt.Errorf("updating Service: %w", err)
		}
	}

	return nil
}
```

### 34.4.7 Step 5 — Handle Scaling

Scaling is handled implicitly by the StatefulSet — when we update `spec.replicas`, the StatefulSet
controller adds or removes Pods. But our operator needs to be *aware* of scaling so it can update
status and manage replication.

```go
// handleScaling detects when the cluster is in the process of scaling
// (the StatefulSet's current replicas don't match the desired count)
// and updates the PostgresCluster phase accordingly.
//
// During scale-up, new Pods need to be configured as streaming replicas
// of the primary. During scale-down, we should ensure connections are
// drained from the Pods being removed.
//
// In this implementation, the StatefulSet handles the mechanics of scaling.
// Our operator observes the state and reports it. A production operator
// would also configure replication slots and streaming replication.
func (r *PostgresClusterReconciler) handleScaling(
	ctx context.Context,
	pg *postgresv1alpha1.PostgresCluster,
) (isScaling bool, err error) {
	log := log.FromContext(ctx)

	// Get the current StatefulSet.
	sts := &appsv1.StatefulSet{}
	err = r.Get(ctx, types.NamespacedName{
			Name:      resourceName(pg),
			Namespace: pg.Namespace,
		}, sts)
	if err != nil {
		if apierrors.IsNotFound(err) {
			// StatefulSet doesn't exist yet — not scaling, just creating.
			return false, nil
		}
		return false, fmt.Errorf("getting StatefulSet for scaling check: %w", err)
	}

	// Compare desired replicas (from our spec) with actual ready replicas.
	desiredReplicas := pg.Spec.Replicas
	currentReplicas := sts.Status.Replicas
	readyReplicas := sts.Status.ReadyReplicas

	if currentReplicas != desiredReplicas {
		log.Info("cluster is scaling",
			"desired", desiredReplicas,
			"current", currentReplicas,
			"ready", readyReplicas,
		)

		if desiredReplicas > currentReplicas {
			r.Recorder.Eventf(pg, corev1.EventTypeNormal, "ScalingUp",
				"Scaling up from %d to %d replicas", currentReplicas, desiredReplicas)
		} else {
			r.Recorder.Eventf(pg, corev1.EventTypeNormal, "ScalingDown",
				"Scaling down from %d to %d replicas", currentReplicas, desiredReplicas)
		}

		return true, nil
	}

	// Replicas match, but not all may be ready yet.
	if readyReplicas != desiredReplicas {
		return true, nil
	}

	return false, nil
}
```

### 34.4.8 Step 6 — Handle Version Upgrades

PostgreSQL version upgrades are performed via a rolling update of the StatefulSet. The operator
detects when the spec version differs from the running version and triggers the update.

```go
// handleVersionUpgrade detects when the PostgreSQL version in the spec
// differs from the version currently running in the cluster. Version
// changes are applied via the StatefulSet's rolling update strategy,
// which updates Pods one at a time starting from the highest ordinal.
//
// This handles minor version upgrades (e.g., 16.1 → 16.2) which are
// generally safe to do via rolling restart. Major version upgrades
// (e.g., 15.x → 16.x) require pg_upgrade and are NOT handled here —
// a production operator would implement a separate upgrade workflow.
//
// Returns true if an upgrade is in progress.
func (r *PostgresClusterReconciler) handleVersionUpgrade(
	ctx context.Context,
	pg *postgresv1alpha1.PostgresCluster,
) (isUpgrading bool, err error) {
	log := log.FromContext(ctx)

	// If this is the first reconcile and there's no current version, no upgrade.
	if pg.Status.CurrentVersion == "" {
		return false, nil
	}

	// Compare the desired version with the current version.
	if pg.Spec.Version != pg.Status.CurrentVersion {
		log.Info("version upgrade detected",
			"currentVersion", pg.Status.CurrentVersion,
			"desiredVersion", pg.Spec.Version,
		)

		r.Recorder.Eventf(pg, corev1.EventTypeNormal, "VersionUpgrade",
			"Upgrading PostgreSQL from %s to %s",
			pg.Status.CurrentVersion, pg.Spec.Version)

		// The actual update is handled by reconcileStatefulSet, which will
		// detect the image change and update the StatefulSet. The StatefulSet
		// controller then performs a rolling update.
		return true, nil
	}

	// Check if the rolling update is still in progress.
	sts := &appsv1.StatefulSet{}
	err = r.Get(ctx, types.NamespacedName{
			Name:      resourceName(pg),
			Namespace: pg.Namespace,
		}, sts)
	if err != nil {
		if apierrors.IsNotFound(err) {
			return false, nil
		}
		return false, fmt.Errorf("getting StatefulSet for upgrade check: %w", err)
	}

	// If UpdatedReplicas < Replicas, the rolling update is still in progress.
	if sts.Status.UpdatedReplicas < sts.Status.Replicas {
		return true, nil
	}

	return false, nil
}
```

### 34.4.9 Step 7 — Update Status

The status subresource communicates the cluster's observed state back to users and other systems.
Updating status correctly is critical — it's how `kubectl get pg` shows useful information.

```go
// updateStatus sets the PostgresCluster's status fields based on the
// current state of the cluster. It reads the StatefulSet's status to
// determine replica counts and sets conditions based on cluster health.
//
// Status updates use the status subresource, which has separate RBAC
// from the main resource. This means a user can be granted read/write
// access to the spec without being able to modify the status, and
// vice versa.
func (r *PostgresClusterReconciler) updateStatus(
	ctx context.Context,
	pg *postgresv1alpha1.PostgresCluster,
	phase postgresv1alpha1.PostgresClusterPhase,
) error {
	// Get the current StatefulSet to read replica counts.
	sts := &appsv1.StatefulSet{}
	err := r.Get(ctx, types.NamespacedName{
			Name:      resourceName(pg),
			Namespace: pg.Namespace,
		}, sts)

	if err == nil {
		// StatefulSet exists — update replica counts from its status.
		pg.Status.CurrentReplicas = sts.Status.Replicas
		pg.Status.ReadyReplicas = sts.Status.ReadyReplicas
	} else if !apierrors.IsNotFound(err) {
		return fmt.Errorf("getting StatefulSet for status: %w", err)
	}

	// Set the phase.
	pg.Status.Phase = phase

	// Set the current version. During upgrades, this is updated only after
	// all Pods are running the new version.
	if phase != postgresv1alpha1.PhaseUpgrading {
		pg.Status.CurrentVersion = pg.Spec.Version
	}

	// Set the observed generation. This tells consumers whether the status
	// is up-to-date with the latest spec change.
	pg.Status.ObservedGeneration = pg.Generation

	// --- Conditions ---

	now := metav1.Now()

	// Initialized condition: True once the StatefulSet, Services, and
	// ConfigMap have been created.
	meta.SetStatusCondition(&pg.Status.Conditions, metav1.Condition{
			Type:               postgresv1alpha1.ConditionTypeInitialized,
			Status:             metav1.ConditionTrue,
			Reason:             "ResourcesCreated",
			Message:            "All managed resources have been created",
			ObservedGeneration: pg.Generation,
			LastTransitionTime: now,
		})

	// Available condition: True when at least one replica is ready.
	if pg.Status.ReadyReplicas > 0 {
		meta.SetStatusCondition(&pg.Status.Conditions, metav1.Condition{
				Type:               postgresv1alpha1.ConditionTypeAvailable,
				Status:             metav1.ConditionTrue,
				Reason:             "ReplicasAvailable",
				Message:            fmt.Sprintf("%d replica(s) available", pg.Status.ReadyReplicas),
				ObservedGeneration: pg.Generation,
				LastTransitionTime: now,
			})
	} else {
		meta.SetStatusCondition(&pg.Status.Conditions, metav1.Condition{
				Type:               postgresv1alpha1.ConditionTypeAvailable,
				Status:             metav1.ConditionFalse,
				Reason:             "NoReplicasAvailable",
				Message:            "No replicas are ready",
				ObservedGeneration: pg.Generation,
				LastTransitionTime: now,
			})
	}

	// Ready condition: True when ALL desired replicas are ready.
	if pg.Status.ReadyReplicas == pg.Spec.Replicas {
		meta.SetStatusCondition(&pg.Status.Conditions, metav1.Condition{
				Type:               postgresv1alpha1.ConditionTypeReady,
				Status:             metav1.ConditionTrue,
				Reason:             "AllReplicasReady",
				Message:            fmt.Sprintf("All %d replicas are ready", pg.Spec.Replicas),
				ObservedGeneration: pg.Generation,
				LastTransitionTime: now,
			})
	} else {
		meta.SetStatusCondition(&pg.Status.Conditions, metav1.Condition{
				Type:               postgresv1alpha1.ConditionTypeReady,
				Status:             metav1.ConditionFalse,
				Reason:             "ReplicasNotReady",
				Message: fmt.Sprintf("%d of %d replicas are ready",
					pg.Status.ReadyReplicas, pg.Spec.Replicas),
				ObservedGeneration: pg.Generation,
				LastTransitionTime: now,
			})
	}

	// Progressing condition: True when the operator is actively making changes.
	isProgressing := phase == postgresv1alpha1.PhaseCreating ||
	phase == postgresv1alpha1.PhaseScaling ||
	phase == postgresv1alpha1.PhaseUpgrading
	if isProgressing {
		meta.SetStatusCondition(&pg.Status.Conditions, metav1.Condition{
				Type:               postgresv1alpha1.ConditionTypeProgressing,
				Status:             metav1.ConditionTrue,
				Reason:             string(phase),
				Message:            fmt.Sprintf("Cluster is %s", phase),
				ObservedGeneration: pg.Generation,
				LastTransitionTime: now,
			})
	} else {
		meta.SetStatusCondition(&pg.Status.Conditions, metav1.Condition{
				Type:               postgresv1alpha1.ConditionTypeProgressing,
				Status:             metav1.ConditionFalse,
				Reason:             "Stable",
				Message:            "Cluster is stable",
				ObservedGeneration: pg.Generation,
				LastTransitionTime: now,
			})
	}

	// Write the status subresource.
	if err := r.Status().Update(ctx, pg); err != nil {
		return fmt.Errorf("updating PostgresCluster status: %w", err)
	}

	return nil
}
```

### 34.4.10 Step 8 — The Complete Reconcile Function

Now we bring all the steps together into the main `Reconcile` method. This is the function that
controller-runtime calls each time a reconciliation is triggered.

```go
// Reconcile is the main entry point for the PostgresCluster controller.
// It is called by controller-runtime whenever a PostgresCluster resource
// is created, updated, or deleted, or when any owned resource changes.
//
// The reconcile function is designed to be idempotent: calling it multiple
// times with the same input produces the same result. It follows the
// "level-triggered" pattern — it always reconciles based on the current
// state, not based on what event triggered the reconciliation.
//
// The reconciliation flow:
//  1. Fetch the PostgresCluster resource.
//  2. Handle deletion (if the resource is being deleted, run finalizer logic).
//  3. Ensure the finalizer is present.
//  4. Reconcile child resources (ConfigMap, Services, StatefulSet).
//  5. Detect scaling and version upgrade operations.
//  6. Update the status subresource.
//
// If any step fails, the function returns an error, which causes
// controller-runtime to requeue the reconciliation with exponential backoff.
//
// +kubebuilder:rbac:groups=postgres.example.com,resources=postgresclusters,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=postgres.example.com,resources=postgresclusters/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=postgres.example.com,resources=postgresclusters/finalizers,verbs=update
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=configmaps,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=events,verbs=create;patch
// +kubebuilder:rbac:groups=core,resources=pods,verbs=get;list;watch
func (r *PostgresClusterReconciler) Reconcile(
	ctx context.Context,
	req ctrl.Request,
) (ctrl.Result, error) {
	log := log.FromContext(ctx)

	// ---------------------------------------------------------------
	// Step 1: Fetch the PostgresCluster resource.
	// ---------------------------------------------------------------
	pg := &postgresv1alpha1.PostgresCluster{}
	if err := r.Get(ctx, req.NamespacedName, pg); err != nil {
		if apierrors.IsNotFound(err) {
			// The resource was deleted before we could reconcile.
			// Nothing to do — owned resources will be garbage collected
			// via owner references.
			log.Info("PostgresCluster not found — probably deleted")
			return ctrl.Result{}, nil
		}
		return ctrl.Result{}, fmt.Errorf("fetching PostgresCluster: %w", err)
	}

	log.Info("reconciling PostgresCluster",
		"name", pg.Name,
		"generation", pg.Generation,
		"version", pg.Spec.Version,
		"replicas", pg.Spec.Replicas,
	)

	// ---------------------------------------------------------------
	// Step 2: Handle deletion with finalizer.
	// ---------------------------------------------------------------
	if !pg.DeletionTimestamp.IsZero() {
		// Resource is being deleted. Run cleanup logic.
		if controllerutil.ContainsFinalizer(pg, finalizerName) {
			log.Info("running finalizer cleanup")

			// Perform any external cleanup here (e.g., delete DNS records,
				// revoke cloud IAM bindings, remove monitoring dashboards).
			// For this operator, we just log it.
			r.Recorder.Event(pg, corev1.EventTypeNormal, "Cleanup",
				"Running pre-deletion cleanup")

			// Remove the finalizer so Kubernetes can proceed with deletion.
			controllerutil.RemoveFinalizer(pg, finalizerName)
			if err := r.Update(ctx, pg); err != nil {
				return ctrl.Result{}, fmt.Errorf("removing finalizer: %w", err)
			}
		}
		return ctrl.Result{}, nil
	}

	// ---------------------------------------------------------------
	// Step 3: Ensure the finalizer is present.
	// ---------------------------------------------------------------
	if !controllerutil.ContainsFinalizer(pg, finalizerName) {
		controllerutil.AddFinalizer(pg, finalizerName)
		if err := r.Update(ctx, pg); err != nil {
			return ctrl.Result{}, fmt.Errorf("adding finalizer: %w", err)
		}
		// Requeue after adding finalizer — the Update triggers a new
		// reconciliation anyway, but being explicit is clearer.
		return ctrl.Result{Requeue: true}, nil
	}

	// ---------------------------------------------------------------
	// Step 4: Reconcile child resources.
	// ---------------------------------------------------------------

	// 4a: ConfigMap (must exist before StatefulSet so volumes can mount it).
	if err := r.reconcileConfigMap(ctx, pg); err != nil {
		r.Recorder.Eventf(pg, corev1.EventTypeWarning, "ReconcileError",
			"Failed to reconcile ConfigMap: %v", err)
		return ctrl.Result{}, err
	}

	// 4b: Headless Service (must exist before StatefulSet for DNS).
	if err := r.reconcileHeadlessService(ctx, pg); err != nil {
		r.Recorder.Eventf(pg, corev1.EventTypeWarning, "ReconcileError",
			"Failed to reconcile headless Service: %v", err)
		return ctrl.Result{}, err
	}

	// 4c: Client-facing Service.
	if err := r.reconcileService(ctx, pg); err != nil {
		r.Recorder.Eventf(pg, corev1.EventTypeWarning, "ReconcileError",
			"Failed to reconcile Service: %v", err)
		return ctrl.Result{}, err
	}

	// 4d: StatefulSet (depends on ConfigMap and headless Service).
	if err := r.reconcileStatefulSet(ctx, pg); err != nil {
		r.Recorder.Eventf(pg, corev1.EventTypeWarning, "ReconcileError",
			"Failed to reconcile StatefulSet: %v", err)
		return ctrl.Result{}, err
	}

	// ---------------------------------------------------------------
	// Step 5: Detect operational state (scaling, upgrading).
	// ---------------------------------------------------------------

	// Determine the cluster phase for status reporting.
	phase := postgresv1alpha1.PhaseRunning

	isScaling, err := r.handleScaling(ctx, pg)
	if err != nil {
		return ctrl.Result{}, err
	}
	if isScaling {
		phase = postgresv1alpha1.PhaseScaling
	}

	isUpgrading, err := r.handleVersionUpgrade(ctx, pg)
	if err != nil {
		return ctrl.Result{}, err
	}
	if isUpgrading {
		phase = postgresv1alpha1.PhaseUpgrading
	}

	// If the cluster is not yet running (no ready replicas), it's still creating.
	if pg.Status.ReadyReplicas == 0 && !isScaling && !isUpgrading {
		phase = postgresv1alpha1.PhaseCreating
	}

	// ---------------------------------------------------------------
	// Step 6: Update status.
	// ---------------------------------------------------------------
	if err := r.updateStatus(ctx, pg, phase); err != nil {
		return ctrl.Result{}, err
	}

	// ---------------------------------------------------------------
	// Step 7: Requeue if not stable.
	// ---------------------------------------------------------------
	// If the cluster is in a transitional state, requeue after a short
	// delay to continue monitoring progress.
	if phase != postgresv1alpha1.PhaseRunning {
		log.Info("cluster not yet stable, requeueing",
			"phase", phase,
			"readyReplicas", pg.Status.ReadyReplicas,
			"desiredReplicas", pg.Spec.Replicas,
		)
		return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
	}

	log.Info("reconciliation complete — cluster is stable",
		"readyReplicas", pg.Status.ReadyReplicas,
	)
	return ctrl.Result{}, nil
}

// SetupWithManager registers the PostgresClusterReconciler with the
// controller-runtime Manager. It configures which resources the controller
// watches and how changes to those resources trigger reconciliation.
//
// The controller watches:
//   - PostgresCluster resources (primary watch)
//   - StatefulSets owned by a PostgresCluster (secondary watch)
//   - Services owned by a PostgresCluster (secondary watch)
//   - ConfigMaps owned by a PostgresCluster (secondary watch)
//
// When an owned resource changes, the controller looks up the owning
// PostgresCluster via the owner reference and enqueues a reconciliation
// for that PostgresCluster (not for the child resource itself).
func (r *PostgresClusterReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
	// Primary watch: PostgresCluster custom resources.
	For(&postgresv1alpha1.PostgresCluster{}).
	// Secondary watches: owned resources.
	// When a StatefulSet/Service/ConfigMap changes, reconcile the
	// owning PostgresCluster.
	Owns(&appsv1.StatefulSet{}).
	Owns(&corev1.Service{}).
	Owns(&corev1.ConfigMap{}).
	Complete(r)
}
```

---

## 34.5 Owner References and Garbage Collection

Every resource the operator creates (StatefulSet, Services, ConfigMap) has an **owner reference**
pointing back to the PostgresCluster resource. This has two critical effects:

1. **Automatic garbage collection** — when the PostgresCluster is deleted, Kubernetes automatically
   deletes all owned resources.
2. **Watch propagation** — when an owned resource changes (e.g., a Pod in the StatefulSet becomes
   ready), controller-runtime maps the event back to the owning PostgresCluster and triggers a
   reconciliation.

### 34.5.1 How Owner References Work

```go
// controllerutil.SetControllerReference sets the owner reference on the
// child resource. The parameters are:
//   owner: the PostgresCluster resource
//   controlled: the child resource (StatefulSet, Service, etc.)
//   scheme: the runtime scheme (needed to look up the owner's GVK)
//
// This sets:
//   ownerReferences:
//   - apiVersion: postgres.example.com/v1alpha1
//     kind: PostgresCluster
//     name: my-database
//     uid: <uid>
//     controller: true       # <-- marks this as THE controller
//     blockOwnerDeletion: true
```

### 34.5.2 The Controller Field

The `controller: true` field is special. Each resource can have at most **one** controller owner.
The controller owner is the one that "manages" the resource — when the controller owner is deleted,
the owned resource is deleted. Non-controller owners are informational only.

### 34.5.3 Garbage Collection Policies

When the owner is deleted, Kubernetes uses one of three policies:

| Policy | Behavior |
|--------|----------|
| **Background** (default) | Owner is deleted immediately; GC deletes children asynchronously |
| **Foreground** | Children are deleted first; owner is deleted only after all children are gone |
| **Orphan** | Owner is deleted; children are NOT deleted (they become orphans) |

You can specify the policy when deleting:

```bash
# Background (default)
kubectl delete postgrescluster my-database

# Foreground — wait for children to be deleted
kubectl delete postgrescluster my-database --cascade=foreground

# Orphan — leave children running
kubectl delete postgrescluster my-database --cascade=orphan
```

### 34.5.4 Verifying Garbage Collection

```bash
# Create the cluster
kubectl apply -f config/samples/postgres_v1alpha1_postgrescluster.yaml

# Verify owned resources exist
kubectl get statefulset,svc,configmap -l app.kubernetes.io/instance=my-database

# Delete the cluster
kubectl delete postgrescluster my-database

# Verify owned resources are deleted
kubectl get statefulset,svc,configmap -l app.kubernetes.io/instance=my-database
# No resources found
```

---

## 34.6 RBAC Configuration

The operator needs permissions to read and write the Kubernetes resources it manages. These
permissions are declared via **RBAC** (Role-Based Access Control) manifests.

### 34.6.1 Kubebuilder RBAC Markers

The `+kubebuilder:rbac` markers on the `Reconcile` method are processed by `controller-gen` to
produce a ClusterRole. The markers we defined:

```go
// +kubebuilder:rbac:groups=postgres.example.com,resources=postgresclusters,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=postgres.example.com,resources=postgresclusters/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=postgres.example.com,resources=postgresclusters/finalizers,verbs=update
// +kubebuilder:rbac:groups=apps,resources=statefulsets,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=services,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=configmaps,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=core,resources=events,verbs=create;patch
// +kubebuilder:rbac:groups=core,resources=pods,verbs=get;list;watch
```

### 34.6.2 Generated ClusterRole

Running `make manifests` produces `config/rbac/role.yaml`:

```yaml
# This ClusterRole is auto-generated from +kubebuilder:rbac markers.
# Do not edit manually — run 'make manifests' to regenerate.
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: manager-role
rules:
- apiGroups:
  - postgres.example.com
  resources:
  - postgresclusters
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch
- apiGroups:
  - postgres.example.com
  resources:
  - postgresclusters/status
  verbs:
  - get
  - patch
  - update
- apiGroups:
  - postgres.example.com
  resources:
  - postgresclusters/finalizers
  verbs:
  - update
- apiGroups:
  - apps
  resources:
  - statefulsets
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch
- apiGroups:
  - ""
  resources:
  - services
  - configmaps
  verbs:
  - create
  - delete
  - get
  - list
  - patch
  - update
  - watch
- apiGroups:
  - ""
  resources:
  - events
  verbs:
  - create
  - patch
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
  - list
  - watch
```

### 34.6.3 Principle of Least Privilege

Always follow the principle of least privilege:

- Only request the verbs you actually use (e.g., don't add `delete` for Pods if you never delete
  Pods).
- Scope to specific resource types, not `*`.
- Use namespaced Roles instead of ClusterRoles when the operator operates in a single namespace.

---

## 34.7 Testing the Operator

Testing is critical for operators because they manage stateful infrastructure. A bug in the
reconciler can take down databases. We use three levels of testing:

### 34.7.1 Unit Tests with Fake Client

The controller-runtime fake client lets you test reconcile logic without a real API server.

```go
// Package controller contains unit and integration tests for the
// PostgresCluster reconciler. Unit tests use a fake client to verify
// reconcile logic without a real API server. Integration tests use
// envtest to run against a real API server (without nodes).
package controller

import (
	"context"
	"testing"

	"github.com/stretchr/testify/assert"
	"github.com/stretchr/testify/require"
	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/resource"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/tools/record"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client/fake"

	postgresv1alpha1 "github.com/example/postgres-operator/api/v1alpha1"
)

// newTestReconciler creates a PostgresClusterReconciler backed by a fake
// Kubernetes client. The fake client is pre-populated with the given objects.
// This is used in unit tests to verify reconcile logic without a real cluster.
func newTestReconciler(objs ...runtime.Object) *PostgresClusterReconciler {
	scheme := runtime.NewScheme()
	_ = postgresv1alpha1.AddToScheme(scheme)
	_ = appsv1.AddToScheme(scheme)
	_ = corev1.AddToScheme(scheme)

	client := fake.NewClientBuilder().
	WithScheme(scheme).
	WithRuntimeObjects(objs...).
	WithStatusSubresource(&postgresv1alpha1.PostgresCluster{}).
	Build()

	return &PostgresClusterReconciler{
		Client:   client,
		Scheme:   scheme,
		Recorder: record.NewFakeRecorder(100),
	}
}

// newTestPostgresCluster creates a PostgresCluster resource with sensible
// defaults for testing.
func newTestPostgresCluster() *postgresv1alpha1.PostgresCluster {
	return &postgresv1alpha1.PostgresCluster{
		ObjectMeta: metav1.ObjectMeta{
			Name:       "test-pg",
			Namespace:  "default",
			Generation: 1,
		},
		Spec: postgresv1alpha1.PostgresClusterSpec{
			Version:  "16.2",
			Replicas: 3,
			Storage: postgresv1alpha1.StorageSpec{
				Size: resource.MustParse("10Gi"),
			},
		},
	}
}

// TestReconcile_CreatesResources verifies that the first reconciliation
// creates all expected child resources: StatefulSet, headless Service,
// client Service, and ConfigMap.
func TestReconcile_CreatesResources(t *testing.T) {
	pg := newTestPostgresCluster()
	r := newTestReconciler(pg)
	ctx := context.Background()

	// First reconcile: adds finalizer and requeues.
	result, err := r.Reconcile(ctx, ctrl.Request{
			NamespacedName: types.NamespacedName{
				Name:      pg.Name,
				Namespace: pg.Namespace,
			},
		})
	require.NoError(t, err)
	assert.True(t, result.Requeue, "expected requeue after adding finalizer")

	// Second reconcile: creates resources.
	result, err = r.Reconcile(ctx, ctrl.Request{
			NamespacedName: types.NamespacedName{
				Name:      pg.Name,
				Namespace: pg.Namespace,
			},
		})
	require.NoError(t, err)

	// Verify StatefulSet was created.
	sts := &appsv1.StatefulSet{}
	err = r.Get(ctx, types.NamespacedName{
			Name:      "test-pg",
			Namespace: "default",
		}, sts)
	require.NoError(t, err)
	assert.Equal(t, int32(3), *sts.Spec.Replicas)
	assert.Equal(t, "postgres:16.2", sts.Spec.Template.Spec.Containers[0].Image)

	// Verify headless Service was created.
	headlessSvc := &corev1.Service{}
	err = r.Get(ctx, types.NamespacedName{
			Name:      "test-pg-headless",
			Namespace: "default",
		}, headlessSvc)
	require.NoError(t, err)
	assert.Equal(t, corev1.ClusterIPNone, headlessSvc.Spec.ClusterIP)

	// Verify client Service was created.
	clientSvc := &corev1.Service{}
	err = r.Get(ctx, types.NamespacedName{
			Name:      "test-pg",
			Namespace: "default",
		}, clientSvc)
	require.NoError(t, err)

	// Verify ConfigMap was created.
	cm := &corev1.ConfigMap{}
	err = r.Get(ctx, types.NamespacedName{
			Name:      "test-pg-config",
			Namespace: "default",
		}, cm)
	require.NoError(t, err)
	assert.Contains(t, cm.Data, "postgresql.conf")
	assert.Contains(t, cm.Data, "pg_hba.conf")
}

// TestReconcile_HandlesNotFound verifies that reconciliation succeeds
// gracefully when the PostgresCluster has been deleted.
func TestReconcile_HandlesNotFound(t *testing.T) {
	// No PostgresCluster object exists in the fake client.
	r := newTestReconciler()
	ctx := context.Background()

	result, err := r.Reconcile(ctx, ctrl.Request{
			NamespacedName: types.NamespacedName{
				Name:      "does-not-exist",
				Namespace: "default",
			},
		})
	require.NoError(t, err)
	assert.Equal(t, ctrl.Result{}, result)
}

// TestReconcile_UpdatesStatefulSetOnScaleChange verifies that changing
// the replica count in the PostgresCluster spec triggers an update
// to the StatefulSet.
func TestReconcile_UpdatesStatefulSetOnScaleChange(t *testing.T) {
	pg := newTestPostgresCluster()
	r := newTestReconciler(pg)
	ctx := context.Background()

	// Initial reconcile (adds finalizer).
	_, _ = r.Reconcile(ctx, ctrl.Request{
			NamespacedName: types.NamespacedName{
				Name:      pg.Name,
				Namespace: pg.Namespace,
			},
		})

	// Second reconcile (creates resources).
	_, _ = r.Reconcile(ctx, ctrl.Request{
			NamespacedName: types.NamespacedName{
				Name:      pg.Name,
				Namespace: pg.Namespace,
			},
		})

	// Update the replica count.
	updatedPG := &postgresv1alpha1.PostgresCluster{}
	err := r.Get(ctx, types.NamespacedName{
			Name:      pg.Name,
			Namespace: pg.Namespace,
		}, updatedPG)
	require.NoError(t, err)

	updatedPG.Spec.Replicas = 5
	err = r.Update(ctx, updatedPG)
	require.NoError(t, err)

	// Reconcile again.
	_, err = r.Reconcile(ctx, ctrl.Request{
			NamespacedName: types.NamespacedName{
				Name:      pg.Name,
				Namespace: pg.Namespace,
			},
		})
	require.NoError(t, err)

	// Verify the StatefulSet was updated.
	sts := &appsv1.StatefulSet{}
	err = r.Get(ctx, types.NamespacedName{
			Name:      "test-pg",
			Namespace: "default",
		}, sts)
	require.NoError(t, err)
	assert.Equal(t, int32(5), *sts.Spec.Replicas)
}
```

### 34.7.2 Integration Tests with envtest

`envtest` starts a real API server and etcd (but no kubelet, scheduler, or controller manager). This
tests the full reconcile loop including actual API calls, validation, and RBAC.

```go
// Package controller_test contains integration tests using envtest.
// envtest starts a real API server and etcd, allowing us to test the
// full reconcile loop with real API calls, validation, and RBAC.
package controller_test

import (
	"context"
	"path/filepath"
	"testing"
	"time"

	. "github.com/onsi/ginkgo/v2"
	. "github.com/onsi/gomega"
	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/resource"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
	"k8s.io/client-go/kubernetes/scheme"
	"k8s.io/client-go/rest"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/envtest"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/log/zap"

	postgresv1alpha1 "github.com/example/postgres-operator/api/v1alpha1"
	"github.com/example/postgres-operator/internal/controller"
)

// testEnv holds the envtest environment — a real API server and etcd
// running as test fixtures. Shared across all integration tests.
var (
	cfg       *rest.Config
	k8sClient client.Client
	testEnv   *envtest.Environment
	ctx       context.Context
	cancel    context.CancelFunc
)

// TestAPIs runs the Ginkgo test suite.
func TestAPIs(t *testing.T) {
	RegisterFailHandler(Fail)
	RunSpecs(t, "Controller Suite")
}

// BeforeSuite starts the envtest environment and the controller manager.
var _ = BeforeSuite(func() {
		logf.SetLogger(zap.New(zap.WriteTo(GinkgoWriter), zap.UseDevMode(true)))

		ctx, cancel = context.WithCancel(context.TODO())

		// Start envtest with CRD paths.
		testEnv = &envtest.Environment{
			CRDDirectoryPaths: []string{
				filepath.Join("..", "..", "config", "crd", "bases"),
			},
			ErrorIfCRDPathMissing: true,
		}

		var err error
		cfg, err = testEnv.Start()
		Expect(err).NotTo(HaveOccurred())
		Expect(cfg).NotTo(BeNil())

		// Register schemes.
		err = postgresv1alpha1.AddToScheme(scheme.Scheme)
		Expect(err).NotTo(HaveOccurred())

		// Create a client.
		k8sClient, err = client.New(cfg, client.Options{Scheme: scheme.Scheme})
		Expect(err).NotTo(HaveOccurred())

		// Start the controller manager in a goroutine.
		mgr, err := ctrl.NewManager(cfg, ctrl.Options{Scheme: scheme.Scheme})
		Expect(err).NotTo(HaveOccurred())

		err = (&controller.PostgresClusterReconciler{
				Client:   mgr.GetClient(),
				Scheme:   mgr.GetScheme(),
				Recorder: mgr.GetEventRecorderFor("test"),
			}).SetupWithManager(mgr)
		Expect(err).NotTo(HaveOccurred())

		go func() {
			defer GinkgoRecover()
			err = mgr.Start(ctx)
			Expect(err).NotTo(HaveOccurred())
		}()
	})

// AfterSuite tears down the envtest environment.
var _ = AfterSuite(func() {
		cancel()
		err := testEnv.Stop()
		Expect(err).NotTo(HaveOccurred())
	})

// Integration test: verify that creating a PostgresCluster resource
// results in the expected child resources being created.
var _ = Describe("PostgresCluster Controller", func() {
		Context("When creating a PostgresCluster", func() {
				It("Should create the expected child resources", func() {
						pg := &postgresv1alpha1.PostgresCluster{
							ObjectMeta: metav1.ObjectMeta{
								Name:      "integration-test-pg",
								Namespace: "default",
							},
							Spec: postgresv1alpha1.PostgresClusterSpec{
								Version:  "16.2",
								Replicas: 1,
								Storage: postgresv1alpha1.StorageSpec{
									Size: resource.MustParse("1Gi"),
								},
							},
						}

						Expect(k8sClient.Create(ctx, pg)).Should(Succeed())

						// Wait for the StatefulSet to be created.
						stsKey := types.NamespacedName{
							Name:      "integration-test-pg",
							Namespace: "default",
						}
						Eventually(func() error {
								return k8sClient.Get(ctx, stsKey, &appsv1.StatefulSet{})
							}, 30*time.Second, time.Second).Should(Succeed())

						// Wait for the headless Service.
						headlessKey := types.NamespacedName{
							Name:      "integration-test-pg-headless",
							Namespace: "default",
						}
						Eventually(func() error {
								return k8sClient.Get(ctx, headlessKey, &corev1.Service{})
							}, 30*time.Second, time.Second).Should(Succeed())

						// Cleanup.
						Expect(k8sClient.Delete(ctx, pg)).Should(Succeed())
					})
			})
	})
```

### 34.7.3 E2E Tests with a Real Cluster

For end-to-end tests, use [kind](https://kind.sigs.k8s.io/) (Kubernetes in Docker):

```bash
#!/bin/bash
# e2e-test.sh — End-to-end test for the postgres-operator.
# Requires: kind, kubectl, docker

set -euo pipefail

CLUSTER_NAME="postgres-operator-e2e"
IMG="postgres-operator:e2e"

echo "=== Creating kind cluster ==="
kind create cluster --name "$CLUSTER_NAME"

echo "=== Building operator image ==="
make docker-build IMG="$IMG"

echo "=== Loading image into kind ==="
kind load docker-image "$IMG" --name "$CLUSTER_NAME"

echo "=== Installing CRDs ==="
make install

echo "=== Deploying operator ==="
make deploy IMG="$IMG"

echo "=== Waiting for operator to be ready ==="
kubectl -n postgres-operator-system wait --for=condition=Available \
  deployment/postgres-operator-controller-manager --timeout=120s

echo "=== Creating PostgresCluster ==="
kubectl apply -f config/samples/postgres_v1alpha1_postgrescluster.yaml

echo "=== Waiting for PostgresCluster to be ready ==="
kubectl wait --for=jsonpath='{.status.phase}'=Running \
  postgrescluster/my-database --timeout=180s

echo "=== Verifying resources ==="
kubectl get statefulset my-database
kubectl get svc my-database my-database-headless
kubectl get configmap my-database-config

echo "=== Scaling up ==="
kubectl patch postgrescluster my-database -p '{"spec":{"replicas":3}}' --type=merge

echo "=== Waiting for scale-up ==="
sleep 10
kubectl get statefulset my-database

echo "=== Cleaning up ==="
kubectl delete postgrescluster my-database
kind delete cluster --name "$CLUSTER_NAME"

echo "=== E2E tests passed ==="
```

---

## 34.8 Deploying the Operator

### 34.8.1 Building the Container Image

The Dockerfile generated by kubebuilder uses a multi-stage build:

```dockerfile
# Build stage: compile the operator binary
FROM golang:1.22 AS builder
ARG TARGETOS
ARG TARGETARCH

WORKDIR /workspace
# Cache dependencies
COPY go.mod go.sum ./
RUN go mod download
# Copy source
COPY cmd/ cmd/
COPY api/ api/
COPY internal/ internal/
# Build
RUN CGO_ENABLED=0 GOOS=${TARGETOS:-linux} GOARCH=${TARGETARCH} \
    go build -a -o manager cmd/main.go

# Runtime stage: minimal image
FROM gcr.io/distroless/static:nonroot
WORKDIR /
COPY --from=builder /workspace/manager .
USER 65532:65532
ENTRYPOINT ["/manager"]
```

Build and push:

```bash
# Build for your architecture
make docker-build IMG=myregistry.com/postgres-operator:v0.1.0

# Push to registry
make docker-push IMG=myregistry.com/postgres-operator:v0.1.0
```

### 34.8.2 Deploying with Kustomize

Kubebuilder projects use kustomize for deployment manifests:

```bash
# Install CRDs into the cluster
make install

# Deploy the operator
make deploy IMG=myregistry.com/postgres-operator:v0.1.0

# This creates:
# - Namespace: postgres-operator-system
# - Deployment: postgres-operator-controller-manager
# - ServiceAccount, ClusterRole, ClusterRoleBinding
# - CRD: postgresclusters.postgres.example.com
```

### 34.8.3 Operator Lifecycle Manager (OLM) Basics

For production distribution, package your operator for OLM:

1. **ClusterServiceVersion (CSV)** — describes the operator, its requirements, and RBAC.
2. **Bundle** — a directory containing the CSV, CRDs, and metadata.
3. **Catalog** — an index of bundles (like an app store for operators).

```bash
# Generate the bundle
make bundle IMG=myregistry.com/postgres-operator:v0.1.0

# Build the bundle image
make bundle-build BUNDLE_IMG=myregistry.com/postgres-operator-bundle:v0.1.0

# Push the bundle
make bundle-push BUNDLE_IMG=myregistry.com/postgres-operator-bundle:v0.1.0
```

We cover OLM in much more detail in Chapter 35.

---

## 34.9 Using the Operator

### 34.9.1 Creating a PostgresCluster

```yaml
# my-database.yaml
apiVersion: postgres.example.com/v1alpha1
kind: PostgresCluster
metadata:
  name: my-database
  namespace: default
spec:
  version: "16.2"
  replicas: 3
  storage:
    size: 50Gi
    storageClassName: fast-ssd
  resources:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2"
      memory: "4Gi"
  parameters:
    max_connections: "200"
    shared_buffers: "1GB"
```

```bash
kubectl apply -f my-database.yaml

# Watch the cluster come up
kubectl get pg my-database -w
# NAME          VERSION   REPLICAS   READY   PHASE      AGE
# my-database   16.2      3          0       Creating   5s
# my-database   16.2      3          1       Creating   25s
# my-database   16.2      3          2       Creating   35s
# my-database   16.2      3          3       Running    45s
```

### 34.9.2 Inspecting the Cluster

```bash
# Detailed status
kubectl describe pg my-database

# Events show the operator's actions
# Events:
#   Type    Reason             Age   From                        Message
#   ----    ------             ----  ----                        -------
#   Normal  ConfigMapCreated   45s   postgrescluster-controller  Created ConfigMap my-database-config
#   Normal  ServiceCreated     45s   postgrescluster-controller  Created headless Service my-database-headless
#   Normal  ServiceCreated     45s   postgrescluster-controller  Created Service my-database
#   Normal  StatefulSetCreated 45s   postgrescluster-controller  Created StatefulSet my-database with 3 replicas

# Check status conditions
kubectl get pg my-database -o jsonpath='{.status.conditions}' | jq .
```

### 34.9.3 Scaling

```bash
# Scale up to 5 replicas
kubectl patch pg my-database -p '{"spec":{"replicas":5}}' --type=merge

# Or edit interactively
kubectl edit pg my-database

# Watch the scaling
kubectl get pg my-database -w
# NAME          VERSION   REPLICAS   READY   PHASE     AGE
# my-database   16.2      5          3       Scaling   1m
# my-database   16.2      5          4       Scaling   1m15s
# my-database   16.2      5          5       Running   1m30s

# Scale down to 2 replicas
kubectl patch pg my-database -p '{"spec":{"replicas":2}}' --type=merge
```

### 34.9.4 Upgrading PostgreSQL Version

```bash
# Upgrade from 16.2 to 16.3
kubectl patch pg my-database -p '{"spec":{"version":"16.3"}}' --type=merge

# Watch the rolling upgrade
kubectl get pg my-database -w
# NAME          VERSION   REPLICAS   READY   PHASE       AGE
# my-database   16.3      3          3       Upgrading   2m
# my-database   16.3      3          2       Upgrading   2m15s
# my-database   16.3      3          3       Running     2m45s
```

### 34.9.5 Connecting to the Database

```bash
# Port-forward to the client Service
kubectl port-forward svc/my-database 5432:5432

# Connect with psql
psql -h localhost -U postgres -d postgres
```

### 34.9.6 Deleting the Cluster

```bash
# Delete the cluster — all child resources are automatically cleaned up
kubectl delete pg my-database

# Verify everything is gone
kubectl get all -l app.kubernetes.io/instance=my-database
# No resources found
```

---

## 34.10 Idempotency and Error Handling — Deep Dive

### 34.10.1 Why Idempotency Matters

The reconciler may be called multiple times for the same generation of a resource due to:

- **Informer resync** — periodic re-list of all resources
- **Child resource events** — a Pod restarting triggers reconciliation
- **Transient errors** — a failed API call causes a requeue
- **Leader election transitions** — the new leader re-reconciles everything

If the reconciler creates a new StatefulSet every time it's called, you'll end up with thousands
of StatefulSets. The reconciler must be idempotent: calling it N times produces the same result
as calling it once.

### 34.10.2 The Create-or-Update Pattern

The pattern we used throughout this chapter:

```go
// Pseudocode for idempotent reconciliation:
//
// 1. Build the desired state
// 2. Try to get the existing resource
// 3. If not found → create
// 4. If found → compare → update if different
// 5. If error → return error (requeue)
```

controller-runtime also provides `controllerutil.CreateOrUpdate` and
`controllerutil.CreateOrPatch` as helpers:

```go
// reconcileServiceAlternative demonstrates using CreateOrUpdate, which
// handles the get-or-create logic in a single call. The mutate function
// is called with the existing resource (or an empty one if creating),
// and you set the desired fields.
func (r *PostgresClusterReconciler) reconcileServiceAlternative(
	ctx context.Context,
	pg *postgresv1alpha1.PostgresCluster,
) error {
	labels := labelsForCluster(pg)

	svc := &corev1.Service{
		ObjectMeta: metav1.ObjectMeta{
			Name:      resourceName(pg),
			Namespace: pg.Namespace,
		},
	}

	// CreateOrUpdate: get the resource, call the mutate func, then
	// create or update as needed. The result tells you what happened.
	result, err := controllerutil.CreateOrUpdate(ctx, r.Client, svc, func() error {
			// Mutate function: set the desired fields.
			// This is called with either an empty Service (create) or the
			// existing Service (update).
			svc.Labels = labels
			svc.Spec.Selector = labels
			svc.Spec.Type = corev1.ServiceTypeClusterIP
			svc.Spec.Ports = []corev1.ServicePort{
				{
					Name:       "postgresql",
					Port:       5432,
					TargetPort: intstr.FromString("postgresql"),
					Protocol:   corev1.ProtocolTCP,
				},
			}
			return controllerutil.SetControllerReference(pg, svc, r.Scheme)
		})
	if err != nil {
		return fmt.Errorf("reconciling Service: %w", err)
	}

	log.FromContext(ctx).Info("Service reconciled",
		"name", svc.Name, "result", result)
	return nil
}
```

### 34.10.3 Error Handling and Requeue Strategy

```go
// The reconciler returns one of three outcomes:
//
// 1. Success (no requeue needed):
//    return ctrl.Result{}, nil
//
// 2. Transient error (requeue with exponential backoff):
//    return ctrl.Result{}, err
//
// 3. Explicit requeue (e.g., waiting for something):
//    return ctrl.Result{RequeueAfter: 10 * time.Second}, nil
//    return ctrl.Result{Requeue: true}, nil
```

For transient errors (e.g., API server temporarily unavailable), returning an error causes
controller-runtime to requeue with exponential backoff (starting at ~5ms, capped at ~16 minutes).

For expected waiting (e.g., Pods still starting), use `RequeueAfter` with a reasonable interval
(10–30 seconds). This avoids unnecessarily aggressive retries.

---

## 34.11 Summary

In this chapter, we built a complete Kubernetes operator from scratch:

| Component | What We Built |
|-----------|---------------|
| **CRD Types** | `PostgresCluster` with spec, status, conditions, and kubebuilder markers |
| **Reconciler** | Full reconcile loop that creates and manages StatefulSet, Services, ConfigMap |
| **Scaling** | Handles replica count changes via StatefulSet |
| **Upgrades** | Rolling version upgrade via StatefulSet rolling update |
| **Status** | Conditions (Ready, Available, Initialized, Progressing), phases, replica counts |
| **Garbage Collection** | Owner references on all child resources |
| **RBAC** | Least-privilege ClusterRole generated from markers |
| **Testing** | Unit tests (fake client), integration tests (envtest), E2E tests (kind) |
| **Deployment** | Dockerfile, kustomize, OLM basics |

The patterns we've learned — level-triggered reconciliation, idempotent operations, owner
references, status conditions, and the create-or-update pattern — are universal. They apply to
any operator, whether you're managing databases, message queues, certificates, or custom platform
resources.

In Chapter 35, we'll explore advanced operator patterns: finalizers for external cleanup, webhooks
for validation and mutation, leader election, multi-resource watching, custom metrics, and how
production-grade operators like cert-manager and the Prometheus Operator are designed.

---

## Exercises

1. **Add a Secret**: Modify the operator to generate a random password and store it in a Secret.
   Update the StatefulSet to read `POSTGRES_PASSWORD` from the Secret instead of a hardcoded value.

2. **Add a PodDisruptionBudget**: Create a PDB that ensures at least `replicas - 1` Pods are
   available during voluntary disruptions. Reconcile it alongside the other resources.

3. **Add separate Services for primary and replicas**: Create a `my-database-primary` Service that
   routes only to the Pod with ordinal 0, and a `my-database-replicas` Service that routes to all
   other Pods. (Hint: use `statefulset.kubernetes.io/pod-name` label.)

4. **Implement a health check**: Add a goroutine that periodically connects to each PostgreSQL
   instance and verifies it can execute a simple query. Report results in a new status condition.

5. **Add backup support**: Implement a CronJob that runs `pg_basebackup` according to the backup
   schedule in the spec. Store backups in a PersistentVolume or S3-compatible storage.
