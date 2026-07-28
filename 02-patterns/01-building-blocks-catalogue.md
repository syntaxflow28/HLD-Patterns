# Building Blocks Catalogue

The reusable sub-systems that appear inside larger designs. Interviewers often ask for
one of these *as* the whole question.

---

## The reference architecture (everything in one picture)

![Reference architecture: edge, stateless services, state, async, cross-cutting](../assets/reference-architecture.svg)

Memorize this shape. Almost every design is a subset of it. In an interview, draw only
the boxes your requirements justify — and be able to say why each one is there.

---

## 1. ID Generator

| Option | Sortable | Coordination | Size | Notes |
|---|---|---|---|---|
| DB auto-increment | Yes | Central | 8 B | SPOF, doesn't shard |
| DB ticket server + ranges | Yes | Light | 8 B | Each node leases 1000 ids |
| UUIDv4 | No | None | 16 B | Kills B-tree locality |
| **UUIDv7 / ULID** | Yes | None | 16 B | Modern default |
| **Snowflake** | Yes | Machine id only | 8 B | `[41b time][10b node][12b seq]` |

![Snowflake 64-bit ID layout](../assets/snowflake-id.svg)

Snowflake caveats to mention: clock skew/NTP going backwards (halt or use a logical
clock), node-id assignment (ZooKeeper/etcd or k8s ordinal), and that time-sortable IDs
leak creation time and volume — use an opaque public id if that matters.

---

## 2. Rate Limiter

```mermaid
flowchart LR
    R[Request] --> K["Key = user / apiKey / ip / endpoint"]
    K --> L{Local token bucket<br/>has lease?}
    L -->|yes| ALLOW[Allow]
    L -->|no| RED[(Redis: atomic Lua<br/>token bucket per key)]
    RED -->|tokens left| ALLOW
    RED -->|empty| DENY["429 + Retry-After"]
```
Decisions to state: algorithm (token bucket), storage (Redis with Lua for atomicity),
granularity (per user + per endpoint), distributed strategy (local lease + central
budget), and failure mode (fail open for availability, fail closed for abuse-critical
endpoints).

---

## 3. Notification Service

```mermaid
flowchart LR
    SRC[Producers] --> API[Notification API]
    API --> PREF[(User preferences<br/>+ quiet hours + opt-outs)]
    API --> TMPL[Template + localization]
    TMPL --> Q[[Per-channel queues]]
    Q --> P1["Push worker to APNs / FCM"]
    Q --> P2["Email worker to SES / SendGrid"]
    Q --> P3["SMS worker to Twilio"]
    Q --> P4["In-app worker to WebSocket"]
    P1 & P2 & P3 & P4 --> ST[(Delivery status + retries + DLQ)]
    ST --> AN[Analytics: delivered, opened, unsubscribed]
```
Talking points: dedup/idempotency (never send twice), rate limiting per user
(notification fatigue), digest/batching, provider failover, priority lanes (OTP beats
marketing), device-token invalidation, and compliance (unsubscribe, quiet hours).

---

## 4. Distributed Lock / Leader Election

```mermaid
flowchart TD
    A[Need mutual exclusion] --> B{How critical?}
    B -->|"best effort, e.g. cron dedup"| R["Redis SET key val NX PX ttl<br/>+ release only if value matches"]
    B -->|"correctness critical"| Z["etcd / ZooKeeper lease<br/>+ FENCING TOKEN"]
    Z --> F["Storage rejects writes with<br/>a stale token -> safe under GC pause"]
```
The classic failure: process holds the lock, GC-pauses past the TTL, another process
acquires it, then the first wakes up and writes. **Only fencing tokens fix this.**
Better still: design so you don't need a lock (partition ownership, idempotency,
conditional writes).

---

## 5. Scheduler / Delayed Jobs

```mermaid
flowchart LR
    J[Schedule job at T] --> ST{Delay range}
    ST -->|"seconds to minutes"| DQ["Delay queue<br/>SQS delay, Redis ZSET by score"]
    ST -->|"hours to years"| TB["Time-bucketed table<br/>PK = (bucket_minute, job_id)<br/>poller scans the current bucket"]
    DQ --> EXEC[Executor pool]
    TB --> EXEC
    EXEC --> IDEM[Idempotent execution + lease]
    EXEC -->|failure| RETRY[Retry with backoff -> DLQ]
```
Cover: at-least-once execution → idempotency; leases so two workers don't run the same
job; clock skew; thundering herd at round times (jitter the schedule); and cancellation.

