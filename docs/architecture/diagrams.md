# Architecture Diagrams

Twenty purpose-specific diagrams. Each states its purpose, applicability, the invariant it protects, the failure mode it makes visible, and whether it describes PMVP or a later scale.

---

## 1. PMVP request architecture

**Purpose:** the whole PMVP in one picture. **Applies:** Scenario A. **Invariant:** I10 — cache loss cannot destroy authoritative state. **Failure mode:** none singular; this is the baseline. **Phase:** PMVP.

```mermaid
flowchart TD
    PWA[Citizen PWA] --> API
    PUB[Publisher console] --> API
    MOD[Moderation console] --> API
    API[FastAPI monolith: api process] --> PG[(PostgreSQL 16 + PostGIS)]
    API -.optional.-> REDIS[(Redis: rate limit, cache)]
    WORKER[worker process] --> PG
    WORKER --> S3[(S3: immutable raw documents)]
    SCHED[scheduler] --> PG
    PG -. commands .-> WORKER
    style REDIS stroke-dasharray: 5 5
    style PG stroke-width:3px
```

Redis is dashed because every path through it must work when it is absent.

---

## 2. Three-tier logical architecture

**Purpose:** enforce dependency direction. **Applies:** all scales. **Invariant:** module boundaries; I22. **Failure mode:** boundary erosion via cross-module joins. **Phase:** PMVP onward.

```mermaid
flowchart TD
    subgraph Presentation
        A[HTTP routers, serialisers, auth middleware]
    end
    subgraph Application
        B[Application services: ingestion, projection, notification, moderation]
    end
    subgraph Domain
        C[Trust rules, state machine, matching, impact classification]
    end
    subgraph Persistence
        D[Repositories, PostGIS, object storage, outbox]
    end
    A --> B --> C
    B --> D
    C -. defines interfaces implemented by .-> D
```

Dependencies point inward only. Enforced by `import-linter` in CI, not by convention.

---

## 3. Source-ingestion pipeline

**Purpose:** the full write path. **Applies:** all scales. **Invariants:** I1, I2, I7, I11. **Failure mode:** duplicate effects; dangling object references. **Phase:** PMVP.

```mermaid
flowchart TD
    SCHED[Scheduler tick] --> CMD[INSERT ingestion_commands<br/>ON CONFLICT idempotency_key DO NOTHING]
    CMD --> CLAIM[SELECT ... FOR UPDATE SKIP LOCKED<br/>set CLAIMED + lease]
    CLAIM --> FETCH[Fetch source<br/>breaker-aware, backoff+jitter]
    FETCH --> ATT[INSERT ingestion_attempts append-only]
    FETCH --> HASH[sha256 full width]
    HASH --> DUP{digest already<br/>known?}
    DUP -->|yes| SKIP[No new version<br/>duplicates_total++]
    DUP -->|no| PUT[PUT S3 object, verify]
    PUT --> TXN
    subgraph TXN[Single transaction]
        V[INSERT source_document_versions]
        P[INSERT parser_executions]
        E[INSERT evidence_items ON CONFLICT DO NOTHING]
        Q[INSERT projection command]
        S[UPDATE command SUCCEEDED]
    end
    TXN --> ACK[COMMIT = acknowledgement]
```

Object is written and verified **before** the referencing row commits. An orphan object is harmless; a dangling reference is not.

---

## 4. Acknowledgement boundary

**Purpose:** show precisely where durability is established. **Applies:** all scales. **Invariant:** I11. **Failure mode:** a crash losing an acknowledged command. **Phase:** PMVP.

```mermaid
sequenceDiagram
    participant W as Worker
    participant S3
    participant PG as PostgreSQL
    W->>PG: claim command (lease 5 min)
    W->>W: fetch + parse
    W->>S3: PUT object (content-addressed)
    S3-->>W: verified
    rect rgb(220,240,220)
    W->>PG: BEGIN
    W->>PG: INSERT version, parser_execution, evidence
    W->>PG: INSERT projection command
    W->>PG: UPDATE command SUCCEEDED
    W->>PG: COMMIT
    end
    Note over W,PG: Acknowledgement IS the commit.<br/>There is no separate ack step to lose.
```

