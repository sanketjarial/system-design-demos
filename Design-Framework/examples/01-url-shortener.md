# Example 01 — URL Shortener (TinyURL-style)

> The same `worksheet.md`, fully filled in. Read this end-to-end after your first attempt at a design — see how a real walk through the framework looks.

**Date:** 2026-05-07 &nbsp; **Time budget:** 45 min

---

## 1. Problem

**Top 3 features:**
1. POST a long URL → get a short URL back
2. GET short URL → 301 redirect to long URL (the hot path)
3. Per-link analytics (click count over time)

**Out of scope:** custom aliases, link expiration, user accounts, abuse/spam filtering

**NFRs:**
- DAU: 100M
- Peak QPS: **1k write / 100k read** (1:100 ratio — reads dominate)
- Latency p99: 100 ms (redirect must feel instant)
- Availability: 99.99% (4 nines — redirect outage breaks every share)
- Consistency: **eventual** OK; analytics can lag minutes
- Data: 1B URLs now, +100M/year
- Compliance: none

**Assumptions:** average 500 bytes per record, 90% of traffic on top 10% of URLs (Pareto), no per-user rate limits beyond IP throttling.

---

## 2. Scale estimate

- Reads: ~100k QPS &nbsp; Writes: ~1k QPS &nbsp; Ratio: **100:1**
- Storage: 500 B × 1B records × 3× replication = **~1.5 TB** (fits any modern DB easily)
- Bandwidth: 1 KB × 100k QPS = **100 MB/s egress** (this is the cost driver)
- Hot set: top 10% = 100M records × 500 B = **50 GB** → fits in Redis cluster

**Sanity check:** big QPS, small data. Read path is the problem, not storage.

---

## 3. API

**Style:** REST (public, browser clients hit redirect directly)

| Verb | Path | Request | Response | Notes |
|---|---|---|---|---|
| POST | /shorten | `{url}` | `{short_code, short_url}` | Idempotent on hash(url+user); auth optional |
| GET | /:code | — | `301` to long_url | The hot path. Must be cache-friendly. |
| GET | /:code/stats | — | `{clicks, last_24h, ...}` | Low QPS; can be slow |

---

## 4. Data model

**Access patterns:**
1. **Lookup by `short_code` → long_url** — HOT (100k QPS, p99 100ms)
2. Insert new mapping — 1k QPS
3. Increment click count for `short_code` — 100k QPS (same as reads)
4. Read stats for `short_code` — low QPS, can be slow

**Storage choice:** **DynamoDB** (or Cassandra) for the URL table + **Redis** for hot read cache.
Why: pure key-value access pattern, no joins, predictable latency at QPS, autoscaling. *(decision-trees.md → storage: "Key-value / simple lookups" branch)*

**Schema sketch:**
```
table: urls
  short_code  STRING  PK     // base62, 7 chars
  long_url    STRING         // up to 2KB
  created_at  TIMESTAMP
  user_id     STRING  (optional)

table: clicks  (separate, write-heavy, eventually consistent)
  short_code  STRING  PK
  bucket      TIMESTAMP (hour)  SK
  count       NUMBER (atomic increment)
```

**Shard key:** `short_code` — high cardinality (62^7 ≈ 3.5T possible), uniform distribution by construction.

---

## 5. High-level architecture

```
                    ┌─── Redis Cluster ───┐
                    │   (hot URL cache)   │
                    └──────────▲──────────┘
                               │
Client ──► CDN ──► LB ──► API Server ──► DynamoDB (urls)
                               │
                               └──► Kafka ──► Analytics Worker ──► DynamoDB (clicks)
```

**Read path (GET /:code):**
Client → CDN → LB → API → Redis (hit, 95%+) → 301
                                ↓ (miss)
                              DynamoDB → fill Redis → 301
*Also publish a click event to Kafka (fire-and-forget, async).*

**Write path (POST /shorten):**
Client → LB → API → generate `short_code` → DynamoDB write → return `{short_code}`

---

## 6. DEEP DIVE — short code generation

**Why hardest:** every write needs a unique 6–7 char code, fast, no collisions, at scale, across many app servers.

**Alternatives considered:**

- **A) Random + collision check** — generate random base62 string, check DB, retry on collision.
  *Rejected:* at 1B records, collision probability rises; every write needs a read first → doubles DB load.

- **B) Hash of long URL (e.g., MD5 → base62 first 7)** — deterministic, no coordination.
  *Rejected:* identical URLs from different users collapse to same code (some products want this, but here we want per-submission uniqueness for analytics). Also: hash collisions still possible.

- **C) Counter + base62 encode (chosen)** — central monotonic counter, each write gets next ID, base62-encoded to 7 chars.
  *Chosen:* no collisions by construction; trivially fast; predictable.

