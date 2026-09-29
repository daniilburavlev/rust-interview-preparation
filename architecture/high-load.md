# High-Load Web Services Architecture: Senior Interview Questions

Questions on designing, scaling and operating services under heavy traffic. Senior answers should name trade-offs, failure modes and how you'd measure.

## Contents

1. [Scalability fundamentals](#1-scalability-fundamentals)
2. [Caching](#2-caching)
3. [Databases and storage](#3-databases-and-storage)
4. [Distributed systems](#4-distributed-systems)
5. [Resilience](#5-resilience)
6. [API design and traffic management](#6-api-design-and-traffic-management)
7. [Observability and performance](#7-observability-and-performance)
8. [System design practice](#8-system-design-practice)

---

## 1. Scalability fundamentals

### 1.1 Vertical vs horizontal scaling?

Vertical: a bigger machine; simple, no code changes, but has a ceiling and a single point of failure. Horizontal: more machines behind a load balancer; near-unlimited and fault tolerant, but requires stateless services, shared or partitioned state, and handling distributed failures.

### 1.2 Why should services be stateless, and where does state go?

Stateless instances can be added, removed and restarted freely, and any instance can serve any request. State moves to databases, caches (Redis), object storage, or message logs (Kafka). Sessions go into signed tokens (JWT) or a shared session store.

### 1.3 What are latency percentiles and why not use averages?

p50/p95/p99/p999 show the distribution; averages hide tail latency. With fan-out (one request calling 100 backends), a p99 on each backend becomes the typical user experience. SLOs should be defined on percentiles.

### 1.4 What is Little's Law and how do you use it?

`L = λ × W`: concurrent requests = arrival rate × latency. At 10k RPS and 50 ms latency you have ~500 requests in flight, which sizes connection pools, thread pools and concurrency limits. If latency rises, in-flight work (and memory) rises with it.

### 1.5 L4 vs L7 load balancing? Which algorithms?

L4 (TCP/UDP, e.g. AWS NLB) is fast and protocol-agnostic. L7 (HTTP, e.g. ALB, Envoy, Nginx) can route by path/header, terminate TLS, retry and balance per request (important for HTTP/2 and gRPC, where one connection carries many requests). Algorithms: round robin, least connections/requests, consistent hashing (for cache affinity), and "power of two random choices".

---

## 2. Caching

### 2.1 Cache-aside vs read-through vs write-through vs write-behind?

- **Cache-aside**: the app reads the cache, on miss reads the DB and populates the cache. Most common.
- **Read-through**: the cache itself loads from the DB on miss.
- **Write-through**: writes go to cache and DB synchronously; consistent, slower writes.
- **Write-behind**: writes go to cache and are flushed to the DB asynchronously; fast, risk of loss.

### 2.2 How do you handle cache invalidation?

TTLs as a safety net, explicit deletes on write (delete rather than update, to avoid races), versioned keys, and change-data-capture (Debezium → Kafka → invalidator) for cross-service invalidation. Accept bounded staleness where the business allows.

### 2.3 What is a cache stampede and how do you prevent it?

When a hot key expires, many requests hit the DB at once. Mitigations: request coalescing / single-flight (one loader per key), locks with stale-while-revalidate, probabilistic early expiration, and TTL jitter so keys don't expire together.

### 2.4 What are the layers of caching in a web system?

Browser/HTTP cache (`Cache-Control`, `ETag`), CDN, reverse proxy, in-process cache (e.g. `moka` in Rust), distributed cache (Redis/Memcached), and database buffer pool. Each layer trades freshness for latency and load.

### 2.5 How do you handle hot keys in Redis?

Local in-process caching of the hot key, replicating the key under several names (`key#1..N`) and picking randomly, read replicas, and client-side caching (Redis 6 tracking). Detect with `redis-cli --hotkeys` or proxy metrics.

---

## 3. Databases and storage

### 3.1 How do you scale a relational database?

In order: query and index optimization, connection pooling (PgBouncer), caching, read replicas (with replication lag in mind), vertical scaling, partitioning tables, functional splitting (separate DBs per domain), and finally sharding.

### 3.2 What is sharding and how do you choose a shard key?

Splitting data across independent databases by key. A good key has high cardinality, spreads load evenly, and keeps most queries within one shard (e.g. `tenant_id`). Strategies: range (good for scans, risks hotspots), hash (even spread, bad for ranges), directory/lookup. Consistent hashing reduces data movement when adding nodes.

### 3.3 Explain transaction isolation levels and anomalies.

Read uncommitted, read committed (no dirty reads), repeatable read (no non-repeatable reads; in Postgres also no phantoms), serializable (no write skew). Higher isolation means more aborts or locking. Senior answer: know your DB's default (Postgres: read committed; MySQL InnoDB: repeatable read) and use `SELECT ... FOR UPDATE` or optimistic versioning where needed.

### 3.4 Optimistic vs pessimistic locking?

Pessimistic: lock the row (`FOR UPDATE`) before modifying; safe under high contention, but blocks. Optimistic: store a `version` column and `UPDATE ... WHERE version = ?`; retry on conflict. Better under low contention and across long user interactions.

### 3.5 SQL vs NoSQL: how do you choose?

SQL: strong consistency, joins, transactions, flexible queries. NoSQL types solve specific problems: key-value (DynamoDB, Redis) for simple access at massive scale, document (MongoDB), wide-column (Cassandra, ScyllaDB) for high write throughput, search (Elasticsearch/OpenSearch), time-series. Choose by access patterns, consistency needs and operational cost, not hype.

### 3.6 What is an N+1 query problem?

Loading a list then issuing one query per item. Fix with joins, `WHERE id = ANY($1)` batch loading, or a DataLoader pattern. It's a common cause of latency spikes under load.

---

## 4. Distributed systems

### 4.1 Explain the CAP theorem and PACELC.

CAP: during a network **P**artition, choose **C**onsistency or **A**vailability. PACELC adds: **E**lse (no partition), choose **L**atency or **C**onsistency. E.g. DynamoDB/Cassandra are PA/EL (tunable), Spanner is PC/EC.

### 4.2 Strong vs eventual consistency? What is read-your-writes?

Strong: every read sees the latest write. Eventual: replicas converge over time. Useful middle grounds: read-your-writes (route a user's reads to the leader after their write), monotonic reads, and causal consistency.

### 4.3 How do you keep data consistent across microservices without distributed transactions?

The **Saga** pattern: a sequence of local transactions with compensating actions on failure, coordinated by orchestration (central coordinator) or choreography (events). Combine with the **transactional outbox** to publish events reliably.

### 4.4 What is the transactional outbox pattern?

Write the business change and an event row to an `outbox` table in the same DB transaction. A relay (poller or CDC like Debezium) publishes outbox rows to Kafka. This avoids the dual-write problem where the DB commit succeeds but the publish fails (or vice versa).

### 4.5 What is idempotency and how do you implement it?

An operation that can be applied multiple times with the same effect. Needed because retries and at-least-once delivery cause duplicates. Implement with client-provided idempotency keys stored with the result, unique constraints, upserts, and deduplication tables keyed by message ID.

### 4.6 What is CQRS and event sourcing? When are they worth it?

CQRS separates write models from read models (often denormalized projections). Event sourcing stores state as an append-only log of events and rebuilds state by replay. They give audit trails and scalable reads but add complexity and eventual consistency; use them for domains that really need them (finance, ledgers), not by default.

### 4.7 How does leader election / consensus work at a high level?

Raft (etcd, Consul) and Paxos: nodes elect a leader by majority vote with terms; the leader replicates a log, and entries commit once a majority acknowledges. Requires `2f+1` nodes to tolerate `f` failures. Use existing systems rather than implementing it yourself.

---

## 5. Resilience

### 5.1 How do you configure timeouts and retries correctly?

Every network call has a timeout, and deadlines propagate downstream (the remaining budget shrinks). Retry only idempotent operations, with exponential backoff and jitter, a small max attempts, and a retry budget (e.g. retries ≤ 10% of traffic) to avoid retry storms that amplify outages.

### 5.2 What is a circuit breaker?

It tracks failures to a dependency; after a threshold it "opens" and fails fast instead of calling the failing service, then "half-opens" to probe recovery. Prevents cascading failures and gives the dependency room to recover.

### 5.3 What is the bulkhead pattern?

Isolating resources (separate connection pools, thread pools, or semaphores per dependency or tenant) so one slow dependency can't exhaust everything.

### 5.4 Load shedding vs rate limiting vs backpressure?

- **Rate limiting**: enforce a per-client quota (token bucket, sliding window), returning 429.
- **Load shedding**: drop excess work when the server itself is overloaded (e.g. by queue time or concurrency), returning 503, preferably dropping low-priority traffic first.
- **Backpressure**: signal upstream to slow down (bounded queues, TCP flow control, Kafka consumer lag).

### 5.5 How do you implement a distributed rate limiter?

Token bucket or sliding window counters in Redis using atomic Lua scripts or `INCR` + `EXPIRE`, or at the gateway (Envoy, API Gateway). Trade-offs: extra network hop per request vs local approximate limiting with periodic sync.

### 5.6 What is graceful degradation?

Serving a reduced but useful experience when dependencies fail: stale cache data, default recommendations, disabling non-critical features via feature flags, instead of returning errors.

---

## 6. API design and traffic management

### 6.1 REST vs gRPC vs GraphQL?

REST: simple, cacheable, universal. gRPC: HTTP/2, binary protobuf, streaming and strong contracts; great for internal service-to-service. GraphQL: clients choose fields, reducing over/under-fetching; harder to cache and to protect against expensive queries.

### 6.2 How do you paginate large datasets efficiently?

Prefer cursor/keyset pagination (`WHERE id > $last ORDER BY id LIMIT n`), which is stable and O(log n) per page. Offset pagination gets slower with depth and shifts when data changes.

### 6.3 How do you version and evolve APIs without breaking clients?

Additive changes only (new optional fields), tolerant readers, versioning in URL or header for breaking changes, deprecation periods with telemetry on old version usage, and schema registries/protobuf field numbering rules for event and RPC contracts.

### 6.4 What does an API gateway or service mesh give you?

Gateway (edge): auth, rate limiting, routing, TLS termination, request transformation. Service mesh (Istio, Linkerd): mTLS, retries, timeouts, traffic splitting and telemetry between services via sidecars or eBPF, without code changes. Cost: latency and operational complexity.

### 6.5 How do you handle WebSocket or long-lived connections at scale?

Sticky routing or connection-aware load balancing, a pub/sub backbone (Redis, NATS, Kafka) to fan out messages to the node holding the connection, heartbeats, reconnect with backoff and jitter, and graceful draining on deploys. Watch file descriptor limits and memory per connection.

---

## 7. Observability and performance

### 7.1 What are the three pillars of observability?

Metrics (aggregated, cheap, for alerting), logs (detailed events, structured JSON), and traces (request flow across services). Tie them together with trace IDs. OpenTelemetry is the standard for instrumentation.

### 7.2 RED vs USE methods?

RED for services: **R**ate, **E**rrors, **D**uration. USE for resources: **U**tilization, **S**aturation, **E**rrors (CPU, memory, disks, connection pools).

### 7.3 What are SLI, SLO, SLA and error budgets?

SLI: a measured indicator (e.g. % of requests < 300 ms). SLO: the target (99.9% over 30 days). SLA: a contractual commitment with penalties. Error budget: `1 - SLO`; when it's exhausted, prioritize reliability over features.

### 7.4 How would you investigate a sudden latency increase in production?

Check what changed (deploys, config, traffic), compare dashboards (RED per endpoint, dependencies' latency, saturation of CPU/DB/pools), look at traces for slow spans, check GC/allocations or lock contention via profiling, and roll back if a deploy correlates. Then do a blameless postmortem.

### 7.5 How do you load test a service?

Define the target (RPS, latency SLO), use realistic traffic mixes and data, tools like k6, Gatling, wrk2 or vegeta (open-model generators avoid coordinated omission), ramp up to find the knee, and watch saturation metrics. Test in a production-like environment and include dependencies.

---

## 8. System design practice

Common prompts to practice end to end (requirements, estimates, API, data model, architecture, bottlenecks, failure modes):

- Design a URL shortener (hashing vs counters, redirect latency, cache, analytics via Kafka).
- Design a rate limiter as a service.
- Design a news feed (fan-out on write vs on read, celebrity problem).
- Design a chat system (WebSockets, message ordering, delivery receipts, storage).
- Design a payment system (idempotency, ledger, sagas, reconciliation).
- Design a notification service (priorities, retries, per-channel rate limits, deduplication).
- Design a metrics/logging pipeline (Kafka ingestion, time-series storage, downsampling).
- Design a distributed job scheduler (leases, exactly-once execution semantics, retries).

### 8.1 How do you do back-of-the-envelope estimates?

Start from users and actions: e.g. 100M DAU × 10 requests/day ≈ 1B/day ≈ 12k RPS average, ×3–5 for peak. Storage = objects/day × size × retention. Know rough numbers: memory ~100 ns, SSD read ~100 µs, same-region network round trip ~0.5 ms, cross-continent ~100 ms.