This is the design consequence of the audit finding that Cerevra's recovery engine reads a WAL stack its ingestion path never writes to (`cerevra/wal/recovery.py:17`). Making the log a table removes that class of mismatch entirely.

---

## 5. Crash and replay sequence

**Purpose:** show recovery from each crash point. **Applies:** all scales. **Invariants:** I2, I11. **Failure mode:** lost or duplicated work. **Phase:** PMVP.

```mermaid
sequenceDiagram
    participant W1 as Worker A
    participant PG as PostgreSQL
    participant W2 as Worker B
    W1->>PG: claim cmd#42 (lease → T+5m)
    W1->>PG: (fetch, parse in progress)
    Note over W1: SIGKILL
    Note over PG: cmd#42 CLAIMED, lease expiring
    W2->>PG: scan for lease_expires_at < now()
    PG-->>W2: cmd#42
    W2->>PG: re-claim
    W2->>PG: full pipeline
    Note over PG: ON CONFLICT absorbs any partial<br/>evidence Worker A had written
    W2->>PG: COMMIT (SUCCEEDED)
```

Recovery is lease expiry. There is no separate startup recovery pass, therefore none to get wrong.

---

## 6. Evidence-to-event projection

**Purpose:** the derivation of public truth. **Applies:** all scales. **Invariants:** I3, I6, I12, I13. **Failure mode:** projection divergence. **Phase:** PMVP.

```mermaid
flowchart TD
    subgraph AppendOnly[APPEND_ONLY - authoritative history]
        EV[evidence_items<br/>ordered by sequence]
        TR[event_transitions]
    end
    RS[rulesets<br/>versioned, immutable]
    CP[(projection_checkpoints<br/>last_applied_sequence<br/>projection_hash)]
    EV --> FOLD[Pure fold<br/>no now, no network, no map order]
    RS --> FOLD
    CP --> FOLD
    FOLD --> TR
    FOLD --> PE[public_events - DERIVED<br/>written only by projector]
    FOLD --> HASH[compute projection hash]
    HASH --> CMP{matches<br/>checkpoint?}
    CMP -->|yes| OK[store checkpoint]
    CMP -->|no| DIV[replay_divergence_total++<br/>PAGE]
```

The divergence check is what Cerevra lacks entirely — it stores no projection hash, so a code change silently yields a different manifest. Without this branch, "every event is reconstructable" is an unfalsifiable claim.

---

## 7. Event state machine

**Purpose:** legal transitions. **Applies:** all scales. **Invariants:** I3, I4. **Failure mode:** an event resolving through silence, or community reports producing official confirmation. **Phase:** PMVP.

```mermaid
stateDiagram-v2
    [*] --> DRAFT: ambiguous location
    [*] --> DETECTED: evidence received
    DRAFT --> DETECTED: moderator resolves location
    DRAFT --> REJECTED: moderator rejects
    DETECTED --> UNVERIFIED: published, single source
    UNVERIFIED --> COMMUNITY_CORROBORATED: N independent reports,<br/>distinct cells
    UNVERIFIED --> OFFICIALLY_CONFIRMED: official evidence<br/>from authorised node
    COMMUNITY_CORROBORATED --> OFFICIALLY_CONFIRMED: official evidence
    COMMUNITY_CORROBORATED --> CONFLICTING: contradictory evidence
    OFFICIALLY_CONFIRMED --> ACTIVE: start time reached
    OFFICIALLY_CONFIRMED --> CONFLICTING: contradictory official evidence
    ACTIVE --> DISPUTED: credible contradiction
    ACTIVE --> STATUS_UNKNOWN: evidence past freshness budget
    ACTIVE --> PARTIALLY_RESOLVED: partial resolution evidence
    ACTIVE --> RESOLVED: resolution evidence
    STATUS_UNKNOWN --> ACTIVE: fresh evidence
    STATUS_UNKNOWN --> STALE: prolonged silence
    PARTIALLY_RESOLVED --> RESOLVED: resolution evidence
    DISPUTED --> ACTIVE: dispute resolved
    CONFLICTING --> ACTIVE: conflict resolved
    RESOLVED --> REOPENED: new contradicting evidence
    REOPENED --> ACTIVE
    RESOLVED --> ARCHIVED: retention
    STALE --> ARCHIVED: retention
    REJECTED --> [*]
    ARCHIVED --> [*]
```

