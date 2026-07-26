# Security and Federation

Companion: [threat-model.md](threat-model.md). Data classifications: [data-model.md](data-model.md). Invariants referenced as I1–I24 from [reliability-invariants.md](reliability-invariants.md).

Guiding position: JanSetu is public infrastructure that citizens may act on during emergencies. Its security failures are not data-breach failures — they are **people making wrong decisions during a flood**. The controls below are ordered by that consequence, not by conventional severity.

---

## 1. Authentication

| Principal | Mechanism | Notes |
| --- | --- | --- |
| Citizen | OIDC (Google, phone-OTP) → session in **Postgres**, not Redis | Session loss on cache failure is unacceptable (I10) |
| Publisher operator | OIDC with mandatory MFA | Publishes on behalf of a civic node |
| Moderator | OIDC + MFA + explicit grant | |
| Maintainer / admin | OIDC + MFA + hardware key at city scale and beyond | |
| Civic node (machine) | Request signing with the node's registered key | Not a bearer token — see §4 |
| API consumer (navigation, delivery, research) | API key with scoped read grants + rate tier | Read-only by construction |

Anonymous read access to public events is permitted and expected — this is public information and requiring login to see a road closure would defeat the product.

---

## 2. Authorisation

Atomic permissions, evaluated by one decision function. No handler inspects a role name.

```
decide(principal, permission, scope) → ALLOW | DENY(reason)
```

Permissions: `event.publish`, `event.confirm`, `event.resolve`, `event.dispute`, `report.moderate`, `report.view_precise_location`, `source.configure`, `node.register`, `node.suspend`, `key.rotate`.

**Scope is territorial and categorical.** A traffic-police node holds `event.publish` and `event.confirm` for road categories inside its own jurisdiction subtree and nothing else. Scope checks resolve against the `jurisdictions.path` `ltree`, so "all of Pune" is an ancestor query, not an enumeration.

Enforcement is layered on purpose:

1. **Decision function** — application-level, with a logged deny reason.
2. **Postgres RLS** on `civic_node_id` / `user_id` — stops a bad write even when the application is wrong (I14, I17).
3. **Runtime assertions** that raise on any cross-node or cross-jurisdiction row observed on a read path — adopted from `cerevra/query/invariants.py:39-77`, which is genuinely good practice.

Assertions alone are insufficient (they only see reads); RLS alone is insufficient (it cannot express category scope). Both are needed.

**No effective-permission cache at PMVP.** Caching authorisation creates a revocation-latency window, and cutting off a compromised node immediately matters more than saving microseconds when there are ten publishers.

---

## 3. Secrets and keys

- Application secrets in a managed secret store (cloud KMS-backed), injected at runtime. Never in the repository, never in the image, never in a database column.
- **JanSetu never holds a civic node's private signing key.** Nodes sign in their own environment; JanSetu stores only public keys (`node_keys`). This is deliberate: JanSetu cannot forge a node's events even if JanSetu itself is compromised, which is the property that makes the federation claim meaningful.
- Database credentials rotated on a schedule; separate roles for `api`, `worker`, `projector`, `analytics` with least-privilege grants (the projector is the only role with `UPDATE` on `public_events`).
- TLS everywhere. mTLS between services once there is more than one deployable and a mandate for it; not before.

---

## 4. Civic node signing

### Envelope

```
{
  protocol_version, node_slug, node_id, jurisdiction_path,
  category, event_payload, evidence_references[],
  issued_at, nonce, update_policy, expiry_policy
}
+ signature over a canonical serialisation, key_id
```

Canonicalisation is fixed and versioned (deterministic key ordering, UTC timestamps, no floats). A signature that cannot be reproduced byte-for-byte is not a signature.

### Verification

- On ingest **and again on read** (I20). Read-time verification is what detects tampering with data at rest.
- `envelope_bytes` stored verbatim and immutable; local commentary lives in a separate annotation table and cannot touch the envelope.
- Replay protection: nonce uniqueness within an issued-time window (±5 minutes clock skew tolerance).
- Failure → quarantine, `federation_signature_failures_total` increments, page.

