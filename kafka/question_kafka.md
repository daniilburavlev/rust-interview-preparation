# Apache Kafka: Questions Only

Questions without answers, for self-testing. Model answers are in [kafka.md](kafka.md).

## 1. Core concepts

- 1.1 What is Kafka and how is it different from a traditional message queue?
- 1.2 Explain topics, partitions, offsets and brokers.
- 1.3 What ordering guarantees does Kafka give?
- 1.4 How do you choose the number of partitions?
- 1.5 What is ZooKeeper's role, and what is KRaft?

## 2. Producers

- 2.1 What does `acks` control?
- 2.2 What is the idempotent producer?
- 2.3 How do `batch.size`, `linger.ms` and compression affect throughput?
- 2.4 How does the partitioner work, and what are the pitfalls?

## 3. Consumers

- 3.1 What is a consumer group?
- 3.2 What is a rebalance and why is it a problem?
- 3.3 What causes "consumer left group" / `max.poll.interval.ms` exceeded?
- 3.4 How do offset commits work? Auto vs manual?
- 3.5 What is consumer lag and how do you monitor it?
- 3.6 How do you handle a poison message?

## 4. Delivery semantics and transactions

- 4.1 At-most-once, at-least-once, exactly-once: how does each happen?
- 4.2 How do Kafka transactions work?
- 4.3 Does exactly-once apply when writing to a database?

## 5. Replication and durability

- 5.1 What are leaders, followers and ISR?
- 5.2 What does `min.insync.replicas` do?
- 5.3 What is unclean leader election?
- 5.4 What is the high watermark?
- 5.5 How does Kafka achieve durability without fsync on every write?

## 6. Performance and operations

- 6.1 Why is Kafka so fast?
- 6.2 What are retention and log compaction?
- 6.3 How would you size and monitor a Kafka cluster?
- 6.4 What happens when a broker fails?
- 6.5 How do you migrate or rebalance partitions across brokers?
- 6.6 Which security features does Kafka support?

## 7. Design patterns and ecosystem

- 7.1 What is a schema registry and why use it?
- 7.2 What are Kafka Connect and Debezium?
- 7.3 Kafka Streams / Flink: what problems do they solve?
- 7.4 How do you design event schemas and topics?
- 7.5 Kafka vs RabbitMQ vs SQS vs Kinesis?

## 8. Kafka with Rust

- 8.1 Which Rust Kafka clients exist?
- 8.2 How do you build a reliable consumer in Rust with rdkafka?
- 8.3 How do you process messages concurrently without breaking ordering?
