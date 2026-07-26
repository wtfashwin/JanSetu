# Test and Chaos Strategy

Six tiers, separated by cost and dependency. Each is independently runnable and independently gated.

**Contributor constraint, treated as a hard requirement:** tiers 1 and 2 must run with **no Docker, no network, no external service**. An open-source contributor fixing a parser fixture will not stand up Postgres, MinIO and Redis to do it. Any change that breaks this property is a regression.

The Cerevra audit is a source of specific test requirements here: several tests below exist because their absence in Cerevra allowed a real defect to survive (a WAL stack disconnected from its recovery engine; dead code with 33 passing test invocations; a truncated dedup hash with no collision test).

---

## Tier 1 — Unit (no dependencies)

| Area | Tests |
| --- | --- |
| Fingerprinting | Full-width SHA-256; stability across processes; **explicit test that identity is not truncated** |
| Idempotency keys | Pure, stable, collision-resistant; identical inputs → identical key |
| State machine | Every legal transition; every illegal transition rejected; **exhaustive proof that no input lacking official evidence reaches `OFFICIALLY_CONFIRMED`** (I3) |
| Trust rules | Full decision table per ruleset version; no branch reaches `RESOLVED` without resolution evidence (I4) |
| Parser normalisation | Date/time formats in Indian civic notices; place-name transliteration variants; unit normalisation |
| Namespace / jurisdiction | `ltree` ancestor queries; scope containment; no cross-jurisdiction match |
| Impact classification | Route × event geometry → `blocked` / `delayed` / `nearby` / `clear` boundaries |
| Serialisation | Freshness fields always present (I5); unknown renders as `null` with `known: false`, never a substituted value (I16) |
| Projector purity | **Lint + runtime stub asserting no `now()`, no network, no map-order dependence** (I12) |

Runtime target: under 30 seconds.

---

## Tier 2 — Property-based (Hypothesis, no dependencies)

Adopted directly from Cerevra's practice (`cerevra/tests/property/test_wal_replay_invariants.py`), which is one of its genuinely strong points.

| Property | Statement |
| --- | --- |
| **Replay equivalence** | For any valid evidence sequence, two independent folds produce identical projection hashes (I12) |
| **Idempotency** | Applying any command N times produces the same state as applying it once (I2) |
| **Transition legality** | No sequence of evidence produces an illegal transition |
| **Official-evidence gate** | No generated input reaches `OFFICIALLY_CONFIRMED` without official evidence (I3) |
| **Merge conservation** | Evidence links before a merge == evidence links after; none lost (I18) |
| **Split partition** | Child evidence sets partition the parent's exactly once — no drop, no duplicate (I19) |
| **Node isolation** | No generated request sequence yields a cross-node write (I14) |
| **Ordering** | Sequences are strictly monotonic; no duplicates under concurrent generation |
| **Geometry** | Invalid geometry is rejected, never silently repaired |
| **Hash canonicality** | Projection hash is independent of row order, insertion order, and JSON key order |

---

## Tier 3 — Integration (Docker Compose: Postgres+PostGIS, MinIO, Redis)

| Area | Tests |
| --- | --- |
| Transactions | Command state and derived rows commit atomically; rollback leaves no partial effect |
| Constraints | Every `UNIQUE` and `CHECK` in [data-model.md](data-model.md) is exercised and proven to fire |
| Append-only | `UPDATE`/`DELETE` on append-only tables raise, as the application role (I6) |
| RLS | Node A cannot read or write node B's rows — **verified at the database, not only at the application** (I14) |
| Route isolation | Public-API role cannot reach route tables even with crafted parameters (I17) |
| PostGIS | Route × event intersection correctness; jurisdiction containment; SRID handling; buffer semantics |
| Object storage | Content-addressed keys; overwrite denied; digest verified on read (I7) |
| Ordering | Object written and verified **before** the referencing row commits; no dangling reference |
| Source adapters | Fixture-driven, against recorded real notice formats |
| Projection | Evidence → transitions → projection, end to end |
| Search | Index rebuild produces identical results to pre-drop (I21) |
| Notifications | Unique delivery key prevents duplicates across retries (I9) |
| Erasure | User deletion removes routes, subscriptions and identity mapping; reports survive pseudonymously; event history intact |

---

## Tier 4 — Replay and recovery

The tier that most directly encodes the audit findings.

| Test | Assertion |
| --- | --- |
| **Worker crash (5 points)** | `SIGKILL` before claim / after claim / after fetch / after object write / mid-transaction. Every case: exactly one observable effect, nothing lost (I2, I11) |
| **Partial object upload** | Interrupted PUT → no version row → command retried → clean result |
| **Duplicate delivery** | Same payload 1,000× → one document version, one evidence set, one event (I1) |
| **Parser upgrade** | New version produces new evidence; historic evidence rows byte-identical (I8) |
| **Projection rebuild** | Full rebuild from evidence reproduces the identical projection hash (I12) |
| **Checkpoint recovery** | Rebuild from checkpoint == full rebuild |
| **Replay divergence detection** | Deliberately perturb a rule → divergence detected and attributed to the rule (I13) |
| **Lease expiry** | Worker hangs → lease expires → re-claim → one effect |
| **WAL-to-recovery coherence** | **Every command type written by ingestion is recoverable by the recovery path.** This test exists specifically because its absence in Cerevra let a recovery engine wired to the wrong WAL stack survive (`cerevra/wal/recovery.py:17`) |

