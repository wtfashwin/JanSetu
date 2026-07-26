# Cerevra Capability Audit

Subject: `https://github.com/wtfashwin/cerevra` @ `14ca9dc` (public `main`, depth-50 clone)
Method: source reading, symbol tracing, test inspection, local suite execution.
Rule applied: a README claim is never evidence. A capability is `PROVEN` only when implementation *and* a test that exercises the real mechanism both exist.

Licence: Apache-2.0. Author: Ashwin Upadhyay. No third-party copyleft in `[project].dependencies`. **Reuse is legally clear** for an open-source JanSetu.

---

## 1. Classification table

| Capability | Classification | Implementation evidence | Test evidence | Limitation | JanSetu decision |
| --- | --- | --- | --- | --- | --- |
| WAL-before-mutation ordering | **PROVEN** | `ingestion/handler.py::_do_ingest` L188–208 appends+fsyncs WAL record, *then* writes chunk files. `wal/writer.py::append_record` L47–60 | `tests/wal/test_recovery_real.py` (real FS crash sim, commit `36f6bfe`); `tests/ingestion/test_handler.py` | Effect and log are on the same filesystem; no cross-store transaction | **ADAPT** — port the discipline, not the code |
| Binary WAL framing + CRC32 | **PROVEN** | `writer.py::_encode_record` L300–318: `struct.pack("<II", len, crc)` + msgpack payload; `scanner.py::_scan_file` L49–108 verifies CRC and yields `CorruptedRecord` | `tests/wal/`, `tests/property/test_wal_replay_invariants.py` | CRC32 detects corruption, not tampering (not a MAC) | **EXTRACT_AS_LIBRARY** |
| Corruption-tolerant scan (skip-and-continue) | **PROVEN** | `scanner.py` L86–95: CRC mismatch yields `CorruptedRecord`, advances offset, continues | `tests/wal/` | Length-field corruption ends the file scan (L67–74), silently dropping the tail | **ADAPT** — must also alert |
| Deterministic replay → manifest projection | **PROVEN** | `wal/replay.py::ReplayEngine.rebuild_manifest` L79–131; `_apply_*` L146–306 are pure fold operations over ordered records | `tests/property/test_wal_replay_invariants.py` (Hypothesis) | Fold is over *file order*, not sequence order — see §3.4 | **ADAPT** — the strongest idea in the repo |
| Refuse-to-answer gating (CQS) | **PROVEN** | `query/cqs.py`: `CQSDecision{CONFIDENT,WARNING,RAW_ONLY,REFUSE}`, `ContextQualityScore.composite` L76–84, `decide()` L86–96; ordered-threshold invariant enforced in `CQSThresholds.__post_init__` L46–54 | `tests/query/test_cqs.py`, `tests/synthesis/test_cqs_synthesis_gating.py`, `tests/e2e/test_cqs_e2e.py` | `consistency` dimension is **stubbed to 1.0 with weight 0.0** (L82–84) — contradiction detection does not exist | **ADAPT** — concept is exactly right for civic trust; scoring must be replaced by deterministic rules |
| Runtime namespace-isolation assertions | **PROVEN** | `query/invariants.py::assert_namespace_isolation` L39–50, `assert_cache_namespace_match` L53–63, `assert_all` L66–77 | `tests/security/`, `tests/invariants/` | Assertions are on the read path only; no DB-level enforcement | **REUSE_DIRECTLY** (pattern) — pair with Postgres RLS |
| Path-traversal defence | **PROVEN** | `security/namespace.py::ensure_namespace_isolation` L70–97 (abspath prefix check), `_is_safe_path` L139–162 | `tests/security/` | Correct for a filesystem store; irrelevant once storage is Postgres+S3 | **DO_NOT_USE** (JanSetu has no per-tenant filesystem) |
| Ingestion backpressure | **PROVEN** | `handler.py` L95, L113–118: non-blocking `threading.Semaphore`, raises `BackpressureError`, emits `ingestion.backpressure_rejected` | `tests/ingestion/` | Per-process only; reads `Semaphore._value` (private API) for the gauge | **ADAPT** |
| Snapshot / restore | **PARTIALLY_PROVEN** | `wal/snapshot.py::SnapshotManager.create_snapshot` L22, `restore_snapshot` L147, `cleanup_old_snapshots` L218 | `tests/wal/` | Built against the **legacy** WAL stack; not wired to `ReplayEngine` checkpointing | **REWRITE** |
| Crash recovery engine | **PARTIALLY_PROVEN / UNSAFE** | `wal/recovery.py::CrashRecoveryEngine.recover` L21 — but `__init__` L17 takes `WriteAheadLog` (legacy JSON stack) | `tests/wal/test_recovery_real.py` | **Cannot see records written by `WALWriter`**, which is what ingestion uses. See §3.1 | **DO_NOT_USE** |
| Content fingerprinting / dedup | **PARTIALLY_PROVEN / UNSAFE** | `handler.py` L148–157: `doc_id = sha256(content).hexdigest()[:16]`; membership test against loaded manifest | `tests/ingestion/test_handler.py` | 64-bit truncation; check-then-act race; no revision concept. See §3.2 | **REWRITE** |
| Idempotent re-ingestion | **PARTIALLY_PROVEN** | Same code path; returns `IngestionResult(already_exists=True)` | `tests/ingestion/test_handler.py`, `tests/e2e/test_upload_to_query.py` | Single-threaded only; no idempotency key, no request-scoped dedup | **REWRITE** |
| WAL rotation | **PARTIALLY_PROVEN** | `writer.py::_should_rotate` L282–290, `_rotate` L292–298 | `tests/wal/` | Rotation decided *after* the write; a single oversized record still lands | **ADAPT** |
| BM25 lexical retrieval | **PROVEN** | `utils/bm25_store.py`, `indexing/bm25_windowing.py`, `indexing/bm25_wrapper.py`, dep `rank-bm25` | `tests/bm25/`, `tests/query/` | Index state persisted as **pickle** (`namespace.py::resolve_index_path` L68 → `bm25_state.pickle`) — arbitrary-code-execution on load of an untrusted file | **DO_NOT_USE** — use Postgres FTS |
| Hybrid BM25 + vector rerank | **PARTIALLY_PROVEN** | `indexing/hybrid_rerank.py::HybridReRanker`, `query/service.py` L261, alpha-weighted | `tests/query/`, `tests/retrieval_pipeline_blackbox.py` | Embeddings are optional and generated in a **daemon thread with 2 retries and no durable queue** (`handler.py` L411–449) — silent permanent loss on failure | **ADAPT**, and only for moderator search |
| Namespace/tenant model | **PARTIALLY_PROVEN** | `security/namespace.py` L15 `^[a-zA-Z0-9_-]+$`, ≤64 chars | `tests/security/` | **Flat single token.** Cannot express `india/maharashtra/pune/...`. See §3.3 | **REWRITE** |
| Metrics + Prometheus text export | **PARTIALLY_PROVEN** | `monitoring/metrics.py::export_prometheus_format` L63 | `tests/monitoring/`, `tests/observability/` | Hand-rolled registry; no `prometheus_client`; no histogram bucket semantics | **DO_NOT_USE** (use `prometheus_client`); keep the *metric taxonomy* |
| Tracing | **CLAIMED_ONLY** | `monitoring/tracing.py`: bespoke `TraceContext`/`Span`/`Tracer` L12–98 | `tests/observability/` | **Not OpenTelemetry.** No W3C context propagation, no exporter, no cross-process correlation | **DO_NOT_USE** — adopt OTel |
| Structured logging | **PROVEN** | `structlog` via `monitoring/logging.py`; call sites carry `what`/`why`/`outcome`/`safe` keys (e.g. `handler.py` L297–310) | Throughout | `monitoring/unified_logger.py` and `monitoring/error_envelope.py` are **0 bytes** — declared, unimplemented | **ADAPT** — the `what/why/outcome` log convention is genuinely good |
| Circuit breaker | **PROVEN** | `plugins/circuit_breaker.py`, `common/circuit_breaker.py` (two implementations) | `tests/plugins/` | Duplicated; in-process only | **EXTRACT_AS_LIBRARY** (one of them) |
| Chaos / fault injection | **PARTIALLY_PROVEN** | `utils/failure_simulation.py`; `tests/chaos/` | Directory exists | Injection is in-process; no process kill, no disk-full, no fsync-lie simulation | **ADAPT** |
| Property-based testing | **PROVEN** | `tests/property/test_wal_replay_invariants.py`, `hypothesis` in `[dev]` | Yes | Only 4 property test files for a system this size | **REUSE_DIRECTLY** (practice) |
| Distributed consensus / quorum | **INFERRED** | `distributed/consensus_guard.py`, `coordinator.py`, `replica.py`, `wal_writer.py`; `WALOpType.COMMIT`, `LEADER_EPOCH`; `replay.py::_apply_commit` L249, `_apply_leader_epoch` L274 | `tests/distributed/`; coverage gate lowered to 40% for this area (commit `43ff172 "M3 work-in-progress"`) | Self-declared work-in-progress. No Jepsen-class verification. Leader election correctness unestablished | **DO_NOT_USE** |
| Sharding | **INFERRED** | `sharding/shard_map.py`, `shard_router.py`, `shard_index_manager.py` | `tests/sharding/` (3 files) | Shards are directories on one host — not distribution | **DO_NOT_USE** |
| Adaptive ranking window builder | **UNSAFE / dead code** | `query/ranker.py::build_window` L132–196 | 33 invocations in `tests/query/test_ranker.py` | L187 `EvidenceSet()` misses 3 required args; L191 `add_block` does not exist; L190 mutates a `frozen=True` dataclass. **Cannot execute.** Production uses only `.min_k`/`.max_k` (`query/service.py` L156, L261) | **DO_NOT_USE** — delete upstream |
| Packaging / distribution | **UNSAFE** | `pyproject.toml` L13–30 omits `structlog`, `redis`, `aiohttp`, `stripe`, `gunicorn`, all present in `requirements.txt` and imported at import time | Reproduced locally: clean `pip install -e ".[dev]"` → `ModuleNotFoundError: No module named 'structlog'` at conftest import | Published wheel is not importable | **REWRITE** before any dependency |
| SDKs (Python + TypeScript) | **PARTIALLY_PROVEN** | `sdk/python/` (18 files), `sdk/typescript/` (20 files), `sdk-ci.yml`, `sdk-publish.yml` | `sdk/python/tests/test_client.py` | Tied to Cerevra's document/query API, not to a civic-event protocol | **REWRITE** for JanSetu |
| GDPR deletion / audit / lineage | **PARTIALLY_PROVEN** | `compliance/deletion.py`, `audit.py`, `lineage.py`; `WALOpType.DELETE` with `user_principal` (`wal/models.py` L31–37) | `tests/compliance/` (5 files) | Deletion is manifest-tombstone + cascade; raw chunk files removed by cascade, so "immutable raw" and "erasure" are in unresolved tension | **ADAPT** — JanSetu needs this tension resolved explicitly |
| Billing / metering | PROVEN but out of scope | `billing/` (7 files), Stripe | `tests/billing/` (10 files) | — | **DO_NOT_USE** |

