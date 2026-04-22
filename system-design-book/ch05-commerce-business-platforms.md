# Chapter 5: Commerce & Business Platforms

> Money, marketplaces, and mission-critical transactions — designing systems where correctness and availability are non-negotiable.

Commerce platforms sit at the intersection of high throughput, strict consistency, and real-money consequences. A dropped packet in a social feed is invisible; a dropped packet in a payment flow can cost millions. This chapter dissects eight systems that collectively power the global digital economy — from the split-second matching of stock orders to the geospatial search that finds you a nearby restaurant. Each case study follows a progression: understand the problem, define the contract, estimate scale, design the data and service layers, then dive deep into the hardest engineering problems that make these systems fascinating.

---

## 1. E-Commerce Platform

### Problem Statement

An e-commerce platform enables millions of buyers and sellers to transact in real time. At its core, the system must manage a product catalog containing hundreds of millions of SKUs, maintain accurate inventory counts even under extreme concurrency, orchestrate a multi-step checkout flow that spans payment processing, fraud checks, and fulfillment, and deliver a personalized, sub-second search experience.

The engineering challenge lies in the tension between availability and consistency. Showing a product as "in stock" when the last unit was just sold leads to overselling — an expensive customer-service nightmare. Conversely, being overly conservative with inventory locks can reduce sell-through rates. Meanwhile, the system must handle flash-sale traffic spikes that can exceed 100× normal load in seconds.

Real-world platforms like Amazon process over 4,000 orders per second at peak and manage a catalog of 350+ million products. The system must be horizontally scalable, fault-tolerant across regions, and optimized for the 99th percentile latency to keep conversion rates high.

### Use Cases

- Browse and search the product catalog with faceted filters (category, price, brand, ratings)
- View detailed product pages with images, reviews, and availability
- Add/remove items to/from a persistent shopping cart
- Complete checkout with address selection, payment, and order confirmation
- Track order status from placement through delivery
- Sellers list new products, manage inventory, and view sales analytics
- Apply promotional codes and dynamic pricing rules
- Handle returns, refunds, and exchanges

### Functional Requirements

- FR1: Users can search products by keyword, category, and filters with relevance-ranked results
- FR2: The product detail page displays real-time price, availability, and estimated delivery date
- FR3: Authenticated users can maintain a persistent cart across devices
- FR4: Checkout must atomically reserve inventory, charge payment, and create an order
- FR5: Users receive order confirmation and can track status in real time
- FR6: Sellers can create, update, and delete product listings with bulk upload support
- FR7: The system supports multiple payment methods (credit card, wallet, COD)
- FR8: Promotional codes, coupons, and tiered discounts are applied at checkout

### Non-Functional Requirements

- NFR1: **Latency** — Search results returned within 200ms at p99; product pages within 100ms
- NFR2: **Throughput** — Handle 50,000 concurrent checkouts during flash sales
- NFR3: **Availability** — 99.99% uptime for the checkout path (≤ 52 minutes downtime/year)
- NFR4: **Consistency** — Inventory counts must be strongly consistent; overselling rate < 0.01%
- NFR5: **Durability** — Zero order data loss; all financial transactions persisted with WAL
- NFR6: **Scalability** — Horizontally scale to 500M products and 100M DAU
- NFR7: **Fault Tolerance** — Graceful degradation: catalog browsing works even if payment is down
- NFR8: **Security** — PCI-DSS compliance for payment data; encryption at rest and in transit

### Capacity Estimation

- **Users**: 100 million DAU, 300 million MAU
- **Product catalog**: 500 million SKUs, average 5 KB per product → 500M × 5 KB = **2.5 TB** catalog data
- **Search queries**: 100M DAU × 10 searches/day = **1 billion searches/day** ≈ **11,574 QPS** (avg), **~50K QPS** (peak)
- **Orders**: 10 million orders/day → **115 orders/sec** (avg), **~4,000 orders/sec** (peak flash sale)
- **Cart operations**: 100M DAU × 5 cart ops/day = 500M/day → **5,787 ops/sec** (avg)
- **Storage (orders)**: 10M orders/day × 2 KB/order × 365 days = **7.3 TB/year**
- **Storage (images)**: 500M products × 5 images × 500 KB = **1.25 PB** (served via CDN)
- **Bandwidth**: Peak 50K search QPS × 10 KB response = **500 MB/s** outbound for search alone
- **Write throughput**: 4,000 orders/sec × 3 DB writes each (order, inventory, payment) = **12,000 writes/sec** peak

### API Design

```http
# Product Catalog
GET /api/v1/products?q=laptop&category=electronics&minPrice=500&maxPrice=2000&cursor=eyJpZCI6MTIzfQ&limit=20
Response: { "items": [...], "nextCursor": "eyJpZCI6MTQzfQ", "totalEstimate": 4521 }

GET /api/v1/products/{productId}
Response: { "id": "prod_abc", "title": "...", "price": 999.99, "availability": "IN_STOCK", ... }

# Shopping Cart
POST /api/v1/cart/items
Body: { "productId": "prod_abc", "quantity": 2 }
Header: Idempotency-Key: "cart-add-uuid-123"

DELETE /api/v1/cart/items/{itemId}

GET /api/v1/cart
Response: { "items": [...], "subtotal": 1999.98, "estimatedTax": 160.00 }

# Checkout & Orders
POST /api/v1/orders
Header: Idempotency-Key: "order-uuid-456"
Body: { "cartId": "cart_xyz", "shippingAddressId": "addr_1", "paymentMethodId": "pm_visa_1", "promoCode": "SAVE10" }
Response: 201 Created { "orderId": "ord_789", "status": "CONFIRMED", "estimatedDelivery": "2025-01-20" }

GET /api/v1/orders/{orderId}
GET /api/v1/orders?status=SHIPPED&cursor=abc&limit=10

PUT /api/v1/orders/{orderId}/cancel
Header: If-Match: "etag-abc123"   # Optimistic concurrency control
Response: 200 OK | 409 Conflict (if already shipped)

# Webhooks (server → merchant/integration)
POST {callbackUrl}
Body: { "event": "order.status_changed", "orderId": "ord_789", "newStatus": "SHIPPED", "timestamp": "..." }
```

### Data Model

```sql
CREATE TABLE products (
    id              BIGINT PRIMARY KEY,            -- Snowflake ID
    seller_id       BIGINT NOT NULL,
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    category_id     INT NOT NULL,
    price_cents     BIGINT NOT NULL,               -- Store in cents to avoid float issues
    currency        CHAR(3) DEFAULT 'USD',
    status          ENUM('ACTIVE','INACTIVE','DELETED') DEFAULT 'ACTIVE',
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW(),
    INDEX idx_category (category_id, status),
    INDEX idx_seller (seller_id)
) PARTITION BY HASH(id) PARTITIONS 64;

CREATE TABLE inventory (
    product_id      BIGINT PRIMARY KEY,
    warehouse_id    INT NOT NULL,
    quantity         INT NOT NULL DEFAULT 0,
    reserved        INT NOT NULL DEFAULT 0,        -- Soft-locked during checkout
    version         INT NOT NULL DEFAULT 0,        -- Optimistic locking
    updated_at      TIMESTAMP DEFAULT NOW(),
    UNIQUE INDEX idx_product_warehouse (product_id, warehouse_id)
);

CREATE TABLE carts (
    id              BIGINT PRIMARY KEY,
    user_id         BIGINT NOT NULL,
    status          ENUM('ACTIVE','CHECKED_OUT','ABANDONED') DEFAULT 'ACTIVE',
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW(),
    INDEX idx_user (user_id, status)
);

CREATE TABLE cart_items (
    id              BIGINT PRIMARY KEY,
    cart_id         BIGINT NOT NULL,
    product_id      BIGINT NOT NULL,
    quantity        INT NOT NULL,
    price_snapshot  BIGINT NOT NULL,               -- Price at time of addition
    FOREIGN KEY (cart_id) REFERENCES carts(id)
);

CREATE TABLE orders (
    id              BIGINT PRIMARY KEY,
    user_id         BIGINT NOT NULL,
    status          ENUM('PENDING','CONFIRMED','PROCESSING','SHIPPED','DELIVERED','CANCELLED','REFUNDED'),
    total_cents     BIGINT NOT NULL,
    currency        CHAR(3) DEFAULT 'USD',
    shipping_addr   JSONB NOT NULL,
    idempotency_key VARCHAR(64) UNIQUE,            -- Prevent duplicate orders
    etag            VARCHAR(64) NOT NULL,           -- Optimistic concurrency
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW(),
    INDEX idx_user_status (user_id, status),
    INDEX idx_created (created_at)
) PARTITION BY RANGE (created_at);                 -- Monthly partitions

CREATE TABLE order_items (
    id              BIGINT PRIMARY KEY,
    order_id        BIGINT NOT NULL,
    product_id      BIGINT NOT NULL,
    quantity        INT NOT NULL,
    unit_price      BIGINT NOT NULL,
    FOREIGN KEY (order_id) REFERENCES orders(id)
);

-- Event sourcing: append-only order events
CREATE TABLE order_events (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    order_id        BIGINT NOT NULL,
    event_type      VARCHAR(50) NOT NULL,          -- CREATED, PAID, SHIPPED, DELIVERED, CANCELLED
    payload         JSONB NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW(),
    INDEX idx_order (order_id, created_at)
);
```

**Search index**: Products are indexed in Elasticsearch with denormalized fields (title, description, category path, price, rating, sales velocity) for full-text and faceted search. A CDC pipeline (Debezium) streams changes from PostgreSQL to Elasticsearch with < 2-second lag.

### High-Level Design

```mermaid
graph TB
    subgraph Clients
        Web[Web App]
        Mobile[Mobile App]
    end

    subgraph Edge
        CDN[CDN - CloudFront]
        LB[Load Balancer]
        GW[API Gateway<br/>Rate Limiting / Auth]
    end

    subgraph Services
        PS[Product Service]
        SS[Search Service]
        CS[Cart Service]
        OS[Order Service]
        IS[Inventory Service]
        PAY[Payment Service]
        NS[Notification Service]
        US[User Service]
    end

    subgraph Data Stores
        PG_P[(PostgreSQL<br/>Products)]
        PG_O[(PostgreSQL<br/>Orders)]
        ES[(Elasticsearch<br/>Search Index)]
        Redis_Cart[(Redis Cluster<br/>Cart / Sessions)]
        Redis_Inv[(Redis<br/>Inventory Cache)]
        S3[(S3<br/>Product Images)]
    end

    subgraph Async
        Kafka[Apache Kafka<br/>Event Bus]
        Workers[Background Workers<br/>Email / Analytics]
    end

    Web --> CDN
    Mobile --> CDN
    CDN --> LB
    LB --> GW
    GW --> PS
    GW --> SS
    GW --> CS
    GW --> OS

    PS --> PG_P
    PS --> S3
    SS --> ES
    CS --> Redis_Cart
    OS --> PG_O
    OS --> IS
    OS --> PAY
    IS --> PG_P
    IS --> Redis_Inv

    OS --> Kafka
    Kafka --> NS
    Kafka --> Workers
    Kafka --> SS
```

### Deep Dive

#### Inventory Management: Optimistic vs. Pessimistic Locking

The inventory problem boils down to: how do you decrement a counter under high concurrency without overselling?

**Pessimistic locking** (`SELECT ... FOR UPDATE`) acquires a row-level lock. It guarantees correctness but serializes all concurrent requests for the same product — a death sentence during flash sales where thousands of users compete for the same item.

**Optimistic locking** uses a version column:

```sql
UPDATE inventory
SET quantity = quantity - 1, version = version + 1
WHERE product_id = ? AND quantity >= 1 AND version = ?;
-- If affected_rows = 0 → conflict, retry or fail
```

This allows concurrent reads and only conflicts on write. For most products with moderate concurrency, optimistic locking achieves high throughput with minimal contention.

**Hybrid approach for flash sales**: Pre-segment inventory into N buckets (e.g., 100 units across 10 Redis keys of 10 each). Each checkout request is routed to a random bucket via consistent hashing. This reduces contention by 10× while maintaining a global invariant through eventual reconciliation.

#### Checkout Saga Pattern

Checkout spans multiple services (inventory, payment, order) and cannot use a traditional distributed transaction. We use the **saga pattern** with an orchestrator:

```mermaid
sequenceDiagram
    participant Client
    participant OrderSvc as Order Service (Orchestrator)
    participant InvSvc as Inventory Service
    participant PaySvc as Payment Service
    participant NotifSvc as Notification Service

    Client->>OrderSvc: POST /orders (idempotency_key)
    OrderSvc->>OrderSvc: Check idempotency_key (dedup)
    OrderSvc->>InvSvc: Reserve inventory
    InvSvc-->>OrderSvc: Reserved ✓
    OrderSvc->>PaySvc: Charge payment
    PaySvc-->>OrderSvc: Payment confirmed ✓
    OrderSvc->>OrderSvc: Mark order CONFIRMED
    OrderSvc->>NotifSvc: Send confirmation email
    OrderSvc-->>Client: 201 Created

    Note over OrderSvc: If payment fails:
    OrderSvc->>InvSvc: Release inventory (compensating action)
    OrderSvc->>OrderSvc: Mark order FAILED
```

Each step publishes an event to Kafka. If a step fails, compensating transactions undo previous steps. The outbox pattern ensures events are published atomically with local DB writes.

#### Transactional Outbox Pattern

```sql
BEGIN;
  INSERT INTO orders (...) VALUES (...);
  INSERT INTO outbox_events (aggregate_id, event_type, payload)
    VALUES ('ord_789', 'ORDER_CREATED', '{"items": [...]}');
COMMIT;
-- A CDC connector (Debezium) tails the outbox table and publishes to Kafka
```

