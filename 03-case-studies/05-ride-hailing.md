# Case Study 5 — Ride-Hailing (Uber / Lyft)

**Archetype:** geospatial + real-time matching + long-lived transactional state.
This problem is popular because it has three genuinely different sub-systems, and
weak candidates design only one of them.

---

## 1. Requirements

**Functional (in scope)**
1. Drivers stream their location continuously while online.
2. Riders request a ride from A → B; the system shows an ETA and price, then matches a
   nearby driver.
3. Matching: offer to a driver, they accept/decline, then the trip runs through a state
   machine to completion.
4. Both parties track each other live during the trip.

**Out of scope:** payments (reuse the payment case study), ratings, pooling/carpool
(mention it as a much harder routing problem), driver onboarding.

**Non-functional**

| Dimension | Target |
|---|---|
| Scale | 10 M drivers, 1 M concurrent online, 20 M trips/day |
| Location updates | Every 4 s per online driver |
| Latency | Match found p99 < 5 s; nearby-driver query p99 < 200 ms |
| Consistency | **A driver must never be double-booked** — strong here, eventual elsewhere |
| Availability | 99.99%; must degrade gracefully per-city |
| Durability | Trip and location history retained (regulatory) |

---

## 2. Estimation

```
Location writes:
  1 M online drivers / 4 s = 250,000 writes/s   (peak 3x = 750 K/s)
  Record: driver_id, lat, lng, heading, ts ≈ 50 B
  -> 250 K/s x 50 B = 12.5 MB/s = ~1 TB/day raw

Ride requests:
  20 M trips/day = 20/10^5 x 10^4... = ~230 req/s  (peak 3x-5x = ~1,200/s)
  Each request triggers ONE geo query but MANY candidate evaluations.

Current-location state:
  1 M drivers x ~100 B = 100 MB  -> trivially fits in memory. This is the key insight.
```

**Conclusions**
1. **Writes dominate reads by ~1000:1** — the opposite of most systems. This alone
   dictates the storage design.
2. Current location is 100 MB → keep it **in memory**, not in a database.
3. Location *history* is 1 TB/day → completely separate, append-only, cheap store.
   Never mix the two.
4. Trips are low volume but high value → a real transactional store with strong
   guarantees.

**Say this out loud:** "There are two location systems here — a tiny hot one for
matching and a huge cold one for history. Conflating them is the classic mistake."

---

## 3. Geospatial index — the core decision

![Geospatial indexing: geohash, quadtree, H3](../assets/geo-index.svg)

| Approach | Fit for this problem |
|---|---|
| **Lat/lng B-tree columns** | Two range scans then intersect — poor selectivity, dies at scale |
| **Geohash** | String prefix = bounding box → works with any KV store; **boundary problem** (neighbours can have totally different prefixes) needs the 8 neighbour cells too |
| **Quadtree** | Adapts to density beautifully; but it's a mutable tree — expensive to rebalance at 250 K writes/s |
| **H3 / S2 (hex or sphere cells)** | Uniform cell sizes, neighbours are trivially computable, no boundary pathology, hierarchical resolutions | 

**Choice: H3 at a resolution where a cell is roughly 1 km across.**

```
driver_location:  driver_id -> (lat, lng, h3_cell, heading, ts)     [hash map]
cell_index:       h3_cell   -> Set<driver_id>                        [inverted index]

Nearby query:
  1. cell = h3(rider_lat, rider_lng)
  2. cells = h3_k_ring(cell, k=1)            # 7 cells ≈ 3 km radius
  3. candidates = union of cell_index[c]
  4. if |candidates| < 10: expand k          # dense downtown vs rural
  5. filter by true haversine distance, driver status, vehicle type
  6. rank by ETA from a routing service — NOT by straight-line distance
```

**Ranking by road ETA rather than crow-flies distance** is a detail that separates
candidates: a driver 200 m away across a river is useless.

---

## 4. Handling 250 K location writes/s

The naive design ("write each update to Postgres") is the trap. Instead:

