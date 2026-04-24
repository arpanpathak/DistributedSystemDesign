# Chapter 32: The Controller Pattern — Reconciliation Loops

> *"The controller pattern is the beating heart of Kubernetes. Every behavior in the
> cluster — from scheduling pods to scaling deployments to issuing certificates — is
> driven by a controller watching the world, comparing it to what should be, and
> taking action to close the gap."*

If you have ever wondered how Kubernetes actually *works* — how a Deployment
creates ReplicaSets, how a ReplicaSet creates Pods, how a Service gets an IP —
the answer is always the same: **a controller did it**. Kubernetes is not a
monolithic orchestrator that executes a master plan. It is a collection of
dozens of independent controllers, each responsible for one small piece of
reality, all running concurrently, all following the same elegant pattern.

In this chapter we will dissect that pattern down to the metal. We will build
controllers from scratch using nothing but `client-go`, then rebuild them with
the `controller-runtime` framework to see what it buys us. By the end, you
will understand not just *how* to write a controller, but *why* the pattern
works — and why it is one of the most powerful ideas in distributed systems.

---

## 32.1 The Controller Pattern — Heart of Kubernetes

### 32.1.1 Desired State vs Actual State

Every object in Kubernetes has two halves:

| Field    | Meaning                                          |
|----------|--------------------------------------------------|
| `spec`   | **Desired state** — what you *want*              |
| `status` | **Actual state** — what *is*                     |

When you submit a Deployment with `replicas: 3`, you are declaring desired
state. You are *not* issuing a command ("create three pods"). You are making a
statement of intent: "I want three replicas to exist."

The job of the Deployment controller is to continuously compare the desired
state (3 replicas) with the actual state (however many pods exist right now)
and take whatever action is needed to close the gap. If there are only 2 pods,
it creates one. If there are 4 pods, it deletes one. If all 3 exist but one
is unhealthy, it replaces it.

This is fundamentally different from imperative systems. In an imperative
system, you issue commands: "create pod A", "delete pod B", "scale to 3".
Each command is a point-in-time instruction. If the command is lost, or if the
system crashes halfway through, you may end up in an inconsistent state.

In a declarative system, the desired state is persistent (stored in etcd). The
controller continuously reconciles reality toward that state. If the
controller crashes and restarts, it simply reads the desired state again and
picks up where it left off. This is why Kubernetes is so resilient — the
desired state is the *source of truth*, and controllers are stateless workers
that can be restarted at any time.

### 32.1.2 The Reconciliation Loop

Every controller follows the same fundamental loop:

```text
┌─────────────────────────────────────────────────────────────────┐
│                     THE RECONCILIATION LOOP                     │
│                                                                 │
│    ┌──────────┐     ┌──────────┐     ┌──────────┐              │
│    │          │     │          │     │          │              │
│    │ OBSERVE  │────▶│ COMPARE  │────▶│   ACT    │              │
│    │          │     │          │     │          │              │
│    └──────────┘     └──────────┘     └──────────┘              │
│         ▲                                  │                    │
│         │                                  │                    │
│         └──────────────────────────────────┘                    │
│                      REPEAT                                     │
│                                                                 │
│  OBSERVE: Read current state from the cluster                   │
│  COMPARE: Diff desired state (spec) vs actual state (reality)   │
│  ACT:     Create, update, or delete resources to close the gap  │
│  REPEAT:  Go back to OBSERVE and do it again                    │
└─────────────────────────────────────────────────────────────────┘
```

**Step 1: Observe.** The controller reads the current state of the world. This
might mean listing all Pods that match a label selector, checking the status of
a node, or querying an external system.

**Step 2: Compare.** The controller compares the current state to the desired
state stored in the Kubernetes API (etcd). This is the "diff" step. The
controller determines what, if anything, needs to change.

**Step 3: Act.** The controller takes action to close the gap. This might mean
creating new resources, updating existing ones, deleting resources that should
not exist, or calling external APIs.

**Step 4: Repeat.** The controller goes back to step 1. This is not a one-shot
process — it runs continuously for the lifetime of the controller.

### 32.1.3 Level-Triggered vs Edge-Triggered

This distinction is borrowed from electrical engineering, and it is *critical*
to understanding why the controller pattern is so robust.

**Edge-triggered** systems react to *changes* (edges). "A pod was created" is
an edge. "A pod was deleted" is an edge. An edge-triggered controller would
say: "When a pod is deleted, create a replacement."

**Level-triggered** systems react to the *current state* (levels). "There are
2 pods but we want 3" is a level. A level-triggered controller says: "If the
number of pods is less than desired, create pods until it matches."

Why is level-triggered better? Because edges can be missed. If your controller
is down when a pod is deleted, it never sees the deletion event. When it comes
back up, it has no idea that anything happened. But if it is level-triggered,
it does not need to know *what* happened — it just looks at the current state
(2 pods) and the desired state (3 pods) and takes action.

```text
Edge-triggered (fragile):
  Event: "Pod X deleted"  ──▶  Handler: "Create replacement"
  Problem: If event is missed, no replacement is ever created.

Level-triggered (robust):
  State: "2 pods exist, 3 desired"  ──▶  Reconcile: "Create 1 pod"
  If controller restarts, it reads the same state and takes the same action.
  Events are hints, not commands.
```

In Kubernetes, controllers are fundamentally level-triggered. Events from
informers (which we will cover next) serve as *hints* that something may have
changed, triggering a reconciliation. But the reconciler itself always reads
the full current state and computes the full diff. This means:

- Events can be lost → controller will still converge
- Events can be duplicated → controller will still converge
- Events can arrive out of order → controller will still converge

This is an incredibly powerful property for distributed systems.

### 32.1.4 Idempotency

Because controllers are level-triggered and may be called at any time (on
restart, on re-watch, on duplicate events), the reconcile function **must be
idempotent**. That is: calling it multiple times with the same input must
produce the same result as calling it once.

This means:

- **Don't create if already exists.** Check first, create only if missing.
- **Don't update if already correct.** Compare current state to desired state,
  update only if different.
- **Don't delete if already gone.** Handle "not found" errors gracefully.
- **Don't assume order.** Your reconcile function may be called before, after,
  or during other operations.

Idempotency is not optional. It is a fundamental requirement of the controller
pattern. If your reconcile function is not idempotent, your controller will
behave unpredictably under real-world conditions (network partitions, restarts,
leader election failover).

### 32.1.5 Kubernetes Is Controllers All the Way Down

Here are just a few of the controllers running in a typical Kubernetes cluster:

| Controller              | Watches            | Reconciles                          |
|-------------------------|--------------------|-------------------------------------|
| Deployment controller   | Deployments        | Creates/updates ReplicaSets         |
| ReplicaSet controller   | ReplicaSets, Pods  | Creates/deletes Pods                |
| StatefulSet controller  | StatefulSets       | Creates Pods with stable identities |
| Job controller          | Jobs, Pods         | Creates Pods for batch work         |
| Service controller      | Services           | Configures load balancers           |
| Endpoint controller     | Services, Pods     | Maintains Endpoints objects         |
| Namespace controller    | Namespaces         | Cleans up on namespace deletion     |
| Node controller         | Nodes              | Monitors node health                |
| Certificate controller  | CertificateRequests| Issues TLS certificates             |

The `kube-controller-manager` binary runs ~30 controllers in a single process.
Each one follows the same pattern we are describing in this chapter. When you
write your own controller, you are joining this ecosystem — your controller is
a first-class citizen, no different from the built-in ones.

---

## 32.2 Informers — Efficient State Watching

### 32.2.1 The Problem: Why Not Just Poll?

The simplest way to implement the "Observe" step of the reconciliation loop
would be to poll the API server: every N seconds, list all the resources you
care about and compare them to what you had last time.

This works for small clusters. It does *not* work at scale, for several reasons:

1. **Bandwidth.** If you have 10,000 pods and poll every second, you are
   transferring enormous amounts of data, most of which has not changed.

2. **Latency.** If you poll every 10 seconds, you may not notice a change for
   up to 10 seconds. If you poll every second, you are wasting bandwidth.

3. **API server load.** The API server has to serialize all those objects and
   send them over the wire. If 50 controllers are all polling, the API server
   becomes the bottleneck.

4. **etcd load.** Every list operation may hit etcd. etcd is the most
   constrained component in a Kubernetes cluster.

Kubernetes solves this with a mechanism called **watch**. Instead of polling,
the client opens a long-lived HTTP connection to the API server and receives
a stream of events as resources change. This is efficient: the API server
only sends data when something actually changes.

But raw watches have their own problems:

- Watches can disconnect (network issues, API server restarts).
- When a watch disconnects, you need to re-list to get the current state,
  then re-watch from that point.
- You need to handle the "too old resource version" error (the watch history
  has been compacted).
- Multiple controllers watching the same resource type each open a separate
  watch, multiplying the load.

The **informer** abstraction in `client-go` solves all of these problems.

### 32.2.2 SharedInformer Architecture

The informer is one of the most elegant pieces of engineering in the
Kubernetes ecosystem. Let us trace the data flow:

```text
┌──────────────────────────────────────────────────────────────────────┐
│                      INFORMER ARCHITECTURE                           │
│                                                                      │
│  ┌─────────────┐                                                     │
│  │ API Server  │                                                     │
│  │             │  list + watch                                       │
│  └──────┬──────┘                                                     │
│         │                                                            │
│         ▼                                                            │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐            │
│  │  Reflector  │────▶│ DeltaFIFO   │────▶│  Indexer /   │            │
│  │             │     │   Queue     │     │   Store     │            │
│  └─────────────┘     └──────┬──────┘     └─────────────┘            │
│                             │                                        │
│                             ▼                                        │
│                     ┌───────────────┐                                │
│                     │    Event      │                                │
│                     │  Handlers     │                                │
│                     │               │                                │
│                     │  • OnAdd()    │                                │
│                     │  • OnUpdate() │                                │
│                     │  • OnDelete() │                                │
│                     └───────┬───────┘                                │
│                             │                                        │
│                             ▼                                        │
│                     ┌───────────────┐                                │
│                     │  Work Queue   │  ◀── Controller enqueues keys  │
│                     │               │                                │
│                     └───────┬───────┘                                │
│                             │                                        │
│                             ▼                                        │
│                     ┌───────────────┐                                │
│                     │  Reconciler   │  ◀── Controller dequeues keys  │
│                     │               │      and reconciles            │
│                     └───────────────┘                                │
└──────────────────────────────────────────────────────────────────────┘
```

