# High-Load Web Services Architecture: Questions Only

Questions without answers, for self-testing. Model answers are in [high-load.md](high-load.md).

## 1. Scalability fundamentals

- 1.1 Vertical vs horizontal scaling?
- 1.2 Why should services be stateless, and where does state go?
- 1.3 What are latency percentiles and why not use averages?
- 1.4 What is Little's Law and how do you use it?
- 1.5 L4 vs L7 load balancing? Which algorithms?

## 2. Caching

- 2.1 Cache-aside vs read-through vs write-through vs write-behind?
- 2.2 How do you handle cache invalidation?
- 2.3 What is a cache stampede and how do you prevent it?
- 2.4 What are the layers of caching in a web system?
- 2.5 How do you handle hot keys in Redis?

## 3. Databases and storage

- 3.1 How do you scale a relational database?
- 3.2 What is sharding and how do you choose a shard key?
- 3.3 Explain transaction isolation levels and anomalies.
- 3.4 Optimistic vs pessimistic locking?
- 3.5 SQL vs NoSQL: how do you choose?
- 3.6 What is an N+1 query problem?

## 4. Distributed systems

- 4.1 Explain the CAP theorem and PACELC.
- 4.2 Strong vs eventual consistency? What is read-your-writes?
- 4.3 How do you keep data consistent across microservices without distributed transactions?
- 4.4 What is the transactional outbox pattern?
- 4.5 What is idempotency and how do you implement it?
- 4.6 What is CQRS and event sourcing? When are they worth it?
- 4.7 How does leader election / consensus work at a high level?

## 5. Resilience

- 5.1 How do you configure timeouts and retries correctly?
- 5.2 What is a circuit breaker?
- 5.3 What is the bulkhead pattern?
- 5.4 Load shedding vs rate limiting vs backpressure?
- 5.5 How do you implement a distributed rate limiter?
- 5.6 What is graceful degradation?

## 6. API design and traffic management

- 6.1 REST vs gRPC vs GraphQL?
- 6.2 How do you paginate large datasets efficiently?
- 6.3 How do you version and evolve APIs without breaking clients?
- 6.4 What does an API gateway or service mesh give you?
- 6.5 How do you handle WebSocket or long-lived connections at scale?

## 7. Observability and performance

- 7.1 What are the three pillars of observability?
- 7.2 RED vs USE methods?
- 7.3 What are SLI, SLO, SLA and error budgets?
- 7.4 How would you investigate a sudden latency increase in production?
- 7.5 How do you load test a service?

## 8. System design practice

- Design a URL shortener (hashing vs counters, redirect latency, cache, analytics via Kafka).
- Design a rate limiter as a service.
- Design a news feed (fan-out on write vs on read, celebrity problem).
- Design a chat system (WebSockets, message ordering, delivery receipts, storage).
- Design a payment system (idempotency, ledger, sagas, reconciliation).
- Design a notification service (priorities, retries, per-channel rate limits, deduplication).
- Design a metrics/logging pipeline (Kafka ingestion, time-series storage, downsampling).
- Design a distributed job scheduler (leases, exactly-once execution semantics, retries).
- 8.1 How do you do back-of-the-envelope estimates?
