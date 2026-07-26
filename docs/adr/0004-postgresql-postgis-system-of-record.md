# 0004 — PostgreSQL + PostGIS as the system of record

Status: Accepted
Date: 2026-07-27

## Context

JanSetu needs: multi-table transactional integrity (command and effect must commit together), exact geometry operations (route × event intersection is a core feature), full-text search at PMVP scale, row-level security for civic-node isolation, append-only enforcement via triggers and grants, and a workload that is 1,060 writes/s at national peak — modest.

The Cerevra audit shows the alternative failing. Its storage layer is a local directory tree with JSON manifests and a **pickled** BM25 index (`cerevra/security/namespace.py:68` → `bm25_state.pickle`). Consequences observed: no cross-record transactions, per-process `threading.Lock` as the only concurrency control (so two processes corrupt shared rotation state), a manifest rewritten in full on every ingest, `mark_applied` at O(all files) per call, arbitrary code execution on index load, and index state that breaks across library versions — defeating the "deterministic rebuild" claim.

## Decision

PostgreSQL 16 with PostGIS is the single system of record for all `AUTHORITATIVE` and `APPEND_ONLY` data.

- Geometry: `geography(*, 4326)`, GiST-indexed.
- Search: `tsvector` + GIN at PMVP and city scale.
- Isolation: row-level security on `civic_node_id` and `user_id`.
- Immutability: `REVOKE` + triggers.
- Ordering: identity columns — the **sole** sequence authority.
- Queue: a table claimed with `FOR UPDATE SKIP LOCKED`.
- Jurisdiction hierarchy: `ltree`.

Raw document bytes live in object storage (ADR 0005); everything else lives here.

## Alternatives considered

| Alternative | Rejected because |
| --- | --- |
| Filesystem + manifests (Cerevra's model) | Every failure above |
| MongoDB / document store | No transactional multi-document integrity at the time of design choice; no exact geometry; weaker constraint enforcement |
| Cassandra / DynamoDB | Write throughput is not the problem; relational integrity is load-bearing; no exact geometry |
| Separate spatial database | An extra store to keep consistent for a capability PostGIS already provides natively |
| SQLite for PMVP | Would not survive the transition to concurrent workers; the migration would land at the worst moment |

## Consequences

One store for transactional integrity, geometry, search, isolation and queueing — which is what allows the PMVP to be operated by one to three engineers. Constraints enforce invariants rather than documenting them. Managed hosting is universally available with PITR.

Cost: a single point of failure at PMVP (accepted; mitigated by replicas at city scale). Vertical scaling has a ceiling — reached, per [workload-model.md](../architecture/scaling/workload-model.md), only for reads, which replicas address.

## Risks

- **Table bloat on append-only tables.** Mitigated by partitioning (trigger: 50 GB) and tuned autovacuum.
- **Connection exhaustion** with many workers. Mitigated by PgBouncer at city scale.
- **PostGIS query cost at scale.** Mitigated by GiST tuning, simplified geometry, and an H3 pre-filter only if measured ([migration-triggers.md](../architecture/scaling/migration-triggers.md)).
- **FTS insufficiency.** Explicit trigger for OpenSearch (ADR 0010).

## Measurement

- Primary CPU, replication lag, top wait events.
- `geospatial_query_duration_seconds` p95 against the 500 ms SLO.
- Table sizes against the 50 GB partitioning trigger.
- Autovacuum duration against the availability budget.

## Introduction trigger

n/a — day one.

## Rollback strategy

None wholesale. Individual workloads migrate out on their own triggers: search to OpenSearch (0010), analytics to ClickHouse, distribution to a broker (0006) — each with its own rollback. Postgres remains authoritative in every case.
