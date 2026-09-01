# `pg_mdm` OSS v1 design

## A small PostgreSQL-native contract for entity resolution and golden records

**Status:** Proposed first open-source release  
**Goal:** Ship a useful, deterministic MDM extension without making long-term operational features part of the first compatibility promise  
**Product contract:** Five nouns, five actions, and three outputs

---

## 1. Release boundary

`pg_mdm` v1 resolves records already available through PostgreSQL relations. A user defines a real-world entity, maps source columns to logical fields, describes matching evidence, and chooses golden values. The extension publishes a durable membership map, one golden record per entity, and a review queue.

The public model has five nouns:

1. **source**
2. **field**
3. **match**
4. **entity**
5. **golden value**

Normal operation has five actions:

1. **create**
2. **describe**
3. **preview**
4. **refresh**
5. **explain**

Every entity has three public outputs:

1. `mdm_out.<entity>`
2. `mdm_out.<entity>_members`
3. `mdm_out.<entity>_review`

This boundary is the main compatibility promise. Candidate pairs, normalized values, graph edges, publication revisions, invalidation state, and physical indexes remain engine details.

V1 deliberately limits its other promises. PostgreSQL supplies transactions, MVCC, locking, backup, replication, permissions, and workload controls. The extension does not duplicate those systems in its first release.

### 1.1 What v1 includes

```text
Definition
  source
  field
  match
  entity
  golden_value

Actions
  create
  describe
  preview
  refresh
  explain

Outputs
  entity
  members
  review

Matching
  exact
  fuzzy
  identity / strong / supporting

Stewardship
  MATCH
  NOT_MATCH
  golden override

Sources
  snapshot
  soft_delete
  changed_at

Engine
  deterministic normalization
  bounded candidate generation
  conservative clustering
  stable mdm_id
  incremental refresh
  atomic publication
  golden provenance
```

### 1.2 What v1 does not promise

V1 does not include:

- namespace or multi-tenant resource isolation
- formal service-level objectives
- workload fairness or extension-managed backpressure
- resumable multi-worker coordination
- backup, failover, or restore verification beyond normal PostgreSQL behavior
- configurable storage lifecycle or compaction
- valid-time reconstruction or late-event correction
- downstream entity dependency graphs
- a semantic downstream change feed
- organization fragment graphs or nested configuration composition
- statistically representative global merge and split estimates
- custom candidate generation or component checks
- merge, split, move-member, or lock workflow operations
- multiple clustering policies

These are candidates for later releases, not hidden v1 requirements. [DESIGN_V2.md](DESIGN_V2.md) records them without expanding the v1 contract.

---

## 2. Product model

### 2.1 Source

A source is a PostgreSQL table, view, materialized view, or compatible foreign relation. It provides:

- a unique and stable source ID
- mappings from physical columns to logical fields
- one of the three v1 source modes
- optional source authority for matching or golden selection

Within one entity, `(source, source_id)` identifies one source record. Reusing a key for a different real-world record is unsupported.

### 2.2 Field

A field is a logical attribute such as `name`, `email`, `phone`, or `tax_id`. Sources may map different columns to the same field. A field defines its type, cleaner, and basic missing-value rules.

V1 recognizes these value states:

- `value`
- `absent`
- `empty`
- `invalid`
- `unknown`
- `redacted`
- `unsupported`

The engine keeps source values unchanged and stores normalized values separately. Missing or invalid values provide no positive or negative matching evidence unless a built-in comparison explicitly says otherwise.

### 2.3 Match

A match says how fields provide identity evidence. It combines:

- `exact` or `fuzzy` comparison
- `identity`, `strong`, or `supporting` evidence

Identity evidence may decide a match and blocks a merge when valid authoritative values disagree. Strong evidence can produce an automatic match under the built-in decision table. Supporting evidence can strengthen a decision or create a review, but cannot produce an automatic match by itself.

### 2.4 Entity

An entity is the real-world thing being mastered, such as a company, person, supplier, or product. Its definition owns sources, fields, matches, golden values, and conservative safety options.