---

## 2. Five strongest proven mechanisms

Ranked by value to JanSetu.

1. **Durable-command-before-effect, enforced at the call site.** Not a framework, a *habit*: `log_delete`, `log_vector_index_build`, `log_embedding_generated` in `writer.py` all document "must be called **before**" the mutation. JanSetu's entire trust story depends on this ordering, and Cerevra demonstrates it working under real crash tests.

2. **Deterministic replay as a pure fold into a projection.** `ReplayEngine._apply_*` are side-effect-free functions of `(manifest, record)`. This is precisely the shape JanSetu needs for `evidence → public_event` projection, and it is the single most portable idea in the repository.

3. **Refusal as a first-class outcome.** `CQSDecision.REFUSE` and the ordered-threshold invariant encode "a system that says *I don't have enough context* is more trustworthy than one that always answers" (`cqs.py` docstring). JanSetu's `STATUS_UNKNOWN` / `UNVERIFIED` states are the same idea applied to civic truth.

4. **Runtime invariant assertions on the hot path.** `query/invariants.py` raises `InvariantViolation` rather than logging a warning. Cross-namespace leakage becomes a crash, not a silent breach. JanSetu should do the same for cross-node mutation.

5. **Corruption is data, not an exception.** `CorruptedRecord` is yielded into the stream and counted on the manifest (`total_corrupted`). Damage becomes observable instead of fatal. JanSetu needs exactly this for poison source documents.

