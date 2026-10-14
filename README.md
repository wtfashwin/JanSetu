# JanSetu

JanSetu is an architecture project for helping citizens and public institutions share verifiable, location-aware civic updates in India.

## Current status

The repository contains product plans, architecture documents, and decision records. There is no running backend, frontend, database implementation, source adapter, federation service, or deployment.

The `architecture-v0.1` baseline is frozen for PMVP implementation. The approved next milestone is the **Verified Event Vertical Slice**: a source fixture becomes inspectable evidence, an event transition, and a citizen evidence page. See [Architecture Status](ARCHITECTURE_STATUS.md) for scope and acceptance criteria.

All city, national, and billion-record figures are design scenarios based on documented assumptions. No scale or runtime claims have been measured.

## What is being designed

- Civic nodes for public authorities, institutions, and communities.
- Updates about road restrictions, waterlogging, transport, utility interruptions, and public services.
- Location relevance, source provenance, freshness, and explicit uncertainty.
- Separate official evidence and community observations, with an inspectable event history.

Privacy and trust are design requirements: unknown information stays unknown, community reports cannot become official confirmation, silence cannot resolve an event, and search or language models cannot decide event truth. The proposed system excludes personal tracking, political profiling, facial recognition, and automated emergency authority.

## Start here

1. Read the [architecture index](docs/architecture/README.md) and [overview](docs/architecture/overview.md).
2. Review the [PMVP scope](docs/architecture/pmvp.md), [reliability rules](docs/architecture/reliability-invariants.md), and [threat model](docs/architecture/threat-model.md).
3. Read the [accepted decisions](docs/adr/) before proposing structural changes.
4. Use the [contribution architecture](docs/architecture/contribution-architecture.md) to find a bounded contribution.

The proposed stack includes PostgreSQL/PostGIS and immutable source storage. These are architecture decisions, not installed services. Setup and executable tests are not available yet.

## Contributing

Start with [CONTRIBUTING.md](CONTRIBUTING.md), the [Code of Conduct](CODE_OF_CONDUCT.md), and [security reporting](SECURITY.md). Useful early work includes public-source research, anonymized parser fixtures, local-language support, accessibility review, and implementation within the approved slice. Architecture changes require a reviewed ADR.

An issue should describe the problem, affected users and location, available evidence, proposed contribution, privacy/licensing concerns, and how it can be checked. Do not contribute private or employer code.

## License and attribution

The repository includes the [Apache License 2.0](LICENSE). Preserve map, dataset, and source attributions and check their own reuse terms.

The [capability audit](docs/architecture/cerevra-capability-audit.md) and [integration proposal](docs/architecture/cerevra-integration.md) document the external design influences and their limits.
