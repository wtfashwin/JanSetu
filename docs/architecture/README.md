# JanSetu Architecture

The architecture baseline for JanSetu — open, verifiable, location-aware civic-event infrastructure.

**Status:** frozen at `architecture-v0.1` for PMVP implementation. See [ARCHITECTURE_STATUS.md](../../ARCHITECTURE_STATUS.md).

> All city, national and billion-record figures in these documents are architectural scenarios based on documented assumptions. They are not production claims. Nothing here has been measured against a running system.

---

## Start here

Read in this order. Each document assumes the previous one.

1. **[overview.md](overview.md)** — what the architecture is and why, in fourteen answers.
2. **[pmvp.md](pmvp.md)** — the architecture being built now.
3. **[diagrams.md](diagrams.md)** — twenty diagrams, each with its purpose, invariant and failure mode.
4. **[reliability-invariants.md](reliability-invariants.md)** — the twenty-four rules the system must never break, with their enforcement mechanisms.
5. **[data-model.md](data-model.md)** — tables, keys, constraints, classifications, PII handling.
6. **[threat-model.md](threat-model.md)** — fourteen threats, ranked by civic consequence.
7. **[test-and-chaos-strategy.md](test-and-chaos-strategy.md)** — six test tiers and what each proves.
8. **[../adr/](../adr/)** — the sixteen accepted architecture decisions.

---

## All documents

### Core

| Document | Purpose |
| --- | --- |
| [overview.md](overview.md) | Executive summary; the fourteen questions the architecture answers |
| [pmvp.md](pmvp.md) | PMVP architecture — the shape being implemented |
| [diagrams.md](diagrams.md) | Twenty Mermaid diagrams |
| [data-model.md](data-model.md) | Relational model, identity strategy, retention, PII |
| [reliability-invariants.md](reliability-invariants.md) | I1–I24 with mechanism, constraint, test, metric, alert, recovery |
| [security-and-federation.md](security-and-federation.md) | AuthN/AuthZ, signing, privacy, abuse prevention |
| [threat-model.md](threat-model.md) | T1–T14, ranked by consequence |
| [observability-and-slos.md](observability-and-slos.md) | Metrics, traces, SLOs per scenario, alerts |
| [test-and-chaos-strategy.md](test-and-chaos-strategy.md) | Test pyramid, chaos scenarios, CI gating |
| [contribution-architecture.md](contribution-architecture.md) | Module boundaries, extension interfaces, first 30 issues |
| [pattern-applicability.md](pattern-applicability.md) | Which distributed-systems patterns apply, and when |

### Cerevra evaluation

[Cerevra](https://github.com/wtfashwin/cerevra) is a public, Apache-2.0 retrieval engine whose WAL and replay design informed this architecture.

| Document | Purpose |
| --- | --- |
| [cerevra-capability-audit.md](cerevra-capability-audit.md) | Evidence-based audit of what is proven, partial or unsafe |
| [cerevra-integration.md](cerevra-integration.md) | Per-capability mapping to JanSetu responsibilities |

### Future scale — `scaling/`

**Not PMVP.** These describe architectures that are introduced only when a documented, measurable trigger is crossed.

| Document | Purpose |
| --- | --- |
| [scaling/workload-model.md](scaling/workload-model.md) | Four scenarios with explicit assumptions and derivations |
| [scaling/city.md](scaling/city.md) | City-scale evolution |
| [scaling/national-federated.md](scaling/national-federated.md) | National / federated architecture |
| [scaling/migration-triggers.md](scaling/migration-triggers.md) | Measurable conditions for introducing — and removing — components |

### Reviews — `reviews/`

| Document | Purpose |
| --- | --- |
| [reviews/adversarial-review.md](reviews/adversarial-review.md) | Five reviews written to attack the design; what changed as a result |
| [reviews/quality-review.md](reviews/quality-review.md) | Sixteen categories scored, two revision passes, remaining gaps |

---

## Reading conventions

- **I1–I24** refer to invariants in [reliability-invariants.md](reliability-invariants.md).
- **T1–T14** refer to threats in [threat-model.md](threat-model.md).
- **P/C/N** in tables mean PMVP / city scale / national scale.
- Table classifications (`AUTHORITATIVE`, `APPEND_ONLY`, `DERIVED`, `CACHE`, `SEARCH_INDEX`, `ANALYTICAL`, `ARCHIVE`) are defined in [data-model.md](data-model.md).
- Where a guarantee is weaker than it first appears, the document says so. Absence of a caveat is meant to be informative.

## Changing the architecture

The baseline is frozen. Structural changes require an ADR — see [ARCHITECTURE_STATUS.md](../../ARCHITECTURE_STATUS.md) for the list of changes that qualify, and [../adr/TEMPLATE.md](../adr/TEMPLATE.md) for the format.
