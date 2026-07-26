# Architecture Status

```
Status:               Frozen for PMVP implementation
Baseline:             architecture-v0.1
Implementation target: Verified Event Vertical Slice
```

---

## What exists today

JanSetu is at the **architecture baseline** stage. The repository currently contains design documents and decision records. There is no running system.

### Implemented

| Artifact | Location |
| --- | --- |
| Product README | [README.md](README.md) |
| Architecture baseline | [docs/architecture/](docs/architecture/) |
| Accepted ADRs | [docs/adr/](docs/adr/) |
| Threat model | [docs/architecture/threat-model.md](docs/architecture/threat-model.md) |
| Workload assumptions | [docs/architecture/scaling/workload-model.md](docs/architecture/scaling/workload-model.md) |
| Reliability invariants | [docs/architecture/reliability-invariants.md](docs/architecture/reliability-invariants.md) |
| Implementation roadmap | [docs/architecture/contribution-architecture.md](docs/architecture/contribution-architecture.md) |

### Not yet implemented

- Backend service
- Frontend / citizen PWA
- Database schema and migrations
- Civic-node federation
- Real source adapters
- Citizen reporting
- Production deployment
- **Measured scale of any kind**

> **All city, national and billion-record figures are architectural scenarios based on documented assumptions. They are not production claims.**

Every number in [workload-model.md](docs/architecture/scaling/workload-model.md) is a labelled assumption or a derivation from one. That document also lists the five specific findings that would falsify it. Nothing in this repository has been measured against a running system.

---

## Approved vertical slice

The next implementation milestone is **v0.1.0 — Verified Event Vertical Slice**. It is the narrowest path that exercises every load-bearing invariant end to end:

```
source fixture
  → durable ingestion command
    → immutable source document
      → document version
        → parser execution
          → evidence item
            → deterministic trust rules
              → append-only event transition
                → public-event projection
                  → citizen evidence page
```

Anything outside that path is out of scope for v0.1.0.

### Exit criteria

The slice is complete when:

1. A fixture source produces a public event with inspectable evidence and honest uncertainty.
2. Killing the worker at any pipeline point loses nothing and duplicates nothing.
3. A full projection rebuild reproduces an identical projection hash.
4. Replay divergence is detected and alerts.
5. `OFFICIALLY_CONFIRMED` is unreachable without official evidence — proven by property test *and* rejected by a database constraint.
6. Missing end times render "unknown"; nothing is estimated.
7. Unit and property tests run with no Docker and no network.

---

## Changes requiring a new ADR

The baseline is frozen. The following changes are **not** ordinary pull requests — each requires an ADR that is reviewed and accepted before implementation:

- Replacing PostgreSQL or PostGIS as the system of record
- Removing or weakening append-only evidence
- Changing official-confirmation requirements
- Introducing a message broker
- Introducing microservices
- Adding an external search engine
- Changing civic-node isolation
- Changing signing or federation semantics

Superseding an accepted ADR is done by writing a new one that references it, never by editing the original.

---

## Baseline history

| Version | Status | Scope |
| --- | --- | --- |
| architecture-v0.1 | Frozen | PMVP architecture, 16 accepted ADRs, invariants, threat model, scaling scenarios |
