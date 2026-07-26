# JanSetu City-Scale Architecture

Target workload ([workload-model.md](workload-model.md), Scenario B): 26 evidence writes/s peak, **1,390 citizen reads/s peak**, 200,000-delivery notification bursts, 2 TB objects, 190 GB Postgres, full replay 42 minutes.

Two of those numbers force change. The rest do not.

---

## 1. What actually forces change

| Pressure | Number | Consequence |
| --- | --- | --- |
| **Read amplification** | 1,390 reads/s vs 26 writes/s | Read replica + cache + CDN |
| **Replay duration** | 42 min full rebuild | Checkpointed incremental rebuild becomes mandatory, not optional |
| **Notification bursts** | 200 K deliveries in minutes | Separate queue and worker so fan-out cannot starve ingestion or reads |
| Table size | 190 GB | Partitioning for *maintenance* (vacuum, retention deletes), not for query speed |
| Object storage | 2 TB | Lifecycle policy |

Everything else — write throughput, geospatial query cost, search — is still comfortably inside what the PMVP shape handles. **The monolith is not split.** It is deployed as more process types from the same image.

---

## 2. Shape

```
                         CDN (public event JSON, assets)
                              │
                    ┌─────────┴─────────┐
                    │   Load balancer   │
                    └─────────┬─────────┘
                              ▼
                      api  (N replicas)
                              │
      ┌───────────┬───────────┼───────────┬──────────────┐
      ▼           ▼           ▼           ▼              ▼
 ingestion   projection  notification  moderation    scheduler
  worker       worker       worker      (in api)      (single)
      │           │           │
      └───────────┴─────┬─────┴──────────────┐
                        ▼                    ▼
            PostgreSQL primary  ──async──▶ read replica
            (+PostGIS, partitioned)       (citizen reads)
                        │
                        ▼
            S3 (lifecycle: Standard → IA → Glacier)
                        │
                     Redis (cache, rate limits, coordination)
```

Same image, five process types. Still one database cluster, still one bucket, still no broker.

---

## 3. Changes from PMVP, each with its trigger

### 3.1 Read replica

**Trigger.** Sustained read CPU on the primary > 60% for 1 hour, **and** measured replica lag stays inside the 30-second freshness budget.

Citizen reads go to the replica. All writes, and any read that must observe its own write (publisher console after publishing, moderation actions), go to the primary. Routing is explicit per endpoint, not a heuristic.

**Failure mode that matters.** Replica lag renders a just-published closure as "not found". Mitigation: every response carries `as_of`; the publisher console reads the primary; and if lag exceeds the freshness budget, the router fails back to the primary and raises an alert rather than serving stale civic data silently.

**Rollback.** Point reads back at the primary. One config change.

### 3.2 Cache-aside + CDN

**Trigger.** p95 read latency breaches the 200 ms SLO **and** profiling attributes > 40% of read load to repeated identical queries.

- Redis cache-aside for event lists and event detail, TTL 30 s, key includes filters and `ruleset_version`.
- CDN in front of public event JSON with a short TTL and **explicit purge on every event transition**.

**Failure mode that matters.** A cached "road open" surviving a real closure. Purge-on-transition is therefore a correctness requirement, not an optimisation, and the purge must be issued in the same worker that commits the transition, with failures retried.

### 3.3 Checkpointed incremental projection rebuild

**Trigger.** Full replay exceeding 10 minutes — reached at Scenario B (42 min).

`projection_checkpoints(projection_name, partition_key, last_applied_sequence, projection_hash, ruleset_version)`. Routine rebuilds start from the checkpoint and process only new evidence, completing in under 60 seconds. Full rebuilds remain possible and are exercised weekly against a sampled partition to keep the path warm.

This is the point where Cerevra's model — `rebuild_manifest` as an unbounded O(full history) fold with no checkpoint — stops being viable, and it is the reason checkpoints are designed in from the PMVP even though they are not needed there.