```mermaid
flowchart LR
    D["Driver app<br/>update every 4 s"] --> GW["Location Gateway<br/>WebSocket / QUIC, stateless"]
    GW --> K[["Kafka: raw locations<br/>partitioned by city/region"]]

    GW -->|"direct, low latency"| LS["Location Service (sharded by region)"]
    LS --> MEM[("In-memory: driver -> position<br/>+ H3 cell -> driver set")]

    K --> ARCH["Archiver / stream job"]
    ARCH --> LAKE[("Object storage: trip traces")]
    K --> MAP["Map-matching & analytics"]

    R[Rider app] --> API[Trip API]
    API --> MATCH["Matching Service"]
    MATCH --> LS
    MATCH --> ETA["Routing / ETA Service"]
    MATCH --> TRIP[("Trip store (sharded RDBMS)")]
    MATCH --> OFFER[["Offer queue -> driver push"]]
```

Key moves:
1. **Never write current location to durable storage on the hot path.** It's overwritten
   in 4 seconds; durability is pointless. Memory + replica is enough.
2. **Shard the Location Service by geographic region** (city). Queries are inherently
   local, so a rider in Berlin never touches the Tokyo shard. This is a rare case where
   geo-sharding is genuinely correct — usually it isn't, and you should say why it works
   here: the query has a natural locality key.
3. **Adaptive update rate:** stationary or off-shift drivers report every 30 s instead
   of 4 s. This alone can cut write volume by more than half and saves phone battery —
   a great "optimise the client, not just the server" answer.
4. Fan the raw stream to Kafka for history/analytics, entirely off the critical path.

---

## 5. Matching — the hard part

Matching is not "find the nearest driver". It's an assignment problem under
uncertainty.

### Naive: first-come-first-served, offer to nearest
Fast, simple, but globally suboptimal and vulnerable to a driver declining repeatedly.

### Better: **batched matching**
Accumulate requests for a short window (~2–5 s) per region, then solve a bipartite
assignment (Hungarian algorithm / min-cost flow) over the batch.