---

## 3. Defects, hazards, and things that must be rewritten

### 3.1 Two disjoint WAL stacks — the recovery engine is wired to the wrong one

| | Modern stack | Legacy stack |
| --- | --- | --- |
| Writer | `wal/writer.py::WALWriter` | `wal/wal.py::WriteAheadLog` |
| Format | msgpack, len+CRC32 framed, binary | JSON lines |
| Layout | `{root}/wal/{matter_id}/{seq:010d}.wal` | `{root}/wal/{seq:06d}.log` (global) |
| Reader | `wal/scanner.py::WALScanner` | `WriteAheadLog.scan_logs` |
| Applier | `wal/replay.py::ReplayEngine` | `wal/recovery.py::CrashRecoveryEngine` |
| Used by ingestion? | **Yes** (`handler.py` L66, L198) | **No** |
| Used by recovery? | **No** | **Yes** (`recovery.py` L17) |

`CrashRecoveryEngine.recover()` scans legacy JSON logs. Ingestion writes binary per-matter WAL files. **The recovery engine will find nothing to recover from a real ingestion crash.** Crash safety in the ingestion path comes solely from `ReplayEngine.rebuild_manifest`, which is invoked inline on the *success* path (`handler.py` L233) — not from a startup recovery pass.

*Consequence for JanSetu:* the "crash recovery" capability cannot be adopted as-is. JanSetu must implement its own startup recovery over its own log. This is the finding that most changes the integration plan.

