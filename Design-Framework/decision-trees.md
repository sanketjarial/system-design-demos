# Decision Trees

Visual tie-breakers for the recurring decisions in any design. Walk the relevant tree from top to bottom while filling the worksheet.

---

## 1. Storage choice

```mermaid
flowchart TD
    A[Need to store data] --> B{Need ACID transactions<br/>or complex joins?}
    B -->|Yes| C{Data > 10 TB OR<br/>write QPS > 50k?}
    C -->|No| D[Postgres / MySQL]
    C -->|Yes| E{Can split by tenant<br/>or natural key?}
    E -->|Yes| F[Sharded Postgres<br/>or CockroachDB / Spanner]
    E -->|No| G[Reconsider:<br/>do you really need ACID at scale?]
    B -->|No| H{What's the access pattern?}
    H -->|Key-value /<br/>simple lookups| I[DynamoDB / Cassandra]
    H -->|Document /<br/>flexible schema| J[MongoDB / Firestore]
    H -->|Full-text search| K[Elasticsearch / OpenSearch]
    H -->|Time-series /<br/>append-only events| L[Kafka + Timescale / Influx]
    H -->|Blobs / media| M[S3 / GCS / Azure Blob]
    H -->|Highly connected /<br/>graph traversals| N[Neo4j / Neptune]
    H -->|Hot reads /<br/>ephemeral / cache| O[Redis / Memcached]

    style D fill:#d1fae5
    style F fill:#d1fae5
    style I fill:#d1fae5
    style J fill:#d1fae5
    style K fill:#d1fae5
    style L fill:#d1fae5
    style M fill:#d1fae5
    style N fill:#d1fae5
    style O fill:#d1fae5
    style G fill:#fee2e2
```

**Heuristic:** start with Postgres unless you have a concrete reason not to. It scales further than people think.

---

## 2. Sync vs async

```mermaid
flowchart TD
    A[New operation] --> B{Is the user<br/>waiting on the result?}
    B -->|No| C[Async: queue + worker]
    B -->|Yes| D{Can it complete<br/>in under 200ms?}
    D -->|Yes| E[Sync]
    D -->|No| F{Can we return an ack<br/>and finish in background?}
    F -->|Yes| G[Sync ack + async work<br/>poll or webhook for result]
    F -->|No| H[Sync — but optimize:<br/>parallelize, cache, precompute]

    style C fill:#d1fae5
    style E fill:#d1fae5
    style G fill:#d1fae5
    style H fill:#fef3c7
```

**Rule of thumb:** anything > 200ms with a user waiting needs to become an ack-and-poll pattern.

---

## 3. Consistency level

```mermaid
flowchart TD
    A[Data write] --> B{Money / inventory /<br/>identity / locks /<br/>uniqueness?}
    B -->|Yes| C[Strong consistency<br/>single-leader, sync replication]
    B -->|No| D{Will the same user read<br/>their own write immediately?}
    D -->|Yes| E[Read-your-writes<br/>route reads to leader,<br/>or sticky session]
    D -->|No| F{Tolerable staleness?}
    F -->|Seconds| G[Eventual<br/>async replication]
    F -->|Minutes-hours| H[Eventual + cache TTL]

    style C fill:#fee2e2
    style E fill:#fef3c7
    style G fill:#d1fae5
    style H fill:#d1fae5
```

**Cost order:** strong > read-your-writes > eventual. Default to eventual unless the use case demands stronger.

---

## 4. Cache strategy

```mermaid
flowchart TD
    A[Repeated reads<br/>of same data?] -->|No| B[No cache.<br/>Don't add complexity.]
    A -->|Yes| C{Read:write ratio > 10:1?}
    C -->|No| D[Cache may not help much.<br/>Measure before adding.]
    C -->|Yes| E{How fresh must reads be?}
    E -->|Eventual /<br/>minutes OK| F[Long TTL,<br/>refresh in background]
    E -->|Seconds OK| G[Short TTL cache-aside]
    E -->|Always fresh| H[Write-through cache,<br/>or invalidate on write]

    style B fill:#fee2e2
    style D fill:#fef3c7
    style F fill:#d1fae5
    style G fill:#d1fae5
    style H fill:#fef3c7
```

**Where to cache** (cheapest to most expensive miss):
client → CDN → app-local → distributed cache (Redis) → DB query cache

---

## 5. When to shard

```mermaid
flowchart TD
    A[Is the DB at risk?] --> B{Storage > 80% capacity<br/>OR write QPS at limit?}
    B -->|No| C[Don't shard yet.<br/>- Vertical scale<br/>- Read replicas<br/>- Caching<br/>- Indexes]
    B -->|Yes| D{Tried all of:<br/>read replicas, cache,<br/>indexes, partitioning?}
    D -->|No| E[Try those first.<br/>Sharding is one-way.]
    D -->|Yes| F{Have a key with<br/>high cardinality<br/>+ low skew?}
    F -->|No| G[Find one or denormalize.<br/>Bad shard key = pain.]
    F -->|Yes| H[Shard.<br/>Plan resharding strategy<br/>BEFORE you ship.]

    style C fill:#d1fae5
    style E fill:#fef3c7
    style G fill:#fee2e2
    style H fill:#d1fae5
```

**Bad shard keys:** timestamp (hot tail), low-cardinality enums, anything skewed (one big tenant).
**Good shard keys:** user_id, tenant_id, hash of natural key.

---

## 6. Build vs buy

```mermaid
flowchart TD
    A[Need a capability] --> B{Is it a core differentiator<br/>for your product?}
    B -->|Yes| C[Build it.<br/>Own the roadmap.]
    B -->|No| D{Mature off-the-shelf option?}
    D -->|No| E[Build minimal version,<br/>plan to replace later]
    D -->|Yes| F{Cost + lock-in acceptable?}
    F -->|Yes| G[Buy / use SaaS]
    F -->|No| H{Can the team<br/>maintain it long-term?}
    H -->|Yes| I[Self-host open source]
    H -->|No| J[Buy. Accept lock-in.<br/>Migration cost < ops cost.]

    style C fill:#d1fae5
    style E fill:#fef3c7
    style G fill:#d1fae5
    style I fill:#d1fae5
    style J fill:#fef3c7
```

---

## 7. Cross-cutting tradeoff matrix

For decisions that don't deserve a full tree:

| Question | Lean A when… | Lean B when… |
|---|---|---|
| **Push vs pull** | Low-latency notify, few consumers | Many consumers, batching OK |
| **Stateful vs stateless** | Sticky sessions, in-memory state | Need easy horizontal scale |
| **Monolith vs microservices** | Small team, early stage, tight coupling | Many teams, independent deploys |
| **Pessimistic vs optimistic lock** | High contention, short txn | Low contention, long txn |
| **Leader-follower vs multi-leader** | One source of truth, simpler | Multi-region writes, conflict-tolerant |
| **CAP under partition** | CP: banking, locks, inventory | AP: feeds, social, search |
| **REST vs gRPC** | Public API, browser clients | Internal, low-latency, typed |
