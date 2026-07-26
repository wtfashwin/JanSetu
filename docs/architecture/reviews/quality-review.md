# Architecture Quality Review

Sixteen categories scored 0–10. Any category below 8 requires a documented weakness, a revision, and a re-score. Two revision passes were run.

**Critical categories** — `Correctness`, `Evidence integrity`, `Reliability`, `Security`, `Privacy`, `Operability` — block approval if below 8, regardless of the average.

Scoring is against *this architecture as documented*, not against an implementation that does not yet exist. That distinction bounds what these scores can mean, and it is stated again in the closing section.

---

## Pass 1 — initial scores

| Category | Score | Note |
| --- | --- | --- |
| Correctness | 8 | Invariants enumerated with DB enforcement |
| Evidence integrity | 9 | Append-only + divergence detection |
| Simplicity | **6** | Fifteen states, bitemporality, five dedup keys at PMVP |
| Reliability | 8 | Transactional acknowledgement, lease recovery |
| Replayability | 9 | Checkpoints + canonical hash |
| Scalability | 8 | Workload-modelled with triggers |
| Security | 8 | Layered enforcement, threat-modelled |
| Privacy | **7** | Route isolation strong; coarse-cell weakness unaddressed |
| Observability | 8 | Metrics, traces, SLOs per scenario |
| Testability | 8 | Six tiers, property + chaos |
| Operability | **7** | Runbooks absent; replica-lag behaviour undefined |
| Open-source extensibility | 8 | Adapter SDK, no-Docker test tiers |
| Geospatial design | 8 | PostGIS authoritative, H3 trigger-gated |
| Federation readiness | 8 | Signed envelopes, key lifecycle |
| Cost awareness | **7** | Infra costed; notification cost under-analysed |
| Migration clarity | 9 | Measurable triggers with derivations |

**Average 7.9. Two critical categories below 8 (Privacy 7, Operability 7). Approval blocked.**

---

## Revision pass 1

### Simplicity 6 → 8

*Weakness:* internal modelling richness leaking into citizen-facing complexity, and PMVP carrying MVP machinery.

*Revisions (from [adversarial-review.md](adversarial-review.md) §1):*
- Merge/split **UI and workflow** deferred to MVP; tables and id-redirect retained (a dead public URL is unacceptable, but the workflow is not needed at 5,000 events).
- `DISPUTED` and `CONFLICTING` collapsed to one citizen-facing presentation; the internal distinction earns its keep only for moderation.
- "One implementation per concern" added as an explicit review criterion, after the audit found two circuit breakers and two WAL stacks in Cerevra.

*Retained after challenge:* bitemporality (invariant 1 — "closed at 06:00" and "we learned at 07:30" are different facts, and retrofitting a time dimension is a migration nobody wants); five dedup keys (the audit shows a collapsed key silently discarding civic notices); checkpoints at PMVP (they carry the projection hash, which is the point).

**New score: 8.** Not higher — this architecture is genuinely more complex than a CRUD event app, and that complexity is the cost of the trust guarantees. Claiming 9 would be flattering it.

### Privacy 7 → 9

*Weakness:* coarse cells are weaker anonymity in sparse areas; the "we do not collect X" boundary lived only in the schema, where a future product manager would not find it.

*Revisions:*
- Adaptive cell sizing by population density named as a requirement (§4.5 of the adversarial review).
- The deliberately-not-collected list written as an explicit product boundary in [security-and-federation.md](../security-and-federation.md) §6 — no health, caste, religion, political affiliation, continuous location history, or contact graph.
- Media-bucket orphan sweeper moved to daily, because there an orphan is a privacy issue rather than a cost issue.
- Immutability/erasure resolved **structurally** (two buckets with different lock configurations) rather than by policy, in ADR 0013.

**New score: 9.**

### Operability 7 → 9

*Weakness:* replica-lag behaviour undefined; the disaster fallback was scoped too late; runbooks absent.

*Revisions:*
- Replica router **fails back to the primary and alerts** when lag exceeds the freshness budget. Serving a stale "road open" is worse than serving a slower correct answer.
- Static disaster fallback moved from city scale into **PMVP** scope (ADR 0016) — a PMVP that cannot survive its first flood has not been tested against its purpose.
- Runbooks recorded as a named gap with a specific list.

**New score: 9.** Not 10: the runbooks are still unwritten, and national scale genuinely needs a team.

### Cost awareness 7 → 8

*Weakness:* infrastructure costed per scenario, but notification delivery — plausibly the dominant cost at Scenario C+ — was under-analysed.

