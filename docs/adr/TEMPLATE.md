# ADR Template

Copy this file to `NNNN-short-kebab-title.md`, using the next free number.

An accepted ADR is never edited to change its decision. To change a decision, write a new ADR that references and supersedes it, and set the old one's status to `Superseded by NNNN`. The record of what was believed, and when, is the point.

See [ARCHITECTURE_STATUS.md](../../ARCHITECTURE_STATUS.md) for the list of changes that require an ADR rather than an ordinary pull request.

---

```markdown
# NNNN — Title

Status: Proposed | Accepted | Superseded by NNNN
Date: YYYY-MM-DD

## Context

What forces are at work? What requirement, invariant, or measurement makes this
decision necessary now? Cite numbers where they exist, and label them as
assumptions where they are not measured.

## Decision

What is being decided, stated so a reader can act on it.

## Alternatives considered

| Alternative | Rejected because |
| --- | --- |

Include the simplest alternative, even — especially — when it was rejected.
An ADR with no rejected simpler option usually has not looked for one.

## Consequences

What becomes easier, and what becomes harder. Both.

## Risks

What could go wrong, and what mitigates it. State residual risk plainly rather
than implying it was eliminated.

## Measurement

How we will know whether this decision was right. Name the metric.

## Introduction trigger

When this takes effect. `n/a — day one` for foundational decisions; a
measurable condition for anything deferred. No round numbers without a
derivation.

## Rollback strategy

How to undo this. `None` is a valid answer for foundational decisions, but it
must be stated deliberately rather than left blank.
```
