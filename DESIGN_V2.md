# `pg_mdm` v2 design roadmap

## Advanced source, stewardship, integration, and operational capabilities

**Status:** Deferred design after the OSS v1 contract  
**Baseline:** [DESIGN_V1.md](DESIGN_V1.md)  
**Goal:** Add capabilities in response to demonstrated use without changing the five nouns, five actions, or three primary outputs

---

## 1. Relationship to v1

V2 is cumulative. It begins with the complete v1 contract and adds the capabilities in this document. V1 remains independently useful and shippable; no v2 capability is a hidden prerequisite for v1 acceptance.

The public boundary stays fixed:

```text
Five nouns
  source, field, match, entity, golden_value

Five actions
  create, describe, preview, refresh, explain

Three primary outputs
  <entity>, <entity>_members, <entity>_review
```

V2 may add optional projections, expert configuration, and privileged stewardship or administration functions. It does not add another normal workflow.

Every v2 addition must satisfy four conditions:

1. A real deployment has a use case that v1 cannot represent safely.
2. The feature has deterministic semantics and an explanation contract.
3. It composes with incremental refresh and atomic publication.
4. Existing v1 definitions retain their pinned behavior until an administrator opts into a new semantic version.

This is a roadmap, not a promise that every section ships in one release. The sequencing in Section 17 is based on technical dependencies. User demand decides whether a stage should exist at all.

---

## 2. Capability boundary

V2 is where the original broad design belongs:

- full source lifecycle and ordered event contracts
- late-event correction and valid-time history
- reusable organization fragments and nested composition
- schema inference and labeled regression tests
- broader preview and stewardship impact analysis
- advanced stable-ID continuity heuristics
- a semantic downstream change feed
- custom candidate generation and component checks
- entity-level stewardship and decision precedence
- alternate conservative clustering policies
- resolved outputs as sources with dependency propagation
- complete resolution manifests and engine upgrade classes
- deeper matching, cluster, golden, and operational diagnostics
- data-class retention and compaction
- namespace isolation, quotas, workload control, and resumable work
- backup, restore, failover verification, and formal service objectives

These features may enrich `describe()`, `preview()`, and `explain()`. They must not force ordinary users to configure engine internals.

---

## 3. Full source contracts

V1 supports three source modes and one honest deletion-completeness flag. V2 can generalize those modes into a source contract when deployments need event ordering, replay, changing row scope, or valid-time correction.

A full source contract may define:

- stable-key policy, uniqueness scope, and permitted key reuse
- supported fields, types, and collations
- source row scope and filter semantics
- change relation, tombstone feed, trigger log, snapshot token, or watermark
- deletion, deactivation, supersession, and temporary-absence behavior
- source sequence, event ID, effective time, source commit time, and observed time
- duplicate-event and late-event handling
- snapshot completeness and reconciliation boundaries
- schema compatibility and lifecycle contract version

The v1 modes become presets for this larger contract:

| V1 mode | V2 expansion |
|---|---|
| `snapshot` | Complete snapshot with a required completeness token |
| `soft_delete` | Current relation with a retained deletion predicate and ordered change boundary |
| `changed_at` | Append/update relation with unknown hard-deletion completeness |

The core deletion rule does not change. A missing row means deletion only when the active contract proves that the relevant source boundary is complete.

### 3.1 Lifecycle states

V2 may distinguish:

- active
- deleted
- deactivated
- superseded
- temporarily absent
- outside configured scope

Each state needs an explicit effect on current membership, matching eligibility, golden eligibility, and retained history. A partial extraction can never masquerade as a complete snapshot.

### 3.2 Event order and idempotency

An ordered source declares a total order, normally an event ID or a tuple such as `(effective_at, change_sequence)`. Repeated delivery of one event is a no-op. A correction names the event or source version it supersedes. The engine never substitutes PostgreSQL ingestion time for missing business order.

Watermarks state completeness, not merely the greatest timestamp observed. A publication records the exact complete boundary it consumed. Events beyond that boundary remain pending.

### 3.3 Schema evolution

