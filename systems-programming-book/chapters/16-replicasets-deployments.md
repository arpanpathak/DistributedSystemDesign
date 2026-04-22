# Chapter 16: ReplicaSets & Deployments

> *"A single Pod is a liability. A ReplicaSet is a guarantee. A Deployment is a
> strategy."*

In Chapter 15, we dissected the Pod — the atomic unit of deployment in Kubernetes.
But Pods alone are fragile. If a Pod crashes, nobody recreates it. If a node dies,
its Pods die with it. If you need ten instances of your application, you don't want
to manage ten Pod manifests by hand.

This chapter introduces the two controllers that solve these problems:

- **ReplicaSet** — ensures a specified number of identical Pods are always running.
- **Deployment** — manages ReplicaSets to provide declarative updates, rollbacks,
  and deployment strategies.

Together, they form the backbone of stateless workload management in Kubernetes.
By the end of this chapter, you will understand not just how to use these objects,
but how their controllers work internally — the watch-list-reconcile loop that
is the heartbeat of Kubernetes.

---

## 16.1 ReplicaSet

### 16.1.1 Purpose

A ReplicaSet's single responsibility is to **maintain a stable set of replica Pods
running at any given time**. If a Pod fails, the ReplicaSet creates a replacement.
If there are too many Pods (e.g., because you scaled down), it deletes the excess.

You rarely create ReplicaSets directly. Deployments create and manage them for you.
But understanding ReplicaSets is essential because they are the mechanism through
which Deployments work.

### 16.1.2 Label Selectors — How a ReplicaSet Finds Its Pods

A ReplicaSet doesn't track Pods by name. It uses **label selectors** to identify
which Pods it owns. Any Pod whose labels match the selector is counted as part of
the ReplicaSet's replica count.

```yaml
selector:
  matchLabels:
    app: web
    tier: frontend
```

This matches any Pod with **both** labels `app=web` AND `tier=frontend`.

You can also use more expressive `matchExpressions`:

```yaml
selector:
  matchLabels:
    app: web
  matchExpressions:
  - key: environment
    operator: In
    values: [production, staging]
  - key: version
    operator: NotIn
    values: [deprecated]
```

**Critical rule:** The `.spec.selector` must match the labels in
`.spec.template.metadata.labels`. If they don't match, the API server rejects
the ReplicaSet.

### 16.1.3 Owner References — How a ReplicaSet Claims Its Pods

When a ReplicaSet creates a Pod, it sets an **ownerReference** on the Pod's metadata:

```yaml
metadata:
  ownerReferences:
  - apiVersion: apps/v1
    kind: ReplicaSet
    name: web-frontend-6d4f5b7c8
    uid: 3e5c7a2f-...
    controller: true
    blockOwnerDeletion: true
```

This establishes a parent-child relationship:
- The ReplicaSet is the **owner** (controller).
- The Pod is the **dependent**.
- If the ReplicaSet is deleted with `--cascade=foreground`, its Pods are deleted first.
- If the ReplicaSet is deleted with `--cascade=orphan`, Pods continue running without
  an owner.

Owner references also prevent two ReplicaSets from fighting over the same Pod. A Pod
can have at most one controller owner.

### 16.1.4 ReplicaSet Spec Deep Dive

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-frontend
  namespace: production
  labels:
    app: web
    tier: frontend
spec:
  # replicas specifies the desired number of Pod instances.
  # The ReplicaSet controller continuously reconciles the actual
  # count to match this value.
  replicas: 3

  # selector defines which Pods the ReplicaSet manages. It MUST
  # match the labels in the Pod template below.
  selector:
    matchLabels:
      app: web
      tier: frontend

  # template is the Pod template used to create new Pods when the
  # actual count is below the desired count. This is a full PodSpec
  # embedded in the ReplicaSet.
  template:
    metadata:
      labels:
        app: web
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
        readinessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 10
```

### 16.1.5 How the ReplicaSet Controller Works Internally

The ReplicaSet controller is a control loop running in the `kube-controller-manager`.
It follows the standard Kubernetes controller pattern: **watch → compare → act**.

```
┌─────────────────────────────────────────────────────────────┐
│                  ReplicaSet Controller                       │
│                                                              │
│  1. WATCH                                                    │
│     ├── Watch all ReplicaSet objects for changes             │
│     └── Watch all Pod objects for changes                    │
│                                                              │
│  2. RECONCILE (triggered by any watch event)                 │
│     │                                                        │
│     ├── For the affected ReplicaSet:                         │
│     │   ├── List all Pods matching the selector              │
│     │   ├── Filter to only Pods with matching ownerRef       │
│     │   │   (or adopt orphan Pods that match)                │
│     │   └── Count active Pods (not Terminating)              │
│     │                                                        │
│     ├── COMPARE actual count vs desired count                │
│     │                                                        │
│     ├── IF actual < desired:                                 │
│     │   └── Create (desired - actual) new Pods               │
│     │       using the Pod template                           │
│     │                                                        │
│     ├── IF actual > desired:                                 │
│     │   └── Delete (actual - desired) Pods                   │
│     │       (prefer deleting unscheduled, then newest)       │
│     │                                                        │
│     └── IF actual == desired:                                │
│         └── No action needed                                 │
│                                                              │
│  3. UPDATE STATUS                                            │
│     └── Update .status.replicas, .status.readyReplicas,     │
│         .status.availableReplicas                            │
└─────────────────────────────────────────────────────────────┘
```

**Key implementation details:**

1. **Batch creation:** When creating multiple Pods, the controller uses a
   burst-and-check pattern. It creates Pods in exponentially increasing batches
   (1, 2, 4, 8, ...) up to 500 at a time, checking for errors between batches.
   This prevents a misconfigured ReplicaSet from flooding the API server.

2. **Pod adoption:** If the controller finds a Pod whose labels match the selector
   but has no ownerReference, it **adopts** the Pod by setting itself as the owner.
   This is how orphaned Pods are reclaimed.

3. **Pod orphaning:** If a Pod's labels are changed so they no longer match the
   selector, the controller releases ownership. The Pod continues running but is
   no longer managed.

4. **Deletion priority:** When scaling down, the controller preferentially deletes:
   - Pods on nodes that are being drained.
   - Pods that are Pending (unscheduled).
   - Pods whose creation timestamp is newer (LIFO).
   - Pods that have been restarted more.

### 16.1.6 Hands-on: Creating and Managing a ReplicaSet

```bash
# Create a ReplicaSet with 3 replicas
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
  labels:
    app: web
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
EOF

# Check the ReplicaSet status
kubectl get rs web-rs
# NAME     DESIRED   CURRENT   READY   AGE
# web-rs   3         3         3       10s

# List Pods with owner info
kubectl get pods -l app=web -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.phase,\
OWNER:.metadata.ownerReferences[0].name
# NAME            STATUS    OWNER
# web-rs-abc12    Running   web-rs
# web-rs-def34    Running   web-rs
# web-rs-ghi56    Running   web-rs

# Delete a Pod and watch it get recreated
kubectl delete pod web-rs-abc12
kubectl get pods -l app=web --watch
# web-rs-abc12   1/1     Terminating   0     5m
# web-rs-xyz78   0/1     Pending       0     0s
# web-rs-xyz78   0/1     ContainerCreating   0     0s
# web-rs-xyz78   1/1     Running       0     2s

# Scale the ReplicaSet
kubectl scale rs web-rs --replicas=5
kubectl get rs web-rs
# NAME     DESIRED   CURRENT   READY   AGE
# web-rs   5         5         5       2m

# Scale down
kubectl scale rs web-rs --replicas=2
kubectl get pods -l app=web
# Only 2 Pods remain — the newest 3 were deleted

# Delete the ReplicaSet (Pods are deleted too by default)
kubectl delete rs web-rs
# Or keep Pods running (orphan them):
# kubectl delete rs web-rs --cascade=orphan
```

### 16.1.7 Hands-on: Go client-go Code to Manage ReplicaSets

```go
// file: replicaset_manager.go
//
// Package main demonstrates programmatic management of Kubernetes
// ReplicaSets using the client-go library. This program creates a
// ReplicaSet, scales it, lists its Pods, and cleans up — all through
// the Kubernetes API.
//
// This is instructive for understanding how controllers interact with
// the API server, and forms the foundation for the Deployment manager
// in §16.8.
//
// Prerequisites:
//   - A running Kubernetes cluster
//   - A valid kubeconfig file
//   - go get k8s.io/client-go@latest k8s.io/apimachinery@latest
package main

import (
	"context"
	"flag"
	"fmt"
	"os"
	"time"

	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
)

// int32Ptr is a helper to get a pointer to an int32 value.
// Kubernetes API objects use pointers for optional numeric fields
// to distinguish "not set" (nil) from "set to zero."
func int32Ptr(i int32) *int32 { return &i }