### 3.4 Separate notification worker and queue

**Trigger.** A single fan-out burst measurably delays ingestion or projection by more than the notification-decision SLO.

Notification decisions move to their own command type with their own worker pool and their own claim budget, so a 200 K burst cannot starve ingestion. Deliveries are ordered by **impact severity, not FIFO** — a road-closure alert to a route-affected user outranks a general area notice.

Per-user, per-event and per-day caps apply throughout (I9, I23).

### 3.5 Table partitioning

**Trigger.** `evidence_items` or `source_document_versions` exceeding ~50 GB, where retention deletes and autovacuum become measurably expensive — not where queries slow down.

Monthly range partitions on `created_at` for `evidence_items`, `source_document_versions`, `ingestion_commands`, `ingestion_attempts`, `notification_deliveries`, and `audit_entries`. Retention becomes `DETACH PARTITION` + archive instead of a bulk `DELETE`.

`public_events`, `civic_nodes`, `sources`, `locations` stay unpartitioned — they are small and heavily joined.

### 3.6 Object-storage lifecycle

**Trigger.** Bucket exceeding 1 TB.

Standard for 90 days → Infrequent Access to 1 year → Glacier-class thereafter. Object-lock retained throughout (I7). Restore-from-archive latency (hours) is documented and is acceptable because archived raw documents are needed for audit and replay, not for citizen reads.

### 3.7 Geospatial tuning

Still PostGIS, still no H3. Additions: pre-simplified geometry columns per zoom level for map rendering, a materialised `active_events_geom` view refreshed on transition, and `CLUSTER` on the GiST index during maintenance windows.

**H3/S2 trigger.** Route × event intersection p95 breaching 500 ms with a correctly-tuned GiST index and simplified geometry. Not before.

---

## 4. What is still not introduced

| Not introduced | Why not at Scenario B |
| --- | --- |
| Kafka / Pulsar | No consumer requires offset replay. The outbox table sustains 26 writes/s with transactional acknowledgement, which a broker would take away. |
| OpenSearch | Postgres FTS + GIN has not been shown insufficient on production-shaped data. |
| ClickHouse | Dashboards can run on the replica at this volume. |
| Sharding | 190 GB is one comfortable cluster. |
| Kubernetes | Five process types are manageable on a managed container platform. |
| Microservices | The modules have not yet demonstrated divergent scaling profiles across two releases. |
| Multi-region | One city. |
| Service mesh | Not applicable. |

Each retains a trigger in [migration-triggers.md](migration-triggers.md).

---

## 5. Operational envelope

- **RPO ≤ 5 s** — streaming replication plus WAL archiving.
- **RTO ≤ 15 min** — replica promotion, rehearsed quarterly.
- **Availability 99.5%.**
- **On-call: still 2–3 engineers.** The system has five process types, one database, one bucket, one cache. A flood at 02:00 is diagnosable from one dashboard and one log stream.

Preserving that last property is the constraint that rejects most of §4.

---

## 6. Disaster mode at city scale

An audited operational switch. It may:

- raise community-report intake limits and add report-worker capacity;
- shed non-essential read features (historical search, analytics endpoints);
- reprioritise notification queues by severity;
- extend cache TTLs on stable content to protect the database.

It may **not** lower confirmation thresholds, auto-confirm, skip moderation, or relax signature verification (I24). There is no disaster variant of the trust ruleset, so there is nothing to loosen under pressure.

---

## 7. Exit criteria (→ national scale)

Move to Architecture C when any holds:

1. A second region is required for data residency or failure isolation.
2. One Postgres cluster cannot meet capacity or its maintenance window exceeds the availability budget.
3. Three or more external consumers require push delivery with replay from an offset.
4. Independent civic nodes begin federating, requiring signed-envelope exchange at volume.
5. Partitioned replay is needed because single-worker replay exceeds the incident window even from a checkpoint.
