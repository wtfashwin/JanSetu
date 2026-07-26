# Workload Model

"Billions of records" is not a requirement. This document converts each scenario into record types, rates, sizes, and objectives, so that every architectural decision downstream can point at a number.

**Every figure here is a labelled assumption or a derivation from one.** None is measured. They exist to make architecture falsifiable: when production data arrives, these numbers get corrected and any decision that depended on a wrong one gets revisited.

---

## 0. Method

### Base units

| Symbol | Meaning | Value |
| --- | --- | --- |
| `Y` | seconds in a year | 31,557,600 |
| `A` | accumulation period for "total records" figures | **3 years** (assumption A0) |
| `3Y` | seconds in `A` | 94,672,800 |

**A0.** Scenario totals (e.g. "500 million evidence items") are read as *accumulated over three years of operation*, not as a per-day rate. Without this the totals have no rate interpretation at all. If the real accumulation period is shorter, every average rate scales up inversely.

### Core formulas

```
avg_rate            = total_records / (A × Y)
peak_rate           = avg_rate × burst_multiplier
daily_growth_bytes  = avg_rate × 86400 × avg_record_bytes
annual_growth_bytes = avg_rate × Y × avg_record_bytes
replay_seconds      = records_to_fold / fold_throughput_per_second
notification_burst  = subscriptions × affected_fraction
```

### Record sizes (assumption A1)

| Record | Avg size | Basis |
| --- | --- | --- |
| Raw source document (object storage) | **200 KB** | Municipal PDF/HTML notices; PDFs dominate the mean |
| `source_document_versions` row | 1.5 KB | Metadata + digests + provenance, bytes live in S3 |
| `evidence_items` row | 2 KB | Extracted fields, time window, location ref, provenance, parser/ruleset versions |
| `public_events` row (projection) | 4 KB | Denormalised for read, includes geometry reference and current state |
| `event_transitions` row | 0.8 KB | from/to state, cause, rule version, actor, timestamp |
| `community_reports` row | 1.2 KB | Category, coarse cell, pseudonymous reporter, optional media ref |
| `notification_deliveries` row | 0.3 KB | Subscription, event, transition, status |
| Geometry (LineString road segment) | 2–8 KB | Simplified at multiple zoom levels |

### Burst multipliers (assumption A2)

| Scenario | Ingestion burst | Read burst | Rationale |
| --- | --- | --- | --- |
| A | ×5 | ×10 | Single city, low absolute base |
| B | ×50 | ×20 | One flood event saturates one city's sources and its readership simultaneously |
| C | ×50 | ×20 | Regional disasters are correlated within a state, not across 100 cities |
| D | ×100 | ×50 | Cyclone or national emergency; many regions concurrently, plus press amplification |

The asymmetry is deliberate: **civic reads spike harder and faster than civic writes.** A road closure is written once and read a million times. Architecture must therefore protect read paths first.

### Fold throughput (assumption A3)

Projection fold throughput: **20,000 evidence rows/second per worker**, for a pure in-memory fold with batched reads on commodity hardware. Conservative; unvalidated. Recorded because replay time is a first-class SLO and this is the constant it hinges on.

### Access-pattern assumptions (A4)

- Daily active users: **30%** of registered (20% at Scenario A).
- Event views per active user per day: **4**.
- Saved routes per active user: **2**; route-impact evaluated on **each transition of an event intersecting that route**, not on a timer.
- Subscription → delivery fraction for a typical city event: **1%**; for a major regional event: **5%**.
- Read:write ratio overall: **~1000:1**.

---

## Scenario A — Developer / PMVP

Given: 1 city · 5 source adapters · 10 civic publishers · 10,000 source-document versions · 50,000 evidence items · 5,000 public events · 10,000 users · 1,000 saved routes.

Accumulation for A is taken as **6 months** (A0-a), not 3 years — a PMVP does not run for three years before being evaluated.