func main() {
	// kubeconfig is the path to the Kubernetes config file.
	kubeconfig := flag.String("kubeconfig", os.Getenv("KUBECONFIG"), "Path to kubeconfig")

	// namespace is the Kubernetes namespace to operate in.
	namespace := flag.String("namespace", "default", "Kubernetes namespace")

	flag.Parse()

	// Build the REST config from the kubeconfig file.
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error building config: %v\n", err)
		os.Exit(1)
	}

	// Create the Kubernetes clientset.
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error creating clientset: %v\n", err)
		os.Exit(1)
	}

	ctx := context.Background()

	// ─── Step 1: Create a ReplicaSet ────────────────────────────────
	fmt.Println("=== Creating ReplicaSet ===")

	// Define the ReplicaSet object. This is equivalent to writing
	// a YAML manifest but expressed as a Go struct.
	rs := &appsv1.ReplicaSet{
		ObjectMeta: metav1.ObjectMeta{
			Name:      "demo-rs",
			Namespace: *namespace,
			Labels: map[string]string{
				"app":        "demo",
				"managed-by": "go-client",
			},
		},
		Spec: appsv1.ReplicaSetSpec{
			// Start with 3 replicas.
			Replicas: int32Ptr(3),

			// The selector must match the Pod template labels.
			Selector: &metav1.LabelSelector{
				MatchLabels: map[string]string{
					"app": "demo",
				},
			},

			// Pod template — defines what each replica looks like.
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: map[string]string{
						"app":        "demo",
						"managed-by": "go-client",
					},
				},
				Spec: corev1.PodSpec{
					Containers: []corev1.Container{
						{
							Name:  "nginx",
							Image: "nginx:1.25",
							Ports: []corev1.ContainerPort{
								{ContainerPort: 80},
							},
						},
					},
				},
			},
		},
	}

	// Create the ReplicaSet via the API server.
	createdRS, err := clientset.AppsV1().ReplicaSets(*namespace).Create(ctx, rs, metav1.CreateOptions{})
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error creating ReplicaSet: %v\n", err)
		os.Exit(1)
	}
	fmt.Printf("Created ReplicaSet: %s (UID: %s)\n", createdRS.Name, createdRS.UID)

	// ─── Step 2: Wait for Pods to be Ready ──────────────────────────
	fmt.Println("\n=== Waiting for Pods to be Ready ===")
	for i := 0; i < 30; i++ {
		time.Sleep(2 * time.Second)

		// Fetch the current state of the ReplicaSet.
		currentRS, err := clientset.AppsV1().ReplicaSets(*namespace).Get(ctx, "demo-rs", metav1.GetOptions{})
		if err != nil {
			fmt.Fprintf(os.Stderr, "Error getting ReplicaSet: %v\n", err)
			continue
		}

		fmt.Printf("  Desired: %d, Current: %d, Ready: %d\n",
			*currentRS.Spec.Replicas,
			currentRS.Status.Replicas,
			currentRS.Status.ReadyReplicas)

		if currentRS.Status.ReadyReplicas == *currentRS.Spec.Replicas {
			fmt.Println("  All replicas ready!")
			break
		}
	}

	// ─── Step 3: List Pods Owned by the ReplicaSet ──────────────────
	fmt.Println("\n=== Pods Owned by ReplicaSet ===")

	// Use the label selector to find matching Pods.
	labelSelector := labels.Set{"app": "demo"}.AsSelector().String()
	pods, err := clientset.CoreV1().Pods(*namespace).List(ctx, metav1.ListOptions{
			LabelSelector: labelSelector,
		})
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error listing pods: %v\n", err)
		os.Exit(1)
	}

	for _, pod := range pods.Items {
		// Verify this Pod is actually owned by our ReplicaSet
		// by checking the ownerReference.
		owner := "none"
		for _, ref := range pod.OwnerReferences {
			if ref.Kind == "ReplicaSet" && ref.Name == "demo-rs" {
				owner = ref.Name
				break
			}
		}
		fmt.Printf("  Pod: %-40s Phase: %-10s Owner: %s\n",
			pod.Name, pod.Status.Phase, owner)
	}

	// ─── Step 4: Scale the ReplicaSet to 5 ──────────────────────────
	fmt.Println("\n=== Scaling ReplicaSet to 5 Replicas ===")

	// Fetch the current ReplicaSet, update replicas, and apply.
	currentRS, err := clientset.AppsV1().ReplicaSets(*namespace).Get(ctx, "demo-rs", metav1.GetOptions{})
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error getting ReplicaSet: %v\n", err)
		os.Exit(1)
	}

	currentRS.Spec.Replicas = int32Ptr(5)
	_, err = clientset.AppsV1().ReplicaSets(*namespace).Update(ctx, currentRS, metav1.UpdateOptions{})
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error scaling ReplicaSet: %v\n", err)
		os.Exit(1)
	}
	fmt.Println("Scaled to 5 replicas. Waiting for convergence...")

	// Wait for scale-up.
	for i := 0; i < 30; i++ {
		time.Sleep(2 * time.Second)
		rs, err := clientset.AppsV1().ReplicaSets(*namespace).Get(ctx, "demo-rs", metav1.GetOptions{})
		if err != nil {
			continue
		}
		fmt.Printf("  Ready: %d / %d\n", rs.Status.ReadyReplicas, *rs.Spec.Replicas)
		if rs.Status.ReadyReplicas == 5 {
			fmt.Println("  Scale-up complete!")
			break
		}
	}

	// ─── Step 5: Clean Up ───────────────────────────────────────────
	fmt.Println("\n=== Cleaning Up ===")

	// Delete the ReplicaSet and its Pods (foreground cascading delete).
	propagation := metav1.DeletePropagationForeground
	err = clientset.AppsV1().ReplicaSets(*namespace).Delete(ctx, "demo-rs", metav1.DeleteOptions{
			PropagationPolicy: &propagation,
		})
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error deleting ReplicaSet: %v\n", err)
		os.Exit(1)
	}
	fmt.Println("ReplicaSet deleted (cascading to Pods)")
}
```

---

## 16.2 Deployment

### 16.2.1 Purpose

A Deployment provides **declarative updates** for Pods and ReplicaSets. You describe
a desired state in a Deployment, and the Deployment controller changes the actual
state to match — creating new ReplicaSets, scaling them up, and scaling old ones
down in a controlled manner.

### 16.2.2 The Ownership Chain

```
┌─────────────┐     owns      ┌──────────────┐     owns      ┌───────┐
│  Deployment  │─────────────▶│  ReplicaSet   │─────────────▶│  Pods  │
│              │              │  (revision 3) │              │  (×3)  │
│  web-app     │              │  web-app-7f.. │              │        │
└─────────────┘              └──────────────┘              └───────┘
       │                           ▲
       │          also owns        │
       │         ┌──────────────┐  │
       └────────▶│  ReplicaSet   │──┘
                 │  (revision 2) │
                 │  web-app-5c.. │  scaled to 0
                 └──────────────┘
```

The Deployment owns multiple ReplicaSets (one per revision). Only the latest
ReplicaSet has a non-zero replica count. Older ReplicaSets are kept (with 0 replicas)
for rollback purposes.

### 16.2.3 Deployment Spec Deep Dive

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: production
  labels:
    app: web-app
  annotations:
    # Record the kubectl command that created/modified this Deployment.
    kubernetes.io/change-cause: "Upgraded to v2.1.0"
spec:
  # replicas is the desired number of Pod instances.
  replicas: 5

  # selector identifies which Pods belong to this Deployment.
  # Must match .spec.template.metadata.labels.
  selector:
    matchLabels:
      app: web-app

  # template is the Pod specification for each replica.
  template:
    metadata:
      labels:
        app: web-app
        version: v2.1.0
    spec:
      containers:
      - name: app
        image: myregistry/web-app:v2.1.0
        ports:
        - containerPort: 8080
        resources:
          requests:
            cpu: 200m
            memory: 256Mi
          limits:
            cpu: "1"
            memory: 512Mi
        readinessProbe:
          httpGet:
            path: /healthz/ready
            port: 8080
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /healthz/live
            port: 8080
          periodSeconds: 10

  # strategy controls how old Pods are replaced by new ones.
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1            # 1 extra Pod during update
      maxUnavailable: 0      # Never have fewer than desired count

  # revisionHistoryLimit is how many old ReplicaSets to keep for
  # rollback. Default is 10.
  revisionHistoryLimit: 10

  # progressDeadlineSeconds is how long to wait for a rollout to
  # make progress before marking it as failed. Default is 600.
  progressDeadlineSeconds: 600

  # minReadySeconds is how long a new Pod must be Ready without
  # any container crashes before it is considered Available.
  # This prevents fast-failing Pods from being counted as successful.
  minReadySeconds: 10

  # paused freezes the Deployment — changes to the template don't
  # trigger a rollout until unpaused.
  paused: false
```

---

## 16.3 Deployment Strategies

### 16.3.1 RollingUpdate (Default)

The RollingUpdate strategy incrementally replaces old Pods with new ones. At no
point are all Pods down simultaneously. Two parameters control the pace:

- **maxSurge** — how many extra Pods can exist above the desired count during the
  update. Can be an absolute number or a percentage. Default: 25%.
- **maxUnavailable** — how many Pods can be unavailable during the update. Can be
  an absolute number or a percentage. Default: 25%.