---

## 6. Counter / Metrics Aggregator

```mermaid
flowchart LR
    EV[Events] --> LOCAL[In-process pre-aggregation<br/>per instance, flush every 10s]
    LOCAL --> Q[[Stream]]
    Q --> AGG[Windowed aggregation]
    AGG --> HOT[(Redis: real-time counters)]
    AGG --> TS[(Time series / columnar store)]
    HOT --> API[Read API]
    TS --> API
```
Never `UPDATE counter SET n = n+1` on a hot row — it serializes. Options: pre-aggregate
in-process, sharded counters (`counter:{id}:{0..15}` summed on read), Redis `INCR`,
or a stream aggregation. For approximate uniques use HyperLogLog.

---

## 7. Feature Flag / Config Service

```mermaid
flowchart LR
    UI[Admin UI] --> CFG[(Config store)]
    CFG --> PUB[[Pub/Sub or long-poll]]
    PUB --> S1[Service instance<br/>local cache + last-known-good]
    S1 -->|"store unreachable"| LKG["Serve last-known-good<br/>static stability"]
```
Requirements: sub-second propagation, targeting rules (% rollout, user cohort), audit
trail, and — critically — the service must keep working when the config store is down.

---

## 8. Audit Log / Event Store

Append-only, immutable, tamper-evident (hash chain), with a defined retention policy.
Serves compliance, debugging, and replay. If you also need to *rebuild state* from it,
you are doing event sourcing — see the next document.

---

## 9. Deduplication Service

```mermaid
flowchart LR
    M[Incoming item] --> BF{Bloom filter<br/>seen before?}
    BF -->|"definitely not"| PROC[Process]
    BF -->|"maybe"| EX[(Exact check in KV store<br/>with TTL window)]
    EX -->|new| PROC
    EX -->|dup| DROP[Drop]
```
Bloom filter as a cheap pre-filter; exact store for confirmation. Bound memory with a
time window (you rarely need to dedup forever).

---

## 10. WebSocket / Presence Gateway

```mermaid
flowchart LR
    C[Clients] --> GWS[WS Gateway fleet<br/>~50K conns per node]
    GWS --> REG[("Registry: user to node<br/>Redis, TTL + heartbeat")]
    SVC[Backend] --> REG
    SVC --> PS[[Pub/Sub per node or per topic]]
    PS --> GWS
    GWS --> C
```
Handle: heartbeats and dead-connection cleanup, reconnect with a resume cursor to
backfill missed messages, graceful drain on deploy (connections must be re-established
gradually, not all at once), and presence as a TTL key refreshed by heartbeat.

---

## 11. Sequencer / Ordering Service

Needed for chat message ordering, collaborative editing, and event replay. Options:
per-conversation monotonic sequence in the DB (`UPDATE ... RETURNING seq+1`), Kafka
partition offsets, or Lamport/vector clocks when there is no single writer.

---

## 12. Payments / Ledger

```mermaid
flowchart LR
    REQ["POST /payments + Idempotency-Key"] --> IDEMP[(Idempotency store)]
    IDEMP --> AUTH[Authorize with PSP]
    AUTH --> LEDGER[("Double-entry ledger:<br/>every txn = balanced debits/credits,<br/>append only")]
    LEDGER --> OB[(Outbox)]
    OB --> EVT[[Events]]
    EVT --> RECON[Reconciliation job<br/>vs PSP settlement file]
```
Non-negotiables: idempotency keys, an append-only double-entry ledger (never mutate
balances; derive them), a state machine for payment status, webhooks from the PSP
being unordered and duplicated, and a daily reconciliation job. Money questions are
graded on correctness, not throughput.

---

## Practice: which blocks does each problem need?

| Problem | Blocks |
|---|---|
| URL shortener | ID generator, cache, KV store, rate limiter, analytics counter |
| Twitter | ID gen, fan-out workers, timeline cache, blob+CDN, search, notification |
| Uber | Geo index, WS gateway, matching service, scheduler, ledger |
| Google Docs | WS gateway, sequencer/CRDT, snapshot store, presence |
| Ticketmaster | Reservation with TTL, optimistic locking, queue/waiting room, ledger |
| Metrics system | Counter aggregator, time-series store, stream processor, alerting |
