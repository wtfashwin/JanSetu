# Cerevra → JanSetu Mapping

Companion to [cerevra-capability-audit.md](cerevra-capability-audit.md). Only `PROVEN` and `PARTIALLY_PROVEN` capabilities are mapped. Reuse modes: `REUSE_DIRECTLY`, `EXTRACT_AS_LIBRARY`, `ADAPT`, `REWRITE`, `DO_NOT_USE`.

**Headline decision: JanSetu does not take a runtime dependency on the `cerevra` package.** Justification is in the audit (§5); the short form is that the published wheel is not importable, the crash-recovery engine is wired to a WAL stack the ingestion path does not use, and the namespace model cannot express a jurisdiction. What transfers is the *engineering discipline*, which is the genuinely valuable part.

---

## 1. Mapping table

| Cerevra capability | JanSetu use | Reuse mode | Required adaptation | Invariant protected | Test needed |
| --- | --- | --- | --- | --- | --- |
| WAL-before-mutation ordering (`ingestion/handler.py:188-208`) | Durable ingestion command written before any fetch, parse, or projection effect | **ADAPT** | Log becomes a Postgres `ingestion_commands` table; the "log" and the "effect" share one transaction, so ordering is enforced by the database rather than by call-site discipline | 11, 13 | Kill worker between claim and effect; assert command re-claimed and effect applied once |
| Deterministic replay as a pure fold (`wal/replay.py:146-306`) | `evidence_items` + `event_transitions` → `public_events` projection | **ADAPT** | Fold ordered by a DB-generated monotonic sequence, not by file order; store a projection hash per rebuild so divergence is *detectable* — the gap Cerevra never closed | 6, 10, 12, 15 | Replay-equivalence property test: same inputs → identical projection hash |
| Refusal gating (`query/cqs.py`) | Civic trust gate: no evidence → do not publish; one anonymous report → `UNVERIFIED`; ambiguous location → moderation; no end time → unknown | **ADAPT** | Replace the weighted composite score with **deterministic, versioned rules**. A civic state must never hinge on a tunable float | 1, 2, 3, 4, 7, 16 | Property test: no rule path reaches `OFFICIALLY_CONFIRMED` without official evidence |
| Runtime invariant assertions (`query/invariants.py:39-77`) | Cross-node mutation and cross-jurisdiction leakage raise, not warn | **REUSE_DIRECTLY** (pattern) | Add Postgres RLS underneath — assertions catch read-path leaks, RLS stops bad writes | 11, 14, 17 | Attempt cross-node write; assert DB rejection *and* raised violation |
| Corruption-as-data (`wal/scanner.py:86-95`) | Malformed source documents recorded as `parser_executions` failures, never as exceptions that halt a source | **ADAPT** | Poison document gets a terminal failure state and a visible metric; the source stays healthy for other documents | 15, 24 | Feed malformed HTML/PDF; assert one failed execution, source continues, failure is visible |
| CRC32 length-prefixed framing (`wal/writer.py:300-318`, `wal/scanner.py:49-108`) | Archive/export format for cold evidence bundles and federation transfer | **EXTRACT_AS_LIBRARY** | Only if a file-based archive log is needed. Replace CRC32 with SHA-256 where tamper-evidence (not just corruption detection) is required | 7 | Round-trip + bit-flip detection test |
| Ingestion backpressure (`handler.py:95,113-118`) | Bounded concurrent source fetches and parser executions | **ADAPT** | Bound becomes a claimed-row count in the database, not a per-process semaphore; stop reading `Semaphore._value` | 24 | Load test: assert bound holds across multiple worker processes |
| Circuit breaker (`plugins/circuit_breaker.py`) | Per-source-adapter breaker | **EXTRACT_AS_LIBRARY** | Pick one of the two implementations; breaker state must drive `source_health` and therefore the public `STALE` presentation | 15 | Chaos: source 503/hang; assert breaker opens and citizens see `STALE` |
| Structured logging `what`/`why`/`outcome`/`safe` (`handler.py:297-310`) | Distinguish "correctly refused to publish" from "failed to publish" | **REUSE_DIRECTLY** (convention) | Same field names; `safe=true` means a deliberate refusal | 16 | Log-assertion test on refusal paths |
| Property-based replay testing (`tests/property/`) | State-machine legality, idempotency, merge/split history preservation | **REUSE_DIRECTLY** (practice) | Hypothesis strategies over civic event transitions | 6, 12, 18, 19 | The tests themselves |
| Fault injection (`utils/failure_simulation.py`) | Chaos suite | **ADAPT** | Must add out-of-process failures: worker kill, disk full, S3 timeout, Redis loss | 10, 11 | See [test-and-chaos-strategy.md](test-and-chaos-strategy.md) |
| Content fingerprinting (`handler.py:148-157`) | Document-bytes identity | **REWRITE** | Full-width SHA-256; **five separate keys**, not one; identity is a DB unique constraint, not a truncated hash used as a primary key | 1, 13 | Collision and race tests; concurrent identical ingest yields one document |
| Namespace isolation (`security/namespace.py:15`) | Jurisdiction + civic-node scoping | **REWRITE** | Relational rows + `ltree` path. Cerevra's flat `^[a-zA-Z0-9_-]+$` ≤64 cannot express `india/maharashtra/pune/traffic-police/advisories` | 11, 12 | Ancestor-scoped authorisation tests |
| Sequence allocation (`handler.py:350-355`, `writer.py:271-280`) | Ordering of commands, evidence, transitions | **REWRITE** | Single source: a Postgres identity column with a uniqueness constraint. Never read-compute-write | 6, 12 | Concurrency test asserting strict monotonicity and no duplicates |
| Crash recovery (`wal/recovery.py`) | Startup recovery | **DO_NOT_USE** | Engine reads the legacy JSON WAL; ingestion writes the binary WAL. It cannot recover a real ingestion crash | 11 | JanSetu's own recovery: unclaimed/expired commands are re-claimed on worker start |
| Snapshot/restore (`wal/snapshot.py`) | Projection checkpoints | **REWRITE** | Checkpoint = `(last_applied_sequence, projection_hash)` row, enabling incremental rebuild | 12, 13 | Rebuild-from-checkpoint equivalence test |
| BM25 store (`utils/bm25_store.py`, pickled) | Search | **DO_NOT_USE** | Postgres FTS. Pickle load is arbitrary code execution and breaks determinism across library versions | 21 | Index rebuild test |
| Hybrid rerank (`indexing/hybrid_rerank.py`) | Moderator evidence recall only | **ADAPT** | Advisory only. **Vector similarity must never determine event truth** | 7 | Assert no code path lets a similarity score change event state |
| Metrics registry (`monitoring/metrics.py`) | Prometheus metrics | **DO_NOT_USE** implementation / **ADAPT** taxonomy | Use `prometheus_client`; keep the *metric naming discipline* | — | Metric-presence test |
| Tracing (`monitoring/tracing.py`) | Distributed tracing | **DO_NOT_USE** | Bespoke, not OpenTelemetry, no context propagation. Adopt OTel | — | Trace-continuity test across worker boundary |
| GDPR deletion/audit/lineage (`compliance/`) | Reporter erasure vs immutable evidence | **ADAPT** | Resolve the tension Cerevra leaves open: *personal data* is erasable, *civic evidence* is immutable. Separation must be structural — see §6 | 7, 12(privacy), 17 | Erasure test: reporter data gone, evidence and event history intact |
| SDKs (`sdk/python`, `sdk/typescript`) | Source-adapter SDK, node SDK | **REWRITE** | Different domain entirely | — | SDK contract tests |
| Distributed consensus, sharding, billing | — | **DO_NOT_USE** | Self-declared work-in-progress (coverage gate lowered to 40%, commit `43ff172`); shards are directories on one host | — | — |

