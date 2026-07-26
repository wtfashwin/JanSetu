# Data Model

PostgreSQL 16 + PostGIS. All timestamps `timestamptz` in UTC. All surrogate keys **UUIDv7** (time-ordered, index-friendly, no coordination). SRID 4326 throughout; `geography` where distance semantics matter.

Classifications: `AUTHORITATIVE` · `APPEND_ONLY` · `DERIVED` · `CACHE` · `SEARCH_INDEX` · `ANALYTICAL` · `ARCHIVE`

PII classes: `NONE` · `PSEUDONYMOUS` · `PERSONAL` · `SENSITIVE`

---

## 1. Identity strategy

**Rule: no mutable name, URL, or human label is ever an identifier.**

| Entity | Identifier | Survives |
| --- | --- | --- |
| Organisation | UUIDv7 | Rename, merger |
| Civic node | UUIDv7 + DNS-style `node_slug` (immutable once issued) | Rebranding, key rotation |
| Source | UUIDv7 | URL change (URL is an attribute, and URL history is retained) |
| Source document | UUIDv7, keyed on `(source_id, stable_locator)` | Republication at a new URL — `stable_locator` is a notice number or a normalised path, never the raw URL |
| Document version | UUIDv7 + `content_sha256` (full width) | Re-fetch, re-parse |
| Evidence item | UUIDv7, unique on `(document_version_id, parser_version, extraction_ordinal)` | Parser upgrade — new parser yields new rows, old rows untouched |
| Public event | UUIDv7 | Merge (old id redirects, never dies), split (old id lists children), geometry correction, more evidence |
| Location / road segment | UUIDv7 + external reference (e.g. OSM way id) as an *attribute* | Upstream reference-data changes |
| Community report | UUIDv7 | Reporter erasure — the report survives pseudonymously |
| Saved route | UUIDv7, user-scoped | Never public, never referenced by an event |
| Notification decision | UUIDv7, unique on `(subscription_id, event_id, transition_sequence)` | Retry |

### Behaviour under change

| Change | Behaviour |
| --- | --- |
| Source URL changes | New URL recorded on the source; `stable_locator` keeps document identity; no new document is created |
| Document republished elsewhere | Content digest matches → cross-reference recorded. **Not auto-merged** — the same notice issued by two departments is corroboration, not duplication |
| Event gains evidence | New `event_evidence_links` row; projection re-runs; a transition may or may not result |
| Two events merged | `event_merges` row; loser `ARCHIVED` with `superseded_by`; **both id's resolve forever**; all evidence links retained (I18) |
| One event split | `event_splits` row; parent `ARCHIVED` with child pointers; evidence links partitioned exactly once across children (I19) |
| Geometry corrected | New `event_locations` row, `valid_from` set; prior row retained with `valid_to`. Geometry is bitemporal |
| Node republishes another's event | A federated reference, never a copy. Original signature preserved (I20) |
| Parser version changes | New evidence rows; historic rows immutable (I8) |
| Historic evidence replayed | Idempotent — unique keys absorb it (I1) |

---

## 2. Control plane

### `organisations` — `AUTHORITATIVE` · PII `NONE`
`id` PK · `legal_name` · `org_type` · `verified_at` · `created_at` · `created_by_principal`
Unique: `(legal_name, org_type)`. Retention: indefinite.

### `jurisdictions` — `AUTHORITATIVE` · PII `NONE`
`id` PK · `parent_id` FK→self · `path ltree NOT NULL` · `level` (`country|state|district|city|ward`) · `name` · `boundary geography(MultiPolygon,4326)`
Unique `path`. GiST on `path` and on `boundary`. **This replaces Cerevra's flat namespace token** (`cerevra/security/namespace.py:15`, `^[a-zA-Z0-9_-]+$` ≤64 chars, `/` rejected), which cannot express `india/maharashtra/pune/traffic-police`.
Retention: indefinite.

