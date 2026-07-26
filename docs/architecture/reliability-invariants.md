# Reliability Invariants

Twenty-four invariants. Each has a mechanism, a database constraint where one is possible, tests at two levels, a metric, an alert, and a recovery procedure.

An invariant with no enforcing mechanism is a wish. Where enforcement is only partial, this document says so rather than implying coverage it does not have.

Naming: tables are as defined in [data-model.md](data-model.md). Metrics are as defined in [observability-and-slos.md](observability-and-slos.md).

---

## I1 — The same source-document version cannot create duplicate effects

| | |
| --- | --- |
| **Mechanism** | Content-addressed document identity plus a unique evidence key. A re-parse of the same `(document_version, parser_version)` produces rows that collide with the existing ones and are discarded by `ON CONFLICT DO NOTHING`. |
| **DB constraint** | `source_document_versions UNIQUE (source_document_id, content_sha256)`; `evidence_items UNIQUE (document_version_id, parser_version, extraction_ordinal)` |
| **Unit test** | Fingerprint of identical bytes is stable and full-width; extraction ordinals are deterministic for a fixed parser and input. |
| **Integration test** | Ingest the same document twice → one version row, one evidence set, one event. Re-run the parser at the same version → zero new rows. |
| **Metric** | `source_document_duplicates_total`, `evidence_items_total` |
| **Alert** | `evidence_items_total` grows while `source_document_versions_total` is flat for >15 min → duplicate amplification. |
| **Recovery** | Identify the offending parser version; delete evidence for that `(version, parser)` pair; re-run; rebuild affected projections from checkpoint. |

Directly addresses the audit finding that Cerevra used a 64-bit truncated hash as identity, dedup key and primary key simultaneously (`cerevra/ingestion/handler.py:148-157`).

---

## I2 — A retried ingestion command is safe

| | |
| --- | --- |
| **Mechanism** | Every command carries an idempotency key derived from its cause. Effects and the command's terminal state commit in one transaction. Claiming uses a lease. |
| **DB constraint** | `ingestion_commands UNIQUE (idempotency_key)`; `state` transitions guarded by a `CHECK` and by `WHERE state = 'CLAIMED' AND claimed_by = $me` on the completing update. |
| **Unit test** | Idempotency-key generation is pure and stable across processes. |
| **Integration test** | Submit N identical commands concurrently → one row, one effect. Expire a lease mid-flight → re-claim → still one effect. |
| **Metric** | `ingestion_commands_total{state}`, `ingestion_retries_total` |
| **Alert** | `ingestion_retries_total / ingestion_commands_total > 0.2` over 30 min. |
| **Recovery** | Reset stuck `CLAIMED` rows whose lease expired; they re-run safely by construction. |

---

## I3 — A community report cannot produce official confirmation

| | |
| --- | --- |
| **Mechanism** | `OFFICIALLY_CONFIRMED` is reachable only from a rule branch requiring at least one `evidence_items` row whose provenance class is `OFFICIAL` and whose issuing node holds `event.confirm` for that jurisdiction and category. |
| **DB constraint** | `public_events CHECK (state <> 'OFFICIALLY_CONFIRMED' OR official_evidence_count >= 1)`, where `official_evidence_count` is maintained only by the projection worker. |
| **Unit test** | Exhaustive enumeration of rule inputs: no input set lacking official evidence yields `OFFICIALLY_CONFIRMED`. |
| **Integration test** | 10,000 community reports on one event → state remains `COMMUNITY_CORROBORATED`. |
| **Metric** | `events_by_state`, `federation_signature_failures_total` |
| **Alert** | Any transition into `OFFICIALLY_CONFIRMED` where `official_evidence_count = 0` → page immediately. Should be structurally impossible; if it fires, the constraint has been bypassed. |
| **Recovery** | Halt the projector, quarantine the ruleset version, rebuild affected events under the prior ruleset, publish a correction. |

This is the invariant most load-bearing on public trust. It is enforced in three independent places — rules, database constraint, and alert — on purpose.

---

## I4 — An event cannot resolve due only to silence

