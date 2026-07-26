# Migration Triggers

Every component deferred in [pmvp.md](../pmvp.md) has a measurable condition for introduction, a simpler alternative that must first be shown insufficient, and a rollback.

**Rules.**
1. A trigger is a *measurement*, not a date and not an intuition.
2. Round numbers are derived, not chosen. Where a number is a judgement call, it says so.
3. Two conditions must usually both hold: a **pressure** (something is genuinely hurting) and a **readiness** (the new component will actually help). Introducing a read replica when replica lag exceeds the freshness budget makes things worse, not better.
4. Every trigger has a rollback. A component that cannot be removed should not be added.

---

## Database

### Read replica

**Introduce when both hold:**
- Sustained read CPU on the primary > 60% for 1 hour during normal (non-incident) load. *Derivation: 60% leaves headroom for the ×20 read burst in [workload-model.md](workload-model.md) before saturation.*
- Measured replication lag stays under **30 s**, the freshness budget for civic events.

**Simpler alternatives first:** query optimisation and index review; cache-aside; CDN on public event JSON. All three must be in place and still insufficient.

**Cost:** ~$300/mo at city scale. **Risk:** stale reads rendering a live closure as absent.

**Rollback:** point reads at the primary — one config change.

---

### Table partitioning

**Introduce when either holds:**
- `evidence_items` or `source_document_versions` exceeds **50 GB**. *Derivation: the point at which autovacuum and index maintenance on a single table begin to require a maintenance window longer than the availability budget permits.*
- Retention deletes take longer than 5 minutes or cause replication lag spikes.

Note this is a **maintenance** trigger, not a query-performance one — Postgres queries these tables fine well past 50 GB with correct indexes.

**Simpler alternatives first:** `BRIN` indexes on time columns; tuned autovacuum.

**Rollback:** partitioned tables cannot be trivially un-partitioned. This is close to one-way, which is why the trigger is conservative.

---

### Regional shards / separate clusters

**Introduce when any holds:**
- A data-residency mandate requires in-region storage.
- One regional cluster cannot meet capacity with vertical scaling exhausted.
- A requirement exists that one region's failure not affect another.

**Explicitly not triggered by size.** 4 TB (Scenario D) is one large cluster. Sharding buys isolation and residency, not throughput.

**Simpler alternatives first:** vertical scaling; read replicas; partitioning.

**Rollback:** consolidation is expensive. Treat as one-way.

---

## Streaming and queueing

### Durable event streaming (Kafka / Pulsar)

**Introduce when any holds:**
- **≥3 independent external consumers** require replay from an offset. *Derivation: with one or two, per-consumer polling against the API is simpler; at three the coordination cost inverts.*
- The Postgres command table is a measured bottleneck — claim contention or lock waits appear in the top-3 wait events during peak.
- Ingestion and projection require independent scaling, evidenced by one starving the other for more than the projection-lag SLO.
- Cross-region event propagation is required.

**Simpler alternatives first:** outbox + polling; separate worker pools; `SKIP LOCKED` tuning. These carry a system to a surprisingly large scale.

**Cost:** ~$5,000/mo plus meaningful operational burden.

**Critical constraint:** the broker is **never** the system of record. The outbox stays as the transactional boundary; the broker distributes.

**Rollback:** stop the producer; consumers fall back to API polling.

---

### Separate notification worker

**Introduce when:**
- A fan-out burst delays ingestion or projection beyond the projection-lag SLO (p95 15 s), measured during a real event.

*Reached at Scenario B — a 200,000-delivery burst measurably dominates shared worker capacity.*

**Simpler alternatives first:** a separate command type with a bounded claim budget within the existing pool.

**Rollback:** merge the pools.

---

## Search

### OpenSearch (or equivalent)

**Introduce when either holds:**
- Postgres FTS p95 breaches the search SLO on **production-shaped data** — real notice text, real query distribution, not synthetic.
- Search load measurably degrades transactional query latency (search queries appear in the top-3 by total time).

**Simpler alternatives first:** GIN index tuning; materialised search columns; result caching; moving search to the read replica. **The replica option usually resolves the second condition alone**, and should be tried first.

**Cost:** ~$1,200/mo at Scenario C.

**Constraint:** `DERIVED` and rebuildable (I21); never determines event truth.

**Rollback:** point search back at Postgres FTS with reduced functionality.

---

### Vector / semantic search

**Introduce when:**
- Moderators measurably fail to find relevant prior evidence with lexical search — tracked as a "no useful result" rate above 20% on moderator searches over 30 days.

**Constraint:** advisory recall only. No code path may let a similarity score influence event state.

**Rollback:** disable the field; lexical search is unaffected.

---

## Geospatial

### H3 / S2 coarse indexing

**Introduce when:**
- Route × event intersection p95 exceeds **500 ms** with a correctly-tuned GiST index **and** pre-simplified geometry already in place.