**There is no edge from any state to `RESOLVED` that is not labelled with resolution evidence.** Silence leads to `STATUS_UNKNOWN` and then `STALE` — never to resolution (I4). Likewise, no edge reaches `OFFICIALLY_CONFIRMED` without official evidence (I3).

---

## 8. Cache-aside read path

**Purpose:** read path with correct staleness handling. **Applies:** city scale onward. **Invariant:** I5, I10. **Failure mode:** a cached "road open" surviving a real closure. **Phase:** city.

```mermaid
sequenceDiagram
    participant C as Citizen
    participant CDN
    participant API
    participant R as Redis
    participant DB as Read replica
    C->>CDN: GET /v1/events?bbox=
    alt CDN hit and fresh
        CDN-->>C: cached JSON + as_of
    else miss
        CDN->>API: forward
        API->>R: GET key(bbox, filters, ruleset_version)
        alt hit
            R-->>API: payload
        else miss
            API->>DB: query public_events + PostGIS
            DB-->>API: rows + as_of
            API->>R: SETEX 30s
        end
        API-->>CDN: JSON + as_of + freshness
        CDN-->>C: JSON
    end
    Note over API,CDN: On any event transition the committing worker<br/>purges Redis keys AND issues a CDN purge, with retries.
```

Purge-on-transition is a correctness requirement, not an optimisation.

---

## 9. Geospatial impact calculation

**Purpose:** route × event intersection. **Applies:** all scales. **Invariant:** I17 — routes never become public. **Failure mode:** route data leaking into a public response. **Phase:** PMVP.

```mermaid
flowchart TD
    T[Event transition committed] --> CAND[Candidate routes:<br/>bbox pre-filter on event geometry]
    CAND --> GIST[GiST index scan<br/>ST_DWithin route.geom, event.geom, buffer]
    GIST --> CLASS{Classify}
    CLASS -->|geometry intersects| B[blocked]
    CLASS -->|within buffer| D[delayed]
    CLASS -->|nearby, not on route| N[nearby]
    CLASS -->|no interaction| CL[clear - no row]
    B --> RIA[(route_impact_assessments<br/>PERSONAL, separate schema)]
    D --> RIA
    N --> RIA
    RIA --> NOTIF[notification decision]
    RIA -.->|NO GRANT| PUBAPI[public API]
    style PUBAPI stroke-dasharray: 5 5
    style RIA stroke-width:3px
```

The dashed, ungranted edge is the point of the diagram. At city scale an H3 coarse pre-filter may replace the bbox step; PostGIS remains authoritative.

---

## 10. Notification fan-out

**Purpose:** delivery with dedup and prioritisation. **Applies:** all scales. **Invariants:** I9, I23. **Failure mode:** duplicate alerts; alert fatigue. **Phase:** PMVP (simple) → national (prioritised).