V2 source validation fingerprints mapped columns, types, collations, stable-key expressions, relevant view definitions, filters, and contract versions. Adding an unrelated column is normally compatible. Changing a mapped type, key, row scope, lifecycle flag, or view meaning requires a new previewed definition.

Refresh checks the active source contract before semantic work. A broken contract leaves the prior complete publication visible and reports the last known good source boundary.

---

## 4. Publication history and valid time

V1 answers one question: what source state was visible to a published refresh transaction?

V2 may answer two separate historical questions:

1. What did `pg_mdm` publish at revision N?
2. What does the source now say was effective at business time T?

Publication history is immutable. A late correction may change the valid-time projection, but it never rewrites an earlier publication. `mdm.explain()` must label these timelines clearly.

An ordered source record version may store:

- source event ID
- source sequence
- effective interval
- observed time
- lifecycle state
- payload fingerprint
- superseded event reference

Point-in-time membership and golden projections are optional v2 outputs. If retention prevents a complete answer, the query returns an explicit incomplete-history status rather than inventing state.

This feature should ship only after at least one real source requires correction of business-effective history. Publication revision history alone remains the cheaper default.

---

## 5. Configuration composition

V1 allows versioned built-in presets with one-level composition and entity-local overrides. V2 may add reusable organization fragments after repeated definitions show what users need to share.

A fragment is a versioned partial definition. It may contain fields, source conventions, matches, entity rules, golden policies, or expert candidate hints. It is configuration, not a sixth noun and not a runtime object.

V2 composition rules are:

1. Every reference pins an exact fragment or preset version.
2. The compiler rejects cycles.
3. Conflicting scalar values require an explicit entity-local override.
4. There is no implicit last-writer-wins behavior.
5. An entity never inherits another entity.
6. Expansion produces one canonical complete definition before validation.
7. `mdm.describe()` shows the expanded value and optional per-path provenance.

Built-in presets, organization presets, inferred proposals, fragments, and direct declarations all pass through one expansion and compilation path.

Publishing a new fragment version does not mutate existing entities. Adoption creates a new entity configuration version and goes through preview and refresh.

The first implementation should allow one reusable fragment level. Nested fragment graphs belong in a later increment and require evidence that flat fragments cause harmful duplication.

---

## 6. Expanded preview and configuration testing

V1 preview validates, estimates work from a deterministic sample, and computes exact effects for named subjects. V2 may add broader semantic and operational analysis:

- schema inference that emits an ordinary proposed definition
- deterministic samples with stated selection and coverage
- exact isolated preview for a complete entity when explicitly requested
- labeled pair and entity regression fixtures
- precision, recall, conflict, and golden-value regression reports
- expected merge, split, review, and stable-ID effects
- index and temporary-storage analysis
- downstream event and entity-dependency impact
- exact local impact for proposed stewardship directives

Every result remains labeled `validation`, `sampled`, `bounded_estimate`, `exact`, or `unknown`. Global merge and split counts are not called exact unless the engine resolves the complete relevant source state.

Preview reuses the normal compiler, matching engine, clustering engine, and stable-ID reconciler in isolated staging. It does not grow a second implementation of resolution.

Schema inference never creates an entity automatically. It proposes source IDs, field mappings, cleaners, match roles, and presets with confidence and rationale. Users accept or edit the ordinary definition.

Labeled cases evaluate rules but do not become live stewardship decisions unless recorded separately through `mdm_steward`.

---

## 7. Stable-ID continuity policies

V1 uses one simple merge and split rule. V2 may improve continuity for deployments where oldest-ID and oldest-member rules cause unacceptable downstream churn.

Candidate continuity inputs include:

- active shared membership
- authoritative source members
- durable manual anchors
- entity creation time
- steward-locked canonical ID
- stable source-member order

A v2 continuity score must use fixed weights, a complete total order, and an explanation showing why one ID survived. Merge aliases and split history remain mandatory.

This is a semantic compatibility boundary. Adding a heuristic does not change active v1 definitions. It receives a new identity-policy version, appears in `preview()`, and requires explicit adoption. Rebuild under the v1 policy continues to reproduce v1 IDs.