---

## 2. WAL-backed official-source ingestion

Target flow:

```
Source item discovered
→ ingestion command durably appended
→ source fetched
→ raw object stored (immutable)
→ content fingerprinted
→ document version created
→ parser executed
→ evidence candidates created
→ effects committed
→ command acknowledged
```

### Record schema — `ingestion_commands`

| Column | Type | Notes |
| --- | --- | --- |
| `sequence` | `bigint GENERATED ALWAYS AS IDENTITY` | **Sole** ordering authority. Fixes Cerevra §3.4 |
| `command_id` | `uuid` (v7) | Stable identity |
| `idempotency_key` | `text NOT NULL UNIQUE` | `sha256(source_id ‖ discovered_url ‖ discovery_watermark)` |
| `civic_node_id`, `source_id` | `uuid` | Scope; RLS anchor |
| `command_type` | enum | `FETCH_SOURCE`, `REPARSE_DOCUMENT_VERSION`, `REBUILD_PROJECTION` |
| `payload` | `jsonb` | Typed per command |
| `state` | enum | `PENDING`, `CLAIMED`, `SUCCEEDED`, `FAILED`, `DEAD_LETTER` |
| `attempts` | `int` | Bounded |
| `claimed_by`, `claimed_at`, `lease_expires_at` | | Lease-based claim |
| `created_at` | `timestamptz` | |