```mermaid
flowchart TD
    T[Event transition] --> DEC[Decision worker]
    DEC --> CHK{Checks}
    CHK -->|evidence stale| S1[suppress_stale]
    CHK -->|per-user cap hit| S2[suppress_cap]
    CHK -->|quiet hours| S3[suppress_quiet]
    CHK -->|below threshold| S4[suppress_threshold]
    CHK -->|pass| SEND[outcome = send]
    S1 --> ND[(notification_decisions<br/>APPEND_ONLY, reason recorded)]
    S2 --> ND
    S3 --> ND
    S4 --> ND
    SEND --> ND
    SEND --> DL[(notification_deliveries<br/>UNIQUE subscription, event, transition)]
    DL --> PQ[Severity-prioritised queues]
    PQ --> P1[push]
    PQ --> P2[webhook]
    PQ --> P3[SMS - budget capped]
    P1 --> RETRY{failed?}
    RETRY -->|yes| DL
    style DL stroke-width:3px
```

Retries re-attempt the **same row**; the unique key makes duplicate sends impossible. Suppressions are recorded with a reason — "we chose not to alert you" must be auditable.

---

## 11. Circuit breaker for failing sources

**Purpose:** protect the source and surface the failure to citizens. **Applies:** all scales. **Invariants:** I4, I15. **Failure mode:** an event silently freezing, or silently resolving, because its source went dark. **Phase:** PMVP.

```mermaid
stateDiagram-v2
    [*] --> CLOSED
    CLOSED --> OPEN: consecutive_failures >= threshold
    OPEN --> HALF_OPEN: cooldown elapsed
    HALF_OPEN --> CLOSED: probe succeeds
    HALF_OPEN --> OPEN: probe fails
    note right of OPEN
        source_health updated
        → events show STALE
        → source_health_age_seconds alerts
        → NO event resolves (I4)
    end note
```

The note is the important part: an open breaker must produce a **visible** civic state, not merely an operational metric.

---

## 12. City-scale architecture

**Purpose:** what changes at Scenario B. **Applies:** city. **Invariant:** I13 — checkpointed replay. **Failure mode:** replica lag rendering a live closure absent. **Phase:** city.

```mermaid
flowchart TD
    CDN --> LB[Load balancer]
    LB --> API[api x N]
    API --> PGP[(PG primary)]
    API --> PGR[(PG read replica)]
    API --> R[(Redis)]
    ING[ingestion workers] --> PGP
    ING --> S3[(S3 + lifecycle)]
    PROJ[projection worker<br/>checkpointed] --> PGP
    NOT[notification worker<br/>separate pool] --> PGP
    SCHED[scheduler - single] --> PGP
    PGP -.async replication.-> PGR
    style PGR stroke-dasharray: 5 5
```

Same image, five process types. Writes and read-your-own-write paths go to the primary. If lag exceeds the freshness budget the router fails back to the primary and alerts, rather than serving stale civic data.

---

## 13. Database partitioning

**Purpose:** partition strategy per table. **Applies:** city onward. **Invariant:** retention without bulk deletes. **Failure mode:** maintenance windows exceeding the availability budget. **Phase:** city.

```mermaid
flowchart TD
    subgraph Partitioned[Monthly range partitions on time]
        E[evidence_items - known_at]
        V[source_document_versions - fetched_at]
        C[ingestion_commands - created_at]
        A[ingestion_attempts - started_at]
        N[notification_deliveries - first_attempt_at]
        AU[audit_entries - occurred_at]
    end
    subgraph Unpartitioned[Small, heavily joined]
        PE[public_events]
        CN[civic_nodes]
        SR[sources]
        LO[locations]
    end
    Partitioned --> DET[Retention = DETACH PARTITION + archive]
    DET --> COLD[(Cold object storage - Parquet)]
```

At national scale the partition key becomes `(jurisdiction, month)` — jurisdiction first, because that is what makes 128-way parallel replay possible (6.5 min versus 13.9 h).

---

## 14. National / federated architecture

**Purpose:** Scenario D shape. **Applies:** national. **Invariant:** I22 — analytics is never operational truth. **Failure mode:** the broker becoming the de facto source of truth. **Phase:** national.

