# Pattern Applicability Matrix

## Scope

This is an architecture review checklist, not a menu. **No pattern is adopted because it appears on a list.** Each accepted pattern below is tied to a specific JanSetu requirement and a measurable introduction trigger; each rejected one records why.

---

## Matrix

| Pattern | JanSetu use case | PMVP | Future | Reject | Introduction trigger | Main risk |
| --- | --- | :-: | :-: | :-: | --- | --- |
| Universal web architecture (client → LB → app → DB) | Citizen PWA, publisher console, public API | **P** | — | — | Day one | None; this is the baseline |
| Three-tier architecture | Presentation / application / data separation inside the monolith | **P** | — | — | Day one | Boundary erosion without import linting |
| Modular monolith | All 14 PMVP modules in one deployable | **P** | — | — | Day one | Modules coupling through shared tables |
| Microservices | Independent scaling of ingestion, projection, notification | — | **C/N** | — | A single module's resource profile diverges enough that co-scaling wastes >30% of provisioned capacity, **and** its interface has been stable for ≥2 releases | Distributed transactions; 3 engineers cannot run 12 services during a flood |
| Cache-aside | Hot event reads; jurisdiction lookups; source-health summary | — | **C** | — | p95 read latency breaches SLO **and** profiling attributes >40% to repeated identical reads | Stale civic data displayed as current — must carry `as_of` and honour `STALE` |
| Database replication (read replicas) | Citizen read traffic during a major incident | — | **C** | — | Sustained read CPU on primary >60% for 1h, **and** replica lag stays inside the freshness budget (≤30 s) | Reading a not-yet-replicated event and rendering "not found" for a live closure |
| Database sharding | Regional/jurisdictional data separation, residency | — | **N** | — | One regional cluster cannot meet capacity or a residency mandate requires in-region storage | Cross-shard event merges; operational complexity |
| Consistent hashing | Distributing ingestion work across workers | — | **N** | — | Worker pool >20 nodes **and** rebalancing cost measured as material | Premature. `FOR UPDATE SKIP LOCKED` covers this to a very large scale |
| Async message processing | Every ingestion, parse, projection, notification effect | **P** | — | — | Day one, as a Postgres outbox + worker | Poison messages blocking a queue without a dead-letter path |
| Event-driven processing | Evidence → event projection; federation fan-out | **P** | — | — | Day one, in-process | Hidden causality; needs correlation ids in traces |
| Circuit breaker | Failing or rate-limiting official source websites | **P** | — | — | Day one — sources *will* be flaky | An open breaker must surface as `STALE` to citizens, not as silence |
| Retry with exponential backoff + jitter | Source fetch, parser, notification delivery | **P** | — | — | Day one | Retry storms against a struggling government site — cap attempts and honour breaker state |
| Rate limiting | Public API, community reports, notification sends | **P** | — | — | Day one | Rate-limiting real reporters during a genuine disaster; needs a trusted-reporter lane |
| Real-time push / WebSockets | Live event updates on an open event page | — | **C** | — | Measured poll traffic on active event pages exceeds the cost of holding connections | Connection storms during a disaster — the exact moment it is needed most |
| Publish / subscribe | Third-party consumers (navigation, delivery, transport ops) | — | **C/N** | — | ≥3 external consumers request push delivery, **or** any consumer needs replay from an offset | Fan-out amplification; per-subscriber backpressure |
| CDN | Static assets; cached public event JSON; tiles | **P** (assets) | **C** (API) | — | Assets: day one. API caching: when read traffic is geographically dispersed and cacheable with a bounded staleness budget | Cached stale closure surviving a resolution — needs short TTL + purge on transition |
| Read/write separation | Citizen reads vs ingestion writes | — | **C** | — | Same trigger as read replicas | Application must tolerate reading its own stale write |
| Distributed locks / leases | Single-writer guarantee for projection rebuild; scheduled source polls | **P** | — | — | Day one, as Postgres advisory locks | Lease expiry mid-work causing two projectors — projector must be idempotent regardless |
| Saga / compensation | Multi-step flows spanning stores (object storage + DB) | **P** (light) | — | — | Day one, in the narrow form: object write first, DB commit second, orphan-sweeper for unreferenced objects | Full orchestrated sagas are unjustified; the only real cross-store case is S3+Postgres |
| CQRS | Separate write model (evidence) from read model (public events) | **P** | — | — | Day one — this is JanSetu's core shape, not an optimisation | Projection lag presented as truth; needs `projection_lag_seconds` and an `as_of` on every response |
| Event sourcing | Append-only evidence and transitions as the source of truth | **P** (domain-scoped) | — | — | Day one, for evidence and event state **only** | Full event-sourcing of every entity is over-engineering; identity, orgs and nodes stay ordinary CRUD |
| Blockchain / DLT | Tamper-evident federation | — | — | **✗** | Never, absent a threat model showing signatures + append-only audit are insufficient | Enormous cost, no additional guarantee over signed envelopes; see [threat-model.md](threat-model.md) |
| Service mesh | Inter-service mTLS, retries, observability | — | **N** | — | >10 services **and** a compliance requirement for in-cluster mTLS | Operational burden vastly exceeding benefit before that point |
| Kubernetes | Orchestration | — | **C/N** | — | >3 distinct deployables with independent scaling, or multi-region | A single container on a managed host is correct for PMVP |
| Search engine (OpenSearch) | Moderator evidence search; historic event search | — | **C** | — | Postgres FTS p95 breaches the search SLO on production-shaped data, **or** search load measurably degrades transactional queries | A second store to keep consistent; must be rebuildable and must never determine event truth |
| Vector / semantic search | Similar-event discovery; moderator recall | — | **C** | — | Moderators measurably fail to find relevant prior evidence with lexical search | **Must never determine event truth.** Advisory recall only |
| Analytical store (ClickHouse) | Public dashboards, research datasets | — | **N** | — | Analytical queries measurably degrade the operational database | Analytics becoming operational truth — forbidden by invariant 22 |
| Geospatial coarse indexing (H3/S2) | Candidate discovery before exact PostGIS intersection | — | **C** | — | PostGIS route×event intersection p95 breaches SLO with a correctly-tuned GiST index | Two spatial models to keep consistent; PostGIS remains authoritative |

