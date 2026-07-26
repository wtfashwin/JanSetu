# 0006 — Cerevra integration: adopt the discipline, not the dependency

Status: Accepted
Date: 2026-07-27

## Context

JanSetu's README states an intent to integrate Cerevra for WAL-durable, namespace-isolated ingestion. This ADR records what the forensic audit found and what follows from it. Full evidence: [cerevra-capability-audit.md](../architecture/cerevra-capability-audit.md); per-capability mapping: [cerevra-integration.md](../architecture/cerevra-integration.md).

Cerevra is Apache-2.0 and authored by a JanSetu maintainer, so **reuse is legally clear**. The question is purely technical.

### What is genuinely proven and valuable

1. WAL-before-mutation ordering with fsync, enforced at call sites (`cerevra/ingestion/handler.py:188-208`, `cerevra/wal/writer.py:47-60`), backed by real filesystem crash tests.
2. Deterministic replay as a pure fold into a projection (`cerevra/wal/replay.py:146-306`).
3. Refusal as a first-class outcome (`cerevra/query/cqs.py` — `CQSDecision.REFUSE`).
4. Runtime invariants that raise rather than log (`cerevra/query/invariants.py:39-77`).
5. Corruption modelled as data, not exception (`cerevra/wal/scanner.py:86-95`).

### What blocks a runtime dependency

| Finding | Evidence |
| --- | --- |
| Two disjoint WAL stacks; the recovery engine reads the one ingestion never writes to | `cerevra/wal/recovery.py:17` takes the legacy JSON `WriteAheadLog`; ingestion uses binary `WALWriter` |
| Legacy WAL is not append-only | `cerevra/wal/wal.py:216-255` rewrites log files in truncating mode, no fsync, no atomic rename |
| Dedup is a 64-bit truncated hash with a check-then-act race | `handler.py:148-157` |
| Two disagreeing sequence sources; duplicates never raise | `handler.py:350-355` vs `writer.py:271-280`; `replay.py:149` uses `max(...)` |
| Namespaces are flat (`^[a-zA-Z0-9_-]+$`, ≤64, `/` rejected) | `cerevra/security/namespace.py:15` — cannot express `india/maharashtra/pune/traffic-police` |
| Silent permanent loss in the async path | `handler.py:411-449` — daemon thread, 2 in-memory retries, `logger.error` |
| Pickled index state | `namespace.py:68` → `bm25_state.pickle` (RCE on untrusted load; breaks determinism) |
| Published wheel is not importable | `pyproject.toml:13-30` omits `structlog`, `redis`, `aiohttp`, `stripe`, `gunicorn`. Reproduced locally |
| No replay-divergence detection | No projection hash is stored anywhere |

## Decision

**Do not take a runtime dependency on the `cerevra` package. Adopt its engineering discipline; re-implement the mechanisms on Postgres.**

Specifically:

- **ADAPT**: WAL-before-effect → a Postgres `ingestion_commands` table (ADR 0003). Replay-as-pure-fold → the evidence→event projector (ADR 0007). CQS refusal → deterministic versioned trust rules. Backpressure → bounded claim budget. Fault injection → chaos suite.
- **REUSE_DIRECTLY (patterns)**: raising invariant assertions; the `what`/`why`/`outcome`/`safe` log convention; property-based replay testing.
- **EXTRACT_AS_LIBRARY (if needed later)**: length-prefixed CRC32 framing and the corruption-tolerant scanner, for a file-based archive/federation transfer format.
- **DO_NOT_USE**: `CrashRecoveryEngine`, legacy `WriteAheadLog`, `AdaptiveRanker.build_window` (dead — `cerevra/query/ranker.py:187-191` cannot execute), pickled BM25 store, `distributed/`, `sharding/`, `billing/`, bespoke metrics and tracing.

## Alternatives considered

| Alternative | Rejected because |
| --- | --- |
| Depend on `cerevra` as published | Wheel is not importable; recovery engine does not cover the ingestion path; namespaces cannot express a jurisdiction |
| Vendor the WAL modules | Inherits the dual-stack confusion and the flat namespace model; forfeits transactional acknowledgement |
| Fork and fix Cerevra | Two products, one team. JanSetu's needs are narrower and stricter |
| Fix Cerevra upstream, then depend | Worth doing on its own merits (see below), but JanSetu should not block on another project's release cycle |

## Consequences

JanSetu gets transactional acknowledgement, a single sequence authority, full-width hashes, hierarchical jurisdictions, and replay-divergence detection — none of which the existing implementation provides. Cerevra's genuine contribution is preserved: it shaped the design, and several JanSetu invariants exist because its audit revealed the failure mode.

Cost: writing the ingestion log rather than importing it. Given the defect list, this is a smaller cost than the alternative.

## Risks

- **Perception of rejecting a maintainer's work.** Addressed by recording the five genuinely strong mechanisms explicitly, and by the fact that JanSetu's design is visibly shaped by them.
- **Re-introducing the same defects.** Mitigated: each is encoded as a named invariant with a test in [reliability-invariants.md](../architecture/reliability-invariants.md) and [test-and-chaos-strategy.md](../architecture/test-and-chaos-strategy.md) — including a specific test that every command type written by ingestion is recoverable by the recovery path, which is the test whose absence let the dual-stack defect survive.

## Measurement

- `replay_divergence_total` = 0 (capability Cerevra lacks).
- Concurrent identical ingest → exactly one document version (the race Cerevra has).
- Every ingestion command type covered by a recovery test (the coherence Cerevra lacks).

## Introduction trigger

Reconsider a direct dependency if Cerevra: unifies its WAL stacks, fixes packaging, adopts full-width content hashes, adds hierarchical namespaces, and adds projection-hash divergence detection. At that point extraction of a shared library becomes attractive for both projects.

## Rollback strategy

n/a. **Recommended follow-up regardless:** file the nine specific defects upstream with file and line references. They are real, precisely located, and fixing them benefits Cerevra independently of JanSetu.
