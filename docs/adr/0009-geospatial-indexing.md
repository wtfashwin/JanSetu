# 0009 — Geospatial: PostGIS authoritative, coarse indexing only on evidence

Status: Accepted
Date: 2026-07-27

## Context

JanSetu's core citizen feature is "does this closure affect my commute?" — an exact geometric question over points, road-segment LineStrings, polygons, transit lines and user routes. Wrong answers here are the product failing.

Workload: 0.002 impact calculations/s at PMVP, 300/s peak at city scale, 10,000/s peak at national.

Coarse geospatial indexes (H3, S2, geohash) are widely used and frequently adopted before they are needed.

## Decision

**PostGIS is authoritative at every scale.** SRID 4326, `geography` type where distance semantics matter, GiST indexes on all geometry columns.

Route impact: `ST_DWithin(route.geom, event.geom, tolerance)` → classification into `blocked` / `delayed` / `nearby` / `clear`. Computed **per event transition**, not on a timer — one route × one transitioning event, never a periodic cross-product.

**No H3 or S2 at PMVP or city scale.** Coarse indexing is introduced only when route × event intersection p95 exceeds 500 ms *with a correctly-tuned GiST index and pre-simplified geometry already in place*. Even then it is a candidate-discovery pre-filter; the exact PostGIS intersection always runs.

Coarse H3 cells (~res 8, 0.7 km²) **are** used from day one for a different purpose: coarse public display of community-report locations, which is a privacy control rather than an index.

Progressive optimisations before any coarse index: GiST tuning and `CLUSTER`; pre-simplified geometry columns per zoom level; a materialised active-event geometry view; bounding-box pre-filter.

## Alternatives considered

| Alternative | Rejected because |
| --- | --- |
| H3 from day one | At 1,000 routes and 5,000 events GiST is sub-millisecond. Two spatial models to keep consistent, for nothing |
| Geohash | Rectangular cells distort badly at the latitudes of interest; H3 is superior if a coarse index is ever needed |
| Bounding boxes only | Wrong answers. A road parallel to a closure shares a bbox |
| A dedicated spatial engine | PostGIS is the reference implementation; a separate store adds a consistency problem |
| Precomputing all route × event pairs | O(routes × events); at Scenario C that is 10¹³ pairs |

## Consequences

Exact answers at every scale. One spatial model. Jurisdiction containment, event geometry and route intersection all use the same engine and the same indexes.

Cost: PostGIS query cost grows with data; mitigated by the optimisation ladder above, whose steps are cheap and reversible.

## Risks

- **Geometry validity.** Invalid geometry is rejected on write, never silently repaired — a silently "fixed" closure geometry is a wrong answer that looks right.
- **Coordinate confusion.** SRID 4326 enforced by column type; `geography` chosen deliberately so distance is in metres.
- **Cross-jurisdiction events** (a highway spanning cities). Owned by the issuing node's jurisdiction; neighbouring jurisdictions hold references.
- **Reference-data drift** — road segments change. `road_segments` carries an external reference as an *attribute*, and geometry corrections append rather than overwrite.

## Measurement

- `geospatial_query_duration_seconds` p95 against the 500 ms SLO — this is the H3 trigger.
- `route_impact_calculations_total` by impact class.
- Index bloat and `CLUSTER` frequency.

## Introduction trigger

H3/S2 coarse index: route × event p95 > 500 ms with GiST tuned and simplified geometry in place. Not expected before Scenario C.

## Rollback strategy

Drop the coarse index; queries return to pure PostGIS with no correctness change, because PostGIS was authoritative throughout.