```mermaid
flowchart TD
    EDGE[Edge / CDN per region] --> RW[Region West]
    EDGE --> RN[Region North]
    subgraph RW[Region West]
        AW[api + workers] --> PW[(PG cluster)]
        AW --> SW[(S3)]
        AW --> OW[(OpenSearch)]
    end
    subgraph RN[Region North]
        AN[api + workers] --> PN[(PG cluster)]
        AN --> SN[(S3)]
        AN --> ON[(OpenSearch)]
    end
    PW --> STREAM[Durable stream<br/>partitioned by region, city]
    PN --> STREAM
    STREAM --> CH[(ClickHouse - ANALYTICAL)]
    STREAM --> FED[Federation gateway]
    CP[Control plane - global] -.policy, keys, revocations.-> RW
    CP -.-> RN
    CH -.->|NO operational grant| AW
    style CH stroke-dasharray: 5 5
    style CP stroke-width:3px
```

Postgres remains the system of record. The stream distributes; it never becomes authoritative.

---

## 15. Control plane / data plane separation

**Purpose:** show what survives a control-plane outage. **Applies:** national. **Invariant:** availability of civic reads during control-plane failure. **Failure mode:** control-plane outage stopping the data plane. **Phase:** national (logical from PMVP).

```mermaid
flowchart TD
    subgraph CP[Control plane - global, small]
        N[node registration]
        K[signing keys]
        PM[permissions, jurisdiction]
        SC[schema, protocol versions]
        FL[feature flags, retention policy]
    end
    subgraph DP[Data plane - regional, high volume]
        I[ingestion]
        PJ[projection]
        Q[querying]
        G[geospatial]
        NT[notification]
    end
    CP -->|policy push + TTL cache| DP
    DP -->|continues on cached policy| DP
    CP -.->|revocations: immediate, short TTL 5 min| DP
    Note[Control plane down:<br/>ingestion, projection, reads, delivery CONTINUE.<br/>No new node, no key rotation, no new grant.]
    CP -.-> Note
```

Revocation is the deliberate exception — a 5-minute signing-key TTL trades availability for the ability to cut off a compromised node quickly. That is the one place this design chooses consistency over availability, and it is chosen knowingly.

---

## 16. Civic node event signing

**Purpose:** federation trust. **Applies:** V1 onward. **Invariant:** I20. **Failure mode:** silent alteration of another node's event. **Phase:** V1.

```mermaid
sequenceDiagram
    participant N as Civic node
    participant KMS as Node KMS (JanSetu never holds this)
    participant J as JanSetu
    participant C as Citizen
    N->>N: build envelope (canonical serialisation)
    N->>KMS: sign
    KMS-->>N: signature
    N->>J: POST envelope + signature + key_id
    J->>J: lookup node_keys, check validity + revocation
    J->>J: verify signature; check nonce + issued_at window
    alt valid
        J->>J: store envelope_bytes VERBATIM, immutable
        J-->>N: accepted
    else invalid
        J->>J: federation_signature_failures_total++, quarantine, PAGE
        J-->>N: rejected
    end
    C->>J: read event
    J->>J: RE-VERIFY signature at read time
    J-->>C: event + provenance + verification status
```

Read-time re-verification is what detects tampering with data at rest. Another node may add supporting evidence, contradictory evidence, a correction request, or a note — never a mutation.

---

## 17. Hot / warm / cold lifecycle

**Purpose:** data movement and restoration. **Applies:** city onward. **Invariant:** I7 immutability preserved across tiers. **Failure mode:** archived evidence unrestorable when an audit needs it. **Phase:** city.

