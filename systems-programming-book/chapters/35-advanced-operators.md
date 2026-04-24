# Chapter 35: Advanced Operator Patterns

> *"The difference between a good operator and a great operator is how it handles the things that go wrong."*

In Chapter 34 we built a complete PostgresCluster operator that creates and manages Kubernetes
resources. That operator handles the happy path well — provisioning, scaling, upgrading. But
production operators must handle much more: graceful cleanup on deletion, input validation,
default injection, sophisticated status reporting, multi-resource coordination, leader election,
and observability.

This chapter covers the advanced patterns that separate toy operators from production-grade ones.
Every pattern is demonstrated with complete, documented Go code that you can adapt for your own
operators.

---

## 35.1 Finalizers

### 35.1.1 The Problem Finalizers Solve

When a user deletes a Kubernetes resource, the API server removes it immediately (or after a
grace period). But what if your operator needs to clean up **external** resources first?

Examples:
- A database operator needs to revoke cloud IAM bindings before the database is deleted
- A DNS operator needs to remove DNS records from an external provider
- A certificate operator needs to revoke certificates from an ACME provider
- A storage operator needs to delete cloud storage buckets

Without finalizers, the custom resource would be deleted before the operator has a chance to clean
up, leaving orphaned external resources.

### 35.1.2 How Finalizers Work

A **finalizer** is a string added to `metadata.finalizers` on a resource. When a resource has
finalizers, the API server does **not** delete it immediately. Instead:

1. The API server sets `metadata.deletionTimestamp` to the current time.
2. The resource enters a "terminating" state.
3. The API server waits for all finalizers to be removed.
4. Once `metadata.finalizers` is empty, the API server deletes the resource.

The operator's reconcile loop detects `deletionTimestamp != nil`, performs cleanup, and then
removes its finalizer string.

### 35.1.3 The Finalizer Flow

```
User: kubectl delete postgrescluster my-database

API Server:
  1. Sets deletionTimestamp = now
  2. Does NOT delete the resource (finalizers present)

Operator (next reconcile):
  1. Sees deletionTimestamp is set → "this resource is being deleted"
  2. Checks if our finalizer is in the list
  3. YES → run cleanup logic
  4. Remove our finalizer from the list
  5. Update the resource

API Server:
  1. All finalizers removed
  2. Actually deletes the resource
  3. Garbage collector deletes owned resources (StatefulSet, Services, etc.)
```

### 35.1.4 Complete Finalizer Implementation

```go
// Package controller implements advanced reconciliation patterns for the
// PostgresCluster operator. This file demonstrates finalizer-based cleanup
// for external resources that cannot be managed via Kubernetes owner references.
package controller

import (
	"context"
	"fmt"

	corev1 "k8s.io/api/core/v1"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/log"

	postgresv1alpha1 "github.com/example/postgres-operator/api/v1alpha1"
)

// finalizerName is a globally unique identifier for our operator's finalizer.
// It must be a valid DNS subdomain name (RFC 1123). Using the operator's
// domain ensures uniqueness across all controllers in the cluster.
const finalizerName = "postgres.example.com/cleanup"

// handleFinalizer manages the finalizer lifecycle for a PostgresCluster resource.
// It is called at the beginning of every reconcile invocation and handles two cases:
//
//  1. Resource is NOT being deleted: ensure the finalizer is present. If we add it,
//     we return immediately and requeue — the Update will trigger a new reconcile.
//
//  2. Resource IS being deleted (deletionTimestamp is set): run cleanup, remove the
//     finalizer, and return. The API server will then delete the resource.
//
// Returns:
//   - shouldReturn=true: the caller should return immediately (either we added the
//     finalizer and requeued, or we completed deletion cleanup).
//   - result, err: the ctrl.Result and error to return from Reconcile.
func (r *PostgresClusterReconciler) handleFinalizer(
	ctx context.Context,
	pg *postgresv1alpha1.PostgresCluster,
) (shouldReturn bool, result ctrl.Result, err error) {
	log := log.FromContext(ctx)

	// Case 1: Resource is NOT being deleted — ensure finalizer is present.
	if pg.DeletionTimestamp.IsZero() {
		if !controllerutil.ContainsFinalizer(pg, finalizerName) {
			log.Info("adding finalizer to PostgresCluster")
			controllerutil.AddFinalizer(pg, finalizerName)
			if err := r.Update(ctx, pg); err != nil {
				return true, ctrl.Result{}, fmt.Errorf("adding finalizer: %w", err)
			}
			// Requeue: the Update triggers a new reconcile with the finalizer present.
			return true, ctrl.Result{Requeue: true}, nil
		}
		// Finalizer already present — proceed with normal reconciliation.
		return false, ctrl.Result{}, nil
	}

	// Case 2: Resource IS being deleted — run cleanup and remove finalizer.
	if !controllerutil.ContainsFinalizer(pg, finalizerName) {
		// Our finalizer has already been removed (perhaps by a previous reconcile
		// that succeeded at cleanup but failed at a later step). Nothing to do.
		return true, ctrl.Result{}, nil
	}

	log.Info("PostgresCluster is being deleted — running cleanup")
	r.Recorder.Event(pg, corev1.EventTypeNormal, "Cleanup",
		"Running pre-deletion cleanup for external resources")

	// Perform external cleanup. This is where you would:
	// - Delete DNS records from Route53, CloudDNS, etc.
	// - Revoke IAM bindings in cloud providers
	// - Remove monitoring dashboards from Grafana
	// - Deregister from service meshes
	// - Clean up cloud storage buckets
	if err := r.cleanupExternalResources(ctx, pg); err != nil {
		// If cleanup fails, we return an error, which causes a requeue.
		// The finalizer is NOT removed, so the resource cannot be deleted
		// until cleanup succeeds.
		r.Recorder.Eventf(pg, corev1.EventTypeWarning, "CleanupFailed",
			"External cleanup failed: %v", err)
		return true, ctrl.Result{}, fmt.Errorf("cleaning up external resources: %w", err)
	}

	// Cleanup succeeded — remove the finalizer.
	log.Info("external cleanup complete — removing finalizer")
	controllerutil.RemoveFinalizer(pg, finalizerName)
	if err := r.Update(ctx, pg); err != nil {
		return true, ctrl.Result{}, fmt.Errorf("removing finalizer: %w", err)
	}

	r.Recorder.Event(pg, corev1.EventTypeNormal, "CleanupComplete",
		"Pre-deletion cleanup completed successfully")

	// Return without error. The API server will now delete the resource
	// (since no finalizers remain), and the garbage collector will delete
	// owned resources (StatefulSet, Services, etc.).
	return true, ctrl.Result{}, nil
}

// cleanupExternalResources performs cleanup of resources that live outside
// Kubernetes and cannot be managed via owner references. This function
// must be idempotent — it may be called multiple times if a previous
// reconcile failed after partial cleanup.
//
// In a real operator, this would call external APIs:
//   - Cloud provider APIs to delete DNS records
//   - IAM APIs to revoke service account bindings
//   - Monitoring APIs to remove dashboards and alerts
//
// Each cleanup step should be independently idempotent and should not
// fail if the external resource has already been deleted.
func (r *PostgresClusterReconciler) cleanupExternalResources(
	ctx context.Context,
	pg *postgresv1alpha1.PostgresCluster,
) error {
	log := log.FromContext(ctx)

	// Example: Delete DNS record from external DNS provider.
	dnsName := fmt.Sprintf("%s.%s.db.example.com", pg.Name, pg.Namespace)
	log.Info("deleting external DNS record", "dnsName", dnsName)
	if err := r.deleteExternalDNSRecord(ctx, dnsName); err != nil {
		return fmt.Errorf("deleting DNS record %s: %w", dnsName, err)
	}

	// Example: Remove monitoring configuration.
	log.Info("removing monitoring configuration", "cluster", pg.Name)
	if err := r.removeMonitoringConfig(ctx, pg); err != nil {
		return fmt.Errorf("removing monitoring config: %w", err)
	}

	return nil
}

// deleteExternalDNSRecord deletes a DNS record from an external DNS provider.
// This is a placeholder — in a real operator, this would call the Route53 API,
// Google Cloud DNS API, or another DNS provider's API.
//
// The function is idempotent: if the record doesn't exist, it returns nil.
func (r *PostgresClusterReconciler) deleteExternalDNSRecord(
	ctx context.Context,
	dnsName string,
) error {
	log := log.FromContext(ctx)
	// Placeholder: in production, call the DNS provider API.
	// Example with AWS Route53:
	//   _, err := route53Client.ChangeResourceRecordSets(ctx, &route53.ChangeResourceRecordSetsInput{
	//       HostedZoneId: aws.String(hostedZoneID),
	//       ChangeBatch: &types.ChangeBatch{
	//           Changes: []types.Change{{
	//               Action: types.ChangeActionDelete,
	//               ResourceRecordSet: &types.ResourceRecordSet{
	//                   Name: aws.String(dnsName),
	//                   Type: types.RRTypeCname,
	//               },
	//           }},
	//       },
	//   })
	log.Info("external DNS record deleted (simulated)", "dnsName", dnsName)
	return nil
}

// removeMonitoringConfig removes monitoring dashboards and alert rules
// associated with this PostgreSQL cluster from external monitoring systems.
//
// The function is idempotent: if the configuration doesn't exist, it returns nil.
func (r *PostgresClusterReconciler) removeMonitoringConfig(
	ctx context.Context,
	pg *postgresv1alpha1.PostgresCluster,
) error {
	log := log.FromContext(ctx)
	// Placeholder: in production, call the Grafana/Prometheus API.
	log.Info("monitoring configuration removed (simulated)", "cluster", pg.Name)
	return nil
}
```

### 35.1.5 Finalizer Pitfalls

