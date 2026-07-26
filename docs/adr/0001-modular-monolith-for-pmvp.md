# 0001 — Modular monolith for the PMVP

Status: Accepted
Date: 2026-07-27

## Context

JanSetu is greenfield, built by one to three engineers, and must be operable by that same team during a civic emergency. The PMVP workload ([workload-model.md](../architecture/scaling/workload-model.md), Scenario A) is 0.02 ingestion commands/s and 9 citizen reads/s — three to four orders of magnitude below what a single process handles.

The Cerevra audit provides a direct cautionary example: a system with `distributed/`, `sharding/`, `billing/`, `agentic/` and `plugins/` subsystems built on top of a storage layer that is a local directory tree, plus four overlapping query-service objects totalling ~80 KB and two independent WAL implementations. Complexity arrived before the requirement did, and the resulting seams hid a real defect for a long time.

## Decision

One deployable, one container image, internally divided into fourteen modules with a strictly inward dependency direction (`api → application services → domain → persistence`), enforced in CI by `import-linter`. Multiple process types (`api`, `worker`, `scheduler`) run from the same image.

Module boundaries are drawn so that `ingestion`, `notifications`, and the projection worker can each become separate deployables **without redrawing them**.

## Alternatives considered

| Alternative | Rejected because |
| --- | --- |
| Microservices from day one | Distributed transactions for a workload one process handles; three engineers cannot operate twelve services during a flood |
| Single-file application | Boundaries would need discovery later, under pressure, with no seams |
| Serverless functions | Long-running ingestion, PostGIS connection pooling, and durable leases all fit poorly |

## Consequences

One transaction scope covering command and effect — which is what makes invariant I11 achievable at all. One deployment, one log stream, one dashboard. Refactoring a boundary is a rename, not a migration. Modules can be extracted later on measured evidence.

Cost: boundaries erode without enforcement, and a shared schema tempts cross-module joins. Hence the lint rule rather than a convention.

## Risks

- **Boundary decay.** Mitigated by `import-linter` with explicit forbidden edges (`evidence → notifications`, `geospatial → evidence`).
- **Shared-schema coupling.** Mitigated by module-owned table ranges and review discipline.
- **Scaling the wrong thing.** Mitigated by separate process types from the same image, so ingestion can scale without scaling the API.

## Measurement

- CI lint failures on forbidden imports: must be zero.
- Cross-module direct table access in review: must be zero.
- Deploy frequency and mean time to restore, tracked as operability indicators.

## Introduction trigger

n/a — this is the starting state. Extraction triggers are in [migration-triggers.md](../architecture/scaling/migration-triggers.md): >30% capacity waste from co-scaling, plus an interface stable for ≥2 releases, plus an independent failure domain.

## Rollback strategy

n/a. The reverse operation — merging an extracted service back — is documented as a reverse trigger in [migration-triggers.md](../architecture/scaling/migration-triggers.md).