V2 may allow a steward to lock a canonical ID before a planned merge. That lock is part of the stewardship ledger and must participate in contradiction checks.

---

## 8. Semantic downstream change feed

V1 users can apply ordinary PostgreSQL CDC to the three output relations. V2 may add `mdm_out.<entity>_changes` when consumers need entity-aware merge and split semantics.

The feed is ordered by `(publication_revision, event_no)` and published atomically with current output. Candidate event types include:

```text
ENTITY_CREATED
ENTITY_RETIRED
ENTITY_MERGED
ENTITY_SPLIT
MEMBER_ADDED
MEMBER_REMOVED
MEMBER_MOVED
GOLDEN_VALUE_CHANGED
REVIEW_OPENED
REVIEW_RESOLVED
IDENTITY_ALIAS_CHANGED
```

Each event has a stable event ID, publication revision, event number, affected IDs, relevant source members or field, reason code, causal operation reference, and non-sensitive before-and-after digests. Raw PII is absent by default.

Merge events identify one canonical successor. Split events identify every child and moved member because a split cannot be represented as a simple alias. Consumers advance by publication revision and retain a full-resynchronization path.

The feed cannot ship piecemeal. Event order, retry, retention, merge, split, alias, review, golden, and resynchronization semantics form one external contract.

---

## 9. Advanced PostgreSQL extensions

V1 permits custom cleaners, comparators, and golden selectors that depend only on arguments, immutable functions, and pinned options. V2 may add:

- custom bounded candidate-key generation
- custom component checks
- relation-backed cleaner, comparator, and selector dependencies
- declared collation, extension, function, relation, and setting dependencies

Illustrative additional contracts are:

```sql
(value mdm.cleaned_value, options jsonb) -> text[]

(left mdm.entity_summary, right mdm.entity_summary, options jsonb)
    -> mdm.entity_check
```

Registration records exact signatures, volatility, parallel safety, option schema, output cardinality, cost class, test vectors, function fingerprint, and every mutable dependency. A relation dependency says how changes are observed and whether a changed key maps to affected inputs.

If a dependency cannot be observed or localized, refresh invalidates every dependent result or refuses the function. Undeclared clock, random, network, relation, setting, or session-state dependencies remain invalid.

Custom code does not own relationship lifecycle, decision precedence, clustering order, stable-ID reconciliation, invalidation, or publication. Those remain engine responsibilities.

---

## 10. Expanded stewardship

V1 has `MATCH`, `NOT_MATCH`, and golden override. V2 may add entity-level workflow only when pair decisions no longer express real stewardship work efficiently:

- merge named entities
- split an entity into an explicit member partition
- move one member to a target entity
- lock identity, membership, selected golden fields, or the complete entity
- unlock or supersede a directive
- authorize a scoped invariant override

These operations append directives. They never edit published output tables directly. Preview computes the complete affected closure, stable-ID outcome, golden changes, review changes, and downstream events before a consequential write.

### 10.1 Decision precedence

V2 formalizes precedence:

1. non-overridable integrity conditions
2. entity and field locks
3. active stewardship directives and scoped overrides
4. entity invariants and authoritative identity conflicts
5. automatic identity, strong, and supporting evidence

Timestamps record history; they do not resolve contradictions.

The engine reduces pair and entity directives to signed constraints and membership requirements. Before accepting a directive, it checks whether the must-link closure contains a cannot-link. A rejected write returns the shortest contradiction chain available and names the directives that must be superseded.

If a source or configuration change exposes a new contradiction, the engine retains the last valid publication and opens a high-severity review. It does not silently discard a directive or invariant.

### 10.2 Review workflow

V2 review priority may use conflict severity, entity size, authoritative-source involvement, age, stable-ID impact, and downstream effects. The score must be decomposable in `explain()`.

Bulk action requires a reproducible predicate, homogeneous action, aggregate impact, representative explanations, and deterministic bounds. It cannot convert an incompletely evaluated population into automatic matches.

---

## 11. Advanced clustering policies