---

## Required justification for each accepted pattern

Only patterns accepted for **PMVP** are justified in full here; deferred patterns carry their trigger in [migration-triggers.md](scaling/migration-triggers.md).

### Async processing via a Postgres outbox

1. **Problem solved.** Source fetching, parsing, projection and notification are slow, failure-prone, and must survive process death without losing an acknowledged command.
2. **Measured condition.** None required — this is a correctness requirement (invariant 11), not a performance one.
3. **Simplest alternative.** Background threads. **Rejected on evidence:** the Cerevra audit found exactly this pattern losing work permanently and silently (`cerevra/ingestion/handler.py:411-449` — daemon thread, two in-memory retries, `logger.error`, no durable record).
4. **Operational cost.** One extra table, one worker loop, a pruning job.
5. **Removal path.** If a broker is later introduced, the outbox becomes its relay source. The pattern survives the migration; it does not need removing.
6. **Failure modes.** Poison rows blocking progress (mitigated by attempt counts and a dead-letter state); relay lag (mitigated by `projection_lag_seconds`); unbounded table growth (mitigated by retention).
7. **Testing.** Kill the worker mid-claim and assert the row is re-claimed and the effect applied exactly once observably; property-test that N deliveries of one command yield one effect.

### CQRS + domain-scoped event sourcing

1. **Problem solved.** "Why is this event `OFFICIALLY_CONFIRMED`?" must be answerable and reproducible.
2. **Measured condition.** None — product invariants 4, 10, 15.
3. **Simplest alternative.** A mutable `events` table with a status column. Rejected: it destroys the decision record, and correction becomes indistinguishable from tampering.
4. **Operational cost.** Two structures per entity; a projection worker; rebuild tooling.
5. **Removal path.** None wanted. If projection cost ever dominates, snapshot the projection at checkpoints rather than abandoning the model.
6. **Failure modes.** Projection lag rendered as truth; projector non-determinism; unbounded history.
7. **Testing.** Replay-equivalence property test (same inputs → identical projection hash); divergence detection asserted by a dedicated metric and alert.

### Circuit breaker + backoff-with-jitter on source adapters