| Metric | Value | Derivation |
| --- | --- | --- |
| Source polls | 0.017 /s | 5 adapters ÷ 300 s poll interval |
| Document versions discovered | 6.4×10⁻⁴ /s avg · 0.003 /s peak | 10,000 ÷ 15.8 Ms; ×5 |
| Ingestion commands | 0.02 /s avg · 0.1 /s peak | polls + reparse |
| Evidence writes | 0.003 /s avg · 0.016 /s peak | 50,000 ÷ 15.8 Ms; ×5 |
| Event transitions | ~0.001 /s | 5,000 events × ~4 transitions ÷ 15.8 Ms |
| Citizen reads | 0.9 /s avg · 9 /s peak | 10,000 × 20% × 4 ÷ 86,400; ×10 |
| Geospatial impact calcs | ~0.002 /s | 1,000 routes × transitions intersecting |
| Notification decisions | ~0.02 /s peak | |
| Notification deliveries | ~500 in a burst | 10,000 users × 5% |
| Raw object storage | **2 GB** total | 10,000 × 200 KB |
| Postgres size | **~400 MB** | evidence 100 MB + events 20 MB + docs 15 MB + indexes/geometry |
| Daily growth | ~11 MB/day | |
| Replay (full) | **2.5 s** | 50,000 ÷ 20,000 |
| RPO | **0** | Single Postgres, synchronous commit |
| RTO | **1 hour** | Manual restore from managed backup |
| Availability target | 99.0% | ~7 h/month budget |
| Latency target | p95 event read < 300 ms; p95 route impact < 800 ms | |
| Cost drivers | One managed Postgres instance; object storage is negligible | |

**Architectural consequence.** Every number is trivial. A single application instance and a single Postgres instance are not merely sufficient — anything more is unjustifiable. Full replay in 2.5 seconds means projection rebuild can run in a request-response cycle during development, which is a significant productivity property worth preserving as long as possible.

---

## Scenario B — City scale

Given: 1 large city · 100 adapters · 500 publishers · 10M document versions · 50M evidence · 5M events · 5M users · 20M subscriptions. Accumulation `A` = 3 years.

| Metric | Avg | Peak (×50 ingest / ×20 read) | Derivation |
| --- | --- | --- | --- |
| Document versions | 0.106 /s | 5.3 /s | 10⁷ ÷ 94.67 Ms |
| Ingestion commands | 0.3 /s | 15 /s | versions + reparses + retries |
| Evidence writes | 0.53 /s | 26 /s | 5×10⁷ ÷ 94.67 Ms |
| Event transitions | 0.21 /s | 10.6 /s | 5×10⁶ × 4 ÷ 94.67 Ms |
| Citizen reads | **69 /s** | **1,390 /s** | 5×10⁶ × 30% × 4 ÷ 86,400 |
| Geospatial impact calcs | 3 /s | 300 /s | routes intersecting transitioning events |
| Notification decisions | 20 /s | 2,000 /s | |
| Notification deliveries | — | **200,000 per major event** | 2×10⁷ × 1% |
| Raw object storage | — | **2 TB** | 10⁷ × 200 KB |
| Postgres size | — | **~190 GB** | evidence 100 GB + events 20 GB + transitions 16 GB + versions 15 GB + indexes ~40 GB |
| Daily growth | 130 MB/day DB · 1.8 GB/day objects | | |
| Annual growth | 47 GB/yr DB · 670 GB/yr objects | | |
| Hot index (90 days of active/recent events) | ~8 GB | | must stay in RAM |
| Replay (full, 1 worker) | **42 min** | | 5×10⁷ ÷ 20,000 |
| Replay (from checkpoint, typical) | **< 60 s** | | last 24 h of evidence |
| RPO | **≤ 5 s** | | streaming replica + WAL archive |
| RTO | **≤ 15 min** | | replica promotion |
| Availability | 99.5% | | |
| Latency | p95 read < 200 ms; p95 route impact < 500 ms; p99 notification decision < 5 s | | |

**Architectural consequences.**

1. **The read path is the problem, not the write path.** 1,390 reads/s against 26 writes/s. This is what justifies a read replica and cache-aside — and nothing else.
2. **Full replay at 42 minutes exceeds any tolerable incident window.** Checkpointed incremental rebuild stops being an optimisation and becomes a requirement. This is the first hard trigger in the whole model.
3. **A 200,000-delivery burst dominates every other workload for its duration.** Notification fan-out must be throttled and separately queued so it cannot starve ingestion or reads.
4. Object storage at 2 TB needs a lifecycle policy; Postgres at 190 GB needs partitioning on `evidence_items` and `source_document_versions` for maintenance, not for query speed.

---

## Scenario C — Multi-city India