| | |
| --- | --- |
| **Mechanism** | `RESOLVED` requires positive evidence of resolution (an official notice, or corroborated reports of normality). Absence of updates drives freshness decay toward `STATUS_UNKNOWN`, never toward `RESOLVED`. |
| **DB constraint** | `event_transitions CHECK (to_state <> 'RESOLVED' OR resolution_evidence_id IS NOT NULL)` |
| **Unit test** | Rule table: `(no new evidence for N hours)` maps to `STATUS_UNKNOWN`, and `RESOLVED` is absent from that branch's outputs. |
| **Integration test** | Create an event; advance time 7 days with no evidence; assert `STATUS_UNKNOWN` and `resolved_at IS NULL`. |
| **Metric** | `stale_events_total`, `events_by_state{state="STATUS_UNKNOWN"}` |
| **Alert** | `stale_events_total` > 5% of active events for 24 h → source-coverage problem. |
| **Recovery** | None needed for the invariant itself; investigate source health. |

---

## I5 — Every public event exposes freshness

| | |
| --- | --- |
| **Mechanism** | Every event API response carries `last_evidence_at`, `evidence_age_seconds`, `source_health`, and `as_of`. The UI has no code path that renders an event without them. |
| **DB constraint** | `public_events` has `last_evidence_at timestamptz NOT NULL`. |
| **Unit test** | Response serialiser omits no freshness field; snapshot test on the schema. |
| **Integration test** | Contract test asserting every event payload validates against a schema in which the freshness fields are `required`. |
| **Metric** | `source_health_age_seconds` |
| **Alert** | Any 2xx event response failing schema validation in the contract-test canary. |
| **Recovery** | Roll back the serialiser. |

---

## I6 — Every state transition is append-only

| | |
| --- | --- |
| **Mechanism** | `event_transitions` has no `UPDATE`/`DELETE` grant for the application role. Corrections are new transitions. |
| **DB constraint** | `REVOKE UPDATE, DELETE ON event_transitions FROM app_role` plus a `BEFORE UPDATE OR DELETE` trigger that raises. |
| **Unit test** | Projector emits transitions; never mutates. |
| **Integration test** | Attempt `UPDATE`/`DELETE` as the app role → both raise. |
| **Metric** | `event_corrections_total` |
| **Alert** | Any trigger firing → page (indicates a code path attempting mutation). |
| **Recovery** | The trigger prevented the damage; fix the caller. |

Directly addresses `cerevra/wal/wal.py:216-255`, where acknowledgement state was recorded by rewriting log files in truncating mode.

---

## I7 — Raw documents remain immutable

| | |
| --- | --- |
| **Mechanism** | Object keys are content-addressed (`{node}/{source}/{sha256}`). Bucket versioning plus object-lock in governance mode. No overwrite path exists in application code. |
| **DB constraint** | `source_document_versions.content_sha256` and `object_key` immutable via trigger. |
| **Unit test** | Key derivation is a pure function of content. |
| **Integration test** | Store, then attempt overwrite → denied by bucket policy. Verify stored digest matches recomputed digest on read. |
| **Metric** | `source_document_versions_total`; object-integrity check failures |
| **Alert** | Any digest mismatch on read → page. |
| **Recovery** | Restore the object version from bucket versioning; audit for tampering. |

**Known tension, stated not hidden:** this conflicts with erasure requests when a raw document contains personal data. Resolution: personal data is never stored in raw civic documents by design (they are public notices); community-report media is stored in a *separate*, erasable bucket that is explicitly not covered by object-lock. See [security-and-federation.md](security-and-federation.md).

---

## I8 — Parser upgrades cannot silently rewrite historic evidence

| | |
| --- | --- |
| **Mechanism** | `parser_version` is part of the evidence unique key. A new parser produces *new* evidence rows alongside the old. Which version a projection uses is an explicit, versioned choice. |
| **DB constraint** | `evidence_items UNIQUE (document_version_id, parser_version, extraction_ordinal)`; `parser_version` immutable. |
| **Unit test** | Same document + two parser versions → two disjoint evidence sets. |
| **Integration test** | Upgrade parser, re-run, assert historic evidence rows are byte-identical and new rows are added. |
| **Metric** | `parser_failures_total`, `evidence_items_total{parser_version}` |
| **Alert** | Historic evidence row count for a superseded parser version decreasing → page. |
| **Recovery** | Restore from backup; the count should never decrease. |

---

## I9 — Notification retries are deduplicated

| | |
| --- | --- |
| **Mechanism** | A delivery is keyed on `(subscription_id, event_id, transition_sequence)`. Retries re-attempt the same row; they cannot create a second one. |
| **DB constraint** | `notification_deliveries UNIQUE (subscription_id, event_id, transition_sequence)` |
| **Unit test** | Key derivation stable. |
| **Integration test** | Force provider timeout then success → one delivery row, `attempts = 2`. |
| **Metric** | `notification_decisions_total`, `notification_delivery_failures_total` |
| **Alert** | Deliveries per user per hour exceeding a policy cap → possible notification storm. |
| **Recovery** | Suspend the affected subscription batch; the unique key already prevents duplicate sends. |