This eliminates the dual-write problem (writing to DB + Kafka independently).

### Bottlenecks & Mitigations

| Bottleneck | Mitigation |
|---|---|
| Inventory hot-key contention during flash sales | Shard inventory into Redis buckets; use Lua scripts for atomic decrement |
| Search index lag causing stale results | CDC with Debezium for < 2s lag; show "limited stock" instead of exact counts |
| Cart data loss during Redis failure | Redis Cluster with AOF persistence; fall back to DB-backed cart |
| Payment service timeout holding inventory | Reservation TTL (10 min); background reaper releases expired reservations |
| Catalog DB overwhelmed by read traffic | Read replicas + aggressive CDN caching for product pages (TTL 60s) |
| Order spikes during promotions | Queue-based load leveling: orders enter Kafka, processed at sustainable rate |

### Key Takeaways

- Use the **saga pattern** with compensating transactions for distributed checkout — never 2PC across microservices
- Inventory accuracy requires **optimistic locking** (or Redis atomic ops) — not DB-level pessimistic locks at scale
- The **transactional outbox** pattern eliminates dual-write bugs between DB and message broker
- Store prices in **integer cents** to avoid floating-point arithmetic errors in financial calculations
- **Idempotency keys** on every mutation endpoint prevent duplicate orders from retries
- Search is eventually consistent by design — use CDC to keep Elasticsearch in near-real-time sync

---

## 2. Payment System

### Problem Statement

A payment system is the financial backbone of the internet. It handles the movement of money between buyers, merchants, and financial institutions — processing charges, authorizations, refunds, and settlements across multiple currencies and payment methods. Systems like Stripe and PayPal process billions of dollars daily and must guarantee that money is never lost, duplicated, or misattributed.

The fundamental engineering challenge is **exactly-once processing** in a distributed system. Network partitions, timeout-induced retries, and partial failures can all cause a single payment intent to be executed multiple times. Unlike most distributed systems where "at-least-once" is acceptable, payments require true idempotency: charging a customer twice is unacceptable.

Additionally, payment systems must comply with PCI-DSS, maintain a double-entry ledger that balances to the penny, reconcile with external banking networks that operate on batch cycles (ACH settles in 1-3 business days), and detect fraudulent transactions in real time — all while maintaining sub-second latency for the merchant's checkout experience.

### Use Cases

- Merchant integrates payment SDK and processes credit card charges
- Customer pays via credit card, debit card, bank transfer, or digital wallet
- System authorizes, captures, and settles payment in multi-step flow
- Merchant issues full or partial refunds
- Recurring subscription billing with automatic retry on failure
- Multi-currency transactions with real-time exchange rate conversion
- Internal double-entry ledger tracks every money movement
- Daily reconciliation with payment processor and bank statements

### Functional Requirements

- FR1: Process payments across multiple methods (card, ACH, wallet) with authorization and capture
- FR2: Support idempotent charge creation — retries must not result in duplicate charges
- FR3: Handle full and partial refunds linked to the original transaction
- FR4: Maintain a double-entry accounting ledger for all money movements
- FR5: Support webhooks to notify merchants of payment status changes
- FR6: Process recurring/subscription payments on a schedule
- FR7: Provide a merchant dashboard with transaction history, analytics, and payouts
- FR8: Perform real-time fraud risk scoring before authorizing transactions

### Non-Functional Requirements

- NFR1: **Latency** — Payment authorization within 500ms p99 (excluding external processor)
- NFR2: **Availability** — 99.999% for the payment processing path (≤ 5 min downtime/year)
- NFR3: **Consistency** — Strongly consistent ledger; balance must always be correct
- NFR4: **Durability** — Zero data loss; synchronous replication with WAL
- NFR5: **Idempotency** — Every payment operation must be idempotent end-to-end
- NFR6: **Security** — PCI-DSS Level 1 compliance; tokenization of card data; encryption at rest
- NFR7: **Auditability** — Complete audit trail; immutable event log for every transaction
- NFR8: **Throughput** — 10,000 transactions per second at peak

### Capacity Estimation

- **Merchants**: 1 million active merchants
- **Transactions**: 500 million/day → **5,787 TPS** (avg), **~15,000 TPS** (peak, Black Friday)
- **Transaction size**: Avg $50, so daily volume = $25 billion
- **Storage per transaction**: ~1 KB (transaction) + ~500 bytes (ledger entries) + ~200 bytes (events) = **1.7 KB**
- **Storage (transactions)**: 500M/day × 1.7 KB × 365 = **310 TB/year** (before compression)
- **Ledger entries**: 2 entries per transaction (debit + credit) → 1 billion ledger rows/day
- **Webhook delivery**: 500M transactions × 1.5 events avg = 750M webhooks/day → **8,680/sec**
- **Bandwidth**: 15K TPS × 2 KB avg request/response = **30 MB/s**

### API Design

```http
# Create a Payment Intent (idempotent)
POST /api/v1/payment_intents
Header: Idempotency-Key: "pi_uuid_abc123"
Header: Authorization: Bearer sk_live_merchant_key
Body: {
  "amount": 4999,
  "currency": "usd",
  "payment_method": "pm_card_visa_4242",
  "capture_method": "automatic",         # or "manual" for auth-then-capture
  "metadata": { "order_id": "ord_789" }
}
Response: 201 Created
{
  "id": "pi_1234",
  "status": "succeeded",
  "amount": 4999,
  "currency": "usd",
  "created_at": "2025-01-15T10:30:00Z"
}

# Capture an authorized payment
POST /api/v1/payment_intents/{piId}/capture
Header: Idempotency-Key: "capture_uuid_def456"
Body: { "amount_to_capture": 4999 }

# Refund
POST /api/v1/refunds
Header: Idempotency-Key: "refund_uuid_ghi789"
Body: { "payment_intent": "pi_1234", "amount": 2000, "reason": "customer_request" }
Response: 201 Created { "id": "rf_5678", "status": "pending", ... }

# List transactions (cursor-based pagination)
GET /api/v1/transactions?merchant_id=m_abc&status=succeeded&created_gte=2025-01-01&cursor=tx_999&limit=50

# Webhook registration
POST /api/v1/webhooks
Body: { "url": "https://merchant.com/webhook", "events": ["payment_intent.succeeded", "refund.created"] }

# Webhook payload (server → merchant)
POST https://merchant.com/webhook
Header: Stripe-Signature: t=1234,v1=sig_hash
Body: {
  "id": "evt_abc",
  "type": "payment_intent.succeeded",
  "data": { "object": { "id": "pi_1234", "amount": 4999, ... } }
}
```

### Data Model

```sql
-- Core payment intent
CREATE TABLE payment_intents (
    id              VARCHAR(64) PRIMARY KEY,        -- "pi_" prefix + ULID
    merchant_id     VARCHAR(64) NOT NULL,
    amount          BIGINT NOT NULL,                -- In smallest currency unit (cents)
    currency        CHAR(3) NOT NULL,
    status          ENUM('requires_payment','processing','succeeded','failed','cancelled'),
    payment_method  VARCHAR(64),
    capture_method  ENUM('automatic','manual') DEFAULT 'automatic',
    idempotency_key VARCHAR(128) UNIQUE,
    metadata        JSONB,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    INDEX idx_merchant_created (merchant_id, created_at),
    INDEX idx_status (status)
) PARTITION BY RANGE (created_at);

-- Double-entry ledger (immutable, append-only)
CREATE TABLE ledger_entries (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    transaction_id  VARCHAR(64) NOT NULL,
    account_id      VARCHAR(64) NOT NULL,           -- e.g., "merchant:m_abc", "platform:fees"
    entry_type      ENUM('DEBIT','CREDIT') NOT NULL,
    amount          BIGINT NOT NULL,                -- Always positive
    currency        CHAR(3) NOT NULL,
    balance_after   BIGINT NOT NULL,                -- Running balance
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    INDEX idx_account (account_id, created_at),
    INDEX idx_transaction (transaction_id)
);
-- Invariant: SUM(CREDIT) = SUM(DEBIT) globally at all times

-- Idempotency store
CREATE TABLE idempotency_keys (
    key             VARCHAR(128) PRIMARY KEY,
    merchant_id     VARCHAR(64) NOT NULL,
    request_hash    VARCHAR(64) NOT NULL,           -- SHA-256 of request body
    response_code   INT,
    response_body   JSONB,
    created_at      TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMP NOT NULL              -- TTL: 24-72 hours
);

-- Refunds
CREATE TABLE refunds (
    id              VARCHAR(64) PRIMARY KEY,
    payment_intent  VARCHAR(64) NOT NULL,
    amount          BIGINT NOT NULL,
    reason          VARCHAR(100),
    status          ENUM('pending','succeeded','failed'),
    created_at      TIMESTAMP DEFAULT NOW(),
    FOREIGN KEY (payment_intent) REFERENCES payment_intents(id)
);

-- Event log (immutable audit trail)
CREATE TABLE payment_events (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    entity_id       VARCHAR(64) NOT NULL,
    event_type      VARCHAR(50) NOT NULL,
    payload         JSONB NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW(),
    INDEX idx_entity (entity_id, created_at)
);
```

### High-Level Design

```mermaid
graph TB
    subgraph Client Layer
        SDK[Merchant SDK / API Client]
    end

    subgraph API Layer
        GW[API Gateway<br/>Auth / Rate Limit / Idempotency]
    end

    subgraph Core Services
        PIS[Payment Intent Service]
        FS[Fraud Service]
        LS[Ledger Service]
        RS[Refund Service]
        WH[Webhook Service]
        PS[Payout Service]
    end

    subgraph External
        PP[Payment Processor<br/>Visa/Mastercard Network]
        BANK[Bank / ACH]
    end

    subgraph Data
        PG[(PostgreSQL<br/>Payments + Ledger)]
        Redis[(Redis<br/>Idempotency Cache)]
        Kafka[Kafka<br/>Event Stream]
        S3[(S3<br/>Reconciliation Files)]
    end

    SDK --> GW
    GW --> PIS
    PIS --> FS
    PIS --> PP
    PIS --> LS
    PIS --> Redis
    LS --> PG
    PIS --> PG
    PIS --> Kafka
    RS --> PIS
    Kafka --> WH
    Kafka --> PS
    PS --> BANK
    PS --> S3
```

### Deep Dive

#### Exactly-Once Payment Processing

The idempotency layer is the single most critical component. Every mutating API call requires an `Idempotency-Key` header.

**Flow**:
1. Request arrives with `Idempotency-Key: "abc"`.
2. Check Redis for the key → if found, return cached response immediately.
3. If not found, acquire a distributed lock (Redlock) on the key.
4. Execute the payment logic (authorize, capture).
5. Within a single DB transaction: persist payment result + write idempotency record.
6. Set the Redis cache with TTL (24h).
7. Release the lock.

If the process crashes between steps 4 and 5, the incomplete transaction is detected by a reaper that checks for orphaned locks and rolls back. The key insight: the **idempotency key is stored in the same database transaction** as the payment result — they succeed or fail atomically.

#### Double-Entry Ledger

Every money movement creates exactly two ledger entries: a debit and a credit. For a $50 charge:

| Account | Debit | Credit |
|---|---|---|
| customer:cust_123 | $50.00 | |
| merchant:merch_456 | | $48.50 |
| platform:fees | | $1.50 |

The invariant `SUM(debits) = SUM(credits)` is enforced at the application layer and verified by a nightly reconciliation job. Ledger entries are **append-only** — corrections are made by adding new compensating entries, never by mutating existing rows.

#### Reconciliation Pipeline

```mermaid
sequenceDiagram
    participant Cron as Daily Cron Job
    participant Ledger as Internal Ledger
    participant Processor as Payment Processor
    participant Recon as Reconciliation Service
    participant Alert as Alert System

    Cron->>Ledger: Export day's transactions
    Cron->>Processor: Download settlement file
    Cron->>Recon: Three-way match
    Recon->>Recon: Match by transaction ID
    alt All matched
        Recon->>Ledger: Mark reconciled
    else Discrepancy found
        Recon->>Alert: Raise alert with details
    end
```

### Bottlenecks & Mitigations

| Bottleneck | Mitigation |
|---|---|
| Idempotency key lookup latency | Redis with local L1 cache; 99% hit rate avoids DB round-trip |
| Ledger write throughput at 10K TPS | Batch ledger inserts (micro-batching every 10ms); partitioned by month |
| External payment processor timeout | Circuit breaker (Hystrix pattern); fallback to secondary processor |
| Webhook delivery failures | Exponential backoff retry (1s, 2s, 4s, ... up to 24h); dead letter queue |
| Reconciliation file size (millions of rows) | Spark-based parallel reconciliation; partition by merchant |
| Hot merchant accounts (high volume) | Shard ledger by merchant_id; in-memory balance cache with write-behind |

### Key Takeaways

- **Idempotency is non-negotiable**: Store idempotency keys atomically with payment results in the same transaction
- **Double-entry bookkeeping** catches bugs that unit tests miss — if debits ≠ credits, something is wrong
- **Append-only ledger** entries provide a complete audit trail; never mutate financial records
- **Reconciliation** is a first-class system, not an afterthought — run it daily against processor files
- Use **circuit breakers** for external payment processor calls to prevent cascading failures
- Store all monetary values as **integers in the smallest currency unit** (cents, paise)

---

## 3. Hotel/Flight Booking System

### Problem Statement

A travel booking platform aggregates availability from thousands of hotels, airlines, and other travel providers, enabling users to search, compare, and book travel in real time. Systems like Booking.com manage over 28 million listings and handle more than 1.5 million room-night reservations per day.