V1 keeps one conservative algorithm. V2 may add policies only after observed cases show that algorithm is inadequate.

Possible policies include:

- stricter multi-edge requirements for joining established components
- representative or anchor compatibility
- field-specific component invariants
- custom registered component checks

Every policy must retain:

- deterministic edge and tie ordering
- component-level `NOT_MATCH` enforcement
- authoritative identity conflict checks
- bounded component summaries
- stable-ID reconciliation
- explanation of accepted and rejected bridges

There is no unconstrained custom clustering entry point. Policy changes are semantic definition changes and require preview.

---

## 12. Resolved outputs as sources

V2 may let one stable `mdm_out` relation act as a source for another entity. The downstream definition uses the upstream `mdm_id` as its stable source ID and maps ordinary golden columns to logical fields.

The compiler records an entity dependency edge. Dependencies must form a directed acyclic graph within one administrative namespace. Creation rejects cycles and reports the shortest dependency path.

An upstream publication provides the downstream completeness boundary. The semantic change feed reports inserted, changed, retired, merged, and split identities. Field-level events localize downstream invalidation when possible; missing feed history or an unlocalizable identity change forces broader source reconciliation.

Refresh processes stale prerequisites in topological order when policy permits. Each entity publishes independently. A downstream failure cannot roll back a valid upstream publication, and every downstream resolution manifest pins the exact upstream revisions consumed.

This feature depends on the complete change-feed contract in Section 8 and the resolution manifest in Section 13.

---

## 13. Reproducibility and engine upgrades

V1 pins the expanded definition, custom function fingerprints, decisions, and identity ledger needed for deterministic current-state rebuilds. V2 may add a complete resolution manifest for audit-grade replay.

A manifest may include:

- canonical entity-definition digest
- composition manifest and exact fragment versions
- source-contract versions and source schema fingerprints
- source snapshot, transaction, or event boundaries
- upstream entity revisions
- source-record version references and digests
- parent publication and identity-ledger digest
- custom function signatures, fingerprints, and dependency boundaries
- PostgreSQL, extension, encoding, and collation dependencies
- fixed-point evidence and clustering policy versions
- stewardship decision epoch
- compiler, optimizer, and semantic engine versions

The manifest defines a stated reproducibility horizon. If source history has expired, `describe()` says which inputs are no longer replayable. It does not imply indefinite replay.

Engine upgrades are classified as:

- storage-only
- physical-plan-only with the same logical candidates
- semantic and opt-in
- mandatory correctness repair

A storage or physical-plan upgrade may preserve publication hashes. A semantic change never reinterprets an active definition. A mandatory repair records the defect, previews impact where possible, reconciles IDs through the pinned policy, and publishes an explicit repair revision.

Configuration changes may use detailed invalidation classes such as metadata-only, output-only, golden-only, normalization-local, match-local, cluster-global, source-resnapshot, rebuild-required, and incompatible. The compiler records the reason and safe fallback so `create()`, `preview()`, and `refresh()` agree.

---

## 14. Diagnostics and service objectives

V1 reports enough state to operate an entity. V2 may add scorecards for sustained production use.

### 14.1 Matching and cluster diagnostics

Per-match diagnostics may include value coverage, candidate count, candidate yield, duplicate-channel overlap, review yield, conflict rate, labeled precision and recall, expensive blocks, and records for which a rule was the only connection.

Cluster diagnostics may include size distribution, weakest accepted bridge, independent cross-component links, anchor compatibility, identity diversity, cannot-link pressure, lock count, and candidate truncation.

Golden diagnostics may include source win rates, ties, stale selection, override rate, invalid authoritative values, and provenance completeness.

Metrics are descriptive. They may recommend a new candidate plan or rule, but refresh cannot silently adopt a semantic change.

### 14.2 Freshness boundaries

With full source contracts, `describe()` can distinguish:

- `current`, all configured boundaries are proven and published
- `pending`, later complete changes are known
- `stale`, processing failed or exceeded policy
- `unknown`, a source cannot prove completeness

The status derives from source boundaries, active configuration, decisions, and publication. A recent timestamp is not proof of currentness.