**Example configurations:**

| maxSurge | maxUnavailable | Behavior |
|----------|----------------|----------|
| 1 | 0 | Zero-downtime: always at least `replicas` Pods running. One extra created, one old removed. Slow but safe. |
| 0 | 1 | No extra capacity: remove one old Pod, wait for new one. Slightly reduced capacity during update. |
| 25% | 25% | Default: with 4 replicas, surge 1, unavailable 1. Balanced speed and safety. |
| 100% | 0 | Blue-green style: create all new Pods first, then remove all old Pods. Requires 2× cluster capacity. |

**Step-by-step: How a rolling update works**

Let's trace a rolling update of a Deployment from v1 to v2 with `replicas: 4`,
`maxSurge: 1`, `maxUnavailable: 1`:

```
Initial state:
  RS-v1: 4 Pods [v1] [v1] [v1] [v1]    (desired=4, current=4)
  RS-v2: does not exist

Step 1: User updates image to v2. Deployment controller creates RS-v2.
  RS-v1: 4 Pods [v1] [v1] [v1] [v1]    (desired=4, current=4)
  RS-v2: 0 Pods                          (desired=0, current=0)

Step 2: Controller calculates:
  - Total allowed Pods = replicas + maxSurge = 4 + 1 = 5
  - Min available = replicas - maxUnavailable = 4 - 1 = 3
  - Can create 5 - 4 = 1 new Pod
  - Can delete to reach 3 available

  RS-v1: 3 Pods [v1] [v1] [v1]  ██      (desired=3, scaling down)
  RS-v2: 1 Pod  [v2]                     (desired=2, scaling up)
  Total: 4 Pods (1 may not be ready yet)

Step 3: v2 Pod becomes Ready. Controller can now create more.
  RS-v1: 3 Pods [v1] [v1] [v1]          (desired=2, scaling down)
  RS-v2: 2 Pods [v2] [v2]               (desired=3, scaling up)
  Total: 5 Pods (at maxSurge)

Step 4: RS-v1 scales down, RS-v2 scales up.
  RS-v1: 2 Pods [v1] [v1]               (desired=1, scaling down)
  RS-v2: 3 Pods [v2] [v2] [v2]          (desired=4, scaling up)
  Total: 5 Pods

Step 5: Continue scaling.
  RS-v1: 1 Pod  [v1]                    (desired=0, scaling down)
  RS-v2: 4 Pods [v2] [v2] [v2] [v2]    (desired=4, complete)
  Total: 5 Pods

Step 6: Final state.
  RS-v1: 0 Pods                          (kept for rollback history)
  RS-v2: 4 Pods [v2] [v2] [v2] [v2]    (desired=4, current=4)
  Total: 4 Pods ✓
```

**ASCII timeline diagram:**

```
Time ──────────────────────────────────────────────────────────▶

         t0          t1          t2          t3          t4
RS-v1:  ████████    ██████      ████        ██          ░░░░
        4 pods      3 pods      2 pods      1 pod       0 pods

RS-v2:  ░░░░        ██          ████        ██████      ████████
        0 pods      1 pod       2 pods      3 pods      4 pods

Total:  4           4           4-5         4-5         4
        (steady)    (rolling)   (rolling)   (rolling)   (steady)

        ▲ image     ▲ new RS    ▲ gradual   ▲ gradual   ▲ rollout
          updated     created     swap        swap        complete
```

### 16.3.2 Recreate Strategy

The Recreate strategy kills all existing Pods before creating new ones. There is
downtime between the two phases.

```yaml
strategy:
  type: Recreate
```

```
Time ──────────────────────────────────────────────────────────▶

         t0          t1          t2          t3
RS-v1:  ████████    ████████    ░░░░░░░░    ░░░░░░░░
        4 pods      terminating  0 pods      0 pods (deleted)

RS-v2:  ░░░░░░░░    ░░░░░░░░    ████████    ████████
        0 pods      0 pods      creating     4 pods ready

Service: ████████    ░░░░░░░░    ░░░░░░░░    ████████
         serving     DOWNTIME    DOWNTIME    serving

         ▲ image     ▲ kill all  ▲ create    ▲ all
           updated     old pods    new pods    ready
```

**When to use Recreate:**

- **Schema migrations:** The new version requires a database schema change that is
  incompatible with the old version. You can't run both simultaneously.
- **Stateful applications with exclusive access:** The app holds locks on files,
  databases, or external resources that only one instance can hold.
- **License constraints:** Your software license allows only N concurrent instances.
- **Testing/development:** You don't care about downtime and want faster rollouts.

### 16.3.3 Hands-on: RollingUpdate with Different Parameters

```bash
# Create a Deployment
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rolling-demo
  annotations:
    kubernetes.io/change-cause: "Initial deployment v1"
spec:
  replicas: 6
  selector:
    matchLabels:
      app: rolling-demo
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: rolling-demo
    spec:
      containers:
      - name: app
        image: nginx:1.24
        ports:
        - containerPort: 80
EOF

# Wait for all Pods to be ready
kubectl rollout status deployment/rolling-demo

# Update the image (triggers rolling update)
kubectl set image deployment/rolling-demo app=nginx:1.25 \
  --record  # deprecated but still widely used for change-cause

# Watch the rollout in real time
kubectl rollout status deployment/rolling-demo --watch
# Waiting for deployment "rolling-demo" rollout to finish:
#   2 out of 6 new replicas have been updated...
#   4 out of 6 new replicas have been updated...
#   6 out of 6 new replicas have been updated...
#   Waiting for deployment spec update to be observed...
#   deployment "rolling-demo" successfully rolled out

# Observe ReplicaSets during the rollout
kubectl get rs -l app=rolling-demo
# NAME                      DESIRED   CURRENT   READY   AGE
# rolling-demo-5f4b6c7d8    6         6         6       2m    ← new (v1.25)
# rolling-demo-8a9b0c1d2    0         0         0       5m    ← old (v1.24)
```

### 16.3.4 Hands-on: Watching a Rolling Update in Real-Time

```bash
# In one terminal, watch Pods:
kubectl get pods -l app=rolling-demo --watch

# In another terminal, watch ReplicaSets:
kubectl get rs -l app=rolling-demo --watch

# In a third terminal, trigger the update:
kubectl set image deployment/rolling-demo app=nginx:1.25

# You'll see output like:
# PODS:
# rolling-demo-8a9b0-x1   1/1   Running     0     5m
# rolling-demo-8a9b0-x2   1/1   Running     0     5m
# rolling-demo-8a9b0-x3   1/1   Running     0     5m
# rolling-demo-8a9b0-x4   1/1   Running     0     5m
# rolling-demo-8a9b0-x5   1/1   Running     0     5m
# rolling-demo-8a9b0-x6   1/1   Running     0     5m
# rolling-demo-5f4b6-a1   0/1   Pending     0     0s     ← new Pod creating
# rolling-demo-5f4b6-a2   0/1   Pending     0     0s     ← maxSurge=2
# rolling-demo-8a9b0-x1   1/1   Terminating 0     5m     ← old Pod removed
# rolling-demo-5f4b6-a1   1/1   Running     0     3s     ← new Pod ready
# ...continuing until all 6 are replaced...
```

---

## 16.4 Rollback

### 16.4.1 Deployment Revision History

Every time you update a Deployment's Pod template (changing the image, environment
variables, resource requests, etc.), Kubernetes creates a new ReplicaSet and
records it as a new **revision**. Old ReplicaSets are kept (scaled to 0) up to the
`revisionHistoryLimit`.

```
Deployment: web-app
│
├── Revision 1: RS web-app-6d4f5 (nginx:1.23)   ← scaled to 0
├── Revision 2: RS web-app-7a8b9 (nginx:1.24)   ← scaled to 0
├── Revision 3: RS web-app-9c0d1 (nginx:1.25)   ← current, 4 replicas
```

### 16.4.2 Viewing Rollout History

```bash
kubectl rollout history deployment/web-app
# REVISION  CHANGE-CAUSE
# 1         Initial deployment v1.23
# 2         Upgraded to v1.24
# 3         Upgraded to v1.25

# View details of a specific revision
kubectl rollout history deployment/web-app --revision=2
# Shows the full Pod template for that revision
```

### 16.4.3 Rolling Back

```bash
# Roll back to the previous revision
kubectl rollout undo deployment/web-app

# Roll back to a specific revision
kubectl rollout undo deployment/web-app --to-revision=1

# The rollback is itself a new revision:
kubectl rollout history deployment/web-app
# REVISION  CHANGE-CAUSE
# 2         Upgraded to v1.24
# 3         Upgraded to v1.25
# 4         Initial deployment v1.23    ← rolled back to rev 1
```

**What happens during a rollback:**

1. The Deployment controller finds the old ReplicaSet corresponding to the target
   revision.
2. It scales up the old ReplicaSet and scales down the current one, using the same
   `strategy` (RollingUpdate or Recreate).
3. A new revision number is created (rollbacks are not "going back" — they are
   new forward updates that happen to use an old template).

### 16.4.4 Hands-on: Deploying a Broken Version and Rolling Back