**Reflector.** The reflector is the component that talks to the API server. It
performs an initial **list** to get all existing resources, then opens a
**watch** to receive a stream of changes. If the watch disconnects, the
reflector automatically re-lists and re-watches. It handles all the complexity
of resource versions, bookmark events, and watch timeouts.

The reflector pushes events into the DeltaFIFO queue.

**DeltaFIFO.** This is a queue of *deltas* — changes to objects. Each delta
has a type (Added, Modified, Deleted, Replaced, Sync) and the object that
changed. The FIFO part means items are processed in order. The "Delta" part
means that if an object changes multiple times before being processed, all
the changes are preserved (unlike a simple cache which only keeps the latest).

Key property: DeltaFIFO deduplicates by key. If a pod changes 5 times before
the consumer processes it, the consumer sees all 5 deltas, but they are grouped
under one key. This prevents the consumer from falling behind.

**Indexer / Store.** The store is a local in-memory cache of all resources the
informer is watching. When a delta is processed from the DeltaFIFO, the store
is updated. This means your controller can read the current state of any
resource *without* making an API call — it just reads from the local cache.

The indexer adds indices on top of the store. For example, you can index pods
by node name, by namespace, or by any other field. This makes lookups O(1)
instead of O(n).

**Event Handlers.** After the store is updated, the informer calls registered
event handlers:

- `OnAdd(obj)` — called when a new object appears (either on initial list or
  when a new object is created).
- `OnUpdate(oldObj, newObj)` — called when an existing object is modified.
- `OnDelete(obj)` — called when an object is deleted.

These handlers are where your controller plugs in. Typically, the handler
extracts the key (namespace/name) of the object and enqueues it into a work
queue.

**SharedInformer.** A "shared" informer means that multiple controllers can
share the same informer (and therefore the same API server watch connection).
If you have three controllers that all watch Pods, you do not need three
separate watch connections — they all share one SharedInformer.

The `SharedInformerFactory` manages this sharing. You create one factory, and
each controller asks the factory for the informer it needs. If two controllers
ask for a Pod informer, they get the same one.

### 32.2.3 Hands-On: Building a Simple Informer in Go

Let us build a minimal program that uses an informer to watch Pods and print
events. This is the foundation for everything that follows.

```go
// Package main demonstrates the informer pattern — the foundational
// mechanism by which Kubernetes controllers efficiently watch for
// resource changes without polling the API server.
//
// This program:
//  1. Creates a Kubernetes client from the local kubeconfig.
//  2. Creates a SharedInformerFactory that manages informer lifecycle.
//  3. Obtains a Pod informer from the factory.
//  4. Registers event handlers (OnAdd, OnUpdate, OnDelete) that print
//     a human-readable summary of each event.
//  5. Starts the informer and blocks until interrupted.
//
// This is intentionally simple — no work queue, no reconciler. We will
// add those in subsequent examples.
package main

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"
	"time"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
)

func main() {
	// ---------- Step 1: Build a Kubernetes client ----------
	//
	// clientcmd.BuildConfigFromFlags reads the kubeconfig file (typically
	// ~/.kube/config) and returns a *rest.Config suitable for creating
	// API clients. The first argument is the master URL (empty string
	// means "use kubeconfig"), and the second is the kubeconfig path.
	kubeconfig := os.Getenv("KUBECONFIG")
	if kubeconfig == "" {
		kubeconfig = os.Getenv("HOME") + "/.kube/config"
	}

	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error building kubeconfig: %v\n", err)
		os.Exit(1)
	}

	// kubernetes.NewForConfig creates a typed Kubernetes clientset that
	// provides access to every built-in API group (core/v1, apps/v1, etc.).
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		fmt.Fprintf(os.Stderr, "Error creating clientset: %v\n", err)
		os.Exit(1)
	}

	// ---------- Step 2: Create a SharedInformerFactory ----------
	//
	// The factory creates and manages informers. The second argument is the
	// resync period — how often the informer will redeliver ALL objects to
	// the event handlers (as Update events) even if nothing changed. This
	// acts as a safety net to ensure controllers do not miss changes.
	//
	// A resync period of 30 seconds is typical for development. In
	// production, 10-15 minutes is more common to reduce API server load.
	factory := informers.NewSharedInformerFactory(clientset, 30*time.Second)

	// ---------- Step 3: Get a Pod informer ----------
	//
	// factory.Core().V1().Pods() returns a PodInformer. The first time
	// this is called, the factory creates the informer. Subsequent calls
	// return the same instance (this is the "shared" part).
	podInformer := factory.Core().V1().Pods()

	// ---------- Step 4: Register event handlers ----------
	//
	// The Informer() method returns the underlying SharedIndexInformer,
	// which provides AddEventHandler for registering callbacks.
	//
	// IMPORTANT: Event handlers should be fast and non-blocking. Heavy
	// work belongs in the reconciler, not here. The typical pattern is
	// to extract the object key and enqueue it into a work queue.
	podInformer.Informer().AddEventHandler(
		// cache.ResourceEventHandlerFuncs is a struct that implements
		// the ResourceEventHandler interface. Each field is optional.
		&informerEventHandler{},
	)

	// ---------- Step 5: Start the factory and block ----------
	//
	// factory.Start() starts all informers that have been created by the
	// factory. The stopCh channel controls when informers shut down.
	stopCh := make(chan struct{})

	// Set up signal handling so we can gracefully shut down on Ctrl-C.
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		<-sigCh
		fmt.Println("\nShutting down...")
		close(stopCh)
	}()

	fmt.Println("Starting Pod informer... (Ctrl-C to stop)")
	factory.Start(stopCh)

	// factory.WaitForCacheSync waits until ALL informers have completed
	// their initial list and populated their local caches. This is
	// critical — you must not start reconciling until the cache is warm,
	// or you will see "phantom deletes" (objects that exist but are not
	// yet in the cache).
	factory.WaitForCacheSync(stopCh)
	fmt.Println("Cache synced. Watching for Pod events...")

	// Block until stopCh is closed.
	<-stopCh
}

// informerEventHandler implements cache.ResourceEventHandler for Pods.
// Each method is called by the informer when a Pod event occurs.
type informerEventHandler struct{}

// OnAdd is called when a new Pod is observed. This includes both Pods
// that already exist when the informer starts (initial list) and Pods
// that are created after the watch is established.
func (h *informerEventHandler) OnAdd(obj interface{}, isInInitialList bool) {
	pod, ok := obj.(*corev1.Pod)
	if !ok {
		return
	}
	fmt.Printf("[ADD]    %s/%s  phase=%s  initial=%v\n",
		pod.Namespace, pod.Name, pod.Status.Phase, isInInitialList)
}

// OnUpdate is called when a Pod is modified. The informer provides both
// the old and new versions of the object so you can compute a diff if
// needed. In practice, most controllers ignore the old object and just
// enqueue the key for reconciliation.
func (h *informerEventHandler) OnUpdate(oldObj, newObj interface{}) {
	oldPod, ok1 := oldObj.(*corev1.Pod)
	newPod, ok2 := newObj.(*corev1.Pod)
	if !ok1 || !ok2 {
		return
	}
	// Only print if something interesting changed (phase, conditions, etc.)
	if oldPod.Status.Phase != newPod.Status.Phase {
		fmt.Printf("[UPDATE] %s/%s  phase=%s → %s\n",
			newPod.Namespace, newPod.Name,
			oldPod.Status.Phase, newPod.Status.Phase)
	}
}

// OnDelete is called when a Pod is deleted. The obj parameter may be a
// *corev1.Pod or a cache.DeletedFinalStateUnknown (if the delete event
// was missed and only discovered during a re-list). Production code
// should handle both cases.
func (h *informerEventHandler) OnDelete(obj interface{}) {
	pod, ok := obj.(*corev1.Pod)
	if !ok {
		return
	}
	fmt.Printf("[DELETE] %s/%s\n", pod.Namespace, pod.Name)
}
```

This program is a complete, runnable informer client. It watches all Pods in
all namespaces and prints events as they occur. Notice that we never poll —
the informer handles list+watch, reconnection, and caching for us.

---

## 32.3 Work Queues

### 32.3.1 Why Not Process Events Directly?

In the previous example, we processed events directly in the event handlers.
This works for printing, but it is a terrible idea for actual controllers.
Here is why:

1. **Event handlers run in the informer's goroutine.** If your handler blocks
   (e.g., making an API call that takes 5 seconds), the informer cannot
   process any other events during that time. This causes the DeltaFIFO to
   back up, which can trigger a re-list, which puts even more load on the
   API server.

2. **No retry mechanism.** If your handler fails, the event is gone. You have
   no way to retry it unless you build your own retry logic.

3. **No deduplication.** If a Pod changes 10 times in one second, your handler
   is called 10 times. But you probably only need to reconcile once — using
   the latest state. Without deduplication, you waste 9 reconciliation cycles.

4. **No rate limiting.** If 1000 events arrive simultaneously, your handler
   tries to process them all at once, potentially overwhelming the API server
   with 1000 concurrent update calls.

The solution is a **work queue**. Event handlers are kept lightweight — they
extract the object key (namespace/name) and enqueue it. A separate goroutine
(or pool of goroutines) dequeues keys and runs the reconciliation logic.

### 32.3.2 Work Queue Properties

The `client-go/util/workqueue` package provides work queues with several
important properties:

**Deduplication.** If the same key is enqueued multiple times before being
processed, it only appears once in the queue. This means if a Pod is updated
10 times in rapid succession, the reconciler only runs once — with the latest
state from the cache.

**Ordering.** Items are processed in FIFO order. No item is processed
concurrently with itself — if key "default/my-pod" is being reconciled, and
someone enqueues "default/my-pod" again, the second reconciliation waits until
the first one finishes.