The core technical challenge is the **inventory perishability problem**. Unlike e-commerce products that can be restocked, a hotel room on January 15th that goes unbooked is permanently lost revenue. This creates intense pressure to maximize occupancy while maintaining accurate availability — showing a room as available when it has already been booked (and vice versa) directly impacts revenue and customer trust.

Search is also uniquely complex: a query for "hotels in Paris, Jan 15-18, 2 adults" must evaluate availability across thousands of properties, apply dynamic pricing that changes by date, room type, and demand, filter by amenities and policies, and return sorted results — all within 200ms. The system must also handle the "look-to-book" ratio problem: for every 100 searches, only 1-2 result in a booking, meaning the read load dwarfs writes by 50-100×.

### Use Cases

- Search hotels/flights by destination, dates, guests, and filters (stars, amenities, airlines)
- View property/flight details with room types, photos, policies, and dynamic pricing
- Select a room or seat and proceed to reservation
- Complete booking with guest details and payment
- Receive booking confirmation via email and in-app notification
- Modify or cancel a reservation per the cancellation policy
- Hotels/airlines update availability and pricing through an extranet/API
- Aggregate reviews and ratings from past guests

### Functional Requirements

- FR1: Users can search for hotels by location, check-in/check-out dates, guest count, and filters
- FR2: Search returns available properties with real-time pricing and room options
- FR3: Users can select a room and hold it temporarily while completing booking (5-10 min TTL)
- FR4: Booking atomically confirms reservation, charges payment, and sends confirmation
- FR5: Users can modify dates or cancel bookings per the property's policy
- FR6: Hotels manage inventory, pricing, and blackout dates through a partner portal
- FR7: Dynamic pricing adjusts based on demand, seasonality, and competitor rates
- FR8: Support multi-leg flight bookings with layovers and fare combinations

### Non-Functional Requirements

- NFR1: **Latency** — Search results within 300ms at p95; booking confirmation within 2 seconds
- NFR2: **Availability** — 99.99% for search; 99.999% for booking confirmation path
- NFR3: **Consistency** — Room availability is eventually consistent for search but strongly consistent at booking time
- NFR4: **Throughput** — 100,000 search QPS at peak (holiday season)
- NFR5: **Scalability** — Support 28 million property listings across 200+ countries
- NFR6: **Durability** — Zero booking data loss; confirmed reservations are sacrosanct
- NFR7: **Freshness** — Inventory changes reflected in search within 30 seconds
- NFR8: **Multi-region** — Active-active deployment for global low-latency access

### Capacity Estimation

- **Users**: 50 million DAU during peak travel season
- **Listings**: 28 million properties × 50 room-types avg = **1.4 billion room-date combinations** (for a 365-day window)
- **Search queries**: 50M DAU × 15 searches/day = 750M/day → **8,680 QPS** (avg), **~100K QPS** (peak)
- **Bookings**: 1.5 million/day → **17 bookings/sec** (avg), **~100 bookings/sec** (peak)
- **Look-to-book ratio**: ~500:1 (search to booking)
- **Storage (listings)**: 28M properties × 10 KB = **280 GB** base; with room-date availability: 1.4B × 50 bytes = **70 GB**
- **Storage (bookings)**: 1.5M/day × 2 KB × 365 = **1.1 TB/year**
- **Bandwidth**: 100K QPS × 5 KB response = **500 MB/s** peak search egress

### API Design

```http
# Search availability
GET /api/v1/search/hotels?location=paris&lat=48.8566&lng=2.3522&checkin=2025-03-15&checkout=2025-03-18&guests=2&rooms=1&stars=4,5&amenities=wifi,pool&sort=price_asc&cursor=abc&limit=20
Response: {
  "results": [
    { "propertyId": "htl_123", "name": "...", "rating": 4.5, "lowestPrice": 189.00, "currency": "EUR", "thumbnailUrl": "..." }
  ],
  "nextCursor": "...",
  "totalResults": 342,
  "searchId": "srch_xyz"   // For analytics tracking
}

# Get property details and room options
GET /api/v1/properties/{propertyId}/rooms?checkin=2025-03-15&checkout=2025-03-18&guests=2
Response: {
  "property": { ... },
  "rooms": [
    { "roomTypeId": "rt_1", "name": "Deluxe King", "pricePerNight": 189.00, "totalPrice": 567.00,
      "cancellationPolicy": "FREE_CANCEL_BEFORE_72H", "available": 3, "maxGuests": 2 }
  ]
}

# Create a temporary hold (pre-booking)
POST /api/v1/holds
Header: Idempotency-Key: "hold_uuid_123"
Body: { "propertyId": "htl_123", "roomTypeId": "rt_1", "checkin": "2025-03-15", "checkout": "2025-03-18" }
Response: 201 Created { "holdId": "hld_456", "expiresAt": "2025-01-15T10:40:00Z" }

# Confirm booking
POST /api/v1/bookings
Header: Idempotency-Key: "book_uuid_456"
Body: { "holdId": "hld_456", "guestDetails": { ... }, "paymentMethodId": "pm_visa_1" }
Response: 201 Created { "bookingId": "bk_789", "confirmationCode": "ABC123XYZ", "status": "CONFIRMED" }

# Cancel booking
POST /api/v1/bookings/{bookingId}/cancel
Response: 200 OK { "refundAmount": 567.00, "refundStatus": "processing" }
```

### Data Model

```sql
CREATE TABLE properties (
    id              BIGINT PRIMARY KEY,
    name            VARCHAR(500) NOT NULL,
    location        GEOGRAPHY(POINT, 4326),        -- PostGIS
    city            VARCHAR(100),
    country         CHAR(2),
    star_rating     SMALLINT,
    amenities       JSONB,                          -- ["wifi", "pool", "parking"]
    description     TEXT,
    images          JSONB,
    avg_rating      DECIMAL(3,2),
    review_count    INT DEFAULT 0,
    status          ENUM('ACTIVE','INACTIVE'),
    INDEX idx_location USING GIST (location),
    INDEX idx_city (city, star_rating)
);

CREATE TABLE room_types (
    id              BIGINT PRIMARY KEY,
    property_id     BIGINT NOT NULL,
    name            VARCHAR(200),
    max_guests      SMALLINT,
    base_price      BIGINT NOT NULL,               -- Cents
    amenities       JSONB,
    FOREIGN KEY (property_id) REFERENCES properties(id)
);

-- Availability & pricing per room-type per date
CREATE TABLE room_inventory (
    room_type_id    BIGINT NOT NULL,
    date            DATE NOT NULL,
    total_rooms     SMALLINT NOT NULL,
    booked_rooms    SMALLINT NOT NULL DEFAULT 0,
    held_rooms      SMALLINT NOT NULL DEFAULT 0,    -- Temporary holds
    price_cents     BIGINT NOT NULL,                -- Dynamic price for this date
    version         INT DEFAULT 0,                  -- Optimistic locking
    PRIMARY KEY (room_type_id, date),
    INDEX idx_availability (room_type_id, date, total_rooms, booked_rooms)
) PARTITION BY RANGE (date);

CREATE TABLE holds (
    id              BIGINT PRIMARY KEY,
    room_type_id    BIGINT NOT NULL,
    checkin         DATE NOT NULL,
    checkout        DATE NOT NULL,
    user_id         BIGINT,
    status          ENUM('ACTIVE','CONVERTED','EXPIRED') DEFAULT 'ACTIVE',
    expires_at      TIMESTAMP NOT NULL,
    created_at      TIMESTAMP DEFAULT NOW()
);

CREATE TABLE bookings (
    id              BIGINT PRIMARY KEY,
    confirmation_code VARCHAR(20) UNIQUE NOT NULL,
    user_id         BIGINT NOT NULL,
    property_id     BIGINT NOT NULL,
    room_type_id    BIGINT NOT NULL,
    checkin         DATE NOT NULL,
    checkout        DATE NOT NULL,
    total_price     BIGINT NOT NULL,
    currency        CHAR(3),
    status          ENUM('CONFIRMED','MODIFIED','CANCELLED','COMPLETED','NO_SHOW'),
    guest_details   JSONB,
    payment_id      VARCHAR(64),
    idempotency_key VARCHAR(128) UNIQUE,
    created_at      TIMESTAMP DEFAULT NOW(),
    INDEX idx_user (user_id, status),
    INDEX idx_property_dates (property_id, checkin, checkout)
);
```

### High-Level Design

```mermaid
graph TB
    subgraph Clients
        Web[Web / Mobile]
    end

    subgraph Edge
        CDN[CDN]
        GW[API Gateway]
    end

    subgraph Services
        SRS[Search Service]
        PRS[Property Service]
        AVS[Availability Service]
        BKS[Booking Service]
        PRC[Pricing Engine]
        PAY[Payment Service]
        NTF[Notification Service]
    end

    subgraph Data
        ES[(Elasticsearch<br/>Property Search)]
        PG_Prop[(PostgreSQL<br/>Properties)]
        PG_Book[(PostgreSQL<br/>Bookings)]
        Redis_Avail[(Redis<br/>Availability Cache)]
        Redis_Hold[(Redis<br/>Hold TTLs)]
    end

    subgraph Async
        Kafka[Kafka]
        Workers[Workers]
    end

    Web --> CDN --> GW
    GW --> SRS
    GW --> PRS
    GW --> BKS

    SRS --> ES
    SRS --> Redis_Avail
    PRS --> PG_Prop
    AVS --> PG_Prop
    AVS --> Redis_Avail
    BKS --> AVS
    BKS --> PRC
    BKS --> PAY
    BKS --> PG_Book
    BKS --> Redis_Hold

    BKS --> Kafka
    Kafka --> NTF
    Kafka --> Workers
    Kafka --> AVS
```

### Deep Dive

#### Availability Search Optimization

The naive approach — querying `room_inventory` for every property in a city across all dates — is far too slow at 100K QPS.

**Layered caching strategy**:
1. **L1 — CDN/Edge cache**: Popular city + date combinations cached for 60 seconds. Serves ~40% of traffic.
2. **L2 — Redis availability bitmap**: Each room-type/date has an availability count in Redis. Search checks Redis first. Updated via Kafka events whenever bookings/cancellations occur. ~50% of remaining traffic.
3. **L3 — Database**: Only queried for cache misses or booking-time strong consistency checks.

**Search flow**: Elasticsearch finds properties matching location/amenities/stars → for each candidate property, check Redis availability bitmap → filter to only available properties → apply pricing engine → sort and return.

#### Temporary Hold Mechanism

When a user selects a room, a **hold** is placed to prevent double-booking during the 5-10 minute checkout window:

```sql
BEGIN;
  UPDATE room_inventory
  SET held_rooms = held_rooms + 1, version = version + 1
  WHERE room_type_id = ? AND date BETWEEN ? AND ?
    AND (booked_rooms + held_rooms) < total_rooms
    AND version = ?;
  -- If affected_rows = num_dates → success, insert hold record
  INSERT INTO holds (...) VALUES (...);
COMMIT;
```

A background job reaps expired holds every 30 seconds and decrements `held_rooms`. Redis is used to track hold TTLs with `EXPIREAT`, triggering the release on expiry.

#### Dynamic Pricing

Price is computed by the pricing engine using: `base_price × demand_multiplier × seasonality_factor × competitor_adjustment`. Demand is measured by the ratio of searches to availability. The pricing engine pre-computes prices for the next 365 days per room type and stores them in the `room_inventory` table. Real-time adjustments are published via Kafka and applied within seconds.

### Bottlenecks & Mitigations

| Bottleneck | Mitigation |
|---|---|
| Search latency for broad queries ("hotels in Europe") | Geo-sharded Elasticsearch; pre-aggregated city-level results |
| Availability cache staleness | Kafka-driven cache invalidation with < 5s propagation |
| Hold expiration thundering herd | Jittered expiry times (±30s); batch release in cron job |
| Double booking race condition | Optimistic locking on `room_inventory.version`; serialized booking at DB level |
| Partner inventory sync delays | Async polling + webhook from partner extranet; reconciliation job |
| Hot properties (viral destinations) | Read replicas for search; write-through cache for availability |

### Key Takeaways

- **Eventual consistency for search, strong consistency for booking** — this split is fundamental to travel systems
- Temporary **holds with TTL** prevent double-booking during checkout without permanently locking inventory
- **Layered caching** (CDN → Redis → DB) is essential to handle the extreme read amplification (500:1 look-to-book)
- **Dynamic pricing** is pre-computed and cached, not calculated per-request at search time
- Partition `room_inventory` by date to enable efficient range queries and automatic cleanup of past dates
- The booking confirmation path must be treated as the most critical flow — design for 99.999% availability

---

## 4. Ticket Booking System

### Problem Statement

A ticket booking system like BookMyShow or Ticketmaster enables users to browse events (concerts, movies, sports), view venue seat maps, select specific seats, and complete a purchase — often under extreme time pressure. When a popular artist announces a tour, millions of users may simultaneously attempt to book from a limited pool of seats, creating one of the hardest concurrency problems in system design.

The defining characteristic is the **extreme write contention on a tiny dataset**. A venue might have 20,000 seats, and 2 million users might try to book within 60 seconds of sale opening. Unlike e-commerce where inventory is across millions of SKUs, ticket sales concentrate all demand on a single event's seat map. Traditional database locking approaches crumble under this load.

Additionally, every seat is unique — unlike hotel rooms of the same type, seat A12 is not interchangeable with seat B7. Users expect to see a real-time seat map showing exactly which seats are available, held, or sold. The system must handle this visual real-time state while maintaining transactional correctness under millions of concurrent requests.

### Use Cases