**Pitfall 1: Infinite finalizer loop.** If cleanup always fails, the resource can never be
deleted. Always implement a timeout or manual override (e.g., an annotation that skips cleanup).

**Pitfall 2: Forgetting to add the finalizer before creating external resources.** If you create
external resources first, then the operator crashes before adding the finalizer, the resource
can be deleted without cleanup. Always add the finalizer *first*.

**Pitfall 3: Non-idempotent cleanup.** The cleanup function may be called multiple times (if it
succeeds but the finalizer removal fails). Each step must be safe to repeat.

---

## 35.2 Webhooks

Kubernetes **admission webhooks** intercept API requests before they are persisted to etcd.
They come in two flavors:

1. **Validating webhooks** — reject invalid resources (e.g., replicas must be odd for HA).
2. **Mutating webhooks** — modify resources before they are stored (e.g., set default values).

Webhooks run as HTTPS servers inside the operator. The API server calls them for every relevant
request.

### 35.2.1 Validating Webhooks

A validating webhook examines the incoming resource and returns either "allowed" or "denied" with
a reason. This is more powerful than CRD validation (kubebuilder markers) because you can implement
complex cross-field validation logic.

```go
// Package v1alpha1 contains webhook implementations for the PostgresCluster
// custom resource. Validating webhooks reject invalid resources before they
// are persisted; mutating webhooks inject defaults into resources.
package v1alpha1

import (
	"context"
	"fmt"

	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	logf "sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/webhook"
	"sigs.k8s.io/controller-runtime/pkg/webhook/admission"
)

// webhookLog is the logger for webhook operations.
var webhookLog = logf.Log.WithName("postgrescluster-webhook")

// SetupWebhookWithManager registers the validating and mutating webhooks
// with the controller-runtime Manager. The Manager's webhook server handles
// TLS termination and request routing.
func (r *PostgresCluster) SetupWebhookWithManager(mgr ctrl.Manager) error {
	return ctrl.NewWebhookManagedBy(mgr).
		For(r).
		Complete()
}

// Compile-time interface checks. These ensure that our type implements
// the webhook.Validator and webhook.Defaulter interfaces. If the methods
// are missing or have wrong signatures, this won't compile.
var _ webhook.CustomValidator = &PostgresCluster{}
var _ webhook.CustomDefaulter = &PostgresCluster{}

// --- Validating Webhook ---

// ValidateCreate implements webhook.CustomValidator. It is called when a new
// PostgresCluster is created. Returns an error if the resource is invalid.
//
// Validations performed:
//   - Replicas must be odd for high availability (so there's always a clear
//     majority for leader election in a Patroni/Stolon HA setup).
//   - Storage size must be at least 1Gi.
//   - PostgreSQL version must be in the supported list.
//   - Backup retention must not exceed 365 days.
//
// +kubebuilder:webhook:path=/validate-postgres-example-com-v1alpha1-postgrescluster,mutating=false,failurePolicy=fail,sideEffects=None,groups=postgres.example.com,resources=postgresclusters,verbs=create;update,versions=v1alpha1,name=vpostgrescluster.kb.io,admissionReviewVersions=v1
func (r *PostgresCluster) ValidateCreate(
	ctx context.Context,
	obj runtime.Object,
) (admission.Warnings, error) {
	pg, ok := obj.(*PostgresCluster)
	if !ok {
		return nil, fmt.Errorf("expected PostgresCluster, got %T", obj)
	}
	webhookLog.Info("validating create", "name", pg.Name)
	return validatePostgresCluster(pg)
}

// ValidateUpdate implements webhook.CustomValidator. It is called when an
// existing PostgresCluster is updated. Validates both the new spec and
// any immutable field changes.
//
// Additional validations for updates:
//   - StorageClassName cannot be changed after creation (PVC storage class
//     is immutable in Kubernetes).
func (r *PostgresCluster) ValidateUpdate(
	ctx context.Context,
	oldObj, newObj runtime.Object,
) (admission.Warnings, error) {
	oldPG, ok := oldObj.(*PostgresCluster)
	if !ok {
		return nil, fmt.Errorf("expected PostgresCluster for old, got %T", oldObj)
	}
	newPG, ok := newObj.(*PostgresCluster)
	if !ok {
		return nil, fmt.Errorf("expected PostgresCluster for new, got %T", newObj)
	}
	webhookLog.Info("validating update", "name", newPG.Name)

	// First, run all creation-time validations on the new spec.
	warnings, err := validatePostgresCluster(newPG)
	if err != nil {
		return warnings, err
	}

	// Then, check immutable fields.
	if oldPG.Spec.Storage.StorageClassName != nil &&
		newPG.Spec.Storage.StorageClassName != nil &&
		*oldPG.Spec.Storage.StorageClassName != *newPG.Spec.Storage.StorageClassName {
		return warnings, fmt.Errorf(
			"spec.storage.storageClassName is immutable: cannot change from %q to %q",
			*oldPG.Spec.Storage.StorageClassName,
			*newPG.Spec.Storage.StorageClassName,
		)
	}

	return warnings, nil
}

// ValidateDelete implements webhook.CustomValidator. It is called when a
// PostgresCluster is deleted. We allow all deletions — cleanup is handled
// by the finalizer.
func (r *PostgresCluster) ValidateDelete(
	ctx context.Context,
	obj runtime.Object,
) (admission.Warnings, error) {
	return nil, nil
}

// supportedVersions is the list of PostgreSQL versions this operator supports.
// When a user specifies a version not in this list, the validating webhook
// rejects the resource with a clear error message.
var supportedVersions = map[string]bool{
	"14.11": true,
	"14.12": true,
	"15.6":  true,
	"15.7":  true,
	"16.2":  true,
	"16.3":  true,
}

// validatePostgresCluster performs comprehensive validation of a PostgresCluster
// spec. It returns admission warnings (non-blocking) and an error (blocking).
//
// Validation rules:
//   - Replicas > 1 must be odd for HA quorum
//   - Version must be in the supported list
//   - Storage size must be at least 1Gi
//   - Backup schedule must be valid (basic check)
//   - Resource limits must be >= requests
func validatePostgresCluster(pg *PostgresCluster) (admission.Warnings, error) {
	var warnings admission.Warnings

	// Validate replicas: must be odd for HA (Patroni requires a quorum).
	// Single replica (1) is allowed for development environments.
	if pg.Spec.Replicas > 1 && pg.Spec.Replicas%2 == 0 {
		return warnings, fmt.Errorf(
			"spec.replicas must be odd for high availability (got %d); "+
				"use 1, 3, 5, 7, or 9 replicas",
			pg.Spec.Replicas,
		)
	}

	// Validate PostgreSQL version.
	if !supportedVersions[pg.Spec.Version] {
		supported := make([]string, 0, len(supportedVersions))
		for v := range supportedVersions {
			supported = append(supported, v)
		}
		return warnings, fmt.Errorf(
			"spec.version %q is not supported; supported versions: %v",
			pg.Spec.Version, supported,
		)
	}

	// Validate storage size: at least 1Gi.
	oneGi := MustParseQuantity("1Gi")
	if pg.Spec.Storage.Size.Cmp(oneGi) < 0 {
		return warnings, fmt.Errorf(
			"spec.storage.size must be at least 1Gi (got %s)",
			pg.Spec.Storage.Size.String(),
		)
	}

	// Warn if storage is less than 10Gi (likely too small for production).
	tenGi := MustParseQuantity("10Gi")
	if pg.Spec.Storage.Size.Cmp(tenGi) < 0 {
		warnings = append(warnings,
			"spec.storage.size is less than 10Gi — this may be too small for production workloads")
	}

	// Warn if no backup is configured.
	if pg.Spec.Backup == nil {
		warnings = append(warnings,
			"spec.backup is not configured — no automated backups will be performed")
	}

	// Validate that resource limits >= requests (if both are specified).
	if pg.Spec.Resources.Limits != nil && pg.Spec.Resources.Requests != nil {
		cpuLimit := pg.Spec.Resources.Limits.Cpu()
		cpuRequest := pg.Spec.Resources.Requests.Cpu()
		if cpuLimit != nil && cpuRequest != nil && cpuLimit.Cmp(*cpuRequest) < 0 {
			return warnings, fmt.Errorf(
				"spec.resources.limits.cpu (%s) must be >= spec.resources.requests.cpu (%s)",
				cpuLimit.String(), cpuRequest.String(),
			)
		}

		memLimit := pg.Spec.Resources.Limits.Memory()
		memRequest := pg.Spec.Resources.Requests.Memory()
		if memLimit != nil && memRequest != nil && memLimit.Cmp(*memRequest) < 0 {
			return warnings, fmt.Errorf(
				"spec.resources.limits.memory (%s) must be >= spec.resources.requests.memory (%s)",
				memLimit.String(), memRequest.String(),
			)
		}
	}

	return warnings, nil
}
```

### 35.2.2 Mutating Webhooks (Defaulting)

A mutating webhook modifies the resource before it is stored. This is how operators inject
default values, add labels, or fill in computed fields.

