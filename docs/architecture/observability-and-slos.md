# Observability and SLOs

Stack: `prometheus_client` for metrics, OpenTelemetry for traces, `structlog` for logs, Grafana for dashboards. All three are standard libraries.

The Cerevra audit found a bespoke metrics registry (`cerevra/monitoring/metrics.py`, hand-rolled with its own `export_prometheus_format`) and a bespoke tracer (`cerevra/monitoring/tracing.py` — `TraceContext`/`Span`/`Tracer`, **not** OpenTelemetry, no context propagation). Neither is adopted. What *is* adopted is Cerevra's metric-naming discipline and its `what` / `why` / `outcome` / `safe` structured-log convention, both of which are genuinely good.

---

## 1. The required trace

One trace, one correlation id, from discovery to delivery.

```
source.discovery
 └─ ingestion.command.append          ← command row inserted
     └─ ingestion.command.claim
         └─ source.fetch              ← breaker state as span attribute
             └─ object.store          ← S3 PUT, verified
                 └─ document.fingerprint
                     └─ document.version.create
                         └─ parser.execute      ← parser_version attribute
                             └─ evidence.create
                                 └─ event.match
                                     └─ projection.apply   ← ruleset_version
                                         └─ geospatial.impact
                                             └─ notification.decide
                                                 └─ notification.deliver
```

The trace **crosses the worker boundary**: the command row carries the trace context, so a span started by the scheduler continues in whichever worker claims it. This is the specific capability Cerevra's bespoke tracer lacks, and it is why OpenTelemetry is non-negotiable.

Required span attributes throughout: `civic_node_id`, `source_id`, `jurisdiction_path`, `parser_version`, `ruleset_version`, `command_id`. Never: user id, precise location, reporter identity.

---

## 2. Metrics

### Ingestion

| Metric | Type | Labels |
| --- | --- | --- |
| `ingestion_commands_total` | Counter | `command_type`, `state` |
| `ingestion_retries_total` | Counter | `command_type`, `error_class` |
| `wal_lag` | Gauge | — · age of the oldest `PENDING` command, seconds |
| `wal_unacknowledged_total` | Gauge | — · count in `PENDING` + `CLAIMED` |
| `source_fetch_duration_seconds` | Histogram | `source_id`, `outcome` |
| `source_health_age_seconds` | Gauge | `source_id` · seconds since last success |
| `source_document_duplicates_total` | Counter | `source_id` |
| `source_document_versions_total` | Counter | `source_id` |
| `parser_failures_total` | Counter | `parser_version`, `error_class` |

`wal_lag` and `wal_unacknowledged_total` keep their names despite the log being a table rather than a file — the semantics are identical and the names are established.

### Evidence and events

| Metric | Type | Labels |
| --- | --- | --- |
| `evidence_items_total` | Counter | `authority_class`, `category`, `parser_version` |
| `events_by_state` | Gauge | `state`, `category`, `jurisdiction` |
| `stale_events_total` | Gauge | `category` |
| `event_corrections_total` | Counter | `correction_type` (`merge`, `split`, `dispute`, `reopen`) |
| `projection_lag_seconds` | Gauge | `partition_key` |
| `replay_duration_seconds` | Histogram | `projection_name`, `scope` (`full`, `incremental`) |
| `replay_divergence_total` | **Counter** | `projection_name`, `partition_key` |

**`replay_divergence_total` is the single most important metric in JanSetu.** Any non-zero value means a projection cannot be reproduced from its evidence, which falsifies the platform's central claim. It pages immediately. Cerevra has no equivalent because it stores no projection hash.

### Geospatial and notifications

| Metric | Type | Labels |
| --- | --- | --- |
| `geospatial_query_duration_seconds` | Histogram | `query_type` |
| `route_impact_calculations_total` | Counter | `impact_class` |
| `notification_decisions_total` | Counter | `outcome` (`send`, `suppress_cap`, `suppress_stale`, `suppress_quiet`, `suppress_threshold`) |
| `notification_delivery_failures_total` | Counter | `channel`, `error_class` |
| `notification_delivery_duration_seconds` | Histogram | `channel` |

Suppression outcomes are labelled, not aggregated. "We chose not to alert you, and here is why" must be measurable (I23).

### Moderation, federation, trust

| Metric | Type | Labels |
| --- | --- | --- |
| `moderation_queue_age_seconds` | Gauge | `queue`, quantile |
| `federation_signature_failures_total` | Counter | `origin_node_id`, `failure_reason` |
| `authorisation_denials_total` | Counter | `permission`, `reason` |
| `invariant_violations_total` | **Counter** | `invariant_id` |

`invariant_violations_total` should be permanently zero. It exists so that a violation is loud rather than silent.

### Cardinality

`source_id` and `jurisdiction` are bounded by registration and are safe. `civic_node_id` is bounded by node count. **Never** label by user, event id, or precise location.

---

## 3. SLOs

### Scenario A — PMVP

| SLO | Target | Window |
| --- | --- | --- |
| Event-query availability | 99.0% | 30 d |
| Ingestion availability | 99.0% | 30 d |
| p95 event read latency | < 300 ms | 7 d |
| p95 route impact latency | < 800 ms | 7 d |
| Max source staleness | < 3× poll interval | continuous |
| Event publication latency (evidence → visible) | p95 < 60 s | 7 d |
| Notification decision latency | p95 < 120 s | 7 d |
| Full replay completion | < 30 s | per run |
| Replay divergence rate | **0** | absolute |
| Moderation queue age | p95 < 24 h | 7 d |