### 3.2 Fingerprinting collapses five distinct duplicate classes into one 64-bit hash

`doc_id = sha256(content).hexdigest()[:16]` is used simultaneously as identity, dedup key, and primary key.

- **Truncation.** 64 bits. Birthday collision probability reaches 50% at ~5.1×10⁹ items and is already ~10⁻³ at 10⁸. JanSetu Scenario C alone posits 10⁸ document versions. Two unrelated civic notices colliding would cause one to be **silently discarded** as a duplicate.
- **Race.** L151–157 load-manifest → check → (L198) append. No lock spans it. Concurrent identical uploads both miss and both append.
- **No revision model.** An amended official notice at the same URL produces a different hash and therefore an unrelated document with no link to its predecessor. JanSetu must distinguish *document duplicate* / *document revision* / *evidence duplicate* / *event duplicate* / *notification duplicate*, and one hash cannot do that.

### 3.3 Namespace model cannot represent a jurisdiction

`^[a-zA-Z0-9_-]+$`, ≤64 chars, and `_is_safe_path` rejects `/` outright. Cerevra namespaces are flat opaque tenant tokens.

JanSetu requires `country / state / district / node / department / source / visibility class / retention class`, with:

- ancestor-scoped access control ("all of Pune Municipal");
- retention policy inherited down the tree;
- replay scoped to a subtree;
- search filtered by jurisdiction;
- federation trust attached at node level.

None of that is expressible. **Rewrite, do not adapt.** JanSetu should model jurisdiction as first-class relational rows plus an `ltree` (or closure-table) path — not as a string token.

### 3.4 Sequence numbers: two sources, no uniqueness, silent collapse

`handler.py::_get_next_sequence` L350–355 returns `manifest.last_sequence + 1`.
`writer.py::_get_next_sequence` L271–280 returns `int(highest .wal filename) + 1`.

These are different quantities used in adjacent code. The manifest-derived sequence is computed outside any lock, so N concurrent ingests into one matter can all read the same `last_sequence`. `ReplayEngine._apply_record` L149 does `max(...)` rather than asserting monotonicity, so duplicates never raise — ordering information is lost quietly.

*Consequence for JanSetu:* sequence allocation must be a database-generated monotonic value (Postgres sequence / identity column) with a uniqueness constraint, never a read-compute-write.

### 3.5 The legacy WAL is not append-only

`wal/wal.py::mark_applied` L216–255 reads a whole log file, mutates `status` in memory, and rewrites it with `open(filepath, "w")` — truncating mode, no temp file, no atomic rename, no fsync. A crash mid-write destroys durable records. This directly contradicts the class docstring ("Append-only, immutable entries", L48–49).

*Consequence for JanSetu:* acknowledgement state must never be stored by mutating the log. Use a separate checkpoint/offset row, or an append-only `ACK` record.

### 3.6 Silent permanent loss in the embedding path

`handler.py` L411–449: embeddings are generated on a **daemon thread**, with 2 in-memory retries; on final failure the outcome is `logger.error` and nothing else. There is no durable queue, no dead letter, no repair job. Process exit during retry loses the work with no record. `EMBEDDING_GENERATED` WAL records are written by `writer.py::log_embedding_generated`, but nothing schedules a reconciliation from them.

*Consequence for JanSetu:* every asynchronous effect must be driven from a durable row with a visible failure state, never from a fire-and-forget thread. This is a design rule extracted directly from a real defect.

### 3.7 Pickle as index persistence

`bm25_state.pickle`. `pickle.load` on a file an attacker can influence is arbitrary code execution. Even without an attacker, pickles break across library versions, defeating "deterministic rebuild".

### 3.8 Dead code inflating coverage

`AdaptiveRanker.build_window` cannot execute (§1). Yet `tests/query/test_ranker.py` instantiates `AdaptiveRanker` 33 times. Coverage gate is `fail_under = 76`; a portion of that number is unreachable code. JanSetu should gate on *mutation* or branch coverage of invariant-bearing modules, not global line coverage.

### 3.9 Other observations