```go
// --- Mutating Webhook (Defaulter) ---

// Default implements webhook.CustomDefaulter. It is called before validation
// on both create and update operations. It sets default values for fields
// that the user did not specify.
//
// Defaults applied:
//   - Version: "16.2" (latest stable)
//   - Replicas: 1 (single instance for development)
//   - Storage size: "10Gi"
//   - Backup retention: 7 days (if backup is configured)
//   - PostgreSQL parameters: sensible defaults for max_connections, shared_buffers
//
// +kubebuilder:webhook:path=/mutate-postgres-example-com-v1alpha1-postgrescluster,mutating=true,failurePolicy=fail,sideEffects=None,groups=postgres.example.com,resources=postgresclusters,verbs=create;update,versions=v1alpha1,name=mpostgrescluster.kb.io,admissionReviewVersions=v1
func (r *PostgresCluster) Default(ctx context.Context, obj runtime.Object) error {
	pg, ok := obj.(*PostgresCluster)
	if !ok {
		return fmt.Errorf("expected PostgresCluster, got %T", obj)
	}
	webhookLog.Info("applying defaults", "name", pg.Name)

	// Default version to the latest stable release.
	if pg.Spec.Version == "" {
		pg.Spec.Version = "16.2"
		webhookLog.Info("defaulted spec.version", "value", pg.Spec.Version)
	}

	// Default replicas to 1 (single instance — suitable for development).
	if pg.Spec.Replicas == 0 {
		pg.Spec.Replicas = 1
		webhookLog.Info("defaulted spec.replicas", "value", pg.Spec.Replicas)
	}

	// Default storage size to 10Gi.
	if pg.Spec.Storage.Size.IsZero() {
		pg.Spec.Storage.Size = MustParseQuantity("10Gi")
		webhookLog.Info("defaulted spec.storage.size", "value", "10Gi")
	}

	// Default backup retention to 7 days (if backup is configured).
	if pg.Spec.Backup != nil && pg.Spec.Backup.RetentionDays == 0 {
		pg.Spec.Backup.RetentionDays = 7
		webhookLog.Info("defaulted spec.backup.retentionDays", "value", 7)
	}

	// Default PostgreSQL parameters.
	if pg.Spec.Parameters == nil {
		pg.Spec.Parameters = make(map[string]string)
	}
	defaultParams := map[string]string{
		"max_connections": "100",
		"shared_buffers":  "128MB",
		"wal_level":       "replica",
	}
	for k, v := range defaultParams {
		if _, exists := pg.Spec.Parameters[k]; !exists {
			pg.Spec.Parameters[k] = v
			webhookLog.Info("defaulted parameter", "key", k, "value", v)
		}
	}

	// Add standard labels if not present.
	if pg.Labels == nil {
		pg.Labels = make(map[string]string)
	}
	if _, exists := pg.Labels["app.kubernetes.io/managed-by"]; !exists {
		pg.Labels["app.kubernetes.io/managed-by"] = "postgres-operator"
	}

	return nil
}

// MustParseQuantity is a helper that parses a Kubernetes resource quantity
// string and panics on error. Only use for known-good constant values.
func MustParseQuantity(s string) resource.Quantity {
	q, err := resource.ParseQuantity(s)
	if err != nil {
		panic(fmt.Sprintf("BUG: failed to parse quantity %q: %v", s, err))
	}
	return q
}
```

### 35.2.3 Webhook Configuration and Deployment

Webhooks require TLS certificates because the API server communicates with them over HTTPS. The
standard approach is to use **cert-manager** to automatically provision and rotate certificates.

**Step 1: Generate webhook manifests**

Add the webhook marker flag to the Makefile's `manifests` target:

```makefile
manifests: controller-gen
	$(CONTROLLER_GEN) rbac:roleName=manager-role crd webhook paths="./..." output:crd:artifacts:config=config/crd/bases
```

Run `make manifests` to generate `config/webhook/manifests.yaml`.

**Step 2: Configure cert-manager**

```yaml
# config/default/kustomization.yaml additions:

# Enable webhook and cert-manager patches
resources:
- ../crd
- ../rbac
- ../manager
- ../webhook    # <-- enable
- ../certmanager # <-- enable

patches:
- path: manager_webhook_patch.yaml
- path: webhookcainjection_patch.yaml
```

**Step 3: Install cert-manager**

```bash
# Install cert-manager into the cluster
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml

# Wait for cert-manager to be ready
kubectl -n cert-manager wait --for=condition=Available deployment --all --timeout=120s
```

**Step 4: Deploy the operator with webhooks**

```bash
make deploy IMG=myregistry.com/postgres-operator:v0.1.0
```

cert-manager automatically creates a self-signed CA, issues a certificate for the webhook
server, and injects the CA bundle into the webhook configurations.

### 35.2.4 Testing Webhooks Locally

For local development with `make run`, webhooks need special handling because the API server
needs to reach the webhook server over HTTPS.

```bash
# Option 1: Disable webhooks for local development
# In main.go, conditionally register webhooks based on an environment variable:
# if os.Getenv("ENABLE_WEBHOOKS") != "false" {
#     if err = (&postgresv1alpha1.PostgresCluster{}).SetupWebhookWithManager(mgr); err != nil {
#         ...
#     }
# }

ENABLE_WEBHOOKS=false make run

# Option 2: Use envtest (it handles webhook certificates automatically)
make test
```

---

## 35.3 Status Conditions — The Right Way

### 35.3.1 Why Conditions Over Phases

The Kubernetes community has converged on **conditions** as the standard way to report resource
status. Conditions are superior to a single "phase" field because:

1. **Multiple independent signals** — a resource can be simultaneously "Available" (at least one
   replica works), "Degraded" (fewer replicas than desired), and "Progressing" (upgrade in progress).
   A single phase cannot represent this.

2. **Standard tooling** — `kubectl wait --for=condition=Ready` works generically. Scripts and
   controllers can watch specific conditions without understanding your operator's custom phases.

3. **Machine-parseable** — each condition has a structured type, status (True/False/Unknown),
   reason (CamelCase enum), and human-readable message.

### 35.3.2 The Condition Struct

```go
// metav1.Condition represents a single status condition. The standard fields are:
//
//   Type:               string  — The condition type (e.g., "Ready", "Available").
//                                  Should be CamelCase and unique within the conditions list.
//
//   Status:             string  — "True", "False", or "Unknown".
//
//   ObservedGeneration: int64   — The generation of the resource when this condition was set.
//                                  If this doesn't match metadata.generation, the condition is stale.
//
//   LastTransitionTime: Time    — When the condition last changed status (True→False, etc.).
//                                  Does NOT update when the reason/message changes without a
//                                  status change.
//
//   Reason:             string  — A CamelCase, machine-readable reason for the condition.
//                                  Examples: "AllReplicasReady", "InsufficientReplicas".
//
//   Message:            string  — A human-readable description of the condition.
//                                  Examples: "All 3 replicas are ready", "Only 2 of 3 replicas are ready".
```

### 35.3.3 Standard Condition Types

The Kubernetes API conventions recommend these standard condition types:

| Condition Type | Meaning | True When |
|---------------|---------|-----------|
| **Ready** | The resource is fully operational | All replicas are ready, no errors |
| **Available** | The resource is at least partially operational | At least one replica is ready |
| **Progressing** | The operator is making changes | Scaling, upgrading, or creating |
| **Degraded** | The resource is operational but below desired state | Fewer replicas than desired |

### 35.3.4 Complete Status Condition Management

