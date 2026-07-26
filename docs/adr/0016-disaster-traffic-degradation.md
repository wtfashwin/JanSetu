# 0016 — Disaster-mode degradation: relax latency, never correctness

Status: Accepted
Date: 2026-07-27

## Context

JanSetu's peak load coincides exactly with its peak value. A cyclone, a flood, or a major incident produces a ×50 read burst, a ×100 write burst, a report flood, and a fan-out of up to 100 M notifications — at the moment when citizens most need correct information.

Two failure modes are available under that pressure, and both are tempting:

1. **Fall over.** The platform is unavailable during the emergency it exists for.
2. **Lower the bar.** Auto-confirm reports, skip moderation, relax verification — because "people need to know now".

The second is worse. An unavailable platform is a failure; a platform confidently publishing unverified emergency information is a hazard.

## Decision

**Disaster mode is an explicit, audited operational switch that relaxes latency and completeness, and never relaxes correctness.**

### Permitted

- Raise community-report intake limits and add report-worker capacity.
- Shed non-essential read features: historical search, analytics endpoints, similar-event discovery.
- Reprioritise notification queues by impact severity.
- Extend cache TTLs on stable content to protect the database.
- Relax latency SLOs (p95 read 200 ms → 500 ms; publication p95 30 s → 60 s).
- Serve a **static fallback**: a CDN-cached list of the highest-severity active events, servable **with the database down**. The read path degrades to reduced information, never to an error page.

### Forbidden

- Lowering confirmation thresholds.
- Auto-confirming community reports.
- Skipping moderation for ambiguous or high-severity events.
- Relaxing signature verification.
- Publishing without evidence.
- Estimating unknown end times.

**There is no disaster variant of the trust ruleset.** Ruleset selection is by version id only; no runtime threshold-override column exists. Structurally, there is nothing to loosen.

Rate limiting keeps a **trusted-reporter lane**, so genuine reports are not suppressed alongside abuse — the highest-consequence rate-limiting failure available.

## Alternatives considered

| Alternative | Rejected because |
| --- | --- |
| No disaster mode; scale to peak always | Cost at Scenario D peak, provisioned permanently, is prohibitive |
| Auto-confirm during emergencies | Violates invariants 2 and 7. The moment verification matters most is not the moment to abandon it |
| Ad-hoc on-call degradation | Unrepeatable, unaudited, and the decision gets made at 03:00 under pressure |
| Shed writes, keep reads | Partially adopted — writes degrade in latency, but report intake is *raised*, since citizen reports are most valuable during a disaster |
| Runtime-tunable thresholds | Creates the exact lever this ADR exists to remove |

## Consequences

Behaviour under the highest-consequence conditions is defined in advance and tested. The read path stays available even with the database down. Citizen reports are welcomed rather than throttled. Trust guarantees hold identically at 03:00 during a cyclone and at noon on a Tuesday.

Cost: a static fallback pipeline to build and keep warm; mode transitions to rehearse; explicit SLO documentation per mode.

## Risks

- **Pressure to relax the forbidden list.** This is the real risk, and it is organisational. Mitigated by making relaxation structurally impossible (no override exists) rather than merely discouraged — so bypassing it requires a code change and a review, not a config toggle.
- **Static fallback going stale.** Mitigated by continuous regeneration and by displaying its `as_of` prominently.
- **Mode not activated in time.** Mitigated by automatic activation on load thresholds, with manual override.
- **Mode not deactivated.** Mitigated by an auto-expiry with explicit renewal.

## Measurement

- Time to activate after threshold breach.
- Read availability for highest-severity events during disaster mode (target 99.99%).
- Notification delivery p95 for severity ≥ high (target < 5 min).
- **Any `OFFICIALLY_CONFIRMED` transition during disaster mode lacking official evidence: must be zero** — should be structurally impossible (I3); if it fires, a constraint has been bypassed.
- Post-incident audit of every transition made during the mode.

## Introduction trigger

Static fallback and the mode switch at PMVP — a PMVP that cannot survive its first flood has not been tested against its actual purpose. Automatic activation and severity-prioritised queues at city scale.

## Rollback strategy

Disable the switch; the system runs at normal SLOs. The forbidden list is not a feature that can be rolled back — it is the absence of a capability.
