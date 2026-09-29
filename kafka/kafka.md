# Apache Kafka: Senior Interview Questions

Core concepts, delivery guarantees, performance tuning and operations. Senior answers should explain *how* Kafka provides a guarantee and what it costs.

## Contents

1. [Core concepts](#1-core-concepts)
2. [Producers](#2-producers)
3. [Consumers](#3-consumers)
4. [Delivery semantics and transactions](#4-delivery-semantics-and-transactions)
5. [Replication and durability](#5-replication-and-durability)
6. [Performance and operations](#6-performance-and-operations)
7. [Design patterns and ecosystem](#7-design-patterns-and-ecosystem)
8. [Kafka with Rust](#8-kafka-with-rust)

---

## 1. Core concepts

### 1.1 What is Kafka and how is it different from a traditional message queue?

A distributed, partitioned, replicated **commit log**. Messages are not deleted on consumption; they're retained by time or size, and each consumer group tracks its own offset. This enables replay, multiple independent consumers, and very high throughput via sequential disk I/O. Traditional queues (RabbitMQ, SQS) delete messages once acknowledged and focus on per-message routing.

### 1.2 Explain topics, partitions, offsets and brokers.

A topic is a named stream split into partitions. Each partition is an ordered, append-only log where every record has a monotonically increasing offset. Partitions are distributed across brokers (servers). Partitions are the unit of parallelism, ordering and replication.

### 1.3 What ordering guarantees does Kafka give?

Order is guaranteed **only within a partition**. Records with the same key go to the same partition (default partitioner hashes the key), so per-key ordering holds, as long as the partition count doesn't change and retries don't reorder (see idempotent producer).

### 1.4 How do you choose the number of partitions?

Based on target throughput / per-partition throughput (for both producers and consumers) and the maximum consumer parallelism you need. More partitions give more parallelism but increase metadata, open files, rebalance and failover time. Increasing partitions later breaks key → partition mapping, so plan ahead; decreasing isn't possible.

### 1.5 What is ZooKeeper's role, and what is KRaft?

Older Kafka used ZooKeeper for metadata and controller election. KRaft (Kafka Raft) replaces it with an internal Raft quorum of controller nodes, simplifying operations and scaling to more partitions. ZooKeeper mode was removed in Kafka 4.0.

---

## 2. Producers

### 2.1 What does `acks` control?

- `acks=0`: fire and forget, possible loss.
- `acks=1`: the leader wrote it; loss if the leader dies before followers replicate.
- `acks=all` (`-1`): all in-sync replicas have it; combined with `min.insync.replicas=2` and replication factor 3, this is the durable setup. Default since Kafka 3.0.

### 2.2 What is the idempotent producer?

With `enable.idempotence=true` (default since 3.0), the producer gets a producer ID and sequence numbers per partition; the broker discards duplicates caused by retries and preserves order with up to 5 in-flight requests. It gives exactly-once writes **per partition per producer session**.

### 2.3 How do `batch.size`, `linger.ms` and compression affect throughput?

The producer batches records per partition. `linger.ms` waits a bit to fill batches; `batch.size` caps the batch in bytes. Bigger batches plus compression (`lz4`, `zstd`) improve throughput and reduce network/disk usage at the cost of a little latency.

### 2.4 How does the partitioner work, and what are the pitfalls?

With a key: `murmur2(key) % partitions`. Without a key: the sticky partitioner fills a batch for one partition before switching. Pitfalls: skewed keys create hot partitions; changing partition count remaps keys; custom partitioners must be consistent across all producers (including non-Java clients, which may use different hash functions by default).

---

## 3. Consumers

### 3.1 What is a consumer group?

Consumers sharing a `group.id` split a topic's partitions: each partition is consumed by exactly one member of the group. Different groups consume independently. Consumers beyond the partition count sit idle.

### 3.2 What is a rebalance and why is it a problem?

Reassigning partitions when members join, leave, or time out. With the classic eager protocol all consumers stop processing. Mitigations: cooperative sticky assignor (incremental rebalancing), static membership (`group.instance.id`) to survive restarts, tuning `session.timeout.ms` and `max.poll.interval.ms`, and the new consumer rebalance protocol (KIP-848) with broker-side assignment.

### 3.3 What causes "consumer left group" / `max.poll.interval.ms` exceeded?

Processing a batch took longer than `max.poll.interval.ms` between `poll()` calls, so the coordinator considered the consumer dead and rebalanced. Fix: process faster, reduce `max.poll.records`, move slow work off the poll loop (with care for offsets), or increase the interval.

### 3.4 How do offset commits work? Auto vs manual?

Committed offsets are stored in the internal `__consumer_offsets` topic, and mean "next offset to read". Auto commit (every `auto.commit.interval.ms`) can commit before processing is done (loss) or after reprocessing (duplicates). Manual commit after processing gives at-least-once. Commit sync for safety, async for throughput, and sync on shutdown/rebalance.

### 3.5 What is consumer lag and how do you monitor it?

`log end offset - committed offset` per partition. Rising lag means consumers can't keep up. Monitor with Burrow, kafka-lag-exporter, or broker metrics, and alert on lag growth or time-based lag, not raw numbers alone. Scale consumers up to the partition count, or optimize processing.

### 3.6 How do you handle a poison message?

Retry a limited number of times (possibly via retry topics with delays), then send it to a dead-letter topic (DLT) with error metadata and continue. Never block the partition forever. Alert on DLT volume and provide a replay tool.

---

## 4. Delivery semantics and transactions

### 4.1 At-most-once, at-least-once, exactly-once: how does each happen?

- At-most-once: commit offsets before processing; crashes lose messages.
- At-least-once: process then commit; crashes cause reprocessing (duplicates). The common default; pair with idempotent consumers.
- Exactly-once: Kafka transactions for read-process-write within Kafka, or idempotent/transactional sinks for external systems.

### 4.2 How do Kafka transactions work?

A producer with a `transactional.id` begins a transaction, writes to multiple partitions, and includes consumer offsets via `sendOffsetsToTransaction`, then commits. A transaction coordinator writes commit/abort markers. Consumers with `isolation.level=read_committed` see only committed data. The `transactional.id` fences zombie producers (older instances with a lower epoch).

### 4.3 Does exactly-once apply when writing to a database?

Not automatically. Kafka EOS covers Kafka-to-Kafka. For external sinks, make writes idempotent (upsert by key, dedup table by message ID) or store the consumer offset in the same DB transaction as the result and seek to it on startup.

---

## 5. Replication and durability

### 5.1 What are leaders, followers and ISR?

Each partition has one leader handling reads/writes and followers replicating it. The in-sync replica set (ISR) contains replicas caught up within `replica.lag.time.max.ms`. With `acks=all`, a write is committed once all ISR members have it.

### 5.2 What does `min.insync.replicas` do?

The minimum ISR size for `acks=all` writes to succeed. With RF=3 and `min.insync.replicas=2`, you tolerate one broker down without losing writes or availability; if two are down, producers get `NotEnoughReplicas` errors instead of silently losing durability.

### 5.3 What is unclean leader election?

Allowing an out-of-sync replica to become leader when no ISR member is available. It restores availability but loses committed data. Disabled by default (`unclean.leader.election.enable=false`): a CP-over-AP choice.

### 5.4 What is the high watermark?

The offset up to which all ISR replicas have replicated. Consumers can only read up to the high watermark, so they never see data that might be lost on leader failover.

### 5.5 How does Kafka achieve durability without fsync on every write?

It relies on replication across brokers (ideally racks/AZs) rather than per-message fsync; data is written to the OS page cache and flushed by the OS. Durability comes from multiple replicas acknowledging.

---

## 6. Performance and operations

### 6.1 Why is Kafka so fast?

Sequential append-only disk writes, heavy use of the OS page cache, zero-copy transfer (`sendfile`) from disk to socket, batching and compression end to end, and partition-level parallelism.

### 6.2 What are retention and log compaction?

Retention deletes old segments by time (`retention.ms`) or size (`retention.bytes`). Compaction (`cleanup.policy=compact`) keeps at least the latest value per key and deletes a key when a tombstone (null value) is written. Compacted topics act as changelogs or state snapshots (e.g. `__consumer_offsets`, KTables).

### 6.3 How would you size and monitor a Kafka cluster?

Size by throughput (MB/s in and out, times replication factor), retention × throughput for disk, and partition count per broker. Key metrics: under-replicated partitions, offline partitions, ISR shrink/expand rate, request latency, network/disk utilization, consumer lag, and controller health.

### 6.4 What happens when a broker fails?

The controller detects it, elects new leaders for its partitions from the ISR, and clients refresh metadata and reconnect. With RF=3 and proper ISR settings, no committed data is lost. When the broker returns, it catches up and preferred leader election rebalances leadership.

### 6.5 How do you migrate or rebalance partitions across brokers?

Use `kafka-reassign-partitions` (or Cruise Control for automated balancing) with replication throttling to avoid saturating the network. Plan for data movement time.

### 6.6 Which security features does Kafka support?

TLS for encryption in transit, SASL (SCRAM, OAUTHBEARER, GSSAPI/Kerberos, AWS IAM on MSK) or mTLS for authentication, and ACLs for authorization per topic/group. Encryption at rest comes from disk/volume encryption.

---

## 7. Design patterns and ecosystem

### 7.1 What is a schema registry and why use it?

A service storing Avro/Protobuf/JSON Schema versions with compatibility rules (backward, forward, full). Producers register schemas and embed a schema ID; consumers fetch the schema. It prevents breaking changes in event contracts across teams.

### 7.2 What are Kafka Connect and Debezium?

Kafka Connect runs source/sink connectors (JDBC, S3, Elasticsearch) without custom code. Debezium is a CDC source connector that streams DB changes from the write-ahead log (Postgres, MySQL), which is the backbone of the outbox pattern.

### 7.3 Kafka Streams / Flink: what problems do they solve?

Stateful stream processing: joins, windowed aggregations, deduplication, with local state stores backed by changelog topics and exactly-once processing. Kafka Streams is a library; Flink is a separate cluster with more advanced features.

### 7.4 How do you design event schemas and topics?

Topic per event type or per aggregate, keyed by entity ID for ordering. Events should be facts in past tense (`OrderPlaced`), carry an event ID, timestamp and schema version, and evolve compatibly. Decide between thin events (IDs only) and fat events (full state) based on coupling and consumer needs.

### 7.5 Kafka vs RabbitMQ vs SQS vs Kinesis?

Kafka: high-throughput log with replay and stream processing. RabbitMQ: flexible routing, per-message acks, priority queues, lower throughput. SQS: fully managed simple queue, no ordering (except FIFO) or replay. Kinesis: AWS-managed log similar to Kafka with shard limits. Choose based on replay, ordering, throughput, and operational budget (MSK is managed Kafka).

---

## 8. Kafka with Rust

### 8.1 Which Rust Kafka clients exist?

`rdkafka` (bindings to librdkafka; mature, full-featured, async via `StreamConsumer`/`FutureProducer`) is the production standard. Pure-Rust alternatives (`rskafka`, `kafka-rust`) exist with fewer features.

### 8.2 How do you build a reliable consumer in Rust with rdkafka?

Disable auto offset store (`enable.auto.offset.store=false`), process each message, then `store_offset` so the background auto-commit only commits processed offsets (at-least-once). Handle rebalance callbacks via a custom `ConsumerContext`, bound concurrency per partition to preserve ordering, and on shutdown commit synchronously before closing.

### 8.3 How do you process messages concurrently without breaking ordering?

Parallelize across partitions (or across keys within a partition via hashing to worker queues), but keep sequential processing per key. Track the lowest fully processed offset per partition before committing, since out-of-order completion must not commit past unfinished messages.
