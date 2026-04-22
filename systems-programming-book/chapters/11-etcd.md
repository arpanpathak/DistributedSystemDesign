# Chapter 11: etcd — The Brain of Kubernetes

> *"etcd is to Kubernetes what the hippocampus is to the human brain —
> without it, no memory, no coordination, no identity."*

Every Kubernetes cluster, from the smallest single-node lab to the largest
production fleet spanning thousands of nodes, relies on a single source of
truth: **etcd**. It is the persistent store that holds every object definition,
every configuration value, every secret, and every piece of state that makes
a Kubernetes cluster what it is. If the API server is the nervous system, etcd
is the brain — the place where all memories are formed and recalled.

In this chapter we will go deep. We will understand the theory that makes etcd
correct (the Raft consensus protocol), the engineering that makes it fast (MVCC,
BoltDB, WAL), the API that makes it useful (KV, Watch, Lease, Txn), and the
operational practices that keep it alive in production. Every concept will be
accompanied by hands-on Go code with generous documentation.

Let us begin.

---

## 11.1 What Is etcd?

**etcd** (pronounced "et-see-dee", from the Unix `/etc` directory + "d" for
distributed) is a distributed, reliable key-value store written in Go. It was
created by CoreOS in 2013 and is now a graduated project of the Cloud Native
Computing Foundation (CNCF).

### Core Properties

| Property              | Description                                                  |
|-----------------------|--------------------------------------------------------------|
| **Consistency model** | Strongly consistent (linearizable by default)                |
| **Consensus**         | Raft protocol                                                |
| **Data model**        | Flat key-value with multi-version concurrency control (MVCC) |
| **Watch support**     | Streaming watch on key ranges with revision history          |
| **Storage engine**    | bbolt (fork of BoltDB) — a B+tree on-disk store              |
| **Transport**         | gRPC over HTTP/2 with TLS                                    |
| **Language**          | Go                                                           |

### Why etcd?

Before etcd, distributed coordination relied on systems like Apache ZooKeeper
(Java, complex API, heavyweight) or Consul (broader scope, eventual consistency
by default for KV). etcd carved out a niche by being:

1. **Simple** — a flat key-value API, no hierarchical znodes.
2. **Strongly consistent** — every read can be linearizable.
3. **Watch-friendly** — first-class support for streaming change notifications.
4. **Small and embeddable** — a single Go binary, easy to deploy.
5. **gRPC-native** — efficient, strongly-typed client communication.

Kubernetes chose etcd as its **only** persistent store. The entire cluster state
— pods, services, deployments, secrets, config maps, RBAC rules, custom
resources — lives in etcd. Nothing else. This makes etcd the single most
critical component in any Kubernetes deployment.

### The /etc Analogy

In Unix systems, `/etc` holds system configuration files. etcd generalizes this
idea to a distributed cluster: it is the `/etc` for your distributed system,
replicated across multiple nodes for fault tolerance.

---

## 11.2 Raft Consensus Protocol — Deep Dive

To understand etcd, you must understand Raft. Raft is the consensus algorithm
that ensures all etcd nodes agree on the same sequence of operations, even when
some nodes fail. It was designed by Diego Ongaro and John Ousterhout in 2014
specifically to be **understandable** — a reaction to the notoriously difficult
Paxos algorithm.

### 11.2.1 The Problem: Distributed Consensus

Imagine three servers that need to maintain identical copies of a key-value
store. Clients can send writes to any server. The fundamental problem:

> **How do N servers agree on a single, ordered sequence of operations, even
> when some servers crash or the network partitions?**

This is the **distributed consensus problem**. It is provably impossible to
solve in an asynchronous system where nodes can crash (the FLP impossibility
result). Raft sidesteps this by using **timeouts** — it assumes the system is
*partially synchronous* (messages eventually arrive within some bound).

### 11.2.2 Raft Basics: Roles and Terms

Every Raft node is in one of three states at any time:

```text
┌────────────┐     timeout     ┌─────────────┐    majority votes    ┌────────┐
│  FOLLOWER  │ ──────────────► │  CANDIDATE   │ ──────────────────► │ LEADER │
│            │                 │              │                     │        │
└────────────┘                 └─────────────┘                     └────────┘
      ▲                              │                                  │
      │         discovers leader     │       discovers higher term      │
      │         or higher term       │                                  │
      │◄─────────────────────────────┘◄─────────────────────────────────┘
```

**Terms** are logical clocks. Each term has at most one leader. When a node
starts an election, it increments its term. Terms allow nodes to detect stale
leaders.

```text
Term 1          Term 2          Term 3          Term 4
┌──────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
│ Leader: A│   │ Leader: B│   │ Election │   │ Leader: C│
│          │   │          │   │ (no win) │   │          │
└──────────┘   └──────────┘   └──────────┘   └──────────┘
```

### 11.2.3 Leader Election

The leader election process works as follows:

1. All nodes start as **followers**.
2. Each follower has a randomized **election timeout** (typically 150–300ms).
3. If a follower receives no heartbeat from a leader before its timeout, it
   becomes a **candidate**.
4. The candidate increments its term, votes for itself, and sends
   `RequestVote` RPCs to all other nodes.
5. Each node votes for at most one candidate per term (first-come-first-served).
6. If the candidate receives votes from a **majority** (⌊N/2⌋ + 1), it becomes
   the **leader**.
7. The leader immediately sends heartbeats (empty `AppendEntries` RPCs) to all
   followers to assert its authority.

**Split brain prevention**: Because a candidate needs a **majority** of votes,
and each node votes for at most one candidate per term, at most one leader can
be elected per term. This is a fundamental safety guarantee.

**Randomized timeouts**: If two candidates start elections simultaneously, they
might split the vote. Raft resolves this by randomizing election timeouts, so
one candidate almost always times out first in the next round.

#### ASCII Diagram: Leader Election Flow

```text
Time ──────────────────────────────────────────────────────────────────►

Node A (Follower)          Node B (Follower)          Node C (Follower)
    │                          │                          │
    │  election timeout fires  │                          │
    │                          │                          │
    ├─── becomes Candidate ────┤                          │
    │    term = 2              │                          │
    │    votes for self        │                          │
    │                          │                          │
    │── RequestVote(term=2) ──►│                          │
    │── RequestVote(term=2) ──────────────────────────────►
    │                          │                          │
    │                          │  grants vote             │
    │◄── VoteGranted ──────────┤  (hasn't voted           │
    │                          │   in term 2)             │
    │                          │                          │  grants vote
    │◄── VoteGranted ────────────────────────────────────┤
    │                          │                          │
    │  has 3/3 votes           │                          │
    │  BECOMES LEADER          │                          │
    │                          │                          │
    │── AppendEntries ────────►│  (heartbeat)             │
    │── AppendEntries ─────────────────────────────────────►
    │                          │                          │
    ▼                          ▼                          ▼
```

### 11.2.4 Log Replication

Once a leader is elected, all client writes go through it. The leader appends
each write as a new **log entry** and replicates it to followers:

1. Client sends a write request to the leader.
2. Leader appends the entry to its local log.
3. Leader sends `AppendEntries` RPC to all followers with the new entry.
4. Each follower appends the entry to its log and responds with success.
5. Once the leader receives acknowledgments from a **majority**, the entry is
   **committed**.
6. The leader applies the committed entry to its state machine and responds
   to the client.
7. Followers learn of the commit via subsequent `AppendEntries` RPCs (the
   leader includes its `commitIndex`).

#### Key Indices

| Index          | Description                                                    |
|----------------|----------------------------------------------------------------|
| `commitIndex`  | Highest log entry known to be committed (replicated to majority) |
| `lastApplied`  | Highest log entry applied to the state machine                 |
| `nextIndex[]`  | For each follower: next log entry to send (leader tracks this) |
| `matchIndex[]` | For each follower: highest entry known to be replicated        |

#### ASCII Diagram: Log Replication Flow

```text
Client          Leader (A)         Follower (B)       Follower (C)
  │                 │                    │                   │
  │── Put(x=42) ──►│                    │                   │
  │                 │                    │                   │
  │                 │── append to log    │                   │
  │                 │   [idx=5, term=3,  │                   │
  │                 │    Put(x=42)]      │                   │
  │                 │                    │                   │
  │                 │── AppendEntries ──►│                   │
  │                 │   (entries=[5],    │                   │
  │                 │    prevIdx=4,      │                   │
  │                 │    prevTerm=3,     │                   │
  │                 │    commitIdx=4)    │                   │
  │                 │                    │                   │
  │                 │── AppendEntries ───────────────────────►
  │                 │                    │                   │
  │                 │                    │── append to log   │
  │                 │◄── Success ────────┤   [idx=5]        │
  │                 │                    │                   │
  │                 │                    │                   │── append to log
  │                 │◄── Success ─────────────────────────────┤   [idx=5]
  │                 │                    │                   │
  │                 │  majority ack'd    │                   │
  │                 │  commit idx=5      │                   │
  │                 │  apply to state    │                   │
  │                 │  machine           │                   │
  │                 │                    │                   │
  │◄── OK ─────────┤                    │                   │
  │                 │                    │                   │
  ▼                 ▼                    ▼                   ▼
```

### 11.2.5 Safety Guarantees

Raft provides five key safety properties:

1. **Election Safety**: At most one leader per term. Guaranteed by majority
   voting and one-vote-per-term rule.

2. **Leader Append-Only**: A leader never overwrites or deletes entries in its
   log. It only appends new entries.

3. **Log Matching**: If two logs contain an entry with the same index and term,
   then the logs are identical in all entries up through that index. This is
   enforced by the `AppendEntries` consistency check: the leader sends the
   `prevLogIndex` and `prevLogTerm`, and the follower rejects if they don't
   match.

4. **Leader Completeness**: If a log entry is committed in a given term, that
   entry will be present in the logs of all leaders for all higher terms. This
   is guaranteed because a candidate must have an up-to-date log to win an
   election (voters reject candidates with less complete logs).

5. **State Machine Safety**: If a server has applied a log entry at a given
   index, no other server will ever apply a different log entry at that index.

### 11.2.6 Joint Consensus — Membership Changes

Changing cluster membership (adding or removing nodes) is dangerous because
during the transition, two different majorities might exist. Raft solves this
with **joint consensus**:

1. The leader creates a special configuration entry `C_old,new` that requires
   majorities from **both** the old and new configurations.
2. Once `C_old,new` is committed, the leader creates `C_new`.
3. Once `C_new` is committed, nodes not in `C_new` can be shut down.

```text
  C_old              C_old,new                    C_new
┌───────┐    ┌─────────────────────┐    ┌──────────────────┐
│ A B C │ ──►│ old majority: A B C │───►│ A B C D          │
│       │    │ new majority: A B C D│    │                  │
└───────┘    └─────────────────────┘    └──────────────────┘
                 both majorities
                 must agree
```

In practice, etcd implements the simpler **single-node membership change**
approach: only one node is added or removed at a time, which avoids the
complexity of joint consensus while still being safe.

### 11.2.7 Hands-on: Observing Raft Leader Election

Let us start a local 3-node etcd cluster and observe leader election:

```bash
#!/bin/bash
# start-etcd-cluster.sh
# Starts a 3-node etcd cluster on localhost for development/learning.
# Each node gets its own data directory and distinct ports.

# Node 1
etcd --name node1 \
  --initial-advertise-peer-urls http://localhost:2380 \
  --listen-peer-urls http://localhost:2380 \
  --listen-client-urls http://localhost:2379 \
  --advertise-client-urls http://localhost:2379 \
  --initial-cluster "node1=http://localhost:2380,node2=http://localhost:2480,node3=http://localhost:2580" \
  --initial-cluster-state new \
  --data-dir /tmp/etcd-node1 &

# Node 2
etcd --name node2 \
  --initial-advertise-peer-urls http://localhost:2480 \
  --listen-peer-urls http://localhost:2480 \
  --listen-client-urls http://localhost:2479 \
  --advertise-client-urls http://localhost:2479 \
  --initial-cluster "node1=http://localhost:2380,node2=http://localhost:2480,node3=http://localhost:2580" \
  --initial-cluster-state new \
  --data-dir /tmp/etcd-node2 &

# Node 3
etcd --name node3 \
  --initial-advertise-peer-urls http://localhost:2580 \
  --listen-peer-urls http://localhost:2580 \
  --listen-client-urls http://localhost:2579 \
  --advertise-client-urls http://localhost:2579 \
  --initial-cluster "node1=http://localhost:2380,node2=http://localhost:2480,node3=http://localhost:2580" \
  --initial-cluster-state new \
  --data-dir /tmp/etcd-node3 &

wait
```

