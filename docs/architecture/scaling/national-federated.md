# JanSetu National / Federated Architecture

Target workload ([workload-model.md](workload-model.md), Scenarios C–D): 1,060 evidence writes/s peak, **50,000 citizen reads/s peak**, 100 M-delivery notification events, 80 TB objects, 4 TB Postgres, 1 B+ durable records, thousands of independently-operated civic nodes, multi-region with residency constraints.

**The central finding of the workload model applies here:** write throughput at national scale (1,060/s peak) is still within a single well-tuned Postgres primary. Reads (50,000/s) and notification fan-out (27,800/s sustained) are the actual scaling problems, together with residency and failure isolation. This architecture is shaped by that asymmetry, not by record count.

---

## 1. Shape

```
                     Edge / CDN (per region)
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
     ┌──── Region: West ────┐        ┌──── Region: North ────┐
     │  api replicas         │        │  api replicas          │
     │  read replicas        │        │  read replicas         │
     │  ingestion workers    │        │  ingestion workers     │
     │  projection workers   │        │  projection workers    │
     │  notification workers │        │  notification workers  │
     │  PG cluster (cities)  │        │  PG cluster (cities)   │
     │  S3 (regional)        │        │  S3 (regional)         │
     │  OpenSearch           │        │  OpenSearch            │
     └───────────┬───────────┘        └───────────┬────────────┘
                 │                                │
                 └──────────┬─────────────────────┘
                            ▼
              Durable event stream (Kafka/Pulsar)
              partitioned by (region, city)
                            │
              ┌─────────────┼──────────────┐
              ▼             ▼              ▼
      Control plane   ClickHouse    Federation gateway
      (global, small)  (analytics)   (signed envelopes)
```

---

## 2. Control plane / data plane separation

At PMVP and city scale the separation is **logical only** — different modules, one deployable. Here it becomes **physical**, for one reason: a global control plane serving thousands of nodes must not be able to take the data plane down.

| Control plane (global, small, low write rate) | Data plane (regional, high volume) |
| --- | --- |
| Organisation and civic-node registration | Source ingestion |
| Node identity and public signing keys | Raw document storage |
| Permissions and jurisdiction assignment | Parsing and evidence generation |
| Authorised event categories | Event projection |
| Schema and protocol versions | Event querying |
| Source-adapter configuration | Geospatial impact |
| Federation metadata and trust levels | Community reports |
| Retention policy | Notification decisions and delivery |
| Feature flags | |

**Control-plane failure must not stop the data plane.** Every data-plane process caches control-plane policy locally with a TTL and a *last-known-good* fallback that never expires into a permissive state. If the control plane is unreachable:

- ingestion, projection, querying and delivery continue on cached policy;
- **no new node may be registered, no key rotated, no permission granted** — the control plane is the only authority for those;
- a **revocation** is the exception that must not be cached away. Revocations are pushed over the event stream and applied immediately; if the stream is also unreachable, cached-policy TTL for signing keys is deliberately short (5 minutes), trading availability for the ability to cut off a compromised node quickly. That trade is stated here because it is the one place this design chooses consistency over availability.

---

## 3. Regional sharding

**Trigger.** A residency mandate, or one regional cluster unable to meet capacity, or a requirement that one region's failure not affect another.

**Not** triggered by size. 4 TB is one large cluster; sharding here buys isolation and residency, not throughput.

Shard key: **city/jurisdiction**, derived from the jurisdiction tree. This choice follows directly from the replay requirement — partitioned replay (128-way, 6.5 minutes vs 13.9 hours single-threaded) is only possible because evidence partitions cleanly by jurisdiction. Sharding by time or by source would not give that.

Cross-shard cases and their handling:

| Case | Handling |
| --- | --- |
| Event spanning two cities | Owned by the jurisdiction of the issuing node; the neighbouring shard holds a federated reference, not a copy |
| Citizen route crossing shards | Route impact fans out to each shard and merges client-side; per-shard results carry their own `as_of` |
| National-level source (e.g. central weather) | Owned by a national shard; regions subscribe |
| Event merge across shards | Not permitted. Cross-shard relationships are expressed as federation links, preserving both histories (I18) |

---

## 4. Durable event streaming