Given: 100 cities · 5,000 adapters · 100M document versions · 500M evidence · 50M events · 50M users · 250M subscriptions. `A` = 3 years.

| Metric | Avg | Peak | Derivation |
| --- | --- | --- | --- |
| Document versions | 1.06 /s | 53 /s | 10⁸ ÷ 94.67 Ms |
| Ingestion commands | 3 /s | 150 /s | |
| Evidence writes | **5.3 /s** | **264 /s** | 5×10⁸ ÷ 94.67 Ms |
| Event transitions | 2.1 /s | 106 /s | |
| Citizen reads | **694 /s** | **13,900 /s** | 5×10⁷ × 30% × 4 ÷ 86,400 |
| Geospatial impact calcs | 30 /s | 3,000 /s | |
| Notification decisions | 200 /s | 20,000 /s | |
| Notification deliveries | — | **12.5M per major regional event** | 2.5×10⁸ × 5% |
| Raw object storage | — | **20 TB** | |
| Postgres (all regions) | — | **~1.9 TB** | evidence 1 TB + events 200 GB + transitions 160 GB + versions 150 GB + indexes |
| Daily growth | 1.3 GB/day DB · 18 GB/day objects | | |
| Replay (full, 1 worker) | **6.9 hours** | | 5×10⁸ ÷ 20,000 |
| Replay (32-way partitioned) | **13 min** | | partitioned by city |
| RPO | **≤ 5 s** | | |
| RTO | **≤ 15 min** regional · **≤ 1 h** total | | |
| Availability | 99.9% | | |
| Latency | p95 read < 200 ms; p95 route impact < 500 ms | | |

**Architectural consequences.**

1. **Replay is now only tractable if it partitions.** 6.9 hours single-threaded versus 13 minutes across 32 city partitions. This is the concrete requirement that makes jurisdiction the primary partition key — not an aesthetic preference for "sharding by tenant".
2. 13,900 reads/s is the point at which cached, CDN-fronted public event JSON stops being optional.
3. A 12.5M-delivery fan-out at, say, 20k/s sustained takes **10 minutes** to drain. That latency must be *designed and disclosed*, not discovered. Prioritisation is by impact severity, not FIFO.
4. 1.9 TB with 100 cities is comfortably within one large Postgres cluster. **Sharding is still not justified by size** — only later by residency or regional failure isolation.

---

## Scenario D — National / federated

Modelled rather than given: 1×10⁹+ durable records · thousands of nodes · hundreds of millions of subscriptions · multi-region · long-term retention.

Assumption D1: 1×10⁹ evidence items and 4×10⁸ document versions accumulated over `A` = 3 years; 300M subscriptions.

| Metric | Avg | Normal peak (×100) | Disaster | Derivation |
| --- | --- | --- | --- | --- |
| Document versions | 4.2 /s | 420 /s | 800 /s | 4×10⁸ ÷ 94.67 Ms |
| Evidence writes | **10.6 /s** | **1,060 /s** | ~2,000 /s | 10⁹ ÷ 94.67 Ms |
| Event transitions | 4.2 /s | 420 /s | | |
| Citizen reads | ~1,000 /s | **50,000 /s** | 100,000 /s | ×50 burst |
| Geospatial impact calcs | 100 /s | 10,000 /s | | |
| Notification deliveries | — | **100M in one national event** | | 3×10⁸ × 33% |
| Sustained delivery rate needed | — | **27,800 /s** to drain 100M in 1 h | | |
| Raw object storage | — | **80 TB** at 3 yr, +50 TB/yr | | |
| Postgres (all regions) | — | **~4 TB** | | |
| Replay (full, 1 worker) | **13.9 hours** | | | 10⁹ ÷ 20,000 |
| Replay (128-way) | **6.5 min** | | | |
| RPO | **≤ 5 s** in-region · **≤ 60 s** cross-region | | | |
| RTO | **≤ 15 min** in-region · **≤ 4 h** region loss | | | |
| Availability | 99.95% read path · 99.9% write path | | | Read path matters more during disasters |
| Latency | p95 read < 200 ms in-region; p95 route impact < 500 ms | | | |

**Architectural consequences.**

