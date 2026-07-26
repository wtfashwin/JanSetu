# Adversarial Review

Five independent reviews of the architecture as designed. Each is written to attack, not to confirm. Findings that changed the design are marked **CHANGED**; findings accepted as residual risk are marked **ACCEPTED**; findings rejected are marked **REJECTED** with a reason.

---

## Review 1 — Simplicity

> What has been over-engineered? Remove anything that does not protect an invariant or satisfy a measured requirement.

### Findings

**1.1 Bitemporality on evidence — challenged, retained.**
`observed_at` plus `known_at` doubles the reasoning cost on every query. Is it needed at PMVP?
*Verdict:* **REJECTED (keep).** "The road closed at 06:00" and "we learned at 07:30" are different facts, and invariant 1 forbids collapsing them. Retrofitting a second time dimension onto existing rows is a data migration nobody wants. Cost is two columns.

**1.2 `event_merges` / `event_splits` at PMVP — challenged.**
5,000 events in one city. Will anything merge?
*Verdict:* **ACCEPTED as deferred.** The *tables* and identity redirect land at PMVP (cheap, and a dead public URL is unacceptable). The *UI and workflow* defer to MVP. Issue #17 scope adjusted accordingly.

**1.3 Fifteen event states — challenged.**
`STATUS_UNKNOWN`, `STALE`, `PARTIALLY_RESOLVED`, `REOPENED`, `DISPUTED`, `CONFLICTING` — six states expressing degrees of uncertainty.
*Verdict:* **CHANGED (partially).** `DISPUTED` and `CONFLICTING` are genuinely distinct (one is a process, one is an evidence property) but citizens will not distinguish them. **Decision: keep both internally; the citizen UI renders both as "conflicting information — see evidence".** The internal distinction earns its keep for moderation; the external one does not.

**1.4 Control-plane/data-plane separation at PMVP — challenged.**
*Verdict:* **REJECTED (already minimal).** ADR 0015 already specifies logical-only separation at PMVP: separate modules and schemas, one deployable. That costs nothing.

**1.5 Projection checkpoints at PMVP — challenged.**
Full replay takes 2.5 s at Scenario A. Checkpoints are dead weight.
*Verdict:* **REJECTED (keep).** The checkpoint row also stores the projection hash, which is what makes divergence detectable (ADR 0014). The hash is the point; the checkpoint is nearly free alongside it.

**1.6 Two circuit-breaker code paths — challenged.**
*Verdict:* **CHANGED.** The audit found Cerevra shipping two implementations. **One implementation, one module.** Noted as an explicit review criterion.

**1.7 Five duplicate-key classes — challenged as over-modelling.**
*Verdict:* **REJECTED (keep).** Each class has a distinct correct behaviour, and the audit shows what collapsing them costs: a truncated hash serving as identity, dedup key and primary key simultaneously, with a collision silently discarding a civic notice.

### Removed as a result of this review

- Merge/split **UI and workflow** deferred to MVP (tables retained).
- External `DISPUTED` vs `CONFLICTING` distinction collapsed in the citizen UI.
- Explicit "one implementation per concern" review criterion added.

---

## Review 2 — Scale

> What fails first at city, multi-city and national scale?

### First failure at city scale (Scenario B)

**Full projection replay: 42 minutes.**
*Symptom:* an incident requiring a rebuild cannot complete inside the incident.
*Detection:* `replay_duration_seconds`.
*Mitigation:* checkpointed incremental rebuild (< 60 s typical).
*Trigger:* full replay > 10 min. **Already reached at Scenario B**, which is why checkpoints are designed in from PMVP.

**Second: read amplification.** 1,390 reads/s against 26 writes/s. Detection: primary read CPU. Mitigation: replica + cache + CDN.

**Third: notification bursts.** 200 K deliveries starving ingestion. Detection: `projection_lag_seconds` rising during fan-out. Mitigation: separate worker and queue.

### First failure at multi-city scale (Scenario C)

**Replay becomes intractable single-threaded: 6.9 hours.**
*Mitigation:* 32-way partitioned replay → 13 min. **This is the requirement that makes jurisdiction the primary partition key** — not an aesthetic preference. **CHANGED:** this reasoning is now stated explicitly in [national-federated.md](../scaling/national-federated.md) §3, because it was previously implicit and a future maintainer could have chosen a different shard key.

**Second: 12.5 M-delivery fan-out** taking ~10 minutes to drain. Not fixable by scaling alone; addressed by severity prioritisation and by disclosure.

