# 🏗️ The System Design Bible

> **48 case studies. Staff-to-Distinguished level depth. Zero fluff.**
>
> A comprehensive system design reference covering every major distributed systems pattern — from infrastructure primitives to planet-scale platforms. Each case study includes use cases, functional & non-functional requirements, capacity estimation, RESTful APIs, data models, Mermaid HLD diagrams, and ruthless deep dives.

---

## 📊 Book Stats

| Metric | Value |
|---|---|
| **Total Case Studies** | 48 |
| **Total Size** | ~626 KB |
| **Chapters** | 6 |
| **Diagrams** | 100+ Mermaid diagrams |
| **Target Level** | Staff / Principal / Distinguished Engineer |

---

## 📖 Table of Contents

### [Chapter 1: Infrastructure & Building Blocks](ch01-infrastructure-building-blocks.md)
> Core primitives that power every large-scale distributed system.

| # | Case Study | Key Concepts |
|---|---|---|
| 1 | **Rate Limiter** | Token bucket, sliding window, distributed rate limiting |
| 2 | **Consistent Hashing** | Virtual nodes, bounded loads, ring topology |
| 3 | **Unique/Distributed ID Generator** | Snowflake, UUID, clock skew, datacenter-aware |
| 4 | **Key-Value Store** | LSM trees, SSTables, Merkle trees, gossip protocol |
| 5 | **Distributed Cache** | Write-through, write-behind, cache-aside, eviction policies |
| 6 | **Load Balancer** | L4 vs L7, consistent hashing, health checks, session affinity |
| 7 | **API Gateway** | Routing, auth, rate limiting, circuit breaker, request aggregation |
| 8 | **Content Delivery Network (CDN)** | PoPs, cache hierarchy, origin shield, cache invalidation |

---

### [Chapter 2: Data & Storage Systems](ch02-data-storage-systems.md)
> The backbone of any distributed architecture — storing, querying, and moving data at scale.

| # | Case Study | Key Concepts |
|---|---|---|
| 1 | **Distributed Message Queue** | Partitioned log, ISR replication, exactly-once, zero-copy |
| 2 | **Distributed Logging & Tracing** | W3C trace context, adaptive sampling, ILM tiering |
| 3 | **Object Storage (S3)** | Erasure coding, consistent hashing, 11-nines durability |
| 4 | **Distributed File System** | HDFS/GFS, rack-aware placement, NameNode HA |
| 5 | **Search Engine** | Inverted indexes, BM25 scoring, segment merging |
| 6 | **Distributed Locking Service** | Raft consensus, fencing tokens, lease-based sessions |
| 7 | **Distributed Task Scheduler** | DAG dependencies, exactly-once dispatch, topological sort |
| 8 | **Time-Series Database** | Gorilla compression, tag indexing, downsampling |

---

### [Chapter 3: Social & Communication Platforms](ch03-social-communication-platforms.md)
> Connecting billions of users in real-time — the systems behind modern social networks and messaging.

| # | Case Study | Key Concepts |
|---|---|---|
| 1 | **URL Shortener** | Base62 encoding, 301 vs 302, analytics, bloom filters |
| 2 | **Pastebin** | Content-addressable storage, expiration, access control |
| 3 | **News Feed / Timeline** | Fan-out on write vs read, ranking, hybrid approach |
| 4 | **Twitter / Social Network** | Tweet ingestion, follower graph, trending, search |
| 5 | **Instagram / Photo Sharing** | Upload pipeline, feed generation, stories, explore |
| 6 | **Real-time Chat / WhatsApp** | WebSocket, E2E encryption, message ordering, presence |
| 7 | **Notification System** | Multi-channel delivery, priority queues, dedup, templates |
| 8 | **Email System** | SMTP pipeline, spam filtering, search, threading |

---

### [Chapter 4: Media, Content & Streaming](ch04-media-content-streaming.md)
> Delivering rich media experiences at planetary scale — from upload to playback.

| # | Case Study | Key Concepts |
|---|---|---|
| 1 | **YouTube / Video Sharing** | Resumable upload, GOP-parallel transcode, ABR streaming |
| 2 | **Live Streaming & Broadcast** | RTMP/SRT ingest, LL-HLS/WebRTC, real-time chat |
| 3 | **Spotify / Music Streaming** | Gapless playback, collaborative filtering, royalty counting |
| 4 | **Netflix / Video-on-Demand** | Per-title encoding, Open Connect CDN, A/B testing |
| 5 | **Google Drive / Dropbox** | Block-level delta sync, rolling hash, conflict resolution |
| 6 | **Google Docs / Collaborative Editing** | OT vs CRDTs, cursor presence, operation log |
| 7 | **Web Crawler** | URL frontier, BFS+priority, Simhash dedup, politeness |
| 8 | **Image Processing Pipeline** | DAG orchestration, streaming resize, ML moderation |

