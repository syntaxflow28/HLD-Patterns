# Cheatsheet — Technology Picker

One page. Find the requirement, take the default, know the alternative.

---

## Databases

| Requirement | Default | Alternative | Deciding factor |
|---|---|---|---|
| General OLTP, joins, transactions | **PostgreSQL** | MySQL | Team familiarity |
| Massive writes, time-ordered, no joins | **Cassandra / ScyllaDB** | HBase | Multi-DC needs |
| Managed KV, predictable p99 | **DynamoDB** | Cassandra | Ops budget vs cost |
| Flexible/nested documents | MongoDB | Postgres JSONB | Do you also need transactions? |
| ACID + horizontal scale | **CockroachDB / Spanner / Vitess** | Sharded Postgres | Cost vs complexity |
| Sub-ms reads, counters, structures | **Redis** | Memcached | Need data types/persistence? |
| Full-text search, facets | **Elasticsearch** | Postgres FTS | Scale + relevance needs |
| Analytics over billions of rows | **ClickHouse / BigQuery** | Snowflake, Redshift | Real-time vs batch |
| Time-series metrics | **Prometheus / Timescale** | InfluxDB, M3 | Retention + cardinality |
| Graph traversal queries | Neo4j | Postgres recursive CTE | Depth of traversal |
| Vector similarity | pgvector | Pinecone, Milvus | Scale of vectors |
| Files/media/backup | **S3 / GCS** | — | Always |

---

## Caching

| Need | Pick |
|---|---|
| Distributed cache with data structures, persistence, pub/sub | **Redis** |
| Pure, simple, multi-threaded object cache | Memcached |
| In-process cache | **Caffeine** (Java), `lru-cache` (Node), `functools.lru_cache` (Python) |
| Static assets / media at the edge | **CDN** (CloudFront, Cloudflare, Fastly) |
| API response cache | Varnish / CDN with `s-maxage` |
| Database query cache | Usually a mistake — cache the *object*, not the query |

---

## Messaging

| Need | Pick |
|---|---|
| Event streaming, replay, many consumer groups | **Kafka** (or Pulsar, Kinesis) |
| Task queue with routing, priority, per-message delay | **RabbitMQ** |
| Zero-ops simple queue | **SQS** |
| Low volume, single service, transactional | **DB outbox table + poller** |
| Lightweight streams when Redis already exists | Redis Streams |
| Long-running stateful workflows with retries | **Temporal / Step Functions** |
| Pub/sub fan-out to WebSocket gateways | Redis Pub/Sub, NATS |

---

## Compute & communication

| Need | Pick |
|---|---|
| Public API | REST + JSON over HTTPS |
| Internal service-to-service | **gRPC + protobuf** |
| Mobile with varied screens | GraphQL or a BFF |
| Server → client push | SSE (one-way) / **WebSocket** (two-way) |
| Third-party notification | Webhooks (signed, retried) |
| Bursty/event-driven jobs | Serverless (Lambda) |
| Steady load | Containers on k8s / ECS |
| Batch processing | Spark / Flink / a job scheduler |

---

## Infrastructure

| Need | Pick |
|---|---|
| Service discovery + config | Consul / etcd / Kubernetes |
| Leader election, distributed lock | **etcd / ZooKeeper** (+ fencing tokens) |
| Secrets | Vault / AWS Secrets Manager / KMS |
| Feature flags | LaunchDarkly / Unleash / in-house config service |
| Metrics | Prometheus + Grafana |
| Logs | ELK / OpenSearch / Loki / Datadog |
| Traces | **OpenTelemetry** + Jaeger / Tempo |
| CI/CD | GitHub Actions + ArgoCD |
| IaC | Terraform |

---

## "Which one?" quick answers for common follow-ups

**Redis vs Memcached** — Redis for data structures, persistence, pub/sub, Lua,
clustering. Memcached for a pure multi-threaded LRU object cache with simpler memory
behaviour. Default to Redis.

**Kafka vs RabbitMQ** — Kafka is a replayable, partitioned log for streams and multiple
independent consumers. RabbitMQ is a broker for task distribution with rich routing,
priorities, and per-message delays. Volume + replay → Kafka. Routing + per-message
control → RabbitMQ.

**SQL vs NoSQL** — see [Databases](../01-core-concepts/04-databases-and-data-modeling.md#sql-vs-nosql--how-to-actually-answer).
Short version: relational unless a specific access pattern or throughput requirement
rules it out.

**Cassandra vs DynamoDB** — same data model family. DynamoDB if you want zero ops and
accept the cost and the 400 KB item limit. Cassandra if you need multi-cloud, huge
sustained volume with predictable cost, or on-prem.

**Elasticsearch vs Postgres full-text** — Postgres FTS if you have < ~10 M documents
and modest relevance needs; it removes an entire system from your architecture.
Elasticsearch for fuzzy matching, faceting, boosting, and log analytics.

**gRPC vs REST internally** — gRPC: smaller payloads, HTTP/2 multiplexing, generated
typed clients, streaming. REST: debuggability and universal tooling. Internal → gRPC,
external → REST.

**Monolith vs microservices** — modular monolith until team size or independent
scaling/deploy needs force a split. Splitting too early buys you distributed
transactions, network failures, and deployment coordination in exchange for nothing.

**Polling vs WebSocket** — polling if updates are minutes-fresh or the client count is
small; WebSocket when you need sub-second bidirectional updates and can operate a
stateful connection tier.

**Sync vs async** — synchronous only if the caller needs the result to proceed.
Everything else goes on a queue.

---

## Naming things correctly (small credibility wins)

| Say | Not |
|---|---|
| "read replica" | "slave DB" |
| "partition key / shard key" | "the key we split on" |
| "eventual consistency with a 5 s staleness budget" | "it'll sync eventually" |
| "at-least-once delivery with idempotent consumers" | "exactly once" |
| "p99 latency" | "average response time" |
| "back-pressure / load shedding" | "it'll queue up" |
| "blast radius" | "how bad it'd be" |
| "transactional outbox" | "we'll write to both" |