```bash
# Deploy a working version
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rollback-demo
  annotations:
    kubernetes.io/change-cause: "v1 - working"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: rollback-demo
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: rollback-demo
    spec:
      containers:
      - name: app
        image: nginx:1.25
        readinessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 2
EOF

kubectl rollout status deployment/rollback-demo

# Deploy a "broken" version (non-existent image)
kubectl set image deployment/rollback-demo app=nginx:99.99.99
kubectl annotate deployment/rollback-demo kubernetes.io/change-cause="v2 - broken image" --overwrite

# Watch the rollout stall
kubectl rollout status deployment/rollback-demo --timeout=30s
# error: deployment "rollback-demo" exceeded its progress deadline

# Check status — old Pods are still running thanks to maxUnavailable=0
kubectl get pods -l app=rollback-demo
# rollback-demo-7f4b6-a1   1/1   Running             0   2m   ← v1 (still serving)
# rollback-demo-7f4b6-a2   1/1   Running             0   2m   ← v1 (still serving)
# rollback-demo-7f4b6-a3   1/1   Running             0   2m   ← v1 (still serving)
# rollback-demo-9c8d7-b1   0/1   ImagePullBackOff    0   30s  ← v2 (broken)

# Roll back!
kubectl rollout undo deployment/rollback-demo
kubectl rollout status deployment/rollback-demo
# deployment "rollback-demo" successfully rolled out

# Verify we're back to v1
kubectl get deployment rollback-demo -o jsonpath='{.spec.template.spec.containers[0].image}'
# nginx:1.25
```

---

## 16.5 Blue-Green and Canary Deployments

Kubernetes does not have built-in blue-green or canary deployment strategies. However,
they can be implemented using standard Kubernetes primitives — Deployments, Services,
and label selectors.

### 16.5.1 Blue-Green Deployment

In a blue-green deployment, you run two complete environments (blue and green). At
any time, only one receives production traffic. To deploy a new version, you deploy
to the idle environment and switch the Service selector.

```
                    ┌──────────────────────────────────────────┐
                    │              Service: web                 │
                    │     selector: { app: web, slot: green }   │
                    └────────────────────┬─────────────────────┘
                                         │
                    ┌────────────────────┤
                    ▼                    │ (no traffic)
        ┌───────────────────┐   ┌───────────────────┐
        │ Deployment: green │   │ Deployment: blue   │
        │ image: v2.0       │   │ image: v1.0       │
        │ replicas: 3       │   │ replicas: 3       │
        │ labels:           │   │ labels:           │
        │   app: web        │   │   app: web        │
        │   slot: green     │   │   slot: blue      │
        └───────────────────┘   └───────────────────┘
              ACTIVE ✓              STANDBY
```

### 16.5.2 Hands-on: Full Blue-Green Deployment

```bash
# Step 1: Create the "blue" Deployment (current production)
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
      slot: blue
  template:
    metadata:
      labels:
        app: web
        slot: blue
        version: v1.0
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo:0.2.3
        args: ["-listen=:8080", "-text=Blue v1.0"]
        ports:
        - containerPort: 8080
EOF

# Step 2: Create the Service pointing to "blue"
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
    slot: blue    # ← Traffic goes to blue
  ports:
  - port: 80
    targetPort: 8080
EOF

# Verify: all traffic goes to blue
kubectl run test --rm -it --image=curlimages/curl -- curl -s http://web
# Blue v1.0

# Step 3: Deploy the "green" version alongside blue
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
      slot: green
  template:
    metadata:
      labels:
        app: web
        slot: green
        version: v2.0
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo:0.2.3
        args: ["-listen=:8080", "-text=Green v2.0"]
        ports:
        - containerPort: 8080
EOF

# Wait for green to be ready
kubectl rollout status deployment/web-green

# Step 4: Smoke test green directly (before switching traffic)
kubectl run test --rm -it --image=curlimages/curl -- \
  curl -s http://$(kubectl get pod -l slot=green -o jsonpath='{.items[0].status.podIP}'):8080
# Green v2.0

# Step 5: Switch traffic from blue to green (the actual deployment!)
kubectl patch service web -p '{"spec":{"selector":{"slot":"green"}}}'

# Verify: traffic now goes to green
kubectl run test --rm -it --image=curlimages/curl -- curl -s http://web
# Green v2.0

# Step 6: If everything looks good, scale down blue
kubectl scale deployment web-blue --replicas=0

# If something goes wrong, switch back instantly:
# kubectl patch service web -p '{"spec":{"selector":{"slot":"blue"}}}'
```

### 16.5.3 Canary Deployment

In a canary deployment, you run the old and new versions side by side, directing a
small percentage of traffic to the new version. If the canary is healthy, you
gradually shift more traffic.

```
                    ┌────────────────────────────────────────┐
                    │            Service: web                 │
                    │   selector: { app: web }                │
                    │   (matches BOTH deployments)            │
                    └────────────────┬───────────────────────┘
                                     │
                    ┌────────────────┴──────────────────┐
                    ▼                                    ▼
        ┌───────────────────┐              ┌───────────────────┐
        │ Deployment: stable │              │ Deployment: canary │
        │ image: v1.0        │              │ image: v2.0        │
        │ replicas: 9        │              │ replicas: 1        │
        │ labels:            │              │ labels:            │
        │   app: web         │              │   app: web         │
        │   version: v1.0    │              │   version: v2.0    │
        └───────────────────┘              └───────────────────┘
              90% traffic                       10% traffic
```

Traffic splitting is proportional to replica count because Kubernetes Services use
round-robin load balancing by default. With 9 stable Pods and 1 canary Pod, roughly
10% of traffic goes to the canary.

### 16.5.4 Hands-on: Full Canary Deployment

```bash
# Step 1: Create the stable Deployment
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: web
      track: stable
  template:
    metadata:
      labels:
        app: web
        track: stable
        version: v1.0
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo:0.2.3
        args: ["-listen=:8080", "-text=Stable v1.0"]
        ports:
        - containerPort: 8080
EOF

# Step 2: Create a Service that matches ALL Pods with app=web
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web           # ← Matches both stable and canary!
  ports:
  - port: 80
    targetPort: 8080
EOF

# Step 3: Deploy the canary with 1 replica (≈10% traffic)
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web
      track: canary
  template:
    metadata:
      labels:
        app: web
        track: canary
        version: v2.0
    spec:
      containers:
      - name: app
        image: hashicorp/http-echo:0.2.3
        args: ["-listen=:8080", "-text=Canary v2.0"]
        ports:
        - containerPort: 8080
EOF

# Step 4: Verify traffic distribution
for i in $(seq 1 20); do
  kubectl run test-$i --rm -it --restart=Never --image=curlimages/curl -- curl -s http://web
done
# ~18 responses: "Stable v1.0"
# ~2 responses:  "Canary v2.0"

# Step 5: Monitor canary health (check error rates, latency, etc.)
# If healthy, gradually increase canary replicas:
kubectl scale deployment web-canary --replicas=3    # ~25% traffic
kubectl scale deployment web-stable --replicas=7

kubectl scale deployment web-canary --replicas=5    # ~50% traffic
kubectl scale deployment web-stable --replicas=5

kubectl scale deployment web-canary --replicas=10   # 100% canary
kubectl scale deployment web-stable --replicas=0

# Step 6: Once confident, make canary the new stable
# Update web-stable to v2.0 and scale it up, then delete web-canary.
```

---

## 16.6 Deployment Conditions and Status

### 16.6.1 Deployment Conditions

A Deployment has three condition types:

| Condition | Meaning |
|-----------|---------|
| **Available** | The Deployment has the minimum number of available Pods (accounting for `minReadySeconds`). |
| **Progressing** | The Deployment is creating a new ReplicaSet, scaling up/down, or waiting for Pods to be Ready. |
| **ReplicaFailure** | The Deployment could not create a Pod (e.g., insufficient quota, image pull error). |

```bash
kubectl get deployment web-app -o jsonpath='{range .status.conditions[*]}{.type}: {.status} - {.message}{"\n"}{end}'
# Available: True - Deployment has minimum availability.
# Progressing: True - ReplicaSet "web-app-7f4b6" has successfully progressed.
```

### 16.6.2 Status Fields

```yaml
status:
  replicas: 5              # Total Pods (old + new RS)
  updatedReplicas: 5       # Pods with the latest template
  readyReplicas: 5         # Pods that pass readiness probe
  availableReplicas: 5     # Pods available for minReadySeconds
  observedGeneration: 3    # Last generation observed by the controller
  conditions: [...]
```

### 16.6.3 Hands-on: Checking Deployment Status from Go

