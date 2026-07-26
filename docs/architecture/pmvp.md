# JanSetu PMVP Architecture

Optimised for: one to three engineers · rapid iteration · correctness · low cost · trivial local development · open-source contribution · minimal operational burden.

Workload it must serve ([workload-model.md](scaling/workload-model.md), Scenario A): 0.02 ingestion commands/s peak, 9 citizen reads/s peak, 2 GB objects, 400 MB Postgres, full replay in 2.5 s.

**Every one of those numbers is trivially served by a single application process and a single database.** The engineering difficulty in the PMVP is not scale. It is correctness: shipping a system whose trust claims are actually true.

---

## 1. Shape

```
Citizen PWA (Next.js)  ·  Publisher console  ·  Moderation console
                              │  HTTPS
                              ▼
                    FastAPI modular monolith
                    (API + workers, one image)
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
PostgreSQL 16 + PostGIS   S3-compatible          Redis
(system of record)        object storage         (non-authoritative)
                          (immutable raw docs)
```

One container image. Two processes from it: `api` and `worker`. One managed Postgres. One bucket. One small Redis.

**Redis holds only:** rate-limit counters, short-TTL response cache, worker coordination hints, notification-dedup hints. Every accessor has a correct behaviour when Redis is empty or absent, and the test suite runs a mid-suite `FLUSHALL` that must not cause a single failure (invariant I10).

**There is no message broker.** The queue is a Postgres table. At 0.02 commands/s this is not a compromise — it is the only choice that gives command and effect a shared transaction, which is what invariant I11 requires.

---

## 2. Modules

Dependency direction is strictly inward: `api → application services → domain → persistence`. Enforced in CI by `import-linter`, not by convention.

| Module | Responsibility | May depend on |
| --- | --- | --- |
| `identity` | Users, sessions, OIDC, principals | — |
| `organisations` | Publishing organisations | `identity` |
| `civic_nodes` | Node registration, jurisdiction, categories, keys | `organisations` |
| `sources` | Source registry, adapter config, health, breakers | `civic_nodes` |
| `ingestion` | Command table, workers, fetch, object storage, versioning | `sources` |
| `source_documents` | Document identity, versions, digests, object refs | `ingestion` |
| `evidence` | Parser execution, evidence items, provenance | `source_documents` |
| `public_events` | Event projection, matching, merge/split | `evidence` |
| `event_state_machine` | Versioned trust rules, legal transitions | `evidence` |
| `locations` | Places, road segments, jurisdiction geometry | — |
| `geospatial` | PostGIS ops, route×event intersection, impact class | `locations` |
| `moderation` | Queues, actions, dispute handling | `public_events` |
| `audit` | Append-only audit entries | all (write-only sink) |
| `notifications` | Subscriptions, decisions, deliveries | `public_events`, `geospatial` |

Forbidden edges (lint-enforced): `evidence → notifications`, `geospatial → evidence`, anything → `api`, anything → `moderation` except `api`.

The boundaries are drawn so that `ingestion`, `notifications`, and the projection worker can each become a separate deployable at city scale **without redrawing them**. That is the point of drawing them now.

---

## 3. The write path

```
scheduler tick
  └─ for each due source:
       INSERT ingestion_commands (idempotency_key, type=FETCH_SOURCE, …)
       ON CONFLICT (idempotency_key) DO NOTHING
                              │
worker: SELECT … WHERE state='PENDING' FOR UPDATE SKIP LOCKED LIMIT 10
        UPDATE state='CLAIMED', lease_expires_at = now()+interval '5 min'
                              │
        fetch source (breaker-aware, backoff+jitter)
        INSERT ingestion_attempts (append-only)
                              │
        compute sha256(raw bytes)
        ── if digest already known for this source_document → no new version
        ── else: PUT s3://…/{sha256}  ← object written and verified FIRST
                              │
        BEGIN
          INSERT source_document_versions (content_sha256, object_key, …)
          INSERT parser_executions (parser_version, …)
          INSERT evidence_items (…) ON CONFLICT DO NOTHING
          INSERT ingestion_commands(type=PROJECT_EVENT) for affected events
          UPDATE ingestion_commands SET state='SUCCEEDED' WHERE id=$1 AND claimed_by=$me
        COMMIT                ← this commit IS the acknowledgement
```

Four properties worth stating explicitly, each traceable to an audit finding:

1. **Acknowledgement is the commit.** There is no separate ack step that a crash can lose (I11). Cerevra's recovery engine reads a WAL stack its ingestion path never writes to (`cerevra/wal/recovery.py:17`); making the log a table removes the possibility of that class of mismatch.
2. **Ordering across stores is fixed.** S3 object first, then the row referencing it. An orphaned object is harmless and swept; a dangling reference is not tolerable.
3. **Sequence numbers come from one place** — a Postgres identity column — not from two disagreeing computations (Cerevra `handler.py:350-355` vs `writer.py:271-280`).
4. **No fire-and-forget threads.** Every asynchronous effect is a row. Cerevra's embedding path (`handler.py:411-449`) demonstrates the failure mode this avoids: a daemon thread, two in-memory retries, a log line, and silent permanent loss.

