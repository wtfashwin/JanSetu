## Problem

<!-- What is wrong, missing, or needed? Link the issue if there is one. -->

## Scope

<!-- What this PR changes. Keep it to one concern. -->

## Non-goals

<!-- What this PR deliberately does not do. This is not a formality — stating
     the boundary makes review faster and prevents scope creep in follow-ups. -->

## ADR / invariants affected

<!-- Does this touch an accepted ADR (docs/adr/) or a reliability invariant
     (docs/architecture/reliability-invariants.md)?

     If it changes architecture in a way listed in ARCHITECTURE_STATUS.md,
     link the new ADR. Structural changes are not ordinary PRs.

     If it touches an invariant, name it (e.g. I3, I12) and say how it is
     still enforced. -->

- [ ] No ADR or invariant affected
- [ ] Invariant(s) affected: <!-- I__ -->
- [ ] New/updated ADR: <!-- docs/adr/____ -->

## Tests

<!-- What you added or changed, and which tier (unit / property / integration /
     replay / chaos / load). A change touching an invariant needs a test that
     would fail if the invariant were broken. -->

## Privacy and security considerations

<!-- Does this touch personal data (saved routes, subscriptions, reporter
     identity), signing, authorisation, or anything a citizen sees as a
     truth claim? If not, say "none" — do not leave this blank. -->

## Limitations

<!-- What this does not handle, what is deferred, what could be wrong.
     A PR that states its limits is easier to accept than one implying
     completeness. -->

---

- [ ] Unknown information stays explicit — nothing estimated, defaulted or interpolated
- [ ] Parsers touched are pure (no database, network, or clock)
- [ ] Fast test tiers still run with no Docker and no network