```go
// Package status provides utilities for managing Kubernetes status conditions
// on PostgresCluster resources. It encapsulates the logic for setting,
// updating, and querying conditions following Kubernetes API conventions.
package status

import (
	"fmt"
	"time"

	"k8s.io/apimachinery/pkg/api/meta"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"

	postgresv1alpha1 "github.com/example/postgres-operator/api/v1alpha1"
)

// ConditionManager encapsulates the logic for managing status conditions
// on a PostgresCluster resource. It provides methods for setting common
// conditions and handles the details of LastTransitionTime, ObservedGeneration,
// and condition deduplication.
type ConditionManager struct {
	// pg is the PostgresCluster whose conditions we are managing.
	pg *postgresv1alpha1.PostgresCluster
}

// NewConditionManager creates a ConditionManager for the given PostgresCluster.
func NewConditionManager(pg *postgresv1alpha1.PostgresCluster) *ConditionManager {
	return &ConditionManager{pg: pg}
}

// SetReady sets the "Ready" condition based on whether all desired replicas
// are available. The Ready condition is the primary indicator of cluster health.
//
// Ready is True when:
//   - All desired replicas are running and ready
//   - No errors have occurred during the current reconcile
//
// Ready is False when:
//   - Some replicas are not ready
//   - An error occurred during reconciliation
func (cm *ConditionManager) SetReady(readyReplicas, desiredReplicas int32) {
	if readyReplicas >= desiredReplicas && desiredReplicas > 0 {
		meta.SetStatusCondition(&cm.pg.Status.Conditions, metav1.Condition{
			Type:               postgresv1alpha1.ConditionTypeReady,
			Status:             metav1.ConditionTrue,
			ObservedGeneration: cm.pg.Generation,
			Reason:             "AllReplicasReady",
			Message: fmt.Sprintf(
				"All %d desired replicas are ready", desiredReplicas),
			LastTransitionTime: metav1.Now(),
		})
	} else {
		meta.SetStatusCondition(&cm.pg.Status.Conditions, metav1.Condition{
			Type:               postgresv1alpha1.ConditionTypeReady,
			Status:             metav1.ConditionFalse,
			ObservedGeneration: cm.pg.Generation,
			Reason:             "ReplicasNotReady",
			Message: fmt.Sprintf(
				"%d of %d desired replicas are ready",
				readyReplicas, desiredReplicas),
			LastTransitionTime: metav1.Now(),
		})
	}
}

// SetAvailable sets the "Available" condition based on whether at least one
// replica is ready to serve traffic. A cluster can be Available but not Ready
// (e.g., 1 of 3 replicas is up).
func (cm *ConditionManager) SetAvailable(readyReplicas int32) {
	if readyReplicas > 0 {
		meta.SetStatusCondition(&cm.pg.Status.Conditions, metav1.Condition{
			Type:               postgresv1alpha1.ConditionTypeAvailable,
			Status:             metav1.ConditionTrue,
			ObservedGeneration: cm.pg.Generation,
			Reason:             "AtLeastOneReplicaAvailable",
			Message: fmt.Sprintf(
				"%d replica(s) available for connections", readyReplicas),
			LastTransitionTime: metav1.Now(),
		})
	} else {
		meta.SetStatusCondition(&cm.pg.Status.Conditions, metav1.Condition{
			Type:               postgresv1alpha1.ConditionTypeAvailable,
			Status:             metav1.ConditionFalse,
			ObservedGeneration: cm.pg.Generation,
			Reason:             "NoReplicasAvailable",
			Message:            "No replicas are available for connections",
			LastTransitionTime: metav1.Now(),
		})
	}
}

// SetProgressing sets the "Progressing" condition based on whether the
// operator is actively making changes to the cluster. This includes
// scaling, upgrading, or initial creation.
func (cm *ConditionManager) SetProgressing(isProgressing bool, reason, message string) {
	status := metav1.ConditionFalse
	if isProgressing {
		status = metav1.ConditionTrue
	}
	meta.SetStatusCondition(&cm.pg.Status.Conditions, metav1.Condition{
		Type:               postgresv1alpha1.ConditionTypeProgressing,
		Status:             status,
		ObservedGeneration: cm.pg.Generation,
		Reason:             reason,
		Message:            message,
		LastTransitionTime: metav1.Now(),
	})
}

// SetDegraded sets a "Degraded" condition when the cluster is operational
// but not at full capacity (e.g., 2 of 3 replicas are ready).
func (cm *ConditionManager) SetDegraded(isDegraded bool, reason, message string) {
	status := metav1.ConditionFalse
	if isDegraded {
		status = metav1.ConditionTrue
	}
	meta.SetStatusCondition(&cm.pg.Status.Conditions, metav1.Condition{
		Type:               "Degraded",
		Status:             status,
		ObservedGeneration: cm.pg.Generation,
		Reason:             reason,
		Message:            message,
		LastTransitionTime: metav1.Now(),
	})
}

// SetError sets a "Ready" condition to False with error details. This is
// used when the reconciler encounters an error that prevents the cluster
// from being fully healthy.
func (cm *ConditionManager) SetError(err error) {
	meta.SetStatusCondition(&cm.pg.Status.Conditions, metav1.Condition{
		Type:               postgresv1alpha1.ConditionTypeReady,
		Status:             metav1.ConditionFalse,
		ObservedGeneration: cm.pg.Generation,
		Reason:             "ReconcileError",
		Message:            fmt.Sprintf("Reconciliation error: %v", err),
		LastTransitionTime: metav1.Now(),
	})
}

// IsReady returns true if the "Ready" condition is True. This is a
// convenience method for checking cluster health in other parts of
// the operator code.
func (cm *ConditionManager) IsReady() bool {
	cond := meta.FindStatusCondition(cm.pg.Status.Conditions, postgresv1alpha1.ConditionTypeReady)
	return cond != nil && cond.Status == metav1.ConditionTrue
}

// ConditionChangedRecently returns true if the given condition's
// LastTransitionTime is within the given duration. Useful for debouncing
// notifications or avoiding rapid status flapping.
func (cm *ConditionManager) ConditionChangedRecently(
	conditionType string,
	within time.Duration,
) bool {
	cond := meta.FindStatusCondition(cm.pg.Status.Conditions, conditionType)
	if cond == nil {
		return false
	}
	return time.Since(cond.LastTransitionTime.Time) < within
}
```

---

## 35.4 Owner References — Advanced Patterns

### 35.4.1 Single Owner vs Non-Controller Owners

Each Kubernetes resource can have multiple owner references, but **at most one** can have
`controller: true`. The controller owner is the "primary" manager of the resource.

```go
// SetControllerReference sets an owner reference with controller=true.
// Only ONE controller owner is allowed per resource. This is the standard
// method used by operators.
controllerutil.SetControllerReference(owner, child, scheme)

// SetOwnerReference sets a non-controller owner reference.
// Multiple non-controller owners are allowed. The resource is only
// garbage collected when ALL owners (controller + non-controller) are deleted.
controllerutil.SetOwnerReference(owner, child, scheme)

```

### 35.4.2 Cross-Namespace Ownership

**Kubernetes does not support cross-namespace owner references.** An owner and its owned
resources must be in the same namespace (or the owner must be a cluster-scoped resource).

Workarounds for cross-namespace relationships:

```go
// WorkaroundOption1_Labels uses labels to track relationships that cross
// namespaces. This does not provide automatic garbage collection — you
// must implement cleanup manually (e.g., via finalizers).
//
// The child resource in another namespace gets a label pointing to the owner:
//
//	labels:
//	  postgres.example.com/owner-name: my-database
//	  postgres.example.com/owner-namespace: production
//
// During finalizer cleanup, the operator lists resources with these labels
// and deletes them.
func labelsForCrossNamespace(ownerName, ownerNamespace string) map[string]string {
	return map[string]string{
		"postgres.example.com/owner-name":      ownerName,
		"postgres.example.com/owner-namespace": ownerNamespace,
	}
}

// WorkaroundOption2_Annotation uses annotations to track cross-namespace
// relationships. Similar to labels but with more flexibility for complex
// ownership metadata.
func annotationsForCrossNamespace(ownerName, ownerNamespace, ownerUID string) map[string]string {
	return map[string]string{
		"postgres.example.com/owner-ref": fmt.Sprintf("%s/%s/%s",
			ownerNamespace, ownerName, ownerUID),
	}
}
```

### 35.4.3 Cascading Deletion Policies

```go
// demonstrateDeletionPolicies shows how to use different garbage collection
// policies when deleting a PostgresCluster.
func demonstrateDeletionPolicies(ctx context.Context, c client.Client) {
	pg := &postgresv1alpha1.PostgresCluster{
		ObjectMeta: metav1.ObjectMeta{
			Name:      "my-database",
			Namespace: "default",
		},
	}

	// Background deletion (default): the owner is deleted immediately.
	// The garbage collector asynchronously deletes children.
	bgPolicy := metav1.DeletePropagationBackground
	_ = c.Delete(ctx, pg, &client.DeleteOptions{
		PropagationPolicy: &bgPolicy,
	})

	// Foreground deletion: children are deleted first. The owner enters
	// a "deleting" state and is only removed after all children are gone.
	// This is useful when you need to ensure children are cleaned up
	// before the parent disappears.
	fgPolicy := metav1.DeletePropagationForeground
	_ = c.Delete(ctx, pg, &client.DeleteOptions{
		PropagationPolicy: &fgPolicy,
	})

	// Orphan deletion: the owner is deleted but children are left running.
	// The owner references are removed from the children, making them
	// standalone resources. Useful for "detaching" resources.
	orphanPolicy := metav1.DeletePropagationOrphan
	_ = c.Delete(ctx, pg, &client.DeleteOptions{
		PropagationPolicy: &orphanPolicy,
	})
}
```

---

## 35.5 Event Recording

### 35.5.1 Why Events Matter

Kubernetes Events are the audit trail of your operator. They appear in `kubectl describe` output
and in monitoring dashboards. Good event recording turns debugging from guesswork into reading
a log.

### 35.5.2 Using the EventRecorder

```go
// Package events demonstrates best practices for recording Kubernetes Events
// from an operator. Events provide an audit trail visible via kubectl describe
// and integrated with monitoring and alerting systems.
package controller

import (
	"context"
	"fmt"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/client-go/tools/record"
)

// EventReasons defines standard event reason strings for the PostgresCluster
// operator. Reasons should be CamelCase, short, and machine-parseable.
// They serve as enum values that monitoring systems can alert on.
const (
	// EventReasonCreated indicates a child resource was created.
	EventReasonCreated = "Created"

	// EventReasonUpdated indicates a child resource was updated.
	EventReasonUpdated = "Updated"

	// EventReasonScalingUp indicates the cluster is adding replicas.
	EventReasonScalingUp = "ScalingUp"

	// EventReasonScalingDown indicates the cluster is removing replicas.
	EventReasonScalingDown = "ScalingDown"

	// EventReasonUpgrading indicates a version upgrade is in progress.
	EventReasonUpgrading = "Upgrading"

	// EventReasonUpgradeComplete indicates a version upgrade has completed.
	EventReasonUpgradeComplete = "UpgradeComplete"

	// EventReasonReconcileError indicates an error during reconciliation.
	EventReasonReconcileError = "ReconcileError"

	// EventReasonHealthCheckFailed indicates a health check detected an issue.
	EventReasonHealthCheckFailed = "HealthCheckFailed"

	// EventReasonBackupStarted indicates an automated backup has started.
	EventReasonBackupStarted = "BackupStarted"

	// EventReasonBackupComplete indicates an automated backup has completed.
	EventReasonBackupComplete = "BackupComplete"

	// EventReasonBackupFailed indicates an automated backup has failed.
	EventReasonBackupFailed = "BackupFailed"
)

// recordEvent is a helper that records an event on the PostgresCluster
// resource with consistent formatting. It wraps the EventRecorder to
// add standard metadata.
//
// eventType should be corev1.EventTypeNormal for routine operations or
// corev1.EventTypeWarning for issues that may need attention.
//
// Examples:
//
//	recordEvent(recorder, pg, corev1.EventTypeNormal, EventReasonCreated,
//	    "Created StatefulSet %s with %d replicas", stsName, replicas)
//
//	recordEvent(recorder, pg, corev1.EventTypeWarning, EventReasonReconcileError,
//	    "Failed to create Service: %v", err)
func recordEvent(
	recorder record.EventRecorder,
	pg *postgresv1alpha1.PostgresCluster,
	eventType string,
	reason string,
	messageFmt string,
	args ...interface{},
) {
	recorder.Eventf(pg, eventType, reason, messageFmt, args...)
}

// recordReconcileEvents demonstrates recording events throughout the
// reconcile loop. Events should be recorded for:
//   - Resource creation (Normal)
//   - Resource updates (Normal)
//   - Scaling events (Normal)
//   - Upgrade events (Normal)
//   - Errors (Warning)
//   - Health check failures (Warning)
func (r *PostgresClusterReconciler) recordReconcileEvents(
	ctx context.Context,
	pg *postgresv1alpha1.PostgresCluster,
) {
	// Example: recording a scaling event.
	r.Recorder.Eventf(pg, corev1.EventTypeNormal, EventReasonScalingUp,
		"Scaling cluster from %d to %d replicas",
		pg.Status.CurrentReplicas, pg.Spec.Replicas)

	// Example: recording an error.
	r.Recorder.Eventf(pg, corev1.EventTypeWarning, EventReasonReconcileError,
		"Failed to update StatefulSet: context deadline exceeded")

	// Example: recording a version upgrade.
	r.Recorder.Eventf(pg, corev1.EventTypeNormal, EventReasonUpgrading,
		"Upgrading PostgreSQL from %s to %s",
		pg.Status.CurrentVersion, pg.Spec.Version)
}
```

