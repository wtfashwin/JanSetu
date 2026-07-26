# Threat Model

Method: STRIDE-informed, but ordered by **civic consequence** rather than by CIA impact. A JanSetu compromise does not primarily leak data — it causes people to make wrong decisions during emergencies. Threats are ranked accordingly.

Controls reference [security-and-federation.md](security-and-federation.md) and invariants I1–I24 from [reliability-invariants.md](reliability-invariants.md).

---

## Assets, ranked by consequence of compromise

| Rank | Asset | Consequence if compromised |
| --- | --- | --- |
| 1 | Public event state and its trust label | Citizens act on false information during an emergency |
| 2 | Saved routes and movement patterns | Individual surveillance; physical safety risk |
| 3 | Reporter identity | Retaliation against people reporting infrastructure failures |
| 4 | Civic node signing keys | Impersonation of a government authority |
| 5 | Append-only evidence and audit history | The system's ability to prove anything is destroyed |
| 6 | Notification channel | Mass panic, or alert fatigue that disables the channel when it matters |
| 7 | Raw source documents | Evidence chain broken; disputes unresolvable |

---

## T1 — Fake official node

**Scenario.** An attacker registers a node claiming to be a municipal authority and publishes a fabricated closure or an "all clear" during a real flood.

**Impact.** Highest. Directly causes wrong decisions.

**Controls.**
- Out-of-band organisational verification before registration — a human process, not a form. There is no automated path to `AUTHORITATIVE` trust.
- Jurisdiction and category grants bound what a node can claim even if registration is subverted.
- `OFFICIALLY_CONFIRMED` requires an authorised node for that jurisdiction *and* category, enforced by DB constraint (I3).
- Every event displays its issuing node; citizens see who claimed what.
- Anomaly detection on new-node publishing volume and category drift.

**Residual.** A genuinely compromised or captured legitimate authority. Nothing technical prevents this; suspension is the response. **Accepted, and stated.**

---

## T2 — Compromised node key

**Scenario.** A legitimate node's private key is stolen and used to sign false events.

**Impact.** Highest — indistinguishable from legitimate publication until detected.

**Controls.**
- JanSetu never holds private keys; the blast radius is one node's own environment.
- Immediate revocation pushed over the event stream, not left to cache expiry.
- Signing-key policy TTL deliberately short (5 min), bounding exposure even during a stream outage. This is the one place the design chooses consistency over availability.
- On revocation: re-verify every event signed by that key within the exposure window; publish corrections for any that fail.
- Overlapping-validity rotation makes routine rotation cheap, so keys are actually rotated.

**Residual.** Events published between compromise and detection. Mitigated by correction, not prevented.

---

## T3 — Fake road closure (or fake all-clear)

**Scenario.** Fabricated event via community reports or a compromised source. The "all clear" variant is worse — it removes a warning.

**Impact.** Highest.

**Controls.**
- Single anonymous report → `UNVERIFIED`, visibly labelled (I3).
- Corroboration requires **distinct pseudonyms and distinct coarse cells** within a window — spatial diversity, not just count.
- Resolution requires positive evidence, never silence (I4). **A fake all-clear cannot be produced by suppressing reports.**
- Conflicting evidence produces a visible `CONFLICTING` state showing both claims.
- Moderation queue for ambiguous location or high-severity categories.

**Residual.** A coordinated, geographically-distributed campaign. See T4.

---

## T4 — Coordinated report brigading

**Scenario.** An organised group submits many reports from many accounts and locations to manufacture corroboration — for example, to divert traffic from a road, or to discredit the platform.

**Impact.** High.

**Controls.**
- Independence requirements (distinct pseudonym **and** cell **and** time window); duplicates collapse via `dedup_key`.
- Community corroboration can **never** reach `OFFICIALLY_CONFIRMED` (I3). Brigading has a hard ceiling.
- Account-age and history weighting; new accounts contribute less.
- Rate limiting per pseudonym and per cell.
- Anomaly detection on report velocity within a cell.
- Human moderation for high-severity categories before wide notification.

