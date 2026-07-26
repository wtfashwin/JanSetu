# Security Policy

## Reporting a vulnerability

**Do not open a public issue for a security vulnerability.**

Report privately through [GitHub Security Advisories](https://github.com/wtfashwin/JanSetu/security/advisories/new).

Please include: what you found, how to reproduce it, what an attacker could achieve, and any suggested mitigation. If you are unsure whether something qualifies, report it — a false positive costs a few minutes; an unreported issue in civic infrastructure costs more.

### What to expect

| Stage | Target |
| --- | --- |
| Acknowledgement | 72 hours |
| Initial assessment | 7 days |
| Fix or documented mitigation | Depends on severity; you will be kept informed |

Reporters are credited in the advisory unless they ask not to be. Good-faith research is welcome; please do not access, modify or exfiltrate data belonging to other people while testing.

---

## Current status

JanSetu is at the architecture-baseline stage. There is no deployed system and no running code in this repository. Reports at this stage will mostly concern **design weaknesses**, and those are genuinely valuable — a flaw caught in an ADR is far cheaper than one caught in production.

---

## High-priority risks

These are the failures that matter most. Each is analysed in [docs/architecture/threat-model.md](docs/architecture/threat-model.md), and each maps to invariants in [docs/architecture/reliability-invariants.md](docs/architecture/reliability-invariants.md).

**False official event creation.**
Anything allowing an event to reach `OFFICIALLY_CONFIRMED` without evidence from a node authorised for that jurisdiction and category. This is the highest-severity class in the project: citizens act on official events during emergencies.

**Cross-node mutation.**
Any path by which one civic node alters, deletes or suppresses another node's event or evidence. Nodes may contradict each other publicly; they may never rewrite each other.

**Route-data exposure.**
Any path exposing a user's saved routes, route-impact assessments, or subscription set to another party. A saved commute is a movement pattern, and this is the most sensitive data the system holds.

**Reporter identity exposure.**
Any path linking a community report to a real identity, or narrowing it further than the intended coarse location cell. People reporting infrastructure failures can face retaliation.

**Signature bypass.**
Accepting a federated event with an invalid, missing, replayed or revoked signature — or failing to re-verify a stored envelope on read.

**Append-only history mutation.**
Any path that updates or deletes evidence, event transitions, parser executions, document versions or audit entries. If history can be silently rewritten, nothing else in the design holds.

**Notification amplification.**
Any path allowing unbounded or duplicate notifications to a user. Alert fatigue disables the channel precisely when it is needed most, so this is a safety issue rather than an annoyance.

**Replay divergence.**
Any path where a projection cannot be reproduced from its evidence and ruleset, or where divergence occurs without detection. This falsifies the project's central claim.

**Parser sandbox escape.**
Code execution, filesystem access or network access from a parser processing an untrusted document. Parsers consume attacker-influenceable PDFs and HTML.

---

## Out of scope

- Vulnerabilities in third-party dependencies without a demonstrated impact on JanSetu (report those upstream).
- Findings requiring physical access to a civic authority's infrastructure.
- Social-engineering attacks against civic node operators.
- Automated scanner output with no demonstrated impact.
- Missing hardening headers on documentation-only pages.

## A limitation stated openly

A civic node that is legitimately authorised, and that publishes a false event, cannot be stopped by cryptography. The design's protection is that such an event is **visible, attributed, contradictable and correctable** — not that it is impossible.

If you find a way to make a false event *invisible*, *unattributable*, or *uncorrectable*, that is a serious vulnerability and we want to hear about it.