### Scenario B — city

| SLO | Target |
| --- | --- |
| Event-query availability | 99.5% |
| Ingestion availability | 99.0% |
| p95 event read latency | < 200 ms |
| p95 route impact latency | < 500 ms |
| Event publication latency | p95 < 30 s |
| Notification decision latency | p95 < 30 s |
| Notification delivery latency (non-burst) | p95 < 120 s |
| Incremental replay | < 60 s |
| Full replay | < 60 min |
| Replay divergence rate | **0** |
| Projection lag | p95 < 15 s |
| Moderation queue age | p95 < 4 h |

### Scenario C/D — national

| SLO | Target |
| --- | --- |
| Event-query availability | 99.95% (read) / 99.9% (write) |
| p95 event read latency | < 200 ms in-region |
| p95 route impact latency | < 500 ms |
| Event publication latency | p95 < 30 s |
| Notification decision latency | p95 < 30 s |
| Partitioned full replay | < 15 min |
| Replay divergence rate | **0** |
| Projection lag | p95 < 15 s |
| Federation signature verification | p99 < 50 ms |
| Cross-region RPO | ≤ 60 s |

The read-path availability target exceeds the write path deliberately: during a disaster, being able to *read* a closure matters more than being able to publish a new one.

### Disaster mode

| SLO | Target | Note |
| --- | --- | --- |
| Highest-severity event read availability | **99.99%** | Served from static/CDN fallback if the database is down |
| p95 read latency | < 500 ms | Relaxed |
| Notification delivery (severity ≥ high) | p95 < 5 min | Prioritised |
| Notification delivery (all) | **best effort, disclosed** | Up to ~1 h to drain 100 M at Scenario D |
| Event publication latency | p95 < 60 s | Relaxed |
| **Trust-rule enforcement** | **100%, unrelaxed** | I24 — nothing here loosens |

Disaster mode relaxes latency and completeness. It does not relax correctness. That distinction is the whole point of having a defined mode rather than ad-hoc degradation.

---

## 4. Alerts

### Page immediately

| Condition | Meaning |
| --- | --- |
| `replay_divergence_total > 0` | A projection cannot be reproduced. Platform claim falsified |
| `invariant_violations_total > 0` | A structural guarantee has been breached |
| Transition to `OFFICIALLY_CONFIRMED` with `official_evidence_count = 0` | I3 bypassed — should be impossible |
| `federation_signature_failures_total` rate > 0 | Tampering or key compromise |
| Route-table access by a non-route role | I17 breach |
| Successful cross-node write detected by audit query | I14 breach |
| Event-query availability below SLO burn rate | Citizens cannot read civic information |
| Object digest mismatch on read | Raw document integrity failure (I7) |

### Warn

| Condition | Meaning |
| --- | --- |
| `wal_unacknowledged_total` rising monotonically 15 min | Workers stuck or dead |
| `ingestion_retries_total / ingestion_commands_total > 0.2` over 30 min | Sources degrading |
| Any source with `source_health_age_seconds` > 3× poll interval | Source down (I15) |
| `stale_events_total` > 5% of active for 24 h | Source coverage failing |
| `projection_lag_seconds` p95 > 60 s | Projection falling behind |
| `moderation_queue_age_seconds` p95 > SLO | Moderation under-resourced |
| Redis unavailable > 5 min | Degraded, not down (I10) |
| Notification suppressed-stale share > 20% | Source coverage failing |
| Fraction of active events with unknown end time drops sharply | Something began fabricating values (I16) |

### Deliberately not alerted

- Individual parser failures — expected; only the *rate* matters.
- Individual notification delivery failures — retries handle these.
- Individual `CONFLICTING` events — that is the system working correctly.

---

## 5. Dashboards

| Dashboard | Audience | Content |
| --- | --- | --- |
| Civic health | Public | Events by state and category, source health, evidence freshness distribution |
| Ingestion | Operators | Command throughput, retries, lag, source health, breaker states |
| Trust | Operators | State distribution, transition rates, corrections, conflicts, moderation queue |
| Replay | Operators | Duration, **divergence**, checkpoint lag, last full rebuild |
| Delivery | Operators | Decisions by outcome, delivery latency, failures by channel |
| Federation | Operators | Nodes by trust level, signature failures, key ages, suspensions |

**The civic-health dashboard is public.** A platform asking citizens to trust its evidence should expose its own operational honesty — including how many of its sources are currently failing.

---

## 6. Structured logging

Every operational log carries `what`, `why`, `outcome`, and `safe`, plus `request_id` and `trace_id`.

`safe=true` marks a **deliberate refusal**: we declined to publish, declined to confirm, declined to notify. `safe=false` marks a genuine failure.

```
event=event.publication.declined
what="did not publish event"
why="single anonymous report, no corroboration"
outcome="state=UNVERIFIED"
safe=true
```

These two categories must never look alike in a dashboard. A civic platform that cannot distinguish "correctly refused" from "broke" cannot be operated honestly — and cannot tell whether its trust rules are working or its ingestion is failing.

Never logged: precise location, reporter identity, saved-route geometry, session tokens, private keys.