---

## I10 — Cache loss cannot remove authoritative state

| | |
| --- | --- |
| **Mechanism** | Redis holds only rate-limit counters, short-TTL caches, job coordination hints, and notification-dedup *hints*. Every one has a correct behaviour when absent. Authoritative dedup is the DB unique constraint in I9, not Redis. |
| **DB constraint** | n/a — the invariant is the absence of authoritative data in Redis. |
| **Unit test** | Every Redis accessor has a defined miss path. |
| **Integration test** | **`FLUSHALL` mid-suite; the entire suite must still pass.** This is a standing test, not an occasional exercise. |
| **Metric** | Cache hit ratio; rate-limiter fallback counter |
| **Alert** | Redis unavailable > 5 min → warn (degraded, not down). |
| **Recovery** | Restart Redis. No data recovery required, by construction. |

---

## I11 — A worker crash cannot lose an acknowledged command

| | |
| --- | --- |
| **Mechanism** | Acknowledgement *is* the commit of the transaction containing the effects. There is no separate ack step that can be lost. Leases expire and unfinished work is re-claimed. |
| **DB constraint** | `ingestion_commands.state = 'SUCCEEDED'` written in the same transaction as derived rows. |
| **Unit test** | Command completion and effect writes share one session/transaction. |
| **Integration test** | `SIGKILL` the worker at each of five points in the pipeline; assert the command completes exactly once observably. |
| **Metric** | `wal_unacknowledged_total` (commands in `PENDING`+`CLAIMED`), `wal_lag` |
| **Alert** | `wal_unacknowledged_total` rising monotonically for 15 min → workers stuck or dead. |
| **Recovery** | Automatic on lease expiry. |

This invariant is the reason the ingestion log is a database table rather than a file. In Cerevra, the recovery engine reads a WAL stack the ingestion path never writes to (`cerevra/wal/recovery.py:17`), so an equivalent guarantee does not currently hold there.

---

## I12 — Identical replay inputs produce equivalent projections

| | |
| --- | --- |
| **Mechanism** | The projector is a pure fold over evidence ordered by `(sequence, evidence_id)`, parameterised by `(ruleset_version, parser_version, normalisation_version, geocoder_version, matching_version)`. No `now()`, no network calls, no map-iteration-order dependence. LLM outputs are read from storage, never regenerated. |
| **DB constraint** | `projection_checkpoints UNIQUE (projection_name, partition_key, last_applied_sequence)` storing `projection_hash`. |
| **Unit test** | Projector has no clock or IO dependency (enforced by a lint rule and by injecting a forbidden-clock stub). |
| **Property test** | For random valid evidence sequences, two independent folds produce identical projection hashes. |
| **Metric** | `replay_duration_seconds`, `replay_divergence_total` |
| **Alert** | `replay_divergence_total > 0` → page. |
| **Recovery** | Freeze the projector; bisect by ruleset/parser version; the append-only evidence is unaffected, so any prior projection is reproducible. |

---

## I13 — Replay divergence is detected

| | |
| --- | --- |
| **Mechanism** | A canonical hash over the sorted projection is computed on every rebuild and compared to the stored checkpoint hash. A nightly shadow rebuild of a sampled partition compares against live. |
| **DB constraint** | `projection_hash` stored per checkpoint. |
| **Unit test** | Hash is canonical: independent of row order, insertion order, and JSON key order. |
| **Integration test** | Deliberately perturb one rule and assert divergence is detected and attributed. |
| **Metric** | `replay_divergence_total` |
| **Alert** | Any non-zero value. |
| **Recovery** | See I12. |

Cerevra has no equivalent — it stores no projection hash, so a code change silently produces a different manifest. This invariant closes that gap and is non-negotiable, because JanSetu's entire audit claim rests on it.

---

## I14 — Node isolation prevents cross-node mutation

| | |
| --- | --- |
| **Mechanism** | Postgres RLS on `civic_node_id` bound to a request-scoped session principal, plus runtime assertions on read paths that raise on any cross-node row. Remote federated events live in a separate table with no local write grant. |
| **DB constraint** | RLS policies on every node-scoped table; `REVOKE` on the federated table. |
| **Unit test** | Authorisation decision function denies out-of-jurisdiction writes. |
| **Integration test** | Authenticate as node A, attempt to update node B's event → denied at the database, not only at the application. |
| **Metric** | Authorisation denials by reason |
| **Alert** | Any successful cross-node write detected by an audit query → page. |
| **Recovery** | Revoke the offending credential; replay the audit log to identify affected rows. |

