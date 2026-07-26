# Contributing to JanSetu

JanSetu is open civic infrastructure. Its usefulness scales with the number of cities covered, and that scales with contributors — most of whom will add a **source adapter or a parser fixture for their own city**.

That contribution is the most valuable one, and the project is built so it is also the easiest.

---

## Philosophy

JanSetu publishes information that citizens may act on during emergencies. A bug here does not corrupt a record; it sends someone into a flooded underpass. Everything below follows from that.

### Five rules that are not negotiable

**1. Unknown information stays explicit.**
Missing end time renders "unknown". Never an estimate, never a default, never an interpolation. If the system does not know, it says so.

**2. A community report can never become official confirmation.**
`OFFICIALLY_CONFIRMED` requires evidence from a civic node authorised for that jurisdiction and category. This is enforced by a database constraint, not only by rules. No amount of corroboration substitutes.

**3. An event never resolves through silence.**
`RESOLVED` requires positive evidence of resolution. Reports stopping means `STATUS_UNKNOWN`, not "fixed".

**4. Search and LLMs cannot determine event truth.**
An embedding similarity score is not evidence. An LLM may extract *candidate* event types, place names, time windows and summaries. It may never publish, confirm, resolve, invent an expiry, or override a rule.

**5. Parsers must be deterministic.**
A parser is a pure function of its input. No database access, no network, no clock. This is what makes replay reproducible, and it is enforced by the interface and by lint.

If a change appears to require breaking one of these, it needs an ADR — not a workaround.

---

## Before you start

- **No private or employer code.** Everything contributed must be yours to license under this project's terms, or already public. If a pattern came from proprietary work, re-derive it from public sources.
- **Small, reviewable pull requests.** One concern per PR. A 3,000-line PR touching six modules will sit unreviewed.
- **Architecture changes require an ADR.** See [ARCHITECTURE_STATUS.md](ARCHITECTURE_STATUS.md) for the list of changes that qualify.
- **Tests are required.** A PR touching an invariant must say which invariant and how it is tested.

---

## Contribution areas

| Area | Difficulty | What you need to know |
| --- | --- | --- |
| **Parser fixtures** | Very low | How to read a civic notice |
| **Source adapters** | Low | Your city, its language, its notice formats |
| Localisation | Very low | An Indian language |
| Accessibility | Low–medium | WCAG |
| Observability dashboards | Low | Grafana, PromQL |
| Notification adapters | Medium | The provider's API |
| Citizen web UI | Medium | Next.js, PWA |
| Geospatial utilities | Medium | PostGIS |
| Publisher / moderation consoles | Medium | React, the civic workflow |
| Event protocol contracts | High | Distributed systems, versioning |
| Node SDK | High | Cryptography, protocol design |
| **Trust rules / state machine** | Highest | Every invariant — *maintainer review required* |
| **Projection engine** | Highest | Determinism, replay — *maintainer review required* |

The last two are where the invariants live. A well-meaning change there can make the platform lie, so they carry a `CODEOWNERS` entry and require an ADR reference.

Look for `good-first-issue`. Adapters, fixtures, localisation and accessibility are the best entry points.

---

## Adding a city or source adapter

The highest-value contribution, and the one designed to need the least setup.

**1. Find the source.** A municipal notices page, a traffic-police advisory feed, a utility interruption notice, a transit alert feed.

**2. Capture fixtures.** Save real documents as fixtures under `fixtures/documents/`, with the expected extraction alongside. Anonymise any incidental personal data before committing.

**3. Write the adapter.**

```python
class SourceAdapter(Protocol):
    adapter_type: str
    version: str
    def discover(self, config: dict) -> list[DiscoveredDocument]: ...
    def fetch(self, doc: DiscoveredDocument) -> RawDocument: ...
```

**4. Write the parser.** A pure function. No database, no network, no clock.

```python
class Parser(Protocol):
    parser_version: str
    def parse(self, raw: RawDocument) -> list[EvidenceCandidate]: ...
```

**5. Include a negative fixture.** At least one document that should produce **no** event. "This notice produces nothing" is a normal, testable outcome — not an error — and adapters that never decline are usually wrong.

**6. Run the fast tests.** No Docker, no network:

```bash
make test-unit
make test-property
```

If either tier ever requires a container to run, that is a bug — please report it.

---

## Working with append-only data

Several tables reject `UPDATE` and `DELETE` at the database level: evidence, event transitions, parser executions, document versions, audit entries.

This surprises most new contributors. It is intentional. A correction is a **new row**, never an edit — that is what makes an event's history reconstructable and a correction distinguishable from tampering.

If a trigger raises on your write, the design intends you to append.

---

## Pull requests

Fill in the template. It asks for problem, scope, non-goals, invariants affected, tests, privacy and security considerations, and limitations.

The **limitations** field is not a formality. A PR that states what it does not handle is easier to accept than one that implies completeness.

### Checks that must pass

- `ruff`, `mypy --strict` on domain modules
- `import-linter` — module boundaries and forbidden edges
- Unit and property tiers
- Integration tier
- Dead-code detection

Coverage is gated by **branch coverage on invariant-bearing modules** and by mutation testing on the trust ruleset — not by a global line-coverage number. Exercising unreachable code proves nothing.

---

## Reporting problems

- **Bugs and features:** open an issue using the templates.
- **Security vulnerabilities:** do **not** open a public issue. See [SECURITY.md](SECURITY.md).
- **Wrong civic information in a deployed instance:** report through that instance's correction path, not here.

---

## Licence

Contributions are made under the repository's licence. By opening a pull request you confirm you have the right to license the work under those terms.
