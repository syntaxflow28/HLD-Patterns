# Blob Storage, CDN & Media Pipelines

Whenever the problem mentions images, video, files, or documents, this is the whole
answer: **metadata in the database, bytes in object storage, delivery via CDN.**

---

## The universal media architecture

```mermaid
flowchart LR
    C[Client] -->|"1. request upload"| API[Upload Service]
    API -->|"2. presigned URL"| C
    C -->|"3. PUT bytes directly"| S3[(Object Storage<br/>S3 / GCS / Blob)]
    S3 -->|"4. event: ObjectCreated"| Q[[Queue]]
    Q --> W[Processing Workers]
    W --> S3
    W --> DB[(Metadata DB)]
    R[Reader] --> CDN[CDN]
    CDN -->|"cache miss"| S3
```

**The presigned-URL move is the key insight:** bytes never pass through your API
servers. Your service only issues a short-lived, scoped credential. This removes the
biggest bandwidth and memory bottleneck from your app tier — always say it.

---

## Object storage vs file system vs database

| | Object store (S3) | Block/File (EBS, NFS) | Database BLOB |
|---|---|---|---|
| Scale | Effectively unlimited | Volume-limited | Bloats the DB, kills backups |
| Durability | ~11 nines | Depends | Depends |
| Access | HTTP GET/PUT, whole object | POSIX, random access | SQL |
| Cost | Cheapest | Medium | Most expensive |
| Use for | Media, backups, logs, data lake | Shared working dirs, legacy apps | **Almost never for large blobs** |

Rule: put a URL in the database, not the bytes. Exception: small blobs (< ~64 KB) where
transactional consistency with the row matters.

---

## Uploads at scale

```mermaid
sequenceDiagram
    participant C as Client
    participant A as API
    participant S as Object Store
    C->>A: POST /uploads {filename, size, contentType}
    A->>A: validate size/type, quota, create record status=PENDING
    A->>S: create multipart upload
    A-->>C: uploadId + presigned part URLs
    loop each 5-10 MB part, parallel, resumable
        C->>S: PUT part N
        S-->>C: ETag N
    end
    C->>A: POST /uploads/{id}/complete {parts}
    A->>S: complete multipart upload
    A->>A: status=UPLOADED, emit event
```

Cover these:
- **Multipart/chunked** upload → resumable on flaky networks, parallel parts.
- **Deduplication** by content hash (SHA-256) → same file stored once, instant "upload".
- **Validation server-side** — never trust client-declared content type; sniff magic
  bytes; enforce max size at the presign policy level.
- **Virus/abuse scanning** in the async pipeline before the object becomes public.
- **Lifecycle for orphans**: abort incomplete multipart uploads after N days.

---

## Media processing pipeline

```mermaid
flowchart LR
    UP[Uploaded original] --> Q[[Queue]]
    Q --> V[Validate + probe metadata]
    V --> T[Transcode fan-out]
    T --> R1[240p]
    T --> R2[480p]
    T --> R3[720p]
    T --> R4[1080p]
    T --> TH[Thumbnails / sprite sheet]
    R1 & R2 & R3 & R4 --> SEG["Segment into chunks<br/>HLS / DASH + manifest"]
    SEG --> CDNS[(CDN origin)]
    TH --> CDNS
    SEG --> META[(Metadata DB<br/>status = READY)]
```

Talking points:
- **Split by segment, not by file** — a 2-hour video is chunked and transcoded in
  parallel across many workers, then stitched. Turns hours into minutes.
- **DAG of tasks** with per-step retries and idempotent outputs keyed by
  `(assetId, profile, segment)`.
- **Adaptive bitrate streaming (HLS/DASH)**: the client switches renditions based on
  measured bandwidth. This is why you transcode multiple resolutions.
- **Priority queues** — a 15-second clip should not wait behind a feature film.
- **Cost control**: transcode popular renditions eagerly, rare ones lazily on first
  request.

---

## CDN

```mermaid
flowchart TB
    U1[User Tokyo] --> E1[Edge PoP Tokyo]
    U2[User Berlin] --> E2[Edge PoP Berlin]
    E1 -->|miss| RS[Regional shield / mid-tier cache]
    E2 -->|miss| RS
    RS -->|miss| O[(Origin)]
```

**What a CDN buys you:** lower latency (physically closer), origin offload (95%+ of
bytes), DDoS absorption, TLS termination at the edge, and often edge compute.

| Mode | How | Use |
|---|---|---|
| Pull | Edge fetches from origin on first miss | Default; self-healing |
| Push | You upload to the CDN | Large files, predictable releases |

### Cache-control at the edge
```
static assets with hashed names   Cache-Control: public, max-age=31536000, immutable
API responses that can be stale   Cache-Control: public, s-maxage=60, stale-while-revalidate=600
per-user content                  Cache-Control: private, no-store
```
- **Cache key** matters: including cookies or all query params destroys your hit rate.
  Normalize/whitelist query params.
- **Invalidation** is slow and rate-limited on most CDNs → prefer **versioned URLs**
  (`/static/app.9f2a1c.js`, `/img/123?v=7`). Purge only in emergencies.
- **Shield/tiered caching** prevents 200 edge PoPs from all stampeding the origin.
- **Signed URLs / signed cookies** for private media, with short expiry.
- **Origin protection**: origin should only accept traffic from the CDN.

### The cold-start / thundering herd at the edge
A new viral video is a miss at every PoP simultaneously. Mitigations: tiered caching,
request coalescing at the edge, and pre-warming for known events (product launch,
live sports).

---

## Storage cost & tiering

![Storage tiering and cost](../assets/storage-tiering.svg)
Also mention: **egress is usually the dominant cost**, not storage — which is another
argument for a high CDN hit rate. And erasure coding vs replication: 3x replication
costs 200% overhead; Reed-Solomon (e.g. 10+4) gives similar durability at ~40%.

---

## Delivery of user-specific vs public content

```mermaid
flowchart TD
    D{Is the object public?}
    D -->|yes| PUB["CDN with long TTL,<br/>versioned URL"]
    D -->|no| PRIV["Signed URL with short TTL<br/>+ CDN with private cache key,<br/>authz check at issue time"]
```
For private content, do the authorization when **issuing** the signed URL, not on
every byte — otherwise you lose the CDN entirely.

---

## Interview checklist

- [ ] Metadata in DB, bytes in object storage — stated explicitly
- [ ] Presigned URLs so uploads bypass app servers
- [ ] Multipart/resumable upload for large files
- [ ] Async processing pipeline with idempotent, retryable steps
- [ ] CDN with a cache-control and invalidation strategy (versioned URLs)
- [ ] Signed URLs for private content
- [ ] Lifecycle/tiering for cost, and an egress cost comment