```go
// file: deployment_status.go
//
// Package main demonstrates how to check Deployment status and
// conditions programmatically using client-go. This pattern is
// essential for CI/CD pipelines that need to verify a deployment
// succeeded before proceeding.
//
// The program polls the Deployment status until the rollout is
// complete or a timeout is reached.
package main

import (
	"context"
	"flag"
	"fmt"
	"os"
	"time"

	appsv1 "k8s.io/api/apps/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
)

// isDeploymentComplete checks if a Deployment has finished its rollout
// by comparing the updated replicas count with the desired replicas
// and verifying all replicas are available.
//
// A Deployment is considered complete when:
//  1. All replicas have been updated (no old Pods remain).
//  2. All replicas are available (passed readiness + minReadySeconds).
//  3. No old replicas are running.
func isDeploymentComplete(d *appsv1.Deployment) bool {
	desired := *d.Spec.Replicas
	return d.Status.UpdatedReplicas == desired &&
	d.Status.AvailableReplicas == desired &&
	d.Status.Replicas == desired &&
	d.Status.ObservedGeneration >= d.Generation
}

// getDeploymentCondition returns the condition with the given type,
// or nil if not found.
func getDeploymentCondition(
	status appsv1.DeploymentStatus,
	condType appsv1.DeploymentConditionType,
) *appsv1.DeploymentCondition {
	for i := range status.Conditions {
		if status.Conditions[i].Type == condType {
			return &status.Conditions[i]
		}
	}
	return nil
}

func main() {
	// Parse command-line flags for cluster connection details.
	kubeconfig := flag.String("kubeconfig", os.Getenv("KUBECONFIG"), "Path to kubeconfig")
	namespace := flag.String("namespace", "default", "Kubernetes namespace")
	name := flag.String("deployment", "", "Deployment name to monitor")
	timeout := flag.Duration("timeout", 5*time.Minute, "Timeout for rollout completion")
	flag.Parse()

	if *name == "" {
		fmt.Fprintln(os.Stderr, "Error: --deployment flag is required")
		os.Exit(1)
	}

	// Build Kubernetes client.
	config, err := clientcmd.BuildConfigFromFlags("", *kubeconfig)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error building config: %v\n", err)
		os.Exit(1)
	}
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error creating clientset: %v\n", err)
		os.Exit(1)
	}

	ctx := context.Background()
	deadline := time.Now().Add(*timeout)

	fmt.Printf("Monitoring deployment %s/%s (timeout: %s)\n", *namespace, *name, *timeout)

	// Poll the Deployment status until complete or timeout.
	for time.Now().Before(deadline) {
		// Fetch the current Deployment state.
		d, err := clientset.AppsV1().Deployments(*namespace).Get(ctx, *name, metav1.GetOptions{})
		if err != nil {
			fmt.Fprintf(os.Stderr, "Error getting deployment: %v\n", err)
			time.Sleep(5 * time.Second)
			continue
		}

		// Print current status.
		fmt.Printf("[%s] Replicas: %d desired, %d updated, %d ready, %d available\n",
			time.Now().Format("15:04:05"),
			*d.Spec.Replicas,
			d.Status.UpdatedReplicas,
			d.Status.ReadyReplicas,
			d.Status.AvailableReplicas)

		// Print conditions.
		for _, cond := range d.Status.Conditions {
			fmt.Printf("  Condition: %s=%s (%s)\n", cond.Type, cond.Status, cond.Message)
		}

		// Check for failure conditions.
		failCond := getDeploymentCondition(d.Status, appsv1.DeploymentReplicaFailure)
		if failCond != nil && failCond.Status == "True" {
			fmt.Fprintf(os.Stderr, "Deployment FAILED: %s\n", failCond.Message)
			os.Exit(1)
		}

		// Check for progress deadline exceeded.
		progCond := getDeploymentCondition(d.Status, appsv1.DeploymentProgressing)
		if progCond != nil && progCond.Status == "False" {
			fmt.Fprintf(os.Stderr, "Deployment STALLED: %s\n", progCond.Message)
			os.Exit(1)
		}

		// Check if rollout is complete.
		if isDeploymentComplete(d) {
			fmt.Println("✅ Deployment rollout complete!")
			os.Exit(0)
		}

		time.Sleep(5 * time.Second)
	}

	fmt.Fprintln(os.Stderr, "❌ Timeout waiting for deployment rollout")
	os.Exit(1)
}
```

---

## 16.7 Horizontal Pod Autoscaler (HPA) with Deployments

### 16.7.1 Overview

The Horizontal Pod Autoscaler automatically scales the number of Pods in a
Deployment (or ReplicaSet, or StatefulSet) based on observed metrics — typically
CPU utilization, but also memory, custom metrics, or external metrics.

> **Note:** A comprehensive deep dive into HPA, custom metrics adapters, and
> scaling algorithms is covered in Chapter 37. This section provides a practical
> introduction to using HPA with Deployments.

### 16.7.2 How HPA Works

```
┌────────────────────────────────────────────────────────────┐
│                 HPA Controller Loop                         │
│                 (runs every 15s by default)                  │
│                                                             │
│  1. Fetch current metric values (from metrics-server)       │
│  2. Calculate desired replicas:                             │
│     desired = ceil(current_replicas × (current / target))   │
│  3. Scale Deployment to desired replicas                    │
│                                                             │
│  Example:                                                   │
│    current_replicas = 3                                     │
│    current_cpu = 80%                                        │
│    target_cpu = 50%                                         │
│    desired = ceil(3 × (80/50)) = ceil(4.8) = 5             │
└────────────────────────────────────────────────────────────┘
```

### 16.7.3 Hands-on: Basic HPA with CPU Target

```yaml
# hpa.yaml
#
# This HPA scales the web-app Deployment between 2 and 10 replicas
# based on average CPU utilization. When CPU exceeds 50%, the HPA
# adds more Pods. When CPU drops below 50%, it removes Pods
# (respecting a cooldown period to avoid flapping).
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-app

  minReplicas: 2
  maxReplicas: 10

  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50    # Target 50% CPU utilization

  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 70    # Also consider memory

  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60   # Wait 60s before scaling up
      policies:
      - type: Pods
        value: 4                       # Add at most 4 Pods per 60s
        periodSeconds: 60
      - type: Percent
        value: 100                     # Or double the Pods per 60s
        periodSeconds: 60
      selectPolicy: Max                # Use whichever allows more scaling

    scaleDown:
      stabilizationWindowSeconds: 300  # Wait 5 min before scaling down
      policies:
      - type: Pods
        value: 1                       # Remove at most 1 Pod per 60s
        periodSeconds: 60
      selectPolicy: Min                # Use whichever is more conservative
```

```bash
# Apply the HPA
kubectl apply -f hpa.yaml

# Check HPA status
kubectl get hpa web-app-hpa
# NAME           REFERENCE          TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
# web-app-hpa    Deployment/web-app 12%/50%   2         10        3          1m

# Generate load to trigger scale-up
kubectl run load-generator --image=busybox --restart=Never -- \
  /bin/sh -c "while true; do wget -q -O- http://web-app; done"

# Watch the HPA react
kubectl get hpa web-app-hpa --watch
# NAME           TARGETS    MINPODS   MAXPODS   REPLICAS
# web-app-hpa    12%/50%    2         10        3
# web-app-hpa    67%/50%    2         10        3
# web-app-hpa    67%/50%    2         10        5     ← scaled up!
# web-app-hpa    45%/50%    2         10        5
```

**Important:** Your Pods must have CPU `requests` set. Without requests, the HPA
cannot calculate utilization percentages. Utilization is defined as:

```
utilization = (actual_usage / requested) × 100%
```

---

## 16.8 Go client-go: Managing Deployments Programmatically

### 16.8.1 Overview

This section presents a comprehensive Go program that manages the full lifecycle
of a Kubernetes Deployment using the client-go library. The program:

1. Creates a Deployment
2. Waits for it to be available
3. Updates the image (triggering a rolling update)
4. Watches the rollout progress
5. Optionally rolls back if health checks fail
6. Cleans up

This is the kind of program you'd write for a custom CI/CD pipeline or a
Kubernetes operator.

### 16.8.2 Hands-on: Full Deployment Manager

