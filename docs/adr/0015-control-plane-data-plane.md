# 0015 — Control plane and data plane separation

Status: Accepted
Date: 2026-07-27

## Context

JanSetu's control surface (node registration, signing keys, permissions, jurisdiction, categories, protocol versions, adapter configuration, federation metadata, retention policy, feature flags) has a very low write rate and very high consequence.

Its data surface (ingestion, storage, parsing, evidence, projection, querying, geospatial, reports, notification) has high volume and lower per-operation consequence.

At national scale the control plane is global while data planes are regional. A global component that can halt every region is an availability hazard.

## Decision

**Logical separation from PMVP. Physical separation at national scale.**

At PMVP and city scale: separate modules, separate database schemas, separate roles. One deployable.

At national scale: a small globally-replicated control plane; independent regional data planes.

### Control-plane failure must not stop the data plane

Every data-plane process caches control-plane policy locally with a TTL and a **last-known-good fallback that never expires into a permissive state**.

With the control plane unreachable:

| Continues | Blocked |
| --- | --- |
| Ingestion | New node registration |
| Parsing and evidence | Key rotation |
| Projection | New permission grants |
| Citizen querying | Jurisdiction changes |
| Geospatial impact | Policy changes |
| Notification decisions and delivery | |

### The revocation exception

**Revocations must not be cached away.** They are pushed over the event stream and applied immediately. If the stream is also unreachable, the signing-key policy TTL is deliberately short — **5 minutes** — bounding exposure.

This is the one place in the architecture that chooses consistency over availability, and it is chosen knowingly: a compromised node key ([threat-model.md](../architecture/threat-model.md) T2) allows impersonation of a government authority, and bounded exposure is worth a bounded availability cost.

## Alternatives considered

| Alternative | Rejected because |
| --- | --- |
| No separation | A control-plane outage would halt civic information delivery during exactly the events that matter |
| Physical separation from PMVP | Two deployables for ten publishers; operational cost for no benefit |
| Data plane calls the control plane synchronously per request | Latency, and a hard availability coupling |
| Cache everything including revocations | A compromised key would stay valid for the full TTL — unacceptable |
| No caching at all | Control-plane outage becomes a total outage |

## Consequences

Regional data planes are independently operable and independently promotable during disaster recovery. Control-plane changes are rare, reviewed, and audited. The blast radius of a control-plane incident is "no new configuration", not "no civic information".

Cost: cache-coherence reasoning; a stale-policy window for non-revocation changes; the operational burden of a globally-replicated component at national scale.

## Risks

- **Stale permissive policy.** Mitigated by the never-expire-into-permissive rule: on cache expiry with the control plane unreachable, the last-known-good policy is retained, and *additive* grants are never inferred.
- **Revocation not propagating.** Mitigated by stream push plus a short TTL plus an alert on propagation lag.
- **Split-brain between regions.** Control-plane writes are globally serialised; data planes never write control-plane state.

## Measurement

- Policy propagation lag (control plane → data plane), p99.
- Revocation propagation time — must be under the 5-minute TTL.
- Data-plane availability during control-plane incidents (target: unaffected).
- Cache staleness distribution.

## Introduction trigger

Logical separation: day one. Physical separation: multi-region operation, or thousands of nodes making control-plane load distinct from data-plane load — Scenario C/D.

## Rollback strategy

Physical separation can be collapsed back into a single deployment. The module and schema boundaries make this a deployment change rather than a refactor.