```mermaid
flowchart LR
    subgraph HOT[Hot - ms]
        H1[active events]
        H2[evidence < 30d]
        H3[source health]
        H4[pending notifications]
    end
    subgraph WARM[Warm - 10s ms]
        W1[resolved events < 1y]
        W2[moderation history]
        W3[recent documents]
    end
    subgraph COLD[Cold - hours]
        C1[historic document versions]
        C2[archived evidence]
        C3[audit > 1y]
        C4[replay snapshots]
    end
    HOT -->|30d / resolution| WARM
    WARM -->|1y / DETACH PARTITION| COLD
    COLD -->|restore for audit or replay<br/>documented, tested, hours| WARM
```

Restoration is a **tested** procedure with a stated latency. An archive that has never been restored from is an archive that cannot be.

---

## 18. Multi-region disaster recovery

**Purpose:** RPO/RTO and failover. **Applies:** national. **Invariant:** no acknowledged command lost. **Failure mode:** region loss. **Phase:** national.

```mermaid
flowchart TD
    subgraph P[Primary region]
        PP[(PG primary)] -->|sync| PR[(PG replica)]
        PS[(S3)]
    end
    subgraph D[DR region]
        DP[(PG standby)]
        DS[(S3 replica)]
    end
    PP -->|async, RPO <= 60s| DP
    PS -->|cross-region replication| DS
    CP[(Control plane - globally replicated)]
    CP --> P
    CP --> D
    F1[In-region failure: promote replica, RTO <= 15 min]
    F2[Region loss: promote DR, restore objects, RTO <= 4h]
    P -.-> F1
    D -.-> F2
```

Rehearsed at least twice yearly. A DR plan that has not been executed is a document, not a plan.

---

## 19. Event merge and split

**Purpose:** history preservation through restructuring. **Applies:** all scales. **Invariants:** I18, I19. **Failure mode:** losing evidence links or breaking a public URL. **Phase:** PMVP.

```mermaid
flowchart TD
    subgraph Merge
        EA[Event A + links] --> M{merge}
        EB[Event B + links] --> M
        M --> ET[Event A surviving<br/>links = A ∪ B]
        M --> EAR[Event B ARCHIVED<br/>superseded_by = A]
        M --> MR[(event_merges<br/>APPEND_ONLY)]
        EAR -.id still resolves, redirects.-> ET
    end
    subgraph Split
        EP[Event P + links] --> S{split}
        S --> E1[Child 1 + partition 1]
        S --> E2[Child 2 + partition 2]
        S --> EPA[Event P ARCHIVED<br/>lists children]
        S --> SR[(event_splits<br/>evidence_partition jsonb)]
        EPA -.id still resolves, lists children.-> E1
    end
```

Property-tested: merge conserves the union of links; split partitions them exactly once. **Both original ids resolve forever** — a citizen who shared a link must never get a 404.

---

## 20. Search index rebuild

**Purpose:** prove search is derived. **Applies:** all scales. **Invariant:** I21. **Failure mode:** a field existing only in the index, making rebuild lossy. **Phase:** PMVP (FTS) → national (OpenSearch).

```mermaid
flowchart TD
    subgraph Authoritative
        PE[public_events]
        EV[evidence_items]
        SD[source_document_versions]
        LO[locations]
    end
    PE --> B[Pure document builder<br/>no fields originate here]
    EV --> B
    SD --> B
    LO --> B
    B --> IDX[(Search index - DERIVED)]
    IDX --> Q[citizen / moderator search]
    Q -.->|candidates only| PROJ[projection]
    PROJ -.->|search NEVER determines state| X((✗))
    DROP[DROP index] --> B
    style X stroke-width:3px
```

Rebuild is exercised regularly, not merely documented. The blocked edge is the point: search returns candidates; the projection determines truth.

---

## Coverage map

| Concern | Diagrams |
| --- | --- |
| Write path and durability | 3, 4, 5 |
| Truth derivation | 6, 7, 20 |
| Read path | 8, 12, 14 |
| Geospatial | 9, 13 |
| Delivery | 10 |
| Resilience | 11, 18 |
| Scale evolution | 12, 13, 14, 15, 17 |
| Trust and federation | 16, 19 |
| Structure | 1, 2 |