```go
// file: deployment_manager.go
//
// Package main implements a comprehensive Kubernetes Deployment manager
// using the client-go library. It demonstrates the full lifecycle of a
// Deployment: creation, rolling update, rollout monitoring, automatic
// rollback on failure, and cleanup.
//
// This program illustrates the patterns used in production CI/CD
// pipelines and Kubernetes operators. It uses the watch API to
// observe real-time changes rather than polling, making it both
// efficient and responsive.
//
// Usage:
//
//	go run deployment_manager.go \
//	  --kubeconfig=$HOME/.kube/config \
//	  --namespace=default \
//	  --name=demo-app \
//	  --image=nginx:1.25 \
//	  --replicas=3
//
// Prerequisites:
//   - A running Kubernetes cluster
//   - A valid kubeconfig file
//   - The following Go modules:
//     k8s.io/client-go
//     k8s.io/apimachinery
//     k8s.io/api
package main

import (
	"context"
	"flag"
	"fmt"
	"os"
	"time"

	appsv1 "k8s.io/api/apps/v1"
	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/resource"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/util/intstr"
	"k8s.io/apimachinery/pkg/watch"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
)

// Config holds the command-line configuration for the deployment manager.
type Config struct {
	// Kubeconfig is the path to the Kubernetes configuration file.
	Kubeconfig string

	// Namespace is the Kubernetes namespace in which to manage the Deployment.
	Namespace string

	// Name is the name of the Deployment to manage.
	Name string

	// Image is the container image to deploy.
	Image string

	// Replicas is the desired number of Pod replicas.
	Replicas int32

	// RolloutTimeout is the maximum time to wait for a rollout to complete.
	RolloutTimeout time.Duration
}

// DeploymentManager encapsulates the Kubernetes clientset and
// provides methods for managing Deployments.
type DeploymentManager struct {
	// client is the Kubernetes API clientset.
	client kubernetes.Interface

	// config holds the user-provided configuration.
	config Config
}

// NewDeploymentManager creates a new DeploymentManager with the given
// configuration. It builds the Kubernetes client from the kubeconfig
// file and returns an error if the client cannot be created.
func NewDeploymentManager(cfg Config) (*DeploymentManager, error) {
	// Build the REST configuration from the kubeconfig file.
	restConfig, err := clientcmd.BuildConfigFromFlags("", cfg.Kubeconfig)
	if err != nil {
		return nil, fmt.Errorf("building kubeconfig: %w", err)
	}

	// Create the Kubernetes clientset.
	clientset, err := kubernetes.NewForConfig(restConfig)
	if err != nil {
		return nil, fmt.Errorf("creating clientset: %w", err)
	}

	return &DeploymentManager{
		client: clientset,
		config: cfg,
	}, nil
}

// int32Ptr returns a pointer to the given int32 value.
func int32Ptr(i int32) *int32 { return &i }

// CreateDeployment creates a new Kubernetes Deployment with
// production-ready settings including resource requests/limits,
// health probes, a rolling update strategy, and security contexts.
func (dm *DeploymentManager) CreateDeployment(ctx context.Context) (*appsv1.Deployment, error) {
	fmt.Printf("Creating Deployment %s/%s with image %s (%d replicas)\n",
		dm.config.Namespace, dm.config.Name, dm.config.Image, dm.config.Replicas)

	// Build the Deployment object with all recommended fields.
	deployment := &appsv1.Deployment{
		ObjectMeta: metav1.ObjectMeta{
			Name:      dm.config.Name,
			Namespace: dm.config.Namespace,
			Labels: map[string]string{
				"app":        dm.config.Name,
				"managed-by": "deployment-manager",
			},
			Annotations: map[string]string{
				"kubernetes.io/change-cause": fmt.Sprintf("Created with image %s", dm.config.Image),
			},
		},
		Spec: appsv1.DeploymentSpec{
			// Desired replica count.
			Replicas: int32Ptr(dm.config.Replicas),

			// Label selector — must match Pod template labels.
			Selector: &metav1.LabelSelector{
				MatchLabels: map[string]string{
					"app": dm.config.Name,
				},
			},

			// Rolling update strategy: zero-downtime deploys.
			Strategy: appsv1.DeploymentStrategy{
				Type: appsv1.RollingUpdateDeploymentStrategyType,
				RollingUpdate: &appsv1.RollingUpdateDeployment{
					// Allow 1 extra Pod during updates.
					MaxSurge: &intstr.IntOrString{
						Type:   intstr.Int,
						IntVal: 1,
					},
					// Never reduce below desired count.
					MaxUnavailable: &intstr.IntOrString{
						Type:   intstr.Int,
						IntVal: 0,
					},
				},
			},

			// Keep 5 old ReplicaSets for rollback history.
			RevisionHistoryLimit: int32Ptr(5),

			// A new Pod must be Ready for 10 seconds to be considered
			// Available. This catches Pods that start but crash quickly.
			MinReadySeconds: 10,

			// Mark rollout as failed if no progress in 5 minutes.
			ProgressDeadlineSeconds: int32Ptr(300),

			// Pod template — defines what each replica looks like.
			Template: corev1.PodTemplateSpec{
				ObjectMeta: metav1.ObjectMeta{
					Labels: map[string]string{
						"app":        dm.config.Name,
						"managed-by": "deployment-manager",
					},
				},
				Spec: corev1.PodSpec{
					// Run as non-root for security.
					SecurityContext: &corev1.PodSecurityContext{
						RunAsNonRoot: boolPtr(true),
						RunAsUser:    int64Ptr(65534),
						RunAsGroup:   int64Ptr(65534),
					},

					// Graceful shutdown: give the app 30 seconds to drain.
					TerminationGracePeriodSeconds: int64Ptr(30),

					Containers: []corev1.Container{
						{
							Name:  "app",
							Image: dm.config.Image,
							Ports: []corev1.ContainerPort{
								{
									Name:          "http",
									ContainerPort: 80,
									Protocol:      corev1.ProtocolTCP,
								},
							},

							// Resource management: set both requests and limits.
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

							// Readiness probe: is the app ready for traffic?
							ReadinessProbe: &corev1.Probe{
								ProbeHandler: corev1.ProbeHandler{
									HTTPGet: &corev1.HTTPGetAction{
										Path: "/",
										Port: intstr.FromString("http"),
									},
								},
								InitialDelaySeconds: 5,
								PeriodSeconds:       5,
								TimeoutSeconds:      3,
								FailureThreshold:    3,
								SuccessThreshold:    1,
							},

							// Liveness probe: is the app still alive?
							LivenessProbe: &corev1.Probe{
								ProbeHandler: corev1.ProbeHandler{
									HTTPGet: &corev1.HTTPGetAction{
										Path: "/",
										Port: intstr.FromString("http"),
									},
								},
								InitialDelaySeconds: 10,
								PeriodSeconds:       15,
								TimeoutSeconds:      3,
								FailureThreshold:    3,
							},

							// Container-level security hardening.
							SecurityContext: &corev1.SecurityContext{
								AllowPrivilegeEscalation: boolPtr(false),
								ReadOnlyRootFilesystem:   boolPtr(true),
								Capabilities: &corev1.Capabilities{
									Drop: []corev1.Capability{"ALL"},
								},
							},

							// Mount a writable tmpfs for nginx's temp files.
							VolumeMounts: []corev1.VolumeMount{
								{Name: "tmp", MountPath: "/tmp"},
								{Name: "cache", MountPath: "/var/cache/nginx"},
								{Name: "run", MountPath: "/var/run"},
							},
						},
					},

					// emptyDir volumes for writable directories.
					Volumes: []corev1.Volume{
						{Name: "tmp", VolumeSource: corev1.VolumeSource{EmptyDir: &corev1.EmptyDirVolumeSource{}}},
						{Name: "cache", VolumeSource: corev1.VolumeSource{EmptyDir: &corev1.EmptyDirVolumeSource{}}},
						{Name: "run", VolumeSource: corev1.VolumeSource{EmptyDir: &corev1.EmptyDirVolumeSource{}}},
					},
				},
			},
		},
	}

	// Send the Deployment to the API server.
	created, err := dm.client.AppsV1().Deployments(dm.config.Namespace).Create(
		ctx, deployment, metav1.CreateOptions{},
	)
	if err != nil {
		return nil, fmt.Errorf("creating deployment: %w", err)
	}

	fmt.Printf("✅ Deployment created: %s (UID: %s)\n", created.Name, created.UID)
	return created, nil
}

// WaitForRollout uses the Kubernetes watch API to observe Deployment
// changes in real time and returns when the rollout is complete or
// when the timeout is reached.
//
// This is more efficient than polling because the API server pushes
// events as they occur rather than requiring periodic GET requests.
func (dm *DeploymentManager) WaitForRollout(ctx context.Context) error {
	fmt.Printf("Watching rollout for %s/%s (timeout: %s)\n",
		dm.config.Namespace, dm.config.Name, dm.config.RolloutTimeout)

	// Create a context with the rollout timeout.
	ctx, cancel := context.WithTimeout(ctx, dm.config.RolloutTimeout)
	defer cancel()

	// Start watching the specific Deployment.
	watcher, err := dm.client.AppsV1().Deployments(dm.config.Namespace).Watch(ctx, metav1.ListOptions{
			FieldSelector: fmt.Sprintf("metadata.name=%s", dm.config.Name),
		})
	if err != nil {
		return fmt.Errorf("starting watch: %w", err)
	}
	defer watcher.Stop()

	// Process watch events as they arrive.
	for event := range watcher.ResultChan() {
		switch event.Type {
		case watch.Modified:
			d, ok := event.Object.(*appsv1.Deployment)
			if !ok {
				continue
			}

			// Print current progress.
			fmt.Printf("  [%s] Updated: %d/%d  Ready: %d/%d  Available: %d/%d\n",
				time.Now().Format("15:04:05"),
				d.Status.UpdatedReplicas, *d.Spec.Replicas,
				d.Status.ReadyReplicas, *d.Spec.Replicas,
				d.Status.AvailableReplicas, *d.Spec.Replicas)

			// Check for failure: progress deadline exceeded.
			for _, cond := range d.Status.Conditions {
				if cond.Type == appsv1.DeploymentProgressing &&
				cond.Status == corev1.ConditionFalse {
					return fmt.Errorf("rollout failed: %s", cond.Message)
				}
				if cond.Type == appsv1.DeploymentReplicaFailure &&
				cond.Status == corev1.ConditionTrue {
					return fmt.Errorf("replica failure: %s", cond.Message)
				}
			}

			// Check for success: all replicas updated and available.
			if d.Status.UpdatedReplicas == *d.Spec.Replicas &&
			d.Status.AvailableReplicas == *d.Spec.Replicas &&
			d.Status.Replicas == *d.Spec.Replicas &&
			d.Status.ObservedGeneration >= d.Generation {
				fmt.Println("✅ Rollout complete!")
				return nil
			}

		case watch.Deleted:
			return fmt.Errorf("deployment was deleted during rollout")

		case watch.Error:
			return fmt.Errorf("watch error: %v", event.Object)
		}
	}

	return fmt.Errorf("watch channel closed unexpectedly (timeout?)")
}

// UpdateImage updates the Deployment's container image, triggering a
// rolling update. It annotates the Deployment with the change cause
// for the rollout history.
func (dm *DeploymentManager) UpdateImage(ctx context.Context, newImage string) error {
	fmt.Printf("Updating image to %s\n", newImage)

	// Fetch the current Deployment.
	d, err := dm.client.AppsV1().Deployments(dm.config.Namespace).Get(
		ctx, dm.config.Name, metav1.GetOptions{},
	)
	if err != nil {
		return fmt.Errorf("getting deployment: %w", err)
	}

	// Update the container image.
	for i := range d.Spec.Template.Spec.Containers {
		if d.Spec.Template.Spec.Containers[i].Name == "app" {
			d.Spec.Template.Spec.Containers[i].Image = newImage
			break
		}
	}

	// Record the change cause for rollout history.
	if d.Annotations == nil {
		d.Annotations = make(map[string]string)
	}
	d.Annotations["kubernetes.io/change-cause"] = fmt.Sprintf("Updated image to %s", newImage)

	// Apply the update.
	_, err = dm.client.AppsV1().Deployments(dm.config.Namespace).Update(
		ctx, d, metav1.UpdateOptions{},
	)
	if err != nil {
		return fmt.Errorf("updating deployment: %w", err)
	}

	fmt.Println("Image update applied — rolling update triggered")
	return nil
}

// Rollback reverts the Deployment to its previous revision. This is
// equivalent to `kubectl rollout undo`.
//
// The Kubernetes API does not have a direct "rollback" endpoint in
// apps/v1. Instead, we find the previous ReplicaSet's Pod template
// and apply it as the new template.
func (dm *DeploymentManager) Rollback(ctx context.Context) error {
	fmt.Println("⚠️  Initiating rollback to previous revision...")

	// Get the current Deployment.
	d, err := dm.client.AppsV1().Deployments(dm.config.Namespace).Get(
		ctx, dm.config.Name, metav1.GetOptions{},
	)
	if err != nil {
		return fmt.Errorf("getting deployment: %w", err)
	}

	// List all ReplicaSets owned by this Deployment.
	rsList, err := dm.client.AppsV1().ReplicaSets(dm.config.Namespace).List(ctx, metav1.ListOptions{
			LabelSelector: fmt.Sprintf("app=%s", dm.config.Name),
		})
	if err != nil {
		return fmt.Errorf("listing replicasets: %w", err)
	}

	// Filter to ReplicaSets that are owned by this Deployment
	// and find the second-most-recent one (the one to roll back to).
	var previousRS *appsv1.ReplicaSet
	var maxRevision int64
	var secondMaxRevision int64

	for i := range rsList.Items {
		rs := &rsList.Items[i]

		// Verify ownership.
		isOwned := false
		for _, ref := range rs.OwnerReferences {
			if ref.UID == d.UID {
				isOwned = true
				break
			}
		}
		if !isOwned {
			continue
		}

		// Parse the revision number from the annotation.
		revStr := rs.Annotations["deployment.kubernetes.io/revision"]
		var rev int64
		fmt.Sscanf(revStr, "%d", &rev)

		if rev > maxRevision {
			secondMaxRevision = maxRevision
			previousRS = nil // Reset — we need to find the one at secondMax
			maxRevision = rev
		}
		if rev == secondMaxRevision || (rev > secondMaxRevision && rev < maxRevision) {
			secondMaxRevision = rev
			previousRS = rs
		}
	}

	if previousRS == nil {
		return fmt.Errorf("no previous revision found for rollback")
	}

	fmt.Printf("Rolling back to revision %d (RS: %s)\n", secondMaxRevision, previousRS.Name)

	// Apply the previous ReplicaSet's Pod template to the Deployment.
	d.Spec.Template = previousRS.Spec.Template
	d.Annotations["kubernetes.io/change-cause"] = fmt.Sprintf(
		"Rolled back to revision %d", secondMaxRevision)

	_, err = dm.client.AppsV1().Deployments(dm.config.Namespace).Update(
		ctx, d, metav1.UpdateOptions{},
	)
	if err != nil {
		return fmt.Errorf("applying rollback: %w", err)
	}

	fmt.Println("Rollback applied — waiting for new rollout to complete")
	return nil
}

// RollingRestart triggers a rolling restart of all Pods in the
// Deployment without changing the image or any other spec field.
// This is equivalent to `kubectl rollout restart`.
//
// It works by patching the Pod template's annotations with a
// restart timestamp, causing the Deployment controller to detect
// a template change and initiate a rolling update.
func (dm *DeploymentManager) RollingRestart(ctx context.Context) error {
	fmt.Println("Triggering rolling restart...")

	// Fetch the current Deployment.
	d, err := dm.client.AppsV1().Deployments(dm.config.Namespace).Get(
		ctx, dm.config.Name, metav1.GetOptions{},
	)
	if err != nil {
		return fmt.Errorf("getting deployment: %w", err)
	}

	// Add a restart annotation to the Pod template. This changes the
	// template hash, which triggers the Deployment controller to create
	// a new ReplicaSet and perform a rolling update.
	if d.Spec.Template.Annotations == nil {
		d.Spec.Template.Annotations = make(map[string]string)
	}
	d.Spec.Template.Annotations["kubectl.kubernetes.io/restartedAt"] = time.Now().Format(time.RFC3339)

	_, err = dm.client.AppsV1().Deployments(dm.config.Namespace).Update(
		ctx, d, metav1.UpdateOptions{},
	)
	if err != nil {
		return fmt.Errorf("triggering restart: %w", err)
	}

	fmt.Println("Rolling restart triggered")
	return nil
}

// Cleanup deletes the Deployment and all its Pods using foreground
// cascading deletion.
func (dm *DeploymentManager) Cleanup(ctx context.Context) error {
	fmt.Printf("Deleting Deployment %s/%s\n", dm.config.Namespace, dm.config.Name)

	propagation := metav1.DeletePropagationForeground
	err := dm.client.AppsV1().Deployments(dm.config.Namespace).Delete(
		ctx, dm.config.Name, metav1.DeleteOptions{
			PropagationPolicy: &propagation,
		},
	)
	if err != nil {
		return fmt.Errorf("deleting deployment: %w", err)
	}

	fmt.Println("✅ Deployment deleted")
	return nil
}

// boolPtr returns a pointer to the given bool.
func boolPtr(b bool) *bool { return &b }

// int64Ptr returns a pointer to the given int64.
func int64Ptr(i int64) *int64 { return &i }

func main() {
	// Parse command-line flags.
	cfg := Config{}
	flag.StringVar(&cfg.Kubeconfig, "kubeconfig", os.Getenv("KUBECONFIG"), "Path to kubeconfig")
	flag.StringVar(&cfg.Namespace, "namespace", "default", "Kubernetes namespace")
	flag.StringVar(&cfg.Name, "name", "demo-app", "Deployment name")
	flag.StringVar(&cfg.Image, "image", "nginx:1.25", "Container image")
	flag.Int64Var((*int64)(&cfg.Replicas), "replicas", 3, "Number of replicas")
	cfg.RolloutTimeout = 5 * time.Minute
	flag.Parse()

	// Create the deployment manager.
	dm, err := NewDeploymentManager(cfg)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error: %v\n", err)
		os.Exit(1)
	}

	ctx := context.Background()

	// ─── Phase 1: Create the Deployment ─────────────────────────────
	fmt.Println("\n════════════════════════════════════════")
	fmt.Println("  Phase 1: Create Deployment")
	fmt.Println("════════════════════════════════════════")
	if _, err := dm.CreateDeployment(ctx); err != nil {
		fmt.Fprintf(os.Stderr, "Create failed: %v\n", err)
		os.Exit(1)
	}

	// ─── Phase 2: Wait for Initial Rollout ──────────────────────────
	fmt.Println("\n════════════════════════════════════════")
	fmt.Println("  Phase 2: Wait for Rollout")
	fmt.Println("════════════════════════════════════════")
	if err := dm.WaitForRollout(ctx); err != nil {
		fmt.Fprintf(os.Stderr, "Rollout failed: %v\n", err)
		os.Exit(1)
	}

	// ─── Phase 3: Update Image (Rolling Update) ─────────────────────
	fmt.Println("\n════════════════════════════════════════")
	fmt.Println("  Phase 3: Update Image (Rolling Update)")
	fmt.Println("════════════════════════════════════════")
	newImage := "nginx:1.25-alpine"
	if err := dm.UpdateImage(ctx, newImage); err != nil {
		fmt.Fprintf(os.Stderr, "Update failed: %v\n", err)
		os.Exit(1)
	}

	// ─── Phase 4: Watch Rolling Update Progress ─────────────────────
	fmt.Println("\n════════════════════════════════════════")
	fmt.Println("  Phase 4: Watch Rollout Progress")
	fmt.Println("════════════════════════════════════════")
	if err := dm.WaitForRollout(ctx); err != nil {
		fmt.Fprintf(os.Stderr, "Rollout failed — initiating rollback\n")

		// ─── Phase 4b: Automatic Rollback on Failure ────────────────
		fmt.Println("\n════════════════════════════════════════")
		fmt.Println("  Phase 4b: Automatic Rollback")
		fmt.Println("════════════════════════════════════════")
		if rbErr := dm.Rollback(ctx); rbErr != nil {
			fmt.Fprintf(os.Stderr, "Rollback failed: %v\n", rbErr)
			os.Exit(1)
		}
		if err := dm.WaitForRollout(ctx); err != nil {
			fmt.Fprintf(os.Stderr, "Rollback rollout failed: %v\n", err)
			os.Exit(1)
		}
	}

	// ─── Phase 5: Demonstrate Rolling Restart ───────────────────────
	fmt.Println("\n════════════════════════════════════════")
	fmt.Println("  Phase 5: Rolling Restart")
	fmt.Println("════════════════════════════════════════")
	if err := dm.RollingRestart(ctx); err != nil {
		fmt.Fprintf(os.Stderr, "Restart failed: %v\n", err)
	} else {
		if err := dm.WaitForRollout(ctx); err != nil {
			fmt.Fprintf(os.Stderr, "Restart rollout failed: %v\n", err)
		}
	}

	// ─── Phase 6: Cleanup ───────────────────────────────────────────
	fmt.Println("\n════════════════════════════════════════")
	fmt.Println("  Phase 6: Cleanup")
	fmt.Println("════════════════════════════════════════")
	if err := dm.Cleanup(ctx); err != nil {
		fmt.Fprintf(os.Stderr, "Cleanup failed: %v\n", err)
		os.Exit(1)
	}

	fmt.Println("\n🎉 All phases complete!")
}
```

