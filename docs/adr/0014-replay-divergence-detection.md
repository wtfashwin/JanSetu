# 0014 — Replay divergence detection

Status: Accepted
Date: 2026-07-27

## Context

JanSetu's central claim is that a public event can be rebuilt from append-only evidence and a versioned ruleset (invariant 15). That claim is worthless unless a failure to rebuild is **detected**. An unfalsifiable guarantee is not a guarantee.

The Cerevra audit is the direct motivation. Cerevra has a genuinely good deterministic replay engine (`cerevra/wal/replay.py:146-306`) — pure fold functions producing a manifest — and **stores no projection hash whatsoever**. A parser change, a code change, or a subtle ordering dependence silently produces a different manifest with no signal at all. The mechanism is sound; the verification is absent.

Cerevra also has no assertion of sequence monotonicity (`replay.py:149` uses `max(...)`, so duplicate sequences collapse silently) and no test that replaying the same WAL twice yields an identical result.

## Decision

**Every projection rebuild computes a canonical hash and compares it to a stored checkpoint.**

```sql
projection_checkpoints (
  projection_name, partition_key,
  last_applied_sequence, projection_hash NOT NULL,
  ruleset_version, computed_at
)
```

- Rebuilding at the same `(sequence, ruleset_version, parser_version, normalisation_version, geocoder_version, matching_version)` **must** produce the same hash. A mismatch increments `replay_divergence_total` and **pages immediately**.
- A rebuild under a *new* version tuple is expected to differ and is recorded as a versioned migration, not an alarm. The distinction is what makes the alert trustworthy rather than noisy.
- The hash is **canonical**: independent of row order, insertion order, and JSON key order. Property-tested.
- A nightly **shadow rebuild** of a sampled partition compares against live state, so divergence is found by the system rather than by a citizen.
- Sequence monotonicity is asserted, not maxed. A duplicate sequence raises.

`replay_divergence_total > 0` is the highest-priority alert in [observability-and-slos.md](../architecture/observability-and-slos.md). A non-zero value means the platform's central claim is currently false.

## Alternatives considered

| Alternative | Rejected because |
| --- | --- |
| No divergence detection (Cerevra's position) | Makes the reconstructability guarantee unfalsifiable |
| Compare full projections row by row | O(projection size) per check; a hash is O(1) to compare |
| Rely on tests only | Tests cover generated inputs; production data finds what tests do not |
| Detect on citizen complaint | Detection after public harm |
| Non-canonical hash (e.g. over a serialised dict) | Would produce false positives on key reordering, and the alert would be turned off within a month |

## Consequences

The reconstructability claim becomes **falsifiable and continuously tested**. Parser and ruleset upgrades are safe because their effects are visible and attributable. Incident response has a clear signal.

Cost: a hash computation per rebuild (negligible); a nightly shadow rebuild (bounded by sampling); and discipline in keeping the projector pure.

## Risks

- **A non-canonical hash producing false positives.** The fastest route to a disabled alert. Mitigated by a property test over row order, insertion order and key order.
- **Projector impurity** (a stray `now()`, a network call, map iteration order). Mitigated by lint, an injected forbidden-clock stub, and property tests.
- **Legitimate version changes flagged as divergence.** Mitigated by keying the checkpoint on the full version tuple.
- **Alert fatigue if it fires often.** If it fires often, the projector is broken and that is the correct signal. It should be permanently zero.

## Measurement

- `replay_divergence_total` — target **0**, absolute, at every scenario.
- `replay_duration_seconds` by scope (full / incremental).
- Shadow-rebuild coverage: fraction of partitions verified in the last 7 days.
- Property test: two independent folds of the same input produce identical hashes.

## Introduction trigger

n/a — day one. This is what distinguishes JanSetu's replay from an untested assertion, and it is the specific gap the Cerevra audit identified.

## Rollback strategy

None. Removing divergence detection would forfeit invariant 15.