**Rate limiting.** The rate-limited queue type adds configurable rate limiting
to prevent overwhelming the API server. The default limiter uses a combination
of:
- Per-item exponential backoff (starts at 5ms, doubles each failure, caps at
  1000 seconds)
- Overall rate limit (10 items/second with burst of 100)

**Shutdown.** Queues can be shut down gracefully. New items are rejected, and
workers drain remaining items.

### 32.3.3 Queue Types in client-go

```
workqueue.New()                    → Simple FIFO with deduplication
workqueue.NewRateLimitingQueue()   → FIFO + rate limiting
workqueue.NewDelayingQueue()       → FIFO + delayed enqueue (AddAfter)
```

In practice, you almost always want `NewRateLimitingQueue`. The rate limiter
controls how quickly failed items are retried, and how fast the controller
processes items overall.

### 32.3.4 Hands-On: Work Queue with Rate Limiting

```go
// Package main demonstrates the work queue pattern used by Kubernetes
// controllers. The work queue sits between the informer's event
// handlers and the reconciler, providing deduplication, ordering,
// and rate limiting.
//
// This program:
//  1. Creates a rate-limited work queue.
//  2. Starts a Pod informer that enqueues object keys into the queue.
//  3. Runs a worker loop that dequeues keys and "reconciles" them
//     (in this demo, just prints the pod's current state from cache).
//  4. Demonstrates retry logic — if reconciliation fails, the key is
//     requeued with exponential backoff.
package main

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"
	"time"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/util/runtime"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/workqueue"
	"k8s.io/klog/v2"
)

// maxRetries defines the maximum number of times we will retry
// reconciling a single key before giving up and dropping it from
// the queue. This prevents infinite retry loops for permanently
// broken resources.
const maxRetries = 5

func main() {
	// Build client (same as previous example).
	kubeconfig := os.Getenv("KUBECONFIG")
	if kubeconfig == "" {
		kubeconfig = os.Getenv("HOME") + "/.kube/config"
	}
	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		klog.Fatalf("Error building kubeconfig: %v", err)
	}
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		klog.Fatalf("Error creating clientset: %v", err)
	}

	// ---------- Create the rate-limited work queue ----------
	//
	// workqueue.DefaultControllerRateLimiter returns a rate limiter that
	// combines:
	//   - ItemExponentialFailureRateLimiter: per-item backoff starting
	//     at 5ms, doubling on each failure, capped at 1000s.
	//   - BucketRateLimiter: overall rate limit of 10 qps with burst 100.
	//
	// This combination ensures that:
	//   - Individual failing items are retried with increasing delays.
	//   - The overall reconciliation rate does not overwhelm the API server.
	queue := workqueue.NewRateLimitingQueue(
		workqueue.DefaultControllerRateLimiter(),
	)
	defer queue.ShutDown()

	// ---------- Create informer and register handlers ----------
	factory := informers.NewSharedInformerFactory(clientset, 30*time.Second)
	podInformer := factory.Core().V1().Pods()

	// The event handlers' only job is to compute the object key and
	// enqueue it. They must be fast — no API calls, no heavy computation.
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		// AddFunc is called for new objects. We extract the key
		// (namespace/name) using cache.MetaNamespaceKeyFunc and add
		// it to the work queue.
		AddFunc: func(obj interface{}) {
			key, err := cache.MetaNamespaceKeyFunc(obj)
			if err == nil {
				queue.Add(key)
			}
		},
		// UpdateFunc is called for modified objects. We enqueue the
		// NEW object's key (not the old one — the key does not change,
		// but using newObj is the convention).
		UpdateFunc: func(oldObj, newObj interface{}) {
			key, err := cache.MetaNamespaceKeyFunc(newObj)
			if err == nil {
				queue.Add(key)
			}
		},
		// DeleteFunc is called for deleted objects. We still enqueue
		// the key — the reconciler will notice the object is gone and
		// take appropriate action (e.g., clean up external resources).
		DeleteFunc: func(obj interface{}) {
			key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
			if err == nil {
				queue.Add(key)
			}
		},
	})

	// ---------- Start informer ----------
	stopCh := make(chan struct{})
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		<-sigCh
		close(stopCh)
	}()

	factory.Start(stopCh)

	// CRITICAL: Wait for the cache to sync before starting workers.
	// If we start reconciling before the cache is warm, we might
	// "reconcile" objects that do not actually exist, because the
	// initial list has not completed yet.
	if !cache.WaitForCacheSync(stopCh, podInformer.Informer().HasSynced) {
		klog.Fatal("Timed out waiting for cache sync")
	}
	klog.Info("Cache synced, starting worker")

	// ---------- Run the worker loop ----------
	//
	// In production, you would run multiple workers in parallel:
	//   for i := 0; i < workerCount; i++ {
	//       go worker(queue, podInformer.Lister(), stopCh)
	//   }
	// The work queue ensures that no two workers process the same key
	// concurrently.
	go worker(queue, podInformer.Lister(), stopCh)

	<-stopCh
	klog.Info("Shutting down")
}

// worker is the main processing loop. It continuously dequeues items
// from the work queue and processes them. It runs until stopCh is
// closed or the queue is shut down.
func worker(
	queue workqueue.RateLimitingInterface,
	lister interface{ Pods(string) interface{} },
	stopCh <-chan struct{},
) {
	for {
		// processNextItem blocks until an item is available, then
		// processes it. Returns false when the queue is shut down.
		if !processNextItem(queue) {
			return
		}
	}
}

// processNextItem dequeues a single item from the work queue,
// reconciles it, and handles success/failure. Returns false when
// the queue is shut down (no more items will ever come).
func processNextItem(queue workqueue.RateLimitingInterface) bool {
	// queue.Get() blocks until an item is available or the queue is
	// shut down. The boolean return indicates shutdown.
	key, shutdown := queue.Get()
	if shutdown {
		return false
	}

	// Done() must be called when processing is complete, regardless of
	// success or failure. This tells the queue that the item is no
	// longer being processed, allowing it to be requeued if needed.
	defer queue.Done(key)

	// reconcile is where the actual controller logic lives. In this
	// demo, it just prints the key. In a real controller, it would
	// read the object from the cache, compare desired vs actual state,
	// and take action.
	err := reconcile(key.(string))
	if err == nil {
		// Forget tells the rate limiter to reset the backoff for this
		// key. The item was processed successfully, so next time it is
		// enqueued, it should not be delayed.
		queue.Forget(key)
		return true
	}

	// If reconciliation failed, check if we have exceeded the retry limit.
	if queue.NumRequeues(key) < maxRetries {
		klog.Errorf("Error reconciling %s (retry %d/%d): %v",
			key, queue.NumRequeues(key)+1, maxRetries, err)
		// AddRateLimited re-enqueues the item with a delay determined
		// by the rate limiter (exponential backoff). The item will be
		// retried after the delay.
		queue.AddRateLimited(key)
		return true
	}

	// We have exceeded the retry limit. Give up on this key.
	klog.Errorf("Dropping %s after %d retries: %v", key, maxRetries, err)
	queue.Forget(key)
	runtime.HandleError(fmt.Errorf("dropping %s: %v", key, err))
	return true
}

// reconcile processes a single key. In this demo, it just prints the
// key. A real controller would:
//  1. Split the key into namespace/name.
//  2. Get the object from the informer's cache (lister).
//  3. If the object does not exist in the cache, it was deleted —
//     clean up external resources if needed.
//  4. If the object exists, compare desired state (spec) to actual
//     state and take action to close the gap.
func reconcile(key string) error {
	fmt.Printf("Reconciling: %s\n", key)
	// Real reconciliation logic goes here.
	return nil
}
```

### 32.3.5 The Key Insight: Deduplication + Cache

The work queue and informer cache work together beautifully:

1. A Pod changes 10 times in 1 second.
2. The informer's event handler enqueues the key 10 times.
3. The work queue deduplicates: the key appears only once.
4. The reconciler dequeues the key and reads the **latest** state from the
   informer's cache.
5. Result: one reconciliation with the latest state, not 10 reconciliations
   with intermediate states.

This is why controllers are level-triggered, not edge-triggered. The
reconciler does not care *what* changed — it reads the full current state
and computes the full desired state. The events are merely hints that
"something changed; you should probably reconcile."

---

## 32.4 Leader Election

### 32.4.1 Why Leader Election?

Controllers are typically deployed as Kubernetes Deployments with multiple
replicas for high availability. But only **one** instance should be actively
reconciling at any time. If two instances both try to reconcile the same
resource simultaneously, they can conflict — creating duplicate resources,
issuing contradictory updates, or getting into race conditions.

Leader election solves this. All replicas start up and compete for a "lease."
One wins and becomes the leader. The others wait. If the leader dies (crashes,
network partition), another replica takes over.

### 32.4.2 Lease-Based Leader Election in client-go

Kubernetes provides a built-in mechanism for leader election using **Lease**
objects (in the `coordination.k8s.io/v1` API group). The flow is:

1. Each replica tries to create or update a Lease object in a specific
   namespace.
2. The Lease has an `acquireTime`, `renewTime`, and `leaseDurationSeconds`.
3. The holder must renew the lease before it expires.
4. If the lease expires without renewal, another replica can acquire it.

The `client-go/tools/leaderelection` package handles all of this for you.

### 32.4.3 Hands-On: Implementing Leader Election

