# Contribution Architecture

JanSetu is open civic infrastructure. Its usefulness scales with the number of cities covered, and that scales with contributors — most of whom will contribute a **source adapter or a parser fixture for their own city**, not a distributed-systems change.

The architecture is shaped by that. The most valuable contribution must be the easiest one.

---

## 1. Hard constraints on contributor experience

1. **Unit and property tests run with no Docker, no network, no external service.** A contributor fixing a date-format bug in a Jabalpur municipal notice will not stand up Postgres, MinIO and Redis. Breaking this is a regression.
2. **Parser contributions require only a fixture and a parser function.** No database, no migration, no service knowledge.
3. **Every fixture carries its expected output and its expected projection hash**, so determinism is verifiable locally.
4. **Failure to publish is a normal, testable outcome**, not an error. Contributors must be able to assert "this notice produces no event" as readily as "this notice produces a closure".

---

## 2. Module boundaries and contribution surface

| Module | Difficulty | Domain knowledge needed | Independent? |
| --- | --- | --- | --- |
| **Source adapters** | Low | The city, the language, the notice format | **Yes** — the primary surface |
| **Parser fixtures** | Very low | Ability to read a notice | **Yes** |
| Localisation | Very low | An Indian language | **Yes** |
| Accessibility | Low–medium | WCAG | **Yes** |
| Citizen web UI | Medium | Next.js, PWA | Mostly |
| Geospatial utilities | Medium | PostGIS, GIS concepts | Mostly |
| Notification adapters | Medium | The provider's API | **Yes** |
| Observability dashboards | Low | Grafana, PromQL | **Yes** |
| Testing fixtures | Low | The domain | **Yes** |
| Publisher console | Medium | React, the civic workflow | Mostly |
| Moderation console | Medium | Moderation workflow | Mostly |
| Event protocol contracts | **High** | Distributed systems, versioning | No — maintainer review |
| Node SDK | High | Cryptography, protocol | No |
| Trust rules / state machine | **Highest** | Every invariant | **No — maintainer only** |
| Projection engine | Highest | Determinism, replay | **No — maintainer only** |

The two "maintainer only" modules are where invariants live. A well-meaning change to the trust ruleset can make the platform lie, so those files carry a `CODEOWNERS` entry and require an ADR reference in the PR.

---

## 3. Extension interfaces

### Source adapter

```python
class SourceAdapter(Protocol):
    adapter_type: str
    version: str
    def discover(self, config: dict) -> list[DiscoveredDocument]: ...
    def fetch(self, doc: DiscoveredDocument) -> RawDocument: ...
```

### Parser

```python
class Parser(Protocol):
    parser_version: str
    def parse(self, raw: RawDocument) -> list[EvidenceCandidate]: ...
```

**Parsers are pure functions.** No database access, no network, no clock. This makes them trivially testable and is what keeps replay deterministic (I12). Enforced by the interface and by lint.

### Notification channel

```python
class NotificationChannel(Protocol):
    channel: str
    def send(self, delivery: Delivery) -> DeliveryResult: ...
```

Idempotent on `delivery.id`; retries reuse the row.

---

## 4. Versioning policy

| Artifact | Policy |
| --- | --- |
| Public event API | SemVer. Additive within a major; breaking changes ship a new major with a deprecation window |
| Event protocol | Versioned envelope; nodes declare `protocol_version`; N-1 supported |
| Parsers | Every change is a **new version**. Historic evidence is never rewritten (I8) |
| Trust rulesets | Immutable versioned artifacts; shadow-rolled before activation |
| Adapters | SemVer; a breaking change to `discover`/`fetch` is a major |
| Database schema | Forward-only migrations; expand/contract for destructive changes |

---

## 5. The first 30 issues

Each: context · scope · non-goals · acceptance criteria · tests · dependencies · labels · difficulty (S/M/L/XL).

### PMVP foundation

**#1 — Repository scaffold and CI**
Context: greenfield repo, docs only. Scope: FastAPI skeleton, module directories per ADR 0001, `import-linter` config with forbidden edges, ruff, mypy, pytest tiers, GitHub Actions. Non-goals: any domain logic. Acceptance: `make test` runs tiers 1–2 with no Docker; CI fails on a forbidden import. Tests: a deliberate forbidden import fails CI. Deps: none. Labels: `pmvp`, `infra`, `good-first-issue`. **M**

