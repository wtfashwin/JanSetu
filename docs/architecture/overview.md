# Executive Architecture Summary

Answers to the fourteen questions the program was required to close. Every claim traces to a document in `docs/architecture/` or `docs/adr/`, and every Cerevra claim traces to a file and line in the audited repository.

---

## 1. What Cerevra capabilities are genuinely proven?

Five, established by source reading plus a local test run — not by README claims.

1. **WAL-before-mutation ordering with fsync**, enforced at call sites (`cerevra/ingestion/handler.py:188-208`, `cerevra/wal/writer.py:47-60`), backed by real filesystem crash tests.
2. **Deterministic replay as a pure fold into a projection** (`cerevra/wal/replay.py:146-306`).
3. **Refusal as a first-class outcome** (`cerevra/query/cqs.py` — `CQSDecision.REFUSE`, ordered-threshold invariant).
4. **Runtime invariants that raise rather than log** (`cerevra/query/invariants.py:39-77`).
5. **Corruption modelled as data, not exception** (`cerevra/wal/scanner.py:86-95`).

Cerevra is Apache-2.0 and authored by a JanSetu maintainer, so reuse is legally clear. The constraint is technical, not licensing.

## 2. What can be reused directly?

Three things, as patterns rather than as imported code:

- Raising invariant assertions on hot paths.
- The `what` / `why` / `outcome` / `safe` structured-log convention — `safe=true` marks a deliberate refusal, which must never look like a failure on a dashboard.
- Property-based testing of replay invariants.

## 3. What should become a standalone library?

- Length-prefixed CRC32 record framing plus the corruption-tolerant streaming scanner — useful if JanSetu later needs a file-based archive or federation transfer format. SHA-256 replaces CRC32 where tamper-evidence rather than corruption detection is required.
- One circuit-breaker implementation. Cerevra currently ships two.

## 4. What must be rewritten?

Fingerprinting and deduplication (five distinct keys, full-width hashes); the namespace/jurisdiction model; sequence allocation; snapshots and checkpointing; packaging; SDKs.

**Not used at all:** `CrashRecoveryEngine` (it reads the legacy JSON WAL while ingestion writes the binary one — `cerevra/wal/recovery.py:17`), the legacy `WriteAheadLog` (rewrites log files in truncating mode — `wal/wal.py:216-255`), `AdaptiveRanker.build_window` (cannot execute — `query/ranker.py:187-191`), the pickled BM25 store, `distributed/`, `sharding/`, `billing/`, and the bespoke metrics and tracing.

**Decision (ADR 0006): no runtime dependency on the `cerevra` package.** The published wheel is not importable (`pyproject.toml:13-30` omits five imported runtime dependencies), the recovery engine does not cover the ingestion path, and the namespace model cannot express a jurisdiction. JanSetu's ingestion log becomes a Postgres table, which removes those hazard classes by construction.

## 5. Which general engineering patterns does the architecture adopt?

Thirteen, all standard practice and each justified independently by a JanSetu requirement rather than by precedent: modular monolith with enforced dependency direction; append-only transitions with a derived projection; transactional outbox; row-level tenant isolation; atomic permissions with a policy decision point; versioned configuration authority (scoped to trust rulesets); contract-first APIs with an error registry; required entity fields with an explicit history type per table; ordered forward-only migrations; **no fabricated fallback data**; structured logging with intent fields; cost-tiered test layout; product-neutral design system with an accessibility gate.

Deferred: effective-permission caching — it creates a revocation-latency window, and cutting off a compromised node immediately matters more than saving microseconds. Rejected: entitlement or licensing gates — gating civic-safety information is a product failure, not an architectural option.

The full evaluation, including the patterns considered and rejected, is in [pattern-applicability.md](pattern-applicability.md).

## 6. What is the simplest correct PMVP?