**Simpler alternatives first:** GiST tuning and `CLUSTER`; simplified geometry per zoom; bounding-box pre-filter; materialised active-event geometry.

*Not expected before Scenario C. At Scenario A (1,000 routes) GiST is sub-millisecond.*

**Constraint:** PostGIS remains authoritative. H3 is a candidate-discovery filter only, and the exact intersection always runs.

**Rollback:** drop the coarse index; queries return to pure PostGIS.

---

## Caching and delivery

### Redis cache-aside

**Introduce when:**
- p95 read latency breaches the SLO **and** profiling attributes >40% of read load to repeated identical queries.

**Constraint:** never authoritative (I10). Every accessor has a correct miss path, and the suite runs a mid-suite `FLUSHALL`.

**Rollback:** disable caching.

---

### CDN for API responses

**Introduce when:**
- Read traffic is geographically dispersed and cacheable within a bounded staleness budget (≤30 s for active events).

**Constraint:** **purge on every event transition**, issued by the committing worker with retries. A cached "road open" surviving a real closure is precisely the failure JanSetu exists to prevent — this makes purge a correctness requirement, not an optimisation.

**Rollback:** origin-only serving.

---

## Analytics

### ClickHouse (or equivalent)

**Introduce when either holds:**
- Analytical queries measurably degrade operational latency (>5% of primary CPU on analytical workloads).
- A public-research dataset obligation exists that the operational store cannot serve.

**Simpler alternatives first:** run analytics on the read replica; scheduled materialised views.

**Constraint:** no operational grant (I22), enforced by import-linter and by network policy at scale.

**Rollback:** revert to replica-based analytics.

---

## Deployment

### Kubernetes

**Introduce when either holds:**
- More than 3 deployables with genuinely independent scaling profiles.
- Multi-region deployment is required.

**Simpler alternatives first:** managed container platforms; process-type scaling on one platform.

**Rollback:** possible but disruptive.

---

### Service mesh

**Introduce when both hold:**
- More than 10 services.
- A compliance requirement mandates in-cluster mTLS.

*Not expected at any modelled scenario.*

---

### Service extraction from the monolith

**Extract a module when all three hold:**
1. Its resource profile diverges enough that co-scaling wastes **>30% of provisioned capacity**. *Derivation: below ~30%, the waste costs less than the operational overhead of another deployable.*
2. Its interface has been **stable for ≥2 releases** — an unstable boundary becomes a distributed-refactoring problem.
3. It has an independent failure domain — its failure should not require the rest of the system to fail.

**Expected order if it happens at all:** ingestion workers → notification → projection → search. Note that ingestion and notification are already separate *process types* from city scale; extraction here means separate *deployables with separate release cycles*, which is a further and rarely-necessary step.

---

## Authorisation

### Effective-permission cache

**Introduce when:**
- Authorisation is measured as a top-3 contributor to p95 API latency.

**Constraint:** short TTL; **revocation writes through synchronously**. The whole reason for deferral is that a revocation-latency window is unacceptable when a compromised node must be cut off immediately ([threat-model.md](../threat-model.md) T2).

**Rollback:** disable the cache.

---

## Trigger summary

| Component | Primary trigger | Expected scenario |
| --- | --- | --- |
| Read replica | Read CPU >60% for 1 h + lag <30 s | B |
| Cache-aside | p95 breach + >40% repeated reads | B |
| CDN (API) | Dispersed cacheable read traffic | B |
| Checkpointed replay | Full replay >10 min | **B (already reached at 42 min)** |
| Separate notification worker | Fan-out delays projection beyond SLO | B |
| Table partitioning | Table >50 GB or deletes >5 min | B |
| Object lifecycle | Bucket >1 TB | B |
| OpenSearch | FTS p95 breach on real data | C |
| H3/S2 | Route×event p95 >500 ms with GiST tuned | C |
| Regional shards | Residency mandate or regional capacity limit | C/D |
| Kafka/Pulsar | ≥3 consumers needing offsets | D |
| ClickHouse | Analytics degrading OLTP >5% CPU | D |
| Kubernetes | >3 deployables or multi-region | C/D |
| Service extraction | >30% waste + stable interface + failure domain | D |
| Permission cache | Authz in top-3 latency contributors | D |
| Service mesh | >10 services + mTLS mandate | Not expected |
| Blockchain | A threat signatures cannot address | **Never, currently** |

---

## Reverse triggers

Rarely written down, and worth writing down: a component should be **removed** when its trigger no longer holds and its cost exceeds its benefit.

| Component | Remove when |
| --- | --- |
| Redis cache | Hit ratio < 20% for 30 days |
| OpenSearch | Postgres FTS shown sufficient on current data shape |
| Read replica | Primary read CPU < 30% sustained |
| Vector search | Moderator no-useful-result rate unchanged versus lexical-only |
| A separate deployable | Its scaling profile has converged with the monolith's |

Architecture that only accretes becomes unoperable. Removal is a first-class operation, and reverse triggers make it a measurement rather than an argument.