**#2 — Docker Compose dev environment**
Scope: Postgres 16 + PostGIS, MinIO, Redis; `make dev`, `make seed`, `make reset`. Non-goals: production deployment. Acceptance: one command to a working local stack; seed data present. Deps: #1. Labels: `pmvp`, `infra`, `good-first-issue`. **S**

**#3 — Core schema migrations (control plane)**
Scope: `organisations`, `jurisdictions` (with `ltree`), `civic_nodes`, `node_keys`, `sources`, `source_adapters` per [data-model.md](data-model.md). Non-goals: evidence tables. Acceptance: migrations apply and roll forward cleanly; `ltree` ancestor queries work. Tests: integration tests for every constraint. Deps: #2. Labels: `pmvp`, `database`. **M**

**#4 — Evidence and event schema migrations**
Scope: `ingestion_commands`, `ingestion_attempts`, `source_documents`, `source_document_versions`, `parser_executions`, `evidence_items`, `public_events`, `event_transitions`, `event_evidence_links`, `event_locations`, `projection_checkpoints`. Includes append-only `REVOKE` + triggers and the `official_evidence_count` CHECK. Acceptance: `UPDATE`/`DELETE` on append-only tables raise as the app role; the I3 CHECK rejects a bad insert. Tests: constraint tests for I1, I3, I4, I6. Deps: #3. Labels: `pmvp`, `database`, `invariant`. **L**

**#5 — Identity and OIDC**
Scope: user model, OIDC login, sessions in Postgres, principal resolution. Non-goals: MFA, RBAC. Acceptance: login works; sessions survive a Redis flush. Deps: #3. Labels: `pmvp`, `security`. **M**

**#6 — Authorisation decision function**
Scope: atomic permissions, jurisdiction-scoped `decide(principal, permission, scope)` using `ltree` ancestors, deny reasons logged. Non-goals: caching (ADR 0010 of [migration-triggers.md](scaling/migration-triggers.md) defers it). Acceptance: out-of-jurisdiction and out-of-category actions denied with a reason. Tests: unit table over the permission matrix. Deps: #3, #5. Labels: `pmvp`, `security`, `invariant`. **M**

**#7 — Row-level security policies**
Scope: RLS on all node-scoped and user-scoped tables; per-request principal set and reset. Acceptance: node A cannot read or write node B's rows **at the database**. Tests: RLS integration tests incl. connection-pool reuse. Deps: #4, #6. Labels: `pmvp`, `security`, `invariant`. **M**

### Cerevra-informed ingestion

**#8 — Ingestion command table and lease-based worker**
Scope: insert with `ON CONFLICT (idempotency_key)`, claim via `FOR UPDATE SKIP LOCKED`, lease, terminal states, attempts as append-only rows. Non-goals: adapters. Acceptance: N concurrent identical commands → one row, one effect; expired lease is re-claimed. Tests: concurrency + `SIGKILL` at 5 points (I2, I11). Deps: #4. Labels: `pmvp`, `ingestion`, `invariant`. **L**

**#9 — Object storage with content-addressed keys**
Scope: S3 client, `{node}/{source}/{sha256}` keys, bucket versioning + object lock, digest verify on read, orphan sweeper. Acceptance: overwrite denied; digest mismatch alerts; object written and verified before the referencing commit. Tests: chaos kill between PUT and COMMIT → no dangling reference (I7). Deps: #8. Labels: `pmvp`, `ingestion`, `invariant`. **M**

**#10 — Document identity, versioning and the five dedup keys**
Scope: `stable_locator`, full-width SHA-256, revision chain via `previous_version_id`, and the five distinct duplicate classes from [cerevra-integration.md](cerevra-integration.md) §5. Non-goals: parsing. Acceptance: identical bytes → no new version; changed bytes → linked new version; **no truncated hash anywhere**. Tests: collision test, concurrent-identical-ingest test (I1). Deps: #9. Labels: `pmvp`, `ingestion`, `invariant`. **M**

**#11 — Circuit breaker and backoff with jitter**
Scope: per-source breaker, exponential backoff with full jitter, capped attempts, dead-letter, `source_health` updates. Acceptance: source 503 → breaker opens → `source_health` degrades. Tests: chaos slow/failing/malformed source (I15). Deps: #8. Labels: `pmvp`, `ingestion`, `reliability`. **M**