1. **Write rates remain modest.** 1,060/s peak evidence writes is well within a single well-tuned Postgres primary. The scale story is *not* about write throughput.
2. **Read and fan-out dominate.** 50,000 reads/s and 27,800 deliveries/s are the actual scaling problems. Regional read replicas, CDN, and an independently-scaled notification tier — in that order.
3. **Multi-region exists for residency and failure isolation, not for throughput.**
4. **Storage growth (50 TB/yr) forces hot/warm/cold tiering** with lifecycle transitions to infrequent-access and archive classes.
5. **A 6.5-minute 128-way replay is what makes the audit story credible at national scale.** "Every event is reconstructable" is only true if reconstruction completes inside an incident window.

---

## Cross-scenario summary

| | A (PMVP) | B (city) | C (multi-city) | D (national) |
| --- | --- | --- | --- | --- |
| Peak evidence writes/s | 0.02 | 26 | 264 | 1,060 |
| Peak citizen reads/s | 9 | 1,390 | 13,900 | 50,000 |
| Peak notification burst | 500 | 200 K | 12.5 M | 100 M |
| Object storage | 2 GB | 2 TB | 20 TB | 80 TB |
| Postgres | 400 MB | 190 GB | 1.9 TB | 4 TB |
| Full replay (1 worker) | 2.5 s | 42 min | 6.9 h | 13.9 h |
| Partitioned replay | — | — | 13 min (32×) | 6.5 min (128×) |
| RPO | 0 | ≤5 s | ≤5 s | ≤5 s / ≤60 s |
| RTO | 1 h | 15 min | 15 min / 1 h | 15 min / 4 h |
| Availability | 99.0% | 99.5% | 99.9% | 99.95% read |

**The single most important observation in this document:** the read:write ratio grows from ~450:1 at PMVP to ~47:1 at Scenario D in *average* terms, but from 450:1 to 47,000:1 at *peak*, because reads burst ×50 while writes burst ×100 from a base three orders of magnitude smaller. JanSetu is a read-amplification system with a correctness-critical write path. Optimise the write path for **correctness**, and the read path for **capacity**. Any design that inverts this is wrong.

---

## Estimated monthly infrastructure cost

Order-of-magnitude, managed services, INR at ₹85/USD (assumption A5). Excludes people, LLM inference, and SMS/push provider fees, which at Scenario C–D likely exceed all infrastructure combined.

| | A | B | C | D |
| --- | --- | --- | --- | --- |
| Compute (app + workers) | $30 | $400 | $3,000 | $20,000 |
| PostgreSQL (primary + replicas) | $25 | $600 | $4,000 | $25,000 |
| Object storage + egress | $2 | $80 | $700 | $3,500 |
| Cache / Redis | $10 | $80 | $500 | $2,500 |
| CDN | $0 | $50 | $600 | $4,000 |
| Search (OpenSearch) | $0 | $0 | $1,200 | $6,000 |
| Analytics (ClickHouse) | $0 | $0 | $0 | $4,000 |
| Streaming (Kafka/Pulsar) | $0 | $0 | $0 | $5,000 |
| Observability | $0 (self-host) | $150 | $900 | $5,000 |
| **Monthly total (USD)** | **~$70** | **~$1,360** | **~$10,900** | **~$75,000** |
| **Monthly total (INR)** | **~₹6 K** | **~₹116 K** | **~₹927 K** | **~₹6.4 M** |

**Notification delivery is the sleeper cost.** At Scenario C, 12.5M pushes per major event is free-ish over FCM but catastrophic over SMS. Any SMS channel needs a hard budget cap and severity gating, decided as product policy before it is built.

---

## What would falsify this model

Stated so the model can be checked rather than believed:

1. If average raw document size is 2 MB rather than 200 KB (scanned-image notices are common), object storage costs and ingestion bandwidth rise 10×.
2. If accumulation periods are 1 year rather than 3, all average rates triple.
3. If fold throughput is 2,000 rows/s rather than 20,000, Scenario C partitioned replay goes from 13 minutes to 2.2 hours and the incremental-checkpoint requirement arrives at Scenario B instead of C.
4. If DAU is 60% rather than 30% (plausible for a disaster-driven product with genuine daily utility), all read figures double.
5. If the affected-subscription fraction is 20% rather than 5% during regional events, fan-out becomes the dominant cost at Scenario B rather than C.

Each of these is measurable within weeks of a real pilot. [migration-triggers.md](migration-triggers.md) is written so that being wrong about any of them changes *when* a component is introduced, not *whether* the architecture is sound.