*Revisions:* SMS identified as the sleeper cost in [workload-model.md](../scaling/workload-model.md); a hard budget cap and severity gating required as product policy before the channel is built; reverse triggers added to [migration-triggers.md](../scaling/migration-triggers.md) so components can be *removed* when their trigger no longer holds.

**New score: 8.** Not higher: LLM inference cost at scale is still unmodelled.

---

## Pass 2 — re-score

| Category | Pass 1 | Pass 2 | Change |
| --- | --- | --- | --- |
| Correctness | 8 | **9** | Per-source advisory lock closed a reordering hazard in the revision chain (adversarial §3.2) |
| Evidence integrity | 9 | **9** | — |
| Simplicity | 6 | **8** | Scope deferrals; UI state collapse |
| Reliability | 8 | **9** | Mandatory shadow rollout of rulesets (§3.4) |
| Replayability | 9 | **9** | — |
| Scalability | 8 | **9** | Partition-key choice now justified by replay arithmetic, not asserted |
| Security | 8 | **8** | Managed-signing nodes must be labelled; residual risk unchanged |
| Privacy | 7 | **9** | Explicit boundary; adaptive cells; structural erasure |
| Observability | 8 | **8** | — |
| Testability | 8 | **9** | Standing `FLUSHALL` test; coverage policy replaced with branch + mutation on invariant modules |
| Operability | 7 | **9** | Replica fallback; disaster fallback at PMVP |
| OSS extensibility | 8 | **8** | — |
| Geospatial | 8 | **8** | — |
| Federation readiness | 8 | **8** | — |
| Cost awareness | 7 | **8** | Notification cost surfaced; reverse triggers |
| Migration clarity | 9 | **9** | — |

**Average 8.6. No category below 8. All six critical categories ≥ 8.**

---

## Revision pass 2 — remaining sub-9 categories

Four categories sit at 8. Each was examined again; none was revised further, and the reasons are recorded so that the scores are not mistaken for exhausted effort.

### Security 8

*Residual:* a legitimately-authorised node publishing a false event. Cryptography cannot prevent this; the controls (visible provenance, `CONFLICTING` as a real state, moderation, suspension) are all after-the-fact.

*Not revised because* the honest ceiling is 8. Reaching 9 would require adjudicating which authority is right, which is a power JanSetu should not hold. **The design's actual security property is that a successful attack produces a visible, labelled, correctable claim rather than an invisible authoritative one** — and that property is achieved.

### Observability 8

*Residual:* metrics and traces are specified but unimplemented; SLOs are targets, not measurements.

*Not revised because* the gap closes with implementation, not with more design. Adding paper detail would inflate the score without changing anything.

### Geospatial 8

*Residual:* cross-jurisdiction events and reference-data maintenance (road segments change; boundaries get redrawn) are real ongoing data problems that the architecture handles adequately but does not solve.

*Not revised because* this is a data-curation problem, not an architecture problem.

### OSS extensibility 8

*Residual:* the adapter SDK is designed but unvalidated. No external contributor has attempted it.

*Not revised because* the only meaningful revision is evidence from a real contributor.

### Federation readiness 8

*Residual:* key management is a genuine burden for municipal IT departments, and the managed-signing fallback weakens the guarantee for exactly the nodes most likely to need it.

*Not revised because* the trade-off is real and is now labelled rather than hidden.

---

## Final

| | |
| --- | --- |
| Average | **8.6** |
| Lowest | 8 (Security, Observability, Geospatial, OSS extensibility, Federation) |
| Critical categories | All ≥ 8 — **approval not blocked** |
| Highest | 9 (Correctness, Evidence integrity, Reliability, Replayability, Scalability, Privacy, Testability, Migration clarity) |

## What these scores do not mean

Stated because a high average is exactly the thing that invites over-reading.

1. **This scores a design, not a system.** No code exists. The most likely source of a score being wrong is implementation drift from the documented design.
2. **The workload model is unmeasured.** Every scalability and cost score rests on labelled assumptions in [workload-model.md](../scaling/workload-model.md). That document lists five specific findings that would falsify it. If average document size is 2 MB rather than 200 KB, or fold throughput is 2,000 rows/s rather than 20,000, several triggers move earlier and some scores fall.
3. **The residual risks are real and are not scored away.** A captured authority, a patient brigade, insider access, and national-scale staffing cost are all accepted, not solved.
4. **No category reached 10, and none should have.** A 10 would mean nothing further could be learned from operating the system, which is never true before it has run.

The architecture is approved to proceed to implementation. The first correction to this document should come from production measurement, not from further review.