### Failure handling

- Retry: exponential backoff with full jitter, capped attempts, breaker-aware.
- After N attempts → `DEAD_LETTER`, surfaced in the moderation console, counted by `parser_failures_total`. Never silently dropped, never infinitely retried.
- Lease expiry → automatic re-claim. This is the entire recovery procedure; there is no separate startup recovery pass to get wrong.

---

## 4. The read path

```
GET /v1/events?bbox=…&category=…&state=…
  → Redis cache-aside (TTL 30 s, key includes bbox+filters+ruleset_version)
  → Postgres:  public_events (DERIVED projection)
               JOIN event_locations (PostGIS GiST index)
  → response includes: state, evidence summary, last_evidence_at,
                       evidence_age_seconds, source_health, as_of
```

Every response carries freshness (I5) and an `as_of` timestamp. The cache TTL is short and any event transition purges the affected keys, because a cached "road open" surviving a closure is exactly the failure JanSetu exists to prevent.

---

## 5. Evidence → event projection

`evidence_items` and `event_transitions` are `APPEND_ONLY`. `public_events` is `DERIVED` and written **only** by the projection worker.

```
project(event_id):
    load checkpoint (last_applied_sequence, projection_hash, ruleset_version)
    for each evidence item with sequence > last_applied, ordered by (sequence, evidence_id):
        decision = ruleset[version].evaluate(projection, evidence)   # pure, deterministic
        if decision.transitions:
            INSERT event_transitions (from, to, cause, ruleset_version, evidence_id)
            projection = apply(projection, decision)
    UPSERT public_events   ← the only writer
    store checkpoint (new_sequence, hash(projection), ruleset_version)
```

The projector is a pure fold — the strongest idea carried over from Cerevra (`cerevra/wal/replay.py:146-306`) — with the addition Cerevra lacks: **a stored projection hash, so divergence is detectable** (I12, I13). Cerevra stores no such hash, so a code change silently produces a different manifest with no signal. JanSetu cannot ship without this, because "every event is reconstructable" is its central claim.

Determinism constraints on the projector: no `now()`, no network calls, no map-iteration-order dependence, and LLM outputs are read from storage rather than regenerated. This is what makes invariant 7 ("LLM extraction is never event truth") compatible with invariant 12 ("identical inputs produce identical projections").

### Answering "why is this event OFFICIALLY_CONFIRMED?"

Read its `event_transitions`. Each row names the rule version, the triggering evidence item, and the actor. Follow the evidence to its `parser_execution`, to its `source_document_version`, to the immutable object in S3 and the URL it was fetched from. Every link is stored. Nothing is inferred at read time.

---

## 6. Trust rules

Deterministic, versioned, inspectable. Not a score.

| Condition | Resulting state |
| --- | --- |
| No evidence | Not published |
| One anonymous report | `UNVERIFIED` |
| ≥3 independent reports, distinct reporters and cells, within a window | `COMMUNITY_CORROBORATED` |
| Official evidence from a node authorised for that jurisdiction + category | `OFFICIALLY_CONFIRMED` |
| Official and community evidence disagree | `CONFLICTING` |
| Evidence older than category freshness budget | `STATUS_UNKNOWN` |
| Ambiguous or unresolvable location | `DRAFT` → moderation |
| Positive evidence of resolution | `RESOLVED` |
| No evidence at all for N hours | `STATUS_UNKNOWN` — **never** `RESOLVED` (I4) |

`OFFICIALLY_CONFIRMED` requires official evidence by database `CHECK` constraint, not merely by rule (I3). The rule can have a bug; the constraint is the backstop.

### Where the LLM is allowed

Permitted: extracting candidate event type, place names, time windows, and a summary from unstructured notice text, producing an *evidence candidate* tagged with model id and prompt version.

Forbidden: publishing, confirming, resolving, inventing an expiry, identifying participants, inferring intent, or overriding any rule. An LLM-derived candidate with no corroborating structured extraction does not by itself advance an event past `UNVERIFIED`.

---

## 7. Geospatial

PostGIS, SRID 4326, geography type for distance work. GiST indexes on all geometry columns.

- Events carry a geometry: point, road-segment LineString, or polygon.
- Saved routes are LineStrings with a per-user buffer tolerance.
- Route impact: `ST_DWithin(route.geom, event.geom, tolerance)` → severity classification.
- Jurisdiction boundaries are polygons; an event's jurisdiction is resolved by containment.

**No H3 or S2 at PMVP.** At 1,000 routes and 5,000 events a correctly-indexed GiST query is sub-millisecond. Coarse indexing is introduced only on the measured trigger in [migration-triggers.md](scaling/migration-triggers.md), and even then PostGIS stays authoritative.

Route impact is computed **per event transition**, not on a timer — one route × one transitioning event, not a periodic cross-product.

---

## 8. Search

Postgres full-text search with a generated `tsvector` column and a GIN index, combined with jurisdiction, category, time and authority filters. No OpenSearch. No vector search.