- Reduces total wait time and empty miles across the whole market.
- The 2–5 s delay is invisible to the rider (they already expect a "finding your
  driver" spinner).
- Naturally handles the case where two riders both want the same driver.

### Preventing double-booking
This is the one place you need **strong consistency**:

```
Offer flow:
  1. Matching picks driver D for request R.
  2. Conditional write: SET driver_state[D] = OFFERED(R) IF state == AVAILABLE
     (compare-and-swap, single-homed per driver)  -> if it fails, pick another driver.
  3. Push offer to D with a 15 s TTL.
  4. D accepts -> CAS OFFERED(R) -> ON_TRIP(R). D declines / TTL expires -> back to
     AVAILABLE, and R re-enters the next matching batch.
```

The lease/TTL is essential: a driver whose phone dies mid-offer must not be locked out
forever. **Reservation with expiry** is the reusable pattern here (same as ticket
booking).

### Surge pricing
Computed per H3 cell per few minutes from a demand/supply ratio, published to a cache
that both the pricing quote and the matcher read. It's an eventually-consistent,
approximate signal — and once a rider is quoted a price, that price is **locked** into
the trip record so later surge changes can't alter it.

---

## 6. Data model

```
driver_state          (in-memory, replicated; source of truth for matching)
  driver_id -> {status, lat, lng, h3, heading, ts, current_trip}

trips                 (sharded RDBMS / Spanner-like; strong consistency)
  PK: trip_id (UUID)   shard by rider_id or city
  rider_id, driver_id, state, quoted_price, pickup, dropoff, timestamps...
  + trip_events (append-only audit log of every state transition)

trip_traces           (object storage, columnar, partitioned by day/city)
  trip_id, [(ts, lat, lng)]   -- written after the trip, not during

driver_daily_stats    (analytics store)
```

The **trip state machine** is worth drawing:

```mermaid
stateDiagram-v2
    [*] --> Requested
    Requested --> Matching
    Matching --> Offered
    Offered --> Matching : declined or timed out
    Offered --> Accepted
    Matching --> NoDriversFound : after N rounds
    Accepted --> DriverArrived
    DriverArrived --> InProgress
    InProgress --> Completed
    Accepted --> Cancelled
    DriverArrived --> Cancelled
    Completed --> [*]
    Cancelled --> [*]
    NoDriversFound --> [*]
```

Persist every transition as an event. Reconstructing "what happened on this trip" is a
support, billing, and legal requirement — this is a natural, justified use of **event
sourcing** for one entity rather than the whole system.

---

## 7. Deep dives

### Why geo-sharding works here (and usually doesn't)
Sharding by city gives locality (queries never cross shards), natural failure isolation
(a Tokyo outage doesn't affect Berlin), and matches regulatory boundaries. The usual
objection — hot shards — is real: New Year's Eve in one city. Mitigate by splitting hot
cities into sub-regions and by making the Location Service horizontally scalable within
a region. This is a **cell-based architecture** where the cell key is geography.

### Live trip tracking
During a trip, the rider needs the driver's position. Don't poll the Location Service
from every rider. Instead the Location Gateway publishes that driver's updates to a
per-trip topic/channel; the rider's WebSocket subscribes. One producer, one consumer,
no fan-out problem.

### ETA computation
A separate service over a road graph with live traffic. It is expensive — cache
aggressively: quantise origin/destination to H3 cells and cache cell-pair ETAs for
30–60 s. During matching you need dozens of ETAs per request, so batch them into one
call (matrix API).

### Idempotency
Riders double-tap "Request ride" on bad networks. Every request carries an
`Idempotency-Key`; the Trip API stores it with the created trip and returns the same
trip on retry. Without this you create duplicate trips and charge people twice.

### Offline / degraded mode
If the Matching Service can't reach the routing service, fall back to straight-line
distance ranking. If the surge service is down, use the last published multiplier.
**Static stability**: the system keeps working with stale data rather than failing.

---

## 8. Failure modes

| Failure | Behaviour |
|---|---|
| Location Service node dies | Its region's driver positions are lost. Drivers re-report within 4 s → self-healing. Keep a warm replica to avoid even that gap |
| Kafka down | Matching is unaffected (it reads memory, not Kafka). History/analytics buffer locally and replay |
| Trip DB shard down | Trips in that shard can't transition. Buffer transitions in a durable queue and apply on recovery; do not lose an in-progress trip |
| Driver app loses network mid-trip | Client buffers positions and uploads on reconnect; trip state advances via rider-side signals + timeouts |
| Routing/ETA down | Fall back to haversine ranking; degrade the ETA shown, don't block matching |
| Whole region down | Fail over to a neighbouring region's cell (higher latency, still functional); trips are geo-partitioned so blast radius is one city |

---

## 9. Scale evolution

- **10x location updates:** adaptive reporting rate, delta encoding, binary protocol
  (protobuf over QUIC), and drop the archival stream first if it becomes the
  bottleneck — history is the least critical data.
- **10x trips:** matching is already partitioned by region and batched; add matcher
  instances per region. The Trip DB shards by city.
- **New product (pooling):** matching becomes a multi-passenger routing problem —
  acknowledge that it changes the algorithm (insertion heuristics over existing
  routes), not the architecture.

---

## 10. Trade-offs summary

| Decision | Chose | Alternative | Why |
|---|---|---|---|
| Geo index | H3 cells + in-memory inverted index | Postgres/PostGIS | 250 K writes/s; uniform cells; no boundary pathology |
| Current location storage | In-memory, non-durable | Durable DB writes | Data is obsolete in 4 s; durability buys nothing |
| Sharding | By city/region | By driver_id hash | Queries are inherently geo-local; gives failure isolation |
| Matching | Batched assignment (2–5 s window) | Greedy nearest-first | Globally better assignment; delay is invisible |
| Driver reservation | CAS + TTL lease | Distributed lock | No double-booking, and self-heals when a phone dies |
| Trip state | Event-sourced state machine | Mutable row only | Auditability for billing, support, and regulators |
| Consistency | Strong for driver assignment, eventual everywhere else | Strong everywhere | Only one invariant genuinely needs it |