### `civic_nodes` — `AUTHORITATIVE` · PII `NONE`
`id` PK · `organisation_id` FK · `node_slug` UNIQUE **immutable** · `jurisdiction_id` FK · `authorised_categories text[]` · `trust_level` · `status` (`active|suspended|revoked`) · `protocol_version` · `created_at`
Index on `(jurisdiction_id, status)`. **RLS anchor for every node-scoped table.** Retention: indefinite (suspension is a state, never a delete — deleting a suspended node's history would itself be silent rewriting).

### `node_keys` — `APPEND_ONLY` · PII `NONE`
`id` PK · `civic_node_id` FK · `public_key bytea` · `algorithm` · `valid_from` · `valid_to` · `revoked_at` · `revocation_reason`
Unique `(civic_node_id, public_key)`. Partial index on `revoked_at IS NULL`. Private keys are **never** stored here — they live in the node operator's KMS/HSM. Overlapping validity supports rotation. Retention: indefinite (verifying historic signatures requires historic keys).

### `sources` — `AUTHORITATIVE` · PII `NONE`
`id` PK · `civic_node_id` FK · `adapter_type` · `current_url` · `url_history jsonb` · `poll_interval_seconds` · `categories text[]` · `authority_class` (`OFFICIAL|SEMI_OFFICIAL|COMMUNITY`) · `enabled`
Index `(civic_node_id, enabled)`. RLS on `civic_node_id`. Retention: indefinite.

### `source_adapters` — `AUTHORITATIVE` · PII `NONE`
`id` PK · `adapter_type` UNIQUE · `version` · `config_schema jsonb` · `parser_version`
Versioned; a change creates a new row (I8, and the versioned-configuration-authority pattern).

### `source_health` — `DERIVED` · PII `NONE`
`source_id` PK/FK · `last_success_at NOT NULL` · `last_failure_at` · `consecutive_failures NOT NULL` · `breaker_state NOT NULL` · `updated_at`
Drives the public `STALE` / `STATUS_UNKNOWN` presentation (I15). Rebuildable from `ingestion_attempts`.

---

## 3. Ingestion

### `ingestion_commands` — `AUTHORITATIVE` · PII `NONE`
`sequence bigint GENERATED ALWAYS AS IDENTITY` **— sole ordering authority** · `id` PK · `idempotency_key text NOT NULL UNIQUE` · `civic_node_id` · `source_id` · `command_type` · `payload jsonb` · `state` · `attempts int NOT NULL DEFAULT 0` · `claimed_by` · `claimed_at` · `lease_expires_at` · `created_at`

Indexes: `(state, created_at) WHERE state='PENDING'`; `(lease_expires_at) WHERE state='CLAIMED'`.
Partition: monthly by `created_at` from city scale.
Retention: 90 days after terminal state, then archive.

Two audit findings are designed out here: a single identity-column sequence source (Cerevra had two disagreeing ones — `handler.py:350-355` vs `writer.py:271-280`), and the unique idempotency key (Cerevra's dedup was a read-then-write race on a truncated hash).

### `ingestion_attempts` — `APPEND_ONLY` · PII `NONE`
`id` PK · `command_id` FK · `attempt_number` · `started_at` · `finished_at` · `outcome` · `http_status` · `error_class` · `error_detail`
Unique `(command_id, attempt_number)`. **Attempts are rows, never a mutated counter** — the direct lesson of `cerevra/wal/wal.py:216-255`, which recorded acknowledgement by rewriting log files in truncating mode.
Retention: 90 days.

### `source_documents` — `AUTHORITATIVE` · PII `NONE`
`id` PK · `source_id` FK · `stable_locator text NOT NULL` · `title` · `first_seen_at` · `last_seen_at`
Unique `(source_id, stable_locator)`. Retention: indefinite.

### `source_document_versions` — `APPEND_ONLY` · PII `NONE`
`id` PK · `source_document_id` FK · `content_sha256 char(64) NOT NULL` **immutable** · `object_key text NOT NULL` **immutable** · `byte_size` · `content_type` · `fetched_at` · `fetched_from_url` · `previous_version_id` FK→self

Unique `(source_document_id, content_sha256)` → identical bytes never create a second version (I1).
`previous_version_id` gives the revision chain that a content hash alone cannot express.
Immutability by `BEFORE UPDATE` trigger (I7).
Partition: monthly. Retention: hot 90 d → warm 1 y → cold archive, indefinite.

### `parser_executions` — `APPEND_ONLY` · PII `NONE`
`id` PK · `document_version_id` FK · `parser_version NOT NULL` · `normalisation_version` · `geocoder_version` · `llm_model_id` · `llm_prompt_version` · `started_at` · `finished_at` · `outcome` · `error_detail`
Unique `(document_version_id, parser_version)`.
The `llm_*` columns are what let replay be deterministic while still using an LLM: **stored outputs are reused, the model is never re-invoked during replay** (I12 vs invariant 7).

---

## 4. Evidence and events

### `evidence_items` — `APPEND_ONLY` · PII `NONE`
`id` PK · `sequence bigint IDENTITY` · `parser_execution_id` FK · `document_version_id` FK · `parser_version` · `extraction_ordinal int` · `civic_node_id` · `authority_class` · `category` · `claimed_start_at` · `claimed_end_at NULL` · `location_id` FK NULL · `raw_geometry geography NULL` · `extracted jsonb` · `confidence NULL` · `observed_at` · `known_at`

Unique `(document_version_id, parser_version, extraction_ordinal)` — re-parse is idempotent (I1, I8).
Indexes: `(category, observed_at)`; GiST on `raw_geometry`; `(civic_node_id, known_at)`.
Partition: monthly on `known_at`.
**Bitemporal:** `observed_at` is when the world changed; `known_at` is when JanSetu learned. Conflating them is exactly how a system manufactures false certainty (invariant 1).
`claimed_end_at` is nullable **with no default** — missing end times render "unknown" (I16).
Retention: hot 90 d → warm 1 y → cold indefinite.

### `public_events` — `DERIVED` · PII `NONE`
`id` PK · `jurisdiction_id` FK · `owning_node_id` FK NULL · `category` · `state NOT NULL` · `title` · `summary` · `claimed_start_at` · `claimed_end_at NULL` · `official_evidence_count int NOT NULL DEFAULT 0` · `community_evidence_count int NOT NULL DEFAULT 0` · `last_evidence_at NOT NULL` · `superseded_by` FK→self NULL · `projection_hash` · `ruleset_version` · `updated_at`

```sql
CHECK (state <> 'OFFICIALLY_CONFIRMED' OR official_evidence_count >= 1)   -- I3
```
`last_evidence_at NOT NULL` → freshness is always available (I5).
**Written only by the projection worker.** No other role has `UPDATE`.
Indexes: `(state, last_evidence_at)`; `(jurisdiction_id, category, state)`; GIN on the FTS `tsvector`.
Rebuildable in full from `evidence_items` + `event_transitions` + ruleset (invariant 15).

### `event_transitions` — `APPEND_ONLY` · PII `NONE`
`id` PK · `sequence bigint IDENTITY` · `event_id` FK · `from_state NULL` · `to_state NOT NULL` · `cause` · `triggering_evidence_id` FK NULL · `resolution_evidence_id` FK NULL · `ruleset_version NOT NULL` · `actor_principal NULL` · `evaluated_at NOT NULL` · `created_at`

```sql
CHECK (to_state <> 'RESOLVED' OR resolution_evidence_id IS NOT NULL)      -- I4
```
`REVOKE UPDATE, DELETE` + raising trigger (I6).
`evaluated_at` is passed in explicitly so the projector never calls `now()` (I12).
`ruleset_version` is what makes "why is this OFFICIALLY_CONFIRMED?" reproducible.
Retention: indefinite. This is the audit record.

### `event_evidence_links` — `APPEND_ONLY` · PII `NONE`
`event_id` FK · `evidence_id` FK · `link_type` (`supports|contradicts|resolves|context`) · `linked_at`
PK `(event_id, evidence_id, link_type)`. Preserved across merge and split (I18, I19).

### `event_locations` — `APPEND_ONLY` (bitemporal) · PII `NONE`
`id` PK · `event_id` FK · `geometry geography NOT NULL` · `geometry_kind` (`point|segment|polygon|route`) · `precision_class` · `valid_from` · `valid_to NULL` · `corrected_from_id` FK→self NULL
GiST on `geometry`. Corrections append; they never overwrite.

### `event_merges` / `event_splits` — `APPEND_ONLY` · PII `NONE`
Merges: `source_event_id` · `target_event_id` · `merged_at` · `actor_principal` · `reason`.
Splits: `parent_event_id` · `child_event_id` · `evidence_partition jsonb` · `split_at` · `actor_principal` · `reason`.
Both ids resolve forever. Property-tested for link conservation.

### `federated_events` — `APPEND_ONLY` · PII `NONE`
`id` PK · `origin_node_id` FK · `origin_event_id` · `envelope_bytes bytea NOT NULL` **immutable** · `signature bytea` · `key_id` FK · `verified_at` · `verification_status` · `received_at`
Unique `(origin_node_id, origin_event_id, envelope_hash)`. **No local `UPDATE` grant** (I20). Local commentary goes in a separate annotation table.

---

## 5. Geospatial reference

### `locations` — `AUTHORITATIVE` · PII `NONE`
`id` PK · `jurisdiction_id` FK · `name` · `location_type` · `geometry geography` · `external_ref jsonb` · `aliases text[]`
GiST on `geometry`; GIN on `aliases` (Indian place names have many transliterations, and this is load-bearing for matching).

### `road_segments` — `AUTHORITATIVE` · PII `NONE`
`id` PK · `jurisdiction_id` · `osm_way_id NULL` · `name` · `geometry geography(LineString)` · `road_class` · `geometry_simplified_z10/z14 geometry`
GiST on all geometry columns. Pre-simplified columns exist for map rendering at city scale.

### `transit_objects` — `AUTHORITATIVE` · PII `NONE`
`id` PK · `jurisdiction_id` · `object_type` (`route|stop|station|line`) · `operator_ref` · `geometry geography` · `gtfs_ref NULL`

---

## 6. Citizen surface

### `saved_routes` — `AUTHORITATIVE` · **PII `PERSONAL`**
`id` PK · `user_id` FK · `label` · `geometry geography(LineString)` **encrypted at rest** · `buffer_metres` · `active_windows jsonb` · `created_at`
**Separate schema. No grant to the public-API role. RLS on `user_id`** (I17).
Retention: until user deletion; hard-deleted on erasure request.
This is the most sensitive data in the system — a saved commute is a movement pattern.

### `subscriptions` — `AUTHORITATIVE` · PII `PERSONAL`
`id` PK · `user_id` FK · `subscription_type` (`route|area|category|event`) · `target_ref` · `channels text[]` · `quiet_hours` · `severity_threshold`
RLS on `user_id`.

### `route_impact_assessments` — `DERIVED` · **PII `PERSONAL`**
`id` PK · `saved_route_id` FK · `event_id` FK · `transition_sequence` · `impact_class` (`blocked|delayed|nearby|clear`) · `intersection_geometry` · `computed_at`
Unique `(saved_route_id, event_id, transition_sequence)`. Same isolation as `saved_routes`. Retention: 30 days.

### `community_reports` — `AUTHORITATIVE` · **PII `PSEUDONYMOUS`**
`id` PK · `reporter_pseudonym_id` FK · `category` · `coarse_cell` (H3 res ~8, ≈0.7 km²) · `precise_geometry geography` **restricted** · `reported_at` · `description` · `media_refs jsonb` · `status` · `dedup_key`
Unique `dedup_key = hash(reporter_pseudonym_id, category, coarse_cell, time_bucket)`.
**Public display uses `coarse_cell` only.** Precise geometry is moderator-visible and never leaves the system.
Reporter identity lives in a separate mapping table that is erasable independently, so an erasure request removes the person without destroying the civic evidence (resolving the tension left open in Cerevra's compliance modules).

### `verification_events` — `APPEND_ONLY` · PII `PSEUDONYMOUS`
`id` PK · `event_id` FK · `reporter_pseudonym_id` FK · `verdict` (`confirms|denies|unsure`) · `submitted_at`
Unique `(event_id, reporter_pseudonym_id)` — one verdict per person per event.

### `moderation_actions` — `APPEND_ONLY` · PII `PERSONAL` (moderator identity)
`id` PK · `target_type` · `target_id` · `action` · `reason` · `moderator_principal` · `acted_at`
Retention: 3 years.

---

## 7. Notifications

### `notification_decisions` — `APPEND_ONLY` · PII `PERSONAL`
`id` PK · `subscription_id` FK · `event_id` FK · `transition_sequence` · `outcome` (`send|suppress_cap|suppress_stale|suppress_quiet|suppress_threshold`) · `evidence_age_seconds` · `severity` · `decided_at`

```sql
CHECK (outcome <> 'send' OR evidence_age_seconds <= category_freshness_budget)   -- I23
```
Suppressions are recorded with a reason: "we chose not to alert you" must be auditable, not invisible.
Retention: 90 days.

### `notification_deliveries` — `AUTHORITATIVE` · PII `PERSONAL`
`id` PK · `decision_id` FK · `subscription_id` · `event_id` · `transition_sequence` · `channel` · `attempts int` · `status` · `provider_ref` · `first_attempt_at` · `delivered_at`
**Unique `(subscription_id, event_id, transition_sequence)`** → retries reuse the row; duplicates are impossible (I9).
Partition: monthly. Retention: 90 days.

---

## 8. Cross-cutting

### `audit_entries` — `APPEND_ONLY` · PII `PERSONAL`
`id` PK · `sequence bigint IDENTITY` · `actor_principal` · `action` · `target_type` · `target_id` · `before jsonb NULL` · `after jsonb NULL` · `request_id` · `occurred_at`
`REVOKE UPDATE, DELETE`. Partition monthly. Retention: 7 years, cold after 1 year.

### `projection_checkpoints` — `DERIVED` · PII `NONE`
`projection_name` · `partition_key` · `last_applied_sequence` · `projection_hash NOT NULL` · `ruleset_version` · `computed_at`
PK `(projection_name, partition_key)`.
**This table is what makes replay divergence detectable** (I12, I13) — the single most important structure Cerevra lacks entirely.

### `rulesets` — `AUTHORITATIVE` (versioned) · PII `NONE`
`version` PK · `rules jsonb NOT NULL` **immutable** · `activated_at` · `deactivated_at NULL` · `authored_by`
A rule change is a new version, never an edit. Every transition records the version that produced it.

### `search_documents` — `SEARCH_INDEX` · PII `NONE`
Postgres `tsvector` at PMVP/city; OpenSearch at national. Fully rebuildable; nothing exists only here (I21).

### `event_analytics` — `ANALYTICAL` · PII `NONE`
ClickHouse at national scale only. **No operational grant** (I22).

---

## 9. Classification summary

| Class | Tables |
| --- | --- |
| `AUTHORITATIVE` | organisations, jurisdictions, civic_nodes, sources, source_adapters, source_documents, ingestion_commands, locations, road_segments, transit_objects, saved_routes, subscriptions, community_reports, notification_deliveries, rulesets |
| `APPEND_ONLY` | node_keys, ingestion_attempts, source_document_versions, parser_executions, evidence_items, event_transitions, event_evidence_links, event_locations, event_merges, event_splits, federated_events, verification_events, moderation_actions, notification_decisions, audit_entries |
| `DERIVED` | public_events, source_health, route_impact_assessments, projection_checkpoints |
| `CACHE` | Redis only — never a table |
| `SEARCH_INDEX` | search_documents |
| `ANALYTICAL` | event_analytics |
| `ARCHIVE` | detached partitions + Parquet exports in cold object storage |

## 10. Encryption and PII

| PII class | Tables | Treatment |
| --- | --- | --- |
| `NONE` | Most civic data | TLS in transit; volume encryption at rest |
| `PSEUDONYMOUS` | community_reports, verification_events | Reporter mapping in a separately-erasable table |
| `PERSONAL` | saved_routes, subscriptions, route_impact_assessments, notification_* , audit_entries | Column-level encryption on route geometry; RLS on `user_id`; separate schema; no public-API grant |
| `SENSITIVE` | — | None stored. **JanSetu deliberately stores no health, caste, religion, or political-affiliation data, and no precise individual location history.** This is a product boundary, not an oversight |

**Erasure.** A user's routes, subscriptions, and reporter mapping are hard-deleted. Their community reports survive **pseudonymously** — the civic evidence remains, the person does not. This is the explicit resolution of the immutability-versus-erasure tension that Cerevra's compliance modules leave unaddressed (`cerevra/compliance/deletion.py`).