Adopts `cerevra/query/invariants.py:39-77` (assertions that raise) and adds the database enforcement that assertions alone cannot provide.

---

## I15 — Source failures remain visible

| | |
| --- | --- |
| **Mechanism** | Circuit-breaker and fetch outcomes write `source_health`. Health propagates to every event whose latest evidence came from that source, and is rendered to citizens as `STALE` / `STATUS_UNKNOWN`. |
| **DB constraint** | `source_health` has `last_success_at`, `consecutive_failures`, `breaker_state`, all `NOT NULL`. |
| **Unit test** | Health→presentation mapping is total (no unmapped combination). |
| **Integration test** | Source returns 503 for 1 h → events show `STALE`; no event silently resolves. |
| **Metric** | `source_health_age_seconds`, `source_fetch_duration_seconds` |
| **Alert** | Any source with `last_success_at` older than 3× its poll interval. |
| **Recovery** | Fix or disable the adapter; health clears on next success. |

---

## I16 — Unknown remains explicit

| | |
| --- | --- |
| **Mechanism** | Nullable domain fields (end time, exact extent, cause) render as "unknown". No default, no estimate, no interpolation anywhere in the serialiser or UI. |
| **DB constraint** | Nullable columns with **no** `DEFAULT`. |
| **Unit test** | Serialiser emits `null` and a `known: false` marker, never a substituted value. |
| **Integration test** | Event with no end time → API returns `expected_end: null`; UI snapshot shows "unknown". |
| **Metric** | Fraction of active events with unknown end time |
| **Alert** | A sudden drop in that fraction → something began fabricating values. |
| **Recovery** | Revert. |

---

## I17 — Private saved routes never become public evidence

| | |
| --- | --- |
| **Mechanism** | `saved_routes` and `route_impact_assessments` are in a separate schema with no grant to the read-API role. Impact is computed per user and returned only to that user. Aggregate statistics, if ever published, are k-anonymised with a documented minimum k. |
| **DB constraint** | Schema-level `REVOKE` from the public-API role; RLS on `user_id`. |
| **Unit test** | No serialiser for a public response references route tables. |
| **Integration test** | Static check plus a runtime test: no public endpoint can return a route row even with a crafted parameter. |
| **Metric** | Route-table access by role |
| **Alert** | Any access to route tables by the public-API role → page. |
| **Recovery** | Revoke, rotate, audit, and disclose. |

---

## I18 — Event merging preserves both histories

| | |
| --- | --- |
| **Mechanism** | Merge creates an `event_merges` row and a surviving event; the merged event is `ARCHIVED` with a pointer, never deleted. All `event_evidence_links` from both are retained and re-pointed additively. |
| **DB constraint** | `event_merges (source_event_id, target_event_id, merged_at, actor, reason)`, `APPEND_ONLY`; FK from archived event to its successor. |
| **Property test** | For any merge, the union of evidence links before equals the evidence links after; no link is lost. |
| **Integration test** | Merge two events; both original ids still resolve (to a redirect); both histories readable. |
| **Metric** | Merges per day |
| **Alert** | Evidence-link count decreasing across a merge → page. |
| **Recovery** | Un-merge is a new forward operation, not a rollback. |

---

## I19 — Event splitting remains auditable

| | |
| --- | --- |
| **Mechanism** | Symmetric to I18 via `event_splits`. The original is archived with pointers to all children; every evidence link is assigned to exactly one child, and the assignment is recorded. |
| **DB constraint** | `event_splits`, `APPEND_ONLY`; a check that the child evidence partition covers the parent's links exactly once. |
| **Property test** | Partition property: no evidence link is dropped or duplicated across children. |
| **Integration test** | Split; assert the original id resolves and lists its children. |
| **Metric** | Splits per day |
| **Alert** | Partition-property violation → page. |
| **Recovery** | Forward correction. |

---

## I20 — Signed remote events cannot be silently altered

| | |
| --- | --- |
| **Mechanism** | Federated events are stored as received, with the original signed envelope bytes retained verbatim. Signature verification runs on ingest and again on read. Local commentary lives in a separate annotation table that can never modify the envelope. |
| **DB constraint** | `federated_events.envelope_bytes` immutable via trigger; no `UPDATE` grant. |
| **Unit test** | Verification rejects any single-bit modification. |
| **Integration test** | Ingest a signed event; attempt local edit → denied; tamper with stored bytes → read-time verification fails and the event is quarantined. |
| **Metric** | `federation_signature_failures_total` |
| **Alert** | Any verification failure → page. |
| **Recovery** | Quarantine, re-fetch from the origin node, notify the origin operator. |

