# Chapter 39: Disaster Recovery, Backup & Multi-Cluster

> *"Hope is not a strategy. Every production cluster needs a tested disaster recovery plan
> before disaster strikes — not after."*

In previous chapters we built increasingly sophisticated Kubernetes deployments —
stateful workloads, security policies, observability stacks, and GitOps pipelines.
But none of that matters if a single misconfigured `kubectl delete` or a region-wide
cloud outage wipes it all away. This chapter is about **resilience at the infrastructure
level**: how to back up everything that matters, how to recover when disaster strikes,
and how to architect multi-cluster topologies that survive failures gracefully.

We will cover the full spectrum — from low-level etcd snapshots to high-level
multi-cluster service meshes — and build Go programs that automate every step.

---

## 39.1 etcd Backup and Restore

etcd is the single source of truth for every Kubernetes cluster. Every object —
Pods, Services, ConfigMaps, Secrets, CRDs — lives in etcd. Losing etcd means
losing your cluster state entirely.

### 39.1.1 Understanding etcd's Role

etcd is a distributed key-value store that implements the Raft consensus protocol.
In a typical HA control plane, three or five etcd members form a quorum. The
Kubernetes API server reads from and writes to etcd exclusively.

Key points:

- **All cluster state** is stored in etcd under the `/registry/` prefix.
- etcd stores data in a **B+ tree** backed by a write-ahead log (WAL).
- A **snapshot** captures the entire B+ tree at a point in time.
- etcd **compaction** removes old revisions to reclaim space.

### 39.1.2 etcdctl snapshot save

The `etcdctl` CLI ships with every etcd distribution. Taking a snapshot is
straightforward:

```bash
# Set environment variables for TLS authentication
export ETCDCTL_API=3
export ETCDCTL_ENDPOINTS=https://127.0.0.1:2379
export ETCDCTL_CACERT=/etc/kubernetes/pki/etcd/ca.crt
export ETCDCTL_CERT=/etc/kubernetes/pki/etcd/server.crt
export ETCDCTL_KEY=/etc/kubernetes/pki/etcd/server.key

# Take the snapshot
etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db

# Verify the snapshot
etcdctl snapshot status /backup/etcd-snapshot-20240115-143000.db --write-table
```

The `snapshot status` command outputs:

| HASH       | REVISION | TOTAL KEYS | TOTAL SIZE |
|------------|----------|------------|------------|
| 0x3a8b2c1d | 1548923  | 2847       | 14 MB      |

**Critical considerations:**

1. **Always use TLS certificates.** In kubeadm clusters, certs live under
   `/etc/kubernetes/pki/etcd/`.
2. **Snapshots are consistent.** etcd guarantees point-in-time consistency for
   the snapshot operation.
3. **Do not snapshot from a non-leader member** in clusters under heavy write
   load — the snapshot may be slightly behind.
4. **Store snapshots off-cluster.** If the node dies, a local snapshot is useless.

### 39.1.3 etcdctl snapshot restore

Restoring replaces the entire etcd data directory. This is a destructive operation
that should be performed carefully:

```bash
# Stop the API server and etcd (on kubeadm clusters)
sudo systemctl stop kubelet
sudo mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
sudo mv /etc/kubernetes/manifests/etcd.yaml /tmp/

# Remove old data directory
sudo rm -rf /var/lib/etcd/member

# Restore the snapshot
etcdctl snapshot restore /backup/etcd-snapshot-20240115-143000.db \
  --data-dir=/var/lib/etcd \
  --name=controlplane \
  --initial-cluster=controlplane=https://10.0.0.10:2380 \
  --initial-cluster-token=etcd-cluster-1 \
  --initial-advertise-peer-urls=https://10.0.0.10:2380

# Move manifests back and start kubelet
sudo mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/
sudo mv /tmp/etcd.yaml /etc/kubernetes/manifests/
sudo systemctl start kubelet
```

**For multi-member etcd clusters**, you must restore on every member simultaneously,
each with its own `--name` and `--initial-advertise-peer-urls`. The
`--initial-cluster` flag lists all members.

### 39.1.4 Automated Backup Strategies

Manual backups don't scale. Here are production-grade approaches:

**CronJob-based backup:**

```yaml
# etcd-backup-cronjob.yaml
# Runs every 6 hours, saves snapshot to a PVC and syncs to S3.
apiVersion: batch/v1
kind: CronJob
metadata:
  name: etcd-backup
  namespace: kube-system
spec:
  schedule: "0 */6 * * *"
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:
      template:
        spec:
          hostNetwork: true
          nodeSelector:
            node-role.kubernetes.io/control-plane: ""
          tolerations:
            - effect: NoSchedule
              operator: Exists
          containers:
            - name: backup
              image: bitnami/etcd:3.5
              command:
                - /bin/sh
                - -c
                - |
                  FILENAME="etcd-$(date +%Y%m%d-%H%M%S).db"
                  etcdctl snapshot save "/backup/${FILENAME}"
                  etcdctl snapshot status "/backup/${FILENAME}" --write-table
                  # Upload to S3
                  aws s3 cp "/backup/${FILENAME}" "s3://my-etcd-backups/${FILENAME}"
                  # Retain only last 30 local backups
                  ls -t /backup/*.db | tail -n +31 | xargs rm -f
              env:
                - name: ETCDCTL_API
                  value: "3"
                - name: ETCDCTL_ENDPOINTS
                  value: "https://127.0.0.1:2379"
                - name: ETCDCTL_CACERT
                  value: "/etc/kubernetes/pki/etcd/ca.crt"
                - name: ETCDCTL_CERT
                  value: "/etc/kubernetes/pki/etcd/server.crt"
                - name: ETCDCTL_KEY
                  value: "/etc/kubernetes/pki/etcd/server.key"
              volumeMounts:
                - name: etcd-certs
                  mountPath: /etc/kubernetes/pki/etcd
                  readOnly: true
                - name: backup-volume
                  mountPath: /backup
          volumes:
            - name: etcd-certs
              hostPath:
                path: /etc/kubernetes/pki/etcd
            - name: backup-volume
              persistentVolumeClaim:
                claimName: etcd-backup-pvc
          restartPolicy: OnFailure
```