Now observe who became the leader:

```bash
# Check the cluster status — the "IS LEADER" column shows the current leader.
etcdctl --endpoints=localhost:2379,localhost:2479,localhost:2579 \
  endpoint status --write-out=table

# Sample output:
# +------------------+------------------+---------+---------+-----------+...+------------+
# |     ENDPOINT     |        ID        | VERSION | DB SIZE | IS LEADER |...| RAFT INDEX |
# +------------------+------------------+---------+---------+-----------+...+------------+
# | localhost:2379   | 8e9e05c52164694d |  3.5.9  |   20 kB |     true  |...| 9          |
# | localhost:2479   | 91bc3c398fb3c146 |  3.5.9  |   20 kB |    false  |...| 9          |
# | localhost:2579   | fd422379fda50e48 |  3.5.9  |   20 kB |    false  |...| 9          |
# +------------------+------------------+---------+---------+-----------+...+------------+
```

Kill the leader and watch a new election happen:

```bash
# Kill node1 (the current leader).
# Within ~1 second, a new leader is elected from node2 or node3.
kill %1

# Check again — a new leader has been elected.
etcdctl --endpoints=localhost:2479,localhost:2579 \
  endpoint status --write-out=table

# The cluster continues to serve reads and writes with 2/3 nodes alive.
# This demonstrates Raft's fault tolerance: a 3-node cluster tolerates
# 1 failure (⌊(3-1)/2⌋ = 1).
```

---

## 11.3 etcd Architecture

Now that we understand the consensus layer, let us examine the full
architecture of an etcd node.

### ASCII Diagram: etcd Storage Architecture

```
                          ┌──────────────────────────────────┐
                          │          gRPC Server             │
                          │   (KV, Watch, Lease, Auth, Maint)│
                          └──────────┬───────────────────────┘
                                     │
                          ┌──────────▼───────────────────────┐
                          │          etcd Server             │
                          │   (request routing, auth, quota) │
                          └──────────┬───────────────────────┘
                                     │
                    ┌────────────────┼────────────────────┐
                    │                │                    │
           ┌────────▼──────┐ ┌──────▼───────┐  ┌────────▼────────┐
           │  Raft Module  │ │  MVCC Store  │  │  Lease Manager  │
           │               │ │              │  │                 │
           │  - Log        │ │  - Key Index │  │  - TTL heap     │
           │  - State      │ │    (B-tree)  │  │  - Lease map    │
           │    machine    │ │  - Revisions │  │  - KeepAlive    │
           │  - Snapshots  │ │  - Watchers  │  │    stream       │
           └───────┬───────┘ └──────┬───────┘  └─────────────────┘
                   │                │
          ┌────────▼──────┐ ┌──────▼───────┐
          │     WAL       │ │    bbolt     │
          │ (Write-Ahead  │ │  (B+tree on  │
          │   Log)        │ │   disk)      │
          │               │ │              │
          │ Sequential    │ │ Key-value    │
          │ append-only   │ │ storage with │
          │ for crash     │ │ transactions │
          │ recovery      │ │              │
          └───────────────┘ └──────────────┘
                   │                │
                   └────────┬───────┘
                            │
                    ┌───────▼───────┐
                    │   Disk (SSD)  │
                    │               │
                    │  data-dir/    │
                    │  ├── member/  │
                    │  │  ├── wal/  │
                    │  │  └── snap/ │
                    │  └── ...     │
                    └───────────────┘
```

### 11.3.1 MVCC — Multi-Version Concurrency Control

etcd does not simply store the latest value for each key. It maintains a
**history of revisions**, enabling:

- **Watch from a past revision**: clients can "rewind" and receive changes
  they missed.
- **Consistent snapshots**: read operations see a consistent view of the store
  at a specific revision.