---

## 16.9 The Controller Reconciliation Pattern

Both the ReplicaSet controller and the Deployment controller follow the same
fundamental pattern, which is the heart of Kubernetes' design:

```
┌──────────────────────────────────────────────────────────────────┐
│               The Kubernetes Controller Pattern                   │
│                                                                   │
│                        ┌──────────────┐                           │
│              ┌────────▶│   Informer    │                           │
│              │         │  (Watch API)  │                           │
│              │         └──────┬───────┘                           │
│              │                │ events                             │
│              │                ▼                                    │
│              │         ┌──────────────┐                           │
│              │         │  Work Queue  │                           │
│              │         └──────┬───────┘                           │
│              │                │ dequeue                            │
│              │                ▼                                    │
│              │         ┌──────────────────────────────────┐       │
│              │         │        Reconcile(key)            │       │
│              │         │                                  │       │
│              │         │  1. Get desired state (Spec)     │       │
│              │         │  2. Get actual state (cluster)   │       │
│              │         │  3. Compute diff                 │       │
│              │         │  4. Take action to converge      │       │
│              │         │  5. Update status                │       │
│  retry on    │         └──────────────────────────────────┘       │
│  error       │                │                                   │
│              │                │ error?                             │
│              └────────────────┘                                   │
│                                                                   │
│  This is a LEVEL-TRIGGERED system, not EDGE-TRIGGERED.           │
│  The reconciler checks the full state, not just the change.       │
│  This makes it self-healing: even if events are missed,          │
│  the next reconciliation corrects the state.                      │
└──────────────────────────────────────────────────────────────────┘
```