**Residual.** A patient, well-resourced adversary can reach `COMMUNITY_CORROBORATED`. The label itself is the mitigation — citizens see it is community-sourced, not official.

---

## T5 — Replay attack

**Scenario.** A previously-valid signed envelope is re-submitted to resurrect a stale event — for example, replaying last month's closure during rush hour.

**Impact.** Medium-high.

**Controls.** Nonce uniqueness within an issued-time window; issued-time freshness check (±5 min skew); envelope hash uniqueness constraint on `federated_events`; evidence freshness budgets mean a stale event decays to `STATUS_UNKNOWN` regardless (I23).

**Residual.** Low.

---

## T6 — Duplicate webhook / delivery flood

**Scenario.** A source or federation peer delivers the same payload thousands of times, whether by fault or by intent.

**Impact.** Medium — resource exhaustion, potential duplicate effects.

**Controls.** Idempotency key unique constraint on `ingestion_commands` (I2); content-addressed document identity (I1); per-source rate limits and circuit breakers; backpressure via bounded claim budget.

**Residual.** Low. This threat is largely designed out.

---

## T7 — Parser exploitation

**Scenario.** A malicious PDF or HTML document exploits a parsing library to achieve code execution on an ingestion worker.

**Impact.** High — a worker has database write access.

**Controls.**
- Parsers run with restricted filesystem access and **no network**.
- PDF/HTML libraries pinned, patched, and monitored; dependency scanning in CI.
- Malformed input produces a recorded `parser_executions` failure, never an exception that halts the source (adopting Cerevra's corruption-as-data pattern, `cerevra/wal/scanner.py:86-95`).
- Worker database role has no `UPDATE` on `public_events` and no `DELETE` on append-only tables — a compromised worker cannot rewrite history.
- Resource limits (CPU, memory, wall clock) per execution.

**Residual.** A zero-day in a parsing library. Reduced by least privilege: the blast radius is evidence insertion, which is visible and reversible by projection rebuild.

---

## T8 — Malicious media upload

**Scenario.** Malware, a decoder exploit, or illegal content uploaded as report evidence.

**Impact.** High — legal and reputational as much as technical.

**Controls.** Separate erasable bucket; EXIF stripped (GPS discarded, not stored); magic-byte type verification; re-encoding through a decoder; size caps; malware scanning before the object is referencable; separate serving origin with restrictive CSP; human moderation before public display.

**Residual.** Illegal content in the window before moderation. Mitigated by moderator tooling and a takedown path.

---

## T9 — Source poisoning

**Scenario.** An attacker compromises a government website JanSetu polls, and publishes false notices there.

**Impact.** High — evidence appears genuinely official.

**Controls.**
- Authority class is a property of the **registered source**, not of document content. A poisoned source can only publish within the authority it was already granted.
- Anomaly detection on volume, category drift, and format changes.
- Circuit breakers and health monitoring surface unusual behaviour.
- High-severity categories require moderation before wide notification.
- Immutable raw storage means the poisoned document is preserved as evidence for later investigation (I7).

**Residual.** Real. JanSetu inherits the security posture of the sources it trusts. **This is stated openly rather than implied away** — the honest position is that JanSetu can be no more trustworthy than its sources, and its job is to make the provenance visible so that a citizen can judge.

---

## T10 — Location spoofing

**Scenario.** A reporter falsifies their location to file reports about a place they are not.

**Impact.** Medium.

**Controls.** Coarse cells only (spoofing to cell precision is easy, so precision is not relied upon); corroboration requires spatial diversity across distinct cells; account history weighting; **device attestation explicitly not relied upon** — it excludes rooted devices and older phones, which disproportionately affects exactly the citizens JanSetu must serve.

**Residual.** Accepted. Corroboration diversity is the real control.

---

## T11 — Scraping private routes

**Scenario.** An attacker enumerates or exfiltrates saved routes to build movement profiles.

**Impact.** Very high — individual physical safety.

**Controls.** Separate schema with **no grant** to the public-API role (I17); RLS on `user_id`; column-level encryption on geometry; no bulk endpoint; no aggregate publication without k-anonymity; alert on any access to route tables by a non-route role; 30-day retention on impact assessments.

**Residual.** Insider access or a full database compromise. Mitigated by encryption and audit; not eliminated.

---

## T12 — Cross-node data mutation

**Scenario.** One civic node alters or deletes another node's event.

**Impact.** High — destroys the federation trust model.

**Controls.** RLS on `civic_node_id` (I14); federated events in a separate table with **no local write grant** (I20); envelope bytes immutable via trigger; verification on read detects at-rest tampering; runtime assertions that raise on cross-node rows.

**Residual.** Low. Enforced at three independent layers.

---

## T13 — Notification spam

**Scenario.** Abuse of the notification channel to send mass unwanted alerts, whether by an attacker or by a well-meaning node during an incident.

**Impact.** High — **alert fatigue disables the channel exactly when it is needed most.** This is the subtlest high-consequence threat in the model.

**Controls.** Per-user, per-event, per-day caps (I9); severity thresholds and user-configurable quiet hours; delivery unique constraint so retries cannot duplicate; freshness budget so stale events cannot alert (I23); severity-prioritised queues; per-node publication rate limits.

**Residual.** A legitimate high-severity event generating genuinely many alerts. Not a bug.

---

## T14 — Denial of service during a disaster

**Scenario.** Traffic spike — organic or malicious — during the exact event JanSetu exists to report.

**Impact.** Highest availability consequence.

**Controls.**
- CDN absorbs read bursts; ×50 read burst is designed for ([workload-model.md](scaling/workload-model.md)).
- **Static fallback: a cached list of the highest-severity active events, servable with the database down.** The read path degrades to reduced information, never to an error page.
- Disaster mode sheds non-essential features and reprioritises queues (I24).
- Rate limiting with a **trusted-reporter lane**, so genuine reports are not suppressed alongside abuse.
- Regional isolation at national scale means one region's disaster does not exhaust another's capacity.

**Residual.** Notification fan-out latency (up to ~1 h at Scenario D). Disclosed, not concealed.

---

## Threats explicitly out of scope

| Threat | Why |
| --- | --- |
| Nation-state targeting of individual citizens | JanSetu stores no data that makes this materially easier than existing telecom metadata; the mitigation is not collecting the data (see §6 of [security-and-federation.md](security-and-federation.md)) |
| Physical compromise of a civic office | Outside JanSetu's control; addressed by node suspension |
| Compromise of the underlying cloud provider | Standard shared-responsibility boundary |

---

## Why not blockchain

Every threat above is addressed by asymmetric signatures (T1, T2, T12), immutable verbatim envelope storage (T12), nonce and time-window replay protection (T5), an append-only audit log (T5, T12), and out-of-band identity verification (T1).

The residual risks — T1's captured authority, T9's poisoned source, T4's patient brigade — are **social and organisational**, not cryptographic. A distributed ledger would record a false event immutably rather than prevent it, at very large cost and operational burden.

Blockchain is revisited only if a threat is identified that signatures plus append-only history demonstrably cannot address. None currently is.

---

## Priority summary

| Priority | Threats | Common control theme |
| --- | --- | --- |
| **P0** | T1, T2, T3, T14 | Authority binding, positive-evidence requirements, graceful degradation |
| **P1** | T4, T7, T9, T11, T13 | Independence requirements, least privilege, isolation, caps |
| **P2** | T8, T12 | Sanitisation, RLS, immutability |
| **P3** | T5, T6, T10 | Idempotency, nonces, coarse precision |

The recurring theme across P0: **JanSetu's security model is not primarily about keeping attackers out. It is about ensuring that a successful attack produces a *visible, labelled, correctable* claim rather than an invisible authoritative one.** Every control above is chosen to preserve that property.