- Browse upcoming events by category, location, date, and artist/team
- View event details including venue map, pricing tiers, and available seat count
- Select specific seats from an interactive seat map
- Hold selected seats temporarily while completing payment (5-minute TTL)
- Complete ticket purchase with immediate confirmation
- View and manage purchased tickets (e-ticket, QR code)
- Transfer or resell tickets to other users (secondary market)
- Receive real-time notifications for waitlisted events when tickets become available

### Functional Requirements

- FR1: Users can search and browse events with filters (genre, city, date range, price)
- FR2: Interactive seat map shows real-time seat availability with color-coded status
- FR3: Users can select 1-6 seats that are adjacent (best-available algorithm)
- FR4: Selected seats are held for 5 minutes; auto-released if payment is not completed
- FR5: Ticket purchase is atomic — either all seats are confirmed or none are
- FR6: Generate unique QR-code e-tickets upon purchase confirmation
- FR7: Support waitlist with automatic notification when seats become available
- FR8: Enforce purchase limits (e.g., max 4 tickets per user per event)

### Non-Functional Requirements

- NFR1: **Latency** — Seat map loads within 200ms; seat selection confirmed within 100ms
- NFR2: **Throughput** — Handle 500K+ concurrent users during flash sales
- NFR3: **Availability** — 99.99% uptime; no downtime during on-sale events
- NFR4: **Consistency** — No double-selling of seats; every seat sold exactly once
- NFR5: **Concurrency** — Correctly handle 50,000+ simultaneous seat selection attempts
- NFR6: **Scalability** — Support 100K events with up to 100K seats each
- NFR7: **Fairness** — Virtual queue system to prevent bot-driven scalping
- NFR8: **Real-time** — Seat availability updates pushed to clients within 1 second

### Capacity Estimation

- **Users**: 20 million DAU normal; **5 million concurrent** during major on-sale events
- **Events**: 100,000 active events; avg 10,000 seats each
- **Normal operations**: 500 bookings/sec average
- **Flash sale peak**: 500K users competing for 20,000 seats → **50,000 seat-lock attempts/sec** in a 10-second burst
- **Seat map queries**: 5M concurrent × 1 poll/3s = **1.67M QPS** for seat status during flash sale
- **Storage (events)**: 100K events × 50 KB metadata = **5 GB**
- **Storage (seat maps)**: 100K events × 10K seats × 100 bytes = **100 GB**
- **Storage (tickets)**: 500M tickets/year × 500 bytes = **250 GB/year**
- **WebSocket connections**: 5 million concurrent during peak events

### API Design

```http
# Search events
GET /api/v1/events?city=mumbai&category=concert&dateFrom=2025-03-01&dateTo=2025-03-31&cursor=abc&limit=20

# Get event details with seat map
GET /api/v1/events/{eventId}
GET /api/v1/events/{eventId}/seats?section=A
Response: {
  "sections": [
    { "id": "A", "seats": [
      { "id": "A1", "status": "available", "price": 150.00 },
      { "id": "A2", "status": "held", "price": 150.00 },
      { "id": "A3", "status": "sold", "price": 150.00 }
    ]}
  ]
}

# Enter virtual queue (flash sales)
POST /api/v1/events/{eventId}/queue
Response: 202 Accepted { "queueToken": "qt_abc123", "position": 4521, "estimatedWait": "2m 30s" }

# Hold seats (only callable after queue token is validated)
POST /api/v1/events/{eventId}/holds
Header: Idempotency-Key: "hold_uuid_789"
Header: X-Queue-Token: "qt_abc123"
Body: { "seatIds": ["A1", "A3", "A4", "A5"], "holdDurationSec": 300 }
Response: 201 Created { "holdId": "hld_abc", "expiresAt": "2025-01-15T10:35:00Z", "totalPrice": 600.00 }
| 409 Conflict { "error": "seats_unavailable", "unavailable": ["A3"] }

# Purchase tickets
POST /api/v1/orders
Header: Idempotency-Key: "order_uuid_012"
Body: { "holdId": "hld_abc", "paymentMethodId": "pm_visa_1" }
Response: 201 Created { "orderId": "ord_xyz", "tickets": [ { "id": "tkt_1", "seat": "A1", "qrCode": "..." } ] }

# WebSocket for real-time seat updates
WS /api/v1/events/{eventId}/seat-updates
Message: { "type": "seat_status_change", "seatId": "A1", "newStatus": "held" }
```

### Data Model

```sql
CREATE TABLE events (
    id              BIGINT PRIMARY KEY,
    name            VARCHAR(500) NOT NULL,
    venue_id        BIGINT NOT NULL,
    category        VARCHAR(50),
    event_date      TIMESTAMP NOT NULL,
    sale_start      TIMESTAMP NOT NULL,
    status          ENUM('UPCOMING','ON_SALE','SOLD_OUT','COMPLETED','CANCELLED'),
    max_tickets_per_user SMALLINT DEFAULT 4,
    created_at      TIMESTAMP DEFAULT NOW(),
    INDEX idx_city_date (venue_id, event_date),
    INDEX idx_category (category, event_date)
);

CREATE TABLE venues (
    id              BIGINT PRIMARY KEY,
    name            VARCHAR(300),
    city            VARCHAR(100),
    capacity        INT,
    seat_map        JSONB                          -- Section/row/seat layout
);

CREATE TABLE seats (
    id              BIGINT PRIMARY KEY,
    event_id        BIGINT NOT NULL,
    section         VARCHAR(10),
    row             VARCHAR(5),
    number          VARCHAR(5),
    tier            VARCHAR(20),                   -- VIP, Premium, Standard
    price_cents     BIGINT NOT NULL,
    status          ENUM('AVAILABLE','HELD','SOLD') DEFAULT 'AVAILABLE',
    held_by         BIGINT,                        -- user_id holding
    held_until      TIMESTAMP,
    version         INT DEFAULT 0,                 -- Optimistic locking
    INDEX idx_event_status (event_id, status),
    INDEX idx_event_section (event_id, section, status)
) PARTITION BY HASH(event_id) PARTITIONS 32;

CREATE TABLE ticket_orders (
    id              BIGINT PRIMARY KEY,
    user_id         BIGINT NOT NULL,
    event_id        BIGINT NOT NULL,
    total_cents     BIGINT NOT NULL,
    status          ENUM('PENDING','CONFIRMED','CANCELLED','REFUNDED'),
    payment_id      VARCHAR(64),
    idempotency_key VARCHAR(128) UNIQUE,
    created_at      TIMESTAMP DEFAULT NOW(),
    INDEX idx_user (user_id, created_at)
);

CREATE TABLE tickets (
    id              BIGINT PRIMARY KEY,
    order_id        BIGINT NOT NULL,
    seat_id         BIGINT NOT NULL UNIQUE,        -- Each seat sold once
    event_id        BIGINT NOT NULL,
    user_id         BIGINT NOT NULL,
    qr_code         VARCHAR(256) UNIQUE,
    status          ENUM('ACTIVE','TRANSFERRED','CANCELLED'),
    FOREIGN KEY (order_id) REFERENCES ticket_orders(id)
);

-- Virtual queue for flash sales
CREATE TABLE queue_tokens (
    token           VARCHAR(128) PRIMARY KEY,
    event_id        BIGINT NOT NULL,
    user_id         BIGINT NOT NULL,
    position        INT NOT NULL,
    status          ENUM('WAITING','ADMITTED','EXPIRED'),
    created_at      TIMESTAMP DEFAULT NOW(),
    INDEX idx_event_position (event_id, position)
);
```

### High-Level Design

```mermaid
graph TB
    subgraph Clients
        Web[Web App]
        Mobile[Mobile App]
    end

    subgraph Edge
        CDN[CDN]
        LB[Load Balancer]
        GW[API Gateway]
    end

    subgraph Services
        ES[Event Service]
        QS[Queue Service]
        SMS[Seat Management Service]
        TS[Ticketing Service]
        PAY[Payment Service]
        NTF[Notification Service]
        WSS[WebSocket Server<br/>Seat Updates]
    end

    subgraph Data
        PG[(PostgreSQL<br/>Events / Seats)]
        Redis_Seats[(Redis Cluster<br/>Seat Locks)]
        Redis_Queue[(Redis Sorted Set<br/>Virtual Queue)]
        Kafka[Kafka]
    end

    Web --> CDN --> LB --> GW
    Mobile --> LB
    GW --> ES
    GW --> QS
    GW --> SMS
    GW --> TS
    
    QS --> Redis_Queue
    SMS --> Redis_Seats
    SMS --> PG
    TS --> PAY
    TS --> PG
    TS --> Kafka
    Kafka --> NTF
    Kafka --> WSS
    WSS --> Web
    WSS --> Mobile
```

### Deep Dive

#### Flash Sale Architecture: Virtual Queue + Redis Seat Locking

The flash sale problem requires a fundamentally different architecture than normal operations. Direct database access at 50K TPS for the same event's seats would overwhelm any RDBMS.

**Virtual Queue**: When a flash sale begins, users enter a virtual queue backed by a Redis sorted set (`ZADD queue:{eventId} {timestamp} {userId}`). A rate-controlled admission process (`ZPOPMIN`) dequeues users at a sustainable rate (e.g., 1,000/sec), issuing queue tokens that grant access to the seat selection page.

**Redis-based Seat Locking**: Once admitted, seat selection operates entirely in Redis:

```
# Atomic seat hold using Lua script
EVAL "
  local status = redis.call('HGET', KEYS[1], 'status')
  if status == 'available' then
    redis.call('HMSET', KEYS[1], 'status', 'held', 'held_by', ARGV[1], 'held_until', ARGV[2])
    redis.call('EXPIREAT', KEYS[1], ARGV[2])
    return 1
  end
  return 0
" 1 seat:{eventId}:{seatId} {userId} {expiryTimestamp}
```

This executes atomically within Redis (single-threaded), eliminating race conditions. The DB is only updated when the purchase is confirmed (asynchronous write-behind).

#### Preventing Double-Selling

Three layers of defense:
1. **Redis Lua script** (first line): Atomic check-and-set prevents concurrent holds on the same seat
2. **Database UNIQUE constraint** on `tickets.seat_id`: Even if Redis fails, the DB rejects duplicate seat assignments
3. **Optimistic locking**: `seats.version` column detects concurrent modifications during the DB write

#### Real-Time Seat Map Updates

When a seat status changes, the event is published to Kafka topic `seat-updates.{eventId}`. WebSocket servers subscribe to relevant topics and fan out updates to connected clients. With 5M concurrent viewers, we use a WebSocket server cluster behind a load balancer with sticky sessions (by event ID). Each server handles ~50K connections. Updates are batched (every 500ms) to reduce message volume.

### Bottlenecks & Mitigations

| Bottleneck | Mitigation |
|---|---|
| 50K concurrent seat locks on one event | Redis Lua scripts for atomic operations; shard by section |
| 5M WebSocket connections | Cluster of WS servers; batch updates every 500ms; use SSE as fallback |
| Database write storm when holds convert to purchases | Batch DB writes; queue purchases through Kafka |
| Bot-driven scalping | Virtual queue with CAPTCHA; device fingerprinting; purchase velocity limits |
| Expired hold thundering herd | Jittered TTLs; Redis keyspace notifications trigger gradual release |
| Single event as hot key | Dedicated Redis instance per major event; pre-shard seat data by section |

### Key Takeaways

- **Virtual queues** transform an uncontrollable stampede into a manageable flow — essential for flash sales
- **Redis Lua scripts** provide atomic seat operations without database round-trips under extreme concurrency
- **Defense in depth**: Redis for speed, DB unique constraints for correctness, optimistic locking for conflict detection
- **WebSocket with batched updates** makes real-time seat maps feasible at millions of concurrent connections
- The **hold-then-purchase** pattern with TTL prevents seats from being locked indefinitely by abandoned sessions
- Architect flash sales as a **separate mode** with different infrastructure, not just "scale up" the normal path

---

## 5. Ad Click Aggregation & Real-Time Analytics

### Problem Statement

An ad platform like Google Ads or Meta Ads serves billions of ad impressions per day, tracks every click, and must aggregate this data in near real-time to serve multiple consumers: advertisers monitoring campaign performance, billing systems calculating charges, fraud detection systems identifying invalid clicks, and machine learning pipelines optimizing ad targeting.

The core challenge is performing accurate aggregations over a massive, high-velocity event stream. Google processes approximately 8.5 billion ad impressions per day. Each impression and click generates an event that must be counted, attributed to the correct campaign, deduplicated (users may double-click), filtered for fraud, and aggregated into time-windowed metrics — all with sub-minute latency.

The accuracy-timeliness tradeoff is critical. Advertisers expect real-time dashboards, but late-arriving events (due to network delays, client-side batching, or timezone differences) mean that counts for any given time window may change retroactively. The system must support both fast approximate counts for dashboards and eventually-exact counts for billing.

### Use Cases

- Track ad impressions and clicks across web and mobile surfaces
- Aggregate clicks per campaign/ad group/keyword in real time (1-minute windows)
- Detect and filter fraudulent clicks (bot traffic, click farms, competitor sabotage)
- Calculate advertiser charges based on CPC/CPM billing models
- Provide real-time dashboards showing CTR, spend, conversions, and ROI
- Support historical analytics with arbitrary time range queries
- A/B test ad creatives and landing pages with statistical significance
- Generate billing reports with reconciled, fraud-filtered counts

### Functional Requirements