**Third: geospatial p95** under 3,000 impact calculations/s. Mitigated by the optimisation ladder, then H3 pre-filter.

### First failure at national scale (Scenario D)

**Notification fan-out: 100 M deliveries, ~27,800/s sustained to drain in one hour.**
*Verdict:* **ACCEPTED as a product limitation.** Not an engineering bug. Must be disclosed to citizens and must drive severity prioritisation.

**Second: cross-shard event semantics.** A cyclone spanning six states does not respect the shard key. **ACCEPTED as the hardest unsolved correctness surface**; recorded as an open question.

**Third: the control plane** becoming a global bottleneck. Mitigated by cached policy with last-known-good fallback (ADR 0015).

### What does *not* fail

Write throughput. 1,060 evidence writes/s at national peak sits inside a single well-tuned Postgres primary. **The instinct to shard for write throughput would be wrong at every modelled scenario**, and this review confirms it.

---

## Review 3 — Reliability

> Where can data be lost, duplicated, reordered, or silently corrupted?

Tracing every write path.

| Path | Loss | Duplication | Reordering | Silent corruption |
| --- | --- | --- | --- | --- |
| Command insert | No — unique key, committed | No — `ON CONFLICT` | n/a | No |
| Command claim | No — lease expiry | No — `SKIP LOCKED` + `claimed_by` guard | n/a | No |
| Source fetch | Retried | Absorbed downstream | n/a | Detected by digest |
| S3 write | Retried | Content-addressed | n/a | Digest verified on read |
| **S3 → DB gap** | **Orphan object** | No | n/a | No |
| Version/evidence insert | No — in the ack transaction | No — unique keys | No — identity column | No |
| Projection | No — rebuildable | No — idempotent fold | No — ordered by sequence | **Detected by hash** |
| Notification | Retried on the same row | No — unique key | Severity-ordered by design | No |
| Federation ingest | Retried | Envelope-hash unique | Nonce + time window | Signature verified twice |

### Findings

**3.1 The S3→DB gap is the only genuine cross-store window.** **ACCEPTED**, mitigated by fixed ordering (object first, verified) plus an orphan sweeper. An orphan object is harmless; a dangling reference is not. Chaos-tested.

**3.2 Reordering within a source is possible if two workers claim two commands for the same source.** **CHANGED:** an advisory lock **per source** (not merely per command) is now specified, so a source's documents are processed in discovery order. Previously the design relied on the projection fold being order-insensitive, which is true for the fold but not for the `previous_version_id` revision chain.

**3.3 Silent corruption of the projection is the highest-consequence risk** and is the only one detected rather than prevented. **ACCEPTED** — detection is the correct control here, and it is why `replay_divergence_total` pages.

**3.4 A ruleset bug is loss-equivalent** — it produces wrong civic states without corrupting a byte. **CHANGED:** shadow rollout of new rulesets against a sampled partition is now mandatory before activation, not merely recommended (ADR 0007).

**3.5 An orphaned S3 object containing personal data** would survive an erasure request. **CHANGED:** the sweeper now runs against the erasable media bucket on a **daily** schedule rather than weekly, because there the orphan is a privacy issue rather than a cost issue.

---

## Review 4 — Security and privacy

> How could this become harmful public infrastructure?

### The four ways JanSetu becomes harmful

**4.1 It publishes a false event that people act on.**
Controls: authority binding by jurisdiction and category (I3, DB-enforced); corroboration requiring spatial diversity; visible provenance on every event; moderation for high-severity categories.
*Residual:* a legitimately-authorised node publishing a falsehood. **ACCEPTED.** Cryptography cannot prevent it. The design's actual protection is that a successful attack produces a *visible, labelled, correctable* claim rather than an invisible authoritative one.

**4.2 It becomes a surveillance asset.**
This is the risk that would be *created by success* — a popular civic app knows where millions of people commute.
Controls: routes in a separate schema with no public grant, encrypted, 30-day impact retention, k-anonymity for any aggregate; no continuous location history; no contact graph; no health/caste/religion/political data at all.
**CHANGED:** the "deliberately not collected" list is now stated as an explicit product boundary in [security-and-federation.md](../security-and-federation.md) §6, rather than being implied by the schema. A boundary that is written down survives a future product manager; a schema does not.

**4.3 It becomes a censorship or misinformation instrument.**
A government node could suppress an event by publishing a false "resolved".
Controls: resolution requires positive evidence (I4); contradictory community evidence produces a visible `CONFLICTING` state; **conflict is represented, never resolved by the platform**; all transitions are public and attributed.
*Residual:* **ACCEPTED.** A citizen can see that an authority claimed resolution and that reports contradict it. That is the most a platform can honestly do without arrogating the power to decide which authority is right.