- **Optimistic concurrency**: Kubernetes uses `resourceVersion` (which maps to
  etcd's `mod_revision`) to detect concurrent modifications.

Every mutation to the store increments a global **revision counter**. Each key
has a **create_revision** (when it was first created) and a **mod_revision**
(when it was last modified).

```
Revision 1:  Put(/foo, "bar")      → /foo created at rev 1
Revision 2:  Put(/baz, "qux")      → /baz created at rev 2
Revision 3:  Put(/foo, "updated")  → /foo modified at rev 3
Revision 4:  Delete(/baz)          → /baz deleted at rev 4

Key index for /foo:
  - Generation 1: [create=1, mod=1], [create=1, mod=3]

Key index for /baz:
  - Generation 1: [create=2, mod=2] (tombstone at rev 4)
```

### 11.3.2 BoltDB (bbolt) — The Storage Engine

The underlying storage engine is **bbolt**, a pure Go B+tree key-value database.
It provides:

- **ACID transactions** with serializable isolation.
- **Memory-mapped I/O** for reads.
- **Copy-on-write B+tree** for crash safety.
- **Single-writer, multiple-reader** concurrency.

etcd stores revisions in bbolt with the key being the revision number (encoded
as a big-endian 8-byte integer) and the value being the serialized key-value
pair. This makes sequential reads by revision extremely efficient.

```
bbolt bucket "key":
  ┌────────────────────────────────────────┐
  │ Key (revision)      │ Value            │
  ├─────────────────────┼──────────────────┤
  │ 0x0000000000000001  │ {/foo, "bar"}    │
  │ 0x0000000000000002  │ {/baz, "qux"}    │
  │ 0x0000000000000003  │ {/foo, "updated"}│
  │ 0x0000000000000004  │ {/baz, tombstone}│
  └────────────────────────────────────────┘
```

### 11.3.3 Key Index (B-tree)

The key index is an **in-memory B-tree** that maps each key to its revision
history. When a client requests `Get(/foo)`, etcd:

1. Looks up `/foo` in the key index B-tree → finds the latest revision (3).
2. Reads revision 3 from bbolt → returns the value `"updated"`.

For range queries, etcd iterates over the B-tree and collects all matching keys
and their latest revisions, then batch-reads from bbolt.

### 11.3.4 WAL (Write-Ahead Log)

The Write-Ahead Log is the **durability backbone** of etcd. Before any Raft log
entry is persisted to the state machine, it is first written to the WAL:

1. Raft proposes an entry.
2. The entry is appended to the WAL and `fsync`'d to disk.
3. Only after the WAL write is durable does etcd acknowledge the entry.
4. The entry is then applied to the MVCC store (bbolt).

If etcd crashes, it replays the WAL on startup to recover any committed entries
that were not yet fully applied to bbolt.

**WAL file structure**:
```
wal/
├── 0000000000000000-0000000000000000.wal    (first segment)
├── 0000000000000001-0000000000001000.wal    (second segment)
└── ...
```

Each WAL segment is ~64MB. Segments are rotated when they fill up.

### 11.3.5 Snapshots — Compaction and Space Reclaim

Over time, the WAL and bbolt accumulate old revisions. etcd manages this with:

- **Compaction**: removes all revisions before a given revision number from
  bbolt. After compaction, watches and reads at old revisions will fail with
  `ErrCompacted`.
- **Snapshots**: periodically, etcd takes a snapshot of the entire state machine
  and writes it to disk. This allows old WAL segments to be discarded.
- **Defragmentation**: compaction marks space as free but does not return it to
  the OS. Defragmentation rewrites the bbolt file to reclaim space.

```
Before compaction (revision 1000, compact at 500):
  bbolt contains: revisions 1–1000
  WAL contains: entries since last snapshot

After compaction:
  bbolt contains: revisions 501–1000
  Revisions 1–500 are gone (reads at rev < 501 return ErrCompacted)

After defragmentation:
  bbolt file is physically smaller (free space reclaimed)
```

### 11.3.6 Lease Mechanism — TTL-Based Keys

Leases provide automatic key expiration. A lease has a **TTL** (time-to-live).
Keys attached to a lease are automatically deleted when the lease expires.

Use cases:
- **Service discovery**: register a service key with a lease; if the service
  dies (stops refreshing), the key vanishes.
- **Distributed locking**: a lock key with a lease ensures the lock is released
  if the holder crashes.
- **Session management**: ephemeral session data that cleans up automatically.

The lease lifecycle:

```
Client                        etcd
  │                             │
  │── LeaseGrant(TTL=30s) ────►│   Create lease with 30s TTL
  │◄── LeaseID=1234 ───────────┤
  │                             │
  │── Put(/key, val,           │
  │      lease=1234) ──────────►│   Attach key to lease
  │◄── OK ─────────────────────┤
  │                             │
  │     ... every 10s ...       │
  │── LeaseKeepAlive(1234) ───►│   Refresh the lease
  │◄── TTL=30s ────────────────┤
  │                             │
  │     ... client crashes ...  │
  │                             │
  │                   30s later │
  │                    lease    │
  │                    expires  │
  │                    /key is  │
  │                    deleted  │
  │                             │
```

---

## 11.4 etcd API

etcd exposes its functionality through a gRPC API defined in Protocol Buffers.
The API is organized into five services.

### 11.4.1 KV API

The KV service handles all key-value operations:

| RPC         | Description                                               |
|-------------|-----------------------------------------------------------|
| `Put`       | Set a key to a value, optionally with a lease              |
| `Range`     | Read one key or a range of keys (Get is Range with limit=1)|
| `DeleteRange` | Delete one key or a range of keys                       |
| `Txn`       | Atomic compare-and-swap: if conditions, then ops, else ops |
| `Compact`   | Remove all revisions before a given revision               |

**Transactions (Txn)** are etcd's secret weapon. A Txn takes a list of
comparisons (e.g., "key X has version Y") and two lists of operations (then/
else). If all comparisons succeed, the "then" operations execute atomically;
otherwise the "else" operations execute.

```
Txn:
  IF   key("/lock") version == 0     (key does not exist)
  THEN Put("/lock", "holder-1")      (acquire the lock)
  ELSE Get("/lock")                  (see who holds it)
```

### 11.4.2 Watch API

The Watch service streams change events to clients:

| RPC           | Description                                             |
|---------------|---------------------------------------------------------|
| `Watch`       | Bidirectional stream: client sends WatchCreate/Cancel,  |
|               | server sends WatchResponse with events                  |

Key features:
- **Revision-based**: watch from a specific revision to replay missed events.
- **Key range**: watch a single key, a prefix, or an arbitrary range.
- **Multiplexed**: multiple watches share a single gRPC stream.
- **Fragmentation**: large responses are fragmented and reassembled.

Watch is the backbone of Kubernetes controllers. Every controller (deployment,
replicaset, service, etc.) watches etcd for changes to its resources and
reacts accordingly.

### 11.4.3 Lease API

| RPC              | Description                                          |
|------------------|------------------------------------------------------|
| `LeaseGrant`     | Create a lease with a given TTL                      |
| `LeaseRevoke`    | Immediately revoke a lease (deletes attached keys)   |
| `LeaseKeepAlive` | Bidirectional stream to refresh lease TTL            |
| `LeaseTimeToLive`| Query remaining TTL and attached keys                |
| `LeaseLeases`    | List all active leases                               |

### 11.4.4 Auth API

etcd supports authentication and role-based access control (RBAC):

| RPC                | Description                                       |
|--------------------|---------------------------------------------------|
| `AuthEnable`       | Enable authentication                             |
| `AuthDisable`      | Disable authentication                            |
| `Authenticate`     | Get an auth token                                 |
| `UserAdd/Delete`   | Manage users                                      |
| `RoleAdd/Delete`   | Manage roles                                      |
| `RoleGrantPermission` | Grant key-range permissions to a role          |
| `UserGrantRole`    | Assign a role to a user                           |

### 11.4.5 Maintenance API

| RPC         | Description                                               |
|-------------|-----------------------------------------------------------|
| `Alarm`     | Get/activate/deactivate alarms (e.g., NOSPACE)            |
| `Status`    | Get member status (leader, db size, raft index)           |
| `Defragment`| Reclaim free space in the backend                         |
| `Hash`      | Get hash of the backend for consistency checking          |
| `Snapshot`  | Stream a consistent snapshot of the entire database       |
| `MoveLeader`| Transfer leadership to another member                     |

---

## 11.5 How Kubernetes Uses etcd

Kubernetes uses etcd as its **sole persistent store**. Understanding how
Kubernetes maps its objects into etcd keys is essential for debugging and
operations.

### 11.5.1 Key Structure

All Kubernetes data lives under the `/registry` prefix:

```
/registry/{resource}/{namespace}/{name}

Examples:
/registry/pods/default/nginx-7b449d7b65-x9z2k
/registry/services/specs/kube-system/kube-dns
/registry/deployments/production/web-frontend
/registry/secrets/default/my-secret
/registry/configmaps/default/app-config
/registry/namespaces/production
/registry/clusterroles/cluster-admin
```

Cluster-scoped resources omit the namespace:
```
/registry/namespaces/production
/registry/nodes/worker-01
/registry/clusterroles/cluster-admin
/registry/customresourcedefinitions/crontabs.stable.example.com
```

### 11.5.2 What's Stored

Each key's value is a **serialized Kubernetes object**, typically encoded in
Protocol Buffers (protobuf) for efficiency, though JSON encoding is also
supported. The serialization includes:

- `apiVersion` and `kind`
- `metadata` (name, namespace, uid, resourceVersion, labels, annotations, ...)
- `spec` (desired state)
- `status` (observed state)

### 11.5.3 resourceVersion and Optimistic Concurrency

Every Kubernetes object has a `metadata.resourceVersion` field. This field maps
directly to etcd's `mod_revision` — the global revision at which the key was
last modified.

Kubernetes uses this for **optimistic concurrency control**:

1. Client reads an object (e.g., a Deployment) and receives `resourceVersion: 42`.
2. Client modifies the object and sends an update with `resourceVersion: 42`.
3. The API server translates this into an etcd transaction:
   ```
   Txn:
     IF   mod_revision("/registry/deployments/default/web") == 42
     THEN Put("/registry/deployments/default/web", updated_object)
     ELSE fail with 409 Conflict
   ```
4. If another client modified the object in the meantime (revision is now 43),
   the transaction fails, and the client must retry.

This is how Kubernetes avoids lost updates without pessimistic locking.

### 11.5.4 Watch: How Controllers Receive Changes

The Kubernetes API server maintains long-lived etcd watches on key prefixes.
When a pod is created, updated, or deleted:

1. The change is committed in etcd.
2. etcd sends a watch event to the API server.
3. The API server forwards the event to all clients watching that resource type.
4. Controllers (e.g., ReplicaSet controller) receive the event and reconcile.

```
  etcd                API Server            Controller
    │                     │                     │
    │── WatchResponse ───►│                     │
    │   (Put /registry/   │                     │
    │    pods/default/x)  │                     │
    │                     │── Watch event ─────►│
    │                     │   (ADDED pod/x)     │
    │                     │                     │── reconcile
    │                     │                     │   (ensure desired
    │                     │                     │    replica count)
```

### 11.5.5 Hands-on: Reading Kubernetes Data Directly from etcd

> ⚠️ **Warning**: Never modify etcd data directly in a production cluster.
> This bypasses all Kubernetes validation and can corrupt your cluster.

In a test cluster (e.g., `kind` or `minikube`), you can read etcd directly:

```bash
# For a kind cluster, exec into the etcd pod:
kubectl exec -it -n kube-system etcd-kind-control-plane -- sh

# List all keys under /registry (first 20):
etcdctl get /registry --prefix --keys-only --limit=20 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Read a specific pod (output is protobuf, so it looks binary):
etcdctl get /registry/pods/default/nginx \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# To see JSON, use auger (a tool for decoding Kubernetes etcd values):
# https://github.com/etcd-io/auger
etcdctl get /registry/pods/default/nginx \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --print-value-only | auger decode

# Check the etcd cluster member list:
etcdctl member list \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  --write-out=table
```

---

## 11.6 etcd Go Client (clientv3)

The official etcd Go client, `clientv3`, provides a clean, strongly-typed API
built on gRPC. Let us build several hands-on examples.

### 11.6.1 Hands-on: Connecting to etcd from Go

```go
// File: connect/main.go
//
// This program demonstrates how to establish a connection to an etcd cluster
// using the official Go client library (clientv3). It connects to a local
// 3-node cluster, verifies connectivity by checking the cluster status,
// and prints the leader ID and database size for each endpoint.
//
// Prerequisites:
//   - A running etcd cluster on localhost:2379, :2479, :2579
//   - go get go.etcd.io/etcd/client/v3
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	clientv3 "go.etcd.io/etcd/client/v3"
)

func main() {
	// Create an etcd client configuration.
	// DialTimeout controls how long the client waits for the initial
	// connection to be established. If etcd is unavailable, the client
	// will return an error after this duration.
	//
	// Endpoints is a list of etcd cluster member addresses. The client
	// will automatically load-balance requests across healthy endpoints
	// and failover if a member becomes unreachable.
	cfg := clientv3.Config{
		Endpoints:   []string{"localhost:2379", "localhost:2479", "localhost:2579"},
		DialTimeout: 5 * time.Second,
	}

	// Establish the connection. This does NOT block until the connection
	// is ready — it returns immediately and connects lazily. The
	// DialTimeout applies to each individual RPC, not to this call.
	client, err := clientv3.New(cfg)
	if err != nil {
		log.Fatalf("Failed to create etcd client: %v", err)
	}
	// Always close the client when done to release resources (gRPC
		// connections, goroutines, etc.).
	defer client.Close()

	// Create a context with a 5-second timeout for our status check.
	// In production, use context.WithTimeout for all etcd operations
	// to avoid hanging indefinitely if etcd is slow or unreachable.
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	// Check the status of each endpoint. This is a good way to verify
	// that the client can communicate with the cluster and to identify
	// the current leader.
	for _, ep := range cfg.Endpoints {
		// Status returns information about the etcd member at the
		// given endpoint, including its Raft leader ID, database
		// size, and Raft index.
		status, err := client.Status(ctx, ep)
		if err != nil {
			log.Printf("Failed to get status for %s: %v", ep, err)
			continue
		}

		// Print the status. The Leader field is the Raft node ID of
		// the current leader (same across all healthy members).
		// DbSize is the physical size of the bbolt database file.
		fmt.Printf("Endpoint: %s\n", ep)
		fmt.Printf("  Leader:  %x\n", status.Leader)
		fmt.Printf("  DB Size: %d bytes\n", status.DbSize)
		fmt.Printf("  Raft Index: %d\n", status.RaftIndex)
		fmt.Printf("  Raft Term:  %d\n", status.RaftTerm)
		fmt.Println()
	}

	fmt.Println("Successfully connected to etcd cluster!")
}
```

### 11.6.2 Hands-on: CRUD Operations with the Go Client

```go
// File: crud/main.go
//
// This program demonstrates the fundamental CRUD (Create, Read, Update,
	// Delete) operations on etcd using the Go client. It covers:
//
//   - Put: storing a key-value pair
//   - Get: retrieving a single key
//   - Get with prefix: retrieving all keys under a prefix
//   - Get with revision: reading a past version of a key
//   - Put with lease: attaching a TTL to a key
//   - Delete: removing a key
//   - Delete with prefix: removing all keys under a prefix
//
// Each operation includes detailed comments explaining the response fields
// and their significance in the MVCC model.
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	clientv3 "go.etcd.io/etcd/client/v3"
)

func main() {
	// Connect to the etcd cluster. See connect/main.go for details
	// on the configuration options.
	client, err := clientv3.New(clientv3.Config{
			Endpoints:   []string{"localhost:2379"},
			DialTimeout: 5 * time.Second,
		})
	if err != nil {
		log.Fatalf("Failed to create etcd client: %v", err)
	}
	defer client.Close()

	// All operations use a context with a timeout to prevent hangs.
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	// ──────────────────────────────────────────────────────────────
	// PUT — Store a key-value pair
	// ──────────────────────────────────────────────────────────────

	// Put stores the key "/app/config/db_host" with the value
	// "postgres.default.svc.cluster.local". The response contains
	// the cluster-wide revision at which this write occurred.
	putResp, err := client.Put(ctx, "/app/config/db_host", "postgres.default.svc.cluster.local")
	if err != nil {
		log.Fatalf("Put failed: %v", err)
	}
	// The Header contains the revision — the global, monotonically
	// increasing counter that is incremented with every mutation.
	fmt.Printf("Put succeeded. Cluster revision: %d\n", putResp.Header.Revision)

	// Store a few more keys to demonstrate range queries.
	client.Put(ctx, "/app/config/db_port", "5432")
	client.Put(ctx, "/app/config/db_name", "myapp")
	client.Put(ctx, "/app/config/cache_host", "redis.default.svc.cluster.local")

	// ──────────────────────────────────────────────────────────────
	// GET — Retrieve a single key
	// ──────────────────────────────────────────────────────────────

	// Get retrieves the value for a single key. Internally, Get is
	// a Range RPC with a limit of 1.
	getResp, err := client.Get(ctx, "/app/config/db_host")
	if err != nil {
		log.Fatalf("Get failed: %v", err)
	}

	// The response contains a slice of KeyValue pairs (usually 1
		// for a single-key Get). Each KeyValue has:
	//   - Key:            the key bytes
	//   - Value:          the value bytes
	//   - CreateRevision: the revision when this key was first created
	//   - ModRevision:    the revision when this key was last modified
	//   - Version:        how many times this key has been modified
	//                     (resets to 1 on create, increments on update)
	//   - Lease:          the lease ID attached to this key (0 if none)
	for _, kv := range getResp.Kvs {
		fmt.Printf("Key: %s\n", kv.Key)
		fmt.Printf("  Value:          %s\n", kv.Value)
		fmt.Printf("  CreateRevision: %d\n", kv.CreateRevision)
		fmt.Printf("  ModRevision:    %d\n", kv.ModRevision)
		fmt.Printf("  Version:        %d\n", kv.Version)
		fmt.Printf("  Lease:          %d\n", kv.Lease)
	}

	// ──────────────────────────────────────────────────────────────
	// GET with PREFIX — Retrieve all keys under a prefix
	// ──────────────────────────────────────────────────────────────

	// WithPrefix() modifies the Get to return all keys that start
	// with the given prefix. This is how etcd simulates "directories".
	// Internally, it sets the range_end to the lexicographic successor
	// of the prefix (e.g., "/app/config/" → "/app/config0").
	rangeResp, err := client.Get(ctx, "/app/config/", clientv3.WithPrefix())
	if err != nil {
		log.Fatalf("Range Get failed: %v", err)
	}

	fmt.Printf("\nAll keys under /app/config/ (%d keys):\n", rangeResp.Count)
	for _, kv := range rangeResp.Kvs {
		fmt.Printf("  %s = %s (rev %d)\n", kv.Key, kv.Value, kv.ModRevision)
	}

	// ──────────────────────────────────────────────────────────────
	// UPDATE — Modify an existing key
	// ──────────────────────────────────────────────────────────────

	// In etcd, there is no distinct "update" operation. A Put to an
	// existing key overwrites the value and increments the version.
	// The old value is retained in the revision history until
	// compaction removes it.
	prevRevision := putResp.Header.Revision

	// Use WithPrevKV() to get the previous value in the response.
	// This is useful for compare-and-swap patterns.
	updateResp, err := client.Put(ctx, "/app/config/db_host",
		"postgres-ha.default.svc.cluster.local",
		clientv3.WithPrevKV(),
	)
	if err != nil {
		log.Fatalf("Update failed: %v", err)
	}

	fmt.Printf("\nUpdate succeeded at revision %d\n", updateResp.Header.Revision)
	if updateResp.PrevKv != nil {
		fmt.Printf("  Previous value: %s\n", updateResp.PrevKv.Value)
	}

	// ──────────────────────────────────────────────────────────────
	// GET with REVISION — Read a past version of a key
	// ──────────────────────────────────────────────────────────────

	// WithRev() reads the key as it was at a specific revision.
	// This is the foundation of etcd's time-travel capability.
	// Kubernetes uses this to serve consistent list/watch operations.
	oldResp, err := client.Get(ctx, "/app/config/db_host",
		clientv3.WithRev(prevRevision),
	)
	if err != nil {
		log.Fatalf("Get with revision failed: %v", err)
	}

	fmt.Printf("\nValue at revision %d: %s\n",
		prevRevision, oldResp.Kvs[0].Value)

	// ──────────────────────────────────────────────────────────────
	// DELETE — Remove a key
	// ──────────────────────────────────────────────────────────────

	// Delete removes a key. Use WithPrevKV() to see what was deleted.
	delResp, err := client.Delete(ctx, "/app/config/cache_host",
		clientv3.WithPrevKV(),
	)
	if err != nil {
		log.Fatalf("Delete failed: %v", err)
	}

	fmt.Printf("\nDeleted %d key(s)\n", delResp.Deleted)
	for _, kv := range delResp.PrevKvs {
		fmt.Printf("  Deleted: %s = %s\n", kv.Key, kv.Value)
	}

	// ──────────────────────────────────────────────────────────────
	// DELETE with PREFIX — Remove all keys under a prefix
	// ──────────────────────────────────────────────────────────────

	// WithPrefix() on Delete removes all keys matching the prefix.
	// This is equivalent to "rm -rf /app/config/" in a filesystem.
	delAllResp, err := client.Delete(ctx, "/app/config/", clientv3.WithPrefix())
	if err != nil {
		log.Fatalf("Delete with prefix failed: %v", err)
	}

	fmt.Printf("\nDeleted %d key(s) under /app/config/\n", delAllResp.Deleted)
}
```

### 11.6.3 Hands-on: Watch Implementation in Go

```go
// File: watch/main.go
//
// This program demonstrates etcd's Watch API — the mechanism that enables
// real-time change notifications. Watch is the foundation of Kubernetes
// controllers: every time a pod, service, or deployment changes, the
// relevant controller is notified via a watch event.
//
// This example:
//   1. Starts a watch on the prefix "/events/" from the current revision.
//   2. In a separate goroutine, writes several keys under "/events/".
//   3. The main goroutine prints each watch event as it arrives.
//   4. Demonstrates how to handle watch cancellation and errors.
//
// Prerequisites:
//   - A running etcd instance on localhost:2379
package main

import (
	"context"
	"fmt"
	"log"
	"sync"
	"time"

	clientv3 "go.etcd.io/etcd/client/v3"
	"go.etcd.io/etcd/api/v3/mvccpb"
)

func main() {
	// Create the etcd client.
	client, err := clientv3.New(clientv3.Config{
			Endpoints:   []string{"localhost:2379"},
			DialTimeout: 5 * time.Second,
		})
	if err != nil {
		log.Fatalf("Failed to create etcd client: %v", err)
	}
	defer client.Close()

	// Use a WaitGroup to coordinate the writer and watcher goroutines.
	var wg sync.WaitGroup

	// Create a cancellable context for the watch. We will cancel it
	// after the writer finishes and we have received all expected events.
	watchCtx, watchCancel := context.WithCancel(context.Background())
	defer watchCancel()

	// ──────────────────────────────────────────────────────────────
	// START THE WATCH
	// ──────────────────────────────────────────────────────────────

	// Watch returns a WatchChan — a Go channel that delivers
	// WatchResponse messages. Each WatchResponse contains one or
	// more Events (Put or Delete).
	//
	// clientv3.WithPrefix() watches all keys with the given prefix.
	// clientv3.WithPrevKV() includes the previous key-value in the
	// event, useful for seeing what changed.
	//
	// The watch starts from the current revision by default. To
	// replay from a specific revision, use clientv3.WithRev(rev).
	watchChan := client.Watch(watchCtx, "/events/",
		clientv3.WithPrefix(),
		clientv3.WithPrevKV(),
	)

	// Process watch events in a goroutine.
	wg.Add(1)
	go func() {
		defer wg.Done()

		eventCount := 0

		// Range over the watch channel. This blocks until an event
		// arrives or the watch is cancelled/errored.
		for watchResp := range watchChan {
			// Check for errors. A watch can be cancelled by the
			// server (e.g., if the requested revision was compacted)
			// or by the client.
			if watchResp.Err() != nil {
				log.Printf("Watch error: %v", watchResp.Err())
				return
			}

			// Each WatchResponse may contain multiple events if
			// several mutations happened in quick succession.
			for _, event := range watchResp.Events {
				eventCount++

				// Event types:
				//   - mvccpb.PUT:    a key was created or updated
				//   - mvccpb.DELETE: a key was deleted
				switch event.Type {
				case mvccpb.PUT:
					fmt.Printf("[WATCH] PUT  key=%s value=%s (rev=%d)\n",
						event.Kv.Key, event.Kv.Value, event.Kv.ModRevision)

					// If the previous value is available (because
						// we used WithPrevKV), show what changed.
					if event.PrevKv != nil {
						fmt.Printf("        prev value=%s\n", event.PrevKv.Value)
					}

				case mvccpb.DELETE:
					fmt.Printf("[WATCH] DELETE key=%s (rev=%d)\n",
						event.Kv.Key, event.Kv.ModRevision)
				}
			}

			// After receiving all expected events, cancel the watch.
			if eventCount >= 5 {
				fmt.Println("\n[WATCH] Received all expected events, stopping watch.")
				watchCancel()
			}
		}

		fmt.Println("[WATCH] Watch channel closed.")
	}()

	// ──────────────────────────────────────────────────────────────
	// WRITE EVENTS (in a separate goroutine)
	// ──────────────────────────────────────────────────────────────

	wg.Add(1)
	go func() {
		defer wg.Done()

		// Give the watch a moment to establish.
		time.Sleep(500 * time.Millisecond)

		writeCtx, writeCancel := context.WithTimeout(context.Background(), 10*time.Second)
		defer writeCancel()

		// Simulate a series of events.
		events := []struct {
			key   string
			value string
		}{
			{"/events/order/1001", `{"status":"created","item":"widget"}`},
			{"/events/order/1002", `{"status":"created","item":"gadget"}`},
			{"/events/order/1001", `{"status":"shipped","item":"widget"}`},
			{"/events/payment/1001", `{"amount":29.99,"status":"charged"}`},
		}

		for _, e := range events {
			_, err := client.Put(writeCtx, e.key, e.value)
			if err != nil {
				log.Printf("Put failed: %v", err)
				return
			}
			fmt.Printf("[WRITER] Wrote %s\n", e.key)
			time.Sleep(200 * time.Millisecond)
		}

		// Delete a key to generate a DELETE event.
		_, err := client.Delete(writeCtx, "/events/order/1002")
		if err != nil {
			log.Printf("Delete failed: %v", err)
			return
		}
		fmt.Println("[WRITER] Deleted /events/order/1002")
	}()

	// Wait for both goroutines to finish.
	wg.Wait()
	fmt.Println("\nWatch demo complete.")
}
```

### 11.6.4 Hands-on: Distributed Locking with etcd

```go
// File: lock/main.go
//
// This program demonstrates distributed locking using etcd's concurrency
// package. Distributed locks are essential when multiple processes need
// exclusive access to a shared resource (e.g., a database migration,
	// a cron job that must run on exactly one node, or a critical section
	// in a microservices architecture).
//
// etcd's distributed lock is built on top of:
//   - Leases: the lock is automatically released if the holder crashes
//   - Transactions: atomic compare-and-swap ensures only one holder
//   - Watch: waiters are notified when the lock becomes available
//
// This example simulates 3 workers competing for a lock.
package main

import (
	"context"
	"fmt"
	"log"
	"sync"
	"time"

	clientv3 "go.etcd.io/etcd/client/v3"
	"go.etcd.io/etcd/client/v3/concurrency"
)

func main() {
	// Connect to etcd.
	client, err := clientv3.New(clientv3.Config{
			Endpoints:   []string{"localhost:2379"},
			DialTimeout: 5 * time.Second,
		})
	if err != nil {
		log.Fatalf("Failed to create etcd client: %v", err)
	}
	defer client.Close()

	var wg sync.WaitGroup

	// Simulate 3 workers competing for a lock.
	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go func(workerID int) {
			defer wg.Done()
			acquireAndWork(client, workerID)
		}(i)
	}

	wg.Wait()
	fmt.Println("\nAll workers finished.")
}

// acquireAndWork creates an etcd session, acquires a distributed lock,
// performs some "critical section" work, and then releases the lock.
//
// Parameters:
//   - client:   the etcd client connection shared by all workers
//   - workerID: a unique identifier for this worker (for logging)
//
// The function demonstrates:
//   1. Creating a session with a TTL (lease-based auto-expiry)
//   2. Acquiring a mutex (blocks until the lock is available)
//   3. Performing work while holding the lock
//   4. Releasing the lock (explicit unlock + session close)
func acquireAndWork(client *clientv3.Client, workerID int) {
	// Create a new session. A session is backed by an etcd lease.
	// If this process crashes, the lease expires (after TTL seconds),
	// and the lock is automatically released. This prevents deadlocks
	// caused by crashed lock holders.
	//
	// TTL of 10 seconds means: if we crash, the lock is released
	// within 10 seconds. The session automatically sends KeepAlive
	// RPCs to refresh the lease while the process is alive.
	session, err := concurrency.NewSession(client, concurrency.WithTTL(10))
	if err != nil {
		log.Printf("Worker %d: failed to create session: %v", workerID, err)
		return
	}
	// Close the session when done. This revokes the lease, which
	// immediately releases any locks held by this session.
	defer session.Close()

	// Create a mutex on the key prefix "/my-lock/".
	// All workers using the same prefix compete for the same lock.
	// Under the hood, each mutex creates a unique key under this
	// prefix (e.g., "/my-lock/1234abcd") and uses etcd transactions
	// to determine ordering.
	mutex := concurrency.NewMutex(session, "/my-lock/")

	fmt.Printf("Worker %d: waiting to acquire lock...\n", workerID)

	// Lock() blocks until this worker acquires the lock. It uses
	// etcd's Watch API to efficiently wait — no busy polling.
	// The context controls the maximum wait time.
	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	if err := mutex.Lock(ctx); err != nil {
		log.Printf("Worker %d: failed to acquire lock: %v", workerID, err)
		return
	}

	// ── CRITICAL SECTION ──────────────────────────────────────────
	// Only one worker can be here at a time. The lock guarantees
	// mutual exclusion across all processes in the cluster.
	fmt.Printf("Worker %d: *** ACQUIRED LOCK *** (key: %s)\n",
		workerID, mutex.Key())

	// Simulate some work that requires exclusive access.
	time.Sleep(2 * time.Second)

	fmt.Printf("Worker %d: releasing lock...\n", workerID)
	// ── END CRITICAL SECTION ──────────────────────────────────────

	// Unlock() releases the lock, allowing the next waiting worker
	// to proceed. If we skip this (or crash), the lock is released
	// when the session's lease expires.
	if err := mutex.Unlock(ctx); err != nil {
		log.Printf("Worker %d: failed to release lock: %v", workerID, err)
	}
}
```

### 11.6.5 Hands-on: Leader Election with etcd

```go
// File: election/main.go
//
// This program demonstrates leader election using etcd's concurrency
// package. Leader election is a core pattern in distributed systems:
// among a group of replicas, exactly one is elected as the "leader"
// and performs work that must happen on a single node (e.g., a
	// scheduler, a singleton controller, a primary database).
//
// etcd's leader election differs from distributed locking:
//   - A lock has no concept of "campaigning" — you acquire or wait.
//   - An election allows participants to campaign, observe the current
//     leader, and resign. The leader can announce a value (e.g., its
	//     address) that followers can read.
//
// This example simulates 3 candidates competing for leadership.
package main

import (
	"context"
	"fmt"
	"log"
	"sync"
	"time"

	clientv3 "go.etcd.io/etcd/client/v3"
	"go.etcd.io/etcd/client/v3/concurrency"
)

func main() {
	// Connect to etcd.
	client, err := clientv3.New(clientv3.Config{
			Endpoints:   []string{"localhost:2379"},
			DialTimeout: 5 * time.Second,
		})
	if err != nil {
		log.Fatalf("Failed to create etcd client: %v", err)
	}
	defer client.Close()

	var wg sync.WaitGroup

	// Start 3 candidates competing for leadership.
	for i := 1; i <= 3; i++ {
		wg.Add(1)
		go func(candidateID int) {
			defer wg.Done()
			runCandidate(client, candidateID)
		}(i)
	}

	wg.Wait()
	fmt.Println("\nElection demo complete.")
}

// runCandidate creates a session, campaigns for leadership in the
// "/my-election/" election, performs leader duties if elected,
// then resigns to let another candidate take over.
//
// Parameters:
//   - client:      the shared etcd client connection
//   - candidateID: a unique identifier for this candidate (for logging)
//
// The function demonstrates:
//   1. Creating a session with a TTL for automatic leadership release
//   2. Campaigning for leadership (blocks until elected)
//   3. Announcing a value as leader (e.g., the leader's address)
//   4. Observing the current leader from another goroutine
//   5. Resigning leadership gracefully
func runCandidate(client *clientv3.Client, candidateID int) {
	// Create a session with a 15-second TTL. If this candidate
	// crashes while leading, leadership is released within 15s.
	session, err := concurrency.NewSession(client, concurrency.WithTTL(15))
	if err != nil {
		log.Printf("Candidate %d: session error: %v", candidateID, err)
		return
	}
	defer session.Close()

	// Create an election on the prefix "/my-election/".
	// All candidates using the same prefix participate in the same
	// election. Under the hood, each candidate creates a key under
	// this prefix and uses revision ordering to determine the leader.
	election := concurrency.NewElection(session, "/my-election/")

	// Campaign for leadership. This blocks until this candidate
	// becomes the leader. The second argument is the "proposal" —
	// a value that the leader announces (e.g., its network address).
	// Other participants can observe this value.
	proposal := fmt.Sprintf("candidate-%d-at-host-%d.example.com", candidateID, candidateID)

	fmt.Printf("Candidate %d: campaigning for leadership...\n", candidateID)

	ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
	defer cancel()

	if err := election.Campaign(ctx, proposal); err != nil {
		log.Printf("Candidate %d: campaign failed: %v", candidateID, err)
		return
	}

	// If we reach here, we are the leader!
	fmt.Printf("Candidate %d: *** ELECTED LEADER *** (proposal: %s)\n",
		candidateID, proposal)

	// Observe the current leader. Leader() returns the current
	// leader's key and proposal value.
	leaderResp, err := election.Leader(ctx)
	if err != nil {
		log.Printf("Candidate %d: failed to get leader: %v", candidateID, err)
	} else {
		fmt.Printf("Candidate %d: leader key=%s, value=%s\n",
			candidateID, leaderResp.Kvs[0].Key, leaderResp.Kvs[0].Value)
	}

	// Simulate leader work.
	fmt.Printf("Candidate %d: performing leader duties for 3 seconds...\n", candidateID)
	time.Sleep(3 * time.Second)

	// Resign leadership. This deletes the leader key, allowing the
	// next candidate (by revision order) to become leader.
	fmt.Printf("Candidate %d: resigning leadership.\n", candidateID)
	if err := election.Resign(ctx); err != nil {
		log.Printf("Candidate %d: resign failed: %v", candidateID, err)
	}
}
```

### 11.6.6 Hands-on: Transactions — Compare-and-Swap Operations

```go
// File: transaction/main.go
//
// This program demonstrates etcd transactions (Txn) — the atomic
// compare-and-swap primitive that underpins Kubernetes' optimistic
// concurrency control.
//
// A transaction has the form:
//   IF   (conditions are met)
//   THEN (execute these operations)
//   ELSE (execute these operations instead)
//
// All conditions are evaluated atomically, and either the THEN or ELSE
// branch executes atomically. This is how you build safe concurrent
// operations without locks.
//
// This example demonstrates:
//   1. Basic compare-and-swap: update a key only if its value matches
//   2. Create-if-not-exists: set a key only if it doesn't exist
//   3. Multi-key transactions: atomic operations on multiple keys
//   4. Kubernetes-style optimistic concurrency using mod_revision
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	clientv3 "go.etcd.io/etcd/client/v3"
)

func main() {
	// Connect to etcd.
	client, err := clientv3.New(clientv3.Config{
			Endpoints:   []string{"localhost:2379"},
			DialTimeout: 5 * time.Second,
		})
	if err != nil {
		log.Fatalf("Failed to create etcd client: %v", err)
	}
	defer client.Close()

	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	// ──────────────────────────────────────────────────────────────
	// Example 1: Create-if-not-exists
	// ──────────────────────────────────────────────────────────────
	// This is the simplest transaction pattern: create a key only
	// if it does not already exist. This is how distributed locks
	// work at the lowest level.

	fmt.Println("=== Example 1: Create-if-not-exists ===")

	// First, ensure the key doesn't exist.
	client.Delete(ctx, "/lock/resource-1")

	// Transaction: IF the key has CreateRevision == 0 (does not exist),
	// THEN create it; ELSE return the existing value.
	//
	// clientv3.Compare creates a comparison. CreateRevision(key) == 0
	// means the key does not exist.
	//
	// clientv3.OpPut and clientv3.OpGet create operations for the
	// THEN and ELSE branches.
	txnResp, err := client.Txn(ctx).
	If(clientv3.Compare(clientv3.CreateRevision("/lock/resource-1"), "=", 0)).
	Then(clientv3.OpPut("/lock/resource-1", "holder-A")).
	Else(clientv3.OpGet("/lock/resource-1")).
	Commit()
	if err != nil {
		log.Fatalf("Transaction failed: %v", err)
	}

	// txnResp.Succeeded is true if the IF condition was met (THEN
		// branch executed), false if the ELSE branch executed.
	if txnResp.Succeeded {
		fmt.Println("  Lock acquired by holder-A (key created)")
	} else {
		// The ELSE branch executed — someone else holds the lock.
		kv := txnResp.Responses[0].GetResponseRange().Kvs[0]
		fmt.Printf("  Lock already held by: %s\n", kv.Value)
	}

	// Try again — this time the key exists, so ELSE executes.
	txnResp2, err := client.Txn(ctx).
	If(clientv3.Compare(clientv3.CreateRevision("/lock/resource-1"), "=", 0)).
	Then(clientv3.OpPut("/lock/resource-1", "holder-B")).
	Else(clientv3.OpGet("/lock/resource-1")).
	Commit()
	if err != nil {
		log.Fatalf("Transaction failed: %v", err)
	}

	if txnResp2.Succeeded {
		fmt.Println("  Lock acquired by holder-B")
	} else {
		kv := txnResp2.Responses[0].GetResponseRange().Kvs[0]
		fmt.Printf("  Lock NOT acquired — already held by: %s\n", kv.Value)
	}

	// ──────────────────────────────────────────────────────────────
	// Example 2: Kubernetes-style optimistic concurrency
	// ──────────────────────────────────────────────────────────────
	// This demonstrates how Kubernetes uses mod_revision as
	// resourceVersion for compare-and-swap updates.

	fmt.Println("\n=== Example 2: Optimistic concurrency (resourceVersion) ===")

	// Write initial value.
	client.Put(ctx, "/registry/configmaps/default/myconfig", `{"data":"v1"}`)

	// Read the current value and its mod_revision.
	getResp, _ := client.Get(ctx, "/registry/configmaps/default/myconfig")
	currentKV := getResp.Kvs[0]
	currentRevision := currentKV.ModRevision

	fmt.Printf("  Current value: %s (revision: %d)\n", currentKV.Value, currentRevision)

	// Update only if the revision hasn't changed (no one else modified
		// the key since we read it). This is exactly what the Kubernetes
	// API server does when processing an Update request.
	//
	// ModRevision(key) == currentRevision means "no one has modified
	// this key since we last read it."
	txnResp3, err := client.Txn(ctx).
	If(clientv3.Compare(clientv3.ModRevision("/registry/configmaps/default/myconfig"), "=", currentRevision)).
	Then(clientv3.OpPut("/registry/configmaps/default/myconfig", `{"data":"v2"}`)).
	Else(clientv3.OpGet("/registry/configmaps/default/myconfig")).
	Commit()
	if err != nil {
		log.Fatalf("Transaction failed: %v", err)
	}

	if txnResp3.Succeeded {
		fmt.Println("  Update succeeded (no concurrent modification)")
	} else {
		kv := txnResp3.Responses[0].GetResponseRange().Kvs[0]
		fmt.Printf("  Update FAILED — conflict! Current value: %s (revision: %d)\n",
			kv.Value, kv.ModRevision)
	}

	// Simulate a conflict: try to update with the OLD revision.
	txnResp4, err := client.Txn(ctx).
	If(clientv3.Compare(clientv3.ModRevision("/registry/configmaps/default/myconfig"), "=", currentRevision)).
	Then(clientv3.OpPut("/registry/configmaps/default/myconfig", `{"data":"v3-conflicting"}`)).
	Else(clientv3.OpGet("/registry/configmaps/default/myconfig")).
	Commit()
	if err != nil {
		log.Fatalf("Transaction failed: %v", err)
	}

	if txnResp4.Succeeded {
		fmt.Println("  Conflicting update succeeded (unexpected!)")
	} else {
		kv := txnResp4.Responses[0].GetResponseRange().Kvs[0]
		fmt.Printf("  Conflicting update REJECTED — current: %s (revision: %d)\n",
			kv.Value, kv.ModRevision)
	}

	// ──────────────────────────────────────────────────────────────
	// Example 3: Multi-key atomic transaction
	// ──────────────────────────────────────────────────────────────
	// Transfer "balance" from account A to account B atomically.
	// This demonstrates that etcd transactions can span multiple keys.

	fmt.Println("\n=== Example 3: Multi-key atomic transaction ===")

	// Set initial balances.
	client.Put(ctx, "/accounts/A", "100")
	client.Put(ctx, "/accounts/B", "50")

	// Atomic transfer: debit A and credit B in a single transaction.
	// The IF clause checks that both accounts exist (have been created).
	txnResp5, err := client.Txn(ctx).
	If(
		// Both accounts must exist.
		clientv3.Compare(clientv3.CreateRevision("/accounts/A"), ">", 0),
		clientv3.Compare(clientv3.CreateRevision("/accounts/B"), ">", 0),
	).
	Then(
		// Note: etcd doesn't support arithmetic in transactions.
		// In a real system, you would read the balances first,
		// compute the new values, then use ModRevision checks
		// to ensure no concurrent modification.
		clientv3.OpPut("/accounts/A", "80"),
		clientv3.OpPut("/accounts/B", "70"),
	).
	Else(
		// One or both accounts don't exist.
		clientv3.OpGet("/accounts/A"),
		clientv3.OpGet("/accounts/B"),
	).
	Commit()
	if err != nil {
		log.Fatalf("Transaction failed: %v", err)
	}

	if txnResp5.Succeeded {
		fmt.Println("  Transfer complete: A=80, B=70")
	} else {
		fmt.Println("  Transfer failed: one or both accounts missing")
	}

	// Clean up.
	client.Delete(ctx, "/lock/", clientv3.WithPrefix())
	client.Delete(ctx, "/registry/", clientv3.WithPrefix())
	client.Delete(ctx, "/accounts/", clientv3.WithPrefix())

	fmt.Println("\nTransaction demo complete.")
}
```

---

## 11.7 etcd Operations

Operating etcd in production is where theory meets reality. This section covers
the essential operational knowledge for keeping etcd healthy.

### 11.7.1 Cluster Sizing: 3 vs 5 Nodes

| Cluster Size | Fault Tolerance | Quorum | Use Case                    |
|-------------|-----------------|--------|-----------------------------|
| 1           | 0 failures      | 1      | Development only            |
| 3           | 1 failure       | 2      | Most production clusters    |
| 5           | 2 failures      | 3      | High-availability critical  |
| 7           | 3 failures      | 4      | Rarely needed               |

**Why odd numbers?** A cluster of 2N+1 nodes tolerates N failures (same as 2N+2
nodes), but with lower quorum overhead. A 4-node cluster tolerates 1 failure
(same as 3 nodes) but requires 3 nodes for quorum instead of 2. The extra node
adds latency (more nodes to replicate to) without improving fault tolerance.

**Recommendation**: Use 3 nodes for most clusters. Use 5 nodes only if you need
to tolerate 2 simultaneous failures (e.g., during rolling upgrades in a cluster
that spans 3 availability zones).

### 11.7.2 Performance Tuning

etcd performance is dominated by **disk I/O** and **network latency**:

**Disk**:
- Use **SSD** or NVMe storage. etcd performs sequential writes to the WAL and
  random reads from bbolt. Spinning disks are far too slow.
- Dedicate a disk to etcd. Sharing with other I/O-heavy workloads causes
  latency spikes.
- Target **< 10ms** for WAL `fsync` latency. Measure with:
  ```bash
  # Measure WAL fsync performance. This writes 22MB sequentially
  # and measures the fsync time — simulating etcd's WAL pattern.
  fio --rw=write --ioengine=sync --fdatasync=1 \
      --directory=/var/lib/etcd --size=22m \
      --bs=2300 --name=etcd-wal-test
  ```

**Network**:
- Network round-trip time between etcd members should be **< 10ms** (ideally
  < 2ms within the same data center).
- etcd members in different availability zones within the same region: typically
  1–3ms, acceptable.
- etcd members across regions: typically 50–100ms, **not recommended** (Raft
  heartbeats will timeout frequently).

**Tuning parameters**:
```bash
# Increase heartbeat interval (default: 100ms) and election timeout
# (default: 1000ms) for high-latency networks.
etcd --heartbeat-interval=250 --election-timeout=2500

# Increase the backend batch interval for write-heavy workloads.
# This reduces fsync frequency at the cost of slightly higher latency.
etcd --backend-batch-interval=25ms

# Increase the snapshot count (default: 100000) to reduce snapshot
# frequency for clusters with very high write rates.
etcd --snapshot-count=100000
```

### 11.7.3 Backup and Restore

etcd backup is **non-negotiable** in production. A lost etcd cluster without
backup means a lost Kubernetes cluster.

```bash
# ──────────────────────────────────────────────────────────────────
# BACKUP — Save a consistent snapshot of the entire etcd database.
# This is an atomic, point-in-time snapshot that can be restored
# to recover from data loss or cluster failure.
# ──────────────────────────────────────────────────────────────────
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-$(date +%Y%m%d-%H%M%S).db \
  --endpoints=https://etcd1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/client.crt \
  --key=/etc/etcd/client.key

# Verify the snapshot. This checks the integrity (hash) and prints
# metadata (revision, total keys, db size).
etcdctl snapshot status /backup/etcd-snapshot-20240115-143000.db \
  --write-out=table

# Sample output:
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | 5c32f794 |   182340 |       1256 |     5.4 MB |
# +----------+----------+------------+------------+

# ──────────────────────────────────────────────────────────────────
# RESTORE — Rebuild an etcd data directory from a snapshot.
# This creates a NEW cluster from the snapshot data.
# ──────────────────────────────────────────────────────────────────

# Stop etcd on all nodes first, then restore on each node with its
# own name and peer URLs.

# On node 1:
etcdctl snapshot restore /backup/etcd-snapshot-20240115-143000.db \
  --name node1 \
  --initial-cluster "node1=https://etcd1:2380,node2=https://etcd2:2380,node3=https://etcd3:2380" \
  --initial-cluster-token etcd-cluster-restored \
  --initial-advertise-peer-urls https://etcd1:2380 \
  --data-dir /var/lib/etcd-restored

# On node 2:
etcdctl snapshot restore /backup/etcd-snapshot-20240115-143000.db \
  --name node2 \
  --initial-cluster "node1=https://etcd1:2380,node2=https://etcd2:2380,node3=https://etcd3:2380" \
  --initial-cluster-token etcd-cluster-restored \
  --initial-advertise-peer-urls https://etcd2:2380 \
  --data-dir /var/lib/etcd-restored

# On node 3:
etcdctl snapshot restore /backup/etcd-snapshot-20240115-143000.db \
  --name node3 \
  --initial-cluster "node1=https://etcd1:2380,node2=https://etcd2:2380,node3=https://etcd3:2380" \
  --initial-cluster-token etcd-cluster-restored \
  --initial-advertise-peer-urls https://etcd3:2380 \
  --data-dir /var/lib/etcd-restored

# Start etcd on all nodes pointing to the restored data directory.
# The cluster will form and begin serving immediately.
```

### 11.7.4 Compaction and Defragmentation

```bash
# ──────────────────────────────────────────────────────────────────
# COMPACTION — Remove old revisions to free logical space.
# Without compaction, the database grows indefinitely.
# ──────────────────────────────────────────────────────────────────

# Compact all revisions up to revision 100000. After this, any
# reads or watches at revisions <= 100000 will fail with
# "mvcc: required revision has been compacted".
etcdctl compact 100000

# In production, enable AUTO-COMPACTION:
etcd --auto-compaction-mode=periodic --auto-compaction-retention=1h
# or
etcd --auto-compaction-mode=revision --auto-compaction-retention=10000

# ──────────────────────────────────────────────────────────────────
# DEFRAGMENTATION — Reclaim physical disk space.
# Compaction marks space as free but doesn't shrink the file.
# Defrag rewrites the bbolt file to reclaim that space.
# ──────────────────────────────────────────────────────────────────

# WARNING: Defrag temporarily locks the member and may cause
# increased latency. Run it during maintenance windows and
# defrag ONE member at a time.
etcdctl defrag --endpoints=https://etcd1:2379 \
  --cacert=/etc/etcd/ca.crt \
  --cert=/etc/etcd/client.crt \
  --key=/etc/etcd/client.key

# Check db size before and after:
etcdctl endpoint status --write-out=table
```

### 11.7.5 Monitoring: Prometheus Metrics

etcd exposes Prometheus metrics on `:2379/metrics`. Key metrics to monitor:

| Metric                                  | Description                        | Alert Threshold    |
|-----------------------------------------|------------------------------------|--------------------|
| `etcd_server_has_leader`                | 1 if member has a leader           | Alert if 0         |
| `etcd_server_leader_changes_seen_total` | Leader changes since start         | Alert if > 3/hour  |
| `etcd_server_proposals_failed_total`    | Failed Raft proposals              | Alert if increasing|
| `etcd_server_proposals_pending`         | Pending Raft proposals             | Alert if > 5       |
| `etcd_disk_wal_fsync_duration_seconds`  | WAL fsync latency                  | Alert if p99 > 10ms|
| `etcd_disk_backend_commit_duration`     | Backend commit latency             | Alert if p99 > 25ms|
| `etcd_mvcc_db_total_size_in_bytes`      | Total bbolt database size          | Alert if > 6GB     |
| `etcd_mvcc_db_total_size_in_use_in_bytes`| Logical data size (after compaction)| —                |
| `etcd_network_peer_round_trip_time`     | Peer RTT histogram                 | Alert if p99 > 50ms|
| `etcd_debugging_mvcc_keys_total`        | Total number of keys               | Informational      |
| `grpc_server_handled_total`             | gRPC requests by method and status | Alert on error rate |

**Prometheus scrape config**:

```yaml
# prometheus.yml — etcd scrape configuration.
# Assumes etcd is running with --listen-metrics-urls=http://0.0.0.0:2381
# (separate port for metrics to avoid TLS complexity).
scrape_configs:
  - job_name: 'etcd'
    scrape_interval: 15s
    static_configs:
      - targets:
        - 'etcd1:2381'
        - 'etcd2:2381'
        - 'etcd3:2381'
    # If metrics are on the client port with TLS:
    # scheme: https
    # tls_config:
    #   ca_file: /etc/prometheus/etcd-ca.crt
    #   cert_file: /etc/prometheus/etcd-client.crt
    #   key_file: /etc/prometheus/etcd-client.key
```

**Grafana dashboard**: The etcd team maintains an official Grafana dashboard
(ID: 3070) that visualizes these metrics.

### 11.7.6 Common Failure Modes and Recovery

| Failure                          | Symptom                                | Recovery                                      |
|----------------------------------|----------------------------------------|-----------------------------------------------|
| Single member crash              | Cluster continues (if majority alive)  | Restart the member; it rejoins automatically   |
| Majority members down            | Cluster is read-only (no writes)       | Restart enough members to restore quorum       |
| All members down                 | Complete outage                         | Start all members; they resume from WAL/snap   |
| Disk full                        | `NOSPACE` alarm, writes rejected       | Free space, then: `etcdctl alarm disarm`       |
| Slow disk (fsync > 100ms)        | Leader step-downs, timeouts            | Move to SSD, reduce I/O contention             |
| Network partition (split brain)  | Minority side loses leader             | Fix network; minority side auto-rejoins        |
| Data corruption                  | Hash mismatch between members          | Remove corrupted member, add new member        |
| Database too large (> 8GB)       | Performance degradation                | Compact and defrag, or increase `--quota-backend-bytes` |

### 11.7.7 Hands-on: Setting Up a 3-Node etcd Cluster

```bash
#!/bin/bash
# setup-etcd-cluster.sh
#
# This script sets up a production-like 3-node etcd cluster on a single
# machine (for learning purposes). In production, each node would run
# on a separate host.
#
# The cluster uses:
#   - node1: client=2379, peer=2380
#   - node2: client=2479, peer=2480
#   - node3: client=2579, peer=2580

set -euo pipefail

# Define the cluster configuration. In production, replace localhost
# with actual hostnames or IPs.
CLUSTER="node1=http://localhost:2380,node2=http://localhost:2480,node3=http://localhost:2580"
TOKEN="my-etcd-cluster"

# Clean up any previous data.
rm -rf /tmp/etcd-data-{1,2,3}

# Start node 1 in the background.
etcd \
  --name node1 \
  --data-dir /tmp/etcd-data-1 \
  --initial-advertise-peer-urls http://localhost:2380 \
  --listen-peer-urls http://localhost:2380 \
  --listen-client-urls http://localhost:2379 \
  --advertise-client-urls http://localhost:2379 \
  --initial-cluster "$CLUSTER" \
  --initial-cluster-token "$TOKEN" \
  --initial-cluster-state new \
  --listen-metrics-urls http://localhost:2381 \
  --auto-compaction-mode periodic \
  --auto-compaction-retention 1h &

# Start node 2 in the background.
etcd \
  --name node2 \
  --data-dir /tmp/etcd-data-2 \
  --initial-advertise-peer-urls http://localhost:2480 \
  --listen-peer-urls http://localhost:2480 \
  --listen-client-urls http://localhost:2479 \
  --advertise-client-urls http://localhost:2479 \
  --initial-cluster "$CLUSTER" \
  --initial-cluster-token "$TOKEN" \
  --initial-cluster-state new \
  --listen-metrics-urls http://localhost:2481 \
  --auto-compaction-mode periodic \
  --auto-compaction-retention 1h &

# Start node 3 in the background.
etcd \
  --name node3 \
  --data-dir /tmp/etcd-data-3 \
  --initial-advertise-peer-urls http://localhost:2580 \
  --listen-peer-urls http://localhost:2580 \
  --listen-client-urls http://localhost:2579 \
  --advertise-client-urls http://localhost:2579 \
  --initial-cluster "$CLUSTER" \
  --initial-cluster-token "$TOKEN" \
  --initial-cluster-state new \
  --listen-metrics-urls http://localhost:2581 \
  --auto-compaction-mode periodic \
  --auto-compaction-retention 1h &

# Wait for the cluster to form.
sleep 3

# Verify the cluster.
echo "=== Cluster Member List ==="
etcdctl member list \
  --endpoints=localhost:2379,localhost:2479,localhost:2579 \
  --write-out=table

echo ""
echo "=== Endpoint Status ==="
etcdctl endpoint status \
  --endpoints=localhost:2379,localhost:2479,localhost:2579 \
  --write-out=table

echo ""
echo "=== Endpoint Health ==="
etcdctl endpoint health \
  --endpoints=localhost:2379,localhost:2479,localhost:2579 \
  --write-out=table

echo ""
echo "Cluster is ready! Endpoints: localhost:2379, localhost:2479, localhost:2579"
```

### 11.7.8 Hands-on: Performing Backup and Restore

```go
// File: backup/main.go
//
// This program demonstrates programmatic etcd backup (snapshot) and
// verification using the Go client. In production, you would typically
// run this as a CronJob in Kubernetes or as a scheduled task.
//
// The program:
//   1. Takes a snapshot of the etcd database.
//   2. Verifies the snapshot integrity.
//   3. Reports the snapshot metadata (revision, key count, size).
//
// For restore, use the etcdctl command-line tool (shown in comments).
package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"os"
	"time"

	clientv3 "go.etcd.io/etcd/client/v3"
)

func main() {
	// Connect to the etcd cluster.
	client, err := clientv3.New(clientv3.Config{
			Endpoints:   []string{"localhost:2379"},
			DialTimeout: 5 * time.Second,
		})
	if err != nil {
		log.Fatalf("Failed to create etcd client: %v", err)
	}
	defer client.Close()

	// Generate a timestamped filename for the snapshot.
	snapshotFile := fmt.Sprintf("etcd-snapshot-%s.db",
		time.Now().Format("20060102-150405"))

	fmt.Printf("Taking etcd snapshot to: %s\n", snapshotFile)

	// Create the snapshot context with a generous timeout.
	// Snapshots can take several seconds for large databases.
	ctx, cancel := context.WithTimeout(context.Background(), 60*time.Second)
	defer cancel()

	// Request a snapshot from the maintenance API. This returns a
	// ReadCloser that streams the entire database contents.
	// The snapshot is taken at a consistent point-in-time.
	snapshotReader, err := client.Snapshot(ctx)
	if err != nil {
		log.Fatalf("Snapshot failed: %v", err)
	}
	defer snapshotReader.Close()

	// Write the snapshot to a file.
	f, err := os.Create(snapshotFile)
	if err != nil {
		log.Fatalf("Failed to create file: %v", err)
	}
	defer f.Close()

	// Copy the snapshot stream to the file. For large databases,
	// this streams data incrementally to avoid loading the entire
	// snapshot into memory.
	written, err := io.Copy(f, snapshotReader)
	if err != nil {
		log.Fatalf("Failed to write snapshot: %v", err)
	}

	fmt.Printf("Snapshot saved: %s (%d bytes)\n", snapshotFile, written)

	// Get the cluster status to report the revision at snapshot time.
	status, err := client.Status(ctx, "localhost:2379")
	if err != nil {
		log.Printf("Warning: could not get status: %v", err)
	} else {
		fmt.Printf("Cluster revision at snapshot: %d\n", status.Header.Revision)
		fmt.Printf("Database size: %d bytes\n", status.DbSize)
	}

	fmt.Println("\nTo restore from this snapshot:")
	fmt.Printf("  etcdctl snapshot restore %s \\\n", snapshotFile)
	fmt.Println("    --name node1 \\")
	fmt.Println("    --initial-cluster node1=http://localhost:2380 \\")
	fmt.Println("    --initial-advertise-peer-urls http://localhost:2380 \\")
	fmt.Println("    --data-dir /var/lib/etcd-restored")
}
```

### 11.7.9 Hands-on: Monitoring etcd Health and Metrics

```go
// File: monitor/main.go
//
// This program implements a simple etcd health monitor that periodically
// checks the health of each cluster member and reports key metrics.
//
// In production, you would use Prometheus + Grafana for monitoring.
// This program demonstrates the concepts by querying the same metrics
// programmatically via the Go client.
//
// The monitor checks:
//   - Is each endpoint healthy? (can it serve requests?)
//   - Who is the current leader?
//   - What is the database size on each member?
//   - Are all members at similar Raft indices? (replication lag)
//   - What alarms are active? (e.g., NOSPACE)
package main

import (
	"context"
	"fmt"
	"log"
	"math"
	"time"

	clientv3 "go.etcd.io/etcd/client/v3"
)

func main() {
	// Define the cluster endpoints.
	endpoints := []string{
		"localhost:2379",
		"localhost:2479",
		"localhost:2579",
	}

	// Connect to the cluster.
	client, err := clientv3.New(clientv3.Config{
			Endpoints:   endpoints,
			DialTimeout: 5 * time.Second,
		})
	if err != nil {
		log.Fatalf("Failed to create etcd client: %v", err)
	}
	defer client.Close()

	// Run the health check once. In production, run this in a loop
	// or as a Prometheus exporter.
	checkClusterHealth(client, endpoints)
}

// checkClusterHealth queries each etcd member for its status and
// reports the cluster health. It checks for:
//   - Endpoint reachability
//   - Leader presence
//   - Database size consistency
//   - Raft index divergence (replication lag)
//   - Active alarms
//
// Parameters:
//   - client:    the etcd client connection
//   - endpoints: the list of etcd member addresses to check
func checkClusterHealth(client *clientv3.Client, endpoints []string) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	fmt.Println("╔══════════════════════════════════════════════════════════════╗")
	fmt.Println("║                   etcd Cluster Health Report                ║")
	fmt.Println("╠══════════════════════════════════════════════════════════════╣")

	// Collect status from each endpoint.
	type memberStatus struct {
		endpoint  string
		leader    uint64
		dbSize    int64
		raftIndex uint64
		raftTerm  uint64
		healthy   bool
		err       error
	}

	statuses := make([]memberStatus, len(endpoints))

	for i, ep := range endpoints {
		// Query the status of each member. This returns the member's
		// current Raft state, database size, and leader information.
		status, err := client.Status(ctx, ep)
		if err != nil {
			statuses[i] = memberStatus{endpoint: ep, healthy: false, err: err}
			continue
		}

		statuses[i] = memberStatus{
			endpoint:  ep,
			leader:    status.Leader,
			dbSize:    status.DbSize,
			raftIndex: status.RaftIndex,
			raftTerm:  status.RaftTerm,
			healthy:   true,
		}
	}

	// Print per-member status.
	fmt.Println("║                                                              ║")
	fmt.Println("║  Member Status:                                              ║")

	var maxRaftIndex uint64
	var leaderEndpoint string

	for _, s := range statuses {
		if !s.healthy {
			fmt.Printf("║  ✗ %-20s UNHEALTHY: %v\n", s.endpoint, s.err)
			continue
		}

		isLeader := ""
		// The Status response's Leader field tells us which member ID
		// is the current leader. We compare the endpoint's ID to
		// determine if this endpoint IS the leader.
		// (Simplified: in production, compare member IDs properly.)
		if s.raftIndex > maxRaftIndex {
			maxRaftIndex = s.raftIndex
		}

		// Check if this member's header.member_id equals the leader ID.
		// For simplicity, mark the first healthy endpoint with leader
		// info as the leader endpoint.
		if leaderEndpoint == "" {
			leaderEndpoint = s.endpoint
			isLeader = " ★ LEADER"
		}

		fmt.Printf("║  ✓ %-20s DB: %6d KB  Raft: idx=%d term=%d%s\n",
			s.endpoint, s.dbSize/1024, s.raftIndex, s.raftTerm, isLeader)
	}

	// Check for replication lag.
	fmt.Println("║                                                              ║")
	fmt.Println("║  Replication Status:                                         ║")

	for _, s := range statuses {
		if !s.healthy {
			continue
		}

		lag := maxRaftIndex - s.raftIndex
		status := "OK"
		if lag > 100 {
			status = "⚠ LAGGING"
		}
		if lag > 1000 {
			status = "✗ SEVERELY LAGGING"
		}

		fmt.Printf("║    %-20s lag: %d entries  [%s]\n",
			s.endpoint, lag, status)
	}

	// Check for alarms.
	fmt.Println("║                                                              ║")
	fmt.Println("║  Alarms:                                                     ║")

	alarmResp, err := client.AlarmList(ctx)
	if err != nil {
		fmt.Printf("║    Failed to get alarms: %v\n", err)
	} else if len(alarmResp.Alarms) == 0 {
		fmt.Println("║    No active alarms ✓                                        ║")
	} else {
		for _, alarm := range alarmResp.Alarms {
			fmt.Printf("║    ⚠ ALARM: member=%x type=%v\n",
				alarm.MemberID, alarm.Alarm)
		}
	}

	// Check database size consistency.
	fmt.Println("║                                                              ║")
	fmt.Println("║  Database Size Consistency:                                  ║")

	var minSize, maxSize int64
	minSize = math.MaxInt64

	for _, s := range statuses {
		if !s.healthy {
			continue
		}
		if s.dbSize < minSize {
			minSize = s.dbSize
		}
		if s.dbSize > maxSize {
			maxSize = s.dbSize
		}
	}

	sizeDiffPercent := float64(0)
	if minSize > 0 {
		sizeDiffPercent = float64(maxSize-minSize) / float64(minSize) * 100
	}

	if sizeDiffPercent < 10 {
		fmt.Printf("║    Size variance: %.1f%% ✓ (min=%dKB, max=%dKB)\n",
			sizeDiffPercent, minSize/1024, maxSize/1024)
	} else {
		fmt.Printf("║    ⚠ Size variance: %.1f%% (min=%dKB, max=%dKB)\n",
			sizeDiffPercent, minSize/1024, maxSize/1024)
		fmt.Println("║    Consider running defrag on the larger members.")
	}

	fmt.Println("║                                                              ║")
	fmt.Println("╚══════════════════════════════════════════════════════════════╝")
}
```

---

## 11.8 etcd Security

In production, etcd must be locked down. It contains **every secret** in your
Kubernetes cluster — tokens, passwords, TLS certificates — all in plaintext
(unless encryption at rest is enabled in Kubernetes).

### 11.8.1 TLS for Peer and Client Communication

etcd supports TLS for two communication channels:

1. **Client-to-server**: all gRPC API requests (Put, Get, Watch, etc.)
2. **Peer-to-peer**: Raft messages, snapshot transfers between members

```
                    TLS (client certs)
    Client ◄──────────────────────────► etcd Member 1
                                              │
                                    TLS (peer │ certs)
                                              │
                                        etcd Member 2
                                              │
                                    TLS (peer │ certs)
                                              │
                                        etcd Member 3
```

### 11.8.2 Authentication and RBAC

etcd supports username/password authentication and role-based access control:

```bash
# Enable authentication (requires a root user).
etcdctl user add root --new-user-password="rootpassword"
etcdctl auth enable

# Create a user for the Kubernetes API server.
etcdctl user add kube-apiserver --new-user-password="apiserver-pass"

# Create a role with full access to the /registry prefix.
etcdctl role add kube-role
etcdctl role grant-permission kube-role readwrite /registry/ --prefix

# Assign the role to the user.
etcdctl user grant-role kube-apiserver kube-role

# Now the kube-apiserver user can only access keys under /registry/.
# Attempts to access other keys will be denied.
```

### 11.8.3 Hands-on: Configuring TLS for etcd

```bash
#!/bin/bash
# setup-etcd-tls.sh
#
# This script generates TLS certificates for a 3-node etcd cluster
# using cfssl (CloudFlare's PKI toolkit). It creates:
#
#   1. A Certificate Authority (CA) — the root of trust
#   2. Server certificates — for etcd members (peer + client)
#   3. Client certificates — for etcdctl and API server
#
# In production, use your organization's PKI or a tool like cert-manager.

set -euo pipefail

# Install cfssl if not present.
# go install github.com/cloudflare/cfssl/cmd/cfssl@latest
# go install github.com/cloudflare/cfssl/cmd/cfssljson@latest

CERT_DIR="./certs"
mkdir -p "$CERT_DIR"

# ────────────────────────────────────────────────────────────────
# Step 1: Create the Certificate Authority (CA)
# ────────────────────────────────────────────────────────────────

cat > "$CERT_DIR/ca-csr.json" <<EOF
{
  "CN": "etcd CA",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "etcd",
      "OU": "etcd Security"
    }
  ]
}
EOF

cat > "$CERT_DIR/ca-config.json" <<EOF
{
  "signing": {
    "default": {
      "expiry": "8760h"
    },
    "profiles": {
      "server": {
        "expiry": "8760h",
        "usages": [
          "signing",
          "key encipherment",
          "server auth",
          "client auth"
        ]
      },
      "client": {
        "expiry": "8760h",
        "usages": [
          "signing",
          "key encipherment",
          "client auth"
        ]
      }
    }
  }
}
EOF

# Generate the CA certificate and key.
cfssl gencert -initca "$CERT_DIR/ca-csr.json" | cfssljson -bare "$CERT_DIR/ca"
echo "✓ CA certificate generated"

# ────────────────────────────────────────────────────────────────
# Step 2: Create server certificates for each etcd member
# ────────────────────────────────────────────────────────────────

# Each etcd member needs a certificate that includes both its
# hostname and IP in the SANs (Subject Alternative Names).
for i in 1 2 3; do
  cat > "$CERT_DIR/etcd${i}-csr.json" <<EOF
{
  "CN": "etcd${i}",
  "hosts": [
    "etcd${i}",
    "etcd${i}.example.com",
    "127.0.0.1",
    "localhost"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "etcd",
      "OU": "etcd Node"
    }
  ]
}
EOF

  cfssl gencert \
    -ca="$CERT_DIR/ca.pem" \
    -ca-key="$CERT_DIR/ca-key.pem" \
    -config="$CERT_DIR/ca-config.json" \
    -profile=server \
    "$CERT_DIR/etcd${i}-csr.json" | cfssljson -bare "$CERT_DIR/etcd${i}"

  echo "✓ Server certificate generated for etcd${i}"
done

# ────────────────────────────────────────────────────────────────
# Step 3: Create client certificate (for etcdctl, API server)
# ────────────────────────────────────────────────────────────────

cat > "$CERT_DIR/client-csr.json" <<EOF
{
  "CN": "etcd-client",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "O": "etcd",
      "OU": "etcd Client"
    }
  ]
}
EOF

cfssl gencert \
  -ca="$CERT_DIR/ca.pem" \
  -ca-key="$CERT_DIR/ca-key.pem" \
  -config="$CERT_DIR/ca-config.json" \
  -profile=client \
  "$CERT_DIR/client-csr.json" | cfssljson -bare "$CERT_DIR/client"

echo "✓ Client certificate generated"

# ────────────────────────────────────────────────────────────────
# Step 4: Start etcd with TLS
# ────────────────────────────────────────────────────────────────

echo ""
echo "To start etcd with TLS:"
echo ""
echo "etcd --name node1 \\"
echo "  --cert-file=$CERT_DIR/etcd1.pem \\"
echo "  --key-file=$CERT_DIR/etcd1-key.pem \\"
echo "  --trusted-ca-file=$CERT_DIR/ca.pem \\"
echo "  --client-cert-auth \\"
echo "  --peer-cert-file=$CERT_DIR/etcd1.pem \\"
echo "  --peer-key-file=$CERT_DIR/etcd1-key.pem \\"
echo "  --peer-trusted-ca-file=$CERT_DIR/ca.pem \\"
echo "  --peer-client-cert-auth \\"
echo "  --listen-client-urls https://0.0.0.0:2379 \\"
echo "  --advertise-client-urls https://etcd1:2379 \\"
echo "  --listen-peer-urls https://0.0.0.0:2380 \\"
echo "  --initial-advertise-peer-urls https://etcd1:2380 \\"
echo "  --initial-cluster 'node1=https://etcd1:2380,...'"
echo ""
echo "To use etcdctl with TLS:"
echo ""
echo "etcdctl --endpoints=https://localhost:2379 \\"
echo "  --cacert=$CERT_DIR/ca.pem \\"
echo "  --cert=$CERT_DIR/client.pem \\"
echo "  --key=$CERT_DIR/client-key.pem \\"
echo "  endpoint health"
```

---

## 11.9 Performance Considerations

Understanding etcd's performance characteristics is essential for designing
systems that use it effectively and for diagnosing bottlenecks.

### 11.9.1 Read vs Write Throughput

etcd's read and write performance are fundamentally different:

**Writes** (linearizable):
- Must go through the Raft leader.
- Must be replicated to a majority of members.
- Require at least one WAL `fsync` on the leader.
- Typical throughput: **10,000–30,000 writes/sec** (depends on disk and
  value size).
- Latency: **2–10ms** p99 (SSD, same datacenter).

**Reads** (linearizable, the default):
- Must go through the leader (or the leader must confirm it's still leading).
- ReadIndex optimization: follower asks leader "are you still leader?",
  leader confirms, follower serves the read locally.
- Typical throughput: **50,000–100,000 reads/sec**.
- Latency: **1–5ms** p99.

**Reads** (serializable):
- Can be served by any member, no Raft round-trip.
- May return stale data (the member may lag behind the leader).
- Much higher throughput and lower latency.
- Use for data that tolerates staleness.

```go
// Serializable read — fast but potentially stale.
// Use clientv3.WithSerializable() to opt into serializable reads.
// This skips the Raft consensus check, so the response may lag
// behind the leader by a few milliseconds.
resp, err := client.Get(ctx, "/my-key", clientv3.WithSerializable())
```

### 11.9.2 Linearizable vs Serializable Reads — When to Use Each

| Read Type      | Consistency   | Latency  | Use Case                          |
|---------------|---------------|----------|-----------------------------------|
| Linearizable  | Latest value  | Higher   | Default. Critical state reads.    |
| Serializable  | May be stale  | Lower    | Metrics, UI display, caching.     |

Kubernetes uses **linearizable reads by default** for all API server operations.
This ensures that when you `kubectl get pod`, you see the latest state. However,
the API server's **watch cache** serves most reads from memory, avoiding the
etcd round-trip entirely.

### 11.9.3 Key and Value Size Limits

| Limit                   | Value        | Notes                                |
|------------------------|--------------|--------------------------------------|
| Max key size            | 1.5 MB       | Hard limit. Keep keys short.         |
| Max value size          | 1.5 MB       | Hard limit per value.                |
| Max request size        | 1.5 MB       | Total request including all keys.    |
| Max database size       | 8 GB default | Configurable via `--quota-backend-bytes`. |
| Recommended key size    | < 256 bytes  | Short keys are faster to compare.    |
| Recommended value size  | < 256 KB     | Large values slow Raft replication.  |

In Kubernetes, most objects are well under 1 MB. However, **large Custom
Resources**, **ConfigMaps with embedded files**, or **Secrets with large
certificates** can approach the limit. The Kubernetes API server enforces a
default 1.5 MB limit on object size that matches etcd's.

### 11.9.4 When etcd Becomes the Bottleneck

etcd is designed for **metadata storage**, not bulk data. Signs that etcd is
the bottleneck:

1. **High write latency** (WAL fsync > 10ms): Move to SSD. Dedicate the disk.
2. **Leader changes frequently**: Network latency too high or disk too slow.
   Increase `--heartbeat-interval` and `--election-timeout`.
3. **Database size approaching quota**: Too many objects or too much history.
   Enable auto-compaction. Consider whether all that data belongs in etcd.
4. **Watch event backlog**: Too many watchers or too high event rate.
   This is rare but can happen with thousands of controllers.
5. **gRPC request queue depth high**: Too many concurrent requests. Scale
   the number of API servers (not etcd members — adding members hurts write
   performance).

**General rule**: If you're storing more than **100,000 keys** or your database
is larger than **1 GB**, carefully evaluate whether etcd is the right store for
your workload. etcd is for **coordination data**, not application data.

### 11.9.5 Benchmarking etcd

etcd ships with a built-in benchmarking tool:

```bash
# Write benchmark: 10,000 keys with 256-byte values.
# Uses 50 concurrent clients and 200 total connections.
benchmark put \
  --endpoints=localhost:2379 \
  --total=10000 \
  --val-size=256 \
  --conns=200 \
  --clients=50

# Read benchmark: 10,000 range requests.
benchmark range "/" \
  --endpoints=localhost:2379 \
  --total=10000 \
  --conns=200 \
  --clients=50

# Sequential key write benchmark (simulates ordered key patterns
# like Kubernetes uses for pods in a namespace).
benchmark put \
  --endpoints=localhost:2379 \
  --total=10000 \
  --val-size=1024 \
  --key-size=64 \
  --sequential-keys \
  --conns=100 \
  --clients=50
```

---

## 11.10 Putting It All Together: etcd in the Kubernetes Architecture

Let us trace a complete Kubernetes operation — creating a pod — through etcd:

```
kubectl create pod nginx
        │
        ▼
┌──────────────────┐
│  API Server      │
│                  │
│  1. Authenticate │
│  2. Authorize    │
│  3. Validate     │
│  4. Admission    │
│     control      │
│                  │
│  5. Serialize    │
│     to protobuf  │
│                  │
│  6. etcd Txn:    │ ───► etcd Leader
│     IF key not   │      │
│     exists       │      ├─ Append to Raft log
│     THEN Put     │      ├─ Replicate to followers
│                  │      ├─ Commit after majority ack
│  7. Wait for     │      ├─ Apply to MVCC store
│     commit       │      ├─ Increment revision
│                  │ ◄─── └─ Return success
│  8. Return       │
│     response     │
│     with         │
│     resourceVer  │
└────────┬─────────┘
         │
         │  Watch event (via gRPC stream)
         ▼
┌──────────────────┐
│  Scheduler       │
│                  │
│  Receives ADDED  │
│  event for pod   │
│  with no node    │
│  assigned.       │
│                  │
│  Selects a node  │
│  and updates the │
│  pod's spec.     │
│  nodeName via    │
│  API server.     │
└────────┬─────────┘
         │
         │  Another etcd Txn (update pod with nodeName)
         ▼
┌──────────────────┐
│  Kubelet (node)  │
│                  │
│  Watches for     │
│  pods assigned   │
│  to this node.   │
│                  │
│  Receives the    │
│  pod, pulls      │
│  images, starts  │
│  containers.     │
│                  │
│  Updates pod     │
│  status via      │
│  API server.     │
└──────────────────┘
         │
         │  Another etcd Txn (update pod status)
         ▼
    Pod is running!
```

Every step that modifies state goes through etcd. Every step that reacts to
state changes uses etcd's watch mechanism (via the API server). etcd is the
single source of truth, the memory that makes the entire system work.

---

## 11.11 Summary

In this chapter we have journeyed through etcd from theory to practice:

| Topic                  | Key Takeaway                                                    |
|------------------------|------------------------------------------------------------------|
| **Raft**               | Consensus via leader election + log replication. Majority quorum.|
| **MVCC**               | Every mutation creates a new revision. Enables watch from past.  |
| **Storage**            | bbolt B+tree for data, WAL for durability, snapshots for recovery.|
| **API**                | KV, Watch, Lease, Txn, Auth, Maintenance — all over gRPC.       |
| **Kubernetes**         | `/registry/` prefix. resourceVersion = mod_revision.             |
| **Operations**         | 3 or 5 nodes. SSD required. Backup religiously. Monitor closely. |
| **Security**           | TLS for all communication. RBAC for access control.              |
| **Performance**        | 10K–30K writes/s. Keep keys small. etcd is for metadata only.    |

etcd is the most critical component in your Kubernetes cluster. It is small,
focused, and elegant — but it demands respect. Treat it well: give it fast
disks, reliable networks, proper monitoring, and regular backups. In return,
it will faithfully store every piece of state your cluster needs.

> *"Take care of etcd, and etcd will take care of your cluster."*

---

## Exercises

1. **Raft Visualization**: Set up a 3-node etcd cluster. Kill the leader and
   observe the election logs. Measure the time to elect a new leader. How does
   the election timeout affect this time?

2. **MVCC Exploration**: Write a Go program that writes 100 sequential updates
   to a key, then reads the key at each of the 100 revisions. Verify that each
   revision returns the correct value.

3. **Watch Reliability**: Write a Go program that starts a watch at revision 1,
   then kills and restarts the etcd connection. Does the watch resume correctly?
   What happens if the revision was compacted?

4. **Distributed Lock Race**: Launch 10 goroutines competing for a distributed
   lock. Add a shared counter that each goroutine increments 100 times while
   holding the lock. Verify the final count is 1000 (no race conditions).

5. **Transaction Challenge**: Implement a simple distributed counter using only
   etcd transactions (no distributed lock). The counter should correctly
   increment even with concurrent incrementers.

6. **Backup Automation**: Write a Go program that takes an etcd snapshot every
   hour and retains only the last 24 snapshots. Verify each snapshot's integrity.

7. **Performance Profiling**: Use the etcd benchmark tool to measure write
   throughput with different value sizes (64B, 256B, 1KB, 10KB, 100KB). Plot
   the results and explain the trend.

8. **Kubernetes Deep Dive**: In a `kind` cluster, use `etcdctl` to list all
   keys under `/registry/pods/`. Create a pod and observe the new key appear.
   Delete the pod and observe the key's tombstone.

---

## Further Reading

- [etcd Documentation](https://etcd.io/docs/) — Official documentation
- [The Raft Paper](https://raft.github.io/raft.pdf) — "In Search of an
  Understandable Consensus Algorithm" by Diego Ongaro and John Ousterhout
- [etcd Source Code](https://github.com/etcd-io/etcd) — Written in Go,
  well-structured and readable
- [Kubernetes etcd Administration](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/) — Official Kubernetes guide
- [bbolt Source Code](https://github.com/etcd-io/bbolt) — The storage engine
- [Raft Visualization](https://raft.github.io/) — Interactive Raft animation