- FR1: Ingest billions of impression/click events per day with < 1% data loss
- FR2: Aggregate events by campaign, ad group, creative, and time window
- FR3: Provide real-time (< 1 minute lag) aggregated metrics to advertiser dashboards
- FR4: Detect and flag fraudulent clicks using rule-based and ML-based models
- FR5: Compute billing amounts using fraud-filtered, deduplicated click counts
- FR6: Support drill-down queries: campaign → ad group → keyword → time range
- FR7: Handle late-arriving events and adjust aggregations retroactively
- FR8: Provide conversion tracking by correlating ad clicks with downstream actions

### Non-Functional Requirements

- NFR1: **Throughput** — Ingest 100,000 events/second sustained; 500K/sec peak
- NFR2: **Latency** — Real-time aggregations available within 60 seconds of event
- NFR3: **Accuracy** — Billing counts accurate to 99.99% after reconciliation
- NFR4: **Availability** — 99.99% for ingestion pipeline; no events dropped during deploys
- NFR5: **Durability** — Every event persisted for 7 years (audit and billing requirements)
- NFR6: **Scalability** — Linearly scale ingestion with added Kafka partitions
- NFR7: **Idempotency** — Exactly-once aggregation semantics despite at-least-once delivery
- NFR8: **Query performance** — Dashboard queries return within 500ms over billions of rows

### Capacity Estimation

- **Impressions**: 8 billion/day → **92,593/sec** (avg), **~300K/sec** (peak)
- **Clicks**: 200 million/day (2.5% CTR) → **2,315/sec** (avg), **~10K/sec** (peak)
- **Event size**: ~500 bytes (impression), ~300 bytes (click)
- **Daily ingestion**: 8B × 500 bytes + 200M × 300 bytes = **4 TB + 60 GB ≈ 4.06 TB/day**
- **Annual storage (raw)**: 4.06 TB × 365 = **1.48 PB/year** (before compression; ~300 TB compressed with Parquet)
- **Aggregated data**: ~10 million unique campaign-hour buckets/day × 200 bytes = **2 GB/day** aggregated
- **Bandwidth (ingestion)**: 300K events/sec × 500 bytes = **150 MB/s** peak ingest
- **Dashboard queries**: 1M advertisers × 10 queries/day = **116 QPS** (avg)

### API Design

```http
# Event ingestion (high-throughput, batched)
POST /api/v1/events/batch
Header: Authorization: Bearer api_key
Body: {
  "events": [
    { "eventId": "evt_uuid_1", "type": "impression", "adId": "ad_123", "campaignId": "camp_456",
      "timestamp": "2025-01-15T10:30:00.123Z", "userId": "uid_hash", "ip": "203.0.113.5",
      "userAgent": "Mozilla/5.0...", "deviceType": "mobile", "surface": "search" },
    { "eventId": "evt_uuid_2", "type": "click", "adId": "ad_123", ... }
  ]
}
Response: 202 Accepted { "accepted": 2, "rejected": 0 }

# Query aggregated metrics
GET /api/v1/analytics/campaigns/{campaignId}/metrics?metrics=impressions,clicks,ctr,spend&granularity=hour&from=2025-01-15T00:00:00Z&to=2025-01-15T23:59:59Z
Response: {
  "campaignId": "camp_456",
  "timeSeries": [
    { "timestamp": "2025-01-15T10:00:00Z", "impressions": 45230, "clicks": 1205, "ctr": 0.0266, "spend_cents": 60250 }
  ]
}

# Real-time dashboard (WebSocket)
WS /api/v1/analytics/stream?campaignIds=camp_456,camp_789
Message: { "campaignId": "camp_456", "window": "2025-01-15T10:30:00Z", "impressions": 752, "clicks": 19 }

# Fraud report
GET /api/v1/analytics/campaigns/{campaignId}/fraud-report?from=2025-01-01&to=2025-01-31
Response: { "totalClicks": 500000, "invalidClicks": 12500, "invalidRate": 0.025, "creditsIssued": 6250 }
```

### Data Model

```sql
-- Raw event store (append-only, Kafka + object storage)
-- Stored as Parquet in S3/HDFS, partitioned by date and hour
-- Schema (logical):
-- event_id       STRING (UUID, deduplication key)
-- event_type     STRING (impression | click | conversion)
-- ad_id          STRING
-- campaign_id    STRING
-- advertiser_id  STRING
-- user_id        STRING (hashed)
-- ip_address     STRING
-- user_agent     STRING
-- device_type    STRING
-- surface        STRING (search | display | video)
-- timestamp      TIMESTAMP
-- fraud_score    FLOAT (added by fraud pipeline)
-- is_valid       BOOLEAN (set after fraud filtering)

-- Pre-aggregated metrics (OLAP store, e.g., Apache Druid or ClickHouse)
CREATE TABLE campaign_metrics_hourly (
    campaign_id     VARCHAR(64) NOT NULL,
    ad_group_id     VARCHAR(64),
    ad_id           VARCHAR(64),
    hour            TIMESTAMP NOT NULL,
    impressions     BIGINT DEFAULT 0,
    clicks          BIGINT DEFAULT 0,
    valid_clicks    BIGINT DEFAULT 0,
    conversions     BIGINT DEFAULT 0,
    spend_cents     BIGINT DEFAULT 0,
    fraud_clicks    BIGINT DEFAULT 0,
    PRIMARY KEY (campaign_id, hour)
) ENGINE = AggregatingMergeTree()                 -- ClickHouse
PARTITION BY toYYYYMM(hour)
ORDER BY (campaign_id, ad_group_id, hour);

-- Deduplication store (Redis or RocksDB)
-- Key: event_id, Value: 1, TTL: 24 hours
-- Used by stream processor to detect duplicates

-- Billing ledger
CREATE TABLE billing_entries (
    id              BIGINT PRIMARY KEY AUTO_INCREMENT,
    advertiser_id   VARCHAR(64) NOT NULL,
    campaign_id     VARCHAR(64) NOT NULL,
    billing_date    DATE NOT NULL,
    valid_clicks    BIGINT NOT NULL,
    cpc_cents       BIGINT NOT NULL,
    total_charge    BIGINT NOT NULL,
    status          ENUM('PENDING','RECONCILED','ADJUSTED'),
    INDEX idx_advertiser_date (advertiser_id, billing_date)
);
```

### High-Level Design

```mermaid
graph TB
    subgraph Event Sources
        Web[Web Pixel / SDK]
        App[Mobile SDK]
        Server[Server-Side Events]
    end

    subgraph Ingestion
        LB[Load Balancer]
        Ingest[Ingestion Service<br/>Validation / Batching]
        Kafka[Apache Kafka<br/>Partitioned by campaign_id]
    end

    subgraph Stream Processing
        Flink[Apache Flink<br/>Dedup / Aggregate / Window]
        FraudML[Fraud Detection<br/>ML Pipeline]
    end

    subgraph Storage
        S3[(S3 / HDFS<br/>Raw Events - Parquet)]
        CH[(ClickHouse<br/>Aggregated Metrics)]
        Redis_Dedup[(Redis<br/>Dedup Store)]
        PG_Bill[(PostgreSQL<br/>Billing Ledger)]
    end

    subgraph Serving
        QS[Query Service<br/>Dashboard API]
        BS[Billing Service]
        Dash[Advertiser Dashboard]
    end

    Web --> LB
    App --> LB
    Server --> LB
    LB --> Ingest
    Ingest --> Kafka

    Kafka --> Flink
    Kafka --> S3
    Flink --> Redis_Dedup
    Flink --> FraudML
    Flink --> CH
    FraudML --> CH

    CH --> QS
    QS --> Dash
    CH --> BS
    BS --> PG_Bill
```

### Deep Dive

#### Lambda Architecture: Speed + Batch Layers

The system uses a modified **Lambda architecture**:

- **Speed layer** (Apache Flink): Processes events in real time with 1-minute tumbling windows. Produces approximate, low-latency aggregations. Uses event-time processing with watermarks to handle out-of-order events (watermark delay: 5 minutes).
- **Batch layer** (Spark on S3/Parquet): Runs daily, reprocessing all raw events for the previous day. Produces exact, reconciled counts. Handles late arrivals that missed the speed layer's watermark.
- **Serving layer** (ClickHouse): Stores both real-time and batch-reconciled aggregations. Dashboard queries hit real-time data for "today"; historical queries hit batch-reconciled data.

#### Exactly-Once Aggregation

Kafka provides at-least-once delivery, but we need exactly-once counting:

1. **Event-level deduplication**: Each event has a UUID (`eventId`). Flink checks Redis bloom filter (low memory) + exact lookup for positives. Duplicates are dropped.
2. **Checkpoint-based recovery**: Flink checkpoints consumer offsets + aggregation state atomically. On failure, it replays from the last checkpoint without double-counting.
3. **Idempotent sink**: ClickHouse's `ReplacingMergeTree` deduplicates by primary key on merge.

#### Fraud Detection Pipeline

```mermaid
graph LR
    Click[Click Event] --> Rules[Rule Engine<br/>IP Rate Limit<br/>Device Fingerprint<br/>Geographic Anomaly]
    Rules --> Score{Score > 0.7?}
    Score -->|Yes| Block[Block & Flag]
    Score -->|No| ML[ML Model<br/>Gradient Boosted Trees]
    ML --> MLScore{Score > 0.8?}
    MLScore -->|Yes| Review[Manual Review Queue]
    MLScore -->|No| Valid[Valid Click]
    Block --> Metrics[Update Fraud Metrics]
    Review --> Metrics
    Valid --> Billing[Billing Pipeline]
```

Rule-based checks (real-time, < 10ms): IP rate limiting (> 10 clicks/minute from same IP), known bot user-agents, data center IP ranges, impossibly fast click sequences.

ML-based checks (near-real-time, < 100ms): Feature vector includes click timing patterns, device diversity, conversion rate anomalies, and geographic clustering. Model retrained daily on labeled data.

### Bottlenecks & Mitigations

| Bottleneck | Mitigation |
|---|---|
| Kafka ingestion at 300K events/sec | 256+ partitions; partition by campaign_id for locality |
| Flink state size for dedup (billions of event IDs) | Redis bloom filter (1% false positive OK); TTL 24h |
| ClickHouse query on high-cardinality dimensions | Pre-aggregate at multiple granularities; materialized views |
| Late-arriving events changing historical counts | Batch reconciliation layer; separate "preliminary" vs "final" counts |
| Fraud model false positives blocking valid clicks | Two-tier: rules block obvious fraud; ML flags for review |
| Dashboard query thundering herd | Query result cache (Redis, 30s TTL); rate limit per advertiser |

### Key Takeaways

- **Lambda architecture** provides both real-time dashboards (speed layer) and accurate billing (batch layer)
- **Exactly-once semantics** require deduplication at ingestion + idempotent sinks + checkpointed state
- **Fraud detection** is a multi-stage pipeline: fast rules first, then ML, then human review
- Store raw events permanently in columnar format (Parquet) — they're the source of truth for billing disputes
- Pre-aggregate at write time into ClickHouse for fast dashboard queries; avoid scanning raw events at read time
- **Watermarks** in stream processing handle late-arriving events gracefully

---

## 6. Stock Exchange / Trading Platform

### Problem Statement

A stock exchange is a matching engine that pairs buy and sell orders for financial instruments at the best available price. Systems like NASDAQ, NYSE, and modern crypto exchanges process millions of orders per day with latency requirements measured in microseconds. The matching engine is the core of global capital markets, and its correctness directly impacts the financial well-being of millions of investors.

The defining challenge is **ultra-low latency with deterministic ordering**. Every order must be processed in strict FIFO sequence to ensure price-time priority fairness. A matching engine at NASDAQ processes ~1 million messages per second with median latency under 50 microseconds. This rules out most distributed systems patterns — the hot path cannot involve network round-trips to databases, message brokers, or external services.

Additionally, the system must maintain a real-time order book (the current state of all outstanding buy/sell orders), disseminate market data (prices, volumes, trade confirmations) to thousands of subscribers with microsecond-level latency, and ensure that the exchange's state can be recovered exactly after a crash — all while operating under strict regulatory requirements (SEC, FINRA) for audit trails and fair access.

### Use Cases

- Traders submit limit orders, market orders, and stop orders
- Matching engine pairs compatible buy and sell orders at the best price
- Real-time order book shows current bid/ask depth at each price level
- Market data feed broadcasts trades, quotes, and order book updates to subscribers
- Traders cancel or modify open orders
- End-of-day settlement and clearing
- Historical trade data for compliance, audit, and analytics
- Circuit breakers halt trading during extreme volatility

### Functional Requirements

- FR1: Accept limit, market, stop, and stop-limit order types
- FR2: Match orders using price-time priority (best price first, then earliest order)
- FR3: Support partial fills — a large order may match against multiple smaller orders
- FR4: Maintain a real-time order book per instrument with bid/ask levels
- FR5: Disseminate market data (trades, quotes, book updates) in real time
- FR6: Allow order cancellation and modification (cancel-replace)
- FR7: Enforce pre-trade risk checks (position limits, margin requirements)
- FR8: Generate complete audit trail with nanosecond-precision timestamps

### Non-Functional Requirements

- NFR1: **Latency** — Order-to-acknowledgment: < 100μs (median), < 500μs (p99)
- NFR2: **Throughput** — 1 million orders/second sustained
- NFR3: **Determinism** — All orders processed in strict FIFO sequence; no reordering
- NFR4: **Availability** — 99.999% during trading hours (9:30 AM - 4:00 PM)
- NFR5: **Durability** — All orders and trades journaled to persistent storage before acknowledgment
- NFR6: **Consistency** — Matching engine is a single-threaded state machine; no partial states
- NFR7: **Recovery** — Full state recovery from journal replay in < 30 seconds
- NFR8: **Fairness** — Equal latency for all participants (co-location, deterministic gateway)

### Capacity Estimation