### 35.5.3 Viewing Events

```bash
# Events on a specific resource
kubectl describe pg my-database

# All events in a namespace, sorted by time
kubectl get events --sort-by=.metadata.creationTimestamp

# Watch events in real-time
kubectl get events -w

# Filter by reason
kubectl get events --field-selector reason=ScalingUp
```

---

## 35.6 Multi-Resource Watching

### 35.6.1 The Owns() Pattern

In Chapter 34 we used `Owns()` to watch resources created by our operator:

```go
// SetupWithManager registers watches on owned resources. When a StatefulSet,
// Service, or ConfigMap that has an owner reference pointing to a PostgresCluster
// changes, controller-runtime automatically maps the event to the owning
// PostgresCluster and enqueues a reconciliation.
func (r *PostgresClusterReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&postgresv1alpha1.PostgresCluster{}).
		Owns(&appsv1.StatefulSet{}).
		Owns(&corev1.Service{}).
		Owns(&corev1.ConfigMap{}).
		Complete(r)
}
```

### 35.6.2 Watching External Resources

Sometimes you need to watch resources that your operator doesn't own. For example, watching
Nodes to detect when a node fails and trigger a failover.

```go
// SetupWithManagerAdvanced shows how to watch external resources — resources
// that are not owned by the PostgresCluster but whose changes are relevant
// to reconciliation.
//
// Use cases:
//   - Watch Nodes to detect failures and trigger failover
//   - Watch Secrets to detect credential rotation
//   - Watch NetworkPolicies to detect connectivity changes
//   - Watch StorageClasses to detect storage backend changes
func (r *PostgresClusterReconciler) SetupWithManagerAdvanced(
	mgr ctrl.Manager,
) error {
	return ctrl.NewControllerManagedBy(mgr).
		// Primary watch: PostgresCluster resources.
		For(&postgresv1alpha1.PostgresCluster{}).

		// Owned resources: changes trigger reconcile of the owner.
		Owns(&appsv1.StatefulSet{}).
		Owns(&corev1.Service{}).
		Owns(&corev1.ConfigMap{}).

		// Watch Secrets with a specific label. When a Secret with label
		// "postgres.example.com/watched=true" changes, find all
		// PostgresClusters in the same namespace and reconcile them.
		Watches(
			&corev1.Secret{},
			handler.EnqueueRequestsFromMapFunc(
				r.findPostgresClustersForSecret,
			),
			builder.WithPredicates(predicate.NewPredicateFuncs(func(obj client.Object) bool {
				// Only watch Secrets with our label.
				return obj.GetLabels()["postgres.example.com/watched"] == "true"
			})),
		).
		Complete(r)
}

// findPostgresClustersForSecret maps a Secret change to the PostgresClusters
// that should be reconciled. This is called by controller-runtime when a
// watched Secret changes.
//
// The mapping logic finds all PostgresClusters in the same namespace as the
// Secret. In a more sophisticated operator, you might use annotations on
// the Secret to identify which specific PostgresCluster references it.
func (r *PostgresClusterReconciler) findPostgresClustersForSecret(
	ctx context.Context,
	obj client.Object,
) []ctrl.Request {
	log := log.FromContext(ctx)

	// List all PostgresClusters in the Secret's namespace.
	pgList := &postgresv1alpha1.PostgresClusterList{}
	err := r.List(ctx, pgList, client.InNamespace(obj.GetNamespace()))
	if err != nil {
		log.Error(err, "unable to list PostgresClusters for Secret watch")
		return nil
	}

	// Enqueue a reconcile request for each PostgresCluster.
	requests := make([]ctrl.Request, 0, len(pgList.Items))
	for _, pg := range pgList.Items {
		requests = append(requests, ctrl.Request{
			NamespacedName: types.NamespacedName{
				Name:      pg.Name,
				Namespace: pg.Namespace,
			},
		})
	}

	log.Info("Secret changed — enqueueing PostgresClusters",
		"secret", obj.GetName(),
		"count", len(requests),
	)

	return requests
}
```

### 35.6.3 Watching Pods for Health Monitoring

```go
// watchPodsForHealth demonstrates watching individual Pods (not just the
// StatefulSet) to detect health issues early. When a Pod transitions to
// a failed state, we can trigger immediate reconciliation rather than
// waiting for the StatefulSet controller to report the change.
//
// This is particularly useful for:
//   - Detecting OOMKilled containers before the StatefulSet notices
//   - Monitoring CrashLoopBackOff early
//   - Tracking individual replica health for status conditions
func (r *PostgresClusterReconciler) watchPodsForHealth(
	mgr ctrl.Manager,
) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&postgresv1alpha1.PostgresCluster{}).
		Owns(&appsv1.StatefulSet{}).
		Watches(
			&corev1.Pod{},
			handler.EnqueueRequestForOwner(
				mgr.GetScheme(),
				mgr.GetRESTMapper(),
				&appsv1.StatefulSet{},
			),
			builder.WithPredicates(predicate.NewPredicateFuncs(func(obj client.Object) bool {
				// Only watch Pods with our label.
				labels := obj.GetLabels()
				return labels["app.kubernetes.io/managed-by"] == "postgres-operator"
			})),
		).
		Complete(r)
}
```

---

## 35.7 Leader Election

### 35.7.1 Why Leader Election

When you deploy multiple replicas of your operator (for high availability), only **one** instance
should actively reconcile resources at a time. Without leader election, multiple instances would
compete, causing conflicts and duplicate actions.

controller-runtime provides built-in leader election using Kubernetes Lease objects.

### 35.7.2 How It Works

1. All operator instances start and try to acquire a Lease (a Kubernetes resource).
2. The first instance to acquire the Lease becomes the **leader**.
3. The leader runs reconciliation loops.
4. Non-leader instances idle, watching the Lease.
5. If the leader crashes or loses the Lease, a non-leader takes over.
6. The transition is automatic — typically 15–30 seconds.

### 35.7.3 Configuring Leader Election

```go
// configureLeaderElection shows how to configure leader election for the
// operator Manager. Leader election ensures that only one operator instance
// actively reconciles resources, even when multiple replicas are deployed
// for high availability.
//
// The leader election Lease is a Kubernetes resource (kind: Lease) in the
// operator's namespace. All operator instances compete for this Lease.
// The holder of the Lease is the leader.
func configureLeaderElection() (ctrl.Manager, error) {
	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
		Scheme: scheme,

		// LeaderElection enables leader election. When true, only the leader
		// instance starts controllers. Non-leader instances idle until they
		// acquire the Lease.
		LeaderElection: true,

		// LeaderElectionID is the name of the Lease resource used for
		// leader election. It must be unique per operator (if multiple
		// operators share a namespace).
		LeaderElectionID: "postgres-operator.example.com",

		// LeaderElectionNamespace is the namespace for the Lease.
		// Defaults to the operator's namespace (read from the downward API
		// or the WATCH_NAMESPACE environment variable).
		LeaderElectionNamespace: "postgres-operator-system",

		// LeaderElectionReleaseOnCancel controls whether the leader releases
		// the Lease when it shuts down gracefully. When true, leadership
		// transitions are faster (no need to wait for the Lease to expire).
		// When false (default), the Lease must expire before a new leader
		// is elected (takes LeaseDuration).
		LeaderElectionReleaseOnCancel: true,

		// LeaseDuration is how long a non-leader waits before trying to
		// acquire the Lease after the leader's last heartbeat. Default: 15s.
		// Lower values = faster failover but more API server load.
		LeaseDuration: durationPtr(15 * time.Second),

		// RenewDeadline is how long the leader has to renew its Lease
		// before it expires. Must be < LeaseDuration. Default: 10s.
		RenewDeadline: durationPtr(10 * time.Second),

		// RetryPeriod is how often the operator retries acquiring or
		// renewing the Lease. Must be < RenewDeadline. Default: 2s.
		RetryPeriod: durationPtr(2 * time.Second),
	})
	return mgr, err
}

// durationPtr returns a pointer to a time.Duration value.
func durationPtr(d time.Duration) *time.Duration {
	return &d
}
```

### 35.7.4 Operator Deployment with Leader Election