---

## Tier 5 — Chaos

Cerevra's fault injection is in-process only (`cerevra/utils/failure_simulation.py`). These are **out-of-process** failures, which is where the real ones live.

| Scenario | Expected behaviour |
| --- | --- |
| Worker `SIGKILL` under load | Leases expire; work re-claimed; no loss, no duplication |
| Postgres restart | Connections re-established; in-flight transactions rolled back; commands retried |
| Postgres failover | RTO within budget; no acknowledged command lost |
| Object storage timeout / 503 | Command retried; no dangling reference; breaker opens on sustained failure |
| **Redis `FLUSHALL` mid-suite** | **Zero test failures.** Standing test, not occasional (I10) |
| Redis unavailable entirely | Rate limiting falls back to in-process; caching disabled; system correct but slower |
| Notification provider outage | Deliveries retried against the same row; no duplicates; failures visible |
| Network partition (worker ↔ DB) | Worker stops claiming; lease expiry lets another worker proceed |
| Slow source (30 s responses) | Timeout; breaker opens; events show `STALE`; **no event silently resolves** (I4, I15) |
| Malformed source (bad HTML/PDF/encoding) | Recorded parser failure; source stays healthy for other documents; failure visible (I15) |
| Poison document (repeatedly fails) | `DEAD_LETTER` after N attempts; surfaced in moderation; never infinite retry |
| Federation outage | Cached policy honoured; **revocations still applied**; new registrations blocked |
| Disaster traffic spike (×50 reads) | CDN absorbs; static fallback serves highest-severity events with the DB down |
| Duplicate report flood (10 K in 60 s) | Dedup and rate limits hold; corroboration requirements unchanged; **trust rules unrelaxed** (I24) |
| Clock skew (±10 min across workers) | Nonce window tolerates; ordering unaffected (sequences are DB-generated, not clock-derived) |
| Disk full on the database | Writes fail loudly; no partial commit; alerts fire |

---

## Tier 6 — Load

| Scenario | Profile | Success criteria |
| --- | --- | --- |
| Normal day (B) | 26 writes/s, 69 reads/s | All SLOs met |
| City incident | ×50 write, ×20 read for 2 h | SLOs met; projection lag < 60 s |
| Multi-city incident (C) | 264 writes/s, 13.9 K reads/s | Regional SLOs met |
| National emergency (D) | 2 K writes/s, 100 K reads/s | Read path holds via CDN; write path degrades gracefully |
| Read-heavy single event | 50 K reads/s on one event | Cache and CDN absorb; origin load bounded |
| Notification fan-out | 200 K (B) / 12.5 M (C) deliveries | Drain time measured and **published**; ingestion unaffected |
| Full projection rebuild | Scenario C volume | Partitioned rebuild < 15 min |
| Concurrent ingest of identical documents | 100 workers, same document | Exactly one version, one evidence set (I1) |

Load tests validate the [workload-model.md](scaling/workload-model.md) assumptions. Where measurement contradicts an assumption, **the model is corrected and any decision that depended on it is revisited** — that is what the model is for.

---

## CI gating

| Tier | Trigger | Blocks merge |
| --- | --- | --- |
| 1 Unit | Every push | Yes |
| 2 Property | Every push | Yes |
| 3 Integration | Every PR | Yes |
| 4 Replay/recovery | Every PR | Yes |
| 5 Chaos | Nightly + pre-release | Release only |
| 6 Load | Weekly + pre-release | Release only |

Also gating every PR: `ruff`, `mypy --strict` on domain modules, `import-linter` (module boundaries, forbidden edges, no analytics client in operational code — I22), dependency vulnerability scan, and a migration-safety check.

### Coverage policy

**Global line coverage is not a gate.** The audit found `AdaptiveRanker.build_window` — code that cannot execute (`cerevra/query/ranker.py:187-191`) — carrying 33 test invocations under a `fail_under = 76` global gate. A number that can be satisfied by exercising unreachable code measures nothing.

Instead:

- **Branch coverage ≥ 95% on invariant-bearing modules** (`event_state_machine`, `evidence`, `ingestion`, `public_events`).
- **Mutation testing on the trust ruleset and the projector** — a surviving mutant in the state machine fails the build.
- **Dead-code detection** (`vulture` or equivalent) as a gate, so unreachable code is caught rather than tested.

---

## Fixtures

Public, versioned, contributor-usable — the primary good-first-issue surface.

```
fixtures/
  sources/          real notice formats (municipal PDF, traffic HTML, utility notice, GTFS alert)
  documents/        raw bytes + expected extraction, per parser version
  geometry/         road segments, jurisdiction boundaries, sample routes
  events/           evidence sequences → expected projection + expected hash
  rulesets/         versioned trust rules with decision tables
```

Every fixture carries its expected output **and its expected projection hash**, so a contributor can verify determinism locally without any infrastructure.

Real notices are anonymised of any incidental personal data before being committed.

---

## What is deliberately not tested

| Not tested | Why |
| --- | --- |
| LLM output quality | Nondeterministic and not correctness-critical — LLMs produce candidates, never truth (invariant 7). What *is* tested: that no LLM output path can advance an event state |
| Third-party provider internals | Contract-tested at the boundary only |
| Cloud provider durability | Shared-responsibility boundary |
| UI pixel rendering | Snapshot-tested for state presentation (`STALE`, `UNVERIFIED`, unknown end time), not for appearance |