---

### [Chapter 5: Commerce & Business Platforms](ch05-commerce-business-platforms.md)
> Money, marketplaces, and mission-critical transactions — where correctness is non-negotiable.

| # | Case Study | Key Concepts |
|---|---|---|
| 1 | **E-Commerce Platform** | Catalog, cart, inventory, order management, microservices |
| 2 | **Payment System** | Idempotency, double-entry ledger, reconciliation, PCI-DSS |
| 3 | **Hotel/Flight Booking** | Availability search, reservation hold, overbooking |
| 4 | **Ticket Booking** | Seat locking, flash sale queues, fairness |
| 5 | **Ad Click Aggregation** | Real-time counting, fraud detection, lambda architecture |
| 6 | **Stock Exchange / Trading** | Order book, matching engine, market data feed |
| 7 | **Uber / Ride-Sharing** | Driver matching, ETA, surge pricing, trip management |
| 8 | **Proximity Service / Yelp** | Geohash, Quadtree, R-tree, S2 cells, ranking |

---

### [Chapter 6: Observability, Location & Advanced Systems](ch06-observability-maps-misc.md)
> The invisible backbone — monitoring, navigating, and orchestrating the systems that run the world.

| # | Case Study | Key Concepts |
|---|---|---|
| 1 | **Metrics Monitoring & Alerting** | Gorilla compression, downsampling, anomaly detection |
| 2 | **DNS System** | Recursive resolution, anycast, DNSSEC, GeoDNS |
| 3 | **Search Autocomplete / Typeahead** | Trie with top-K, trending detection, personalization |
| 4 | **Google Maps / Navigation** | Contraction hierarchies, vector tiles, traffic |
| 5 | **Distributed Configuration Service** | Raft consensus, watches, MVCC, leases |
| 6 | **Online Coding Judge** | Container sandboxing, seccomp, cgroups, DAG judging |
| 7 | **Digital Wallet / Ledger** | Double-entry bookkeeping, optimistic locking, idempotency |
| 8 | **CI/CD Pipeline** | DAG scheduling, ephemeral runners, secret management |

---

## 🧩 Case Study Format

Every case study follows a consistent, interview-ready structure:

```
1. Problem Statement        — What are we solving and why?
2. Use Cases                — Real-world scenarios
3. Functional Requirements  — What the system must do
4. Non-Functional Req.      — Latency, throughput, availability, durability
5. Capacity Estimation      — Back-of-envelope math with real numbers
6. API Design               — RESTful, collection-oriented, paginated
7. Data Model               — SQL schemas, indexes, partitioning
8. High-Level Design        — Mermaid architecture diagrams
9. Deep Dive                — Algorithms, trade-offs, failure modes
10. Bottlenecks             — Problems and mitigations table
11. Key Takeaways           — Summary bullets
```

---

## 🎯 How to Use This Book

### For Interview Prep
- Pick a case study → read Problem Statement + Requirements
- Try designing it yourself on a whiteboard
- Compare against the HLD and Deep Dive sections
- Review Bottlenecks — interviewers love asking about failure modes

### For Real-World Engineering
- Use Capacity Estimation as templates for your own systems
- Adapt the API designs and data models
- Reference the Deep Dive sections for production-grade decisions
- Cross-reference patterns across chapters (e.g., saga pattern appears in payments AND e-commerce)

### Cross-Cutting Patterns to Master
| Pattern | Where It Appears |
|---|---|
| Consistent Hashing | KV Store, Cache, CDN, Object Storage, Message Queue |
| Write-Ahead Log | Message Queue, DB, Collaborative Editing, Task Scheduler |
| Fan-out | News Feed, Notifications, Live Streaming, Chat |
| Saga / Outbox | E-Commerce, Payments, Booking, Wallet |
| CQRS / Event Sourcing | E-Commerce, Trading, Ad Analytics, Metrics |
| Bloom Filters | URL Shortener, Web Crawler, Cache, Search |
| Geospatial Indexing | Uber, Yelp, Google Maps, Delivery |
| Rate Limiting | API Gateway, Chat, Notifications, Trading |
| Circuit Breaker | API Gateway, Microservices, Payment, Booking |
| Leader Election | Distributed Lock, Config Service, Scheduler, FS |

---

## 📐 Diagram Legend

All architecture diagrams use [Mermaid](https://mermaid.js.org/) syntax. Render them with:
- **GitHub** — renders natively in `.md` files
- **VS Code** — install "Markdown Preview Mermaid Support" extension
- **Online** — [mermaid.live](https://mermaid.live)

---

*Built for engineers who refuse to be average.* 🚀
