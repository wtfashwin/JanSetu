# 0005 — Immutable object storage for raw source documents

Status: Accepted
Date: 2026-07-27

## Context

Product invariant 8 requires that original source material remains immutable. When a citizen disputes an event, or a moderator investigates, or a court asks what a municipal notice actually said, the answer must be the bytes as fetched — not a re-render, not a parsed summary.

Volume reaches 80 TB at Scenario D with 200 KB average documents. That is not database data.

## Decision

Raw document bytes go to S3-compatible object storage:

- **Content-addressed keys**: `{node}/{source}/{sha256}` — full-width SHA-256, never truncated.
- **Bucket versioning + object lock** in governance mode.
- **No overwrite path exists in application code.** Immutability is enforced by the bucket policy, not by discipline.
- Digest verified on read; mismatch pages.
- Lifecycle: Standard 90 d → Infrequent Access to 1 y → Archive thereafter.
- Cross-region replication at national scale.

**Ordering with the database is fixed:** the object is written and verified **first**, then the referencing row commits. An orphan object is harmless and swept; a dangling reference is not tolerable.

Community-report media goes to a **separate, erasable bucket** without object lock.

## Alternatives considered

| Alternative | Rejected because |
| --- | --- |
| Postgres large objects / `bytea` | 80 TB in a transactional database; backup and restore times become impossible |
| Filesystem (Cerevra's model) | No durability guarantee, no replication, no lifecycle, no versioning; two processes can corrupt shared state |
| Store only a hash and re-fetch on demand | Government sites remove notices. **The evidence would evaporate exactly when disputed** |
| One bucket for documents and media | Object lock would make erasure impossible — see below |

## Consequences

Raw evidence survives source-website changes and deletions. Content addressing makes duplicate detection free at the storage layer. Storage cost is low and tiers down automatically.

Cost: two stores without a shared transaction, requiring the fixed ordering and an orphan sweeper (the narrow saga in [pattern-applicability.md](../architecture/pattern-applicability.md)). Archive restoration takes hours.

## Risks

- **Dangling reference** if ordering is violated. Mitigated by the fixed order plus a chaos test that kills between the object write and the commit.
- **Orphan accumulation.** Mitigated by a sweeper; orphans are harmless meanwhile.
- **Immutability vs erasure.** This is the reason for two buckets. Raw civic documents are *public notices* and contain no personal data by nature, so object lock is safe. Report media may contain personal data and therefore lives in an erasable bucket with no lock. **The tension is resolved structurally rather than by policy**, which is how it stays resolved.
- **Archive latency** during an audit. Documented and accepted: archived documents serve audit and replay, not citizen reads.

## Measurement

- Object digest mismatches on read: must be zero.
- Orphan objects swept per day.
- Dangling references: must be zero (asserted by a periodic reconciliation job).
- Storage growth against the [workload-model.md](../architecture/scaling/workload-model.md) projection — a large divergence falsifies assumption A1.

## Introduction trigger

n/a — day one. Migrating bytes out of a database later is far more expensive than starting here.

## Rollback strategy

None wanted. If a specific provider fails, the content-addressed key scheme makes migration to another S3-compatible store a copy operation with no reference rewriting.
