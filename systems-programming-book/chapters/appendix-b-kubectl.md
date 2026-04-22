# Appendix B: kubectl Mastery

> A comprehensive reference for operating Kubernetes clusters with `kubectl`.
> Every command is practical — paste it into your terminal and it works.

---

## Table of Contents

1. [Core Operations](#1-core-operations)
2. [Output Formatting](#2-output-formatting)
3. [JSONPath Expressions](#3-jsonpath-expressions)
4. [kubectl explain](#4-kubectl-explain)
5. [Context and Namespace Management](#5-context-and-namespace-management)
6. [Resource Management](#6-resource-management)
7. [Debugging and Troubleshooting](#7-debugging-and-troubleshooting)
8. [Advanced Operations](#8-advanced-operations)
9. [kubectl Plugins and krew](#9-kubectl-plugins-and-krew)
10. [Shell Completion and Aliases](#10-shell-completion-and-aliases)
11. [Custom Columns Reference](#11-custom-columns-reference)
12. [Operational One-Liners](#12-operational-one-liners)

---

## 1. Core Operations

### get — List resources

```bash
kubectl get pods                                # Pods in current namespace
kubectl get pods -A                             # Pods in ALL namespaces
kubectl get pods -n kube-system                 # Pods in specific namespace
kubectl get pods -w                             # Watch for changes (live updates)
kubectl get pods --show-labels                  # Include labels
kubectl get pods -l app=nginx                   # Filter by label
kubectl get pods -l 'app in (nginx,redis)'      # Label set-based selector
kubectl get pods --field-selector status.phase=Running   # Field selector
kubectl get pods --sort-by=.metadata.creationTimestamp   # Sort by field
kubectl get all                                 # Pods, services, deployments, replicasets
kubectl get events --sort-by=.lastTimestamp     # Cluster events sorted by time
kubectl get nodes                               # Cluster nodes
kubectl get svc,deploy,pods                     # Multiple resource types
```

### describe — Show detailed resource information

```bash
kubectl describe pod mypod                      # Detailed pod info (events, conditions, volumes)
kubectl describe node worker-1                  # Node capacity, allocatable, conditions, pods
kubectl describe svc my-service                 # Service endpoints, ports, selectors
kubectl describe deploy my-deployment           # Deployment strategy, replicas, conditions
kubectl describe pv my-pv                       # PersistentVolume details
kubectl describe ingress my-ingress             # Ingress rules and backends
```

### create — Imperative resource creation

```bash
kubectl create namespace staging                                # Create namespace
kubectl create deployment nginx --image=nginx:1.25 --replicas=3 # Create deployment
kubectl create service clusterip my-svc --tcp=80:8080           # Create ClusterIP service
kubectl create configmap my-config --from-file=config.yaml      # ConfigMap from file
kubectl create configmap my-config --from-literal=key1=val1     # ConfigMap from literal
kubectl create secret generic my-secret --from-literal=pass=s3cret  # Secret from literal
kubectl create secret tls my-tls --cert=cert.pem --key=key.pem # TLS secret
kubectl create job my-job --image=busybox -- echo "hello"       # Run a job
kubectl create cronjob my-cron --image=busybox --schedule="*/5 * * * *" -- echo "tick"
kubectl create serviceaccount my-sa                             # Service account
kubectl create role my-role --verb=get,list --resource=pods     # RBAC role
kubectl create rolebinding my-rb --role=my-role --serviceaccount=default:my-sa
kubectl create clusterrole my-cr --verb='*' --resource=pods     # Cluster-wide role
kubectl create clusterrolebinding my-crb --clusterrole=my-cr --serviceaccount=default:my-sa
```

### apply — Declarative resource management

```bash
kubectl apply -f manifest.yaml                  # Apply a manifest
kubectl apply -f directory/                     # Apply all manifests in directory
kubectl apply -f https://url/to/manifest.yaml   # Apply from URL
kubectl apply -k ./kustomize-dir/               # Apply with Kustomize
kubectl apply -f - <<EOF                        # Apply from stdin
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  containers:
  - name: test
    image: nginx
EOF
kubectl apply --dry-run=client -f manifest.yaml  # Client-side dry run (validate locally)
kubectl apply --dry-run=server -f manifest.yaml  # Server-side dry run (validate with API server)
kubectl apply -f manifest.yaml --force           # Force apply (delete and recreate)
```

### delete — Remove resources

```bash
kubectl delete pod mypod                        # Delete a pod
kubectl delete pod mypod --grace-period=0 --force  # Force-delete (skip graceful shutdown)
kubectl delete -f manifest.yaml                 # Delete resources defined in file
kubectl delete pods -l app=test                 # Delete pods matching label
kubectl delete pods --all -n staging            # Delete all pods in namespace
kubectl delete namespace staging                # Delete namespace and everything in it
kubectl delete pod mypod --wait=false           # Don't wait for deletion
```

### edit — Modify resources interactively

```bash
kubectl edit deployment my-deployment           # Open in $EDITOR
kubectl edit svc my-service                     # Edit service
KUBE_EDITOR="code --wait" kubectl edit deploy my-deployment  # Use VS Code
```

### patch — Modify resources programmatically

```bash
# Strategic merge patch (default)
kubectl patch deployment my-deploy -p '{"spec":{"replicas":5}}'

# JSON merge patch
kubectl patch deploy my-deploy --type=merge -p '{"spec":{"replicas":5}}'

# JSON patch (RFC 6902)
kubectl patch deploy my-deploy --type=json \
  -p='[{"op":"replace","path":"/spec/replicas","value":5}]'

# Patch a node (e.g., add a label)
kubectl patch node worker-1 -p '{"metadata":{"labels":{"disk":"ssd"}}}'

# Patch a service account to add an image pull secret
kubectl patch serviceaccount default -p '{"imagePullSecrets":[{"name":"my-registry"}]}'
```

---

## 2. Output Formatting

### Wide output

```bash
kubectl get pods -o wide                        # Extra columns: Node, IP, nominated node
kubectl get nodes -o wide                       # Internal/External IPs, OS, kernel, runtime
```

### YAML / JSON output

```bash
kubectl get pod mypod -o yaml                   # Full YAML (including status, metadata)
kubectl get pod mypod -o json                   # Full JSON
kubectl get pod mypod -o yaml | kubectl neat    # Clean YAML (with kubectl-neat plugin)
```

### JSONPath output

```bash
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
kubectl get pod mypod -o jsonpath='{.status.podIP}'
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\n"}{end}'
```

### Custom columns

```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,IP:.status.podIP
kubectl get pods -o custom-columns=\
  NAME:.metadata.name,\
  NODE:.spec.nodeName,\
  CPU-REQ:.spec.containers[0].resources.requests.cpu,\
  MEM-REQ:.spec.containers[0].resources.requests.memory
```

### Name-only output

```bash
kubectl get pods -o name                        # pod/mypod-abc123
kubectl get pods -o name | cut -d/ -f2          # Just the names
```

---

## 3. JSONPath Expressions

JSONPath lets you extract and transform `kubectl` output into exactly the fields you need.

### Basic syntax

```bash
# Access a field
kubectl get pod mypod -o jsonpath='{.metadata.name}'

# Access nested field
kubectl get pod mypod -o jsonpath='{.spec.containers[0].image}'

# Access all items in a list
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Index into a list
kubectl get pod mypod -o jsonpath='{.spec.containers[0].name}'
```

### Filtering with ?()

```bash
# Find container named "app"
kubectl get pod mypod -o jsonpath='{.spec.containers[?(@.name=="app")].image}'

# Nodes with condition Ready=True
kubectl get nodes -o jsonpath='{.items[?(@.status.conditions[?(@.type=="Ready")].status=="True")].metadata.name}'

# Pods in Running phase
kubectl get pods -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}'
```

### Range iteration

```bash
# Formatted output with range
kubectl get pods -o jsonpath='{range .items[*]}{"Pod: "}{.metadata.name}{"\tNode: "}{.spec.nodeName}{"\n"}{end}'

# Multi-column table
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.capacity.cpu}{"\t"}{.status.capacity.memory}{"\n"}{end}'
```

### Sorting (client-side)

```bash
# Sort pods by restart count
kubectl get pods --sort-by='.status.containerStatuses[0].restartCount'

# Sort events by timestamp
kubectl get events --sort-by='.lastTimestamp'

# Sort pods by creation time
kubectl get pods --sort-by='.metadata.creationTimestamp'
```

### Complex examples

```bash
# All images running in the cluster
kubectl get pods -A -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' | sort -u

# All node IPs
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .status.addresses[*]}{.type}:{.address}{"\t"}{end}{"\n"}{end}'

# PVCs with their bound PV and storage class
kubectl get pvc -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.volumeName}{"\t"}{.spec.storageClassName}{"\t"}{.status.phase}{"\n"}{end}'
```

---

## 4. kubectl explain

`kubectl explain` is your built-in API reference — it describes every field of every resource.

```bash
kubectl explain pod                             # Top-level Pod spec
kubectl explain pod.spec                        # Pod spec fields
kubectl explain pod.spec.containers             # Container spec
kubectl explain pod.spec.containers.resources   # Resource requests/limits
kubectl explain pod.spec.volumes                # Volume types
kubectl explain deployment.spec.strategy        # Deployment strategy
kubectl explain service.spec                    # Service spec
kubectl explain ingress.spec.rules              # Ingress rules
kubectl explain --recursive pod.spec            # Full recursive field listing
kubectl explain pod --api-version=v1            # Specific API version

# Discover fields for any resource
kubectl explain statefulset.spec.volumeClaimTemplates
kubectl explain cronjob.spec.jobTemplate.spec.template.spec
kubectl explain networkpolicy.spec.ingress
```

---

## 5. Context and Namespace Management

### kubeconfig and contexts

```bash
kubectl config view                             # View kubeconfig
kubectl config view --minify                    # Only current context
kubectl config current-context                  # Show current context
kubectl config get-contexts                     # List all contexts
kubectl config use-context my-cluster           # Switch context
kubectl config set-context --current --namespace=staging  # Set default namespace

# Create a new context
kubectl config set-context my-ctx \
  --cluster=my-cluster \
  --user=my-user \
  --namespace=production

kubectl config delete-context old-context       # Remove context
kubectl config set-cluster my-cluster --server=https://api.example.com:6443
kubectl config set-credentials my-user --token=bearer_token_here
```

### Namespace operations

```bash
kubectl get namespaces                          # List namespaces
kubectl create namespace staging                # Create namespace
kubectl delete namespace staging                # Delete namespace (and ALL resources in it)
kubectl get pods --namespace=kube-system        # Resources in specific namespace
kubectl get pods -A                             # Resources across ALL namespaces
kubectl get pods --all-namespaces               # Same as -A

# Shorthand: set default namespace per-context
kubectl config set-context --current --namespace=production
```

---

## 6. Resource Management

### Labels

```bash
kubectl label pod mypod env=production                        # Add label
kubectl label pod mypod env=staging --overwrite                # Update label
kubectl label pod mypod env-                                   # Remove label
kubectl label pods --all release=v1.2                          # Label all pods
kubectl get pods -l env=production                             # Select by label
kubectl get pods -l 'env in (production,staging)'              # Set-based selector
kubectl get pods -l 'env notin (test)'                         # Exclusion selector
kubectl get pods -l env!=test                                  # Inequality selector
kubectl get pods -l app=nginx,env=production                   # Multiple labels (AND)
kubectl get pods --show-labels                                 # Show all labels
```

### Annotations

```bash
kubectl annotate pod mypod description="My important pod"     # Add annotation
kubectl annotate pod mypod description-                        # Remove annotation
kubectl annotate pod mypod description="updated" --overwrite   # Update annotation
```

### Taints and tolerations

```bash
kubectl taint node worker-1 dedicated=gpu:NoSchedule           # Add taint
kubectl taint node worker-1 dedicated=gpu:NoSchedule-          # Remove taint
kubectl taint node worker-1 dedicated:NoSchedule-              # Remove taint by key+effect
kubectl describe node worker-1 | grep -A5 Taints               # View taints

# To tolerate in a pod spec:
# tolerations:
# - key: "dedicated"
#   operator: "Equal"
#   value: "gpu"
#   effect: "NoSchedule"
```

### Cordon, uncordon, drain

```bash
kubectl cordon worker-1                         # Mark node unschedulable
kubectl uncordon worker-1                       # Mark node schedulable again
kubectl drain worker-1 --ignore-daemonsets      # Evict pods, prepare for maintenance
kubectl drain worker-1 --ignore-daemonsets --delete-emptydir-data  # Also delete emptyDir pods
kubectl drain worker-1 --ignore-daemonsets --force  # Force-drain (delete standalone pods)
kubectl drain worker-1 --grace-period=30        # Custom grace period
kubectl drain worker-1 --timeout=120s           # Timeout for drain operation
```

---

## 7. Debugging and Troubleshooting

### logs — View container logs

```bash
kubectl logs mypod                              # Logs from single-container pod
kubectl logs mypod -c mycontainer               # Specific container in multi-container pod
kubectl logs mypod --all-containers             # Logs from all containers
kubectl logs mypod -f                           # Follow (stream) logs
kubectl logs mypod --tail=100                   # Last 100 lines
kubectl logs mypod --since=1h                   # Logs from last hour
kubectl logs mypod --since-time=2024-01-15T10:00:00Z  # Since specific time
kubectl logs mypod -p                           # Logs from previous (crashed) container
kubectl logs -l app=nginx                       # Logs from all pods matching label
kubectl logs -l app=nginx --max-log-requests=10 # Increase concurrent log streams
kubectl logs deployment/my-deploy               # Logs from deployment's pods
kubectl logs job/my-job                         # Logs from job
```

### exec — Execute commands in containers

```bash
kubectl exec mypod -- ls /app                   # Run a command
kubectl exec mypod -c mycontainer -- cat /etc/resolv.conf  # Specific container
kubectl exec -it mypod -- /bin/bash             # Interactive shell
kubectl exec -it mypod -- sh                    # If bash isn't available
kubectl exec mypod -- env                       # View environment variables
kubectl exec mypod -- cat /proc/1/status        # Process info from inside container
```

### port-forward — Local port forwarding

```bash
kubectl port-forward pod/mypod 8080:80          # Forward local:8080 → pod:80
kubectl port-forward svc/my-service 8080:80     # Forward to service
kubectl port-forward deploy/my-deploy 8080:80   # Forward to deployment
kubectl port-forward pod/mypod 8080:80 9090:9090  # Multiple ports
kubectl port-forward --address 0.0.0.0 pod/mypod 8080:80  # Listen on all interfaces
```

### debug — Ephemeral debug containers (K8s 1.23+)

```bash
kubectl debug mypod -it --image=busybox         # Attach debug container to running pod
kubectl debug mypod -it --image=nicolaka/netshoot  # Network debugging tools
kubectl debug mypod -it --image=busybox --target=mycontainer  # Share PID namespace
kubectl debug node/worker-1 -it --image=ubuntu  # Debug a node (creates privileged pod)
kubectl debug mypod --copy-to=mypod-debug --container=debug --image=busybox -it  # Copy pod for debugging
```

### cp — Copy files to/from containers

```bash
kubectl cp mypod:/var/log/app.log ./app.log     # Copy from pod to local
kubectl cp ./config.yaml mypod:/etc/config/     # Copy from local to pod
kubectl cp mypod:/data ./data -c mycontainer    # Specific container
```

### top — Resource usage

```bash
kubectl top nodes                               # Node CPU and memory usage
kubectl top pods                                # Pod CPU and memory usage
kubectl top pods -A                             # All namespaces
kubectl top pods --sort-by=cpu                  # Sort by CPU
kubectl top pods --sort-by=memory               # Sort by memory
kubectl top pod mypod --containers              # Per-container usage
```

### auth — Authentication and authorization checks

```bash
kubectl auth can-i create pods                          # Can I create pods?
kubectl auth can-i delete deployments --namespace=prod  # In specific namespace?
kubectl auth can-i '*' '*'                              # Am I cluster-admin?
kubectl auth can-i get pods --as=system:serviceaccount:default:my-sa  # Check as another identity
kubectl auth can-i list secrets --as=jane               # Check as another user
kubectl auth whoami                                     # Who am I? (K8s 1.28+)
```

---

## 8. Advanced Operations

### diff — Preview changes before applying

```bash
kubectl diff -f manifest.yaml                   # Show what would change
kubectl diff -f directory/                      # Diff entire directory
kubectl diff -k ./kustomize-dir/                # Diff with Kustomize
```

### wait — Wait for conditions

```bash
kubectl wait --for=condition=ready pod mypod --timeout=60s
kubectl wait --for=condition=available deployment/my-deploy --timeout=120s
kubectl wait --for=delete pod/mypod --timeout=60s
kubectl wait --for=condition=complete job/my-job --timeout=300s
kubectl wait --for=jsonpath='{.status.phase}'=Running pod/mypod
```

### api-resources and api-versions

```bash
kubectl api-resources                           # All resource types in the cluster
kubectl api-resources --namespaced=true         # Only namespaced resources
kubectl api-resources --namespaced=false        # Only cluster-scoped resources
kubectl api-resources --verbs=list              # Resources that support list
kubectl api-resources -o name                   # Just resource names
kubectl api-versions                            # All API group/versions
```

### Rollout management

```bash
kubectl rollout status deployment/my-deploy     # Watch rollout progress
kubectl rollout history deployment/my-deploy    # Rollout history
kubectl rollout undo deployment/my-deploy       # Rollback to previous revision
kubectl rollout undo deployment/my-deploy --to-revision=3  # Rollback to specific revision
kubectl rollout restart deployment/my-deploy    # Trigger rolling restart
kubectl rollout pause deployment/my-deploy      # Pause rollout
kubectl rollout resume deployment/my-deploy     # Resume rollout
```

### Scaling

```bash
kubectl scale deployment my-deploy --replicas=5     # Scale deployment
kubectl scale statefulset my-sts --replicas=3       # Scale StatefulSet
kubectl autoscale deployment my-deploy --min=2 --max=10 --cpu-percent=80  # Create HPA
```

### Resource quotas and limits

```bash
kubectl describe quota -n production            # View resource quotas
kubectl describe limitrange -n production       # View limit ranges
kubectl get resourcequota -A                    # All quotas across namespaces
```

---

## 9. kubectl Plugins and krew

### Installing krew (plugin manager)

```bash
# Install krew (see https://krew.sigs.k8s.io/docs/user-guide/setup/install/)
kubectl krew install <plugin>                   # Install a plugin
kubectl krew list                               # List installed plugins
kubectl krew search                             # Search available plugins
kubectl krew upgrade                            # Upgrade all plugins
kubectl krew info <plugin>                      # Plugin information
```

### Essential plugins

```bash
# ctx — Fast context switching
kubectl krew install ctx
kubectl ctx                                     # List contexts
kubectl ctx my-cluster                          # Switch context

# ns — Fast namespace switching
kubectl krew install ns
kubectl ns                                      # List namespaces
kubectl ns staging                              # Switch namespace

# neat — Clean up YAML output (remove managed fields, status)
kubectl krew install neat
kubectl get pod mypod -o yaml | kubectl neat

# images — Show container images in use
kubectl krew install images
kubectl images                                  # All images across cluster
kubectl images -n production                    # Images in namespace

# tree — Show resource hierarchy
kubectl krew install tree
kubectl tree deployment my-deploy               # Deployment → ReplicaSet → Pods

# sniff — Capture pod traffic with Wireshark
kubectl krew install sniff
kubectl sniff mypod                             # Capture traffic

# resource-capacity — Node resource usage summary
kubectl krew install resource-capacity
kubectl resource-capacity                       # Cluster-wide capacity overview

# who-can — RBAC analysis
kubectl krew install who-can
kubectl who-can create pods                     # Who can create pods?
kubectl who-can delete nodes                    # Who can delete nodes?

# access-matrix — RBAC access matrix
kubectl krew install access-matrix
kubectl access-matrix                           # Show access matrix for current user
kubectl access-matrix --as=jane                 # For another user

# stern — Multi-pod log tailing
# (installed separately, not via krew)
stern "nginx-.*" -n production                  # Tail all nginx pods
stern . -n kube-system --since=10m              # All pods in namespace, last 10m
```

---

## 10. Shell Completion and Aliases

### Bash completion

```bash
# One-time setup
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
```

### Zsh completion

```bash
# One-time setup
echo 'source <(kubectl completion zsh)' >> ~/.zshrc
echo 'alias k=kubectl' >> ~/.zshrc
echo 'compdef __start_kubectl k' >> ~/.zshrc
source ~/.zshrc
```

### Recommended aliases

```bash
# Core aliases
alias k='kubectl'
alias kg='kubectl get'
alias kd='kubectl describe'
alias kaf='kubectl apply -f'
alias kdel='kubectl delete'
alias ke='kubectl edit'
alias kl='kubectl logs'
alias klf='kubectl logs -f'
alias kex='kubectl exec -it'

# Resource-specific
alias kgp='kubectl get pods'
alias kgpa='kubectl get pods -A'
alias kgs='kubectl get svc'
alias kgd='kubectl get deploy'
alias kgn='kubectl get nodes'
alias kgns='kubectl get namespaces'
alias kgi='kubectl get ingress'
alias kgcm='kubectl get configmap'
alias kgsec='kubectl get secrets'
alias kgpv='kubectl get pv'
alias kgpvc='kubectl get pvc'

# Describe shortcuts
alias kdp='kubectl describe pod'
alias kdn='kubectl describe node'
alias kds='kubectl describe svc'

# Context / namespace
alias kctx='kubectl config use-context'
alias kns='kubectl config set-context --current --namespace'
alias kcurrent='kubectl config current-context'

# Debugging
alias kpf='kubectl port-forward'
alias ktp='kubectl top pods'
alias ktn='kubectl top nodes'

# Powerful function: get all resources in a namespace
kall() { kubectl api-resources --verbs=list --namespaced -o name | xargs -n1 kubectl get --show-kind --ignore-not-found -n "${1:-default}"; }
```

---

## 11. Custom Columns Reference

Custom column definitions for common resource types.

### Pods

```bash
kubectl get pods -o custom-columns="\
NAME:.metadata.name,\
STATUS:.status.phase,\
IP:.status.podIP,\
NODE:.spec.nodeName,\
RESTARTS:.status.containerStatuses[0].restartCount,\
AGE:.metadata.creationTimestamp"
```

### Nodes

```bash
kubectl get nodes -o custom-columns="\
NAME:.metadata.name,\
STATUS:.status.conditions[-1].type,\
ROLES:.metadata.labels.node-role\.kubernetes\.io/worker,\
CPU:.status.capacity.cpu,\
MEMORY:.status.capacity.memory,\
RUNTIME:.status.nodeInfo.containerRuntimeVersion,\
KERNEL:.status.nodeInfo.kernelVersion"
```

### Deployments

```bash
kubectl get deploy -o custom-columns="\
NAME:.metadata.name,\
READY:.status.readyReplicas,\
DESIRED:.spec.replicas,\
AVAILABLE:.status.availableReplicas,\
STRATEGY:.spec.strategy.type,\
IMAGE:.spec.template.spec.containers[0].image"
```

### Services

```bash
kubectl get svc -o custom-columns="\
NAME:.metadata.name,\
TYPE:.spec.type,\
CLUSTER-IP:.spec.clusterIP,\
EXTERNAL-IP:.status.loadBalancer.ingress[0].ip,\
PORTS:.spec.ports[*].port,\
SELECTOR:.spec.selector"
```

### PersistentVolumeClaims

```bash
kubectl get pvc -o custom-columns="\
NAME:.metadata.name,\
STATUS:.status.phase,\
VOLUME:.spec.volumeName,\
CAPACITY:.status.capacity.storage,\
STORAGECLASS:.spec.storageClassName,\
ACCESS-MODES:.spec.accessModes[*]"
```

---

## 12. Operational One-Liners

Real-world tasks solved with single `kubectl` commands.

### Find pods not in Running state

```bash
kubectl get pods -A --field-selector 'status.phase!=Running' | grep -v Completed
```

### Get all images running in the cluster

```bash
kubectl get pods -A -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' | sort -u
```

### Find pods without resource limits

```bash
kubectl get pods -A -o json | jq -r '.items[] | select(.spec.containers[].resources.limits == null) | .metadata.namespace + "/" + .metadata.name'
```

### Restart all pods in a deployment

```bash
kubectl rollout restart deployment/my-deploy
```

### Get pod resource requests vs actual usage

```bash
kubectl top pods --containers | sort -k3 -rn | head -20
```

### Find evicted pods and clean them up

```bash
kubectl get pods -A --field-selector status.phase=Failed | grep Evicted
kubectl delete pods -A --field-selector status.phase=Failed
```

### Scale all deployments in a namespace to zero

```bash
kubectl get deploy -n staging -o name | xargs -I{} kubectl scale {} --replicas=0 -n staging
```

### Get nodes with their taints

```bash
kubectl get nodes -o custom-columns='NAME:.metadata.name,TAINTS:.spec.taints[*].key'
```

### Watch pod creation (useful during rollouts)

```bash
kubectl get pods -w -l app=my-app
```

### Export a resource (clean, reusable YAML)

```bash
kubectl get deployment my-deploy -o yaml | kubectl neat > my-deploy.yaml
```

### Check which pods are on which nodes

```bash
kubectl get pods -A -o custom-columns='NAMESPACE:.metadata.namespace,POD:.metadata.name,NODE:.spec.nodeName' --sort-by=.spec.nodeName
```

### Find pods with high restart counts

```bash
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{range .status.containerStatuses[*]}{.restartCount}{"\n"}{end}{end}' | awk -F'\t' '$3 > 5'
```

### Check for pending PVCs

```bash
kubectl get pvc -A --field-selector status.phase=Pending
```

### Get all endpoints (which pods back which services)

```bash
kubectl get endpoints -A
```

### Force-delete a stuck namespace

```bash
# When a namespace is stuck in Terminating state
kubectl get namespace stuck-ns -o json | jq '.spec.finalizers = []' | kubectl replace --raw "/api/v1/namespaces/stuck-ns/finalize" -f -
```

### Copy secrets between namespaces

```bash
kubectl get secret my-secret -n source -o yaml | sed 's/namespace: source/namespace: dest/' | kubectl apply -f -
```

### Find the most recent events

```bash
kubectl get events -A --sort-by=.lastTimestamp | tail -20
```

### Quick pod for interactive debugging

```bash
kubectl run debug --rm -it --image=nicolaka/netshoot -- bash
kubectl run debug --rm -it --image=busybox -- sh
kubectl run debug --rm -it --image=curlimages/curl -- sh
kubectl run debug --rm -it --image=ubuntu -- bash
```

### Decode a secret

```bash
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 -d
```

### Check RBAC permissions for a service account

```bash
kubectl auth can-i --list --as=system:serviceaccount:default:my-sa
```

### Get pod logs from all replicas

```bash
kubectl logs -l app=my-app --all-containers --prefix
```

### Find pods consuming the most memory

```bash
kubectl top pods -A --sort-by=memory | head -20
```

### Quickly create a test pod with specific resources

```bash
kubectl run load-test --image=busybox --restart=Never \
  --requests='cpu=100m,memory=128Mi' \
  --limits='cpu=200m,memory=256Mi' \
  -- sleep 3600
```

---

*For the full API reference, use `kubectl explain <resource>` or visit
https://kubernetes.io/docs/reference/kubectl/. Master these commands and
you can operate any Kubernetes cluster with confidence.*