**Algorithm:**
1. App servers reserve **ranges of 10,000 IDs** from a counter service (Zookeeper / etcd / Redis INCRBY) at startup or when range is exhausted.
2. For each write, app server takes next ID from its local range, base62-encodes it.
3. Insert `{short_code, long_url}` into DynamoDB.
4. Counter service is hit only **1 per 10,000 writes** → not a bottleneck even at 100k write QPS.

**Math:** 62^7 ≈ 3.5 trillion codes — 35,000 years at current rate. Range allocation: 100k QPS / 10k = 10 counter calls/sec — trivial.

---

## 7. Bottlenecks & failure modes

| Component | What breaks first | Mitigation |
|---|---|---|
| DynamoDB read | Read QPS, hot keys | Redis cache (target 95%+ hit rate) |
| Counter service | Single point | Range allocation (10k IDs/call); fall back to UUID if down |
| Redis | Memory, node failure | Cluster mode, replicas, consistent hashing |
| Analytics writes | Hot key (popular URL) | Kafka buffer + batch increments per bucket |

**Cache:** Redis, key = `short_code`, value = long_url, TTL 1h, cache-aside on miss. Invalidate on update (rare). *(decision-trees.md → cache: short TTL cache-aside branch)*

**Retries:** writes are idempotent on `short_code` (PK, fail on duplicate). Reads are pure GETs, retry-safe.

**What if Redis dies?** DB takes the full 100k QPS. DynamoDB autoscale kicks in within ~minutes; latency rises but service continues. Pre-warm replacement node from DB.

**What if DynamoDB dies?** Multi-region, multi-AZ replication. RPO ≈ 0, RTO ≈ 30s with automatic failover.

**What if counter dies?** Each app server has 10k IDs buffered → 10s of writes survive without counter. Fall back to UUID-based codes (longer but never collide).

---

## 8. Observability

- **SLO:** 99.95% of redirects under 100 ms, measured per minute, per region.
- **Top metrics (RED):** redirect rate, redirect 5xx rate, redirect p50/p99/p999 duration.
- **Alerts on symptoms:**
  - Cache hit rate < 90% for 5 min (something's wrong with cache)
  - p99 > 200 ms for 3 min
  - DynamoDB throttle errors > 0.1%
- **Trace propagation:** OpenTelemetry across API → cache → DB → Kafka.

---

## 9. Security

- **AuthN:** none for redirect (public); API token for POST /shorten.
- **AuthZ:** rate-limit per IP on /shorten to prevent spam.
- **Top 3 threats:**
  1. **Open redirect abuse** — phishing campaigns hide behind our domain. Mitigate with malicious-URL list (Google Safe Browsing API) at write time.
  2. **DDoS on hot URL** — one viral link saturates a Redis shard. Mitigate with CDN caching of 301 responses (with short TTL).
  3. **Enumeration** — sequential IDs leak business volume. Mitigate by skipping IDs randomly within range, or shuffling base62 alphabet.

---

## 10. Cost

- **Dominant cost:** **egress bandwidth** (100 MB/s × 86400 × 30 ≈ 250 TB/month → ~$15k/mo at $0.06/GB)
- Compute: ~$3k/mo, DynamoDB: ~$2k/mo, Redis: ~$1k/mo
- **Total:** ~$21k/mo at 100k QPS sustained
- **Biggest lever:** push more redirects to CDN edge (cache the 301 itself for 60s) → 50%+ egress reduction

---

## 11. Trade-off log

- **Chose DynamoDB + Redis over Postgres** — access pattern is pure KV at high QPS; Postgres would need read replicas + heavy caching anyway, and offers nothing we need (no joins, no transactions).
- **Chose counter + base62 over hash(url)** — guarantees no collisions, simpler reasoning, allows per-submission uniqueness.
- **Risk accepted:** counter service is critical infra; mitigated by range allocation + UUID fallback.
- **Risk accepted:** eventual analytics — clicks may lag minutes; acceptable per requirements.
- **Revisit when:**
  - Write QPS > 100k → counter service may need sharding
  - Need custom aliases → counter approach doesn't apply for those (hybrid strategy)
  - Egress cost > $50k/mo → invest in edge caching infrastructure

---

## What this exercise teaches

Compare to your first-pass attempt:

1. **Did you push back on requirements?** "100k QPS read" changes everything. Without that number, you'd over-engineer the write path or under-engineer the read path.
2. **Did you list access patterns before storage?** That single step makes "DynamoDB vs Postgres" obvious instead of a debate.
3. **Did you pick the actual hardest part?** It's not the API or the diagram — it's the **short code generation** algorithm. That's where 15 minutes of focus pays off.
4. **Did you log the trade-off?** Six months from now, when traffic 10×s, the "revisit when" line is what saves you.