The entity does not expose graph algorithms, score arithmetic, candidate channels, worker counts, or index choices. The engine derives those details.

### 2.5 Golden value

A golden value selects one output field independently from the entity's members. V1 includes these policies:

- `prefer_source`
- `latest`, when the source has a trustworthy `changed_at`
- `first_non_null`
- `most_common`
- custom registered selector

Every selected value keeps provenance to the source record and source column, or to the registered selector and all contributing source values.

---

## 3. Entity definitions

An entity definition is an immutable, versioned value. `mdm.create()` accepts a complete definition and stores a new version only when its semantics differ from the current version.

V1 supports:

- built-in presets such as `person`, `company`, and `product`
- one optional `preset`, or a one-level `compose` list of versioned built-in presets
- explicit entity-local overrides
- fully expanded output from `mdm.describe()`

One-level means that an entity may compose presets, but those public preset inputs may not compose other user-defined inputs. V1 has no user-defined fragment registry. Conflicting preset values must be resolved by an entity-local override, and unresolved conflicts fail validation. The stored definition contains the expanded result and exact built-in preset versions, so a later preset release cannot silently change an active entity.

`mdm.describe(format => 'definition')` returns a canonical expanded definition that can be passed to `mdm.create()` in another database. Import resolves qualified relation and function names in the target database and fails if their required signatures differ. Physical indexes and batch choices are not part of the portable definition.

The constructors below illustrate the model rather than fix the final PostgreSQL type signatures:

```sql
WITH proposed AS (
    SELECT mdm.entity(
        name   => 'customer',
        preset => 'company',

        sources => ARRAY[
            mdm.source(
                name       => 'crm',
                relation   => 'crm.customer'::regclass,
                source_id  => 'id',
                mode       => 'changed_at',
                changed_at => 'updated_at',
                fields     => jsonb_build_object(
                    'name',   'display_name',
                    'email',  'email_address',
                    'tax_id', 'vat_number'
                )
            ),
            mdm.source(
                name         => 'registry',
                relation     => 'registry.company'::regclass,
                source_id    => 'registry_id',
                mode         => 'snapshot',
                fields       => jsonb_build_object(
                    'name',   'registered_name',
                    'email',  null,
                    'tax_id', 'registered_tax_id'
                ),
                authority    => jsonb_build_object(
                    'identity', 'authoritative',
                    'name',     'authoritative',
                    'tax_id',   'authoritative'
                )
            )
        ],

        fields => ARRAY[
            mdm.field(name => 'name', cleaner => 'company_name'),
            mdm.field(name => 'email', cleaner => 'email'),
            mdm.field(name => 'tax_id', cleaner => 'tax_id')
        ],

        matches => ARRAY[
            mdm.match(
                fields     => ARRAY['tax_id'],
                comparison => 'exact',
                strength   => 'identity'
            ),
            mdm.match(
                fields     => ARRAY['email'],
                comparison => 'exact',
                strength   => 'strong'
            ),
            mdm.match(
                fields     => ARRAY['name'],
                comparison => 'fuzzy',
                strength   => 'supporting'
            )
        ],

        golden_values => ARRAY[
            mdm.golden_value(
                field   => 'name',
                policy  => 'prefer_source',
                sources => ARRAY['registry', 'crm']
            ),
            mdm.golden_value(
                field  => 'email',
                policy => 'latest'
            ),
            mdm.golden_value(
                field   => 'tax_id',
                policy  => 'prefer_source',
                sources => ARRAY['registry', 'crm']
            )
        ]
    ) AS definition
)
SELECT mdm.create(
    definition       => definition,
    expected_version => null,
    comment          => 'Initial customer identity model'
)
FROM proposed;
```

The definition says what identity means. It does not prescribe an index, candidate count, score threshold, graph implementation, or batch size.

---

## 4. Source modes

V1 has three source modes. This is the complete source lifecycle contract for the release.

