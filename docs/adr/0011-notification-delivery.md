# 0011 — Notification delivery semantics

Status: Accepted
Date: 2026-07-27

## Context

Notifications are JanSetu's highest-consequence output. They interrupt people. Get them wrong in either direction and the product fails: too many and alert fatigue disables the channel exactly when it matters ([threat-model.md](../architecture/threat-model.md) T13); too few and a citizen drives into a flooded underpass.

Scale: 500 deliveries per burst at PMVP, 200 K at city scale, 12.5 M at Scenario C, 100 M at Scenario D — requiring ~27,800/s sustained to drain in one hour.

## Decision

**At-least-once delivery with a database-enforced dedup key.**

```
notification_deliveries UNIQUE (subscription_id, event_id, transition_sequence)
```

Retries re-attempt the **same row**. A duplicate send is structurally impossible (I9).

Decisions are recorded **including suppressions**, with a reason: `send`, `suppress_cap`, `suppress_stale`, `suppress_quiet`, `suppress_threshold`. "We chose not to alert you" must be auditable, not invisible.

Gating rules:
- **Freshness budget per category** — an event whose evidence has aged past budget cannot alert, and transitions to `STATUS_UNKNOWN` (I23). Stale evidence must not produce indefinite alerts.
- Per-user, per-event, per-day caps.
- User-configurable severity thresholds and quiet hours.
- Route-based relevance, computed per transition.

**Prioritisation is by impact severity, never FIFO.** A route-affected citizen outranks a general area subscriber. At Scenario D this is the difference between a useful alert and a useless one.

Delivery latency at full fan-out (~10 min at Scenario C, ~1 h at Scenario D) is **designed and disclosed**, not discovered. Citizens are shown when an alert was decided and when it was sent.

SMS requires a hard budget cap and severity gating, settled as product policy before the channel is built.

## Alternatives considered

| Alternative | Rejected because |
| --- | --- |
| Fire-and-forget with in-memory retry | The pattern the Cerevra audit found losing work silently (`handler.py:411-449`) |
| Exactly-once delivery | Not achievable to an external provider. Honest dedup is the unique key |
| FIFO queues | At 12.5 M deliveries, the most affected user could be last |
| No suppression record | Cannot answer "why didn't I get an alert?" — unacceptable for civic infrastructure |
| Send on every evidence item | Alert fatigue. Alerts fire on *transitions*, not on evidence |

## Consequences

Duplicate alerts are impossible. Every decision is explicable to the affected citizen. Fan-out cannot starve ingestion (separate worker and queue from city scale). Severity prioritisation makes large fan-outs useful rather than merely large.

Cost: a decision row per candidate subscription — 12.5 M rows for one Scenario C event. Mitigated by 90-day retention and monthly partitioning.

## Risks

- **Alert fatigue** — the highest risk, and it is a product risk more than a technical one. Mitigated by caps, thresholds, quiet hours, and transition-triggered (not evidence-triggered) alerts.
- **Fan-out latency during a national emergency.** A genuine product limitation. Disclosed, and mitigated by severity prioritisation.
- **SMS cost.** Hard cap plus severity gating, decided before the channel exists.
- **Provider outage.** Per-provider circuit breakers; deliveries retried against the same row; failures visible.

## Measurement

- `notification_decisions_total{outcome}` — the suppression breakdown is the health signal.
- `notification_delivery_failures_total{channel, error_class}`.
- Delivery latency p95 per channel; drain time for large fan-outs.
- Deliveries per user per day — alert on exceeding the policy cap.

## Introduction trigger

Simple in-monolith delivery at PMVP. Separate worker and queue when a fan-out burst delays projection beyond its SLO (Scenario B). Independent notification service when fan-out dominates worker capacity (Scenario D).

## Rollback strategy

Merge worker pools. The dedup key and decision-recording behaviour are unaffected by deployment topology.