```
Next.js PWA → FastAPI modular monolith → PostgreSQL 16 + PostGIS → S3 (immutable raw documents)
                                       ↘ Redis (non-authoritative)
```

One container image, two process types (`api`, `worker`), one managed database, one bucket, one small cache. ~$70/month.

The engineering difficulty is not scale — Scenario A is 0.02 commands/s and 9 reads/s. It is correctness: an ingestion log that is a database table (so acknowledgement *is* the commit), append-only evidence, a pure-fold projection with a stored hash so divergence is detectable, deterministic versioned trust rules, and `OFFICIALLY_CONFIRMED` unreachable without official evidence by database constraint.

## 7. What is the city-scale evolution?

Same image, five process types. Read replica, cache-aside, CDN, checkpointed incremental replay, a separate notification worker and queue, monthly partitioning, object-storage lifecycle. **The monolith is not split.** ~$1,360/month.

Two numbers force all of it: 1,390 reads/s against 26 writes/s, and a 42-minute full replay.

## 8. What is the credible billion-record architecture?

Regional Postgres clusters sharded by jurisdiction; durable streaming for external consumer replay and cross-region propagation; OpenSearch; ClickHouse; CDN; Kubernetes; a global control plane with regionally-cached policy; signed federation envelopes. ~$75,000/month.

**It is credible because the arithmetic is written down.** 1 B evidence items over three years is 10.6 writes/s average and ~1,060/s peak — inside a single tuned primary. The actual scaling problems are 50,000 reads/s, a 27,800/s sustained notification drain, residency, and failure isolation. Jurisdiction is the shard key because it is what makes 128-way partitioned replay possible: **6.5 minutes instead of 13.9 hours.**

## 9. What migration thresholds connect the three?

| Component | Trigger | Scenario |
| --- | --- | --- |
| Checkpointed replay | Full replay > 10 min | **B (already 42 min)** |
| Read replica | Read CPU > 60% for 1 h **and** lag < 30 s | B |
| Cache / CDN | p95 breach **and** > 40% repeated reads | B |
| Separate notification worker | Fan-out delays projection past SLO | B |
| Partitioning | Table > 50 GB or deletes > 5 min | B |
| OpenSearch | FTS p95 breach on production-shaped data | C |
| H3/S2 | Route×event p95 > 500 ms with GiST tuned | C |
| Regional shards | Residency mandate or regional capacity limit | C/D |
| Kafka/Pulsar | ≥3 consumers needing offset replay | D |
| ClickHouse | Analytics > 5% of primary CPU | D |
| Service extraction | > 30% co-scaling waste **and** interface stable ≥2 releases | D |
| Blockchain | A threat signatures cannot address | **Never, currently** |

[migration-triggers.md](scaling/migration-triggers.md) also records **reverse triggers** — when to remove a component. Architecture that only accretes becomes unoperable.

## 10. What are the five most dangerous failure modes?

1. **A false event published under official authority.** People act on it. Controls are jurisdiction/category binding, DB-enforced official-evidence requirement, visible provenance, and moderation. Residual risk: a legitimately-authorised node lying — accepted, because the alternative is JanSetu adjudicating between government departments.
2. **An event resolving through silence.** A closure disappears because reports stopped, and someone drives into it. `RESOLVED` requires positive resolution evidence by database constraint; silence decays to `STATUS_UNKNOWN`.
3. **Undetected projection divergence.** The system's central claim — every event is reconstructable — becomes quietly false. Closed by a stored canonical projection hash, a nightly shadow rebuild, and a paging alert. This is the capability Cerevra lacks entirely.
4. **Notification fan-out failure or storm.** Either citizens are not alerted, or alert fatigue disables the channel before the event that matters. Controls: dedup by unique key, caps, freshness budgets, severity prioritisation.
5. **The platform becoming a surveillance asset.** Created by success rather than by failure. Controls: routes in an isolated encrypted schema with no public grant, 30-day impact retention, and an explicit written boundary on what is never collected.