### 14.3 Service-level objectives

Optional objectives may cover source-to-MDM lag, incremental refresh duration, publication availability, failed-run age, high-risk review age, change-feed retention, bounded explain latency, preview turnaround, and recovery after failover.

Objectives affect alerting and scheduling, not identity truth. An objective cannot turn an unknown source into current state or justify truncating candidates.

---

## 15. Enterprise operations

### 15.1 Namespace isolation

The default remains one namespace. Shared installations may bind an administrative namespace to owning roles, allowed source schemas, output schema, trusted function schemas, masking rules, retention, quotas, and objectives.

Every catalog key, generated relation, lock, run, review, event, and resource counter is scoped by namespace and entity. Cross-namespace matching is impossible by construction and tested through public IDs, aliases, runs, reviews, exports, and error paths.

Namespace is an operational scope, not a sixth MDM noun.

### 15.2 Workload control and resumability

V2 may manage worker count, candidate comparisons, temporary bytes, write-ahead log volume, lock wait, publication duration, and maintenance windows. These budgets may resize batches, delay work, or pause a run. They never lower evidence thresholds or publish partial output.

Run work is checkpointed by deterministic input ranges and dependency digests. Resume verifies source boundary, desired configuration, decision epoch, and function fingerprints. A mismatch supersedes or invalidates the run. Completed idempotent batches can be reused; ambiguous work is discarded.

This machinery is justified only when single-process v1 refresh cannot meet measured workload needs.

### 15.3 Backup, restore, replication, and failover

V2 distinguishes durable identity state from rebuildable work state. Durable state includes definitions, source contracts, source identities and tombstones, stewardship directives, the `mdm_id` registry, aliases, retained memberships, publication manifests, golden provenance, review decisions, and promised downstream events.

`mdm_admin.verify()` may check relation and function references, fingerprints, collation dependencies, publication hashes, membership uniqueness, directive coherence, and whether incremental state is trustworthy after restore or upgrade.

Streaming replicas expose a transactionally complete publication. A promoted replica either resumes staging under a verified run manifest or discards it. Staging never becomes published merely because failover occurred.

PostgreSQL remains responsible for backup transport, WAL retention, replication, promotion, and infrastructure recovery.

---

## 16. Storage lifecycle and data minimization

V1 relies on ordinary PostgreSQL storage management and keeps enough data for current output, stable identity, decisions, and provenance. V2 may define retention by data class:

- identity IDs, aliases, and split relationships
- active and superseded stewardship decisions
- configuration and resolution manifests
- winning golden provenance
- source-record and valid-time history
- normalized values
- non-winning raw values
- rejected evidence and explain details
- previews, failed-run staging, and operational samples

History may use validity intervals, append-only events, periodic checkpoints, and partitioned compaction. Compaction must preserve current output and every stated guarantee, including old-ID resolution, retained point-in-time queries, causal explanation, downstream replay, decision audit, and the advertised reproducibility horizon.

A retention change that removes a capability goes through preview and explicit acceptance. Storage pressure cannot silently delete evidence needed by an active promise.

Field sensitivity and retention classes control raw storage, normalized storage, review display, explain display, logs, change-feed payloads, export, and history. Exact indexes may use namespace-keyed digests when comparison semantics permit.

---

## 17. Suggested delivery order

The smallest dependency-respecting sequence is:

### 17.1 Source depth

Add full source contracts only for sources that need them. Then add ordered event handling. Add valid time last because it depends on retained source versions and explicit event order.

### 17.2 Configuration and verification

Add reusable flat organization fragments, per-path provenance, labeled tests, and richer preview. Add nested composition only if flat reuse proves insufficient.

### 17.3 Stewardship depth

Add entity merge and split, then move and lock if real workflows require them. Formalize the decision lattice before bulk actions or scoped overrides.

### 17.4 Downstream integration

Define the complete semantic change feed. Add resolved-output dependencies only after the feed has retention and split semantics.

### 17.5 Operational scale