- **Instruments**: 10,000 actively traded symbols
- **Orders**: 5 million orders/day (avg); **1 million orders/sec** burst during market open
- **Trades (executions)**: 2 million/day → **~23 trades/sec** (avg), **~10,000/sec** (peak)
- **Market data messages**: Each order/trade generates 2-3 updates → **3 million messages/sec** peak
- **Order size**: ~200 bytes → 1M orders/sec × 200 bytes = **200 MB/s** peak ingest
- **Order book depth**: 10,000 instruments × 20 price levels × 2 sides = **400,000 active price levels**
- **Market data fan-out**: 3M messages/sec × 1,000 subscribers = **3 billion messages/sec** fan-out (via multicast)
- **Journal storage**: 200 MB/s × 6.5 hours trading day = **4.68 TB/day** (raw journal)
- **Historical trade data**: 2M trades/day × 500 bytes × 252 trading days = **252 GB/year**

### API Design

```
# FIX Protocol (industry standard, not REST — latency-critical)
# Simplified representation of FIX messages:

# New Order Single (client → exchange)
8=FIX.4.4 | 35=D | 49=TRADER_A | 55=AAPL | 54=1 (Buy) | 38=100 (qty) |
  40=2 (Limit) | 44=150.25 (price) | 11=ORD_UUID_123 (ClOrdID)

# Execution Report (exchange → client)
8=FIX.4.4 | 35=8 | 17=EXEC_456 | 37=ORD_789 | 55=AAPL | 54=1 |
  39=2 (Filled) | 32=100 (lastQty) | 31=150.25 (lastPrice)

# Order Cancel Request
8=FIX.4.4 | 35=F | 41=ORD_UUID_123 (origClOrdID)
```

```http
# REST API for non-latency-critical operations (market data, history)
GET /api/v1/orderbook/{symbol}?depth=20
Response: {
  "symbol": "AAPL",
  "timestamp": "2025-01-15T14:30:00.123456Z",
  "bids": [ { "price": 150.25, "quantity": 5000, "orders": 12 }, ... ],
  "asks": [ { "price": 150.26, "quantity": 3200, "orders": 8 }, ... ]
}

GET /api/v1/trades/{symbol}?from=2025-01-15T14:00:00Z&limit=100
GET /api/v1/orders?status=open&symbol=AAPL

# WebSocket for market data streaming
WS /api/v1/market-data/stream?symbols=AAPL,GOOGL,MSFT
Message: { "type": "trade", "symbol": "AAPL", "price": 150.25, "qty": 100, "ts": "..." }
Message: { "type": "book_update", "symbol": "AAPL", "side": "bid", "price": 150.24, "qty": 8000 }
```

### Data Model

```sql
-- Orders (write-ahead journal, append-only)
CREATE TABLE orders (
    id              BIGINT PRIMARY KEY,             -- Monotonic sequence
    client_order_id VARCHAR(64) NOT NULL,
    account_id      VARCHAR(64) NOT NULL,
    symbol          VARCHAR(10) NOT NULL,
    side            ENUM('BUY','SELL') NOT NULL,
    order_type      ENUM('LIMIT','MARKET','STOP','STOP_LIMIT'),
    quantity        BIGINT NOT NULL,                -- In shares/units
    price           BIGINT,                         -- Fixed-point: price × 10^6
    time_in_force   ENUM('DAY','GTC','IOC','FOK'),
    status          ENUM('NEW','PARTIALLY_FILLED','FILLED','CANCELLED','REJECTED'),
    filled_qty      BIGINT DEFAULT 0,
    avg_fill_price  BIGINT DEFAULT 0,
    received_at     BIGINT NOT NULL,                -- Nanosecond timestamp
    INDEX idx_symbol_side (symbol, side, price, received_at),
    INDEX idx_account (account_id, status)
);

-- Executions / Trades (immutable)
CREATE TABLE executions (
    id              BIGINT PRIMARY KEY,
    buy_order_id    BIGINT NOT NULL,
    sell_order_id   BIGINT NOT NULL,
    symbol          VARCHAR(10) NOT NULL,
    price           BIGINT NOT NULL,
    quantity        BIGINT NOT NULL,
    executed_at     BIGINT NOT NULL,                -- Nanosecond timestamp
    INDEX idx_symbol_time (symbol, executed_at)
);

-- In-memory order book structure (not persisted to SQL — lives in matching engine RAM)
-- Represented as:
--   BID side: Max-heap (sorted by price DESC, then time ASC)
--   ASK side: Min-heap (sorted by price ASC, then time ASC)
-- Each price level: { price: 15025, orders: LinkedList<Order>, totalQty: 5000 }
```

### High-Level Design

```mermaid
graph TB
    subgraph Clients
        T1[Trader Terminal]
        T2[Algo Trading System]
        MM[Market Maker]
    end

    subgraph Gateway Layer
        FGW[FIX Gateway<br/>Protocol Translation]
        RGW[Risk Gateway<br/>Pre-Trade Checks]
    end

    subgraph Core
        SEQ[Sequencer<br/>Assigns monotonic IDs]
        ME[Matching Engine<br/>Single-threaded<br/>In-memory Order Book]
        Journal[(Write-Ahead Journal<br/>Persistent Log)]
    end

    subgraph Market Data
        MDP[Market Data Publisher<br/>Multicast / WebSocket]
        HIST[(Historical DB<br/>TimescaleDB)]
    end

    subgraph Post-Trade
        CLR[Clearing & Settlement]
        RISK[Risk Management]
        REG[Regulatory Reporting]
    end

    T1 --> FGW
    T2 --> FGW
    MM --> FGW
    FGW --> RGW
    RGW --> SEQ
    SEQ --> Journal
    SEQ --> ME
    ME --> MDP
    ME --> Journal
    MDP --> T1
    MDP --> T2
    MDP --> MM
    MDP --> HIST
    ME --> CLR
    ME --> RISK
    CLR --> REG
```

### Deep Dive

#### The Matching Engine: Single-Threaded by Design

The matching engine is deliberately **single-threaded**. This is counterintuitive for a system handling millions of messages per second, but it eliminates the need for locks, atomic operations, and the non-determinism that comes with concurrent execution.

**Order book data structure** (per symbol):
- **Bid side**: Sorted map (red-black tree) keyed by price (descending). Each price level contains a FIFO queue of orders.
- **Ask side**: Sorted map keyed by price (ascending). Same structure.
- **Matching algorithm** (price-time priority):

```
function matchOrder(incoming):
    if incoming.side == BUY:
        while incoming.remainingQty > 0 AND bestAsk.price <= incoming.price:
            oppositeOrder = bestAsk.firstOrder()
            fillQty = min(incoming.remainingQty, oppositeOrder.remainingQty)
            createExecution(incoming, oppositeOrder, fillQty, oppositeOrder.price)
            incoming.remainingQty -= fillQty
            oppositeOrder.remainingQty -= fillQty
            if oppositeOrder.remainingQty == 0:
                removeOrder(oppositeOrder)
        if incoming.remainingQty > 0 AND incoming.type == LIMIT:
            addToBook(incoming)  // Rest on the bid side
```

A modern matching engine implemented in C++ or Java (with off-heap memory) can process **10+ million messages per second** on a single core, with < 1μs per match.

#### Write-Ahead Journal and Recovery

Every message entering the matching engine is first written to a **write-ahead journal** (sequential append to memory-mapped file). On crash, the matching engine replays the journal from the last snapshot to reconstruct the exact in-memory state.

```
Snapshot (taken every N minutes):
  [Full order book state serialized to disk]

Journal entries (continuous):
  [SEQ_1] NEW_ORDER: BUY AAPL 100@150.25
  [SEQ_2] NEW_ORDER: SELL AAPL 50@150.25
  [SEQ_3] EXECUTION: 50 AAPL @150.25
  [SEQ_4] CANCEL: ORDER_123

Recovery = Load last snapshot + replay journal entries after snapshot
```

#### Market Data Dissemination

Market data is the highest fan-out component. For 10,000 subscribers receiving 3M updates/sec:

- **Kernel bypass networking** (DPDK/Solarflare): Eliminates OS network stack overhead for single-digit microsecond latency.
- **UDP multicast** for co-located participants: One packet serves all subscribers on the same network segment.
- **Incremental updates**: Send only changes (delta), not full order book snapshots.
- **Conflated feeds**: For slower consumers, merge multiple updates for the same price level into one message.

### Bottlenecks & Mitigations

| Bottleneck | Mitigation |
|---|---|
| Single-threaded matching engine throughput ceiling | Shard by symbol; each symbol group gets its own engine instance |
| Journal write latency | Memory-mapped files; NVMe SSDs; batched fsync every 1ms |
| Market data fan-out at 3M msg/sec × 1000 subscribers | UDP multicast for co-located; incremental + conflated feeds |
| Recovery time from crash | Periodic snapshots (every 5 min) + journal replay; hot standby |
| Network latency fairness | Deterministic gateway with equalized cable lengths; randomized batching |
| Hot symbols (AAPL, TSLA) dominating throughput | Dedicated matching engine per hot symbol |

### Key Takeaways

- The matching engine is **single-threaded by design** — determinism and strict ordering trump parallelism
- **Price-time priority** is the industry-standard matching algorithm; implemented with sorted maps and FIFO queues
- **Write-ahead journal** with periodic snapshots enables crash recovery without data loss
- Fixed-point arithmetic (price × 10^6) avoids floating-point rounding in financial calculations
- Market data uses **UDP multicast** and **kernel bypass** networking for microsecond-level distribution
- Shard by **symbol** to horizontally scale matching engine throughput while maintaining per-symbol ordering

---

## 7. Uber / Ride-Sharing Platform

### Problem Statement

A ride-sharing platform like Uber connects riders requesting transportation with nearby drivers in real time. The system must handle location tracking for millions of simultaneously active drivers, match riders to optimal drivers considering distance, ETA, driver rating, and vehicle type, compute dynamic pricing based on real-time supply and demand, and manage the full trip lifecycle from request to payment.

The core engineering challenge is **real-time geospatial matching at scale**. When a rider requests a ride, the system must find the N nearest available drivers, estimate arrival times considering real road networks (not straight-line distance), select the best match, and dispatch the request — all within 2-3 seconds. Meanwhile, every active driver is reporting their GPS location every 3-4 seconds, generating a continuous stream of millions of location updates per second that must be indexed for spatial queries.

Beyond matching, the system must compute **surge pricing** that dynamically adjusts fares based on the ratio of rider demand to driver supply in each geographic zone, maintain **real-time ETAs** that account for traffic conditions, and handle the complex state machine of a trip (requested → matched → en route → in progress → completed → paid) across unreliable mobile network connections.

### Use Cases

- Rider requests a ride specifying pickup location, destination, and ride type (UberX, Black, Pool)
- System matches rider with nearest available driver and shows ETA
- Driver receives ride request with rider location and navigates to pickup
- Real-time trip tracking for both rider and driver with live map
- Dynamic (surge) pricing based on supply/demand in the area
- Fare calculation based on distance, time, base rate, and surge multiplier
- Post-trip rating and payment processing
- Driver location continuously tracked and indexed for spatial queries

### Functional Requirements

- FR1: Riders request rides with pickup/drop-off locations and vehicle preference
- FR2: Match rider with optimal nearby driver within 10 seconds
- FR3: Real-time driver location tracking on rider's map during en-route and in-trip phases
- FR4: Compute fare estimate before ride request; final fare after trip completion
- FR5: Dynamic surge pricing adjusts based on zone-level supply/demand ratios
- FR6: Manage trip state machine: REQUESTED → MATCHED → DRIVER_EN_ROUTE → IN_PROGRESS → COMPLETED
- FR7: Both rider and driver can rate each other post-trip
- FR8: Handle ride cancellations with appropriate fee policies

### Non-Functional Requirements

- NFR1: **Latency** — Driver match within 5 seconds; ETA calculation within 500ms
- NFR2: **Throughput** — 1 million location updates/second from active drivers
- NFR3: **Availability** — 99.99% for ride request and matching
- NFR4: **Freshness** — Driver locations no more than 5 seconds stale in geospatial index
- NFR5: **Scalability** — Support 5 million concurrent drivers and 20 million concurrent riders
- NFR6: **Accuracy** — ETA estimates within ±2 minutes of actual 90% of the time
- NFR7: **Consistency** — A driver can only be matched to one rider at a time
- NFR8: **Resilience** — Graceful handling of GPS drift, network disconnects, and location gaps

### Capacity Estimation

- **Active drivers**: 5 million simultaneously active (globally)
- **Location updates**: 5M drivers × 1 update/4 sec = **1.25 million updates/sec**
- **Ride requests**: 20 million rides/day → **231 requests/sec** (avg), **~2,000/sec** (peak rush hour)
- **Location update size**: ~100 bytes (driver_id, lat, lng, heading, speed, timestamp) → 1.25M × 100 = **125 MB/s**
- **Storage (trips)**: 20M rides/day × 1 KB = **20 GB/day**, **7.3 TB/year**
- **Storage (location history)**: 1.25M/sec × 100 bytes × 86400 sec = **10.8 TB/day** raw (downsampled for archival)
- **ETA calculations**: Each ride request evaluates ~20 nearby drivers × route calculation = **~4,000 route calculations/sec** (peak)
- **Surge pricing recalculation**: Every 2 minutes across 50,000 geo zones = **417 zones/sec**

### API Design

