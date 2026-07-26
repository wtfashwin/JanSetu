# 0013 — Hot / warm / cold retention, and immutability versus erasure

Status: Accepted
Date: 2026-07-27

## Context

Append-only evidence (ADR 0002) plus immutable raw documents (ADR 0005) means storage grows monotonically: 80 TB of objects and 4 TB of Postgres at Scenario D, growing 50 TB/year.

Two forces pull against each other. Invariants 7 and 10 require immutable raw documents and reconstructable history. Privacy law and product principle 6 require that a person can be erased.

Cerevra leaves this tension unresolved: it has GDPR deletion, audit and lineage modules (`cerevra/compliance/`), and it also cascades chunk-file removal on delete — so "immutable raw" and "erasure" collide with no stated resolution.

## Decision

### Tiers

| Tier | Contents | Store | Access |
| --- | --- | --- | --- |
| Hot | Active events, evidence < 30 d, source health, pending notifications | Postgres + cache + CDN | ms |
| Warm | Resolved events < 1 y, moderation history, recent documents | Postgres partitions + search index | 10s of ms |
| Cold | Historic document versions, archived evidence, audit > 1 y, replay snapshots | Object storage archive class + Parquet | hours |

Movement is by `DETACH PARTITION` + archive, never by bulk `DELETE`. Restoration is a **documented and tested** procedure with a stated latency.

### The immutability/erasure resolution — structural, not policy

**Personal data and civic evidence are separated at the storage layer, so they can have different lifecycles:**

| Data | Store | Erasable |
| --- | --- | --- |
| Raw source documents (public notices, no personal data by nature) | Object-locked bucket | **No** — immutable |
| Community-report media (may contain personal data) | **Separate bucket, no object lock** | **Yes** |
| Reporter identity | Separate mapping table | **Yes** |
| Community report content | `community_reports`, pseudonymous | No — survives without the person |
| Saved routes, subscriptions, impact assessments, notification history | Separate schema, encrypted | **Yes** — hard delete |
| Evidence, transitions, audit | Append-only | No |

An erasure request deletes routes, subscriptions, impact assessments, notification history, the pseudonym→identity mapping, and report media. **The civic evidence survives; the person does not.**

This works because the separation is structural. A policy saying "we will not put personal data in the immutable bucket" degrades; two buckets with different lock configurations do not.

### Retention

| Data | Hot | Warm | Cold | Total |
| --- | --- | --- | --- | --- |
| Source documents | 90 d | 1 y | indefinite | indefinite |
| Evidence | 90 d | 1 y | indefinite | indefinite |
| Events + transitions | active | 1 y | indefinite | indefinite |
| Ingestion commands/attempts | 90 d | — | archive | 90 d + archive |
| Notification decisions/deliveries | 90 d | — | — | **90 d** |
| Route impact assessments | 30 d | — | — | **30 d** |
| Audit entries | 1 y | — | 6 y | 7 y |
| Moderation actions | 1 y | 2 y | — | 3 y |

Personal data has the shortest retention in the system, deliberately.

## Alternatives considered

| Alternative | Rejected because |
| --- | --- |
| Delete everything after N years | Destroys the audit and replay guarantee |
| Keep everything hot | Cost; and hot-tier size drives replay and maintenance windows |
| Anonymise in place instead of separating | Anonymisation of free-text reports is unreliable; separation is verifiable |
| One bucket with lifecycle policy only | Object lock would make report-media erasure impossible |
| Legal hold as a manual process | Modelled as a flag that suspends lifecycle transitions for named partitions |

## Consequences

Storage cost tiers down automatically. Erasure is genuinely possible without weakening immutability. Replay from cold is possible, with an honest latency. Legal holds are expressible.

Cost: two buckets and two lifecycles; cold restoration takes hours; partition management is ongoing operational work.

## Risks

- **Personal data leaking into the immutable bucket** — the failure that would break the whole scheme. Mitigated by a write-path check (only the ingestion worker writes there, only content-addressed raw source bytes) and by a periodic scan.
- **Archived data unrestorable when audited.** Mitigated by restoring a sampled partition quarterly. An untested archive is not an archive.
- **Legal hold missed** during a lifecycle transition. Mitigated by the hold flag being checked in the transition job, with a test.

## Measurement

- Storage by tier against [workload-model.md](../architecture/scaling/workload-model.md) projections.
- Restore drill success rate and measured latency.
- Erasure request completion time.
- Objects in the locked bucket failing the "raw source document only" check: must be zero.

## Introduction trigger

Retention policy from day one (it is cheap to define and expensive to retrofit). Tiering when the bucket exceeds 1 TB or a table exceeds 50 GB — Scenario B.

## Rollback strategy

Lifecycle transitions can be paused. Archived data can be restored to warm. The bucket separation is not reversible and should not be.