```yaml
# config/manager/manager.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-operator-controller-manager
  namespace: postgres-operator-system
spec:
  # Multiple replicas for high availability.
  # Only the leader actively reconciles; others are standby.
  replicas: 2
  selector:
    matchLabels:
      control-plane: controller-manager
  template:
    metadata:
      labels:
        control-plane: controller-manager
    spec:
      containers:
      - name: manager
        image: controller:latest
        args:
        - --leader-elect=true       # Enable leader election
        - --health-probe-bind-address=:8081
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8081
          initialDelaySeconds: 15
          periodSeconds: 20
        readinessProbe:
          httpGet:
            path: /readyz
            port: 8081
          initialDelaySeconds: 5
          periodSeconds: 10
        resources:
          limits:
            cpu: 500m
            memory: 128Mi
          requests:
            cpu: 10m
            memory: 64Mi
      terminationGracePeriodSeconds: 10
```

---

## 35.8 Metrics and Observability

### 35.8.1 Built-in controller-runtime Metrics

controller-runtime exposes Prometheus metrics out of the box:

| Metric | Description |
|--------|-------------|
| `controller_runtime_reconcile_total` | Total reconcile attempts, by controller and result |
| `controller_runtime_reconcile_errors_total` | Total reconcile errors |
| `controller_runtime_reconcile_time_seconds` | Reconcile duration histogram |
| `workqueue_adds_total` | Total items added to the work queue |
| `workqueue_depth` | Current work queue depth |
| `workqueue_queue_duration_seconds` | Time items spend in the queue |

### 35.8.2 Adding Custom Metrics

```go
// Package metrics defines custom Prometheus metrics for the PostgresCluster
// operator. These metrics provide operational visibility into the operator's
// behavior and the health of managed PostgreSQL clusters.
//
// Metrics are registered with the controller-runtime metrics registry,
// which serves them on the operator's metrics endpoint (default: :8080/metrics).
// Prometheus scrapes this endpoint to collect the metrics.
package metrics

import (
	"github.com/prometheus/client_golang/prometheus"
	"sigs.k8s.io/controller-runtime/pkg/metrics"
)

var (
	// ClustersTotal is a gauge that tracks the total number of PostgresCluster
	// resources currently managed by this operator. It is updated during each
	// reconcile cycle.
	//
	// Use this metric to:
	//   - Monitor operator load (how many clusters it manages)
	//   - Alert if the count drops unexpectedly (mass deletion)
	//   - Track growth over time
	ClustersTotal = prometheus.NewGauge(prometheus.GaugeOpts{
		Name:      "postgres_operator_clusters_total",
		Help:      "Total number of PostgresCluster resources managed by the operator",
		Namespace: "postgres_operator",
	})

	// ClusterReadyReplicas is a gauge vector that tracks the number of ready
	// replicas for each PostgresCluster, labeled by cluster name and namespace.
	//
	// Use this metric to:
	//   - Monitor individual cluster health
	//   - Alert when ready replicas drop below desired
	//   - Build Grafana dashboards showing cluster status
	ClusterReadyReplicas = prometheus.NewGaugeVec(
		prometheus.GaugeOpts{
			Name:      "postgres_operator_cluster_ready_replicas",
			Help:      "Number of ready replicas for each PostgresCluster",
			Namespace: "postgres_operator",
		},
		[]string{"name", "namespace"},
	)

	// ClusterDesiredReplicas is a gauge vector that tracks the desired number
	// of replicas for each PostgresCluster, labeled by cluster name and namespace.
	ClusterDesiredReplicas = prometheus.NewGaugeVec(
		prometheus.GaugeOpts{
			Name:      "postgres_operator_cluster_desired_replicas",
			Help:      "Desired number of replicas for each PostgresCluster",
			Namespace: "postgres_operator",
		},
		[]string{"name", "namespace"},
	)

	// ClusterVersion is a gauge vector that reports the PostgreSQL version
	// for each cluster. The version is encoded as a label, and the gauge
	// value is always 1. This allows Prometheus queries like:
	//   postgres_operator_cluster_version{version="16.2"}
	ClusterVersion = prometheus.NewGaugeVec(
		prometheus.GaugeOpts{
			Name:      "postgres_operator_cluster_info",
			Help:      "Information about each PostgresCluster (version label). Value is always 1.",
			Namespace: "postgres_operator",
		},
		[]string{"name", "namespace", "version"},
	)

	// ReconcileOperations is a counter vector that tracks the number of
	// reconcile operations by type (create, update, scale, upgrade, delete).
	// This provides insight into what the operator is doing.
	ReconcileOperations = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Name:      "postgres_operator_reconcile_operations_total",
			Help:      "Total reconcile operations by type",
			Namespace: "postgres_operator",
		},
		[]string{"operation", "name", "namespace"},
	)

	// ReconcileErrors is a counter vector that tracks reconcile errors by
	// type and cluster. Use for alerting on operator failures.
	ReconcileErrors = prometheus.NewCounterVec(
		prometheus.CounterOpts{
			Name:      "postgres_operator_reconcile_errors_total",
			Help:      "Total reconcile errors by type",
			Namespace: "postgres_operator",
		},
		[]string{"error_type", "name", "namespace"},
	)
)

func init() {
	// Register all custom metrics with the controller-runtime metrics registry.
	// This registry is served on the operator's metrics endpoint.
	metrics.Registry.MustRegister(
		ClustersTotal,
		ClusterReadyReplicas,
		ClusterDesiredReplicas,
		ClusterVersion,
		ReconcileOperations,
		ReconcileErrors,
	)
}
```

### 35.8.3 Updating Metrics in the Reconciler

```go
// updateMetrics is called at the end of each reconcile to update
// Prometheus metrics based on the current state of the PostgresCluster.
// Metrics should always reflect the CURRENT state, not deltas.
func (r *PostgresClusterReconciler) updateMetrics(
	pg *postgresv1alpha1.PostgresCluster,
) {
	name := pg.Name
	namespace := pg.Namespace

	// Update replica gauges.
	metrics.ClusterReadyReplicas.WithLabelValues(name, namespace).
		Set(float64(pg.Status.ReadyReplicas))
	metrics.ClusterDesiredReplicas.WithLabelValues(name, namespace).
		Set(float64(pg.Spec.Replicas))

	// Update version info gauge.
	// Reset the old version (if it changed) and set the new one.
	metrics.ClusterVersion.DeletePartialMatch(prometheus.Labels{
		"name": name, "namespace": namespace,
	})
	metrics.ClusterVersion.WithLabelValues(name, namespace, pg.Spec.Version).Set(1)
}

// Example Prometheus alerts for the operator:
//
// groups:
// - name: postgres-operator
//   rules:
//   - alert: PostgresClusterDegraded
//     expr: |
//       postgres_operator_cluster_ready_replicas
//       < postgres_operator_cluster_desired_replicas
//     for: 5m
//     labels:
//       severity: warning
//     annotations:
//       summary: "PostgresCluster {{ $labels.name }} is degraded"
//       description: "Only {{ $value }} of desired replicas are ready"
//
//   - alert: PostgresClusterUnavailable
//     expr: postgres_operator_cluster_ready_replicas == 0
//     for: 1m
//     labels:
//       severity: critical
//     annotations:
//       summary: "PostgresCluster {{ $labels.name }} is unavailable"
//       description: "No ready replicas for cluster {{ $labels.name }}"
//
//   - alert: PostgresOperatorHighReconcileErrors
//     expr: rate(postgres_operator_reconcile_errors_total[5m]) > 0.1
//     for: 10m
//     labels:
//       severity: warning
//     annotations:
//       summary: "Postgres operator has elevated reconcile errors"
```

---

## 35.9 Operator Testing Patterns

### 35.9.1 Table-Driven Tests for Reconcile Scenarios

Table-driven tests are ideal for operators because reconciliation has many scenarios.

