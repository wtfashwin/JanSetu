# 0002 — Append-only evidence and transitions

Status: Accepted
Date: 2026-07-27

## Context

JanSetu's product invariants require that every public claim has inspectable evidence (4), every state transition is reconstructable (10), and a public event can be rebuilt from append-only evidence plus a versioned ruleset (15). A citizen must be able to ask "why is this road marked officially closed?" and receive a reproducible answer.

A mutable status column cannot answer that. It also makes a legitimate correction indistinguishable from tampering, which is fatal for infrastructure whose value is its trustworthiness.

The Cerevra audit found the failure mode concretely: `cerevra/wal/wal.py:216-255` records acknowledgement by rewriting whole log files in truncating mode — no temp file, no atomic rename, no fsync — while the class docstring claims "append-only, immutable entries". Append-only that is not enforced is append-only that is not true.

## Decision

`evidence_items`, `event_transitions`, `event_evidence_links`, `event_locations`, `event_merges`, `event_splits`, `parser_executions`, `source_document_versions`, `audit_entries` and `federated_events` are `APPEND_ONLY`.

Enforced by `REVOKE UPDATE, DELETE` from the application role **plus** a `BEFORE UPDATE OR DELETE` trigger that raises. Corrections are new rows. Deletions are tombstones.

`public_events` is `DERIVED` and written only by the projection worker.

## Alternatives considered

| Alternative | Rejected because |
| --- | --- |
| Mutable state with an audit-log side table | The audit log drifts from reality and is not the thing being read |
| Full event sourcing of every entity | Over-engineering. Identity, organisations and nodes are ordinary CRUD |
| Temporal tables / system versioning | Vendor-specific, less explicit, harder for contributors to reason about |
| Application-enforced immutability only | The audit shows exactly this failing — an unenforced claim |

## Consequences

Full auditability and reconstructability. Corrections are visibly corrections. Projections can be rebuilt at any time, which is what makes replay-divergence detection meaningful.

Cost: storage grows monotonically (mitigated by partitioning and cold archival); every write path must be designed as an append; "fixing a typo" becomes a new row, which is unfamiliar to contributors and needs documenting.

## Risks

- **Contributors attempting `UPDATE`.** The trigger converts this from a silent data loss into a loud failure. Documented in [contribution-architecture.md](../architecture/contribution-architecture.md).
- **Unbounded growth.** Partitioning plus hot/warm/cold (ADR 0013).
- **Privacy tension with erasure.** Resolved in ADR 0013 and [data-model.md](../architecture/data-model.md) §10: personal data is separately erasable; civic evidence is immutable and survives pseudonymously.

## Measurement

- Trigger firings: must be zero (each indicates a code path attempting mutation).
- `event_corrections_total` by type — corrections are expected and healthy.
- Evidence-row count for a superseded parser version: must never decrease (I8).

## Introduction trigger

n/a — day one. Retrofitting immutability onto mutable data is not possible.

## Rollback strategy

None. This is foundational; reversing it would forfeit invariants 4, 10 and 15, which are the product.