**Deployment controller specifics:**

1. Watches Deployments, ReplicaSets, and Pods.
2. On each reconciliation:
   - Finds all ReplicaSets owned by the Deployment.
   - Identifies the "new" RS (matching the current Pod template) and "old" RSs.
   - If no matching RS exists, creates one.
   - Scales the new RS up and old RSs down according to the strategy.
   - Updates Deployment status and conditions.

**ReplicaSet controller specifics:**

1. Watches ReplicaSets and Pods.
2. On each reconciliation:
   - Lists Pods matching the selector.
   - Filters to Pods owned by (or adoptable by) this RS.
   - Creates or deletes Pods to match the desired count.
   - Updates RS status.

---

## 16.10 Advanced Topics

### 16.10.1 Deployment Pausing and Resuming

You can pause a Deployment to make multiple changes without triggering a rollout
for each one:

```bash
# Pause — changes to the template won't trigger rollouts
kubectl rollout pause deployment/web-app

# Make multiple changes
kubectl set image deployment/web-app app=nginx:1.26
kubectl set resources deployment/web-app -c=app --limits=cpu=1,memory=512Mi
kubectl set env deployment/web-app -c=app LOG_LEVEL=debug

# Resume — all changes are applied as a single rollout
kubectl rollout resume deployment/web-app
```

### 16.10.2 Proportional Scaling

During a rolling update, if you scale the Deployment, Kubernetes distributes the
new replicas proportionally between the old and new ReplicaSets. For example:

- Deployment has 10 replicas, rolling update is in progress.
- Old RS has 6 Pods, new RS has 4 Pods.
- You scale to 15 replicas.
- Old RS scales to 9 Pods (6/10 × 15 = 9).
- New RS scales to 6 Pods (4/10 × 15 = 6).

This ensures the ratio of old-to-new Pods is maintained during the scale operation.

### 16.10.3 Pod Template Hash

The Deployment controller adds a `pod-template-hash` label to each ReplicaSet and
its Pods. This label is a hash of the Pod template spec, ensuring each template
version gets a unique ReplicaSet.

```bash
kubectl get rs -l app=web-app --show-labels
# NAME                 DESIRED   CURRENT   READY   LABELS
# web-app-6d4f5b7c8    3         3         3       app=web-app,pod-template-hash=6d4f5b7c8
# web-app-7a8b9c0d1    0         0         0       app=web-app,pod-template-hash=7a8b9c0d1
```

**Important:** Never modify the `pod-template-hash` label manually. It is managed
by the Deployment controller and used to associate ReplicaSets with their template
versions.

### 16.10.4 Controlling Rollout Speed

Beyond `maxSurge` and `maxUnavailable`, you can control rollout speed with:

- **`minReadySeconds`** — how long a new Pod must be Ready before being counted
  as Available. Higher values slow the rollout but catch more bugs.
- **`progressDeadlineSeconds`** — how long to wait for progress before marking
  the rollout as failed. Set this higher than your slowest expected startup time.
- **Readiness probe tuning** — slower readiness probes naturally slow the rollout
  because the controller waits for readiness before scaling further.

```yaml
# Slow, cautious rollout for a critical service
spec:
  minReadySeconds: 30              # Pod must be Ready for 30s
  progressDeadlineSeconds: 600     # Allow 10 min for progress
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1                  # Only 1 extra Pod at a time
      maxUnavailable: 0            # Never fewer than desired count
```

---

## 16.11 Summary

In this chapter, we explored the two most important controllers for stateless
workloads in Kubernetes:

**ReplicaSet:**
- Ensures a specified number of identical Pods are always running.
- Uses label selectors to identify owned Pods.
- Establishes ownership via ownerReferences.
- Controller follows the watch → reconcile → act pattern.
- Prefers deleting newer, unscheduled, or more-restarted Pods when scaling down.

**Deployment:**
- Manages ReplicaSets to provide declarative rolling updates and rollbacks.
- Owns multiple ReplicaSets (one per revision) — only the latest has replicas > 0.
- **RollingUpdate strategy** — incrementally replaces Pods, controlled by
  `maxSurge` and `maxUnavailable`.
- **Recreate strategy** — kills all old Pods before creating new ones.
- Maintains revision history for rollbacks.
- Supports pausing, resuming, and proportional scaling during rollouts.

**Advanced deployment patterns:**
- **Blue-green** — two Deployments, switch Service selector between them.
- **Canary** — two Deployments behind the same Service, control traffic split
  via replica count ratios.

**HPA integration:**
- Automatically scales Deployment replicas based on CPU, memory, or custom metrics.
- Uses a configurable algorithm with scale-up and scale-down stabilization windows.

**Programmatic management:**
- client-go provides full CRUD and watch capabilities for Deployments.
- The watch API enables real-time rollout monitoring without polling.
- Rollbacks can be automated by detecting failure conditions during rollout.

**The controller pattern:**
- Both controllers use level-triggered reconciliation — they check full state, not
  just changes.
- This makes the system self-healing: even missed events are corrected on the next
  reconciliation cycle.

In the next chapter, we'll explore **Services and Networking** — how Pods discover
each other and how traffic reaches them from outside the cluster.