## 11. What are the three biggest security and privacy risks?

1. **Compromised node key** — impersonation of a government authority. Mitigated by JanSetu never holding private keys, immediate stream-pushed revocation, and a deliberately short 5-minute key-policy TTL.
2. **Saved-route exfiltration** — movement patterns for millions of citizens. Mitigated by schema isolation with no public-API grant, encryption, short retention, and alerting on any access by a non-route role.
3. **Coordinated brigading manufacturing false corroboration.** Mitigated by spatial-diversity requirements and by a hard ceiling: community evidence can never reach `OFFICIALLY_CONFIRMED`.

## 12. What should not be built in the first three months?

Kafka, Kubernetes, OpenSearch, ClickHouse, vector search, microservices, H3/S2, read replicas, the cross-node federation protocol, effective-permission caching, SMS notifications, and native mobile apps.

Each has a measurable trigger. None protects an invariant or satisfies a measured requirement at PMVP.

## 13. What are the first 30 implementation issues?

[contribution-architecture.md](contribution-architecture.md) §5, grouped as PMVP foundation (1–7), Cerevra-informed ingestion (8–13), trust and evidence (14–17), geospatial (18–20), frontend (21–25), testing (26–27), observability (28), security (29), documentation (30). Each carries context, scope, non-goals, acceptance criteria, tests, dependencies, labels and difficulty.

Critical path: `1 → 2 → 3 → 4 → 8 → 9 → 10 → 13 → 14 → 15 → 16 → 21`.

Issues 14 and 15 (trust ruleset, projection engine) are labelled `maintainer-only` — a well-meaning change there can make the platform lie.

## 14. Expected monthly infrastructure cost

| Scenario | USD/month | INR/month |
| --- | --- | --- |
| A — PMVP, 1 city | ~$70 | ~₹6 K |
| B — city scale | ~$1,360 | ~₹116 K |
| C — 100 cities | ~$10,900 | ~₹927 K |
| D — national | ~$75,000 | ~₹6.4 M |

Excludes people, LLM inference, and delivery-provider fees. **Notification delivery is the sleeper cost** — 12.5 M pushes per major event is near-free over FCM and catastrophic over SMS. A hard budget cap and severity gating must be settled as product policy before any SMS channel is built.

---

## Which decisions remain unresolved

Stated rather than smoothed over.

1. **Cross-shard event semantics.** A cyclone spanning six states does not respect the jurisdiction shard key. This is the hardest unsolved correctness surface at national scale.
2. **Adaptive coarse-cell sizing.** Named as a requirement; not yet specified. A report from a sparsely-populated cell narrows the reporter set more than intended.
3. **Runbooks.** Replica promotion, projection rebuild, source-adapter failure, key revocation, disaster-mode activation, erasure request. Needed before the first public deploy.
4. **SMS cost policy.** A product decision with an architectural consequence.
5. **Kafka versus Pulsar.** Deliberately deferred to operational preference at the time of the trigger.
6. **LLM inference cost at scale.** Unmodelled.
7. **Managed signing for small civic bodies.** The fallback weakens the "JanSetu never holds private keys" guarantee for exactly the nodes most likely to need it. Such nodes must be labelled; whether to offer it at all is unresolved.
8. **The workload model is unmeasured.** [workload-model.md](scaling/workload-model.md) lists five specific findings that would falsify it. First production data should correct it, and any decision that depended on a wrong number should be revisited.

---

## Recommended immediate next steps

1. Open issues 1–7 and build the scaffold.
2. **File the nine Cerevra defects upstream** with file and line references. They are real, precisely located, and fixing them benefits Cerevra independently of JanSetu.
3. Choose the pilot city and secure one real civic publisher — the PMVP exit criteria require both.
4. Write the six runbooks alongside the code they describe.
5. Instrument from the first commit; the workload model needs measurement more than it needs review.