| Mode | Required configuration | Deletion behavior | `deletion_completeness` |
|---|---|---|---|
| `snapshot` | Stable key | A key missing from the relation at the refresh snapshot is deleted | `proven` after a successful complete scan |
| `soft_delete` | Stable key, deletion predicate, and a source guarantee that keys are not hard-deleted | A row matching the predicate is deleted | `proven` through the visible `changed_at` boundary when every change updates that column |
| `changed_at` | Stable key and trustworthy `changed_at` | Hard deletion cannot be detected | `unknown` |

`mdm.describe()` exposes one deletion status per source:

```text
deletion_completeness = proven | unknown
```

The engine never interprets absence as deletion for `changed_at` sources. A `soft_delete` source that cannot guarantee retained tombstones has `deletion_completeness = unknown` and behaves like `changed_at` for hard deletions. Operators may run a broader reconciliation by changing the source to `snapshot`, restoring missing rows as soft-delete tombstones, or rebuilding from another complete relation.

V1 resolves the source state visible to the refresh transaction. It records the publication revision and source observation boundary needed to explain that published state. It does not distinguish `effective_at` from `observed_at`, reconstruct business-valid history, reorder late events, or revise what the source is now believed to have meant in the past.

Source schema checks cover mapped columns, types, stable-key uniqueness, and required change columns. Incompatible source changes stop refresh before publication. V1 does not define a general schema compatibility protocol.

---

## 5. The five actions

### 5.1 `mdm.create()`

`mdm.create()` validates and stores a complete entity definition. Validation covers:

- source access and stable-key uniqueness
- source mode requirements
- field mappings and cleaner signatures
- match completeness and bounded candidate discovery
- golden policy inputs
- registered extension function contracts
- output naming and permissions

The operation stores the user definition, expanded definition, semantic digest, creator, comment, and parent version. It does not publish new entities. `mdm.refresh()` publishes the desired definition.

An existing entity update uses `expected_version` to prevent accidental overwrite. Reusing an identical semantic definition is a no-op.

### 5.2 `mdm.describe()`

`mdm.describe()` shows:

- the concise and fully expanded definition
- active and desired configuration versions
- source modes and deletion completeness
- fields, matching roles, and golden policies
- the bounded candidate plan and required indexes
- active publication revision and pending work
- entity, singleton, conflict, and review counts
- warnings and the last refresh result

`format => 'definition'` returns the canonical expanded definition. V1 does not report namespace quotas, service objectives, change-feed retention, valid-time backlog, fragment graphs, or downstream dependency state.

### 5.3 `mdm.preview()`

V1 preview answers three questions:

1. Is this definition or refresh valid?
2. What index, scan, candidate, and clustering work will it require?
3. What representative resolution changes does it produce?

It has three modes:

| Mode | Behavior |
|---|---|
| `validation` | Validate and compile without resolving records |
| `sampled` | Run a deterministic bounded sample and return representative changes |
| `exact` | Resolve only explicitly supplied source records or `mdm_id` values |

```sql
SELECT *
FROM mdm.preview(
    entity     => 'customer',
    definition => :proposed_definition,
    mode       => 'sampled'
);
```

Preview labels every result as validation, sampled, or exact. It does not promise statistically meaningful global merge, split, or review counts. It does not run labeled regression suites, simulate arbitrary stewardship workflows, or become an alternate publication engine.

### 5.4 `mdm.refresh()`

`mdm.refresh()` resolves the source state visible to one consistent PostgreSQL transaction and atomically publishes a complete revision. Readers see either the prior revision or the new revision.

Only one refresh or rebuild for an entity may publish at a time, enforced by an entity-scoped PostgreSQL advisory or row lock. Different entities may refresh concurrently. V1 does not coordinate multiple extension-managed workers within one refresh.

The normal path is incremental:

1. Detect inserted and changed records.
2. Detect deletions only when the source mode proves them.
3. Recompute changed normalized values.
4. Regenerate affected candidates and pair evidence.
5. Recluster the complete affected components.
6. Reconcile stable IDs and golden values.
7. Validate and publish in one transaction.

If the engine cannot prove that local work is complete, it performs a broader scan or fails with an actionable error. Resource limits may delay or reject work. They may not produce a partial result or weaken matching rules.