**4.4 It becomes a notification weapon.**
Controls: per-user/event/day caps; severity thresholds; quiet hours; freshness budgets; per-node publication rate limits.
*Residual:* alert fatigue from legitimate high-severity events. Not a bug.

### Additional findings

**4.5 Coarse cells are weaker anonymity in sparse areas.** A report from a low-density cell narrows the reporter set considerably. **CHANGED:** adaptive cell sizing by population density is now a named requirement rather than an aside, and is flagged as needing detailed specification.

**4.6 Subscription correlation can infer home and work** even without saved routes. **ACCEPTED**, with subscriptions carrying the same isolation as routes.

**4.7 Insider risk.** A maintainer with database access could alter a projection. Mitigated by rebuild-and-compare (the tampering shows as divergence), audit logging, and least-privilege roles. **ACCEPTED, not eliminated.**

**4.8 Managed signing for small civic bodies** weakens the "JanSetu never holds private keys" guarantee for those nodes. **CHANGED:** ADR 0012 now states that such nodes must be *labelled* as managed-signing, so the weaker guarantee is visible rather than assumed away.

---

## Review 5 — Operability

> Could three engineers operate this during a major incident?

### PMVP: yes

Two process types, one database, one bucket, one cache. One dashboard, one log stream. Recovery from a worker crash is lease expiry — automatic, with no runbook step. Full replay is 2.5 s. `FLUSHALL` on Redis is a non-event by design.

**The 03:00 flood test:** an engineer woken by an alert needs to determine (a) are citizens getting information, (b) are sources healthy, (c) is the projection current. Three panels answer all three.

### City scale: yes, with effort

Five process types, primary + replica, one bucket, one cache, one CDN. The new operational burdens are replica lag and CDN purge correctness.

**CHANGED:** replica-lag handling was under-specified. The router now **fails back to the primary and alerts** when lag exceeds the freshness budget, rather than silently serving stale civic data. Serving a stale "road open" is worse than serving a slower correct answer.

### National scale: no — this needs a team

Multiple regions, a broker, OpenSearch, ClickHouse, Kubernetes, a federation gateway, a control plane. **ACCEPTED and stated plainly** in [national-federated.md](../scaling/national-federated.md) §12: this is the honest staffing cost of Scenario D, and it is precisely why every earlier document works to defer arriving there.

### Findings

**5.1 The static disaster fallback is the highest-value operability feature** in the whole design — it means the read path survives a total database failure during the event that matters most. **CHANGED:** moved from "city scale" to **PMVP** scope (ADR 0016). A PMVP that cannot survive its first flood has not been tested against its actual purpose.

**5.2 Runbooks are not yet written.** Needed before the first public deploy: replica promotion, projection rebuild, source-adapter failure, key revocation, disaster-mode activation, erasure request. **Recorded as a gap.**

**5.3 One scheduler process is a single point of failure.** **ACCEPTED at PMVP** — a missed poll cycle is recoverable and visible via `source_health_age_seconds`. Leader election deferred.

---

## Summary of changes made

| # | Change |
| --- | --- |
| 1.2 | Merge/split UI deferred to MVP; tables and id redirects retained at PMVP |
| 1.3 | `DISPUTED`/`CONFLICTING` collapsed in the citizen UI, retained internally |
| 1.6 | "One implementation per concern" added as a review criterion |
| 2.C | Jurisdiction-as-partition-key justified explicitly by partitioned-replay arithmetic |
| 3.2 | Per-source advisory lock added, protecting the revision chain |
| 3.4 | Shadow rollout of new rulesets made mandatory |
| 3.5 | Media-bucket orphan sweeper moved to daily |
| 4.2 | "Deliberately not collected" written as an explicit product boundary |
| 4.5 | Adaptive cell sizing named as a requirement |
| 4.8 | Managed-signing nodes must be labelled as such |
| 5.1 | Static disaster fallback moved into PMVP scope |

## Accepted residual risks

1. A legitimately-authorised node publishing a false event.
2. Notification fan-out latency at national scale (~1 h for 100 M).
3. Cross-shard event semantics for events spanning many jurisdictions.
4. Insider database access.
5. Coarse-cell anonymity in sparsely-populated areas.
6. National scale requires a team, not three engineers.

## Open gaps

1. Runbooks (5.2).
2. Adaptive cell-sizing specification (4.5).
3. Cross-shard merge semantics (2.D).
4. SMS cost policy — product decision, not architectural.