**Backup rotation policy:**

| Retention Tier | Frequency | Retention Period |
|---------------|-----------|------------------|
| Hourly        | Every 1h  | 24 hours         |
| Daily         | Every 24h | 30 days          |
| Weekly        | Weekly    | 90 days          |
| Monthly       | Monthly   | 1 year           |

### 39.1.5 Hands-on: Complete etcd Backup and Restore Workflow

Let's walk through the full cycle on a kubeadm cluster:

```bash
# Step 1: Verify etcd health
etcdctl endpoint health --cluster
# Output: https://10.0.0.10:2379 is healthy: ...

# Step 2: Create test data we can verify later
kubectl create namespace dr-test
kubectl -n dr-test create configmap canary --from-literal=status=alive

# Step 3: Take backup
etcdctl snapshot save /backup/pre-test.db
etcdctl snapshot status /backup/pre-test.db --write-table

# Step 4: Simulate disaster — delete the namespace
kubectl delete namespace dr-test

# Step 5: Verify data is gone
kubectl get namespace dr-test
# Error from server (NotFound): namespaces "dr-test" not found

# Step 6: Restore from backup (follow the restore procedure above)
# ... (stop API server, restore, restart)

# Step 7: Verify recovery
kubectl get namespace dr-test
# NAME      STATUS   AGE
# dr-test   Active   5m

kubectl -n dr-test get configmap canary -o jsonpath='{.data.status}'
# alive
```

> **Warning:** etcd restore rolls back ALL cluster state to the snapshot time.
> Any resources created after the snapshot will be lost. This is why frequent
> backups and Velero-level application backups are complementary strategies.

---

## 39.2 Velero — Kubernetes Backup

While etcd backups capture raw cluster state, **Velero** provides application-aware
backup and restore. Velero understands Kubernetes resources, can back up persistent
volumes, and supports granular restore operations.

### 39.2.1 Architecture

Velero consists of several components:

```
┌────────────────────────────────────────────────────┐
│                   Velero Architecture               │
│                                                     │
│  ┌─────────────┐     ┌──────────────────────────┐  │
│  │ velero CLI   │────▶│   Velero Server (Pod)     │  │
│  └─────────────┘     │                            │  │
│                      │  ┌──────────────────────┐  │  │
│                      │  │  Backup Controller    │  │  │
│                      │  ├──────────────────────┤  │  │
│                      │  │  Restore Controller   │  │  │
│                      │  ├──────────────────────┤  │  │
│                      │  │  Schedule Controller  │  │  │
│                      │  └──────────────────────┘  │  │
│                      └────────────┬───────────────┘  │
│                                   │                   │
│                      ┌────────────▼───────────────┐  │
│                      │       Plugin System         │  │
│                      │  ┌────────┐ ┌────────────┐ │  │
│                      │  │ AWS S3 │ │ CSI Snapshotter│ │
│                      │  │ Plugin │ │   Plugin    │ │  │
│                      │  └────────┘ └────────────┘ │  │
│                      └────────────┬───────────────┘  │
│                                   │                   │
│                      ┌────────────▼───────────────┐  │
│                      │   Object Storage (S3/GCS)   │  │
│                      │   + Volume Snapshot Store    │  │
│                      └────────────────────────────┘  │
└────────────────────────────────────────────────────┘
```

**Key components:**

- **Velero Server**: Runs as a Deployment in the `velero` namespace. Contains
  controllers for Backup, Restore, and Schedule custom resources.
- **Plugins**: Provider-specific code for object storage (AWS S3, GCP GCS,
  Azure Blob) and volume snapshots (CSI, AWS EBS, etc.).
- **BackupStorageLocation (BSL)**: Defines where backup files are stored.
- **VolumeSnapshotLocation (VSL)**: Defines where volume snapshots are stored.
- **Restic/Kopia**: File-level backup agents for volumes that don't support
  native snapshots. Run as a DaemonSet on every node.

### 39.2.2 Backup Types

Velero supports flexible backup scoping:

```bash
# Full cluster backup (everything except kube-system by default)
velero backup create full-cluster-backup

# Namespace-scoped backup
velero backup create app-backup --include-namespaces=production,staging

# Label-selector backup
velero backup create frontend-backup \
  --selector app=frontend

# Resource-type backup
velero backup create secrets-backup \
  --include-resources=secrets,configmaps

# Exclude specific resources
velero backup create no-logs-backup \
  --exclude-resources=events,pods.metrics.k8s.io

# Include cluster-scoped resources
velero backup create with-cluster-resources \
  --include-cluster-resources=true

# Backup with volume snapshots
velero backup create stateful-backup \
  --include-namespaces=database \
  --snapshot-volumes=true

# Backup with file-level copy (Restic/Kopia)
velero backup create restic-backup \
  --include-namespaces=production \
  --default-volumes-to-fs-backup=true
```

**What Velero backs up:**

| Category              | Included by Default | Notes                                    |
|-----------------------|--------------------|-----------------------------------------|
| Namespaced resources  | Yes                | Pods, Services, Deployments, etc.       |
| Cluster resources     | Optional           | ClusterRoles, PVs, Namespaces           |
| Persistent Volumes    | With flag          | Via snapshots or file-level backup      |
| CRDs                  | Yes                | Custom Resource Definitions             |
| CR instances          | Yes                | Custom Resource instances               |
| Secrets               | Yes                | Backed up (encrypted at rest in store)  |

### 39.2.3 Restore Operations

Restoring can be full or partial:

```bash
# Full restore from a backup
velero restore create --from-backup full-cluster-backup

# Restore only specific namespaces
velero restore create --from-backup full-cluster-backup \
  --include-namespaces=production

# Restore specific resources
velero restore create --from-backup full-cluster-backup \
  --include-resources=deployments,services

# Restore with name mapping (useful for cloning)
velero restore create --from-backup app-backup \
  --namespace-mappings old-namespace:new-namespace

# Restore and overwrite existing resources
velero restore create --from-backup app-backup \
  --existing-resource-policy=update

# Check restore status
velero restore describe <restore-name> --details

# View restore logs
velero restore logs <restore-name>
```

