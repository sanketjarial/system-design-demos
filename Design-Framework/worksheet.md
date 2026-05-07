# Design Worksheet — [System Name]

> Copy this file and rename to `examples/NN-system-name.md` (or wherever) for each new design. Fill top to bottom.

**Date:** ___ &nbsp; **Time budget:** ___ min &nbsp; **Designer:** ___

---

## 1. Problem  *(5 min)*

**Top 3–5 features** (must have):
1.
2.
3.

**Out of scope** (explicit):
-

**Non-functional requirements** *(numbers, not adjectives)*:
- DAU / MAU: ___
- Peak QPS: ___ read &nbsp; / ___ write
- Latency p99: ___ ms (per endpoint if they differ)
- Availability target: ___ % (e.g. 99.9 / 99.99)
- Consistency need: ☐ strong &nbsp; ☐ read-your-writes &nbsp; ☐ eventual
- Data size now: ___ &nbsp; growth: ___ /year
- Compliance: ☐ none ☐ GDPR ☐ HIPAA ☐ SOC2 ☐ other: ___

**Assumptions made** (challenge later):
-

---

## 2. Scale estimate  *(3 min, in head)*

- Reads/sec: ___ &nbsp; Writes/sec: ___ &nbsp; Ratio: ___ : 1
- Storage: ___ bytes/record × ___ records × ___ retention × ___ replication = **___**
- Bandwidth: ___ KB/req × ___ QPS = **___ MB/s** (egress is what costs)
- Memory (hot set): ___ % of data → ___

**Sanity check:** if QPS < 1k and data < 1TB, this is a "small" system — don't over-engineer.

---

## 3. API  *(5 min)*

**Style:** ☐ REST &nbsp; ☐ gRPC &nbsp; ☐ GraphQL &nbsp; ☐ async/events

**Core entities:**
-

**Operations:**
| Verb | Path / Method | Request | Response | Notes (idempotency, auth) |
|---|---|---|---|---|
|  |  |  |  |  |
|  |  |  |  |  |

---

## 4. Data model  *(10 min)*

**Access patterns** (every query the system must serve — list ALL before picking storage):
1.
2.
3.
4.

**Storage choice:** ___ &nbsp; **Why:** ___ &nbsp; *(see decision-trees.md → storage)*

**Schema sketch:**
```
table/collection: ___
  field — type — notes
  ___
```

**Shard / partition key:** ___ &nbsp; cardinality: ___ &nbsp; skew risk: ___

**Indexes / secondary access:**
-

---

## 5. High-level architecture  *(5 min, ≤ 7 boxes)*

```
[ASCII diagram or describe boxes + arrows here]
```

**Read path:** ___ → ___ → ___

**Write path:** ___ → ___ → ___

---

## 6. DEEP DIVE — hardest component  *(15 min — main work)*

**Component:** ___

**Why hardest:** ___

**Alternatives considered:**
- A) ___ &nbsp; *rejected because* ___
- B) ___ &nbsp; *rejected because* ___
- **C) ___ &nbsp; chosen because** ___

**Algorithm / flow** (step by step):
1.
2.
3.

**Math:** capacity ___ &nbsp; latency ___ &nbsp; cost ___

---

## 7. Bottlenecks & failure modes  *(5 min)*

| Component | What breaks first | Mitigation |
|---|---|---|
|  |  |  |
|  |  |  |

**Cache:** where ___ &nbsp; what ___ &nbsp; invalidation ___

**Retry / timeout / idempotency:**
-

**What if [hardest component] dies?** ___

**What if DB dies?** ___ &nbsp; *(RPO ___ &nbsp; RTO ___)*

---

## 8. Observability  *(2 min)*

- **SLO:** ___ % of requests under ___ ms over ___ window
- **Top metrics:** RED (rate, errors, duration) on ___, ___, ___
- **Alerts on symptoms:** ___, ___, ___
- **Trace ID propagation:** ☐ yes (OpenTelemetry?)

---

## 9. Security  *(2 min)*

- AuthN: ___ &nbsp; AuthZ: ___
- PII / sensitive data: ___ &nbsp; encryption at rest: ___ &nbsp; in transit: ___
- **Top 3 threats:**
  1.
  2.
  3.

---

## 10. Cost  *(2 min)*

- **Dominant cost:** ☐ compute &nbsp; ☐ storage &nbsp; ☐ egress &nbsp; ☐ DB &nbsp; ☐ other: ___
- Rough estimate: $___ /month
- Biggest lever to reduce: ___

---

## 11. Trade-off log

- **Chose** ___ **over** ___ **because** ___
- **Chose** ___ **over** ___ **because** ___
- **Risk knowingly accepted:** ___
- **Revisit when:** *(trigger conditions)* ___

---

## Done-checklist

- [ ] Functional requirements (top 3–5)
- [ ] NFRs with numbers
- [ ] Scale estimate
- [ ] API contract
- [ ] Access patterns + storage choice + shard key
- [ ] Read path + write path drawn
- [ ] Deep dive on hardest component
- [ ] Bottleneck + failure mitigations
- [ ] Observability + SLO
- [ ] Security top 3
- [ ] Cost estimate
- [ ] Trade-off log with revisit triggers