Add resolution manifests and restore verification before resumable work. Add namespaces, quotas, fairness, and SLOs only for shared or hosted deployments.

### 17.6 Storage and compliance

Add data-class retention after the history and replay promises are stable. Add compaction only after tests prove it preserves those promises.

Each stage should remain removable until a real deployment depends on it.

---

## 18. V2 invariants

V2 retains all eight v1 invariants and adds these eight when their associated capabilities ship:

9. **Currentness is provable.** A source is current only through a declared complete boundary.
10. **A semantic manifest reproduces one result.** Worker count, batching, and physical indexes may change performance, not identity.
11. **Every semantic change has a bounded explanation.** The initiating change, affected dependencies, identity result, and output consequences are available with controlled expansion.
12. **Every definition has one canonical expansion.** Presets, fragments, inference, and declarations resolve before validation.
13. **Every mutable extension input is declared.** Undeclared dependencies cannot participate in semantic functions.
14. **Entity dependencies are acyclic and revision-bound.** Every downstream publication names the upstream revisions it consumed.
15. **Retention cannot break an active promise silently.** Compaction preserves current output and every advertised history, replay, explanation, and recovery guarantee.
16. **Custom code does not replace engine governance.** Extensions may clean, discover, compare, check, or select; the engine owns decisions, clustering, identity, invalidation, and publication.

---

## 19. Capability allocation

This table makes the release split explicit.

| Area | V1 contract | V2 addition |
|---|---|---|
| Product model | Five nouns, five actions, three outputs | No change |
| Sources | `snapshot`, `soft_delete`, `changed_at` | Full lifecycle, event order, completeness, schema policy |
| Time | Source state visible to refresh; immutable publication revisions | Effective time, late corrections, valid-time projections |
| Configuration | Built-in presets, one-level composition, local overrides | Organization fragments, nested graphs, per-path provenance |
| Matching | Exact/fuzzy and identity/strong/supporting | More diagnostics and expert bounded channels |
| Candidate discovery | Built-in bounded channels | Custom candidate-key functions with dependency contracts |
| Clustering | One conservative deterministic policy | Versioned alternate conservative policies and component checks |
| Stable IDs | Oldest ID on merge; oldest member on split | Weighted continuity, canonical locks, policy versions |
| Golden values | Built-ins plus custom selector; mandatory provenance | Relation-backed selectors and deeper history |
| Stewardship | `MATCH`, `NOT_MATCH`, golden override | Merge, split, move, locks, scoped overrides, bulk workflow |
| Preview | Validation, sampled representatives, exact named subjects | Full exact runs, labeled tests, broad impact analysis, inference |
| Incremental work | Changed records and affected closures; restart on failure | Declared mutable dependencies, checkpoints, safe resume |
| Outputs | Golden, members, review | Optional lineage, metrics, history, identity history, change feed |
| Downstream use | Ordinary SQL and PostgreSQL CDC | Semantic events and resolved outputs as sources |
| Reproducibility | Pinned definition, functions, decisions, identity ledger | Complete resolution manifest and replay horizon |
| Upgrades | Versioned definition and function changes | Storage, physical, semantic, and repair upgrade classes |
| Operations | PostgreSQL locks, backup, replication, and resource controls | Namespaces, quotas, fairness, verify, SLOs, recovery checks |
| Storage | PostgreSQL-managed retention | Data-class lifecycle, checkpoints, compaction, legal holds |
| Security | Grants, protected internals, masked diagnostics | Namespace isolation and sensitivity-driven retention |

---

## 20. V2 acceptance

The complete v2 design is acceptable when each implemented capability:

- remains reachable through the five actions or a clearly privileged stewardship or administration function
- leaves the three v1 output relations stable
- preserves all v1 definitions under their pinned semantic versions
- publishes atomically and explains its effect
- refuses incomplete or contradictory work instead of weakening truth
- has a focused migration and rollback path
- has tests for its new invariant, failure boundary, and interaction with incremental refresh

V2 is not acceptable merely because every roadmap item exists. A smaller set that solves demonstrated deployment needs and preserves the product boundary is preferable to implementing the whole list speculatively.