---

## I21 — Search indexes can be rebuilt

| | |
| --- | --- |
| **Mechanism** | All search indexes are `DERIVED` from authoritative tables, with a documented, tested rebuild job. No field exists only in the index. |
| **DB constraint** | n/a. |
| **Unit test** | Index document builder is a pure function of authoritative rows. |
| **Integration test** | Drop the index entirely; rebuild; assert query results are identical to pre-drop. |
| **Metric** | Index rebuild duration; index/authoritative row-count delta |
| **Alert** | Row-count delta beyond tolerance for 1 h → drift. |
| **Recovery** | Rebuild. |

---

## I22 — Analytics storage cannot become operational truth

| | |
| --- | --- |
| **Mechanism** | The analytical store is written by ETL only and read only by dashboards and researchers. The operational application has no client for it. Enforced by an import-linter rule and by network policy at city scale and beyond. |
| **DB constraint** | Separate credentials; the app role has no grant. |
| **Unit test** | Import-linter forbids the analytics client in application modules. |
| **Integration test** | Assert no operational endpoint's dependency graph reaches the analytics client. |
| **Metric** | Analytics client connections by source service |
| **Alert** | Any operational service connecting → page. |
| **Recovery** | Revoke and refactor. |

---

## I23 — Stale evidence cannot trigger indefinite alerts

| | |
| --- | --- |
| **Mechanism** | Notification decisions require evidence freshness within a per-category budget. An event whose evidence has aged past that budget transitions to `STATUS_UNKNOWN`, and `STATUS_UNKNOWN` does not generate new alerts. Per-user, per-event, per-day delivery caps apply regardless. |
| **DB constraint** | `notification_decisions CHECK (evidence_age_seconds <= category_freshness_budget)` |
| **Unit test** | Decision function returns "suppress" past the budget. |
| **Integration test** | Age an event past budget; assert no further notifications and a state change to `STATUS_UNKNOWN`. |
| **Metric** | `notification_decisions_total{outcome="suppressed_stale"}` |
| **Alert** | Suppressed-stale share exceeding 20% → source coverage is failing. |
| **Recovery** | Fix source health. |

---

## I24 — Disaster traffic cannot bypass trust rules

| | |
| --- | --- |
| **Mechanism** | Disaster mode is an explicit, audited operational switch. It may raise report-intake rate limits, shed non-essential read features, and reprioritise notification queues. It may **not** lower confirmation thresholds, auto-confirm, skip moderation, or relax signature verification. The trust ruleset is a versioned artifact with no disaster variant — there is nothing to loosen. |
| **DB constraint** | Ruleset selection is by version id only; no runtime override column exists. |
| **Unit test** | No code path passes a modified threshold set into the decision function. |
| **Integration test** | Enable disaster mode under a 100× report flood; assert confirmation still requires official evidence and moderation queues still gate ambiguous events. |
| **Metric** | `events_by_state` during disaster mode; moderation queue age |
| **Alert** | Any `OFFICIALLY_CONFIRMED` transition during disaster mode lacking official evidence → page (should be impossible per I3). |
| **Recovery** | Exit disaster mode; audit every transition made during it. |

This is the invariant most likely to be argued away under pressure ("people need to know *now*"). It is recorded here precisely so that relaxing it requires an explicit, reviewed decision rather than an on-call judgement call at 03:00.

---

## Coverage summary

| Enforcement | Invariants |
| --- | --- |
| Database constraint or trigger | I1, I2, I3, I4, I5, I6, I7, I8, I9, I11, I12, I14, I17, I18, I19, I20, I22, I23 |
| Policy / architecture only (weaker) | I10, I13, I15, I16, I21, I24 |
| Property-based test | I12, I18, I19 |
| Chaos test | I2, I7, I10, I11, I15, I24 |

Six invariants (**I10, I13, I15, I16, I21, I24**) cannot be enforced by a constraint because they are properties of behaviour rather than of data. They are the ones most likely to erode silently, and each therefore carries a *continuous* check — a standing `FLUSHALL` test, a nightly shadow rebuild, a contract-test canary, an import-linter rule — rather than a one-off test. Where a guarantee is weaker, this document says so rather than implying uniform coverage.