**#12 — Source adapter SDK and the first three adapters**
Scope: the `SourceAdapter` protocol, a fixture harness, and three real adapters for the pilot city. Acceptance: adapters run against recorded fixtures with no network. Tests: fixture-driven. Deps: #10. Labels: `pmvp`, `adapters`, `good-first-issue`. **L**

**#13 — Parser framework and evidence extraction**
Scope: pure `Parser` protocol, `parser_executions`, `evidence_items` with the unique key, versioning. Non-goals: LLM extraction. Acceptance: parsers have no DB/network/clock access (lint-enforced); re-parse at the same version is a no-op. Tests: parser-upgrade test (I8). Deps: #10. Labels: `pmvp`, `ingestion`, `invariant`. **L**

### Trust and evidence

**#14 — Trust ruleset engine (versioned, deterministic)**
Scope: immutable versioned rulesets, pure `evaluate(projection, evidence) → decision`, decision tables per [pmvp.md](pmvp.md) §6. Non-goals: LLM anything. Acceptance: no input lacking official evidence reaches `OFFICIALLY_CONFIRMED`; no input reaches `RESOLVED` without resolution evidence. Tests: exhaustive unit table + property tests (I3, I4). Deps: #4. Labels: `pmvp`, `trust`, `invariant`, `maintainer-only`. **XL**

**#15 — Projection worker with checkpoints and hash**
Scope: pure fold, `projection_checkpoints`, canonical projection hash, incremental and full rebuild, advisory lock. Acceptance: two independent folds yield identical hashes; rebuild from checkpoint == full rebuild. Tests: property test on hash canonicality (I12). Deps: #14. Labels: `pmvp`, `trust`, `invariant`, `maintainer-only`. **XL**

**#16 — Replay divergence detection and alerting**
Scope: hash comparison, `replay_divergence_total`, nightly shadow rebuild of a sampled partition, sequence-monotonicity assertion. Acceptance: a deliberately perturbed rule is detected and attributed. Tests: divergence-injection test (I13). Deps: #15. Labels: `pmvp`, `trust`, `invariant`. **M**

**#17 — Event matching, merge and split**
Scope: event-match key, `event_merges`, `event_splits`, redirect resolution for archived ids. Acceptance: both original ids resolve forever; evidence links conserved on merge, partitioned exactly once on split. Tests: property tests (I18, I19). Deps: #15. Labels: `pmvp`, `trust`, `invariant`. **L**

### Geospatial

**#18 — Jurisdiction and location reference data import**
Scope: import boundaries and road segments for the pilot city; place aliases including transliterations. Acceptance: containment resolves correctly; alias search matches script variants. Tests: unit tests on transliteration pairs. Deps: #3. Labels: `pmvp`, `geospatial`, `good-first-issue`. **M**

**#19 — Route × event impact engine**
Scope: `ST_DWithin` intersection, impact classification, per-transition computation, `route_impact_assessments` in an isolated schema. Acceptance: correct classification on fixture geometry; **no public endpoint can reach route tables**. Tests: static + runtime isolation tests (I17). Deps: #18, #15. Labels: `pmvp`, `geospatial`, `privacy`, `invariant`. **L**

**#20 — Geometry validation and correction history**
Scope: reject invalid geometry on write (never silently repair); bitemporal `event_locations` corrections. Acceptance: invalid input rejected with a clear error; corrections append. Tests: property test over invalid geometries. Deps: #18. Labels: `pmvp`, `geospatial`. **S**

### Frontend

**#21 — Citizen PWA: event list and detail**
Scope: Next.js PWA, map view, event detail with evidence, freshness, source health, provenance. Acceptance: every event renders freshness; missing end times render "unknown"; no fabricated values anywhere. Tests: snapshot tests for `STALE`, `UNVERIFIED`, `STATUS_UNKNOWN`, unknown end time (I5, I16). Deps: #15. Labels: `pmvp`, `frontend`. **XL**

**#22 — Verification-status design tokens with a11y**
Scope: colour and iconography scale for the 15 event states, contrast-gated in CI. Acceptance: **states are distinguishable in greyscale and under all common colour-blindness simulations** — never by hue alone. Tests: automated contrast and simulation checks. Deps: #21. Labels: `pmvp`, `frontend`, `accessibility`, `good-first-issue`. **M**

