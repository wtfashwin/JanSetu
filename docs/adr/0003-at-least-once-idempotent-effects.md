# 0003 — At-least-once delivery with idempotent effects

Status: Accepted
Date: 2026-07-27

## Context

Exactly-once processing does not exist across a network and a store. Claiming it produces systems that quietly lose or duplicate work while their documentation says otherwise.

JanSetu must guarantee that a retried ingestion command is safe (invariant 13), a worker crash cannot lose an acknowledged command (11), and the same source document cannot create duplicate effects (I1).

The Cerevra audit shows what happens without this discipline. The duplicate check at `cerevra/ingestion/handler.py:151-157` loads a manifest, tests membership, and appends — with no lock spanning the sequence, so two concurrent identical ingests both pass. Separately, `handler.py:411-449` dispatches embedding generation to a daemon thread with two in-memory retries and a `logger.error` on final failure: work is lost permanently, silently, with no durable record that it was ever attempted.

## Decision

```
At-least-once delivery
+ idempotent consumers
+ deduplication keys enforced by database UNIQUE constraints
+ transactional state changes (effect and acknowledgement in one commit)
+ append-only evidence
+ replayable projections
```

Concretely:

- Every command carries `idempotency_key`, unique-constrained.
- Effects and the command's terminal state commit in a single transaction. **Acknowledgement is the commit** — there is no separate ack step a crash can lose.
- Every derived row has a natural unique key; re-processing collides and is absorbed by `ON CONFLICT DO NOTHING`.
- Attempts are append-only rows, never a mutated counter.
- Claiming uses a lease; expiry is the entire recovery mechanism.
- **No fire-and-forget threads.** Every asynchronous effect is a durable row with a visible failure state.

## Alternatives considered

| Alternative | Rejected because |
| --- | --- |
| Claim exactly-once | Not achievable; claiming it is dishonest and leads to unsafe assumptions downstream |
| At-most-once | Loses civic information. Unacceptable |
| Distributed transactions (2PC) across S3 and Postgres | Operationally heavy; S3 does not participate; ordering plus a sweeper solves it |
| Background threads with in-memory retry | The exact pattern the audit found losing work silently |

## Consequences

Duplicates are absorbed structurally rather than avoided by hope. Recovery requires no bespoke engine — lease expiry suffices, so there is no separate recovery path to get wrong. Every effect is inspectable as a row.

Cost: every consumer must be written idempotently, and every effect needs a natural key. This is a discipline, and it is enforced in review and by property tests.

## Risks

- **A missing unique key** creates a duplication window. Mitigated: every table in [data-model.md](../architecture/data-model.md) declares its uniqueness, and the integration suite exercises every constraint.
- **Non-idempotent external side effects** (a sent SMS cannot be un-sent). Mitigated by the delivery unique key on `(subscription_id, event_id, transition_sequence)` — retries reuse the row (I9).
- **Poison commands.** Bounded attempts → `DEAD_LETTER`, surfaced in moderation.

## Measurement

- `ingestion_retries_total / ingestion_commands_total` — alert above 0.2 over 30 min.
- `wal_unacknowledged_total` — alert on monotonic rise for 15 min.
- Property test: N applications of a command produce the same state as one.
- Chaos test: `SIGKILL` at five pipeline points, exactly one observable effect each.

## Introduction trigger

n/a — day one.

## Rollback strategy

None. This is the correctness foundation.