```go
// Package controller_test demonstrates table-driven testing patterns for
// Kubernetes operator reconcilers. Table-driven tests enumerate scenarios
// as data, reducing boilerplate and making it easy to add new test cases.
package controller_test

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
	"github.com/example/postgres-operator/internal/controller"
)

// reconcileTestCase defines a single test scenario for the reconciler.
// Each test case specifies initial state (existing objects), the reconcile
// request, and expected outcomes (result, error, resource state).
type reconcileTestCase struct {
	// name is a human-readable description of the scenario.
	name string

	// existingObjects are Kubernetes objects that exist before reconciliation.
	existingObjects []runtime.Object

	// request is the reconcile request (which PostgresCluster to reconcile).
	request ctrl.Request

	// expectError is true if we expect Reconcile to return an error.
	expectError bool

	// expectRequeue is true if we expect the result to have Requeue=true.
	expectRequeue bool

	// expectRequeueAfter is true if we expect RequeueAfter > 0.
	expectRequeueAfter bool

	// validate is an optional function that inspects the state after
	// reconciliation. It receives the fake client and can assert on
	// the state of any Kubernetes resource.
	validate func(t *testing.T, c client.Client)
}

// TestReconcile_TableDriven runs a comprehensive set of reconcile scenarios
// using table-driven tests. Each scenario creates a fresh fake client with
// the specified initial state, runs one reconcile, and verifies the outcomes.
func TestReconcile_TableDriven(t *testing.T) {
	tests := []reconcileTestCase{
		{
			name:            "resource not found — returns empty result (no error)",
			existingObjects: []runtime.Object{},
			request: ctrl.Request{
				NamespacedName: types.NamespacedName{
					Name: "does-not-exist", Namespace: "default",
				},
			},
			expectError:   false,
			expectRequeue: false,
		},
		{
			name: "new cluster — adds finalizer and requeues",
			existingObjects: []runtime.Object{
				newPostgresCluster("test-pg", "default", 3, "16.2"),
			},
			request: ctrl.Request{
				NamespacedName: types.NamespacedName{
					Name: "test-pg", Namespace: "default",
				},
			},
			expectError:   false,
			expectRequeue: true,
			validate: func(t *testing.T, c client.Client) {
				pg := &postgresv1alpha1.PostgresCluster{}
				err := c.Get(context.Background(), types.NamespacedName{
					Name: "test-pg", Namespace: "default",
				}, pg)
				require.NoError(t, err)
				assert.Contains(t, pg.Finalizers, "postgres.example.com/finalizer",
					"finalizer should be added on first reconcile")
			},
		},
		{
			name: "cluster with finalizer — creates StatefulSet",
			existingObjects: []runtime.Object{
				newPostgresClusterWithFinalizer("test-pg", "default", 3, "16.2"),
			},
			request: ctrl.Request{
				NamespacedName: types.NamespacedName{
					Name: "test-pg", Namespace: "default",
				},
			},
			expectError:        false,
			expectRequeueAfter: true, // cluster is creating (0 ready replicas)
			validate: func(t *testing.T, c client.Client) {
				// Verify StatefulSet exists.
				sts := &appsv1.StatefulSet{}
				err := c.Get(context.Background(), types.NamespacedName{
					Name: "test-pg", Namespace: "default",
				}, sts)
				require.NoError(t, err)
				assert.Equal(t, int32(3), *sts.Spec.Replicas)

				// Verify headless Service exists.
				svc := &corev1.Service{}
				err = c.Get(context.Background(), types.NamespacedName{
					Name: "test-pg-headless", Namespace: "default",
				}, svc)
				require.NoError(t, err)
				assert.Equal(t, corev1.ClusterIPNone, svc.Spec.ClusterIP)

				// Verify ConfigMap exists.
				cm := &corev1.ConfigMap{}
				err = c.Get(context.Background(), types.NamespacedName{
					Name: "test-pg-config", Namespace: "default",
				}, cm)
				require.NoError(t, err)
			},
		},
	}

	for _, tc := range tests {
		t.Run(tc.name, func(t *testing.T) {
			// Create a fresh fake client for each test case.
			scheme := runtime.NewScheme()
			_ = postgresv1alpha1.AddToScheme(scheme)
			_ = appsv1.AddToScheme(scheme)
			_ = corev1.AddToScheme(scheme)

			fakeClient := fake.NewClientBuilder().
				WithScheme(scheme).
				WithRuntimeObjects(tc.existingObjects...).
				WithStatusSubresource(&postgresv1alpha1.PostgresCluster{}).
				Build()

			reconciler := &controller.PostgresClusterReconciler{
				Client:   fakeClient,
				Scheme:   scheme,
				Recorder: record.NewFakeRecorder(100),
			}

			// Run one reconciliation.
			result, err := reconciler.Reconcile(context.Background(), tc.request)

			// Assert error expectation.
			if tc.expectError {
				assert.Error(t, err, "expected reconcile to return an error")
			} else {
				assert.NoError(t, err, "expected reconcile to succeed")
			}

			// Assert requeue expectation.
			if tc.expectRequeue {
				assert.True(t, result.Requeue,
					"expected result.Requeue to be true")
			}
			if tc.expectRequeueAfter {
				assert.True(t, result.RequeueAfter > 0,
					"expected result.RequeueAfter > 0")
			}

			// Run custom validation.
			if tc.validate != nil {
				tc.validate(t, fakeClient)
			}
		})
	}
}

// newPostgresCluster creates a PostgresCluster resource for testing.
func newPostgresCluster(name, namespace string, replicas int32, version string) *postgresv1alpha1.PostgresCluster {
	return &postgresv1alpha1.PostgresCluster{
		ObjectMeta: metav1.ObjectMeta{
			Name:       name,
			Namespace:  namespace,
			Generation: 1,
		},
		Spec: postgresv1alpha1.PostgresClusterSpec{
			Version:  version,
			Replicas: replicas,
			Storage: postgresv1alpha1.StorageSpec{
				Size: resource.MustParse("10Gi"),
			},
		},
	}
}

// newPostgresClusterWithFinalizer creates a PostgresCluster that already
// has the finalizer added (simulating a cluster that has already had its
// first reconcile).
func newPostgresClusterWithFinalizer(name, namespace string, replicas int32, version string) *postgresv1alpha1.PostgresCluster {
	pg := newPostgresCluster(name, namespace, replicas, version)
	pg.Finalizers = []string{"postgres.example.com/finalizer"}
	return pg
}
```

### 35.9.2 Testing with envtest

`envtest` provides a real API server for integration testing:

```go
// envtestSetup shows the standard pattern for setting up envtest in an
// operator test suite. envtest starts a real API server and etcd, allowing
// tests to verify the full reconcile loop with real API calls and webhooks.
//
// Key differences from fake client tests:
//   - Real API validation (schema validation, admission webhooks)
//   - Real status subresource behavior
//   - Real optimistic concurrency (resourceVersion conflicts)
//   - Real list/watch behavior
//
// Key differences from E2E tests:
//   - No kubelet, so Pods are never actually scheduled
//   - No default controllers (no StatefulSet controller, no Deployment controller)
//   - Much faster to start (seconds vs minutes)
func envtestSetup() {
	// See the integration test example in Chapter 34, section 34.7.2.
	// The key setup steps are:
	//   1. Create an envtest.Environment with CRD paths
	//   2. Start it to get a *rest.Config
	//   3. Create a Manager with that config
	//   4. Register your reconciler with the Manager
	//   5. Start the Manager in a goroutine
	//   6. Use a client.Client to create/get/update resources
	//   7. Use Eventually() to wait for async reconcile effects
	//   8. Stop the Manager and Environment in cleanup
}
```

### 35.9.3 E2E Testing with kind

```bash
#!/bin/bash
# e2e-test.sh — Complete E2E test suite for the postgres-operator.
# Uses kind (Kubernetes in Docker) to create a real cluster.

set -euo pipefail

CLUSTER_NAME="pg-operator-e2e"
IMG="postgres-operator:e2e"
NAMESPACE="postgres-operator-system"

cleanup() {
    echo "=== Cleaning up ==="
    kind delete cluster --name "$CLUSTER_NAME" 2>/dev/null || true
}
trap cleanup EXIT

echo "=== Phase 1: Setup ==="
kind create cluster --name "$CLUSTER_NAME" --wait 120s
make docker-build IMG="$IMG"
kind load docker-image "$IMG" --name "$CLUSTER_NAME"
make install
make deploy IMG="$IMG"
kubectl -n "$NAMESPACE" wait --for=condition=Available \
    deployment/postgres-operator-controller-manager --timeout=120s

echo "=== Phase 2: Create Cluster ==="
cat <<EOF | kubectl apply -f -
apiVersion: postgres.example.com/v1alpha1
kind: PostgresCluster
metadata:
  name: e2e-test-pg
spec:
  version: "16.2"
  replicas: 1
  storage:
    size: 1Gi
EOF

echo "Waiting for cluster to be ready..."
kubectl wait --for=jsonpath='{.status.phase}'=Running \
    postgrescluster/e2e-test-pg --timeout=180s
echo "✓ Cluster is running"

echo "=== Phase 3: Verify Resources ==="
kubectl get statefulset e2e-test-pg -o name || { echo "✗ StatefulSet not found"; exit 1; }
kubectl get svc e2e-test-pg -o name || { echo "✗ Service not found"; exit 1; }
kubectl get svc e2e-test-pg-headless -o name || { echo "✗ Headless Service not found"; exit 1; }
kubectl get configmap e2e-test-pg-config -o name || { echo "✗ ConfigMap not found"; exit 1; }
echo "✓ All resources created"

echo "=== Phase 4: Test Scaling ==="
kubectl patch postgrescluster e2e-test-pg -p '{"spec":{"replicas":3}}' --type=merge
sleep 5
REPLICAS=$(kubectl get statefulset e2e-test-pg -o jsonpath='{.spec.replicas}')
[ "$REPLICAS" -eq 3 ] || { echo "✗ Expected 3 replicas, got $REPLICAS"; exit 1; }
echo "✓ Scaling works"

echo "=== Phase 5: Test Deletion ==="
kubectl delete postgrescluster e2e-test-pg --timeout=60s
sleep 5
kubectl get statefulset e2e-test-pg 2>&1 | grep -q "not found" || {
    echo "✗ StatefulSet should be deleted"; exit 1;
}
echo "✓ Garbage collection works"

echo "=== All E2E tests passed ✓ ==="
```

---

## 35.10 Operator Lifecycle Manager (OLM)

### 35.10.1 What OLM Does

The **Operator Lifecycle Manager** is itself a Kubernetes operator that manages other operators.
It provides:

- **Discovery** — users browse a catalog of available operators
- **Installation** — one-click install with proper RBAC and dependencies
- **Upgrades** — automated operator upgrades with rollback support
- **Dependency resolution** — if your operator depends on cert-manager, OLM installs it first
- **Multi-tenancy** — operators can be scoped to specific namespaces

### 35.10.2 Key OLM Concepts

| Concept | Description |
|---------|-------------|
| **ClusterServiceVersion (CSV)** | The metadata file that describes your operator: name, version, description, RBAC, owned CRDs, icon, maintainers |
| **Bundle** | A directory containing the CSV, CRD manifests, and metadata for ONE version of your operator |
| **Catalog** | An index of bundles — think "app store" for operators |
| **Subscription** | A declaration that a user wants to install a specific operator from a catalog |
| **InstallPlan** | The execution plan for installing/upgrading an operator (lists all resources to create) |
| **CatalogSource** | Points to a catalog (can be an OCI image or a gRPC endpoint) |

### 35.10.3 Creating an OLM Bundle