**Restore behavior:**

- By default, Velero **skips** resources that already exist in the cluster.
- Use `--existing-resource-policy=update` to overwrite existing resources.
- PVs are restored from snapshots or Restic/Kopia backups.
- Resource ordering: Namespaces → CRDs → Cluster resources → Namespaced resources.

### 39.2.4 Schedules

Automated recurring backups:

```bash
# Daily backup at 2 AM UTC, retain for 30 days
velero schedule create daily-backup \
  --schedule="0 2 * * *" \
  --ttl 720h

# Hourly production backups
velero schedule create hourly-prod \
  --schedule="0 * * * *" \
  --include-namespaces=production \
  --ttl 168h \
  --snapshot-volumes=true

# Weekly full backup on Sundays
velero schedule create weekly-full \
  --schedule="0 0 * * 0" \
  --include-cluster-resources=true \
  --ttl 2160h

# Check schedule status
velero schedule get
velero schedule describe daily-backup
```

### 39.2.5 Hands-on: Installing Velero

**Prerequisites:** An S3-compatible storage bucket and credentials.

```bash
# Create credentials file
cat > credentials-velero <<EOF
[default]
aws_access_key_id = YOUR_ACCESS_KEY
aws_secret_access_key = YOUR_SECRET_KEY
EOF

# Install Velero with AWS plugin
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket my-velero-backups \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1 \
  --secret-file ./credentials-velero \
  --use-node-agent \
  --default-volumes-to-fs-backup

# Verify installation
kubectl -n velero get pods
# NAME                      READY   STATUS    RESTARTS   AGE
# velero-7d9c8b5f4c-x2k9j  1/1     Running   0          30s
# node-agent-abc12          1/1     Running   0          30s
# node-agent-def34          1/1     Running   0          30s

velero version
# Client:
#   Version: v1.13.0
# Server:
#   Version: v1.13.0

# Verify backup location is available
velero backup-location get
# NAME      PROVIDER   BUCKET/PREFIX       PHASE       LAST VALIDATED
# default   aws        my-velero-backups   Available   2024-01-15 14:30:00
```

### 39.2.6 Hands-on: Backup and Restore of a Namespace

```bash
# Step 1: Create a sample application
kubectl create namespace bookstore
kubectl -n bookstore apply -f - <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bookstore-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: bookstore-api
  template:
    metadata:
      labels:
        app: bookstore-api
    spec:
      containers:
        - name: api
          image: nginx:1.25
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: bookstore-api
spec:
  selector:
    app: bookstore-api
  ports:
    - port: 80
      targetPort: 80
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: bookstore-config
data:
  DATABASE_URL: "postgres://db:5432/bookstore"
  CACHE_TTL: "300"
EOF

# Step 2: Verify everything is running
kubectl -n bookstore get all

# Step 3: Create backup
velero backup create bookstore-backup \
  --include-namespaces=bookstore \
  --wait

# Step 4: Check backup status
velero backup describe bookstore-backup --details
# Phase: Completed
# Items backed up: 12

# Step 5: Simulate disaster
kubectl delete namespace bookstore
kubectl get namespace bookstore
# Error from server (NotFound)

# Step 6: Restore
velero restore create --from-backup bookstore-backup --wait

# Step 7: Verify restoration
kubectl -n bookstore get all
# All resources restored: Deployment, Service, ConfigMap, Pods
kubectl -n bookstore get configmap bookstore-config -o yaml
# Data intact
```

### 39.2.7 Hands-on: Scheduled Backups

```bash
# Create tiered backup schedule
# Tier 1: Production — hourly, 7-day retention
velero schedule create prod-hourly \
  --schedule="0 * * * *" \
  --include-namespaces=production \
  --ttl 168h \
  --snapshot-volumes=true \
  --default-volumes-to-fs-backup=true

# Tier 2: Staging — daily, 14-day retention
velero schedule create staging-daily \
  --schedule="0 3 * * *" \
  --include-namespaces=staging \
  --ttl 336h

# Tier 3: Full cluster — weekly, 90-day retention
velero schedule create cluster-weekly \
  --schedule="0 1 * * 0" \
  --include-cluster-resources=true \
  --ttl 2160h

# Monitor schedules
velero schedule get
# NAME             STATUS    CREATED                         SCHEDULE      BACKUP TTL
# prod-hourly      Enabled   2024-01-15 10:00:00 +0000 UTC   0 * * * *    168h
# staging-daily    Enabled   2024-01-15 10:00:00 +0000 UTC   0 3 * * *    336h
# cluster-weekly   Enabled   2024-01-15 10:00:00 +0000 UTC   0 1 * * 0    2160h

# List backups created by schedule
velero backup get --selector velero.io/schedule-name=prod-hourly
```

---

## 39.3 Disaster Recovery Strategies

Backup is only half the equation. A **disaster recovery (DR) strategy** defines
how quickly you can recover (RTO), how much data you can afford to lose (RPO),
and the architectural patterns that make recovery possible.

### 39.3.1 RPO and RTO Definitions

| Metric | Definition                                         | Example                           |
|--------|----------------------------------------------------|------------------------------------|
| **RPO** | Recovery Point Objective — maximum acceptable data loss measured in time | "We can lose at most 1 hour of data" |
| **RTO** | Recovery Time Objective — maximum acceptable downtime | "We must be back online within 30 minutes" |

These two numbers drive every architectural decision:

```
RPO = 0        → Synchronous replication (expensive, complex)
RPO < 1 hour   → Frequent async backups + streaming replication
RPO < 24 hours → Daily backups

RTO = 0        → Active-active with automatic failover
RTO < 15 min   → Active-passive with automated failover
RTO < 4 hours  → Active-passive with manual failover
RTO < 24 hours → Restore from backup
```

### 39.3.2 Active-Passive DR

In an active-passive configuration, one cluster handles all traffic while a
standby cluster remains ready to take over:

```
                   Normal Operation
┌──────────────────────────────────────────┐
│                                          │
│   ┌──────────┐         ┌──────────┐     │
│   │  Active   │  ──────▶│  Passive  │     │
│   │ Cluster A │  async  │ Cluster B │     │
│   │ (primary) │  replic │ (standby) │     │
│   └─────┬────┘         └──────────┘     │
│         │                                │
│   ┌─────▼────┐                           │
│   │   DNS /   │                           │
│   │ Load Bal  │                           │
│   └──────────┘                           │
└──────────────────────────────────────────┘

                   During Failover
┌──────────────────────────────────────────┐
│                                          │
│   ┌──────────┐         ┌──────────┐     │
│   │  Active   │  ✗      │  Passive  │     │
│   │ Cluster A │         │ Cluster B │     │
│   │  (DOWN)   │         │ (promoted)│     │
│   └──────────┘         └─────┬────┘     │
│                              │           │
│                        ┌─────▼────┐     │
│                        │   DNS /   │     │
│                        │ Load Bal  │     │
│                        └──────────┘     │
└──────────────────────────────────────────┘
```

**Implementation checklist:**

1. Deploy identical infrastructure in the standby region.
2. Continuously replicate data (etcd snapshots, database replication, Velero).
3. Use DNS-based failover (Route 53 health checks, Cloudflare Load Balancing).
4. Automate the failover process; document manual steps as fallback.
5. Test failover monthly.

### 39.3.3 Active-Active DR

Both clusters serve traffic simultaneously. This eliminates failover time but
adds significant complexity:

```
┌─────────────────────────────────────────────┐
│            Active-Active Topology            │
│                                              │
│   ┌──────────┐         ┌──────────┐        │
│   │ Cluster A │◀───────▶│ Cluster B │        │
│   │ Region 1  │  sync   │ Region 2  │        │
│   └─────┬────┘         └─────┬────┘        │
│         │                     │              │
│   ┌─────▼─────────────────────▼────┐        │
│   │     Global Load Balancer        │        │
│   │  (geo-routing, health checks)   │        │
│   └────────────────────────────────┘        │
└─────────────────────────────────────────────┘
```

**Challenges:**

- **Data consistency**: Split-brain risk when both clusters write to the same data.
- **Conflict resolution**: Need a strategy (last-writer-wins, CRDTs, application-level).
- **Stateful workloads**: Databases need multi-master replication (CockroachDB, Vitess).
- **Cost**: 2× infrastructure running at all times.

### 39.3.4 Cross-Region DR

For maximum resilience, deploy clusters across cloud regions:

| Approach                  | RPO       | RTO       | Cost    | Complexity |
|--------------------------|-----------|-----------|---------|------------|
| Backup to remote storage | Hours     | Hours     | Low     | Low        |
| Async replication        | Minutes   | Minutes   | Medium  | Medium     |
| Sync replication         | Zero      | Seconds   | High    | High       |
| Active-active            | Near-zero | Near-zero | Highest | Highest    |

### 39.3.5 DR Testing and Runbooks

A DR plan that hasn't been tested is a DR plan that doesn't work.

**DR testing cadence:**

| Test Type            | Frequency | Scope                              |
|---------------------|-----------|-------------------------------------|
| Backup verification | Daily     | Verify backup completeness          |
| Component failover  | Weekly    | Kill one etcd member, one node      |
| Namespace restore   | Monthly   | Restore a namespace from backup     |
| Full DR drill       | Quarterly | Complete failover to standby cluster|
| Chaos day           | Bi-annual | Unannounced failure injection       |

**Runbook template:**

```markdown
# DR Runbook: Complete Cluster Failover

## Prerequisites
- Access to standby cluster kubeconfig
- DNS management credentials
- Communication channel (Slack #incidents)

## Steps

### 1. Declare Incident (2 min)
- [ ] Page on-call team
- [ ] Open incident channel
- [ ] Assign incident commander

### 2. Assess Damage (5 min)
- [ ] Verify primary cluster is unreachable
- [ ] Check cloud provider status page
- [ ] Determine scope (full outage vs partial)

### 3. Activate Standby (10 min)
- [ ] Verify standby cluster health: kubectl --kubeconfig=standby get nodes
- [ ] Apply latest Velero restore if needed
- [ ] Verify critical services are running
- [ ] Run smoke tests against standby

### 4. Switch Traffic (5 min)
- [ ] Update DNS to point to standby cluster
- [ ] Update CDN origin to standby ingress
- [ ] Verify traffic is flowing to standby

### 5. Verify (10 min)
- [ ] Monitor error rates in standby
- [ ] Check database connectivity
- [ ] Verify external integrations

### 6. Communicate (ongoing)
- [ ] Update status page
- [ ] Notify stakeholders
- [ ] Document timeline

## Rollback
If standby cluster has issues, revert DNS to primary (if recovered).

## Post-Incident
- [ ] Root cause analysis within 48 hours
- [ ] Update runbook with lessons learned
```

---

## 39.4 Multi-Cluster Management

As organizations grow, a single cluster becomes a bottleneck. Multi-cluster
architectures solve problems that no amount of namespaces can address.

### 39.4.1 Why Multi-Cluster?

| Motivation              | Description                                                     |
|------------------------|-----------------------------------------------------------------|
| **Blast radius**       | A bad deploy or etcd corruption affects only one cluster        |
| **Compliance**         | Data residency laws require clusters in specific regions        |
| **Latency**            | Edge clusters serve users closer to their location              |
| **Team isolation**     | Different teams get independent clusters with full autonomy     |
| **Scaling limits**     | Single clusters hit limits (~5,000 nodes, ~150,000 pods)        |
| **Vendor diversity**   | Run on multiple clouds to avoid lock-in                         |
| **Environment separation** | Dev, staging, production on separate clusters               |

### 39.4.2 Cluster API — Managing Cluster Lifecycle

**Cluster API (CAPI)** is a Kubernetes sub-project that uses Kubernetes-style
declarative APIs to create, configure, and manage clusters:

```yaml
# cluster.yaml — Declares a cluster using Cluster API
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: production-east
  namespace: clusters
spec:
  clusterNetwork:
    pods:
      cidrBlocks: ["192.168.0.0/16"]
    services:
      cidrBlocks: ["10.96.0.0/12"]
  controlPlaneRef:
    apiVersion: controlplane.cluster.x-k8s.io/v1beta1
    kind: KubeadmControlPlane
    name: production-east-cp
  infrastructureRef:
    apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
    kind: AWSCluster
    name: production-east
---
apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
kind: AWSCluster
metadata:
  name: production-east
  namespace: clusters
spec:
  region: us-east-1
  sshKeyName: my-key
---
apiVersion: controlplane.cluster.x-k8s.io/v1beta1
kind: KubeadmControlPlane
metadata:
  name: production-east-cp
  namespace: clusters
spec:
  replicas: 3
  version: v1.29.0
  machineTemplate:
    infrastructureRef:
      apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
      kind: AWSMachineTemplate
      name: production-east-cp
---
apiVersion: cluster.x-k8s.io/v1beta1
kind: MachineDeployment
metadata:
  name: production-east-workers
  namespace: clusters
spec:
  clusterName: production-east
  replicas: 5
  selector:
    matchLabels: {}
  template:
    spec:
      clusterName: production-east
      version: v1.29.0
      bootstrap:
        configRef:
          apiVersion: bootstrap.cluster.x-k8s.io/v1beta1
          kind: KubeadmConfigTemplate
          name: production-east-workers
      infrastructureRef:
        apiVersion: infrastructure.cluster.x-k8s.io/v1beta1
        kind: AWSMachineTemplate
        name: production-east-workers
```

**CAPI workflow:**

1. A **management cluster** runs Cluster API controllers.
2. You apply Cluster manifests to the management cluster.
3. CAPI provisions infrastructure (VMs, networks) via provider plugins.
4. CAPI bootstraps Kubernetes on the infrastructure.
5. The resulting **workload cluster** is ready for use.

### 39.4.3 Federation Concepts

Kubernetes federation enables deploying resources across multiple clusters:

**KubeFed (Kubernetes Federation v2):**

```yaml
# Federated Deployment — replicated across member clusters
apiVersion: types.kubefed.io/v1beta1
kind: FederatedDeployment
metadata:
  name: nginx
  namespace: production
spec:
  template:
    metadata:
      labels:
        app: nginx
    spec:
      replicas: 3
      selector:
        matchLabels:
          app: nginx
      template:
        metadata:
          labels:
            app: nginx
        spec:
          containers:
            - name: nginx
              image: nginx:1.25
  placement:
    clusters:
      - name: cluster-east
      - name: cluster-west
  overrides:
    - clusterName: cluster-east
      clusterOverrides:
        - path: "/spec/replicas"
          value: 5
    - clusterName: cluster-west
      clusterOverrides:
        - path: "/spec/replicas"
          value: 3
```

**Modern alternatives to KubeFed:**

- **Argo CD ApplicationSets**: GitOps-based multi-cluster deployment.
- **Flux CD**: Multi-cluster with Kustomize overlays per cluster.
- **Open Cluster Management (OCM)**: CNCF project for multi-cluster lifecycle.
- **Karmada**: Kubernetes management across clouds and clusters.

### 39.4.4 Service Mesh Across Clusters (Istio Multi-Cluster)

Istio supports multiple multi-cluster topologies:

**Primary-Remote (shared control plane):**

```
┌──────────────────────────────────────────────────┐
│                                                    │
│   ┌──────────────────┐   ┌──────────────────┐    │
│   │   Cluster 1       │   │   Cluster 2       │    │
│   │   (Primary)       │   │   (Remote)        │    │
│   │                    │   │                    │    │
│   │  ┌────────────┐  │   │  ┌────────────┐   │    │
│   │  │   istiod    │──┼───┼─▶│  istio-agent│   │    │
│   │  └────────────┘  │   │  └────────────┘   │    │
│   │  ┌────────────┐  │   │  ┌────────────┐   │    │
│   │  │  Service A  │◀─┼───┼──│  Service B  │   │    │
│   │  └────────────┘  │   │  └────────────┘   │    │
│   └──────────────────┘   └──────────────────┘    │
│                                                    │
│   East-West Gateway enables cross-cluster traffic  │
└──────────────────────────────────────────────────┘
```

**Setup steps:**

```bash
# Install Istio on primary cluster
istioctl install --set profile=demo \
  --set values.global.meshID=mesh1 \
  --set values.global.multiCluster.clusterName=cluster1 \
  --set values.global.network=network1 \
  --context="${CTX_CLUSTER1}"

# Install east-west gateway
samples/multicluster/gen-eastwest-gateway.sh \
  --mesh mesh1 --cluster cluster1 --network network1 | \
  istioctl install -y --context="${CTX_CLUSTER1}" -f -

# Expose services to other clusters
kubectl --context="${CTX_CLUSTER1}" apply -n istio-system \
  -f samples/multicluster/expose-services.yaml

# Create remote secret for cluster2
istioctl create-remote-secret \
  --context="${CTX_CLUSTER2}" \
  --name=cluster2 | \
  kubectl apply -f - --context="${CTX_CLUSTER1}"

# Install Istio on remote cluster
istioctl install --set profile=remote \
  --set values.global.meshID=mesh1 \
  --set values.global.multiCluster.clusterName=cluster2 \
  --set values.global.network=network2 \
  --set values.istiodRemote.injectionURL=https://<istiod-ip>:15017/inject \
  --context="${CTX_CLUSTER2}"
```

### 39.4.5 Multi-Cluster Service Discovery

For services to find each other across clusters, you need a cross-cluster
service discovery mechanism:

**Option 1: Kubernetes Multi-Cluster Services (MCS) API:**

```yaml
# Export a service from cluster-east
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceExport
metadata:
  name: my-service
  namespace: production
---
# Import in cluster-west (auto-discovered)
apiVersion: multicluster.x-k8s.io/v1alpha1
kind: ServiceImport
metadata:
  name: my-service
  namespace: production
spec:
  type: ClusterSetIP
  ports:
    - port: 80
      protocol: TCP
```

**Option 2: External DNS with service annotations:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: api-service
  annotations:
    external-dns.alpha.kubernetes.io/hostname: api.global.example.com
spec:
  type: LoadBalancer
  ports:
    - port: 443
