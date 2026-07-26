# 0008 — Civic-node isolation and hierarchical jurisdictions

Status: Accepted
Date: 2026-07-27

## Context

Product invariant 11: civic nodes cannot silently rewrite one another's signed events. A node's authority is bounded by **geography and subject matter** — a traffic-police node may confirm road closures in Pune but not water interruptions in PCMC.

Cerevra's namespace model cannot express this. `cerevra/security/namespace.py:15` is `^[a-zA-Z0-9_-]+$`, max 64 characters, and `_is_safe_path` explicitly rejects `/`. It is a flat opaque tenant token supporting none of: ancestor-scoped access control, inherited retention, subtree replay scoping, jurisdiction-filtered search, encryption boundaries, scoped deletion, or federation trust.

## Decision

**Jurisdictions are first-class relational rows with an `ltree` materialised path**, not a string token:

```
india
india.maharashtra
india.maharashtra.pune
india.maharashtra.pune.traffic_police
india.maharashtra.pcmc.water
```

Civic nodes are bound to a jurisdiction node and to a set of authorised categories. Authorisation is `decide(principal, permission, scope)`, where scope resolves against the `ltree` path — "all of Pune" is an ancestor query, not an enumeration.

Isolation is enforced at three independent layers:

1. **Postgres RLS** on `civic_node_id`, bound to a request-scoped session principal — stops a bad write even when the application is wrong.
2. **The authorisation decision function** — application-level, with logged deny reasons.
3. **Runtime assertions that raise** on any cross-node row observed on a read path — adopted from `cerevra/query/invariants.py:39-77`.

Federated remote events live in a separate table with **no local write grant** (I20).

Assertions alone are insufficient (they only see reads); RLS alone is insufficient (it cannot express category scope). Both are required.

## Alternatives considered

| Alternative | Rejected because |
| --- | --- |
| Flat namespace tokens (Cerevra's model) | Cannot express a jurisdiction hierarchy; every listed requirement fails |
| Application-only filtering | One forgotten `WHERE` clause is a cross-node breach |
| Schema-per-node | Thousands of schemas at national scale; migrations become intractable |
| Database-per-node | Cross-jurisdiction events (a highway spanning cities) become impossible |
| Closure table instead of `ltree` | Viable; `ltree` is simpler and indexed natively. Revisit only if path mutation becomes frequent |

## Consequences

Authority is territorial and categorical, matching how civic institutions actually work. Ancestor queries make "all of Maharashtra" a single indexed lookup. Retention, visibility and encryption classes inherit down the tree. Replay scopes to a subtree, which is what makes partitioned replay possible at national scale.

Cost: jurisdiction reference data must be curated and maintained (boundary changes, new wards, municipal reorganisation) — an ongoing data problem, not a one-off import.

## Risks

- **Jurisdiction boundary disputes** between authorities. Represented as data, not resolved by the platform.
- **Connection-pool principal leakage** — a stale session variable would defeat RLS. Mitigated by setting and clearing the principal per request with an explicit reset, and by an integration test for pool reuse.
- **Path mutation** on reorganisation. Handled as an explicit migration with recorded history; ids never change (only paths).

## Measurement

- `authorisation_denials_total` by permission and reason.
- Cross-node writes detected by a periodic audit query: must be zero.
- RLS integration tests exercising every node-scoped table.

## Introduction trigger

n/a — day one. Retrofitting isolation is not feasible.

## Rollback strategy

None.