```bash
# Step 1: Generate the bundle
# This creates a bundle/ directory with the CSV, CRDs, and metadata.
make bundle \
  IMG=myregistry.com/postgres-operator:v0.1.0 \
  BUNDLE_IMG=myregistry.com/postgres-operator-bundle:v0.1.0 \
  VERSION=0.1.0

# The generated structure:
# bundle/
# ├── manifests/
# │   ├── postgres-operator.clusterserviceversion.yaml  # CSV
# │   ├── postgres.example.com_postgresclusters.yaml    # CRD
# │   ├── postgres-operator-controller-manager_rbac.authorization.k8s.io_v1_clusterrole.yaml
# │   └── ... (other RBAC manifests)
# ├── metadata/
# │   └── annotations.yaml
# └── tests/
#     └── scorecard/
#         └── config.yaml

# Step 2: Build the bundle image
make bundle-build BUNDLE_IMG=myregistry.com/postgres-operator-bundle:v0.1.0

# Step 3: Push the bundle image
make bundle-push BUNDLE_IMG=myregistry.com/postgres-operator-bundle:v0.1.0

# Step 4: Test with operator-sdk run bundle
operator-sdk run bundle myregistry.com/postgres-operator-bundle:v0.1.0

# Step 5: Validate the bundle
operator-sdk bundle validate ./bundle
```

### 35.10.4 CSV Anatomy

The ClusterServiceVersion is the heart of an OLM bundle:

```yaml
# bundle/manifests/postgres-operator.clusterserviceversion.yaml
apiVersion: operators.coreos.com/v1alpha1
kind: ClusterServiceVersion
metadata:
  name: postgres-operator.v0.1.0
  namespace: operators
  annotations:
    alm-examples: |
      [
        {
          "apiVersion": "postgres.example.com/v1alpha1",
          "kind": "PostgresCluster",
          "metadata": {"name": "example"},
          "spec": {
            "version": "16.2",
            "replicas": 3,
            "storage": {"size": "50Gi"}
          }
        }
      ]
    capabilities: Seamless Upgrades
    categories: Database
spec:
  displayName: PostgreSQL Operator
  description: |
    Manages PostgreSQL database clusters on Kubernetes. Handles provisioning,
    scaling, upgrades, backups, and monitoring.
  version: 0.1.0
  maturity: alpha
  replaces: ""  # Previous version this upgrades from

  # Keywords for catalog search
  keywords:
  - postgresql
  - database
  - operator

  # Maintainer information
  maintainers:
  - name: Database Team
    email: db-team@example.com

  # Links
  links:
  - name: Documentation
    url: https://docs.example.com/postgres-operator
  - name: Source Code
    url: https://github.com/example/postgres-operator

  # Installation mode: OwnNamespace, SingleNamespace, MultiNamespace, AllNamespaces
  installModes:
  - type: OwnNamespace
    supported: true
  - type: SingleNamespace
    supported: true
  - type: MultiNamespace
    supported: false
  - type: AllNamespaces
    supported: true

  # CRDs this operator owns
  customresourcedefinitions:
    owned:
    - name: postgresclusters.postgres.example.com
      version: v1alpha1
      kind: PostgresCluster
      displayName: PostgreSQL Cluster
      description: A PostgreSQL database cluster

  # The operator deployment
  install:
    strategy: deployment
    spec:
      deployments:
      - name: postgres-operator-controller-manager
        spec:
          replicas: 1
          selector:
            matchLabels:
              control-plane: controller-manager
          template:
            spec:
              containers:
              - name: manager
                image: myregistry.com/postgres-operator:v0.1.0
                args:
                - --leader-elect

  # RBAC permissions
  clusterPermissions:
  - serviceAccountName: postgres-operator-controller-manager
    rules:
    - apiGroups: [postgres.example.com]
      resources: [postgresclusters]
      verbs: [get, list, watch, create, update, patch, delete]
    - apiGroups: [postgres.example.com]
      resources: [postgresclusters/status]
      verbs: [get, update, patch]
    # ... (other RBAC rules)
```

---

## 35.11 Real-World Operator Examples — Analysis

### 35.11.1 cert-manager

**What it manages:** TLS certificates from various issuers (Let's Encrypt, Vault, self-signed).

**Key design decisions:**

- **Multiple custom resources:** `Certificate`, `Issuer`, `ClusterIssuer`, `CertificateRequest`,
  `Order`, `Challenge`. Each models a different concept in the certificate lifecycle.
- **Separation of concerns:** The user creates a `Certificate`; cert-manager creates a
  `CertificateRequest`, which creates an `Order`, which creates `Challenge`s. Each step is a
  separate resource reconciled by a separate controller.
- **Cross-namespace design:** `ClusterIssuer` is cluster-scoped; `Issuer` is namespace-scoped.
  This enables both shared issuers (used by all namespaces) and private issuers.
- **Status conditions:** Every resource has detailed conditions (Ready, Issuing, etc.).
- **Event recording:** Rich events on every resource at every lifecycle stage.

**What makes it well-designed:**

1. Clean API separation between intent (Certificate) and implementation (Order, Challenge).
2. Idempotent reconciliation — requesting the same certificate twice is a no-op.
3. Automatic renewal with configurable lead time.
4. Extensive webhook validation prevents invalid configurations.

### 35.11.2 Prometheus Operator

**What it manages:** Prometheus, Alertmanager, and related monitoring components.

**Key design decisions:**

- **ServiceMonitor/PodMonitor resources:** Instead of editing Prometheus configuration directly,
  teams create ServiceMonitor resources in their own namespaces. The operator discovers them and
  generates the Prometheus scrape configuration.
- **PrometheusRule resources:** Teams define alerting rules as custom resources; the operator
  aggregates them into Prometheus rule files.
- **StatefulSet-based:** Prometheus and Alertmanager run as StatefulSets for data persistence.
- **Namespace-scoped discovery:** The Prometheus resource specifies which namespaces to search
  for ServiceMonitors, enabling multi-tenant monitoring.

**What makes it well-designed:**

1. Decoupled configuration: application teams manage their own monitoring without cluster-admin access.
2. Automatic configuration generation from custom resources.
3. Safe rolling updates of Prometheus with WAL replay.

### 35.11.3 ArgoCD

**What it manages:** GitOps-based application deployment.

**Key design decisions:**

- **Application resource:** Each Application defines a source (Git repo) and destination
  (Kubernetes cluster/namespace). ArgoCD continuously syncs the two.
- **ApplicationSet:** A template for generating Applications from dynamic sources (Git generators,
  cluster generators, matrix generators).
- **Health assessment:** ArgoCD understands the health of 50+ Kubernetes resource types and
  reports aggregate application health.
- **Sync waves:** Resources are deployed in order (CRDs first, then namespaces, then workloads).

**What makes it well-designed:**

1. Declarative GitOps: the Git repo is the single source of truth.
2. Rich health checking beyond Kubernetes readiness probes.
3. Rollback to any previous Git commit.
4. Multi-cluster management from a single ArgoCD instance.

### 35.11.4 Common Patterns Across Great Operators

| Pattern | cert-manager | Prometheus Operator | ArgoCD |
|---------|-------------|-------------------|--------|
| Multiple CRDs | ✅ 6+ CRDs | ✅ 5+ CRDs | ✅ 3+ CRDs |
| Status conditions | ✅ | ✅ | ✅ |
| Finalizers | ✅ | ✅ | ✅ |
| Webhooks | ✅ | ✅ | ✅ |
| Leader election | ✅ | ✅ | ✅ |
| Prometheus metrics | ✅ | ✅ (obviously) | ✅ |
| Event recording | ✅ | ✅ | ✅ |
| Multi-namespace | ✅ | ✅ | ✅ |
| OLM support | ✅ | ✅ | ✅ |

---

## 35.12 Summary

This chapter covered the advanced patterns that make operators production-ready:

| Pattern | What It Provides |
|---------|-----------------|
| **Finalizers** | Guaranteed cleanup of external resources before deletion |
| **Validating Webhooks** | Reject invalid resources before they're stored |
| **Mutating Webhooks** | Inject defaults and computed fields |
| **Status Conditions** | Rich, structured health reporting |
| **Owner References** | Automatic garbage collection and watch propagation |
| **Event Recording** | Audit trail visible via kubectl describe |
| **Multi-Resource Watching** | React to changes in owned and external resources |
| **Leader Election** | HA operator deployment with only one active reconciler |
| **Custom Metrics** | Prometheus metrics for operator and cluster observability |
| **OLM** | Standard packaging for operator distribution and lifecycle |

These patterns are not optional for production operators. Every operator listed on
OperatorHub.io implements most or all of them. By mastering these patterns, you can build
operators that are reliable, observable, and maintainable.

---

## Exercises

1. **Build a Finalizer for S3 Backups**: Extend the PostgresCluster operator's finalizer to
   delete S3 backup buckets when the cluster is deleted. Handle the case where the S3 bucket
   doesn't exist (idempotency) and the case where AWS credentials are missing (graceful error).

2. **Implement a Conversion Webhook**: Create a v1alpha2 version of the PostgresCluster API that
   renames `spec.version` to `spec.postgresVersion`. Implement a conversion webhook that
   translates between v1alpha1 and v1alpha2.

3. **Add a Degraded Condition**: Implement a "Degraded" condition that is True when the cluster
   has fewer ready replicas than desired but at least one is available. Test it with the table-driven
   test pattern.

4. **Build a Custom Metric for Connection Count**: Add a metric that reports the number of active
   connections to each PostgreSQL instance. (Hint: exec into the Pod and run
   `SELECT count(*) FROM pg_stat_activity`.)

5. **Package for OLM**: Create a complete OLM bundle for the PostgresCluster operator. Include
   the CSV, CRD, RBAC, and a sample CR. Test installation with `operator-sdk run bundle`.

6. **Implement a Backup Operator**: Build a separate operator that watches PostgresCluster
   resources and creates CronJobs for automated backups. Use cross-resource watching to trigger
   reconciliation when a PostgresCluster changes. This exercises multi-operator coordination.