**Trigger.** Three or more independent consumers requiring replay from an offset, **or** ingestion and projection requiring independent scaling with measured contention, **or** cross-region propagation of civic events.

Selected: Kafka or Pulsar (either; the decision is deferred to an operational preference at the time).

| | |
| --- | --- |
| **Requirement** | Cross-region propagation; external consumer replay; decoupling projection from ingestion |
| **Simpler alternative** | Continue with the Postgres outbox and per-consumer polling. Genuinely viable to a surprisingly large scale, and rejected only when external consumers need offsets the database cannot cheaply provide |
| **Rejected alternatives** | Webhooks only (no replay); shared database access for consumers (couples every consumer to the schema) |
| **Operational burden** | High. A broker is a stateful distributed system requiring its own on-call expertise |
| **Failure modes** | Partition skew; consumer lag; poison messages; rebalancing storms; the broker becoming an unintended source of truth |
| **Cost** | ~$5,000/month at Scenario D |
| **Migration path** | The outbox relay becomes a producer. The outbox itself **stays** — it remains the transactional boundary. The broker is a distribution mechanism, never the durable record |
| **Rollback** | Stop the producer; consumers fall back to polling the API |

**The critical rule:** the broker is never the system of record. Postgres remains authoritative; the stream is a distribution channel that can be rebuilt from it. This is what keeps invariant 15 ("an event can be rebuilt from append-only evidence and a versioned ruleset") true at national scale.

---

## 5. Search infrastructure

**Trigger.** Postgres FTS p95 breaching the search SLO on production-shaped data, **or** search load measurably degrading transactional queries.

Selected: OpenSearch, regional, `DERIVED` and fully rebuildable (I21).

- Citizen event search, moderator evidence search, historic search.
- Optional dense-vector field for similar-event recall.
- **Neither lexical nor vector search may determine event truth.** Search returns candidates; the projection determines state. Enforced by module boundary — `public_events` has no dependency on the search client.

Rebuild is exercised regularly, not merely documented, because an index that has never been rebuilt is an index that cannot be.

**Rollback.** Point search back at Postgres FTS with reduced functionality.

---

## 6. Analytical store

**Trigger.** Analytical queries measurably degrading the operational database, or a public-research dataset obligation.

Selected: ClickHouse, ETL-loaded from the stream.

**Invariant 22 is enforced structurally:** the operational application has no client library for the analytical store, verified by an import-linter rule and, at this scale, by network policy. Analytics can be wrong, late, or down without any operational consequence. It must never become a source of truth for a civic event.

---

## 7. Notification fan-out

The dominant workload at Scenario D: 100 M deliveries in a national event, requiring ~27,800/s sustained to drain in one hour.

```
event transition → stream partition (region, city)
                 → decision workers (regional, horizontally scaled)
                 → severity-prioritised delivery queues
                 → provider adapters (push / webhook / SMS)
```

- **Prioritised by impact severity, never FIFO.** A route-affected citizen outranks a general area subscriber.
- Per-user, per-event, per-day caps (I9, I23) throughout.
- Provider-level circuit breakers and per-provider throttles.
- **Delivery latency at full fan-out is ~10 minutes at Scenario C and ~1 hour at Scenario D. This is designed and disclosed, not discovered.** Citizens are told when an alert was decided and when it was sent.
- SMS requires a hard budget cap and severity gating, settled as product policy before the channel exists.

---

## 8. Federation

Each civic node publishes signed envelopes containing: node identity, jurisdiction, authorised categories, protocol version, event payload, evidence references, issued time, and an update/expiry policy.

- Envelopes are stored **verbatim, immutable**, with signature verification on ingest and again on read (I20).
- A node may publish supporting evidence, contradictory evidence, a correction request, a derived local event, or a status note about another node's event. It may never mutate one.
- Key rotation is overlapping-validity; revocation is immediate and pushed over the stream (§2).
- Trust levels gate what a node's events may achieve: only a node authorised for a jurisdiction and category can produce `OFFICIALLY_CONFIRMED` there (I3).
- Replay protection via envelope nonce plus issued-time window.
- Node suspension halts ingest of new envelopes; **previously published events remain, with their signatures intact and their source's suspension visible**. Retroactively deleting a suspended node's history would itself be a form of silent rewriting.