### Key lifecycle

| Operation | Mechanism |
| --- | --- |
| Registration | Out-of-band verification of the organisation, then key upload by an authenticated operator |
| Rotation | Overlapping validity windows; both keys verify during the overlap |
| Revocation | `revoked_at` set; **pushed immediately over the event stream**, not left to cache expiry. Signing-key policy TTL is deliberately short (5 min) so that even a stream outage bounds exposure |
| Compromise | Revoke → suspend node → re-verify every event signed by that key within the exposure window → publish corrections for any that fail |
| Suspension | New envelopes rejected. **Previously published events remain, with signatures intact and the suspension visible.** Retroactively deleting a suspended node's history would itself be a form of silent rewriting |

### Trust levels

| Level | Can achieve |
| --- | --- |
| `AUTHORITATIVE` | `OFFICIALLY_CONFIRMED` within its jurisdiction and categories |
| `RECOGNISED` | `COMMUNITY_CORROBORATED`; contributes corroboration weight |
| `OBSERVER` | Publishes evidence only; cannot advance state |
| `SUSPENDED` | Nothing new; history preserved |

A node's trust level cannot exceed its jurisdiction and category grant. Cryptography proves a node *said* something; it does not prove the node is honest. Trust levels, visible provenance, and suspension are the real controls — signatures are only the transport integrity layer.

---

## 5. Federation rules

One node **cannot** mutate another's event. It may publish:

- supporting evidence,
- contradictory evidence (which produces a visible `CONFLICTING` state, not a silent overwrite),
- a correction request (which the origin node may accept),
- a derived local event referencing the original,
- a status note.

Conflict is **represented, not resolved by the platform.** A citizen sees both claims, both sources, and both timestamps. Automatically deciding which government department is right is not a power JanSetu should hold.

### Why not blockchain

Every federation threat in [threat-model.md](threat-model.md) is addressed by: asymmetric signatures (authenticity), immutable verbatim envelope storage (integrity), an append-only audit log (history), and out-of-band identity verification (node legitimacy). A distributed ledger adds substantial cost and operational burden and solves none of the residual problems — which are **social** (is this node honest?) rather than cryptographic. Revisit only if a threat is identified that signatures demonstrably cannot address.

---

## 6. Privacy

### What is deliberately not collected

No health data, no caste, no religion, no political affiliation, no continuous location history, no contact graph, no device identifiers beyond a push token. This is a product boundary. A civic-information platform that accumulated movement patterns would be a surveillance asset regardless of intent, and its existence would be the risk.

### Reporter protection

- Reports are pseudonymous. The pseudonym→identity mapping is a separate, independently-erasable table.
- Public display uses a **coarse H3 cell (~0.7 km²)** only. Precise geometry is moderator-visible and never leaves the system.
- No public reporter profiles, no reputation leaderboards, no report history exposed by reporter.
- Rate limits and abuse scoring operate on the pseudonym, never on identity.

### Saved routes

The most sensitive data JanSetu holds — a saved commute is a movement pattern.

- Separate schema; **no grant** to the public-API role (I17).
- RLS on `user_id`; column-level encryption on geometry.
- Impact assessments computed per user, returned only to that user, retained 30 days.
- Never referenced by an event, never aggregated into public statistics without k-anonymity and a documented minimum k.
- Hard-deleted on account deletion.

### Erasure

Deleted: routes, subscriptions, impact assessments, notification history, the pseudonym→identity mapping.
Retained: community reports, pseudonymously. The civic evidence survives; the person does not.

This resolves the immutability-versus-erasure tension explicitly, rather than leaving it implicit as `cerevra/compliance/deletion.py` does.

---

## 7. Media handling

Community-report photos are the largest untrusted-input surface.