```go
// Package main demonstrates lease-based leader election using the
// client-go leaderelection package. In production, this pattern
// ensures that exactly one controller instance is active at a time,
// while standby replicas are ready to take over immediately if the
// leader fails.
//
// The program:
//  1. Creates a Kubernetes client.
//  2. Configures a leader election resource lock (Lease object).
//  3. Runs leader election — when elected, runs the main controller
//     logic; when leadership is lost, shuts down gracefully.
package main

import (
	"context"
	"fmt"
	"os"
	"time"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/client-go/kubernetes"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/tools/leaderelection"
	"k8s.io/client-go/tools/leaderelection/resourcelock"
	"k8s.io/klog/v2"
)

func main() {
	// Build client.
	kubeconfig := os.Getenv("KUBECONFIG")
	if kubeconfig == "" {
		kubeconfig = os.Getenv("HOME") + "/.kube/config"
	}
	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		klog.Fatalf("Error building kubeconfig: %v", err)
	}
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		klog.Fatalf("Error creating clientset: %v", err)
	}

	// ---------- Configure the resource lock ----------
	//
	// The resource lock determines what Kubernetes object is used to
	// coordinate leader election. We use a Lease object, which is the
	// recommended choice since Kubernetes 1.17.
	//
	// - LeaseName: name of the Lease object (must be unique per controller).
	// - LeaseNamespace: namespace where the Lease lives.
	// - Identity: unique identity of this instance (typically the pod name).
	//
	// The identity should be unique across all replicas. Using the pod
	// name (from the HOSTNAME env var or downward API) is the standard
	// approach.
	hostname, _ := os.Hostname()
	lock := &resourcelock.LeaseLock{
		LeaseMeta: metav1.ObjectMeta{
			Name:      "my-controller-leader",
			Namespace: "default",
		},
		Client: clientset.CoordinationV1(),
		LockConfig: resourcelock.ResourceLockConfig{
			Identity: hostname,
		},
	}

	// ---------- Run leader election ----------
	//
	// LeaderElectionConfig controls the timing parameters:
	//
	// - LeaseDuration: how long a lease is valid. If the leader does not
	//   renew within this time, another replica can take over. Too short
	//   = unnecessary failovers. Too long = slow failover.
	//
	// - RenewDeadline: how long the leader will try to renew before
	//   giving up. Must be less than LeaseDuration.
	//
	// - RetryPeriod: how often non-leaders check if the lease is available.
	//
	// Production values: 15s / 10s / 2s are common defaults.
	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
		Lock:            lock,
		LeaseDuration:   15 * time.Second,
		RenewDeadline:   10 * time.Second,
		RetryPeriod:     2 * time.Second,
		ReleaseOnCancel: true,

		Callbacks: leaderelection.LeaderCallbacks{
			// OnStartedLeading is called when this instance becomes the
			// leader. The provided context is cancelled when leadership
			// is lost. Your controller logic should respect this context
			// and shut down gracefully when it is cancelled.
			OnStartedLeading: func(ctx context.Context) {
				klog.Info("Became leader — starting controller")
				runController(ctx)
			},

			// OnStoppedLeading is called when this instance loses
			// leadership. This can happen because:
			//   - The context was cancelled (graceful shutdown).
			//   - The lease renewal failed (network issue, API server down).
			//   - Another instance forcibly acquired the lease.
			//
			// The controller should have already stopped via context
			// cancellation, but this callback is a good place for
			// final cleanup.
			OnStoppedLeading: func() {
				klog.Warning("Lost leadership — shutting down")
				cancel()
			},

			// OnNewLeader is called when the leader identity changes.
			// This is called on ALL instances, not just the leader.
			// Useful for logging or metrics.
			OnNewLeader: func(identity string) {
				if identity == hostname {
					klog.Info("Still the leader")
				} else {
					klog.Infof("New leader elected: %s", identity)
				}
			},
		},
	})
}

// runController is a placeholder for the actual controller logic. In a
// real controller, this would start informers, create work queues, and
// run worker goroutines. The ctx is cancelled when leadership is lost.
func runController(ctx context.Context) {
	ticker := time.NewTicker(5 * time.Second)
	defer ticker.Stop()
	for {
		select {
		case <-ctx.Done():
			fmt.Println("Controller stopped (leadership lost)")
			return
		case <-ticker.C:
			fmt.Println("Controller tick — reconciling...")
		}
	}
}
```

### 32.4.4 Leader Election in Practice

A few important notes:

**Use Lease objects.** Older code uses ConfigMaps or Endpoints for leader
election. Lease objects are purpose-built for this use case and create less
load on the API server.

**Identity matters.** Each replica must have a unique identity. The standard
approach is to use the pod name, which Kubernetes guarantees is unique within
a namespace. Set `HOSTNAME` or use the downward API.

**Timing is critical.** The `LeaseDuration`, `RenewDeadline`, and
`RetryPeriod` control the trade-off between fast failover and stability.
Aggressive timings (short durations) mean faster failover but more risk of
unnecessary leader changes. Conservative timings (long durations) mean stable
leadership but slower failover.

**The controller-runtime Manager handles this for you.** If you use
controller-runtime (section 32.6), leader election is built in — you just
set `LeaderElection: true` in the Manager options.

---

## 32.5 Building a Controller from Scratch (No Frameworks)

Now let us put it all together. We will build a complete controller using
nothing but `client-go` — no controller-runtime, no kubebuilder, no frameworks.
This is how controllers were built before the ecosystem matured, and
understanding this foundation is essential.

### 32.5.1 The Pod Monitor Controller

Our controller will:
- Watch all Pods in the cluster.
- Log state changes (phase transitions).
- Implement a rate-limited work queue with retry logic.
- Handle errors and requeue failed items.
- Shut down gracefully on SIGINT/SIGTERM.

This is a "read-only" controller — it does not modify any resources. But the
pattern is identical to a controller that creates, updates, or deletes
resources. The only difference is what happens inside the `reconcile` function.

```go
// Package main implements a complete Kubernetes controller from scratch
// using only the client-go library. This controller watches Pods and
// logs state transitions (phase changes), demonstrating the full
// controller pattern:
//
//	Informer → Event Handlers → Work Queue → Worker → Reconciler
//
// No frameworks (controller-runtime, kubebuilder) are used. Every
// component is wired manually so you can see exactly how the pieces
// fit together.
//
// Components:
//   - SharedInformerFactory: manages Pod informer lifecycle.
//   - Pod informer: watches Pods via list+watch, maintains local cache.
//   - Rate-limited work queue: deduplicates and rate-limits reconciliation.
//   - Worker goroutine(s): dequeue keys and call the reconciler.
//   - Reconciler: reads Pod from cache, logs phase, handles errors.
package main

import (
	"fmt"
	"os"
	"os/signal"
	"syscall"
	"time"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/util/runtime"
	"k8s.io/apimachinery/pkg/util/wait"
	"k8s.io/client-go/informers"
	"k8s.io/client-go/kubernetes"
	corelister "k8s.io/client-go/listers/core/v1"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/workqueue"
	"k8s.io/klog/v2"
)

// PodMonitorController watches Pods and logs phase transitions. It
// demonstrates the complete controller pattern:
//
//  1. An informer watches Pods and calls event handlers.
//  2. Event handlers extract the object key and enqueue it.
//  3. A worker loop dequeues keys and calls the reconciler.
//  4. The reconciler reads the Pod from the informer cache and logs
//     any interesting state.
//  5. If reconciliation fails, the key is requeued with backoff.
//
// This struct holds all the components needed by the controller. In
// production, you would also add a Kubernetes clientset for making
// API calls (create/update/delete) during reconciliation.
type PodMonitorController struct {
	// podLister provides read access to the informer's local cache
	// of Pods. This is much faster than calling the API server and
	// reduces API server load. The lister is namespace-scoped, so
	// you can call podLister.Pods("default").Get("my-pod").
	podLister corelister.PodLister

	// podsSynced is a function that returns true when the Pod
	// informer's cache has completed its initial list and is
	// ready to serve data. Workers must not start until this
	// returns true.
	podsSynced cache.InformerSynced

	// workqueue is a rate-limited queue that deduplicates keys
	// and controls the rate at which reconciliation happens.
	// Keys are strings in the format "namespace/name".
	workqueue workqueue.RateLimitingInterface
}

// maxRetries is the maximum number of times we will retry reconciling
// a single key before giving up. After this many failures, we assume
// something is fundamentally wrong and drop the key.
const maxRetries = 10

// NewPodMonitorController creates a new controller and wires up the
// informer event handlers to the work queue.
//
// Parameters:
//   - clientset: Kubernetes API client (not used in this read-only
//     controller, but included to show the standard pattern).
//   - factory: SharedInformerFactory that provides the Pod informer.
//
// Returns:
//   - A fully configured controller ready to be started with Run().
func NewPodMonitorController(
	clientset kubernetes.Interface,
	factory informers.SharedInformerFactory,
) *PodMonitorController {
	podInformer := factory.Core().V1().Pods()

	c := &PodMonitorController{
		podLister:  podInformer.Lister(),
		podsSynced: podInformer.Informer().HasSynced,
		workqueue: workqueue.NewRateLimitingQueue(
			workqueue.DefaultControllerRateLimiter(),
		),
	}

	// Register event handlers. These are called by the informer
	// whenever a Pod is added, updated, or deleted. The handlers
	// are intentionally minimal — they extract the object key and
	// enqueue it, nothing more.
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			c.enqueue(obj)
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			c.enqueue(newObj)
		},
		DeleteFunc: func(obj interface{}) {
			c.enqueue(obj)
		},
	})

	return c
}

// enqueue extracts the key (namespace/name) from a Kubernetes object
// and adds it to the work queue. It handles the special case of
// DeletedFinalStateUnknown, which occurs when the informer missed the
// delete event and only discovered the deletion during a re-list.
func (c *PodMonitorController) enqueue(obj interface{}) {
	key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
	if err != nil {
		runtime.HandleError(fmt.Errorf("extracting key: %v", err))
		return
	}
	c.workqueue.Add(key)
}

// Run starts the controller. It:
//  1. Waits for the informer cache to sync (initial list complete).
//  2. Starts the specified number of worker goroutines.
//  3. Blocks until stopCh is closed (typically from a signal handler).
//  4. Shuts down the work queue, which causes workers to exit.
//
// Parameters:
//   - workers: number of concurrent worker goroutines. More workers
//     means higher throughput but also higher API server load.
//   - stopCh: channel that signals the controller to shut down.
func (c *PodMonitorController) Run(workers int, stopCh <-chan struct{}) {
	defer runtime.HandleCrash()
	defer c.workqueue.ShutDown()

	klog.Info("Starting PodMonitor controller")

	// Wait for the informer caches to sync. This is CRITICAL.
	// If we start processing before the cache is warm, we might
	// see "stale" data or phantom deletes.
	klog.Info("Waiting for informer caches to sync...")
	if ok := cache.WaitForCacheSync(stopCh, c.podsSynced); !ok {
		klog.Fatal("Failed to sync caches")
	}
	klog.Info("Caches synced — starting workers")

	// Start worker goroutines. wait.Until runs the given function
	// repeatedly with the given period until stopCh is closed.
	// If the function panics, wait.Until recovers and restarts it.
	for i := 0; i < workers; i++ {
		go wait.Until(c.runWorker, time.Second, stopCh)
	}

	klog.Infof("Started %d workers", workers)
	<-stopCh
	klog.Info("Shutting down workers")
}

// runWorker is the main loop for a single worker goroutine. It
// continuously calls processNextItem until the queue is shut down.
func (c *PodMonitorController) runWorker() {
	for c.processNextItem() {
	}
}

// processNextItem dequeues a single key from the work queue,
// reconciles it, and handles success/failure.
//
// Returns false when the queue is shut down (signaling the worker
// to exit).
//
// The error handling flow:
//   - Success: forget the key (reset backoff), mark Done.
//   - Failure (retries remaining): requeue with rate limiting.
//   - Failure (max retries exceeded): forget and drop the key.
func (c *PodMonitorController) processNextItem() bool {
	obj, shutdown := c.workqueue.Get()
	if shutdown {
		return false
	}
	defer c.workqueue.Done(obj)

	key, ok := obj.(string)
	if !ok {
		// If the item in the queue is not a string, it is invalid.
		// Forget it so we do not loop on a bad item.
		c.workqueue.Forget(obj)
		runtime.HandleError(fmt.Errorf("expected string, got %T", obj))
		return true
	}

	// Run the reconciler.
	if err := c.reconcile(key); err != nil {
		if c.workqueue.NumRequeues(key) < maxRetries {
			klog.Warningf("Reconcile failed for %s (attempt %d/%d): %v",
				key, c.workqueue.NumRequeues(key)+1, maxRetries, err)
			c.workqueue.AddRateLimited(key)
			return true
		}
		klog.Errorf("Dropping key %s after %d retries: %v",
			key, maxRetries, err)
		c.workqueue.Forget(key)
		runtime.HandleError(err)
		return true
	}

	// Success — reset the rate limiter for this key.
	c.workqueue.Forget(key)
	return true
}

// reconcile is the heart of the controller. It processes a single key
// (namespace/name) by reading the Pod from the informer cache and
// logging its current state.
//
// In a real controller, this function would:
//  1. Split the key into namespace and name.
//  2. Get the object from the lister (local cache).
//  3. If not found, the object was deleted — clean up external state.
//  4. If found, compare desired state to actual state and take action
//     (create, update, or delete dependent resources).
//  5. Update the object's status subresource to reflect current state.
//
// IMPORTANT: This function must be idempotent. It may be called
// multiple times for the same key, and must produce the same result
// each time.
func (c *PodMonitorController) reconcile(key string) error {
	// Split the key into namespace and name.
	namespace, name, err := cache.SplitMetaNamespaceKey(key)
	if err != nil {
		return fmt.Errorf("invalid key %q: %v", key, err)
	}

	// Read the Pod from the informer's local cache. This does NOT
	// make an API call — it reads from the in-memory store that the
	// informer maintains.
	pod, err := c.podLister.Pods(namespace).Get(name)
	if err != nil {
		if errors.IsNotFound(err) {
			// The Pod was deleted between the time it was enqueued
			// and now. This is normal and expected. If we had
			// external resources to clean up, we would do it here.
			klog.Infof("Pod %s deleted (no longer in cache)", key)
			return nil
		}
		return fmt.Errorf("getting pod %s: %v", key, err)
	}

	// ---------- Reconciliation logic ----------
	//
	// For this demo, we just log the Pod's current state. A real
	// controller would compare spec to status and take action.
	klog.Infof("Reconcile: %s/%s  phase=%s  conditions=%d  containers=%d",
		pod.Namespace,
		pod.Name,
		pod.Status.Phase,
		len(pod.Status.Conditions),
		len(pod.Spec.Containers),
	)

	return nil
}

func main() {
	// Build Kubernetes client.
	kubeconfig := os.Getenv("KUBECONFIG")
	if kubeconfig == "" {
		kubeconfig = os.Getenv("HOME") + "/.kube/config"
	}
	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		klog.Fatalf("Error building kubeconfig: %v", err)
	}
	clientset, err := kubernetes.NewForConfig(config)
	if err != nil {
		klog.Fatalf("Error creating clientset: %v", err)
	}

	// Create informer factory and controller.
	factory := informers.NewSharedInformerFactory(clientset, 10*time.Minute)
	controller := NewPodMonitorController(clientset, factory)

	// Start informers.
	stopCh := make(chan struct{})
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	go func() {
		<-sigCh
		klog.Info("Received shutdown signal")
		close(stopCh)
	}()

	factory.Start(stopCh)

	// Run controller with 2 workers.
	controller.Run(2, stopCh)
}
```