```

**Option 3: Consul service mesh:**

Consul provides a service catalog that spans clusters and cloud providers,
with built-in service discovery, health checking, and intention-based
authorization.

---

## 39.5 Hands-on: Go Program That Manages Backups Across Clusters

Below is a Go program that orchestrates Velero backups across multiple
Kubernetes clusters. It connects to each cluster, triggers backups, monitors
their status, and reports results.

```go
// Package main implements a multi-cluster backup manager that orchestrates
// Velero backups across a fleet of Kubernetes clusters. It connects to each
// cluster using its kubeconfig, triggers a Velero backup, polls for completion,
// and reports the results.
//
// Usage:
//
//	go run main.go --config clusters.yaml
//
// The clusters.yaml file defines the list of clusters and their kubeconfig paths.
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"sync"
	"time"

	"gopkg.in/yaml.v3"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/tools/clientcmd"
)

// ClusterConfig holds the configuration for a single Kubernetes cluster
// that participates in the multi-cluster backup strategy. Each cluster
// is identified by a human-readable name and a path to its kubeconfig file.
type ClusterConfig struct {
	// Name is the human-readable identifier for this cluster (e.g., "prod-east").
	Name string `yaml:"name"`

	// KubeconfigPath is the absolute path to the kubeconfig file that grants
	// access to this cluster's API server.
	KubeconfigPath string `yaml:"kubeconfigPath"`

	// Namespaces lists the namespaces to include in the backup. If empty,
	// all namespaces are backed up.
	Namespaces []string `yaml:"namespaces"`

	// IncludeVolumes controls whether persistent volumes are included in
	// the backup via Velero's file-system backup (Restic/Kopia).
	IncludeVolumes bool `yaml:"includeVolumes"`
}

// BackupManagerConfig is the top-level configuration file that defines
// all clusters and global backup settings.
type BackupManagerConfig struct {
	// Clusters is the list of all clusters to manage.
	Clusters []ClusterConfig `yaml:"clusters"`

	// BackupTTL is the retention period for backups, expressed as a Go
	// duration string (e.g., "720h" for 30 days).
	BackupTTL string `yaml:"backupTTL"`

	// BackupPrefix is prepended to all backup names for easy identification
	// (e.g., "nightly" produces "nightly-prod-east-20240115-143000").
	BackupPrefix string `yaml:"backupPrefix"`
}

// BackupResult captures the outcome of a backup operation on a single cluster.
// It records whether the backup succeeded, how long it took, and any error
// that occurred during the process.
type BackupResult struct {
	// ClusterName identifies which cluster this result belongs to.
	ClusterName string `json:"clusterName"`

	// BackupName is the name of the Velero Backup resource that was created.
	BackupName string `json:"backupName"`

	// Phase is the final phase of the backup (Completed, PartiallyFailed, Failed).
	Phase string `json:"phase"`

	// Duration records how long the backup took from creation to completion.
	Duration time.Duration `json:"duration"`

	// ItemsBackedUp is the count of Kubernetes resources included in the backup.
	ItemsBackedUp int64 `json:"itemsBackedUp"`

	// Error captures any error that prevented the backup from completing.
	Error string `json:"error,omitempty"`
}

// veleroBackupGVR is the GroupVersionResource for Velero Backup custom resources.
// This allows us to use the dynamic client to create and monitor Velero Backups
// without importing the entire Velero client library.
var veleroBackupGVR = schema.GroupVersionResource{
	Group:    "velero.io",
	Version:  "v1",
	Resource: "backups",
}

// createDynamicClient builds a Kubernetes dynamic client from a kubeconfig file.
// The dynamic client allows us to work with any resource type, including CRDs
// like Velero Backups, without generating typed client code.
func createDynamicClient(kubeconfigPath string) (dynamic.Interface, error) {
	config, err := clientcmd.BuildConfigFromFlags("", kubeconfigPath)
	if err != nil {
		return nil, fmt.Errorf("building kubeconfig from %s: %w", kubeconfigPath, err)
	}

	// Set reasonable timeouts to avoid hanging on unreachable clusters.
	config.Timeout = 30 * time.Second

	client, err := dynamic.NewForConfig(config)
	if err != nil {
		return nil, fmt.Errorf("creating dynamic client: %w", err)
	}

	return client, nil
}

// triggerBackup creates a Velero Backup custom resource in the target cluster.
// It constructs the backup spec based on the cluster configuration, including
// namespace filters and volume backup settings. Returns the name of the created
// backup resource.
func triggerBackup(
	ctx context.Context,
	client dynamic.Interface,
	cluster ClusterConfig,
	backupName string,
	ttl string,
) error {
	// Build the Velero Backup spec as an unstructured object. This avoids
	// importing the Velero API types and works with any Velero version.
	backup := &unstructured.Unstructured{
		Object: map[string]interface{}{
			"apiVersion": "velero.io/v1",
			"kind":       "Backup",
			"metadata": map[string]interface{}{
				"name":      backupName,
				"namespace": "velero",
				"labels": map[string]interface{}{
					"managed-by":   "backup-manager",
					"cluster-name": cluster.Name,
				},
			},
			"spec": map[string]interface{}{
				"ttl":                          ttl,
				"defaultVolumesToFsBackup":     cluster.IncludeVolumes,
				"includedNamespaces":           cluster.Namespaces,
				"storageLocation":              "default",
				"volumeSnapshotLocations":      []interface{}{"default"},
			},
		},
	}

	// If no specific namespaces are configured, remove the filter to back up
	// all namespaces (Velero's default behavior).
	if len(cluster.Namespaces) == 0 {
		spec := backup.Object["spec"].(map[string]interface{})
		delete(spec, "includedNamespaces")
	}

	_, err := client.Resource(veleroBackupGVR).Namespace("velero").Create(
		ctx, backup, metav1.CreateOptions{},
	)
	return err
}

