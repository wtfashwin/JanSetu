# 0010 — Search architecture: Postgres FTS first, search never determines truth

Status: Accepted
Date: 2026-07-27

## Context

JanSetu has distinct search use cases: citizen event search, moderator evidence search, historic event search, source-document search, similar-event discovery, and filtered browsing by geography, time, category and authority.

Cerevra's search is BM25 over a **pickled** index (`cerevra/security/namespace.py:68` → `bm25_state.pickle`) — arbitrary code execution on untrusted load, and state that breaks across library versions, defeating the "deterministic rebuild" claim it makes.

## Decision

**PMVP and city scale: PostgreSQL full-text search.** Generated `tsvector` column, GIN index, combined with jurisdiction, category, time and authority filters in the same query — which is exactly what a separate search engine makes harder.

**OpenSearch is introduced only when measured:** FTS p95 breaches the search SLO on *production-shaped* data (real notice text, real query distribution), **or** search load measurably degrades transactional queries. Moving search to the read replica must be tried first, and usually resolves the second condition alone.

**Vector/semantic search** arrives only if moderators measurably fail to find relevant prior evidence with lexical search — tracked as a "no useful result" rate above 20% over 30 days.

Two constraints hold at every scale:

1. **Search is `DERIVED` and fully rebuildable** (I21). No field exists only in the index. Rebuild is exercised regularly, not merely documented — an index that has never been rebuilt is an index that cannot be.
2. **Neither lexical nor vector search may determine event truth.** Search returns candidates; the projection determines state. Enforced structurally: the `public_events` module has no dependency on the search client, checked by `import-linter`.

No pickle anywhere.

## Alternatives considered

| Alternative | Rejected because |
| --- | --- |
| OpenSearch from day one | A second store to keep consistent, before FTS has been shown insufficient. ~$1,200/mo |
| Vector search as primary retrieval | An embedding similarity score is not evidence. Would violate invariant 7 |
| External search SaaS | Data residency; cost at Scenario C+; loss of combined geospatial + text filtering |
| Pickled in-process index (Cerevra's model) | RCE on load; non-deterministic across versions; no concurrent access |

## Consequences

One store through city scale. Geospatial, temporal, authority and text filters compose in a single query plan. Rebuild is a single job. Cost is zero until the trigger fires.

Cost: FTS ranking is weaker than a dedicated engine, and Indian-language and transliteration handling requires explicit configuration (dictionaries, `unaccent`, alias matching against `locations.aliases`).

## Risks

- **Indian-language and transliteration matching.** "Aundh"/"औंध"/"Aundha" must match. Mitigated by curated aliases on `locations` (GIN-indexed) rather than by relying on stemming.
- **Search load affecting OLTP.** Mitigated by the read replica; that is the trigger's first remedy.
- **Index drift.** Mitigated by a row-count delta metric and alert, plus periodic rebuild.

## Measurement

- Search p95 latency against the SLO — the OpenSearch trigger.
- Share of primary query time attributable to search.
- Moderator "no useful result" rate — the vector-search trigger.
- Rebuild duration and index/authoritative row-count delta.

## Introduction trigger

Per [migration-triggers.md](../architecture/scaling/migration-triggers.md). Expected: OpenSearch at Scenario C; vector search possibly never.

## Rollback strategy

Point search back at Postgres FTS with reduced ranking quality. Because nothing originates in the index, no data is at risk.
