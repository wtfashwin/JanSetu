# 0012 — Federation via signed event envelopes

Status: Accepted
Date: 2026-07-27

## Context

JanSetu's long-term shape is independently-operated civic nodes publishing within their authorised jurisdiction and category, consumed through a common protocol. Product invariant 11: nodes cannot silently rewrite one another's signed events.

Threats: a fake official node (T1), a compromised node key (T2), replay of a stale envelope (T5), cross-node mutation (T12).

## Decision

**Asymmetric signatures over canonically-serialised event envelopes.**

Envelope: node identity, jurisdiction path, category, protocol version, event payload, evidence references, issued time, nonce, update/expiry policy — signed with the node's private key.

- **JanSetu never holds a node's private key.** Nodes sign in their own environment; JanSetu stores only public keys. JanSetu therefore cannot forge a node's events even if JanSetu itself is compromised — the property that makes the federation claim meaningful.
- **Envelope bytes stored verbatim and immutable.** Verification on ingest **and again on read**, which is what detects tampering at rest (I20).
- Canonicalisation is fixed and versioned (deterministic key order, UTC, no floats). A signature that cannot be reproduced byte-for-byte is not a signature.
- Replay protection: nonce uniqueness within an issued-time window (±5 min skew tolerance).
- Key rotation with overlapping validity; **revocation pushed immediately over the event stream**, with a deliberately short 5-minute signing-key policy TTL so that even a stream outage bounds exposure.
- Trust levels (`AUTHORITATIVE` / `RECOGNISED` / `OBSERVER` / `SUSPENDED`) gate what a node's events can achieve; only an `AUTHORITATIVE` node within its jurisdiction and category can produce `OFFICIALLY_CONFIRMED`.
- A node may publish supporting evidence, contradictory evidence, a correction request, a derived local event, or a status note — **never a mutation**.
- Conflict is **represented**, not resolved. Citizens see both claims, both sources, both timestamps.
- Node suspension halts new envelopes; **previously published events remain with signatures intact and the suspension visible.** Deleting a suspended node's history would itself be silent rewriting.

## Alternatives considered

| Alternative | Rejected because |
| --- | --- |
| Blockchain / DLT | Addresses no threat that signatures plus append-only audit do not. Would record a false event immutably rather than prevent it. Very large cost and operational burden. See below |
| Bearer tokens per node | Cannot prove *origin* of a stored event after the fact; a token leak is indistinguishable from legitimate use |
| Centralised publishing (no federation) | Contradicts the product's purpose |
| Platform-adjudicated conflict resolution | Deciding which government department is right is not a power JanSetu should hold |
| mTLS only | Authenticates the transport, not the artifact. A stored event needs a signature that survives it |

## Consequences

Origin is provable after the fact. Cross-node mutation is impossible at three layers (signature, immutable storage, RLS). Conflicts become visible information rather than a silent overwrite. A compromised key has a bounded, revocable blast radius.

Cost: key management is a real operational burden for civic node operators, many of whom are municipal IT departments. The node SDK must make correct signing the path of least resistance.

## Risks

- **A legitimately-authorised node publishing a false event.** Cryptography cannot prevent this. Controls are visible provenance, contradiction as a first-class state, moderation and suspension — all after the fact. **This is the residual risk the design accepts, and it is stated rather than obscured.**
- **Key management burden on small civic bodies.** Mitigated by SDK support and a managed-signing option for nodes that cannot run a KMS — which weakens the guarantee for those nodes and must be labelled as such.
- **Canonicalisation drift** breaking verification. Mitigated by versioning the canonical form and by cross-implementation test vectors.

## Measurement

- `federation_signature_failures_total` by origin node and reason — any non-zero value pages.
- Key age distribution; time from revocation to effective across the fleet.
- Verification p99 latency (< 50 ms target).

## Introduction trigger

Key registration and envelope signing land at PMVP (they are cheap and retrofitting identity is expensive). The **federation protocol** — cross-node exchange — lands at V1, when there are at least two real nodes to federate.

## Rollback strategy

Federation ingest can be disabled without affecting locally-published events. Signing is additive and does not need removal.

## Note on blockchain

Revisit only if a threat is identified that signatures plus append-only history demonstrably cannot address. The residual risks in [threat-model.md](../architecture/threat-model.md) (a captured authority, a poisoned source, a patient brigade) are social and organisational; a ledger would record them immutably rather than prevent them.