**No blockchain.** [threat-model.md](../threat-model.md) shows signed envelopes plus an append-only audit log address every identified federation threat. A distributed ledger would add substantial cost and operational burden with no additional guarantee.

Full detail: [security-and-federation.md](../security-and-federation.md).

---

## 9. Multi-region disaster recovery

| Objective | Value |
| --- | --- |
| RPO in-region | ≤ 5 s (streaming replication) |
| RPO cross-region | ≤ 60 s (async replication + WAL shipping) |
| RTO in-region | ≤ 15 min (replica promotion) |
| RTO region loss | ≤ 4 h (promote in a surviving region, restore objects from cross-region replication) |
| Availability | 99.95% read path, 99.9% write path |

The read path carries the higher target deliberately: during a disaster, the ability to *read* a road closure matters more than the ability to publish a new one.

Region loss is rehearsed at least twice yearly. Objects are cross-region replicated; the control plane is globally replicated; each regional data plane is independently promotable.

---

## 10. Hot / warm / cold at scale

| Tier | Contents | Store | Access |
| --- | --- | --- | --- |
| Hot | Active events, evidence < 30 d, current source health, pending notifications | Postgres + cache + CDN | ms |
| Warm | Resolved events < 1 y, moderation history, recent documents | Postgres partitions + OpenSearch | 10s of ms |
| Cold | Historic document versions, archived evidence, audit > 1 y, replay snapshots | S3 Glacier-class + Parquet exports | hours |

Movement is by partition detach and archive. Restoration from cold for a replay or an audit is a documented, tested procedure with a stated latency, not an aspiration.

---

## 11. Technology decisions, consolidated

| Technology | Requirement | Trigger | Simpler alternative rejected | Operational burden | Cost/mo (D) |
| --- | --- | --- | --- | --- | --- |
| Regional PG clusters | Residency, failure isolation | Residency mandate or regional capacity limit | One global cluster | Medium | $25 K |
| PostGIS | Exact geometry — authoritative throughout | Day one | Bounding boxes only (wrong answers) | Low | incl. |
| Kafka / Pulsar | External consumer replay, cross-region propagation | ≥3 consumers needing offsets | Postgres outbox + polling | **High** | $5 K |
| OpenSearch | Search at scale | FTS p95 breach | Postgres FTS | Medium | $6 K |
| ClickHouse | Analytics isolation | Analytics degrading OLTP | Read replica | Medium | $4 K |
| Redis | Cache, rate limits | Read latency | In-process cache | Low | $2.5 K |
| S3 + lifecycle | Immutable raw docs | Day one | Postgres large objects | Low | $3.5 K |
| Kubernetes | Multi-region orchestration | >3 deployables, multi-region | Managed containers | High | incl. |
| CDN | Read burst absorption | 50 K reads/s | Origin only | Low | $4 K |
| OTel + Prometheus + Grafana | Observability | Day one | Logs only | Medium | $5 K |

**Not adopted:** blockchain, service mesh (until >10 services *and* an in-cluster mTLS mandate), Citus (regional clusters are simpler than a distributed extension for the same isolation goal), Cassandra (no workload requiring it — writes are modest and relational integrity is load-bearing).

---

## 12. What can still go wrong

Honest list, carried into [adversarial-review.md](../reviews/adversarial-review.md):

1. **The broker becomes the de facto source of truth** because it is convenient. This is the most likely architectural decay path. Mitigation: rebuild-from-Postgres is exercised continuously, not documented once.
2. **Notification fan-out latency during a national emergency** — one hour to drain 100 M is a genuine product limitation, not a bug to be fixed later. It must be disclosed to citizens and it must drive severity prioritisation.
3. **Cross-shard event semantics** are the hardest correctness surface. A cyclone spanning six states does not respect the shard key.
4. **Federation trust is a social problem with a cryptographic component.** Signatures prove a node said something; they cannot prove the node is honest. Node suspension, trust levels, and visible provenance are the real controls.
5. **Operating this requires a team, not three engineers.** That is the honest cost of Scenario D, and it is the reason every earlier document works so hard to defer arriving here.