```http
# Request a ride
POST /api/v1/rides
Header: Idempotency-Key: "ride_uuid_123"
Body: {
  "riderId": "rider_abc",
  "pickup": { "lat": 37.7749, "lng": -122.4194, "address": "123 Market St" },
  "dropoff": { "lat": 37.3382, "lng": -121.8863, "address": "456 San Jose Blvd" },
  "rideType": "UberX",
  "paymentMethodId": "pm_visa_1"
}
Response: 201 Created {
  "rideId": "ride_xyz",
  "status": "MATCHING",
  "fareEstimate": { "min": 3200, "max": 4500, "currency": "usd", "surgeMultiplier": 1.5 }
}

# Get fare estimate (before requesting)
GET /api/v1/fare-estimate?pickupLat=37.7749&pickupLng=-122.4194&dropoffLat=37.3382&dropoffLng=-121.8863&rideType=UberX

# Driver location update (high frequency)
PUT /api/v1/drivers/{driverId}/location
Body: { "lat": 37.7751, "lng": -122.4180, "heading": 45, "speed": 25.5, "timestamp": "..." }
Response: 204 No Content

# Driver accepts ride
POST /api/v1/rides/{rideId}/accept
Body: { "driverId": "driver_456" }
Response: 200 OK { "pickup": { ... }, "riderName": "John", "riderRating": 4.8 }

# Complete trip
POST /api/v1/rides/{rideId}/complete
Body: { "finalLocation": { "lat": 37.3382, "lng": -121.8863 }, "odometerKm": 72.3 }
Response: 200 OK { "fare": { "baseFare": 250, "distance": 2800, "time": 900, "surge": 750, "total": 4700 } }

# Rate trip
POST /api/v1/rides/{rideId}/rating
Body: { "rating": 5, "comment": "Great driver", "ratedBy": "rider" }

# Real-time tracking (WebSocket)
WS /api/v1/rides/{rideId}/track
Message: { "driverLocation": { "lat": 37.775, "lng": -122.418 }, "eta": 180, "status": "DRIVER_EN_ROUTE" }
```

### Data Model

```sql
CREATE TABLE drivers (
    id              BIGINT PRIMARY KEY,
    name            VARCHAR(200),
    phone           VARCHAR(20),
    vehicle_type    ENUM('UberX','Comfort','Black','XL','Pool'),
    license_plate   VARCHAR(20),
    rating          DECIMAL(3,2) DEFAULT 5.00,
    status          ENUM('OFFLINE','AVAILABLE','ON_TRIP','MATCHING'),
    current_lat     DOUBLE,
    current_lng     DOUBLE,
    last_location_at TIMESTAMP,
    geohash         VARCHAR(12),                    -- For spatial indexing
    INDEX idx_geohash_status (geohash, status, vehicle_type),
    INDEX idx_status (status)
);

CREATE TABLE riders (
    id              BIGINT PRIMARY KEY,
    name            VARCHAR(200),
    phone           VARCHAR(20),
    rating          DECIMAL(3,2) DEFAULT 5.00,
    default_payment VARCHAR(64)
);

CREATE TABLE rides (
    id              BIGINT PRIMARY KEY,
    rider_id        BIGINT NOT NULL,
    driver_id       BIGINT,
    status          ENUM('REQUESTED','MATCHING','MATCHED','DRIVER_EN_ROUTE',
                         'ARRIVED','IN_PROGRESS','COMPLETED','CANCELLED'),
    ride_type       ENUM('UberX','Comfort','Black','XL','Pool'),
    pickup_lat      DOUBLE NOT NULL,
    pickup_lng      DOUBLE NOT NULL,
    dropoff_lat     DOUBLE NOT NULL,
    dropoff_lng     DOUBLE NOT NULL,
    surge_multiplier DECIMAL(3,2) DEFAULT 1.00,
    fare_estimate_cents BIGINT,
    final_fare_cents BIGINT,
    distance_meters  INT,
    duration_seconds INT,
    currency        CHAR(3) DEFAULT 'USD',
    payment_id      VARCHAR(64),
    idempotency_key VARCHAR(128) UNIQUE,
    requested_at    TIMESTAMP DEFAULT NOW(),
    matched_at      TIMESTAMP,
    started_at      TIMESTAMP,
    completed_at    TIMESTAMP,
    INDEX idx_rider (rider_id, status),
    INDEX idx_driver (driver_id, status),
    INDEX idx_status (status, requested_at)
) PARTITION BY RANGE (requested_at);

-- Location history (time-series, high volume)
CREATE TABLE driver_locations (
    driver_id       BIGINT NOT NULL,
    timestamp       TIMESTAMP NOT NULL,
    lat             DOUBLE NOT NULL,
    lng             DOUBLE NOT NULL,
    heading         SMALLINT,
    speed           DECIMAL(5,1),
    ride_id         BIGINT                         -- NULL if not on a trip
) PARTITION BY RANGE (timestamp);                  -- Hourly partitions
-- In practice, stored in a time-series DB (TimescaleDB, InfluxDB) or Kafka → S3

-- Surge pricing zones
CREATE TABLE surge_zones (
    zone_id         VARCHAR(20) PRIMARY KEY,       -- Geohash prefix (e.g., "9q8yy")
    demand          INT DEFAULT 0,                 -- Ride requests in last 5 min
    supply          INT DEFAULT 0,                 -- Available drivers in zone
    surge_multiplier DECIMAL(3,2) DEFAULT 1.00,
    updated_at      TIMESTAMP DEFAULT NOW()
);
```

### High-Level Design

```mermaid
graph TB
    subgraph Clients
        RA[Rider App]
        DA[Driver App]
    end

    subgraph Edge
        LB[Load Balancer]
        GW[API Gateway]
    end

    subgraph Core Services
        RS[Ride Service<br/>Trip State Machine]
        MS[Matching Service]
        LS[Location Service]
        PS[Pricing / Surge Service]
        ETAS[ETA Service]
        PAYS[Payment Service]
        NTF[Notification Service]
    end

    subgraph Data
        PG[(PostgreSQL<br/>Rides / Users)]
        Redis_Geo[(Redis<br/>GEOSEARCH Index)]
        Redis_Surge[(Redis<br/>Surge Zones)]
        TSDB[(TimescaleDB<br/>Location History)]
        Kafka[Kafka<br/>Location Events]
    end

    subgraph External
        Maps[Maps API<br/>Routing / ETA]
    end

    RA --> LB --> GW
    DA --> LB

    GW --> RS
    GW --> LS
    RS --> MS
    MS --> Redis_Geo
    MS --> ETAS
    ETAS --> Maps
    RS --> PS
    PS --> Redis_Surge
    RS --> PAYS
    RS --> NTF
    LS --> Redis_Geo
    LS --> Kafka
    Kafka --> TSDB
    Kafka --> PS
    RS --> PG
```

### Deep Dive

#### Real-Time Driver Location Indexing

Drivers report GPS coordinates every 3-4 seconds. The location service must update a spatial index that supports "find all available drivers within 3 km of point (lat, lng)."

**Redis GEOSEARCH approach**:
```
# Driver location update
GEOADD drivers:UberX:available {lng} {lat} {driverId}

# Find nearby drivers
GEOSEARCH drivers:UberX:available FROMLONLAT {lng} {lat} BYRADIUS 3000 m ASC COUNT 20
```

Redis's geospatial index uses a sorted set with geohash-encoded scores, providing O(N+log(M)) lookups where N is the result count and M is the set size. For 5 million drivers, this returns results in < 1ms.

**Geohash-based sharding**: The driver index is sharded by geohash prefix (4-5 characters, ~5km × 5km cells). Each shard handles drivers in a geographic region, enabling horizontal scaling. Border queries fan out to adjacent geohash cells.

#### Matching Algorithm

When a ride is requested:

```
1. Query Redis GEOSEARCH for 20 nearest available drivers of the right vehicle type
2. For each candidate, request ETA from Maps service (batched, parallel)
3. Score each candidate:
   score = w1 × (1/ETA) + w2 × driverRating + w3 × acceptanceRate + w4 × (1/distanceToDropoff)
4. Select the highest-scoring driver
5. Atomically set driver status to MATCHING (Redis CAS to prevent double-dispatch)
6. Send ride request to driver (push notification)
7. If driver doesn't accept within 15 seconds, release and try next candidate
```

The **atomic status check** (`MATCHING` lock) is critical — without it, the same driver could be dispatched to two riders simultaneously.

#### Surge Pricing

Surge pricing balances supply and demand:

```
surgeMultiplier = f(demand / supply) in a geo-zone

Every 2 minutes, for each surge zone:
  demand = count of ride requests in zone in last 5 minutes
  supply = count of available drivers in zone
  ratio = demand / supply
  if ratio > 1.5: surge = 1.0 + (ratio - 1.0) × 0.5  (capped at 5.0×)
  else: surge = 1.0
```

Surge zones are **geohash cells** (~1km × 1km). Demand is counted from Kafka ride-request events; supply is counted from the Redis geo index. The pricing service publishes updated multipliers to Redis, and the fare calculation reads from Redis at ride-request time.

#### Trip State Machine

```mermaid
stateDiagram-v2
    [*] --> REQUESTED: Rider requests ride
    REQUESTED --> MATCHING: Finding driver
    MATCHING --> MATCHED: Driver found
    MATCHING --> CANCELLED: No driver available / timeout
    MATCHED --> DRIVER_EN_ROUTE: Driver heading to pickup
    DRIVER_EN_ROUTE --> ARRIVED: Driver at pickup
    DRIVER_EN_ROUTE --> CANCELLED: Rider/driver cancels
    ARRIVED --> IN_PROGRESS: Trip started
    IN_PROGRESS --> COMPLETED: Trip ended
    COMPLETED --> [*]: Payment processed
    CANCELLED --> [*]: Cancellation fee (if applicable)
```

Each state transition is persisted as an event in Kafka and the rides table. If the rider app loses connectivity, the last known state is recoverable from the server.

### Bottlenecks & Mitigations

| Bottleneck | Mitigation |
|---|---|
| 1.25M location updates/sec overwhelming a single index | Shard Redis geo index by geohash prefix (geographic partitioning) |
| ETA API rate limits from Maps provider | Cache ETAs for common routes; use Dijkstra on local road graph for approximation |
| Driver double-dispatch (same driver sent to 2 riders) | Atomic CAS on driver status in Redis before dispatch |
| Surge zone computation at global scale | Pre-compute on Flink stream processor; publish to Redis every 2 min |
| GPS drift in urban canyons | Kalman filter to smooth GPS readings; snap to road network |
| Network disconnects during trip | Offline mode in driver app; sync location buffer on reconnect |

### Key Takeaways

- **Redis GEOSEARCH** (or equivalent) is the workhorse for real-time spatial queries on moving objects
- **Geohash-based sharding** enables horizontal scaling of the spatial index by geographic region
- The matching algorithm is a **multi-factor scoring function**, not just nearest-driver
- **Surge pricing** operates on geographic zones with supply/demand ratios recalculated every few minutes
- The trip is a **state machine** with well-defined transitions persisted as events for reliability
- Architect for **unreliable mobile networks**: buffer location updates, handle reconnects, support offline mode

---

## 8. Proximity Service / Yelp / Google Places

### Problem Statement

A proximity service enables users to discover nearby businesses, restaurants, attractions, and points of interest based on their current location. Systems like Yelp, Google Places, and Foursquare manage hundreds of millions of business listings worldwide and serve billions of "nearby" queries per day with sub-second latency.

The fundamental challenge is **efficient geospatial indexing and querying**. A query like "coffee shops within 1 mile of me, open now, rated 4+ stars" must search a spatial index to find geographically nearby candidates, filter by business attributes (category, hours, rating), rank results by a combination of distance, relevance, rating, and popularity, and return results in under 200ms — even when the search area contains thousands of candidates.

Unlike the ride-sharing location problem where objects (drivers) move continuously, businesses have fixed locations that change rarely. This allows for more sophisticated pre-computed spatial indexes. However, the ranking challenge is more complex: the "best" nearby coffee shop depends on distance, rating, number of reviews, price range, current open/closed status, user preferences, and even time of day. Building a search system that balances these signals into a meaningful ranking is as much an information retrieval problem as a geospatial one.

### Use Cases

- Search for nearby businesses by category ("restaurants near me")
- Filter results by rating, price range, open now, and specific attributes
- View business details including hours, photos, menu, reviews, and contact info
- Get directions to a selected business
- Write and read reviews with ratings and photos
- Business owners claim and manage their listing (update hours, respond to reviews)
- Discover trending or popular places in an area
- Autocomplete business names and addresses during search

### Functional Requirements

- FR1: Users can search for businesses within a radius of their location with category filters
- FR2: Results are ranked by a composite score of distance, rating, review count, and relevance
- FR3: Each business listing includes name, address, hours, photos, reviews, and contact info
- FR4: Users can filter by "open now," price range, rating threshold, and custom attributes
- FR5: Users can write reviews with 1-5 star ratings, text, and photos
- FR6: Business owners can claim listings and update business information
- FR7: Autocomplete suggests businesses and addresses as the user types
- FR8: Support map-based exploration (pan/zoom reveals businesses in viewport)

### Non-Functional Requirements

- NFR1: **Latency** — Nearby search results within 200ms at p99
- NFR2: **Throughput** — 100,000 search queries per second at peak
- NFR3: **Availability** — 99.99% for search; 99.9% for write operations (reviews, listing updates)
- NFR4: **Scalability** — Support 200 million business listings globally
- NFR5: **Freshness** — Business data updates reflected in search within 5 minutes
- NFR6: **Accuracy** — Geospatial queries correct to within 10 meters
- NFR7: **Global** — Low-latency access from any continent (multi-region deployment)
- NFR8: **Ranking quality** — Relevant, non-spammy results surfaced at top

### Capacity Estimation