Refresh records counts, stage timings, source observation boundaries, configuration version, and decision version. A no-change refresh does not create a new semantic publication.

### 5.5 `mdm.explain()`

`mdm.explain()` describes one current or retained published subject:

- a source record
- a candidate pair
- an entity
- a golden field
- a review item

It shows normalized values, discovery reason, evidence, conflicts, accepted connections, manual decisions, stable-ID outcome, and golden provenance as relevant.

When a pair was not generated, explain may perform one bounded on-demand comparison and report why normal candidate discovery omitted it. Historical explanation is by immutable publication revision only. V1 does not answer valid-time questions or provide a general cause graph between arbitrary revisions.

---

## 6. Matching and candidate discovery

V1 includes deterministic built-in cleaners for common text, company names, person names, email addresses, phone numbers, identifiers, dates, and numbers. Cleaner versions are part of the entity definition digest.

The engine compiles identity and strong matches into bounded built-in candidate channels. Examples include exact normalized-value indexes, composite exact keys, bounded prefixes, token blocks, and bounded nearest-neighbor fuzzy search. Supporting evidence may reuse those candidates but does not create an unbounded all-pairs comparison.

Candidate discovery has two hard rules:

1. Every channel has a deterministic bound.
2. Truncation or an oversized block is visible as review or failure, never as proof of non-match.

V1 does not allow custom candidate-key functions. A custom comparator must use a complete built-in candidate channel selected during definition validation.

Evidence uses deterministic fixed-point arithmetic and stable tie ordering. The same source snapshot, expanded definition, and active stewardship decisions produce the same pair decisions regardless of worker order or physical query plan.

---

## 7. One conservative clustering algorithm

V1 has one clustering algorithm. It is not configurable.

1. Start with one component per active source record.
2. Apply active `MATCH` decisions in canonical source-key order.
3. Reject any union that would place an active `NOT_MATCH` pair in one component.
4. Reject automatic unions with conflicting authoritative identity values.
5. Process automatic match edges in descending evidence order with canonical source-key tie breaks.
6. Allow a singleton to join a compatible component on one accepted automatic match.
7. Join two non-singleton components only through shared authoritative identity or at least two independent strong cross-component matches.
8. Send a plausible rejected union to review.

This policy prevents one weak bridge from joining two established entities. Supporting evidence can rank or reinforce an edge, but cannot satisfy the strong cross-component requirement by itself.

The engine stores enough accepted-edge and rejection information for `explain()`. Custom component checks and alternate component policies are deferred.

---

## 8. Stable identity

Each active entity has a durable `mdm_id`. V1 uses one published rule set.

**New entity.** Allocate a time-ordered UUID using the supported PostgreSQL generator.

**Unchanged component.** Keep its existing `mdm_id`.

**Merge.** The oldest existing `mdm_id` survives. Break equal creation-time ties by UUID byte order. Retain aliases from retired IDs to the survivor.

**Split.** The child containing the oldest surviving source member keeps the old `mdm_id`. Source-member age is its first published membership time. Break ties by `(source, source_id)`. Other children receive new IDs and retain `split_from` history.

**Deletion and reappearance.** Retain a source-record tombstone. If the same stable source key reappears, reactivate that source-record identity and resolve it through normal evidence.

These rules are deterministic and versioned. Rebuild uses the same identity ledger and must reproduce the same IDs for the same source state, definition, and decisions. Later continuity heuristics require an explicit semantic version and opt-in migration; they cannot reinterpret v1 publications.

---

## 9. Golden records and provenance

The engine selects each golden field independently. Source authority for identity does not automatically make that source authoritative for every output field.

Every published golden value records:

- `mdm_id`
- field
- selected value and normalized value
- winning source and source ID, when source-selected
- policy and policy version
- deterministic tie-break reason
- configuration and publication revision
- custom selector fingerprint and contributing values, when computed

`mdm.explain()` exposes this provenance subject to PostgreSQL permissions and field masking. Provenance is required even when optional detailed history has been pruned.

---