// waitForBackup polls the Velero Backup resource until it reaches a terminal
// phase (Completed, PartiallyFailed, Failed) or the context is cancelled.
// It checks every 10 seconds and returns the final phase and item count.
func waitForBackup(
	ctx context.Context,
	client dynamic.Interface,
	backupName string,
) (string, int64, error) {
	ticker := time.NewTicker(10 * time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return "", 0, ctx.Err()
		case <-ticker.C:
			// Fetch the current state of the Backup resource.
			backup, err := client.Resource(veleroBackupGVR).Namespace("velero").Get(
				ctx, backupName, metav1.GetOptions{},
			)
			if err != nil {
				return "", 0, fmt.Errorf("getting backup status: %w", err)
			}

			// Extract the phase from the status sub-resource.
			phase, found, err := unstructured.NestedString(backup.Object, "status", "phase")
			if err != nil || !found {
				continue // Status not yet populated; keep polling.
			}

			// Check for terminal phases.
			switch phase {
			case "Completed", "PartiallyFailed", "Failed":
				items, _, _ := unstructured.NestedInt64(
					backup.Object, "status", "progress", "itemsBackedUp",
				)
				return phase, items, nil
			}
		}
	}
}

// backupCluster performs the full backup workflow for a single cluster:
// connect, trigger, wait, and report. It is designed to be run as a goroutine
// so that multiple clusters can be backed up concurrently.
func backupCluster(
	ctx context.Context,
	cluster ClusterConfig,
	backupName string,
	ttl string,
) BackupResult {
	result := BackupResult{
		ClusterName: cluster.Name,
		BackupName:  backupName,
	}

	start := time.Now()

	// Step 1: Connect to the cluster.
	client, err := createDynamicClient(cluster.KubeconfigPath)
	if err != nil {
		result.Error = fmt.Sprintf("connection failed: %v", err)
		result.Phase = "ConnectionError"
		return result
	}

	// Step 2: Create the Velero Backup resource.
	if err := triggerBackup(ctx, client, cluster, backupName, ttl); err != nil {
		result.Error = fmt.Sprintf("backup creation failed: %v", err)
		result.Phase = "CreateError"
		return result
	}

	log.Printf("[%s] Backup %s created, waiting for completion...", cluster.Name, backupName)

	// Step 3: Wait for the backup to complete (with a 30-minute timeout).
	backupCtx, cancel := context.WithTimeout(ctx, 30*time.Minute)
	defer cancel()

	phase, items, err := waitForBackup(backupCtx, client, backupName)
	if err != nil {
		result.Error = fmt.Sprintf("waiting for backup: %v", err)
		result.Phase = "Timeout"
		return result
	}

	result.Phase = phase
	result.ItemsBackedUp = items
	result.Duration = time.Since(start)

	return result
}

// loadConfig reads and parses the backup manager configuration file.
// The file must be valid YAML conforming to the BackupManagerConfig schema.
func loadConfig(path string) (*BackupManagerConfig, error) {
	data, err := os.ReadFile(path)
	if err != nil {
		return nil, fmt.Errorf("reading config file: %w", err)
	}

	var config BackupManagerConfig
	if err := yaml.Unmarshal(data, &config); err != nil {
		return nil, fmt.Errorf("parsing config file: %w", err)
	}

	return &config, nil
}

func main() {
	// Load configuration.
	configPath := "clusters.yaml"
	if len(os.Args) > 1 {
		configPath = os.Args[1]
	}

	config, err := loadConfig(configPath)
	if err != nil {
		log.Fatalf("Failed to load config: %v", err)
	}

	// Generate a timestamp for this backup run.
	timestamp := time.Now().UTC().Format("20060102-150405")

	// Launch backups concurrently across all clusters.
	ctx := context.Background()
	var wg sync.WaitGroup
	results := make([]BackupResult, len(config.Clusters))

	for i, cluster := range config.Clusters {
		wg.Add(1)
		go func(idx int, c ClusterConfig) {
			defer wg.Done()
			backupName := fmt.Sprintf("%s-%s-%s", config.BackupPrefix, c.Name, timestamp)
			results[idx] = backupCluster(ctx, c, backupName, config.BackupTTL)
		}(i, cluster)
	}

	wg.Wait()

	// Print results as a formatted report.
	fmt.Println("\n" + "=" + " Multi-Cluster Backup Report " + "=")
	fmt.Printf("Timestamp: %s\n\n", timestamp)

	allSuccess := true
	for _, r := range results {
		status := "✓"
		if r.Phase != "Completed" {
			status = "✗"
			allSuccess = false
		}
		fmt.Printf("[%s] Cluster: %-20s Backup: %-40s Phase: %-20s Items: %d  Duration: %s\n",
			status, r.ClusterName, r.BackupName, r.Phase, r.ItemsBackedUp, r.Duration.Round(time.Second))
		if r.Error != "" {
			fmt.Printf("     Error: %s\n", r.Error)
		}
	}

	// Output JSON for programmatic consumption.
	jsonData, _ := json.MarshalIndent(results, "", "  ")
	os.WriteFile(fmt.Sprintf("backup-report-%s.json", timestamp), jsonData, 0644)

	if !allSuccess {
		fmt.Println("\n⚠ Some backups failed. Check the report above.")
		os.Exit(1)
	}

	fmt.Println("\n✓ All backups completed successfully.")
}
```

**Example `clusters.yaml`:**

```yaml
# clusters.yaml — Configuration for the multi-cluster backup manager.
# Each entry describes a cluster that should be included in the backup run.
backupPrefix: "nightly"
backupTTL: "720h"   # 30 days

clusters:
  - name: prod-east
    kubeconfigPath: /home/ops/.kube/prod-east.yaml
    namespaces:
      - production
      - monitoring
    includeVolumes: true

  - name: prod-west
    kubeconfigPath: /home/ops/.kube/prod-west.yaml
    namespaces:
      - production
      - monitoring
    includeVolumes: true

  - name: staging
    kubeconfigPath: /home/ops/.kube/staging.yaml
    namespaces: []  # All namespaces
    includeVolumes: false

  - name: dev
    kubeconfigPath: /home/ops/.kube/dev.yaml
    namespaces:
      - development
    includeVolumes: false