1. **Problem solved.** Government and utility websites are frequently slow, rate-limited, or down, especially during the disasters JanSetu exists to report.
2. **Measured condition.** None — this is a known property of the source population.
3. **Simplest alternative.** Fixed-interval polling with a timeout. Rejected: it hammers a struggling source and hides the outage.
4. **Operational cost.** Per-source breaker state and health rows.
5. **Removal path.** N/A.
6. **Failure modes.** An open breaker silently freezing an event's evidence. **Mitigation is a product requirement:** breaker state drives `source_health`, which drives the `STALE` / `STATUS_UNKNOWN` presentation. A failing source must be visible to citizens, not merely to operators (invariant 15).
7. **Testing.** Chaos test: source returns 503 / hangs / returns malformed HTML; assert breaker opens, health degrades, affected events render `STALE`, and no event silently resolves.

### Rate limiting

1. **Problem solved.** Public API abuse, community-report brigading, notification spam.
2. **Measured condition.** None — day one, because the abuse surface exists from the first public deploy.
3. **Simplest alternative.** None acceptable.
4. **Operational cost.** Redis counters; a fallback to in-process limits if Redis is unavailable (Redis is never authoritative).
5. **Removal path.** N/A.
6. **Failure modes.** Suppressing genuine reports during a real disaster — the highest-consequence failure in this document. Mitigation: per-category limits, a trusted-reporter lane, and a documented disaster mode that raises report limits while *tightening* trust rules rather than loosening them (invariant 24).
7. **Testing.** Load test the disaster-spike scenario; assert genuine reports are accepted and trust rules are not bypassed.

### Distributed locks (Postgres advisory locks)

1. **Problem solved.** Two projectors or two scheduled pollers doing the same work.
2. **Measured condition.** None — arises as soon as there is more than one worker process.
3. **Simplest alternative.** A single worker process. Acceptable at PMVP, but the lock is cheap and removes a footgun.
4. **Operational cost.** Negligible; no new infrastructure.
5. **Removal path.** Replaced by partition assignment if durable streaming is introduced.
6. **Failure modes.** Lock lost mid-work. **Mitigation: the projector is idempotent regardless of the lock** — the lock is an optimisation, never a correctness dependency. This is the discipline the Cerevra audit found missing, where per-process `threading.Lock` was the only guard and two processes would corrupt shared state.
7. **Testing.** Run two projectors concurrently against the same event and assert one projection and one transition sequence.

### Narrow saga (object storage + database)

1. **Problem solved.** Raw source documents live in S3; their metadata lives in Postgres. There is no shared transaction.
2. **Measured condition.** None — structural.
3. **Simplest alternative.** Store document bytes in Postgres. Viable at PMVP scale and genuinely tempting, but rejected because invariant 7 (raw documents immutable) plus Scenario C storage growth make object storage the right long-term home, and migrating later is more expensive than doing it now.
4. **Operational cost.** An orphan-object sweeper.
5. **Removal path.** N/A.
6. **Failure modes.** Object written, DB commit fails → orphan object (harmless, swept). DB committed, object missing → **dangling reference, which is harmful.** Ordering is therefore fixed: write and verify the object *first*, then commit the row referencing it.
7. **Testing.** Chaos test: kill between the object write and the commit; assert no dangling reference and that the orphan is swept.

---

## Explicitly rejected, with reasons

| Pattern | Why rejected |
| --- | --- |
| Blockchain / DLT | Provides no guarantee that signed envelopes plus an append-only audit log do not already provide. Cost and operational burden are extreme. Revisit only if [threat-model.md](threat-model.md) is amended with a threat that signatures demonstrably cannot address. |
| Entitlement / licensing gates | JanSetu is public civic-safety infrastructure. Gating this information by entitlement is a product failure, not an architectural option. |
| Vector search as event truth | An embedding similarity score is not evidence. It may aid moderator recall and nothing else. |
| Microservices at PMVP | Three engineers must be able to operate the system during a flood at 02:00. |
| Service mesh, Kubernetes, Kafka, OpenSearch, ClickHouse at PMVP | Each protects no invariant and satisfies no measured requirement at PMVP scale. All carry triggers in [migration-triggers.md](scaling/migration-triggers.md). |