### 32.5.2 Code Walkthrough

Let us trace the complete data flow:

```
API Server
    │
    ▼  (list + watch)
Reflector
    │
    ▼  (push events)
DeltaFIFO
    │
    ▼  (pop events, update cache)
Indexer / Store  ◀──── podLister reads from here
    │
    ▼  (call event handlers)
AddFunc / UpdateFunc / DeleteFunc
    │
    ▼  (extract key, enqueue)
Work Queue (rate-limited)
    │
    ▼  (dequeue)
Worker Goroutine
    │
    ▼  (call reconciler)
reconcile(key)
    │
    ├── Get Pod from lister (cache read)
    ├── If not found → deleted, clean up
    ├── If found → compare desired vs actual
    └── Log / Create / Update / Delete as needed
```

Key design decisions in this controller:

1. **Informer, not direct API calls.** We never call `clientset.CoreV1().Pods()
   .List()` directly. The informer handles list+watch, and the lister provides
   read access to the local cache.

2. **Work queue, not direct processing.** Event handlers are one-liners that
   enqueue keys. All heavy work happens in the reconciler.

3. **Rate limiting.** The work queue uses exponential backoff for failed items
   and an overall rate limit.

4. **Idempotent reconciler.** The reconcile function reads the full current
   state and makes decisions based on that. It does not depend on the event
   type (add/update/delete) — it reads the current state and reconciles.

5. **Graceful shutdown.** SIGINT/SIGTERM close the stop channel, which causes
   informers and workers to shut down cleanly.

---

## 32.6 Controller-Runtime — The Framework

### 32.6.1 Why Controller-Runtime Exists

The controller we built in section 32.5 is ~200 lines of boilerplate before
we even get to the reconciliation logic. Every controller needs the same
pieces: informer setup, work queue creation, worker loop, error handling,
cache sync waiting, signal handling, etc.