Search is `DERIVED` and rebuildable from authoritative rows (I21). Nothing exists only in the index.

---

## 9. Notifications

```
event transition committed
  → INSERT notification_decisions (one row per candidate subscription)
  → worker evaluates: relevance, freshness budget, per-user cap, quiet hours
  → INSERT notification_deliveries (subscription_id, event_id, transition_sequence)
       UNIQUE ← retries cannot duplicate (I9)
  → provider send; failures retried against the same row
```

Decisions are recorded even when the outcome is "do not send", with the reason. "We chose not to alert you" must be auditable, not invisible.

---

## 10. Security posture at PMVP

- OIDC for users and maintainers; sessions in Postgres, not Redis.
- Atomic permissions (`event.publish`, `event.confirm`, `report.moderate`, `source.configure`) evaluated against jurisdiction scope by a single decision function. No role-name checks in handlers.
- Postgres RLS on `civic_node_id` for all node-scoped tables (I14).
- `saved_routes` in a separate schema with no grant to the public-API role (I17).
- Community reporters are pseudonymous; public display uses coarse location cells only.
- Uploaded media: EXIF stripped, content-type verified, size-capped, stored in a *separate erasable bucket* — never in the object-locked raw-document bucket (see I7's stated tension).
- Node signing keys registered at PMVP; envelope signing implemented but federation deferred to V1.
- Rate limiting on the public API, report submission, and notification sends.

Full detail in [security-and-federation.md](security-and-federation.md) and [threat-model.md](threat-model.md).

---

## 11. Observability

`prometheus_client` for metrics, OpenTelemetry for traces, `structlog` for logs — all three standard libraries, none hand-rolled. Cerevra's bespoke registry and tracer are explicitly not adopted; its *metric taxonomy* and its `what`/`why`/`outcome`/`safe` log convention are.

One trace spans discovery → command → fetch → object → digest → version → parse → evidence → projection → impact → notification decision → delivery.

The `safe` log field carries real weight: it distinguishes "we correctly declined to publish" from "we failed to publish", and those must never look alike in a dashboard.

---

## 12. Local development

```
docker compose up   # postgres+postgis, minio, redis
make seed           # fixture sources, fixture documents, fixture geometry
make test           # unit + property tiers: no Docker, no network
```

Unit and property tiers must run with **no containers and no network**, because contributors will not spin up three services to check a parser fixture. Integration, e2e and chaos tiers use compose.

Full replay in 2.5 seconds means projection rebuild runs inside a request cycle during development. That is a genuine productivity asset and is worth preserving as long as the workload allows.

---

## 13. Deployment

Single container image, two process types (`api`, `worker`), one managed Postgres with PostGIS, one bucket, one small Redis. Managed backups with PITR. Migrations are forward-only Alembic, run as a pre-deploy step.

Estimated cost: **~$70/month** ([workload-model.md](scaling/workload-model.md)).

---

## 14. What is deliberately not built

Not in the first three months, with reasons:

| Not built | Why |
| --- | --- |
| Kafka / Pulsar | No independent consumer needs replay from an offset. A table serves 0.02 commands/s and gives transactional acknowledgement. |
| Kubernetes | Two process types on one host. |
| OpenSearch | Postgres FTS is untested at this scale and therefore cannot yet be shown insufficient. |
| ClickHouse | No analytical workload exists. |
| Vector search | Would not determine truth anyway; adds a model dependency for no PMVP gain. |
| Microservices | Three engineers must operate this during a flood. |
| H3 / S2 | GiST is sub-millisecond at 5,000 events. |
| Read replicas | 9 reads/s. |
| Federation protocol | Needs ≥2 real nodes to be meaningful; keys and signing land now, the protocol at V1. |
| Effective-permission cache | Ten publishers. Caching authorisation would create a revocation-latency window for no measurable gain — a bad trade when a compromised node must be cut off immediately. |
| SMS notifications | Cost model must be settled as product policy before the channel is built. |
| Native mobile apps | PWA first. |

Each has a measurable introduction trigger in [migration-triggers.md](scaling/migration-triggers.md).

---

## 15. Exit criteria

The PMVP is done when all of the following hold:

1. One real city, ≥3 real source adapters, ≥1 real civic publisher.
2. A citizen can see a real closure with inspectable evidence, correct freshness, and honest uncertainty.
3. Killing the worker at any of five pipeline points loses nothing and duplicates nothing (I2, I11).
4. Projection rebuild from scratch reproduces the identical projection hash (I12).
5. Replay divergence is detected and alerts (I13).
6. `FLUSHALL` mid-suite causes zero test failures (I10).
7. `OFFICIALLY_CONFIRMED` is unreachable without official evidence, proven by property test *and* rejected by the database (I3).
8. Every event page shows freshness and source health (I5); missing end times render "unknown" (I16).
9. A failing source is visible to citizens as `STALE`, and no event resolves through silence (I4, I15).
10. Contributors can run unit and property tests with no Docker and no network.