```

---

## 39.6 Cluster Migration

Moving workloads between clusters is a common operation during upgrades,
cloud migrations, or disaster recovery scenarios.

### 39.6.1 Moving Workloads Between Clusters

**Migration strategies:**

| Strategy               | Downtime | Complexity | Data Loss Risk |
|-----------------------|----------|------------|----------------|
| Blue-green cluster    | Minimal  | Medium     | Low            |
| Velero backup/restore | Minutes  | Low        | Low            |
| GitOps redeploy       | Minutes  | Low        | None (stateless)|
| Live migration        | Zero     | High       | Low            |

**Blue-green cluster migration:**

1. Stand up new cluster (green) alongside old cluster (blue).
2. Deploy all workloads to green cluster using GitOps.
3. Migrate stateful data (database replication, Velero volume restore).
4. Run smoke tests on green cluster.
5. Switch DNS/load balancer from blue to green.
6. Monitor green cluster for issues.
7. Decommission blue cluster after confidence period.

### 39.6.2 Velero Cross-Cluster Migration

Velero's backup/restore works across clusters as long as both clusters
share the same BackupStorageLocation:

```bash
# On source cluster: Create backup
velero backup create migration-backup \
  --include-namespaces=myapp \
  --snapshot-volumes=true \
  --default-volumes-to-fs-backup=true \
  --wait

# Verify backup completed
velero backup describe migration-backup

# On destination cluster: Configure same BSL
velero backup-location create shared-bsl \
  --provider aws \
  --bucket my-velero-backups \
  --config region=us-east-1

# On destination cluster: Verify backup is visible
velero backup get
# NAME                STATUS      CREATED                         EXPIRES
# migration-backup    Completed   2024-01-15 14:30:00 +0000 UTC   29d

# On destination cluster: Restore
velero restore create --from-backup migration-backup --wait

# Verify restoration
kubectl -n myapp get all
```

**Handling differences between clusters:**

```bash
# Change storage class during restore
velero restore create --from-backup migration-backup \
  --storage-class-mappings gp2:gp3

# Change namespace during restore
velero restore create --from-backup migration-backup \
  --namespace-mappings old-ns:new-ns

# Restore only specific resources
velero restore create --from-backup migration-backup \
  --include-resources deployments,services,configmaps
```

### 39.6.3 Hands-on: Migrating an Application Between Clusters

Let's walk through a complete application migration:

```bash
# ── Source Cluster ──

# Step 1: Verify application is running
kubectl --context=source -n webapp get all
# deployment.apps/webapp   3/3   3   3   Running
# service/webapp           ClusterIP   10.96.0.100   80/TCP

# Step 2: Create annotated backup
kubectl --context=source -n webapp annotate pod --all \
  backup.velero.io/backup-volumes=data-volume

velero --kubeconfig=source backup create webapp-migration \
  --include-namespaces=webapp \
  --default-volumes-to-fs-backup=true \
  --wait

# Step 3: Verify backup
velero --kubeconfig=source backup describe webapp-migration --details
# Phase: Completed
# Namespaces included: webapp
# Resources included: *
# Cluster-scoped: auto
# Volume Snapshots: 3 completed

# ── Destination Cluster ──

# Step 4: Ensure BSL is configured and synced
velero --kubeconfig=dest backup-location get
velero --kubeconfig=dest backup get
# webapp-migration should appear

# Step 5: Pre-migration checks on destination
kubectl --context=dest get namespace webapp
# Error: not found (good — clean slate)

# Step 6: Restore on destination
velero --kubeconfig=dest restore create webapp-restore \
  --from-backup webapp-migration \
  --wait

# Step 7: Verify migration
kubectl --context=dest -n webapp get all
kubectl --context=dest -n webapp get pvc
kubectl --context=dest -n webapp logs deployment/webapp --tail=20

# Step 8: Run smoke tests
kubectl --context=dest -n webapp port-forward svc/webapp 8080:80 &
curl http://localhost:8080/health
# {"status":"healthy"}

# Step 9: Switch DNS
# Update DNS record to point to destination cluster's ingress IP

# Step 10: Decommission on source (after confidence period)
kubectl --context=source delete namespace webapp
```

---

## 39.7 Summary

| Topic                    | Key Takeaway                                                              |
|--------------------------|---------------------------------------------------------------------------|
| etcd backup              | `etcdctl snapshot save` — the foundation of cluster state recovery        |
| Velero                   | Application-aware backup with volume support and cross-cluster restore    |
| RPO/RTO                  | Define these numbers first; they drive all architectural decisions         |
| Active-passive DR        | Standby cluster with automated failover; test monthly                     |
| Active-active DR         | Both clusters serve traffic; complex but eliminates failover time         |
| Multi-cluster motivation | Blast radius, compliance, latency, scaling limits                         |
| Cluster API              | Declarative cluster lifecycle management using Kubernetes resources        |
| Service mesh multi-cluster| Istio east-west gateways enable transparent cross-cluster communication |
| Cluster migration        | Velero backup/restore across shared BSLs; blue-green for zero downtime   |

**Golden rules:**

1. **Back up etcd** at least hourly in production.
2. **Back up applications** with Velero — etcd snapshots alone are not enough.
3. **Test your restores.** A backup you've never restored is not a backup.
4. **Define RPO/RTO** before designing your DR strategy.
5. **Automate failover** but maintain manual runbooks as fallback.
6. **Multi-cluster is not optional** at scale — plan for it early.

---

## 39.8 Exercises

1. **etcd Backup Automation**: Write a CronJob that backs up etcd every hour,
   uploads to S3, and sends a Slack notification on failure.

2. **Velero DR Drill**: Install Velero, create a stateful application with a
   PVC, back it up, delete everything, and restore. Measure your RTO.

3. **Multi-Cluster Backup Manager**: Extend the Go program in Section 39.5 to:
   - Send Slack notifications on backup failure.
   - Implement backup verification (restore to a temporary namespace and validate).
   - Add Prometheus metrics for backup duration and success rate.

4. **Cross-Cluster Migration**: Set up two Kind clusters, deploy an application
   with a PVC to cluster A, migrate it to cluster B using Velero, and verify
   data integrity.

5. **DR Runbook**: Write a complete DR runbook for your organization's most
   critical application. Include communication templates, escalation paths,
   and post-incident review procedures.