- **Accidental complexity.** `distributed/`, `sharding/`, `billing/`, `agentic/`, `plugins/` are all present in a system whose storage layer is a local directory tree. `query/` contains four overlapping service objects (`service.py`, `unified_service.py`, `m1_service.py`, `pipeline.py`) totalling ~80 KB. Two circuit-breaker implementations. Two WAL stacks.
- **Zero-byte declared modules.** `monitoring/unified_logger.py`, `monitoring/error_envelope.py`, `utils/mock_data.py`, `utils/rate_limiter.py`.
- **Missing invariants.** No assertion of sequence monotonicity; no assertion that a chunk file referenced by a manifest exists; no assertion that replay of the same WAL twice yields an identical manifest (the property test checks invariants, not equality of two full replays).
- **Replay divergence is not detectable.** There is no stored projection hash to compare a rebuild against. A parser or code change silently produces a different manifest with no signal. This is the gap JanSetu must close on day one, since its whole audit story is "the event can be rebuilt".
- **Data-loss window.** WAL append (fsync'd) → chunk write (fsync'd) → manifest save. A crash between WAL and manifest is recoverable by replay. A crash *after* `mark_applied`-style mutation in the legacy path is not. Cross-store atomicity (filesystem + future object store) is unaddressed.
- **Concurrency hazards.** Per-matter `threading.Lock` only (`writer.py` L251–256) — no file locking, so **two processes** writing one matter corrupt each other's rotation state. `MAX_CONCURRENT_INGESTIONS` is per-process. `ingestion/lock.py` exists but is not used by `handler.py`.
- **Scale bottlenecks.** `scan_logs()` loads all entries into a list. `mark_applied` is O(all files) per call. `rebuild_manifest` is O(full WAL history) with no checkpoint — replay time grows without bound. Manifest is a single JSON file rewritten per ingest.
- **Security.** Pickle load; CRC32 is not a MAC so WAL records are undetectably forgeable by anyone with disk access; API keys in a local store; `.env.example` is 8.4 KB, implying a large untracked secret surface.
- **Missing tests.** No two-process concurrency test; no test that a truncated/partial WAL *tail* replays to a sane state; no replay-equivalence test; no test of the WAL↔recovery-engine mismatch (which is why §3.1 survived).

---

## 4. Reuse decisions, consolidated

**REUSE_DIRECTLY (as patterns, near-verbatim):**
- runtime invariant assertions that raise (`query/invariants.py`)
- `what` / `why` / `outcome` / `safe` structured-log convention
- property-based testing of replay invariants

**EXTRACT_AS_LIBRARY:**
- length-prefixed + CRC32 record framing and the corruption-tolerant streaming scanner
- one circuit-breaker implementation

**ADAPT:**
- WAL-before-effect ordering discipline → Postgres transactional outbox
- replay-as-pure-fold → evidence→event projection
- CQS refusal gating → deterministic civic trust rules
- ingestion backpressure
- fault-injection harness

**REWRITE:**
- fingerprinting and deduplication (five separate keys, full-width hashes)
- namespace / jurisdiction model
- sequence allocation
- snapshots and checkpointing
- packaging
- SDKs

**DO_NOT_USE:**
- `CrashRecoveryEngine` (wired to the wrong WAL)
- `wal/wal.py::WriteAheadLog` (not append-only)
- `AdaptiveRanker.build_window` (dead, non-functional)
- pickle-backed BM25 store
- `distributed/`, `sharding/`, `billing/`
- bespoke tracing and metrics registries

---

## 5. Recommended integration posture

**Do not take a runtime dependency on the `cerevra` PyPI package.** Reasons, in order of severity: the published wheel is not importable (§1, packaging); the crash-recovery engine does not cover the ingestion path (§3.1); the namespace model cannot express JanSetu's domain (§3.3); and pulling in `stripe`, `redis`, and a pickle-backed index to obtain a WAL is a poor trade.

**Take the ideas, and vendor the two small pieces that are genuinely good.** Concretely: JanSetu's ingestion log should be a Postgres table (durable, ordered, transactional with its effects, and free of every hazard in §3.4–3.6), with Cerevra's framing/scanner extracted only if a file-based archive log is later needed.

This is not a negative verdict on Cerevra. It is a young, ambitious system with several genuinely strong ideas and real crash tests, and its author's instincts about durability and refusal are correct. The specific reason to re-implement rather than depend is that JanSetu's correctness requirements are *stricter* than Cerevra's current guarantees, and the gap is in exactly the areas JanSetu cannot compromise on.

See [cerevra-integration.md](cerevra-integration.md) for the per-capability mapping and required adaptations.