**#23 — Community report submission**
Scope: report form, coarse-cell capture, media upload to the erasable bucket, EXIF strip, rate limits. Acceptance: precise location never leaves the device beyond the coarse cell for public display; EXIF GPS discarded, not stored. Tests: metadata-strip tests; dedup-key tests. Deps: #21. Labels: `mvp`, `frontend`, `privacy`. **L**

**#24 — Publisher console**
Scope: node-scoped event publishing, jurisdiction/category enforcement in the UI, evidence attachment. Acceptance: a publisher cannot select an out-of-scope jurisdiction or category. Deps: #6, #21. Labels: `mvp`, `frontend`. **L**

**#25 — Localisation framework**
Scope: i18n scaffolding, English + Marathi + Hindi, script-aware layout. Acceptance: full UI translatable; no hardcoded strings. Deps: #21. Labels: `mvp`, `frontend`, `good-first-issue`. **M**

### Testing and reliability

**#26 — Chaos test harness**
Scope: worker `SIGKILL`, Postgres restart, S3 timeout, **standing mid-suite Redis `FLUSHALL`**, slow/malformed source, duplicate flood. Acceptance: the full suite passes with Redis flushed mid-run (I10); every crash point loses nothing and duplicates nothing. Deps: #8, #15. Labels: `pmvp`, `testing`, `invariant`. **L**

**#27 — Property-based test suite**
Scope: Hypothesis strategies for the ten properties in [test-and-chaos-strategy.md](test-and-chaos-strategy.md) tier 2. Acceptance: all ten pass; each maps to a named invariant. Deps: #14, #15, #17. Labels: `pmvp`, `testing`, `invariant`. **L**

**#28 — Observability: metrics, traces, dashboards**
Scope: `prometheus_client` metrics per [observability-and-slos.md](observability-and-slos.md), OTel tracing **across the worker boundary**, the six Grafana dashboards, alert rules. Acceptance: one trace spans discovery → delivery; `replay_divergence_total` pages. Deps: #15. Labels: `pmvp`, `observability`. **L**

### Security

**#29 — Node key registration and envelope signing**
Scope: `node_keys`, canonical envelope serialisation, sign/verify, rotation with overlapping validity, revocation. Non-goals: the cross-node federation protocol (V1). Acceptance: a single-bit modification fails verification; revocation takes effect within the TTL. Tests: tamper tests, cross-implementation vectors (I20). Deps: #3. Labels: `mvp`, `security`, `federation`. **L**

### Documentation and onboarding

**#30 — Contributor guide and "add your city" walkthrough**
Scope: `CONTRIBUTING.md`, an end-to-end walkthrough for adding a source adapter for a new city, fixture authoring guide, invariant primer explaining why append-only tables reject `UPDATE`. Acceptance: a new contributor can add an adapter with fixtures without running Docker. Deps: #12. Labels: `docs`, `good-first-issue`. **M**

---

## 6. Grouping

| Group | Issues |
| --- | --- |
| PMVP foundation | 1, 2, 3, 4, 5, 6, 7 |
| Cerevra-informed ingestion | 8, 9, 10, 11, 12, 13 |
| Geospatial | 18, 19, 20 |
| Trust and evidence | 14, 15, 16, 17 |
| Frontend | 21, 22, 23, 24, 25 |
| Testing | 26, 27 |
| Security | 29 |
| Documentation | 30 |
| Open-source onboarding | 12, 18, 22, 25, 30 |

## 7. Suggested labels

`good-first-issue` · `pmvp` · `mvp` · `v1` · `invariant` · `maintainer-only` · `ingestion` · `trust` · `geospatial` · `frontend` · `security` · `privacy` · `federation` · `observability` · `testing` · `database` · `adapters` · `accessibility` · `docs` · `infra`

**`invariant`** marks anything protecting a rule in [reliability-invariants.md](reliability-invariants.md) — such a PR must state which invariant it protects and how it is tested.
**`maintainer-only`** marks the trust ruleset and projector, where a well-meaning change can make the platform lie.

## 8. Critical path

```
1 → 2 → 3 → 4 → 8 → 9 → 10 → 13 → 14 → 15 → 16 → 21
```

Everything else parallelises. Issues 12, 18, 22, 25 and 30 are contributor-facing and can start as soon as their dependencies land.
