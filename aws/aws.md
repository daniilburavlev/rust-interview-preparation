# AWS: Senior Interview Questions

AWS services and architecture decisions for backend engineers running high-load services. Senior answers weigh cost, limits and failure domains, not just service names.

## Contents

1. [Global infrastructure and networking](#1-global-infrastructure-and-networking)
2. [Identity and security](#2-identity-and-security)
3. [Compute](#3-compute)
4. [Storage and databases](#4-storage-and-databases)
5. [Messaging and integration](#5-messaging-and-integration)
6. [Load balancing, DNS and edge](#6-load-balancing-dns-and-edge)
7. [Observability](#7-observability)
8. [Reliability, scaling and cost](#8-reliability-scaling-and-cost)
9. [Infrastructure as code](#9-infrastructure-as-code)

---

## 1. Global infrastructure and networking

### 1.1 Regions, Availability Zones and edge locations?

A Region is an independent geographic area with multiple AZs. An AZ is one or more isolated data centers with independent power and networking, connected with low-latency links. Edge locations serve CloudFront and Route 53. High availability means spreading across AZs; disaster recovery often means across Regions.

### 1.2 Describe a typical VPC design for a web service.

A VPC per environment with public subnets (load balancers, NAT gateways) and private subnets (application, databases) in at least three AZs. Private instances reach the internet through NAT gateways (one per AZ to avoid cross-AZ dependency) and AWS services through VPC endpoints.

### 1.3 Security groups vs NACLs?

Security groups are stateful, attached to ENIs, allow-only, and can reference other security groups. NACLs are stateless, subnet-level, support allow and deny, and are evaluated in rule order. Security groups are the primary tool; NACLs are a coarse extra layer.

### 1.4 How do you connect VPCs and on-prem networks?

VPC peering (simple, non-transitive), Transit Gateway (hub-and-spoke, transitive, scales to many VPCs), PrivateLink (expose a single service privately), and Site-to-Site VPN or Direct Connect for on-prem.

### 1.5 Gateway vs interface VPC endpoints? Why use them?

Gateway endpoints (S3, DynamoDB) are free route-table entries. Interface endpoints (most other services) are ENIs with hourly cost. They keep traffic off the internet and avoid NAT gateway data processing charges, which are a common surprise cost.

---

## 2. Identity and security

### 2.1 IAM users vs roles vs policies?

Users are long-term identities (avoid for workloads). Roles are assumed to get temporary credentials via STS; workloads should always use roles (instance profiles, ECS task roles, EKS IRSA/Pod Identity, Lambda execution roles). Policies are JSON documents granting permissions; follow least privilege.

### 2.2 How is an IAM request evaluated?

Explicit deny wins; then there must be an allow from identity or resource policies, and it must not be blocked by SCPs (Organizations), permission boundaries, or session policies. Default is implicit deny.

### 2.3 How do you give CI/CD pipelines access to AWS without long-lived keys?

OIDC federation: GitHub Actions (or GitLab) obtains an OIDC token, and an IAM role trusts that identity provider with conditions on repository and branch. The job assumes the role with `sts:AssumeRoleWithWebIdentity` and gets short-lived credentials.

### 2.4 How do you manage secrets on AWS?

Secrets Manager (rotation, cross-account, cost per secret) or SSM Parameter Store SecureString (cheaper, no native rotation). Encrypt with KMS; grant access via the workload's role; cache secrets in the app to avoid throttling.

### 2.5 What is KMS envelope encryption?

KMS generates a data key; data is encrypted locally with the data key, and the data key itself is encrypted with a KMS key and stored alongside the data. This avoids sending large payloads to KMS and limits KMS calls.

### 2.6 How do you structure AWS accounts?

AWS Organizations with separate accounts per environment and workload (prod, staging, security, logging, shared services), Control Tower or landing zone, SCPs as guardrails, and IAM Identity Center (SSO) for humans. Accounts are the strongest isolation boundary.

---

## 3. Compute

### 3.1 EC2 vs ECS vs EKS vs Lambda vs Fargate: how do you choose?

- **EC2**: full control, cheapest at steady load with Savings Plans, most ops work.
- **ECS**: simpler AWS-native container orchestration.
- **EKS**: Kubernetes, portable ecosystem, higher complexity and cost.
- **Fargate**: serverless compute for ECS/EKS tasks; no node management, higher unit price.
- **Lambda**: event-driven functions, scale to zero, per-ms billing, 15-minute max duration, cold starts.

### 3.2 How does Rust perform on Lambda?

Very well: tiny binaries and fast startup give some of the lowest cold starts, and low memory usage reduces cost. Use the `lambda_runtime`/`lambda_http` crates, the `provided.al2023` runtime, `cargo lambda` for builds, and arm64 (Graviton) for better price/performance.

### 3.3 What are Lambda cold starts and how do you mitigate them?

Initialization of a new execution environment (download code, start runtime, run init code). Mitigate with small packages, fast runtimes, lazy init, provisioned concurrency, and SnapStart (for supported runtimes).

### 3.4 What is Lambda concurrency and why can it hurt?

Each concurrent request uses one execution environment. Account-level concurrency limits are shared by all functions, so one function can starve others; use reserved concurrency. Unbounded Lambda concurrency can also overwhelm databases (use RDS Proxy).

### 3.5 Auto Scaling Groups: which scaling policies exist?

Target tracking (e.g. keep CPU at 50%), step scaling, scheduled scaling, and predictive scaling. Use health checks from the load balancer, warm pools for slow-starting instances, and mixed instance policies with Spot.

### 3.6 When should you use Spot instances?

For fault-tolerant, stateless or batch workloads: they're up to ~90% cheaper but can be reclaimed with a 2-minute notice. Diversify instance types and AZs, handle interruption notices by draining, and keep a baseline on On-Demand.

### 3.7 Why consider Graviton (arm64)?

Typically better price-performance than x86 for many workloads. Rust cross-compiles easily; build multi-arch images and benchmark.

---

## 4. Storage and databases

### 4.1 S3 consistency, storage classes and performance?

S3 has strong read-after-write consistency for all operations. Storage classes: Standard, Intelligent-Tiering, Standard-IA, One Zone-IA, Glacier tiers; use lifecycle policies. Performance scales per prefix (thousands of requests per second per prefix); use multipart uploads for large objects and byte-range fetches for parallel downloads.

### 4.2 RDS vs Aurora?

RDS is managed Postgres/MySQL on EBS with standby replicas. Aurora separates compute from a distributed storage layer replicated six ways across three AZs, with faster failover, up to 15 low-lag read replicas, Aurora Serverless v2 and Global Database. Aurora costs more, especially for I/O-heavy workloads (consider I/O-Optimized).

### 4.3 How does DynamoDB partitioning work and how do you design keys?

Items are distributed by partition key hash; each partition has throughput limits, so keys must spread load evenly. Design from access patterns: partition key + sort key for queries, GSIs for alternate access patterns, single-table design to fetch related items in one query. Avoid hot keys (write sharding by suffix).

### 4.4 DynamoDB on-demand vs provisioned capacity? What are its consistency options?

On-demand: pay per request, handles spiky load. Provisioned (with auto scaling): cheaper for predictable load. Reads are eventually consistent by default; strongly consistent reads cost double and aren't available on GSIs. Transactions (`TransactWriteItems`) and conditional writes support optimistic locking. DynamoDB Streams provide CDC.

### 4.5 ElastiCache: Redis/Valkey vs Memcached?

Redis/Valkey: data structures, persistence, replication, cluster mode sharding, pub/sub, Lua. Memcached: simple, multithreaded key-value cache with no persistence or replication. Redis/Valkey is the usual default.

### 4.6 EBS vs EFS vs instance store?

EBS: block storage attached to one instance in one AZ (gp3 lets you tune IOPS/throughput independently). EFS: managed NFS shared across instances and AZs. Instance store: ephemeral local NVMe, very fast, data lost on stop.

---

## 5. Messaging and integration

### 5.1 SQS standard vs FIFO?

Standard: nearly unlimited throughput, at-least-once delivery, best-effort ordering. FIFO: exactly-once processing within a deduplication window and ordering per message group ID, with lower throughput limits (higher with high-throughput mode).

### 5.2 What is the SQS visibility timeout and what goes wrong with it?

After a consumer receives a message, it's hidden for the visibility timeout; if not deleted in time, it reappears and gets processed again. Set it longer than processing time, extend it for long jobs (`ChangeMessageVisibility`), and use a DLQ via redrive policy with `maxReceiveCount`.

### 5.3 SNS vs SQS vs EventBridge vs Kinesis vs MSK?

- **SNS**: pub/sub push fan-out.
- **SQS**: queue for decoupling and load leveling; common pattern SNS → multiple SQS queues.
- **EventBridge**: event bus with content-based routing, schema registry, SaaS integrations.
- **Kinesis Data Streams**: ordered, replayable shard-based streaming.
- **MSK**: managed Kafka, for Kafka APIs and ecosystem.

### 5.4 What is Step Functions used for?

Orchestrating workflows (sagas, long-running processes) with retries, timeouts, branching and compensation defined declaratively, keeping state outside your services.

---

## 6. Load balancing, DNS and edge

### 6.1 ALB vs NLB vs GWLB?

ALB: L7 HTTP/HTTPS/gRPC, path and host routing, WAF integration, target groups for ECS/EKS/Lambda. NLB: L4 TCP/UDP/TLS, very high throughput, static IPs, preserves source IP. GWLB: for inserting third-party network appliances.

### 6.2 What does connection draining (deregistration delay) do?

When a target is deregistered, the load balancer stops sending new requests but lets in-flight ones finish for the configured delay. Align it with your app's graceful shutdown timeout.

### 6.3 Which Route 53 routing policies exist?

Simple, weighted (canary, blue/green), latency-based, failover (with health checks), geolocation, geoproximity, and multivalue answer. Alias records point to AWS resources at the zone apex for free.

### 6.4 What does CloudFront provide beyond caching?

TLS termination at the edge, origin shielding, WAF and Shield (DDoS) integration, signed URLs/cookies, Lambda@Edge and CloudFront Functions, and reduced load and egress costs on origins.

### 6.5 What is API Gateway and when would you use it over an ALB?

A managed API front door with auth (Cognito, JWT, IAM, Lambda authorizers), throttling, usage plans, request validation and WebSocket APIs. ALB is cheaper at high sustained throughput; API Gateway suits Lambda backends and APIs needing per-client quotas.

---

## 7. Observability

### 7.1 How do you monitor a service on AWS?

CloudWatch metrics and alarms (including custom metrics and Embedded Metric Format from logs), CloudWatch Logs with Logs Insights queries, X-Ray or OpenTelemetry (ADOT) for tracing, CloudTrail for API audit, and often Prometheus/Grafana (Amazon Managed Service for Prometheus/Grafana) or third-party tools.

### 7.2 What is CloudTrail vs CloudWatch vs AWS Config?

CloudTrail records API calls (who did what). CloudWatch collects metrics, logs and alarms (how the system behaves). Config records resource configuration history and evaluates compliance rules.

---

## 8. Reliability, scaling and cost

### 8.1 What are the six pillars of the Well-Architected Framework?

Operational excellence, security, reliability, performance efficiency, cost optimization, and sustainability.

### 8.2 Describe disaster recovery strategies.

From cheapest/slowest to most expensive/fastest: backup and restore, pilot light (core data replicated, minimal infra), warm standby (scaled-down copy running), and multi-site active-active. Choose based on RTO (time to recover) and RPO (acceptable data loss).

### 8.3 How would you design a multi-region active-active service?

Latency or geo routing via Route 53 or Global Accelerator, stateless services in each region, multi-region data (DynamoDB Global Tables, Aurora Global Database with write forwarding or a single writer region), conflict resolution strategy (last writer wins or region ownership of data), and regular failover tests.

### 8.4 What are common AWS service limits and throttling issues?

API rate limits (throttling on KMS, Secrets Manager, DynamoDB partitions), Lambda concurrency, ENI/IP exhaustion in subnets (especially with EKS VPC CNI), NAT gateway bandwidth, and per-account quotas. Use Service Quotas, retries with backoff and jitter (the AWS SDKs do this by default), and plan subnet sizes.

### 8.5 How do you optimize AWS costs?

Right-size instances, use Savings Plans/Reserved Instances for baseline and Spot for flexible load, Graviton, S3 lifecycle and Intelligent-Tiering, reduce data transfer (VPC endpoints, avoid cross-AZ chatter, CloudFront), turn off idle non-prod resources, tag everything and review Cost Explorer and anomaly detection.

### 8.6 Which data transfer costs catch teams by surprise?

Cross-AZ traffic (charged both ways), NAT gateway processing per GB, internet egress, cross-region replication, and chatty microservices or Kafka replication across AZs.

---

## 9. Infrastructure as code

### 9.1 Terraform vs CloudFormation vs CDK?

CloudFormation: AWS-native, managed state, drift detection. CDK: define CloudFormation in TypeScript/Python/etc. with higher-level constructs. Terraform/OpenTofu: multi-cloud, huge provider ecosystem, explicit state file (store in S3 with locking), and `plan` shows diffs clearly. Choose one and apply it consistently.

### 9.2 How do you manage Terraform state and environments safely?

Remote state in S3 with locking (DynamoDB or S3 native locking), separate state per environment and component to limit blast radius, modules for reuse, `plan` in pull requests and `apply` from CI only after approval, and drift detection jobs.
