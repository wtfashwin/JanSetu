# 0007 — Public events as projections from evidence

Status: Accepted
Date: 2026-07-27

## Context

Product invariant 15 requires that a public event can be rebuilt from append-only evidence plus a versioned ruleset. A citizen must be able to ask "why is this `OFFICIALLY_CONFIRMED`?" and get an answer that is reproducible, not merely plausible.

Cerevra demonstrates the fold (`cerevra/wal/replay.py:146-306` — pure `_apply_*` functions of `(manifest, record)`) and simultaneously demonstrates what is missing: no stored projection hash, so a code change silently produces a different manifest with no signal.

## Decision

`public_events` is `DERIVED`, written **only** by the projection worker.

```
project(event):
  load checkpoint (last_applied_sequence, projection_hash, ruleset_version)
  for evidence with sequence > last_applied, ordered by (sequence, evidence_id):
      decision = ruleset[version].evaluate(projection, evidence)   # pure
      emit event_transitions rows
      projection = apply(projection, decision)
  UPSERT public_events
  store checkpoint (sequence, hash(projection), ruleset_version)
```

Determinism constraints, enforced by lint and by an injected forbidden-clock stub: no `now()` (evaluation timestamps are passed in and stored), no network calls, no map-iteration-order dependence, ties broken by `(sequence, evidence_id)`.

**LLM outputs are read from storage, never regenerated during replay.** This is what makes invariant 7 ("LLM extraction is never event truth") compatible with invariant 12 ("identical inputs produce identical projections").

Ruleset versioning: rules are immutable versioned artifacts; every transition records the version that produced it. A rebuild under a *new* ruleset is expected to differ and is recorded as a versioned migration, not an alarm.

Blue/green rollout: a new ruleset runs in shadow against a sampled partition; differences are reviewed before activation.

## Alternatives considered

| Alternative | Rejected because |
| --- | --- |
| Mutable event state updated in place | Cannot answer "why"; correction indistinguishable from tampering |
| Recompute on every read | O(history) per request; unbounded latency |
| Projection without checkpoints | Cerevra's model — replay time grows without bound (42 min at Scenario B, 6.9 h at C) |
| Projection without a hash | Divergence becomes undetectable — the specific gap this ADR closes |

## Consequences

Rebuild is always possible. Divergence is detectable. Parser and rule upgrades are safe by construction. Incremental rebuild is fast; full rebuild partitions by jurisdiction (13 min at 32-way for Scenario C).

Cost: the projector must be kept pure — an ongoing discipline, hence the lint rule.

## Risks

- **Accidental impurity** (a clock call, a lookup). Mitigated by lint, the stub, and property tests.
- **Projection lag rendered as truth.** Mitigated by `projection_lag_seconds` and `as_of` on every response.
- **A ruleset change silently altering history.** Mitigated by ruleset immutability plus shadow rollout.

## Measurement

`replay_divergence_total` (must be 0), `replay_duration_seconds`, `projection_lag_seconds`, and a property test asserting two independent folds produce identical hashes.

## Introduction trigger

n/a — day one. Checkpointing is required from Scenario B (full replay already 42 min).

## Rollback strategy

None. This is the product.