## 10. Stewardship

V1 has three stewardship operations:

```text
MATCH
NOT_MATCH
golden override
```

`MATCH` and `NOT_MATCH` attach to durable source-record identities, not temporary review rows. They survive refresh and become inactive while a source record is deleted. A golden override attaches to one `mdm_id` and field.

Stewardship writes use optimistic concurrency and require a reason. `MATCH` and `NOT_MATCH` for the same pair cannot both remain active. The engine also rejects a `MATCH` whose must-link closure would contain an active `NOT_MATCH`. It returns the shortest contradiction chain it can produce.

V1 does not expose direct merge, partitioned split, move-member, lock, unlock, or invariant-override commands. Stewards express identity corrections through pair decisions and edit or supersede those decisions when needed.

---

## 11. Output contract

### 11.1 `mdm_out.<entity>`

One row per active `mdm_id`, with typed golden fields where practical. Standard metadata includes `mdm_id`, `member_count`, `has_review`, and `publication_revision`.

### 11.2 `mdm_out.<entity>_members`

The current identity map:

```text
(source, source_id) -> mdm_id
```

At minimum it contains `source`, `source_id`, `mdm_id`, active status, first and last published revision, and a concise membership reason. Each active source record maps to exactly one active `mdm_id`.

### 11.3 `mdm_out.<entity>_review`

The review queue contains unresolved ambiguous pairs, authoritative identity conflicts, conservative bridge rejections, split notices, golden ties, invalid authoritative values, candidate incompleteness, and custom-function failures.

Each row has a stable `review_id`, status, severity, subject references, reason code, masked summary, publication revision, and optimistic concurrency version.

### 11.4 No v1 change feed

V1 does not publish `<entity>_changes`. Consumers can use PostgreSQL triggers, logical decoding, or ordinary CDC against the three output relations. A future semantic feed must define merge, split, alias, golden, review, ordering, retention, and replay behavior as one complete contract.

---

## 12. Incremental processing and publication

The engine stores source-row fingerprints, normalized values, candidate dependencies, accepted pair evidence, active decisions, current components, memberships, golden values, and publication revisions.

A changed record invalidates only dependent normalized values and candidate evidence. The affected closure includes its old component and any component reached by a new accepted candidate. Removing an accepted edge includes the complete old component because it may split. Adding an edge includes both components because they may merge.

The engine escalates to a full entity scan when:

- a `snapshot` source must prove deletions
- a definition change alters a global match rule
- the affected closure cannot be bounded safely
- stored dependency state is missing or invalid

Long-running work may use staging tables and multiple internal transactions, but publication is one short transaction. Failed work never becomes visible. V1 does not promise resumable work after process or primary failure; callers start a new refresh.

`mdm_admin.rebuild()` is the recovery escape hatch. It resolves all current source records under the desired definition and active decisions, then publishes through the same atomic path. It preserves stable IDs through the v1 reconciliation rules.

---

## 13. PostgreSQL-native architecture

V1 uses four schemas:

```text
mdm
  source, field, match, entity, golden_value
  create, describe, preview, refresh, explain

mdm_out
  <entity>
  <entity>_members
  <entity>_review

mdm_steward
  decide
  override_golden
  clear_golden_override

mdm_admin
  rebuild

mdm_internal
  extension-owned catalogs and work tables
```

Ordinary roles have no direct access to `mdm_internal`. Entry points resolve relation and function identifiers through PostgreSQL catalogs, use a fixed safe `search_path` where security definer behavior is required, and enforce PostgreSQL grants on source, output, review, and explanation data.

V1 provides grant guidance for administrators, configurators, stewards, and readers. It does not create a multi-tenant authorization system.

Raw sensitive values are kept only where golden output, review, or provenance needs them. Logs use IDs, reason codes, counts, and digests. Fields may be `masked`, `hashed`, or `full` in review and explanation output according to grants.

---

## 14. Custom functions

V1 supports three registered PostgreSQL function types:

```sql
(raw_value anyelement, options jsonb) -> mdm.cleaned_value

(left mdm.cleaned_value, right mdm.cleaned_value, options jsonb)
    -> mdm.match_evidence

(candidates mdm.golden_candidate[], options jsonb)
    -> mdm.golden_choice
```

These correspond to custom cleaners, comparators, and golden selectors. Registration verifies the exact signature, return type, volatility, ownership, parallel-safety declaration, option schema, and function definition fingerprint. Functions must be deterministic and side-effect free.

V1 custom functions may depend only on their arguments, immutable PostgreSQL functions, and pinned literal options. Relation lookups, network access, clock time, random state, custom candidate generation, and custom component checks are unsupported. Replacing a registered function body requires a new entity definition version before refresh.

---

## 15. Operations and errors

Each preview, refresh, rebuild, configuration change, and stewardship write has an operation record. Refresh records source counts, changed records, deleted records where knowable, candidates, pair classifications, affected components, entity changes, review changes, publication revision, stage timings, and failure detail.

`mdm.describe()` reports an entity as:

- `current` when all sources visible to the last refresh are published and every source has proven deletion completeness
- `current_with_unknown_deletions` when change rows are processed but at least one `changed_at` source cannot reveal hard deletes
- `pending` when known source changes exist
- `stale` when refresh failed after known changes
- `unknown` when source state cannot be checked without a scan

Errors use a stable MDM code, a concise message, a concrete consequence, and a next action. Candidate explosion, non-unique keys, unsafe custom functions, incompatible mapped columns, contradictory decisions, stale review versions, and failed publication validation stop before changing public output.

V1 relies on normal PostgreSQL backup, point-in-time recovery, streaming replication, monitoring, scheduling, and resource controls. Extension documentation must identify durable extension tables that backups must include. Automated restore verification, recovery objectives, fair scheduling, run checkpoint recovery, and extension-managed storage retention remain v2 work.

---

## 16. V1 invariants

These eight invariants define the first release more strongly than any table layout or implementation choice.

1. **Atomic publication.** Memberships, entities, golden values, reviews, and required provenance from one revision become visible together or not at all.
2. **No silent candidate incompleteness.** An incomplete bounded search produces review or failure, never proof of non-match.
3. **No implicit deletion.** Absence removes a source record only in `snapshot` mode after a complete scan. Other modes require their declared deletion signal.
4. **Deterministic, versioned semantics.** The same source state, expanded definition, function fingerprints, decisions, and identity ledger produce the same memberships, IDs, goldens, and reviews.
5. **Coherent manual decisions.** Active must-link and cannot-link decisions cannot contradict one another.
6. **Unique active membership.** Each active `(source, source_id)` maps to exactly one active `mdm_id` within an entity.
7. **Golden provenance.** Every published golden value identifies its source or deterministic computation.
8. **Resource pressure never changes truth.** Limits may delay work, broaden work, produce review, or fail. They cannot skip evidence, weaken constraints, or publish partial output.

---

## 17. Acceptance criteria

V1 is ready when a user can:

- define one entity over multiple PostgreSQL relations
- use `snapshot`, `soft_delete`, and `changed_at` sources with honest deletion completeness
- map logical fields and normalize values without changing source relations
- express exact and fuzzy identity, strong, and supporting evidence
- resolve records without an all-pairs comparison
- receive deterministic conservative clusters and stable `mdm_id` values
- query golden, membership, and review relations
- trace every golden value to its source or selector inputs
- record durable `MATCH`, `NOT_MATCH`, and golden override decisions
- preview validation, work, and representative local changes without publication
- explain records, pairs, entities, golden fields, and review items
- process inserts, updates, proven deletions, disappearing edges, local merges, local splits, and golden-only changes incrementally
- fall back to broader work when local completeness cannot be proven
- publish atomically and repeat a no-change refresh without semantic changes
- rebuild current state while preserving IDs under the documented rule set
- reject incomplete, contradictory, non-deterministic, or resource-unsafe work before publication

V1 is not blocked on any capability listed in Section 1.2. Shipping those capabilities early is allowed only when they remain private implementation details and do not enlarge the public compatibility promise.