| Control | Detail |
| --- | --- |
| Location | **Separate erasable bucket.** Never the object-locked raw-document bucket — this is what keeps I7 (immutable raw documents) compatible with erasure |
| Metadata | EXIF stripped on upload, including GPS. The stripped GPS is discarded, not stored |
| Type | Magic-byte verification; declared content-type never trusted |
| Size | Hard cap; images re-encoded through a decoder to normalise |
| Malware | Scanning before the object becomes referencable |
| Serving | From a separate origin with a restrictive CSP; never served from the API origin |
| Moderation | Human review before public display for any report that could advance an event state |

---

## 8. Abuse prevention

| Vector | Control |
| --- | --- |
| Report brigading | Independence requirements — distinct pseudonyms **and** distinct coarse cells **and** a time window. N reports from one cell by one reporter count once (`community_reports.dedup_key`) |
| Fake official node | Out-of-band organisational verification before registration; jurisdiction and category grants bound what a node can claim even if registration is subverted |
| Location spoofing | Coarse cells only; corroboration requires spatial diversity; device attestation considered but **not** relied on |
| Notification spam | Per-user, per-event, per-day caps (I9, I23); severity thresholds; quiet hours |
| Scraping | Rate limits; no bulk endpoint for reports; route data unreachable by construction |
| DoS during disaster | Static/CDN fallback for the highest-severity active events, servable with the database down. **The read path degrades to a cached list of confirmed events rather than to an error page** |
| Parser exploitation | Parsers run with restricted filesystem and no network; PDF/HTML libraries pinned and patched; malformed input yields a recorded failure, never an exception that halts a source |
| Source poisoning | Authority class is a property of the *registered source*, not of document content. A compromised source cannot elevate itself; it can only publish within its granted authority, and anomaly detection flags sudden volume or category shifts |

---

## 9. Disaster mode

An explicit, audited switch. It may raise report-intake limits, shed non-essential read features, reprioritise notification queues by severity, and extend cache TTLs on stable content.

It may **not** lower confirmation thresholds, auto-confirm, skip moderation, or relax signature verification (I24). There is no disaster variant of the trust ruleset — structurally, there is nothing to loosen.

This is the control most likely to be argued away under pressure ("people need to know *now*"). It is recorded here so that relaxing it requires an explicit reviewed decision, not an on-call judgement at 03:00.

---

## 10. Audit

Every state-changing operation writes an `audit_entries` row: actor, action, target, before/after, request id, timestamp. Append-only, `REVOKE UPDATE, DELETE`, 7-year retention, cold after 1 year.

Publicly inspectable audit for *event* state changes — every citizen can see why an event is in its current state and which rule version produced it. Personal and moderator-identity fields are excluded from the public view.

---

## 11. Residency

Regional clusters at national scale, with jurisdiction determining the storing region. Cross-region replication is for disaster recovery only and honours residency constraints. Federation exchanges signed envelopes across regions; personal data never crosses a residency boundary.

---

## 12. Known residual risks

Stated rather than hidden:

1. **A legitimately-authorised node publishing a false event.** Cryptography cannot prevent this. Controls are visible provenance, contradiction as a first-class state, moderation, and suspension — all after-the-fact. This is the residual risk the whole design accepts.
2. **Notification fan-out latency during a national emergency** (up to ~1 hour to drain 100 M at Scenario D). A product limitation to be disclosed, not concealed.
3. **Coarse cells are not perfect anonymity.** A report from a sparsely-populated cell narrows the reporter set. Mitigation: adaptive cell sizing by population density — designed, not yet specified in detail.
4. **Correlation across a user's subscriptions** could infer home and work locations even without saved routes. Mitigation: subscription data carries the same isolation as routes, and no endpoint exposes another user's subscriptions.
5. **Insider risk.** A JanSetu maintainer with database access could alter a projection. Mitigations: append-only evidence means the projection can be rebuilt and the tampering detected (I13); audit logging; least-privilege roles. Not eliminated.