- **Business listings**: 200 million globally
- **Users**: 50 million DAU
- **Search queries**: 50M DAU × 5 searches/day = 250M/day → **2,894 QPS** (avg), **~100K QPS** (peak)
- **Reviews written**: 2 million/day → **23/sec** (avg)
- **Reviews read**: 50M DAU × 3 business views × 10 reviews each = **1.5B review reads/day** → **17,361/sec**
- **Listing size**: avg 5 KB per business → 200M × 5 KB = **1 TB** business data
- **Review storage**: 2M/day × 500 bytes × 365 = **365 GB/year**
- **Photo storage**: 2M reviews/day × 30% with photos × 2 photos × 500 KB = **600 GB/day** → **219 TB/year**
- **Geospatial index size**: 200M businesses × 50 bytes (id + coordinates + geohash) = **10 GB** (fits in memory)

### API Design

```http
# Nearby search
GET /api/v1/search/nearby?lat=37.7749&lng=-122.4194&radius=1500&category=restaurant&minRating=4.0&priceRange=2,3&openNow=true&sort=best_match&cursor=abc&limit=20
Response: {
  "results": [
    {
      "id": "biz_123",
      "name": "Joe's Diner",
      "category": "restaurant",
      "lat": 37.7755,
      "lng": -122.4185,
      "distanceMeters": 85,
      "rating": 4.5,
      "reviewCount": 342,
      "priceRange": 2,
      "isOpenNow": true,
      "thumbnailUrl": "...",
      "snippet": "Best pancakes in the city..."
    }
  ],
  "nextCursor": "...",
  "totalResults": 156
}

# Viewport search (map pan/zoom)
GET /api/v1/search/viewport?swLat=37.77&swLng=-122.42&neLat=37.78&neLng=-122.41&category=cafe&limit=50

# Autocomplete
GET /api/v1/autocomplete?q=star&lat=37.7749&lng=-122.4194&limit=10
Response: { "suggestions": [ { "type": "business", "id": "biz_456", "name": "Starbucks", "address": "..." } ] }

# Get business details
GET /api/v1/businesses/{businessId}
Response: { "id": "biz_123", "name": "...", "hours": { "mon": "8:00-22:00", ... }, "photos": [...], ... }

# Get reviews
GET /api/v1/businesses/{businessId}/reviews?sort=recent&cursor=abc&limit=10

# Write a review
POST /api/v1/businesses/{businessId}/reviews
Header: Idempotency-Key: "review_uuid_123"
Body: { "rating": 5, "text": "Amazing food and service!", "photos": ["photo_url_1"] }
Response: 201 Created { "reviewId": "rev_789", ... }

# Business owner: update listing
PUT /api/v1/businesses/{businessId}
Header: Authorization: Bearer owner_token
Header: If-Match: "etag_xyz"
Body: { "hours": { "mon": "9:00-21:00", ... }, "phone": "+1-555-0123" }
```

### Data Model

```sql
CREATE TABLE businesses (
    id              BIGINT PRIMARY KEY,
    name            VARCHAR(500) NOT NULL,
    category_id     INT NOT NULL,
    lat             DOUBLE NOT NULL,
    lng             DOUBLE NOT NULL,
    geohash         VARCHAR(12) NOT NULL,           -- Pre-computed geohash
    address         JSONB,
    phone           VARCHAR(20),
    website         VARCHAR(500),
    hours           JSONB,                          -- {"mon": "8:00-22:00", ...}
    price_range     SMALLINT,                       -- 1-4
    attributes      JSONB,                          -- {"wifi": true, "outdoor_seating": true}
    avg_rating      DECIMAL(3,2) DEFAULT 0,
    review_count    INT DEFAULT 0,
    popularity_score FLOAT DEFAULT 0,               -- Pre-computed ranking signal
    status          ENUM('ACTIVE','CLOSED','PENDING_VERIFICATION'),
    owner_id        BIGINT,
    created_at      TIMESTAMP DEFAULT NOW(),
    updated_at      TIMESTAMP DEFAULT NOW(),
    INDEX idx_geohash (geohash),
    INDEX idx_category_geohash (category_id, geohash),
    INDEX idx_location USING GIST (ST_MakePoint(lng, lat))  -- PostGIS spatial index
);

CREATE TABLE categories (
    id              INT PRIMARY KEY,
    name            VARCHAR(100),
    parent_id       INT,                            -- Hierarchical categories
    slug            VARCHAR(100) UNIQUE
);

CREATE TABLE reviews (
    id              BIGINT PRIMARY KEY,
    business_id     BIGINT NOT NULL,
    user_id         BIGINT NOT NULL,
    rating          SMALLINT NOT NULL CHECK (rating BETWEEN 1 AND 5),
    text            TEXT,
    photos          JSONB,                          -- ["url1", "url2"]
    helpful_count   INT DEFAULT 0,
    status          ENUM('ACTIVE','FLAGGED','REMOVED') DEFAULT 'ACTIVE',
    idempotency_key VARCHAR(128) UNIQUE,
    created_at      TIMESTAMP DEFAULT NOW(),
    INDEX idx_business_recent (business_id, created_at DESC),
    INDEX idx_user (user_id, created_at DESC)
);

CREATE TABLE business_photos (
    id              BIGINT PRIMARY KEY,
    business_id     BIGINT NOT NULL,
    user_id         BIGINT,
    url             VARCHAR(500) NOT NULL,
    caption         VARCHAR(200),
    created_at      TIMESTAMP DEFAULT NOW(),
    INDEX idx_business (business_id)
);
```

### High-Level Design

```mermaid
graph TB
    subgraph Clients
        Web[Web App]
        Mobile[Mobile App]
    end

    subgraph Edge
        CDN[CDN<br/>Static Assets / Photos]
        LB[Load Balancer]
        GW[API Gateway]
    end

    subgraph Services
        SS[Search Service]
        BS[Business Service]
        RS[Review Service]
        AS[Autocomplete Service]
        RNK[Ranking Service]
        GEO[Geospatial Index Service]
        NS[Notification Service]
    end

    subgraph Data
        ES[(Elasticsearch<br/>Business Search Index)]
        PG[(PostgreSQL<br/>Businesses / Reviews)]
        Redis_Geo[(Redis<br/>Geospatial Cache)]
        Redis_Cache[(Redis<br/>Business Detail Cache)]
        S3[(S3<br/>Photos)]
        Trie[(Trie / Elasticsearch<br/>Autocomplete Index)]
    end

    subgraph Async
        Kafka[Kafka]
        Workers[Index Workers<br/>Ranking Pipeline]
    end

    Web --> CDN
    Mobile --> CDN
    CDN --> LB --> GW
    GW --> SS
    GW --> BS
    GW --> RS
    GW --> AS

    SS --> GEO
    SS --> RNK
    GEO --> ES
    GEO --> Redis_Geo
    BS --> PG
    BS --> Redis_Cache
    RS --> PG
    AS --> Trie

    RS --> Kafka
    BS --> Kafka
    Kafka --> Workers
    Workers --> ES
    Workers --> Redis_Geo
```

### Deep Dive

#### Geospatial Indexing Strategies

For 200 million businesses, we need an efficient spatial index. Four main approaches:

**1. Geohash (used for sharding and coarse filtering)**
- Encodes lat/lng as a base-32 string. Longer strings = smaller cells.
- 6 characters ≈ 1.2km × 0.6km cell. Good for "find businesses in this area."
- **Limitation**: Edge cases at geohash boundaries — a business 10 meters away may be in a different geohash cell. Solution: always query the target cell + 8 neighboring cells.

**2. Quadtree (good for variable-density areas)**
- Recursively divides space into quadrants. Dense areas (Manhattan) get more subdivisions; sparse areas (rural) stay coarse.
- Each leaf node contains up to K businesses. Search walks from root to relevant leaves.
- Build time: O(N log N). Query time: O(log N + K results).

**3. S2 Cells (Google's approach)**
- Projects Earth's surface onto a cube, then uses Hilbert curves for locality-preserving encoding.
- Cells at different levels approximate circles better than geohash rectangles.
- Excellent for covering a circular search radius with minimal cells.

**4. R-tree / PostGIS GIST index (used for precise queries)**
- Balanced tree of bounding boxes. Ideal for range queries and nearest-neighbor.
- PostGIS `ST_DWithin(geography, point, radius)` uses R-tree internally.

**Our approach**: Two-layer indexing:
- **Elasticsearch** with geohash-based geospatial queries for full-text + spatial + attribute search
- **Redis GEOSEARCH** for ultra-fast "nearest N" queries on hot cities (cached, <1ms)

#### Search and Ranking Pipeline

```mermaid
sequenceDiagram
    participant Client
    participant SearchSvc as Search Service
    participant ES as Elasticsearch
    participant Ranker as Ranking Service

    Client->>SearchSvc: nearby?lat=..&lng=..&category=restaurant
    SearchSvc->>ES: Geo-filtered query + category + rating filter
    ES-->>SearchSvc: 200 candidate businesses (coarse ranking)
    SearchSvc->>Ranker: Re-rank top 200 candidates
    Note over Ranker: Score = 0.3×(1/distance) + 0.25×rating<br/>+ 0.2×log(reviewCount) + 0.15×popularity<br/>+ 0.1×recency
    Ranker-->>SearchSvc: Ranked top 20 results
    SearchSvc-->>Client: Final results with pagination cursor
```

**Elasticsearch query** (simplified):
```json
{
  "query": {
    "bool": {
      "must": [ { "term": { "category": "restaurant" } } ],
      "filter": [
        { "geo_distance": { "distance": "1500m", "location": { "lat": 37.77, "lon": -122.42 } } },
        { "range": { "avg_rating": { "gte": 4.0 } } },
        { "term": { "status": "ACTIVE" } }
      ]
    }
  },
  "sort": [ { "_geo_distance": { "location": { "lat": 37.77, "lon": -122.42 }, "order": "asc" } } ]
}
```

The **ranking function** balances multiple signals:
- **Distance decay**: Closer businesses score higher, with exponential decay
- **Rating quality**: Bayesian average (not raw average) to avoid gaming by businesses with few reviews
- **Review velocity**: Recent reviews weigh more than old ones
- **Popularity**: Click-through rate from past search impressions
- **Business freshness**: Recently updated listings get a small boost

#### "Open Now" Filtering

Checking business hours seems simple but is surprisingly tricky:
- Business hours are stored per day-of-week with timezone
- Query-time filter: convert user's current time to business's timezone, check if within operating hours
- **Optimization**: Pre-compute an `is_open` bitfield per 15-minute slot (672 bits for a week) and store in Elasticsearch. This turns an expensive time-range check into a simple bit lookup.

#### Review Anti-Spam

Fake reviews are a critical problem. Defense layers:
1. **Rate limiting**: Max 3 reviews/user/day; no reviewing the same business twice
2. **NLP-based detection**: ML model trained on known fake reviews (flags suspicious language patterns, copied text)
3. **Behavioral signals**: New accounts, burst review patterns, reviews for geographically dispersed businesses
4. **Network analysis**: Graph-based detection of review rings (groups of accounts that review the same businesses)

Flagged reviews are hidden from public view and sent to human moderators.

### Bottlenecks & Mitigations

| Bottleneck | Mitigation |
|---|---|
| 100K search QPS on Elasticsearch | Shard ES by geohash prefix (geographic routing); read replicas per region |
| Ranking computation for 200 candidates per query | Pre-compute popularity scores offline; lightweight online re-ranking |
| Geohash boundary problem (missing nearby results) | Always query target cell + 8 neighbors; use S2 cells for radius coverage |
| Photo storage at 219 TB/year | S3 with lifecycle policies; generate multiple thumbnail sizes; CDN caching |
| Stale business hours (showing "open" when closed) | Business owners push updates via app; crowdsourced corrections; Google-style "popular times" |
| Review spam and fake ratings | Multi-layer: rate limit → NLP model → behavioral analysis → human review |

### Key Takeaways

- **Geohash** is excellent for sharding and coarse spatial filtering but always search neighboring cells to avoid boundary misses
- **Two-phase search**: geo-filtered candidate retrieval (Elasticsearch) → ML-based re-ranking (Ranking Service)
- **Bayesian average** for ratings prevents gaming: `(C × m + Σ ratings) / (C + n)` where C is a confidence parameter and m is the global mean
- Pre-compute what you can: popularity scores, open/closed bitfields, geohash prefixes — avoid expensive computation at query time
- **Review integrity** requires defense in depth: rate limits, NLP detection, behavioral analysis, and human moderation
- The geospatial index (10 GB for 200M businesses) **fits in memory** — leverage this for sub-millisecond spatial queries

---

## Chapter Summary

The eight systems in this chapter share a common DNA despite their diverse domains:

| Pattern | Where It Appears |
|---|---|
| **Saga / Compensating Transactions** | E-Commerce checkout, Payment processing, Hotel booking |
| **Idempotency Keys** | All systems with financial mutations (payments, orders, bookings) |
| **Optimistic Concurrency Control** | Inventory management, Seat booking, Order book |
| **Event Sourcing / Append-Only Logs** | Payment ledger, Order events, Stock exchange journal |
| **Geospatial Indexing** | Ride-sharing, Proximity service, Hotel search |
| **Stream Processing (Flink/Kafka)** | Ad analytics, Surge pricing, Inventory sync |
| **Virtual Queue / Rate Leveling** | Ticket flash sales, E-commerce promotions |
| **CQRS (Read/Write Separation)** | Search (ES) vs. transactional store (PostgreSQL) in all systems |

The overarching lesson: **in commerce systems, correctness is paramount, but you don't need the same level of consistency everywhere**. Apply strong consistency surgically — at the moment of booking, payment capture, or order matching — and let everything else be eventually consistent. This selective consistency model is the foundation of every scalable commerce platform.