Attempts are rows in `ingestion_attempts` (`APPEND_ONLY`) — never counters mutated in place. This is the direct lesson of Cerevra §3.5, where acknowledgement state was recorded by rewriting the log file.

### Acknowledgement boundary

A command is acknowledged **only** when the transaction containing (a) all derived rows and (b) `state = SUCCEEDED` commits. Nothing is acknowledged on dispatch. A crash before commit leaves the lease to expire and the command to be re-claimed.

Object storage sits outside that transaction, so ordering is fixed: **write and verify the S3 object first, then commit the row that references it.** Orphaned objects are harmless and swept; dangling references are not tolerable.

### Retry, poison handling, checkpointing

- Retry: exponential backoff with jitter, capped attempts, breaker-aware. A closed-then-open breaker suspends the source without failing individual commands.
- Poison: after N attempts → `DEAD_LETTER`, visible in `moderation`, counted by `parser_failures_total`. Never silently dropped, never infinitely retried.
- Checkpointing: `projection_checkpoints(projection_name, last_applied_sequence, projection_hash, ruleset_version)`. Cerevra had no checkpoint, so `rebuild_manifest` was O(full history) forever.
- Compaction: succeeded commands older than the retention window are archived to object storage and pruned. Evidence and transitions are **never** compacted away.
- Ordering: per `source_id`, by `sequence`. Global ordering is neither required nor claimed.
- Partitioning: `ingestion_commands` and `ingestion_attempts` partitioned by month at city scale.

### Crash scenarios

| Crash point | Outcome |
| --- | --- |
| Before command insert | Nothing happened; rediscovery re-creates it (same idempotency key) |
| After insert, before claim | Command is `PENDING`; picked up |
| After claim, before fetch | Lease expires; re-claimed |
| After S3 write, before DB commit | Orphan object; swept. No dangling reference |
| After DB commit, before ack | Impossible — ack is in the same transaction |
| During projection | Checkpoint unchanged; rebuild resumes from `last_applied_sequence` |

---

## 3. Deterministic replay

Deterministic inputs, all recorded on the row that produced an evidence item:

```
raw document version (content-addressed)
parser version
ruleset version
normalisation version
geocoder version + reference-data version
event-matching version
```

Legitimate nondeterminism, captured rather than pretended away:

- **Wall-clock time.** Rules read `evidence.observed_at` / `known_at`, never `now()`. Any rule needing "elapsed time" takes an explicit evaluation timestamp, stored on the transition.
- **External geocoding.** Results are cached and content-addressed; replay uses the cached result, not a fresh call.
- **LLM extraction.** Inherently nondeterministic. Therefore: the model's *output* is persisted as an evidence candidate with the model id and prompt version, and replay reuses the stored output. **The LLM is never re-invoked during replay.** This is what keeps "LLM extraction is never equivalent to event truth" (invariant 7) compatible with "replay is deterministic" (invariant 12).
- **Ordering ties.** Broken by `(sequence, evidence_id)`, never by hash-map iteration order.

Replay is used for: parser upgrades, projection rebuild, index rebuild, incident recovery, audit verification, parser-version comparison, and reproducing disputed events.

**Divergence detection** — the capability Cerevra lacks entirely. Every rebuild computes a canonical projection hash. Rebuilding at the same `(sequence, ruleset_version, parser_version)` must produce the same hash; a mismatch increments `replay_divergence_total` and pages. A rebuild at a *new* ruleset version is expected to differ and is recorded as a versioned migration, not an alarm.

---

## 4. Namespace isolation

Cerevra's model (`^[a-zA-Z0-9_-]+$`, ≤64 chars, `/` explicitly rejected) supports **none** of what JanSetu needs:

| Requirement | Cerevra | JanSetu |
| --- | --- | --- |
| Data isolation | Partial — directory prefix | RLS on `civic_node_id` |
| Access control | Flat token | Ancestor-scoped permissions over a jurisdiction tree |
| Retention | Not modelled | Retention class per node/category |
| Search filtering | Not modelled | Jurisdiction + category + authority filters |
| Replay scoping | Per-matter only | Per subtree, per source, per time range |
| Encryption boundary | None | Per-class, node keys held separately |
| Deletion | Namespace-wide evict | Scoped erasure of personal data only |
| Federation | Absent | Node identity + signing keys + trust level |

Hierarchy: `country / state / district-or-city / civic node / department / source`, with orthogonal `visibility class` and `retention class`. Stored as first-class rows plus an `ltree` materialised path for subtree queries. **Rewrite, not adapt.**

---

## 5. Content fingerprinting — five keys, not one

Cerevra's single truncated hash conflates classes that JanSetu must distinguish.

| Class | Key | Effect on collision/match |
| --- | --- | --- |
| Identical document | `sha256(raw_bytes)` full width | Same `source_document_version`; no new version |
| New document version | same `source_document_id`, different content digest | New version row, linked to predecessor |
| Same content at a new URL | content digest matches, `source_document_id` differs | Cross-reference recorded; **not** merged automatically — the same notice from two departments is corroboration, not duplication |
| Same event across documents | event-match key: `(category, normalised location, overlapping time window, jurisdiction)` | Evidence links to an existing `public_event` |
| Duplicate evidence | `(document_version_id, parser_version, extraction_ordinal)` | Unique constraint; re-parse is idempotent |
| Duplicate community report | `(reporter_pseudonym, category, coarse location cell, time bucket)` | Rate-limited and deduped; does **not** increase corroboration count |
| Duplicate notification | `(subscription_id, event_id, transition_sequence)` | Unique constraint; retries cannot re-send (invariant 9) |

Full-width SHA-256 everywhere. Identity is enforced by database unique constraints, not by hoping a truncated hash does not collide.

---

## 6. Refusal gating → civic truth controls

| Situation | Cerevra analogue | JanSetu outcome |
| --- | --- | --- |
| No evidence | `REFUSE` | Not published at all |
| Single anonymous report | low composite | `UNVERIFIED`, visibly labelled |
| Multiple independent reports, no official source | mid composite | `COMMUNITY_CORROBORATED` — **never** `OFFICIALLY_CONFIRMED` (invariant 2, enforced by DB constraint) |
| Official document, ambiguous location | — | `DRAFT`, routed to moderation |
| No official end time | — | End time renders "unknown". Never estimated (invariant 1) |
| Source unavailable | — | `STALE` or `STATUS_UNKNOWN`, with source health shown |
| Conflicting evidence | consistency dimension (**stubbed to 1.0 in Cerevra**) | `CONFLICTING` — a real, displayed state, showing both sides |
| Reports simply stopped | — | Event does **not** resolve. Freshness decays toward `STATUS_UNKNOWN` (invariant 3) |

The crucial adaptation: Cerevra decides with a weighted float (`cqs.py:76-84`, and its consistency dimension carries weight 0.0 — contradiction detection does not exist). JanSetu decides with **deterministic, versioned, inspectable rules**, because a citizen must be able to be told *which rule* produced a state, and a rebuild must reproduce it exactly.

---

## 7. Hybrid retrieval — bounded use

Permitted: moderator evidence search, historic event retrieval, notice search, similar-event discovery, source-document lookup, grounded public explanation with citations.

Forbidden: determining whether an event exists, its state, its location, or its resolution.

PMVP uses Postgres FTS with metadata, geospatial, time and authority filters. Semantic retrieval arrives only on the trigger in [migration-triggers.md](scaling/migration-triggers.md), and only as advisory recall.

---

## 8. Net assessment

Five things transfer with real value: **durable-command-before-effect**, **replay as a pure fold into a projection**, **refusal as a first-class outcome**, **invariants that raise rather than log**, and **corruption modelled as data**. Those are good instincts and JanSetu is better for inheriting them.

Everything else is either domain-mismatched (flat namespaces, document retrieval, billing) or below JanSetu's correctness bar (truncated hashes, dual WAL stacks, in-place log mutation, fire-and-forget threads, pickled indexes). The re-implementation is not a rejection of Cerevra — it is a consequence of JanSetu needing stricter guarantees in exactly the places where Cerevra's are currently weakest.