**controller-runtime** is a framework that provides all of this boilerplate.
It was created by the Kubernetes SIG API Machinery team and is the foundation
of [Kubebuilder](https://book.kubebuilder.io/), the most popular tool for
building Kubernetes operators.

With controller-runtime, the same Pod monitor controller is roughly 50 lines
of actual logic. The framework handles:

- Informer creation and lifecycle
- Work queue management
- Leader election
- Metrics and health check endpoints
- Signal handling and graceful shutdown
- Cache synchronization
- Event filtering (predicates)

### 32.6.2 Core Concepts

**Manager.** The Manager is the top-level component. It:
- Creates and manages a shared cache (informers).
- Runs one or more Controllers.
- Handles leader election.
- Serves metrics (Prometheus) and health check endpoints.
- Manages graceful shutdown.

You create one Manager per process.

**Controller.** A Controller watches one or more resource types and enqueues
reconcile requests when events occur. Under the hood, it uses informers and
work queues — the same components we built manually in section 32.5.

**Reconciler.** The Reconciler interface has a single method:

```go
Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error)
```

- `ctx` carries cancellation and deadline information.
- `req` contains the namespace and name of the object to reconcile.
- Returns `(Result, error)`:
  - `Result{}` + `nil` error = success, do not requeue.
  - `Result{Requeue: true}` = success, but requeue immediately.
  - `Result{RequeueAfter: 5*time.Minute}` = requeue after a delay.
  - Any non-nil error = failure, requeue with backoff.

**Predicate.** Predicates filter which events trigger reconciliation. For
example, you might only want to reconcile when a Pod's phase changes, not
on every annotation update. Predicates are composable:

```go
predicate.GenerationChangedPredicate{} // Only spec changes (not status)
predicate.Or(pred1, pred2)             // Either predicate matches
predicate.And(pred1, pred2)            // Both predicates match
predicate.Not(pred)                    // Invert a predicate

```

### 32.6.3 Hands-On: Pod Monitor with Controller-Runtime

Here is the same Pod monitor controller, rewritten with controller-runtime.
Compare this to the 250+ lines in section 32.5.

```go
// Package main implements the same Pod monitor controller from
// section 32.5, but using the controller-runtime framework instead
// of raw client-go. Notice how much boilerplate disappears:
//
//   - No manual informer setup.
//   - No work queue creation or management.
//   - No worker loop.
//   - No signal handling.
//   - No cache sync waiting.
//
// The framework handles all of this. Our job is to implement the
// Reconcile method — the actual business logic.
package main

import (
	"context"
	"os"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/log/zap"
)

// PodMonitorReconciler implements the reconcile.Reconciler interface.
// It is the only component we need to write — everything else is
// handled by the framework.
//
// Fields:
//   - client.Client: provides read/write access to Kubernetes resources.
//     Reads go through the informer cache (fast, no API call). Writes
//     go directly to the API server.
type PodMonitorReconciler struct {
	client.Client
}

// Reconcile is called by the controller-runtime framework whenever a
// Pod event is observed (add, update, delete). The framework handles
// deduplication, rate limiting, and retry — we just implement the
// business logic.
//
// Parameters:
//   - ctx: carries cancellation, deadline, and logger information.
//   - req: contains the namespace and name of the Pod to reconcile.
//     Note that the req does NOT contain the Pod object itself — you
//     must read it from the cache using client.Get().
//
// Returns:
//   - ctrl.Result: controls requeueing behavior.
//   - error: if non-nil, the framework requeues with backoff.
func (r *PodMonitorReconciler) Reconcile(
	ctx context.Context,
	req ctrl.Request,
) (ctrl.Result, error) {
	logger := log.FromContext(ctx)

	// Read the Pod from the informer cache. This is a local cache
	// read, not an API server call. The cache is kept up to date
	// by the informer that the framework manages for us.
	var pod corev1.Pod
	if err := r.Get(ctx, req.NamespacedName, &pod); err != nil {
		if errors.IsNotFound(err) {
			// Pod was deleted. In a real controller, we would clean
			// up external resources here.
			logger.Info("Pod deleted", "pod", req.NamespacedName)
			return ctrl.Result{}, nil
		}
		// Unexpected error — requeue with backoff.
		return ctrl.Result{}, err
	}

	// ---------- Business logic ----------
	//
	// Log the Pod's current state. A real controller would compare
	// desired state (spec) to actual state (status) and take action.
	logger.Info("Reconciling Pod",
		"pod", req.NamespacedName,
		"phase", pod.Status.Phase,
		"conditions", len(pod.Status.Conditions),
		"containers", len(pod.Spec.Containers),
	)

	// Result{} with nil error means "success, do not requeue."
	return ctrl.Result{}, nil
}

func main() {
	// Configure logging. Zap is the default logger for controller-runtime.
	ctrl.SetLogger(zap.New(zap.UseDevMode(true)))

	// ---------- Create the Manager ----------
	//
	// The Manager is the top-level component. It:
	//   - Creates and manages a shared cache (informers).
	//   - Handles leader election (if enabled).
	//   - Serves metrics and health check endpoints.
	//   - Manages graceful shutdown.
	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
		// LeaderElection: true,
		// LeaderElectionID: "pod-monitor-leader",
	})
	if err != nil {
		ctrl.Log.Error(err, "unable to create manager")
		os.Exit(1)
	}

	// ---------- Register the controller ----------
	//
	// ctrl.NewControllerManagedBy(mgr) returns a builder that lets us
	// configure what the controller watches and how it reconciles.
	//
	// For(&corev1.Pod{}) means: "watch Pods and reconcile when they
	// change." Under the hood, the framework creates a Pod informer,
	// registers event handlers, and wires up a work queue — exactly
	// what we did manually in section 32.5.
	err = ctrl.NewControllerManagedBy(mgr).
		For(&corev1.Pod{}).
		Complete(&PodMonitorReconciler{
			Client: mgr.GetClient(),
		})
	if err != nil {
		ctrl.Log.Error(err, "unable to create controller")
		os.Exit(1)
	}

	// ---------- Start the Manager ----------
	//
	// mgr.Start() blocks until the process receives a shutdown signal
	// (SIGINT, SIGTERM). It:
	//   1. Starts all informers and waits for cache sync.
	//   2. Starts all registered controllers.
	//   3. Starts metrics and health check servers.
	//   4. Runs leader election (if enabled).
	//   5. Blocks until shutdown.
	ctrl.Log.Info("Starting manager")
	if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
		ctrl.Log.Error(err, "manager exited with error")
		os.Exit(1)
	}
}
```

### 32.6.4 Comparison: Raw client-go vs Controller-Runtime

| Aspect                 | Raw client-go (§32.5) | Controller-Runtime (§32.6) |
|------------------------|-----------------------|----------------------------|
| Lines of code          | ~250                  | ~80                        |
| Informer setup         | Manual                | Automatic (from `For()`)   |
| Work queue             | Manual creation       | Automatic                  |
| Worker loop            | Manual goroutines     | Automatic                  |
| Cache sync             | Manual `WaitForCacheSync` | Automatic               |
| Signal handling        | Manual `os/signal`    | `ctrl.SetupSignalHandler()`|
| Leader election        | Manual (§32.4)        | One config flag            |
| Metrics/health         | Not included          | Built-in                   |
| Error handling/retry   | Manual requeue logic  | Return error → auto requeue|
| Multi-resource watches | Manual event handlers | `Owns()`, `Watches()`      |

The trade-off: controller-runtime is less flexible. If you need very fine-
grained control over informer configuration, queue behavior, or worker
management, raw client-go gives you more options. But for the vast majority
of controllers, controller-runtime is the right choice.

---

## 32.7 Controller Best Practices

### 32.7.1 Always Reconcile Full State

Do not try to compute incremental diffs in your reconciler. Always read the
full desired state and the full actual state, compute the diff, and apply it.
This is the level-triggered principle in action.

```go
// BAD: Incremental approach — fragile
func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	// "What changed?" — This is edge-triggered thinking.
	// If we miss an event, we will never converge.
}

// GOOD: Full reconciliation — robust
func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	// Read the full desired state (spec).
	// Read the full actual state (list dependent resources).
	// Compute the diff.
	// Apply changes to close the gap.
	// This always converges, regardless of event history.
}
```

### 32.7.2 Use the Status Subresource

The status subresource allows you to update the `.status` field of a resource
without modifying the `.spec`. This is important because:

1. **Spec changes trigger reconciliation.** If your controller updates both
   spec and status in one call, it triggers another reconciliation, creating
   an infinite loop.

2. **Separation of concerns.** Users write spec. Controllers write status.
   The status subresource enforces this boundary.

```go
// Update status using the status subresource.
// This does NOT trigger another reconciliation because it only
// modifies .status, not .spec or .metadata.
myObj.Status.Phase = "Running"
myObj.Status.Conditions = append(myObj.Status.Conditions, condition)
if err := r.Status().Update(ctx, myObj); err != nil {
	return ctrl.Result{}, err
}

```

### 32.7.3 Implement Exponential Backoff

If reconciliation fails, do not retry immediately in a tight loop. Use
exponential backoff to avoid overwhelming the system. Controller-runtime
handles this automatically when you return an error from `Reconcile()`.
With raw client-go, use the rate-limited work queue.

### 32.7.4 Use Finalizers for Cleanup

If your controller creates external resources (cloud instances, DNS records,
database entries), you need finalizers to ensure those resources are cleaned
up when the Kubernetes object is deleted. We cover finalizers in detail in
Chapter 35.

### 32.7.5 Set Owner References

When your controller creates child resources (e.g., a Deployment controller
creates ReplicaSets), set the `ownerReferences` field on the child. This:

1. Enables automatic garbage collection — when the parent is deleted, the
   children are automatically deleted.
2. Enables the controller-runtime `Owns()` mechanism — events on children
   are mapped back to the parent for reconciliation.

```go
// Set owner reference so the child Pod is garbage-collected
// when the parent CustomResource is deleted.
if err := ctrl.SetControllerReference(parent, childPod, r.Scheme()); err != nil {
	return ctrl.Result{}, err
}

```

### 32.7.6 Do Not Block in Reconcile

The reconcile function should be fast. If you need to wait for an external
operation (cloud API call, long-running computation), start the operation and
requeue:

```go
func (r *Reconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	// Check if the async operation is complete.
	if !isOperationComplete() {
		// Not done yet — requeue after 30 seconds.
		return ctrl.Result{RequeueAfter: 30 * time.Second}, nil
	}
	// Operation complete — continue reconciliation.
	return ctrl.Result{}, nil
}
```

### 32.7.7 Test Your Controllers

- **Unit tests:** Use the `fake` client from controller-runtime to test
  reconciliation logic without a real cluster.
- **Integration tests:** Use `envtest` from controller-runtime to spin up a
  real API server and etcd in-process. This tests the full controller loop
  including informers and work queues.
- **End-to-end tests:** Deploy to a real cluster (kind, minikube) and test
  the full lifecycle.

```go
// Example: Unit test with fake client
func TestReconcile_PodNotFound(t *testing.T) {
	// Create a fake client with no objects.
	fakeClient := fake.NewClientBuilder().Build()

	reconciler := &PodMonitorReconciler{Client: fakeClient}
	result, err := reconciler.Reconcile(context.Background(), ctrl.Request{
		NamespacedName: types.NamespacedName{
			Namespace: "default",
			Name:      "nonexistent-pod",
		},
	})

	// Should succeed (pod deleted is not an error).
	assert.NoError(t, err)
	assert.Equal(t, ctrl.Result{}, result)
}
```

---

## 32.8 Hands-On: Building a Real Controller — ConfigMap Syncer

Now let us build a controller that does real work. The **ConfigMap Syncer**
watches ConfigMaps with the label `sync: "true"` and copies them to all
namespaces in the cluster. This is a common pattern for distributing shared
configuration (CA certificates, common environment variables, etc.).

### 32.8.1 Design

```
┌────────────────────────────────────────────────────────────┐
│              CONFIGMAP SYNCER ARCHITECTURE                  │
│                                                            │
│  Source ConfigMap (labeled sync=true)                       │
│  ┌─────────────────────┐                                   │
│  │ Namespace: config   │                                   │
│  │ Name: shared-ca     │                                   │
│  │ Labels:             │                                   │
│  │   sync: "true"      │                                   │
│  │ Data:               │                                   │
│  │   ca.crt: "..."     │                                   │
│  └─────────┬───────────┘                                   │
│            │ Controller copies to all namespaces            │
│            │                                                │
│            ├──▶  default/shared-ca                          │
│            ├──▶  kube-system/shared-ca                      │
│            ├──▶  monitoring/shared-ca                       │
│            └──▶  app-team/shared-ca                         │
│                                                            │
│  If the source is updated → copies are updated.            │
│  If the source is deleted → copies are deleted.            │
│  If a new namespace is created → copy is created there.    │
└────────────────────────────────────────────────────────────┘
```

### 32.8.2 Full Controller Code

```go
// Package main implements the ConfigMap Syncer controller. This
// controller watches ConfigMaps with the label sync="true" and
// copies them to every namespace in the cluster.
//
// Use case: distributing shared configuration (CA certificates,
// common environment variables, feature flags) to all namespaces
// without manually creating ConfigMaps in each one.
//
// Key behaviors:
//   - When a labeled ConfigMap is created or updated, copies are
//     created/updated in all namespaces.
//   - When a labeled ConfigMap is deleted, all copies are deleted.
//   - When a new namespace is created, existing labeled ConfigMaps
//     are copied into it.
//   - Copies are owned by the source ConfigMap (via owner annotations,
//     since cross-namespace ownerReferences are not allowed).
//   - Copies have a label indicating they are managed by this controller,
//     so they can be identified and cleaned up.
//
// Architecture:
//   - Uses controller-runtime for informer/queue/lifecycle management.
//   - Watches both ConfigMaps (for source changes) and Namespaces
//     (for new namespace creation).
//   - Reconciles on source ConfigMap key (namespace/name).
package main

import (
	"context"
	"os"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/apimachinery/pkg/types"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/builder"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/handler"
	"sigs.k8s.io/controller-runtime/pkg/log"
	"sigs.k8s.io/controller-runtime/pkg/log/zap"
	"sigs.k8s.io/controller-runtime/pkg/predicate"
	"sigs.k8s.io/controller-runtime/pkg/reconcile"
)

const (
	// syncLabel is the label that marks a ConfigMap for syncing.
	// Only ConfigMaps with this label set to "true" are synced.
	syncLabel = "configmap-syncer/sync"

	// managedByLabel marks copies created by this controller.
	// This allows us to identify and clean up managed copies.
	managedByLabel = "configmap-syncer/managed-by"

	// sourceAnnotation records the source ConfigMap's namespace/name
	// on each copy. Since cross-namespace ownerReferences are not
	// allowed in Kubernetes, we use annotations to track lineage.
	sourceAnnotation = "configmap-syncer/source"
)

// ConfigMapSyncReconciler reconciles ConfigMaps labeled for syncing.
// It copies labeled ConfigMaps to all namespaces and keeps the copies
// in sync with the source.
type ConfigMapSyncReconciler struct {
	client.Client
}

// Reconcile handles a single ConfigMap reconciliation. It is called
// whenever a source ConfigMap or any namespace changes.
//
// The reconciliation logic:
//  1. Read the source ConfigMap.
//  2. If deleted: list and delete all managed copies.
//  3. If exists: list all namespaces, ensure a copy exists in each.
//  4. For each namespace: create or update the copy to match source.
func (r *ConfigMapSyncReconciler) Reconcile(
	ctx context.Context,
	req ctrl.Request,
) (ctrl.Result, error) {
	logger := log.FromContext(ctx)

	// Step 1: Read the source ConfigMap from the cache.
	var source corev1.ConfigMap
	if err := r.Get(ctx, req.NamespacedName, &source); err != nil {
		if errors.IsNotFound(err) {
			// Source was deleted — clean up all copies.
			logger.Info("Source ConfigMap deleted, cleaning up copies",
				"source", req.NamespacedName)
			return r.cleanupCopies(ctx, req.NamespacedName)
		}
		return ctrl.Result{}, err
	}

	// Verify the sync label is still present. If someone removes the
	// label, we should clean up the copies.
	if source.Labels[syncLabel] != "true" {
		logger.Info("Sync label removed, cleaning up copies",
			"source", req.NamespacedName)
		return r.cleanupCopies(ctx, req.NamespacedName)
	}

	// Step 2: List all namespaces.
	var namespaceList corev1.NamespaceList
	if err := r.List(ctx, &namespaceList); err != nil {
		return ctrl.Result{}, err
	}

	// Step 3: For each namespace (except the source namespace), ensure
	// a copy exists and is up to date.
	for i := range namespaceList.Items {
		ns := &namespaceList.Items[i]

		// Skip the source namespace — do not copy to itself.
		if ns.Name == source.Namespace {
			continue
		}

		// Skip terminating namespaces — Kubernetes will reject writes.
		if ns.Status.Phase == corev1.NamespaceTerminating {
			continue
		}

		// Ensure the copy exists and matches the source.
		if err := r.ensureCopy(ctx, &source, ns.Name); err != nil {
			logger.Error(err, "Failed to sync copy",
				"source", req.NamespacedName,
				"targetNamespace", ns.Name)
			// Continue syncing to other namespaces rather than
			// failing the entire reconciliation. We will retry
			// on the next reconciliation.
		}
	}

	logger.Info("Synced ConfigMap to all namespaces",
		"source", req.NamespacedName,
		"namespaces", len(namespaceList.Items)-1)

	return ctrl.Result{}, nil
}

// ensureCopy creates or updates a copy of the source ConfigMap in the
// target namespace. It uses CreateOrUpdate for idempotency — if the
// copy already exists and matches, no API call is made.
//
// The copy has:
//   - The same name as the source.
//   - The same data as the source.
//   - A managed-by label for identification.
//   - A source annotation for lineage tracking.
func (r *ConfigMapSyncReconciler) ensureCopy(
	ctx context.Context,
	source *corev1.ConfigMap,
	targetNamespace string,
) error {
	// Build the desired copy object.
	copy := &corev1.ConfigMap{
		ObjectMeta: metav1.ObjectMeta{
			Name:      source.Name,
			Namespace: targetNamespace,
		},
	}

	// CreateOrUpdate is an idempotent operation. It:
	//   1. Tries to get the existing object.
	//   2. If not found, creates it.
	//   3. If found, calls the mutate function and updates it.
	//
	// The mutate function sets the desired state on the object.
	// CreateOrUpdate only calls Update if the object actually changed.
	_, err := controllerutil.CreateOrUpdate(ctx, r.Client, copy,
		func() error {
			// Set labels.
			if copy.Labels == nil {
				copy.Labels = make(map[string]string)
			}
			copy.Labels[managedByLabel] = "configmap-syncer"

			// Set annotations.
			if copy.Annotations == nil {
				copy.Annotations = make(map[string]string)
			}
			copy.Annotations[sourceAnnotation] =
				source.Namespace + "/" + source.Name

			// Copy the data.
			copy.Data = source.Data
			copy.BinaryData = source.BinaryData

			return nil
		},
	)

	return err
}

// cleanupCopies deletes all ConfigMaps managed by this controller
// that were copied from the specified source.
//
// We identify copies by:
//  1. The managed-by label (configmap-syncer/managed-by=configmap-syncer).
//  2. The source annotation (configmap-syncer/source=namespace/name).
func (r *ConfigMapSyncReconciler) cleanupCopies(
	ctx context.Context,
	source types.NamespacedName,
) (ctrl.Result, error) {
	logger := log.FromContext(ctx)

	// List all ConfigMaps with the managed-by label.
	var copies corev1.ConfigMapList
	err := r.List(ctx, &copies, &client.ListOptions{
		LabelSelector: labels.SelectorFromSet(labels.Set{
			managedByLabel: "configmap-syncer",
		}),
	})
	if err != nil {
		return ctrl.Result{}, err
	}

	// Delete copies that match the source annotation.
	sourceKey := source.Namespace + "/" + source.Name
	for i := range copies.Items {
		cm := &copies.Items[i]
		if cm.Annotations[sourceAnnotation] == sourceKey {
			logger.Info("Deleting copy",
				"namespace", cm.Namespace, "name", cm.Name)
			if err := r.Delete(ctx, cm); err != nil && !errors.IsNotFound(err) {
				return ctrl.Result{}, err
			}
		}
	}

	return ctrl.Result{}, nil
}

// namespaceToConfigMaps maps Namespace events to reconcile requests
// for all source ConfigMaps. When a new namespace is created, we need
// to reconcile all source ConfigMaps so copies are created in the
// new namespace.
func (r *ConfigMapSyncReconciler) namespaceToConfigMaps(
	ctx context.Context,
	obj client.Object,
) []reconcile.Request {
	// List all source ConfigMaps (those with the sync label).
	var sources corev1.ConfigMapList
	err := r.List(ctx, &sources, &client.ListOptions{
		LabelSelector: labels.SelectorFromSet(labels.Set{
			syncLabel: "true",
		}),
	})
	if err != nil {
		return nil
	}

	// Enqueue a reconcile request for each source ConfigMap.
	requests := make([]reconcile.Request, len(sources.Items))
	for i, cm := range sources.Items {
		requests[i] = reconcile.Request{
			NamespacedName: types.NamespacedName{
				Namespace: cm.Namespace,
				Name:      cm.Name,
			},
		}
	}

	return requests
}

func main() {
	ctrl.SetLogger(zap.New(zap.UseDevMode(true)))

	mgr, err := ctrl.NewManager(ctrl.GetConfigOrDie(), ctrl.Options{
		LeaderElection:   true,
		LeaderElectionID: "configmap-syncer-leader",
	})
	if err != nil {
		ctrl.Log.Error(err, "unable to create manager")
		os.Exit(1)
	}

	reconciler := &ConfigMapSyncReconciler{Client: mgr.GetClient()}

	// Build the controller.
	//
	// For(&corev1.ConfigMap{}): watch ConfigMaps as the primary resource.
	//   Only ConfigMaps with the sync label trigger reconciliation
	//   (filtered by the predicate).
	//
	// Watches(&corev1.Namespace{}): watch Namespaces as a secondary
	//   resource. When a namespace is created, we map the event to
	//   reconcile requests for all source ConfigMaps.
	err = ctrl.NewControllerManagedBy(mgr).
		For(&corev1.ConfigMap{},
			builder.WithPredicates(predicate.NewPredicateFuncs(
				func(obj client.Object) bool {
					// Only reconcile ConfigMaps with sync=true label.
					return obj.GetLabels()[syncLabel] == "true"
				},
			)),
		).
		Watches(
			&corev1.Namespace{},
			handler.EnqueueRequestsFromMapFunc(reconciler.namespaceToConfigMaps),
		).
		Complete(reconciler)
	if err != nil {
		ctrl.Log.Error(err, "unable to create controller")
		os.Exit(1)
	}

	ctrl.Log.Info("Starting ConfigMap Syncer")
	if err := mgr.Start(ctrl.SetupSignalHandler()); err != nil {
		ctrl.Log.Error(err, "manager exited with error")
		os.Exit(1)
	}
}
```

### 32.8.3 Unit Test

```go
// Package main_test demonstrates unit testing a controller-runtime
// reconciler using the fake client. The fake client simulates the
// Kubernetes API server in memory, allowing us to test reconciliation
// logic without a real cluster.
package main_test

import (
	"context"
	"testing"

	corev1 "k8s.io/api/core/v1"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/types"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client/fake"
)

// TestReconcile_CreatescopiesInAllNamespaces verifies that when a
// source ConfigMap with the sync label exists, the reconciler creates
// copies in all other namespaces.
func TestReconcile_CreatesCopiesInAllNamespaces(t *testing.T) {
	// Create a fake client with:
	//   - A source ConfigMap in "config" namespace.
	//   - Three namespaces: config, default, app-team.
	source := &corev1.ConfigMap{
		ObjectMeta: metav1.ObjectMeta{
			Name:      "shared-config",
			Namespace: "config",
			Labels: map[string]string{
				syncLabel: "true",
			},
		},
		Data: map[string]string{
			"key": "value",
		},
	}

	namespaces := []corev1.Namespace{
		{ObjectMeta: metav1.ObjectMeta{Name: "config"}},
		{ObjectMeta: metav1.ObjectMeta{Name: "default"}},
		{ObjectMeta: metav1.ObjectMeta{Name: "app-team"}},
	}

	clientBuilder := fake.NewClientBuilder().
		WithObjects(source, &namespaces[0], &namespaces[1], &namespaces[2])
	fakeClient := clientBuilder.Build()

	reconciler := &ConfigMapSyncReconciler{Client: fakeClient}

	// Reconcile the source ConfigMap.
	result, err := reconciler.Reconcile(context.Background(), ctrl.Request{
		NamespacedName: types.NamespacedName{
			Namespace: "config",
			Name:      "shared-config",
		},
	})

	if err != nil {
		t.Fatalf("Reconcile failed: %v", err)
	}
	if result.Requeue {
		t.Fatal("Expected no requeue")
	}

	// Verify copies exist in "default" and "app-team" (not "config").
	for _, ns := range []string{"default", "app-team"} {
		var copy corev1.ConfigMap
		err := fakeClient.Get(context.Background(), types.NamespacedName{
			Namespace: ns,
			Name:      "shared-config",
		}, &copy)
		if err != nil {
			t.Errorf("Expected copy in namespace %s, got error: %v", ns, err)
			continue
		}
		if copy.Data["key"] != "value" {
			t.Errorf("Expected data key=value in %s, got %v", ns, copy.Data)
		}
		if copy.Labels[managedByLabel] != "configmap-syncer" {
			t.Errorf("Expected managed-by label in %s", ns)
		}
	}
}

// TestReconcile_CleansUpOnDelete verifies that when a source ConfigMap
// is deleted, all managed copies are cleaned up.
func TestReconcile_CleansUpOnDelete(t *testing.T) {
	// Create a fake client with copies (but no source — simulating
	// deletion).
	copy1 := &corev1.ConfigMap{
		ObjectMeta: metav1.ObjectMeta{
			Name:      "shared-config",
			Namespace: "default",
			Labels: map[string]string{
				managedByLabel: "configmap-syncer",
			},
			Annotations: map[string]string{
				sourceAnnotation: "config/shared-config",
			},
		},
	}
	copy2 := &corev1.ConfigMap{
		ObjectMeta: metav1.ObjectMeta{
			Name:      "shared-config",
			Namespace: "app-team",
			Labels: map[string]string{
				managedByLabel: "configmap-syncer",
			},
			Annotations: map[string]string{
				sourceAnnotation: "config/shared-config",
			},
		},
	}

	fakeClient := fake.NewClientBuilder().
		WithObjects(copy1, copy2).
		Build()

	reconciler := &ConfigMapSyncReconciler{Client: fakeClient}

	// Reconcile the (now-deleted) source ConfigMap.
	result, err := reconciler.Reconcile(context.Background(), ctrl.Request{
		NamespacedName: types.NamespacedName{
			Namespace: "config",
			Name:      "shared-config",
		},
	})

	if err != nil {
		t.Fatalf("Reconcile failed: %v", err)
	}
	if result.Requeue {
		t.Fatal("Expected no requeue")
	}

	// Verify copies are deleted.
	var remainingCopies corev1.ConfigMapList
	if err := fakeClient.List(context.Background(), &remainingCopies); err != nil {
		t.Fatalf("List failed: %v", err)
	}
	if len(remainingCopies.Items) != 0 {
		t.Errorf("Expected 0 remaining copies, got %d", len(remainingCopies.Items))
	}
}
```

### 32.8.4 Deployment Manifests

```yaml
# deployment.yaml — Deploys the ConfigMap Syncer controller.
#
# Key points:
#   - Two replicas for high availability.
#   - Leader election ensures only one is active.
#   - ServiceAccount with RBAC for ConfigMap and Namespace access.
apiVersion: apps/v1
kind: Deployment
metadata:
  name: configmap-syncer
  namespace: configmap-syncer-system
  labels:
    app: configmap-syncer
spec:
  replicas: 2
  selector:
    matchLabels:
      app: configmap-syncer
  template:
    metadata:
      labels:
        app: configmap-syncer
    spec:
      serviceAccountName: configmap-syncer
      containers:
      - name: controller
        image: configmap-syncer:latest
        resources:
          requests:
            cpu: 100m
            memory: 64Mi
          limits:
            cpu: 500m
            memory: 128Mi
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
---
# rbac.yaml — ServiceAccount, ClusterRole, and ClusterRoleBinding.
#
# The controller needs:
#   - Read access to Namespaces (to list all namespaces).
#   - Read/write access to ConfigMaps (to create/update/delete copies).
#   - Coordination leases for leader election.
apiVersion: v1
kind: ServiceAccount
metadata:
  name: configmap-syncer
  namespace: configmap-syncer-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: configmap-syncer
rules:
# ConfigMaps: full CRUD access for syncing.
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Namespaces: read access to discover target namespaces.
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get", "list", "watch"]
# Leases: required for leader election.
- apiGroups: ["coordination.k8s.io"]
  resources: ["leases"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
# Events: for recording controller events.
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: configmap-syncer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: configmap-syncer
subjects:
- kind: ServiceAccount
  name: configmap-syncer
  namespace: configmap-syncer-system
```

---

## 32.9 Advanced Topics

### 32.9.1 Watching Multiple Resources

Controllers often need to watch more than one resource type. For example, a
Deployment controller watches both Deployments and ReplicaSets. When a
ReplicaSet changes, the controller needs to reconcile the *parent Deployment*,
not the ReplicaSet itself.

Controller-runtime makes this easy with `Owns()` and `Watches()`:

```go
// Owns() sets up a watch on the child resource and automatically maps
// events back to the parent (using ownerReferences).
ctrl.NewControllerManagedBy(mgr).
	For(&appsv1.Deployment{}).
	Owns(&appsv1.ReplicaSet{}).
	Complete(reconciler)

// Watches() is more flexible — you provide a custom mapping function.
ctrl.NewControllerManagedBy(mgr).
	For(&myv1.Website{}).
	Watches(
		&corev1.Service{},
		handler.EnqueueRequestsFromMapFunc(mapServiceToWebsite),
	).
	Complete(reconciler)

```

### 32.9.2 Event Recording

Controllers should record Events on the objects they manage. Events show up
in `kubectl describe` and provide visibility into what the controller is doing.

```go
// Create an event recorder from the manager.
recorder := mgr.GetEventRecorderFor("my-controller")

// Record events during reconciliation.
recorder.Event(myObj, corev1.EventTypeNormal, "Synced",
	"Successfully synced to all namespaces")
recorder.Eventf(myObj, corev1.EventTypeWarning, "SyncFailed",
	"Failed to sync to namespace %s: %v", ns, err)

```

### 32.9.3 Metrics

Controller-runtime automatically exposes Prometheus metrics:

- `controller_runtime_reconcile_total` — reconciliation count by result
- `controller_runtime_reconcile_time_seconds` — reconciliation duration
- `workqueue_depth` — current work queue depth
- `workqueue_adds_total` — total items added to work queue

These metrics are served on `:8080/metrics` by default. Use them to monitor
controller health and performance.

### 32.9.4 Dynamic Informers

Sometimes you do not know at compile time what resources you need to watch.
For example, a generic "resource copier" might need to watch any CRD. In
this case, use dynamic informers:

```go
// dynamicInformerFactory creates informers for arbitrary resource types
// using the unstructured (dynamic) client.
dynamicFactory := dynamicinformer.NewDynamicSharedInformerFactory(
	dynamicClient,
	10*time.Minute,
)

// Create an informer for a specific GVR (Group/Version/Resource).
gvr := schema.GroupVersionResource{
	Group:    "example.com",
	Version:  "v1",
	Resource: "websites",
}
informer := dynamicFactory.ForResource(gvr)

```

---

## 32.10 Summary

The controller pattern is the architectural foundation of Kubernetes. Every
behavior in the cluster is driven by a controller that follows the same
loop: observe the current state, compare it to the desired state, and take
action to close the gap.

Key concepts from this chapter:

1. **Desired vs actual state** — the fundamental abstraction of Kubernetes.
2. **Reconciliation loop** — observe, compare, act, repeat.
3. **Level-triggered** — react to current state, not events. This is what
   makes controllers robust against missed events, duplicates, and restarts.
4. **Idempotency** — reconcile functions must be safe to call multiple times.
5. **Informers** — efficient list+watch with local caching, deduplication,
   and shared connections.
6. **Work queues** — deduplication, rate limiting, and retry with exponential
   backoff.
7. **Leader election** — only one controller instance active at a time.
8. **Controller-runtime** — a framework that eliminates boilerplate and
   provides best-practice defaults.

In the next chapter, we will extend the Kubernetes API with **Custom Resource
Definitions (CRDs)** — creating our own resource types that controllers can
watch and reconcile. CRDs + controllers = operators, which is the subject of
Chapter 34.

---

## 32.11 Further Reading

- [Kubernetes Controllers Design
  Documentation](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-api-machinery/controllers.md)
- [client-go Examples](https://github.com/kubernetes/client-go/tree/master/examples)
- [controller-runtime Book (Kubebuilder)](https://book.kubebuilder.io/)
- [Programming Kubernetes (O'Reilly)](https://www.oreilly.com/library/view/programming-kubernetes/9781492047094/)
  by Michael Hausenblas and Stefan Schimanski
- [Writing Kubernetes Controllers for
  CRDs](https://www.youtube.com/watch?v=7wdUa4Ulwxg) — talk by Ahmet Alp
  Balkan at KubeCon
- [Sample Controller](https://github.com/kubernetes/sample-controller) —
  official example of a controller using client-go
