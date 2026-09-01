# `pg_mdm` Design Proposal

## A tiny PostgreSQL-native surface for safe, explainable, reproducible, incremental master data management

**Status:** Revised design covering the v1 engine, the strong product layer, composable definitions, and long-term operational behavior  
**Primary goal:** Make entity resolution and golden-record construction feel like an ordinary PostgreSQL capability  
**Product boundary:** Five nouns—**source, field, match, entity, golden value**—and five normal actions—**create, describe, preview, refresh, explain**  
**Requirements covered:** 82 product, correctness, lifecycle, stewardship, integration, security, and operational traits

---

## 1. Executive summary

`pg_mdm` is a PostgreSQL extension for defining a kind of real-world entity, combining records from multiple PostgreSQL relations, deciding which records describe the same entity, assigning a durable `mdm_id`, and producing one explainable golden record per entity. The product is deliberately small at its boundary and sophisticated inside. A user describes identity intent with familiar concepts such as exact, fuzzy, strong, supporting, and identity; the engine translates that intent into normalization pipelines, candidate indexes, bounded searches, evidence calculations, conflict checks, graph updates, clustering work, stable-ID reconciliation, incremental invalidation, and atomic publication. Those mechanisms exist to make the product safe and fast, but they do not become concepts that ordinary users must orchestrate.

The complete definition of an entity is a first-class, immutable, versioned value. A definition names its sources, maps each source onto logical fields, states how those fields provide matching evidence, states the invariants that keep a cluster coherent, and states how each golden value is selected. Definitions can reuse versioned fragments for common fields, source conventions, match policies, and golden policies. Built-in presets, organization presets, inferred proposals, reusable fragments, and hand-written definitions all pass through one expansion, validation, and compilation pipeline. Composition has deterministic order and explicit overrides, and `mdm.describe()` always shows the fully expanded result.

`mdm.create()` validates and stores that desired definition as a new configuration version. `mdm.describe()` presents the effective definition, its composition provenance, compiled execution plan, source-contract health, quality indicators, configuration history, current publication, freshness boundaries, resource state, and service objectives. `mdm.preview()` evaluates a proposed definition, refresh, or stewardship action without publishing it, including expected merges, splits, reviews, affected entities, identity changes, golden-value changes, labeled-test results, index work, resource risk, and downstream events. `mdm.refresh()` chooses the narrowest provably complete incremental plan and atomically publishes a complete new state. `mdm.explain()` uses one conceptual interface to explain a source record, a pair, an entity, a golden field, a review decision, or the causal chain behind a change between two revisions.

Every configured entity exposes three primary relations. `mdm_out.<entity>` contains one golden record per resolved entity. `mdm_out.<entity>_members` contains the durable mapping from `(source, source_id)` to `mdm_id`. `mdm_out.<entity>_review` contains uncertainty and conflicts that require human attention. Additional lineage, identity-history, metrics, and change-feed relations are stable projections for users who need them, but they do not displace the three primary outputs or enlarge the configuration model. Manual `MATCH`, `NOT_MATCH`, merge, split, move-member, lock, and golden-value corrections are durable stewardship directives in a separate namespace. Administrative recovery, including a full `rebuild()`, storage retention, run control, and verification after restore or upgrade, is also separate from normal operation.

A source contract is part of identity correctness rather than an ingestion convenience. Each source states what makes its key stable, whether a snapshot is complete, how deletion, deactivation, supersession, filtering, and temporary absence are represented, how events are ordered, how late or duplicated events are handled, and which schema changes are compatible. The engine never interprets a missing row as a deletion without a contract that proves that meaning. It tracks source watermarks and completeness boundaries so that “current” means the published state is known to include every eligible source change through a stated point, not merely that a refresh ran recently. A stable `mdm_out` relation can be a source for another entity. Its publication revision and change feed provide the source contract, while an acyclic entity-dependency graph propagates changed rows and fields downstream.

Resolution is reproducible as well as deterministic. Every publication records a resolution manifest containing source snapshot or watermark boundaries, source-schema fingerprints, the canonical entity definition, expanded fragment and preset versions, cleaner and comparator fingerprints, declared relation dependencies, collation-sensitive dependencies, the compiler and engine semantic versions, the stewardship-decision epoch, and the publication plan digest. Given retained source history, the same manifest, and the same durable identity-allocation ledger, the semantic result must be reproducible. An extension upgrade may improve storage or physical planning without changing identity, but it may not silently reinterpret an active configuration. Semantic engine changes are previewed, versioned, and published like configuration changes.

The safety posture is precision-first. Missing a possible match is preferable to silently merging different real-world entities. Authoritative identifier disagreements and explicit `NOT_MATCH` decisions prevent automatic merges. Pairwise evidence is not assumed to be blindly transitive: component-level constraints and conservative bridge rules are evaluated before clusters are joined. Human decisions have explicit scopes and precedence, contradictions are detected rather than resolved by hidden timestamp rules, and unsafe uncertainty becomes prioritized review. Candidate discovery and custom functions are always bounded, and any resource limit that makes evidence incomplete is visible rather than silently ignored.

Identity changes are also a downstream contract. Every semantic publication can emit ordered relational events such as entity created, member added, golden value changed, entity merged, and entity split. Old identifiers remain resolvable through deterministic merge aliases and explicit split history. Consumers can advance by publication revision instead of diffing complete tables, while `mdm.explain()` can describe both the cause of a change and its consequences. This supports warehouses, caches, search indexes, applications, and audit processes without requiring an external service or event broker.

The resulting product remains simple because complexity is deliberately placed behind the boundary. Users think about five nouns and perform five normal actions. Internally, the extension maintains composition manifests, source contracts, record versions, normalized values, candidate dependencies, accepted and rejected relationships, signed manual constraints, entity dependencies, component summaries, publication manifests, dirty closures, history intervals, resource budgets, and resumable work. Users may extend what the five nouns mean through stable contracts, but extensions do not add normal actions or take ownership of edge lifecycle, decision precedence, clustering, identity reconciliation, invalidation, or publication.

---

## 2. Product contract

### 2.1 The five nouns

The public model is intentionally limited to five nouns. A **source** is a PostgreSQL table, view, materialized view, or compatible foreign relation that supplies stable source records. A **field** is a logical attribute such as `name`, `email`, `phone`, or `tax_id`; each source can map a different physical column onto the same field. A **match** says how one or more fields provide identity evidence, using comparison semantics such as exact or fuzzy and evidence roles such as strong, supporting, or identity. An **entity** is the real-world thing being mastered, such as a customer, company, person, supplier, or product, and it owns the rules that keep its membership coherent. A **golden value** says how one output field is chosen independently from the values contributed by the entity’s members.

Terms such as candidate pair, block, edge, component, publication revision, dirty closure, and survivorship execution plan are engine vocabulary, not product vocabulary. They can appear in advanced diagnostics because they are useful when investigating performance or correctness, but a user never has to build or sequence them. A review item is an output state rather than a sixth modeling noun, and a source record is simply one identified row supplied by a source.

| Noun | What the user decides | What the engine derives |
|---|---|---|
| Source | Relation, stable ID, field mappings, change hints, authority | Snapshot strategy, row fingerprints, partitions, source indexes |
| Field | Logical name, cleaner, missing-value semantics, optional entity rule | Normalized storage, quality flags, value indexes, dependency graph |
| Match | Exact or fuzzy comparison; strong, supporting, or identity role | Candidate channels, evidence points, thresholds, blockers, pair plan |
| Entity | Name, sources, fields, matches, safe behavior | Graph state, clustering policy, stable-ID reconciliation, publications |
| Golden value | Field-level selection policy and authority | Candidate ranking, deterministic tie-breaks, lineage, recomputation |

### 2.2 The five normal actions

The normal lifecycle is equally small. **Create** stores the desired entity definition. **Describe** shows what the entity means and whether it is healthy and current. **Preview** shows what a proposed definition or refresh would do before it is applied. **Refresh** brings the published outputs up to date, selecting incremental, local, or full work internally. **Explain** answers why the system produced a particular record, relationship, entity, golden value, or review state. These actions remain stable as data volume grows; scaling does not introduce a second application-facing workflow.

| Action | Normal question it answers |
|---|---|
| `mdm.create(...)` | “What identity behavior do I want?” |
| `mdm.describe(...)` | “What is configured, what is effective, and what is the current state?” |
| `mdm.preview(...)` | “What would change, how risky is it, and how much work will it take?” |
| `mdm.refresh(...)` | “Make the published result current safely.” |
| `mdm.explain(...)` | “Why did this result happen?” |

Stewardship and recovery are intentionally outside the normal five-action path. A data steward may use `mdm_steward.decide()`, `mdm_steward.merge()`, `mdm_steward.split()`, `mdm_steward.move_member()`, `mdm_steward.lock()`, and golden-value correction functions. An administrator may use `mdm_admin.rebuild()` and repair utilities. These operations are important, but presenting them in separate namespaces preserves a small default surface and makes elevated privileges obvious.

### 2.3 Design principles

The first principle is that configuration expresses desired identity behavior rather than execution choreography. Users state what a source means, which logical fields exist, what kind of evidence a match contributes, which conflicts are dangerous, and how golden values should be selected. The compiler derives score arithmetic, candidate channels, blocking keys, indexes, dependency graphs, cluster summaries, batch sizes, and publication plans. Advanced controls may refine those derived choices, but they extend the same model instead of requiring a second, engine-oriented configuration system.

Definition reuse follows the same rule. A reusable fragment is a versioned partial definition, not a new runtime concept. Built-in presets, organization presets, inferred proposals, and direct declarations all reduce to the same canonical entity definition before validation or planning. Composition is ordered, conflicts require an explicit override, and no entity inherits another entity. The compiler pins every referenced fragment version and `describe()` presents both the flattened definition and the origin of each value.

The second principle is that source lifecycle semantics are part of the entity definition. Stable keys, snapshot completeness, deletion, deactivation, supersession, filtering, event order, late arrival, and schema evolution all affect whether a relationship still exists and whether a split is safe. `pg_mdm` therefore refuses to infer destructive meaning from absence alone. A source that cannot prove deletion completeness can still participate, but `describe()` must say that currentness is unknown or that periodic reconciliation is required, and `refresh()` must choose broader work when local processing cannot be proven complete.

The third principle is that every semantic state and every semantic change is deterministic, reproducible, and explainable. The same resolution manifest produces the same decisions, stable IDs, memberships, golden values, and review classifications. Every published value has provenance. Every merge, split, manual override, or changed golden field has a bounded causal explanation that connects the initiating source, configuration, engine, or stewardship change to the relationships and entities it affected. Explanations summarize large paths safely and permit controlled expansion rather than requiring users to read internal graph tables.

The fourth principle is conservative correctness. Uncertainty becomes review rather than silent corruption, hard conflicts and durable cannot-links prevent automatic merges, manual assertions cannot remain in a logically contradictory state, and resource caps never turn incomplete evidence into a confident answer. The engine makes the smallest safe change: normalize only changed values, reconsider only dependent relationships, recluster only complete affected neighborhoods, and republish only changed outputs whenever the dependency model proves that local work is sufficient. When that proof fails, the engine escalates visibly to broader work.

The fifth principle is that identity changes are part of the output contract. Stable current relations are necessary, but downstream consumers also need to know what changed, which old identifiers still resolve, and what a split means for a previously stored reference. Ordered publication events, identity history, explicit revision cursors, and freshness watermarks are therefore first-class relational projections, even though they do not become new configuration nouns or normal actions.

The final principle is that PostgreSQL remains the operational environment. Relations, transactions, MVCC, indexes, functions, permissions, extension migrations, backups, replication, and ordinary SQL are the foundation; no external matching service, queue, or workflow engine is required. The implementation can adapt its physical plan, storage layout, worker count, and batching strategy as volume grows, but a definition that works at ten thousand records remains the definition used at one hundred million records.

---

## 3. The entity definition is a first-class, composable value

The most important architectural choice is to make a complete entity definition an immutable value rather than a sequence of configuration mutations. The normal SQL representation uses five typed constructors named after the five nouns. These constructors build ordinary PostgreSQL values and can therefore be composed in common table expressions, stored in deployment tables, generated by application code, reviewed in version control, and passed to `mdm.create()`. A canonical JSON representation is also supported for import and export, but it is a serialization of the same typed model rather than a separate configuration system. Reusable fragments are named partial values in that representation. They disappear during expansion and do not become a sixth modeling noun.

The signatures below are illustrative rather than a commitment to a particular PostgreSQL type syntax. The product contract is that a definition is complete, portable, validated as one unit, and semantically versioned. Every fragment and preset is expanded and pinned when a configuration version is created, so later improvements to `org.standard_email`, `person`, `company`, or `product` never silently change an existing entity. Qualified PostgreSQL function signatures are resolved during creation and their definitions are fingerprinted, preventing an unnoticed function replacement from changing future results under the same configuration version.

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
                changed_at => 'updated_at',
                fields     => jsonb_build_object(
                    'name',   'display_name',
                    'email',  'email_address',
                    'phone',  'telephone',
                    'tax_id', 'vat_number'
                ),
                authority => jsonb_build_object(
                    'email', 'preferred',
                    'phone', 'preferred'
                )
            ),

            mdm.source(
                name       => 'erp',
                relation   => 'erp.customer'::regclass,
                source_id  => 'customer_no',
                changed_at => 'modified_at',
                fields     => jsonb_build_object(
                    'name',   'legal_name',
                    'email',  'email',
                    'phone',  'phone',
                    'tax_id', 'tax_identifier'
                ),
                authority => jsonb_build_object(
                    'name', 'preferred'
                )
            ),

            mdm.source(
                name      => 'registry',
                relation  => 'registry.company'::regclass,
                source_id => 'registry_id',
                fields    => jsonb_build_object(
                    'name',   'registered_name',
                    'tax_id', 'registered_tax_id',
                    'email',  null,
                    'phone',  null
                ),
                authority => jsonb_build_object(
                    'identity', 'authoritative',
                    'name',     'authoritative',
                    'tax_id',   'authoritative'
                )
            )
        ],

        fields => ARRAY[
            mdm.field(
                name    => 'name',
                cleaner => 'company_name',
                state_rules => jsonb_build_object(
                    'null', 'absent',
                    '',     'empty',
                    'N/A',  'unknown'
                )
            ),
            mdm.field(name => 'email', cleaner => 'email'),
            mdm.field(name => 'phone', cleaner => 'phone'),
            mdm.field(
                name        => 'tax_id',
                cleaner     => 'tax_id',
                entity_rule => 'maximum_one_active_value'
            )
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
                fields     => ARRAY['phone'],
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
                sources => ARRAY['registry', 'erp', 'crm']
            ),
            mdm.golden_value(
                field   => 'email',
                policy  => 'first_non_null',
                sources => ARRAY['crm', 'erp']
            ),
            mdm.golden_value(
                field  => 'phone',
                policy => 'latest'
            ),
            mdm.golden_value(
                field   => 'tax_id',
                policy  => 'prefer_source',
                sources => ARRAY['registry', 'erp', 'crm']
            )
        ],

        options => jsonb_build_object(
            'safety',             'conservative',
            'history',            'current_and_events',
            'change_tracking',    'auto',
            'diagnostic_values',  'masked'
        )
    ) AS definition
)
SELECT mdm.create(
    definition       => definition,
    expected_version => null,
    comment          => 'Initial customer identity model'
)
FROM proposed;
```

This example combines three physical relations without requiring a union view. It maps different physical columns onto the same logical fields, declares missing and unsupported values explicitly, identifies an authoritative registry, uses intent-level matching language, and selects each golden field independently. It does not specify a score formula, a trigram cutoff, an index type, a blocking key, a candidate count, a graph algorithm, or an incremental batch size. Those are compiled choices. An advanced configuration may provide carefully validated hints or custom functions, but the primary definition remains readable without them.

`mdm.create()` is also the update operation. If an entity already exists, the caller supplies its current `expected_version`; a semantically different definition creates a new immutable version, while an identical definition is an idempotent no-op. This compare-and-swap behavior prevents two administrators from unknowingly overwriting one another. A rollback does not reactivate mutable old state. Instead, the old canonical definition is obtained through `mdm.describe(..., format => 'definition')` and passed back to `mdm.create()`, producing a new version whose ancestry records that it restores earlier behavior.

### 3.1 Reusable fragments and canonical expansion

A reusable definition fragment can contain any coherent subset of source conventions, logical fields, match policies, entity rules, golden-value policies, or expert candidate hints. Common examples are `org.standard_email`, `org.standard_phone`, and `org.customer_identity`. Presets use the same mechanism. For example, the built-in `person` preset composes the built-in name, email, phone, and address fragments rather than activating a separate engine mode. An organization preset is simply another versioned composition published under an organization-controlled qualified name.

Fragments have a configuration lifecycle, not a runtime lifecycle. The advanced form of `mdm.create()` accepts a configuration package containing named fragment values and entity definitions. It stores new fragment versions and entity configuration versions in one transaction, then pins exact fragment versions while expanding each entity. Publishing a new fragment version does not mutate entities that use an older version. Adoption requires a new entity configuration, either in the same package or in a later `create()` call. `mdm.preview()` can evaluate the complete package and report every entity change before storage. This reuses the existing actions instead of adding fragment-specific commands.

An entity lists the fragments it composes and then lists explicit overrides. Composition order is significant only for provenance and ordered collections. Two fragments that assign different values to the same scalar path cause validation to fail unless the entity's override names that path. There is no implicit last-writer-wins rule, no `super`, and no inheritance from another entity. Fragments may reuse lower-level fragments, but the compiler flattens that graph, rejects cycles, and records the exact version and digest of every input. The entity body cannot depend on an unpinned floating version in an active configuration.

The canonical JSON form is illustrative:

```json
{
    "name": "customer",
    "compose": [
        "mdm.company@3",
        "org.customer_identity@7",
        "org.contact_fields@4"
    ],
    "override": {
        "sources.registry.authority.identity": "authoritative",
        "golden_values.email.sources": ["crm", "erp"]
    }
}
```

Expansion produces one complete canonical definition before any semantic validation, dependency analysis, preview, or compilation. Schema inference emits the same form, with its guesses represented as ordinary proposed declarations and composition references. Hand-written definitions, built-in presets, organization presets, and inferred definitions therefore have identical semantics once expanded. `mdm.describe(format => 'definition')` returns the fully expanded canonical value by default and can also include a composition manifest that maps each path to its source fragment or explicit override. This keeps reuse practical without forcing operators to chase layers of configuration to learn what an entity means.

---

## 4. Sources and logical fields

A source is first-class configuration, not merely a text label embedded in a pre-unioned staging relation. Each source names a PostgreSQL relation, a stable source-ID column or safe expression, optional change information, logical-field mappings, source-specific missing-value conventions, and authority. The pair `(source, source_id)` is globally stable within one entity definition and remains the fundamental external identity of a source record. Internally, the engine assigns a compact durable record identifier so that graph and history tables do not repeatedly store large text keys, but all public explanations and memberships continue to use the stable pair supplied by the user.

Different sources can map different schemas onto the same field. A CRM can map `display_name` to `name`, an ERP can map `legal_name` to `name`, and a registry can map `registered_name` to `name`. A source may explicitly mark a field as unsupported, which is different from a supported field whose current value is absent. This distinction is important for explainability, quality measurement, schema evolution, and missing-evidence semantics. A source can also state that it is authoritative for identity, preferred for a particular golden field, or merely supporting. Authority is never inferred from relation order.

A field defines the logical data type, normalization behavior, explicit value states, diagnostic sensitivity, and optional entity-level rule. The engine preserves the original source value in protected state and produces a normalized value for matching and selection. It never overwrites a source relation. The built-in value-state model distinguishes `value`, `absent`, `empty`, `invalid`, `unknown`, `redacted`, and `unsupported`. By default, absent and unsupported values contribute no evidence; empty and invalid values are quality problems rather than mismatches; unknown values contribute no positive evidence; and redacted values are excluded from diagnostics and automatic matching unless an explicitly authorized function provides a privacy-preserving comparison.

Source change detection is part of the source definition but is progressively disclosed. In `auto` mode, the compiler chooses the safest available method: an explicit change relation if supplied, a trustworthy `changed_at` watermark where possible, an optional user-approved trigger log, or fingerprint comparison of stable keys and configured fields. Deletions are detected through a change relation, tombstone feed, trigger log, or periodic key reconciliation. The engine may scan keys to prove deletion completeness even when it avoids re-normalizing and re-matching unchanged values. If a source cannot provide enough information for a safe incremental result, `refresh()` expands the work or falls back to a broader rebuild rather than claiming that the entity is current.

### 4.1 Presets and schema inference

Versioned presets provide useful definitions for common domains such as `person`, `company`, `product`, `supplier`, and `location`. They are ordinary compositions of versioned definition fragments, published in the built-in namespace and processed by the same compiler as organization-owned compositions. A preset can suggest logical fields, built-in cleaners, safe match roles, conservative entity rules, and golden-value defaults. Presets are starting points rather than opaque modes: `mdm.describe()` shows every expanded rule and identifies whether it came from a fragment, preset composition, or explicit override. Because the expanded result and its composition manifest are pinned into the configuration version, a preset upgrade appears as an explicit proposed definition change that can be previewed and tested.

Schema inference belongs to `mdm.preview()`, not to automatic creation. Given one or more source relations and an optional preset, preview inspects PostgreSQL catalog metadata, column names, types, null rates, approximate distinctness, safe samples, and recognizable value shapes. It proposes source-ID columns, logical-field mappings, cleaners, value-state sentinels, match roles, fragment references, and explicit overrides with confidence and rationale. The proposal is an ordinary definition input to the canonical expansion pipeline, not a separate inferred-definition type. It never silently applies a guessed identity field or authoritative source. PII-sensitive inference can work from type and aggregate statistics without returning sample values, and the caller can disable sampling entirely.


### 4.2 Source data contracts and lifecycle semantics

Every source has an explicit data contract. The contract defines the stable-key rule, uniqueness scope, allowed key reuse, supported fields and types, change-detection mechanism, event-ordering columns, snapshot-completeness signal, deletion model, schema-compatibility policy, and any source-level invariants. It also states whether timestamps are business-effective time, source-commit time, extraction time, or merely advisory. This information is not decorative metadata. It determines whether `refresh()` can prove that a local change set is complete, whether `latest` has a meaningful order, whether a disappeared relationship can safely trigger a split, and whether the entity can honestly be described as current.

An illustrative contract is shown below. Most sources use a preset such as `complete_snapshot`, `soft_delete`, or `ordered_change_log` and override only the details that differ. The verbose form remains available through `describe()` and exported configuration so that there is never hidden behavior.

```sql
mdm.source(
    name       => 'crm',
    relation   => 'crm.customer'::regclass,
    source_id  => 'id',
    changed_at => 'updated_at',
    fields     => jsonb_build_object(
        'name',  'display_name',
        'email', 'email_address',
        'phone', 'telephone'
    ),
    contract => mdm.source_contract(
        key_policy           => 'immutable',
        uniqueness           => 'required',
        lifecycle            => 'soft_delete',
        deleted_when         => 'deleted_at IS NOT NULL',
        event_time           => 'updated_at',
        sequence             => 'change_version',
        snapshot_completeness => 'watermark',
        schema_policy        => 'strict'
    )
)
```

Deletion semantics are intentionally richer than “row present” or “row missing.” A source can assert that a record is deleted, deactivated, superseded by another source record, temporarily absent because an extraction is incomplete, or outside the configured scope because a filter changed. Each state has an explicit withdrawal policy. A deleted record normally stops contributing current membership and golden values while remaining in history. A deactivated record may remain a member but lose eligibility for selected golden fields. A superseded record can remain historically resolvable while its replacement becomes the active source identity. Temporary absence does not remove membership. A filtered record follows the source contract’s declared scope-exit policy. When a complete snapshot is the deletion signal, the source must provide a completeness token or successful reconciliation boundary; a failed or partial extraction can never be interpreted as mass deletion.

Late-arriving and out-of-order data are resolved using a declared total order, normally an event identifier or a tuple such as `(effective_at, change_sequence)`. `observed_at` records when PostgreSQL received the event, while `effective_at` records when the source says the state became true. Replayed events are idempotent by source event ID or source version. An older event arriving after a newer event may amend valid-time history without replacing the current source state. A correction event can explicitly supersede an earlier version. If a source supplies only an unreliable timestamp, the engine may use it for diagnostics but refuses to treat it as a trustworthy `latest` order. It never silently substitutes ingestion time for business time in a golden-value rule.

Source schema evolution is checked before semantic work begins. The engine fingerprints mapped relation columns, types, collations, generated expressions, relevant view definitions, and the source-contract version. Adding an unrelated column is normally compatible. Removing or renaming a mapped column, changing a field from text to a differently interpreted type, altering a view definition that changes row scope, changing the stable-key expression, or changing the meaning of a lifecycle flag is semantic and requires a new previewed entity definition. A compatible widening cast can be accepted if it preserves normalized values under validation. An incompatible change fails before publication and leaves the previous complete MDM state visible.

Contract validation also runs during normal refresh because upstream systems can drift without a corresponding `pg_mdm` deployment. Duplicate stable keys, non-monotonic source versions, a watermark that moves backward, tombstones for unknown reused keys, unexpectedly missing columns, or a snapshot advertised as complete while violating source counts are reported with source-specific MDM error codes. The engine records the last known good contract boundary and does not silently consume data past a broken boundary. `describe()` shows which sources are proven current, pending, stale, contract-violating, or of unknown completeness and gives the concrete action needed to restore confidence.

A source contract does not turn `pg_mdm` into an ingestion system. The extension does not fetch remote APIs or own source business rules. It states the minimum facts that an already accessible PostgreSQL relation or change relation must provide so that identity resolution can be correct. A simple local table can use a small contract, while a replicated event stream or periodically replaced materialized view can describe its more complex lifecycle without changing the entity, field, match, or golden-value mental model.

### 4.3 Resolved outputs as sources

Any stable `mdm_out` relation can be used as an ordinary source by another entity definition. For example, `mdm_out.location` can supply mastered locations to a company entity, and the company output can later supply account resolution. The downstream definition uses the upstream `mdm_id` as its stable source ID and maps ordinary output columns onto logical fields. It does not need a special cross-entity join or pipeline noun.

The engine synthesizes the source contract from the upstream entity's output contract. The upstream publication revision is an ordered completeness boundary, and its change relation reports inserted, changed, retired, merged, and split identities plus changed fields. The dependency compiler records an edge from the upstream entity and publication to the downstream source. After upstream publication, it marks only downstream records and fields named by those events as dirty when the mapping permits that precision. If a change cannot be localized, it marks the complete downstream source dependency dirty.

Entity dependencies must form a directed acyclic graph within a namespace. Creation rejects self-dependencies and cycles with the shortest dependency path. `preview()` reports upstream revisions, downstream invalidation, affected entities, and propagation cost. `refresh()` evaluates stale prerequisites in topological order under their own atomic publication contracts, or stops with a clear stale-dependency result when policy forbids cascading work. Each entity still publishes independently, so a failure downstream cannot roll back a valid upstream publication. The downstream resolution manifest pins the exact upstream publication revisions it consumed.

---

## 5. The five primary actions

### 5.1 `mdm.create()`

`mdm.create()` accepts a complete entity definition, validates it, compiles it, records it as an immutable configuration version, and prepares any physical structures that can be created safely without changing published results. Validation includes source accessibility, stable-key uniqueness, field mappings, type compatibility, missing-value rules, function contracts, match completeness, conflict semantics, golden policies, temporal settings, authority contradictions, and estimated candidate pressure. Compilation turns intent-level matches into a logical candidate and evidence plan, determines field, function, relation, fragment, and upstream-entity dependencies, selects physical indexes and storage layouts, and records a deterministic plan digest. Creation fails before changing active configuration if the definition is contradictory, unsafe, computationally explosive, cyclic, or dependent on an invalid custom function or undeclared mutable input.

The stored configuration consists of the user definition, the fully expanded effective definition, its composition manifest, the compiled logical plan, the physical recommendations, dependency fingerprints, a canonical portable serialization, and a semantic digest. A new configuration version does not automatically publish new memberships. The entity remains on its previous published revision until `mdm.refresh()` succeeds. This separation permits configuration to be reviewed, previewed, promoted through environments, and rolled back without exposing partially recomputed outputs.

When an entity already exists, creation also performs configuration compatibility analysis. The semantic diff is classified as identical, metadata-only, output-only, golden-only, normalization-local, candidate-and-evidence-local, cluster-global, source-resnapshot, rebuild-required, or incompatible. The classification is derived from explicit dependencies rather than from a list of hard-coded field names, and it is stored with the configuration version so that preview and refresh agree about the necessary work. A physical-plan change that leaves the logical candidate set and evidence semantics unchanged can be applied as an optimization without changing identity; a semantic compiler or engine change must produce a new previewable version rather than silently reinterpreting the existing definition.

### 5.2 `mdm.describe()`

`mdm.describe()` is the single command for understanding an entity. Its default tabular result groups information into identity, sources, fields, matching, golden values, constraints, compiled optimization, current publication, pending work, review, quality, diagnostics, security, and history. It shows both the concise declaration and the effective expanded behavior, including fragment and preset provenance, explicit overrides, custom-function fingerprints and dependencies, generated or declared candidate channels, upstream and downstream entities, indexes, current configuration and publication revisions, last successful and failed runs, source watermarks, estimated staleness, open reviews, merges and splits, cluster-size distribution, singleton rate, field completeness, and actionable warnings. Internal table names are not required to interpret the result.

```sql
SELECT * FROM mdm.describe('customer');
```

A typical summary is designed for a human reading a SQL client:

```text
section       item                         value                         status
------------  ---------------------------  ----------------------------  --------
entity        customer                     company preset v3             ready
sources       configured                   3                             ok
matching      tax_id                       exact / identity              ok
matching      email, phone                 exact / strong                ok
matching      name                         fuzzy / supporting            ok
optimization  candidate plan               4 bounded channels            ok
publication   active revision              42                            current
source state  changes after revision        18 records                    pending
quality       entities / singleton rate     814,220 / 37.8%              healthy
review        open cases                    391                           attention
history       config version                8                             active
```

Alternate formats are still the same conceptual action. `format => 'definition'` returns the canonical portable definition for Git, CI/CD, and environment promotion. `format => 'history'` returns version ancestry and semantic diffs. `detail => 'full'` exposes compiled thresholds and physical choices for experts without making them part of the normal configuration model.

The same result also reports source-contract compliance, completeness watermarks, late-event backlog, change-feed retention, downstream publication lag, storage-retention status, tenant or namespace quotas where configured, and the resolution manifest for the active revision. Operational objectives are shown as objectives rather than as raw counters: for example, source-to-MDM freshness within fifteen minutes, no unresolved failed refresh older than one hour, and ninety-five percent of high-risk reviews handled within one business day. A status is `healthy` only when the underlying boundary is provable; a recent timestamp cannot conceal an unknown or incomplete source snapshot.

### 5.3 `mdm.preview()`

`mdm.preview()` evaluates change without publishing it. With a partial new definition, it performs schema inference and returns suggestions. With a complete proposed definition, it validates the definition, compiles its logical and physical plans, runs labeled regression cases, identifies dependency invalidation, and estimates or computes semantic changes against the active publication. With no proposed definition, it previews the work of the next refresh against the currently active definition. The output clearly marks each result as exact, bounded estimate, sampled estimate, or unknown; it never presents an estimate as a fact.

```sql
SELECT *
FROM mdm.preview(
    entity     => 'customer',
    definition => :proposed_definition,
    mode       => 'auto'
);
```

```text
validation                         valid
configuration version              8 -> proposed 9
candidate plan                     4 channels -> 5 channels
indexes                            +1 trigram index, -0
estimated candidate comparisons    31.2 million
expected merges                    +142
expected splits                    +17
expected review cases              +391
entities affected                  2,814
labeled precision                  99.3% -> 99.1%
labeled recall                     86.7% -> 91.4%
resource warning                   none
publication mode                   incremental, 2,814-entity closure
```

For small or local changes, preview can calculate an exact result in isolated run-scoped staging. For very large global changes, it may use deterministic samples and statistics unless the caller requests an exact preview and accepts the resource cost. A preview receives an identifier that `mdm.explain()` can use to inspect representative proposed merges, splits, conflicts, golden-value changes, and expensive candidate blocks. Preview state is temporary and never becomes active implicitly.

Preview is also the impact-analysis interface for stewardship actions. A review application can submit a proposed `MATCH`, `NOT_MATCH`, merge, split, move-member, unlock, or golden override as an uncommitted directive and receive the exact local membership changes, stable-ID consequences, golden-value changes, newly opened or closed reviews, violated or explicitly overridden invariants, and downstream change events that would result. Configuration previews report the same categories at the largest safe level of precision. This keeps “what will this do?” inside the normal `preview()` concept instead of creating a separate simulation product, while still allowing the eventual write to occur through the privileged stewardship namespace.

### 5.4 `mdm.refresh()`

`mdm.refresh()` means “make this entity current safely,” not “run a particular algorithm.” It snapshots the active configuration and stewardship-decision epoch, establishes a consistent source view, detects changes, chooses the narrowest provably complete recomputation plan, performs work in bounded batches, validates the proposed result, and atomically publishes one new semantic revision. The default operation is incremental. A full scan, component rebuild, or full entity rebuild is selected internally only when source limitations, configuration changes, dependency uncertainty, or graph size make local processing unsafe.

```sql
CALL mdm.refresh('customer');
```

The call records a durable run with stage timings, counts, watermarks, warnings, and failure detail. Work can be staged over multiple transactions for scale, but publication is one short transaction. Readers continue to see the previous complete revision until the new memberships, entities, golden values, review states, lineage, and history events become visible together. If nothing relevant changed, the run is recorded as a semantic no-op and the active revision does not change.

Refresh work is checkpointed and resource-governed. The engine can reduce batch size, pause between batches, yield when database pressure crosses configured limits, stop before a maintenance window closes, or resume reclaimable work after a worker or session failure. A captured change set, source event boundary, or run-owned source snapshot prevents resumed work from mixing incompatible source states. Cancellation and failure leave the active publication untouched. These controls are operational policies, not alternate matching modes, and the normal request remains `refresh('customer')`.

### 5.5 `mdm.explain()`

`mdm.explain()` is one conceptual interface with named subject arguments. A single source record explanation shows its raw and normalized field states, current membership, contributing relationships, manual directives, and history. A pair explanation shows whether the pair was discovered, the channels that found it, field-by-field evidence, conflicts, classification, and why it matched, did not match, or went to review. An entity explanation shows members, accepted connecting evidence, rejected bridges, entity rules, stable-ID lifecycle, and current review conditions. An entity plus field explains the chosen golden value and all losing candidates. A review identifier explains the uncertainty, prior decisions, current evidence, and available next actions.

```sql
-- Explain one record.
SELECT * FROM mdm.explain(
    'customer',
    source    => 'crm',
    source_id => '1234'
);

-- Explain a pair, including a pair that did not match.
SELECT * FROM mdm.explain(
    'customer',
    source          => 'crm',
    source_id       => '1234',
    other_source    => 'erp',
    other_source_id => '9817'
);

-- Explain an entity or one golden field.
SELECT * FROM mdm.explain(
    'customer',
    mdm_id => '019f...'::uuid,
    field  => 'phone'
);

-- Explain a review case or a preview result.
SELECT * FROM mdm.explain(
    'customer',
    review_id => '019f...'::uuid
);
```

When a pair was never generated as a candidate, explain performs a bounded on-demand comparison and says why the normal plan did not discover it. When records belong to the same entity through a path rather than a direct accepted pair, explain shows the path, the component checks that permitted each union, and any weak bridge warnings. Historical explanations can specify `as_of`, `publication_revision`, or `config_version` where retained history permits it.

The interface can also explain a change between publications. Given `from_revision` and `to_revision`, it reports the initiating source event, schema or definition change, engine-semantic version, or stewardship directive; the normalized values and relationships invalidated; the affected closure that was reclustered; the stable-ID reconciliation decision; the memberships, golden fields, reviews, and downstream events that changed; and the reason unrelated entities were outside the closure. Large causal graphs are summarized with deterministic cut points and expandable references so that the explanation is bounded without becoming vague.

```sql
SELECT *
FROM mdm.explain(
    'customer',
    mdm_id       => '019f...'::uuid,
    from_revision => 980,
    to_revision   => 981
);
```

A causal explanation is a product invariant rather than best-effort logging. The engine stores the dependency and lifecycle events required to explain why a published state changed. Retention policy can reduce the raw values available in an old explanation, but it must preserve the structural cause, affected identities, rule versions, and provenance references promised by the entity’s history policy.

### 5.6 The rebuild escape hatch

Normal users do not choose refresh modes. Administrators nevertheless need an explicit recovery and debugging operation, so `mdm_admin.rebuild()` recalculates an entity from all active source records using the current definition and decisions. Its default behavior preserves stable IDs through the same deterministic reconciliation used by normal refreshes. A destructive identity reset is a separate, more privileged option and is never implied by rebuild. Rebuild uses the same staging, validation, and atomic-publication contract as refresh, which makes it suitable for recovery after a software defect, major configuration change, or suspected incremental-state corruption without introducing a second output model.

```sql
CALL mdm_admin.rebuild('customer');
```

---

## 6. Output contract

Every entity produces three primary, ordinary PostgreSQL relations. They can be queried, joined, granted, indexed through supported extension mechanisms, consumed by views and materialized views, and included in normal database backup and migration workflows. Their names and core columns are stable across physical implementation changes.

### 6.1 `mdm_out.<entity>`

The golden relation contains one row per active `mdm_id`. Each configured golden field is an ordinary typed column where practical, rather than being hidden in a JSON-only payload. Standard metadata includes `mdm_id`, `member_count`, `has_review`, `publication_revision`, `last_refreshed_at`, and optional quality indicators. A golden row is published only with the matching membership and lineage state from the same revision.

```sql
SELECT * FROM mdm_out.customer;
```

```text
mdm_id   name                email             phone          tax_id
-------  ------------------  ----------------  -------------  --------
019f...  ACME Holdings AS    info@acme.test    +4712345678    NO123456
```

### 6.2 `mdm_out.<entity>_members`

The members relation is the fundamental identity map. At minimum it contains `source`, `source_id`, `mdm_id`, active status, first and last seen timestamps, publication revision, and a concise membership reason. The uniqueness contract is one active `mdm_id` per `(source, source_id)` within an entity. A source record that disappears is retained historically as inactive; it is not silently forgotten.

```sql
SELECT mdm_id
FROM mdm_out.customer_members
WHERE source = 'crm' AND source_id = '1234';
```

### 6.3 `mdm_out.<entity>_review`

The review relation exposes unresolved ambiguity without requiring access to internal tables. Review types include ambiguous pair, identity conflict, entity-invariant conflict, weak bridge, proposed merge of existing entities, split, move conflict, golden-value tie, invalid authoritative value, resource truncation, custom-function failure, and stale manual directive. Each row has a stable `review_id`, severity, status, subject references, concise reason code, masked summary, creation and update revisions, and an optimistic concurrency version. Sensitive values are shown only when the querying role has the corresponding diagnostic privilege.

### 6.4 Additional relations without a larger mental model

Lineage, metrics, and history may be exposed as `mdm_out.<entity>_lineage`, `mdm_out.<entity>_metrics`, `mdm_out.<entity>_history`, and `mdm_out.<entity>_members_history`. They are ordinary relational projections of the same entity, field, match, and golden-value model. They do not introduce new configuration concepts, and users who need only the current golden record, membership map, and review queue can ignore them entirely.


### 6.5 Downstream change feed and stable references

Current-state relations answer “what is true now,” while downstream systems usually need to answer “what changed since revision 912.” Each entity therefore exposes an optional but stable `mdm_out.<entity>_changes` relation ordered by `(publication_revision, event_no)`. It is populated in the same atomic publication as the golden, membership, review, lineage, and identity-history changes. Consumers keep their last processed publication revision and read forward; they do not have to diff complete output tables or inspect internal run state. PostgreSQL logical replication or an external CDC tool may stream the relation, but ordinary SQL is sufficient and no external broker is required.

```sql
SELECT *
FROM mdm_out.customer_changes
WHERE publication_revision > :last_revision
ORDER BY publication_revision, event_no;
```

Standard events include `ENTITY_CREATED`, `ENTITY_RETIRED`, `ENTITY_MERGED`, `ENTITY_SPLIT`, `MEMBER_ADDED`, `MEMBER_REMOVED`, `MEMBER_MOVED`, `GOLDEN_VALUE_CHANGED`, `REVIEW_OPENED`, `REVIEW_RESOLVED`, and `IDENTITY_ALIAS_CHANGED`. Each event has a stable event ID, publication revision, event number, event type, affected `mdm_id`, predecessor and successor IDs where applicable, source-record references where applicable, field name where applicable, reason code, causal operation reference, and non-sensitive before-and-after digests. Raw PII is not copied into the feed by default. A consumer with output access can join the event to the current relation or an authorized historical projection when it needs values.

Stable downstream references follow explicit merge and split semantics. An active `mdm_id` resolves to itself. After a merge, every retired identifier has one canonical successor and remains queryable in `mdm_out.<entity>_identity_history`; applications that still hold the retired ID can resolve it deterministically. After a split, exactly one child retains the old `mdm_id` according to the stable continuity rules, while the other children receive new IDs. The split event lists every child and the members assigned to it. The old ID therefore remains usable for the continuing child, but a consumer that stored business data about a member moved to another child must process the split event rather than relying on a hidden one-to-many alias.

Applications should normally persist both the `mdm_id` and the publication revision at which they observed it. The revision is not part of identity, but it gives the consumer a precise cursor for checking subsequent merge, split, and golden-value events. Change-feed retention is an explicit operational promise shown by `describe()`. Before events are compacted beyond that promise, operators must know whether registered consumers have another recovery path, such as replay from an identity-history checkpoint or a full resynchronization of the current output.

The change feed is deliberately a projection of the same entity model rather than a sixth noun. Users still define sources, fields, matches, entities, and golden values. Events exist because a correct identity system must communicate the consequences of those definitions to the rest of the database estate.

---

## 7. Normalization and explicit value states

Normalization converts source values into deterministic comparison values while preserving the original values. A normalizer returns a typed normalized value, a value state, quality flags, a version identifier, and optional non-sensitive diagnostics. Built-in cleaners cover conservative text, company name, person name, email, phone, identifiers, dates, numbers, and token sets. They avoid destructive assumptions by default; for example, jurisdiction-specific legal suffix removal or aggressive transliteration must be explicitly enabled. The same raw value and cleaner version always produce the same normalized result.

Normalized values are stored once and reused by candidate discovery, pair comparison, quality metrics, and golden selection. Each value carries a dependency fingerprint formed from its field definition, cleaner signature, cleaner options, source mapping, and source row fingerprint. A change to a phone cleaner therefore invalidates phone values and relationships that depend on phone, but it does not force name normalization or golden fields unrelated to phone. A change to only the golden policy for `name` does not regenerate candidates or recluster entities.

The value-state model prevents overloaded null semantics. `absent` means the source supports the field but did not provide a value. `unsupported` means the source has no such field. `empty` means a present representation contains no meaningful content. `unknown` is an explicit source sentinel such as “N/A” or “not known.” `invalid` means a cleaner could not produce a valid comparable value. `redacted` means a value is known to exist but must not be exposed or normally compared. `value` means a valid raw and normalized representation is available. These states are visible in `describe()` quality metrics and `explain()`, and custom cleaners must return one of the same states rather than inventing incompatible null conventions.

Raw values live in protected internal storage only when required for golden output, review, provenance, or retained history. The default diagnostic tables and server logs contain identifiers, states, reason codes, scores, and digests rather than raw PII. Retention can keep current raw values while expiring old non-winning evidence after a configured period, provided the retained lineage still proves which source record and source value produced each published golden value.

---

## 8. Matching semantics and deterministic evidence

A match combines a comparison method with an evidence role. **Exact** and **fuzzy** describe how values are compared. **Strong**, **supporting**, and **identity** describe what the evidence means. An identity match is decisive positive evidence when equal and a hard conflict when two valid authoritative values disagree. Strong evidence can support an automatic merge, usually in combination with another independent signal or an authoritative value. Supporting evidence improves confidence and helps rank reviews, but under safe defaults it cannot produce an automatic merge by itself. These terms are stable product semantics; internal numeric points are versioned implementation details shown only by `describe(detail => 'full')` and `explain()`.

The default evidence engine uses deterministic fixed-point arithmetic rather than floating-point accumulation. Each comparison yields a similarity in integer basis points, an observable-evidence amount, a strength role, a conflict result, and reason codes. The compiler expands the intent-level configuration into a documented decision table. Presets provide calibrated defaults, and an advanced user may override thresholds only through validated expert options attached to a match or entity. A supporting rule is prevented from becoming decisive merely because stronger fields are missing, because the classifier separately requires enough observed strong or identity evidence.

A representative default classification order is:

| Order | Condition | Result |
|---:|---|---|
| 1 | Integrity guard or active lock prevents the proposed change | `BLOCKED` or review |
| 2 | One internally consistent stewardship directive applies | Directed `MATCH`, `NOT_MATCH`, or placement |
| 3 | Authoritative identity values conflict without a scoped override | `CONFLICT` and review |
| 4 | Entity-level rule would be violated without a scoped override | `CONFLICT` and review |
| 5 | Observable strong evidence is insufficient | `INSUFFICIENT` |
| 6 | Deterministic automatic-match policy is satisfied | `MATCH` |
| 7 | Review policy is satisfied | `REVIEW` |
| 8 | Otherwise | `NO_MATCH` |

The same source snapshot, effective definition, compiled logical plan, and manual-decision epoch must produce the same candidate set, evidence, classifications, clusters, ID reconciliation, and golden values. Candidate top-K ties use stable source and source-ID ordering. Pair keys are canonicalized. Evidence arithmetic is fixed-point. Clustering edges and all later tie-breaks have explicit total orderings. A physical query planner may choose a different index scan without changing the logical candidate set. If automatic optimization proposes a logically different candidate plan, that plan becomes part of a new previewed configuration version rather than changing silently between refreshes. An extension upgrade that would alter cleaner, comparator, decision, or clustering semantics must retain the prior behavior version for existing definitions or require an explicitly previewed new configuration version; the meaning of an existing semantic digest cannot change in place.

### 8.1 Conflict rules and entity invariants

Conflict rules are evaluated both at pair level and component level. An identity disagreement between two records blocks their automatic pair match, but the engine also carries the distinct authoritative identity values of each component so that a seemingly harmless bridge cannot combine components whose identifiers conflict. A durable `NOT_MATCH` is a component-level cannot-link: no automatic path may place its endpoints in the same entity. Field options can express invariants such as maximum one active national ID, maximum one active record from a named registry, required agreement among authoritative identifiers, or a source-specific uniqueness rule. An advanced entity definition may attach a validated PostgreSQL component-check function for domain-specific invariants.

A normal steward `MATCH` does not silently erase a hard conflict. The review interface explains the conflict and requires an explicit forced decision by a role with elevated permission, together with a reason. Once recorded, that forced directive is durable and auditable. This preserves the rule that human decisions win while ensuring that a casual review click cannot bypass an identity invariant accidentally.

---

## 9. Efficient candidate discovery and automatic optimization

The engine never compares every record with every other record. During configuration compilation, it derives a logical candidate plan from identity and strong matches, safe supporting matches, source scopes, available value statistics, and preset knowledge. Exact identity and strong fields normally create equality channels over indexed normalized values. Fuzzy text can create deterministic nearest-neighbor channels, bounded similarity ranges, prefix or token blocks, or combinations chosen to preserve the declared semantics. Composite matches can create joint blocks such as country plus name prefix. Candidate channels are unioned into one canonical pair, and the pair records every channel that caused it to be considered.

Advanced definitions may compose candidate channels independently from final evidence rules when automatic derivation is insufficient. Alternatives between channels are logical `OR`; predicates within one channel are logical `AND`. A company definition could discover pairs through exact email, exact tax ID, postcode plus similar name, or domain plus similar company name, then evaluate every discovered pair under one unchanged evidence policy. Candidate channels decide which pairs receive evaluation. They never decide that two records match.

Candidate channels remain an expert part of a match definition or reusable fragment, not a sixth noun or a new action. A channel can use built-in exact, range, token, geographic, or bounded nearest-neighbor discovery, or a registered candidate-key function. Every channel declares source scope, completeness semantics, deterministic bounds, dependencies, and the match rules it supports. `preview()` can compare channel coverage and cost, while `explain()` distinguishes why a pair was discovered from why it matched. The compiler still derives channels by default, and it rejects a custom comparator that has no complete bounded discovery path.

The logical candidate plan is separate from its physical implementation. The same equality channel may use a B-tree join, grouped expansion, partition-wise join, or cached block-membership table. A fuzzy channel may use an appropriate PostgreSQL operator index and batch strategy. The compiler chooses indexes, generated columns, partitions, parallelism, and micro-batch sizes from relation statistics and data volume. These choices can be rebuilt or retuned without changing matching meaning. `mdm.describe()` shows a concise plan summary, while full detail exposes estimated cardinalities, indexes, block-size distributions, and the reason each channel exists.

Resource safeguards are semantic safeguards, not merely operational limits. Every fuzzy channel has a deterministic candidate bound; every exact or token block has a maximum expansion policy; every run has candidate, temporary-storage, elapsed-time, and component-size budgets; every custom function is sampled and timed in preview. A common placeholder such as `unknown@example.test` is recognized through value state or excessive block size and cannot generate billions of pairs. When a channel is skipped or truncated, the engine marks the affected evidence as incomplete. It either obtains sufficient evidence through independent complete channels, routes the affected cases to review, or fails with an actionable error. It never makes an aggressive automatic merge while pretending the omitted candidates did not exist.

Automatic optimization is conservative and versioned. `preview()` can recommend a different candidate plan when source volume or value distribution changes materially, but `refresh()` does not silently alter the logical plan in a way that changes decisions. Physical indexes may be added or replaced online where PostgreSQL permits, and their migrations are tracked like other extension-owned objects. The engine gathers diagnostics such as candidates per source record, block-size percentiles, accepted-pair yield, review yield, duplicate candidate channels, and time per channel so that an ineffective or noisy rule is visible.

---

## 10. Entity clustering without blind transitivity

Pair decisions do not by themselves define an entity. The engine maintains a durable relationship graph whose relevant current edges include manual matches, manual cannot-links, automatic matches, reviews, and previously accepted edges that may need invalidation. Clustering processes accepted relationship evidence in a deterministic order but evaluates component-level compatibility before every union. The component summary includes authoritative identity sets, source cardinalities, field invariants, locks, manual partitions, maximum-size constraints, and cohesion diagnostics.

The conservative default treats the merger of two existing multi-record components more strictly than attaching a singleton. A manual match or shared authoritative identity can be decisive. Otherwise, two nontrivial components generally require more than one independent strong cross-component relationship, or one decisive relationship plus compatibility with stable component anchors. A single supporting bridge does not automatically collapse two established entities. When the evidence is plausible but not sufficiently cohesive, the proposed bridge becomes a review item that explains the records, components, alternative paths, identity values, and expected merge effect.

This policy avoids assuming that matching is freely transitive while remaining practical. It does not require an all-pairs complete-link calculation for every cluster. The engine can maintain representative or anchor summaries, count independent cross-component edges, inspect the weakest accepted bridge, and enforce hard constraints in near-linear graph work after candidate evaluation. Advanced entity options may select a representative policy, stricter multi-edge policy, or a custom validated component check, but all policies retain deterministic ordering, cannot-link enforcement, explainability, and stable-ID reconciliation.

A cluster explanation shows which accepted edges connected the entity, which edges were rejected by conflicts, whether any member depends on a single bridge, which manual directives apply, and which component rules were evaluated. A record can therefore be explained even when it has no direct accepted match to every other member. The explanation distinguishes “same entity because of this accepted and constraint-checked path” from “all pairs independently matched,” avoiding a misleading impression of pairwise equivalence.

---

## 11. Stable `mdm_id`, membership, merges, and splits

Each active entity has a durable `mdm_id`. New entities receive a time-ordered UUID from the supported PostgreSQL identity generator. Existing IDs are reconciled after provisional clustering by comparing new components with the previous published membership state. Ordinary additions, removals, attribute changes, and golden-value changes retain the existing ID when one new component is the clear continuation of one old entity.

When several old entities merge into one component, the survivor is chosen by an explicit total order. A steward-locked canonical ID wins first. Otherwise the old ID with the greatest continuity score wins; continuity gives extra weight to durable manual anchors and authoritative source records, then considers shared active membership. Remaining ties are resolved by oldest entity creation time and finally by UUID order. Non-surviving IDs are retired but never lost: alias rows point from each historical ID to the current survivor, and a merge event records the source IDs, target ID, evidence, configuration version, decision epoch, and publication revision.

When one old entity splits into several components, exactly one child inherits the old `mdm_id`. The child with the greatest continuity score wins, with authoritative and manually anchored members weighted before ordinary membership count; stable member-key and UUID ordering resolve any final tie. Other children receive new IDs and record `split_from` relationships to the old entity. A split always produces history and normally produces review because downstream systems may care even when the split is logically correct. The algorithm is deterministic, so repeated rebuilds against the same state choose the same inheriting child.

The membership relation remains the public contract:

```text
(source, source_id) -> mdm_id
```

Internally, source-record tombstones preserve identity across deletion and reappearance. If a record disappears, it becomes inactive, its incident automatic relationships are invalidated, and its former entity is locally reconsidered. If it later reappears with the same source and source ID, the durable internal record identity is reactivated and normal evidence determines whether it rejoins its historical entity. History is retained even when the final member disappears and an entity is retired.

### 11.1 Entity-level correction and locking

Stewards need more than pair review. `mdm_steward.merge()` records a durable directive joining named entities. `mdm_steward.split()` records a durable partition of members or a rule separating specified groups. `mdm_steward.move_member()` places a source record into a target entity and records the exclusions needed to keep the placement coherent. `mdm_steward.lock()` can protect identity, membership, selected golden values, or the entire entity from automatic change. These directives are stored separately from transient review rows, survive refreshes, participate in preview, and are fully explainable.

Corrections do not directly edit published output tables. They append versioned stewardship directives and increment the entity’s decision epoch. The next refresh, or a targeted refresh initiated by the stewardship function, recomputes the affected local closure and publishes atomically. This keeps manual behavior durable without creating a second source of truth or bypassing clustering invariants.

---

## 12. Golden records, source authority, and provenance

A golden record is assembled independently field by field. The engine never assumes that one source row is the complete winner. A registry may provide the legal name and tax ID, the CRM may provide the current email and phone, and the ERP may provide a billing classification. Each golden-value specification receives the members’ raw values, normalized values, value states, source authority, quality flags, source timestamps, manual overrides, and deterministic tie-break keys.

Built-in policies include `prefer_source`, `latest`, `first_non_null`, `most_common`, `highest_quality`, `oldest_known`, `manual`, and source-plus-recency hybrids. `prefer_source` uses an explicit ordered source list or declared field authority. `latest` requires a trustworthy source timestamp and fails validation rather than substituting ingestion time silently. `first_non_null` respects source order and value states. `most_common` groups by normalized value, selects a representative raw value from the best source, and surfaces ties. A custom golden selector is a registered PostgreSQL function with a strict result contract.

Every published golden value has mandatory provenance. For a source-selected value, provenance includes `mdm_id`, field, published raw value, normalized value, winning source, source ID, source column, source value state, policy, policy version, tie-break detail, configuration version, publication revision, and run. For a computed custom value, provenance names every contributing source value and the function fingerprint. Manual overrides include the steward, reason, time, prior value, and applicable lock. Provenance is queryable through `mdm.explain()` and optionally through the lineage relation; it is not best-effort logging that may disappear after a run.

Source authority applies to both matching and golden selection but remains explicit in each context. A source authoritative for legal identity may cause conflicting tax IDs to block a merge, while another source may be preferred for current contact information. Authority can be entity-wide for identity or field-specific. `mdm.describe()` presents the effective authority matrix and warns about contradictions, such as two sources both declared exclusively authoritative for the same single-valued identifier without a conflict policy.

Golden-value recomputation is independently incremental. A changed phone value can update one entity’s phone and lineage without reselecting unaffected fields. A change to only the phone policy recomputes phone choices for affected entities without regenerating candidates or changing memberships. A membership change recomputes all golden fields for the affected old and new entities because their candidate sets changed.

---

## 13. Human review and durable decisions

Human review is a normal state of uncertain identity, not an engine failure. The review queue contains cases where evidence falls between automatic thresholds, authoritative values conflict, a weak bridge would merge established components, a previous entity would split, a source authority rule is ambiguous, a golden selection ties, a custom function fails safely, or a resource guard prevents complete automatic evaluation. Review priority is based on risk and impact rather than score alone; a proposed merge affecting many members or a change to a heavily referenced `mdm_id` can outrank a higher-scoring singleton pair.

A pair decision records `MATCH` or `NOT_MATCH`, the two durable source-record identities, reason, actor, privilege level, evidence and configuration observed at decision time, active interval, and supersession history. The decision is not attached only to a transient review row. It remains active across refreshes, becomes dormant if a record disappears, and applies again if that record returns. A `NOT_MATCH` becomes a component-level cannot-link. A `MATCH` becomes a durable positive relationship and, when required, records a separately authorized conflict override.

Review actions use optimistic concurrency. A steward supplies the review row’s current version; if a refresh or another steward has changed the evidence, status, or entity membership, the write fails with a message explaining what changed and asks the steward to review the new state. Decisions and refreshes increment a decision epoch. A refresh snapshots that epoch at the start and verifies it before publication; if relevant decisions changed during processing, it recomputes the affected closure or abandons the stale publication rather than overwriting human work.

Entity-level directives and golden overrides follow the same durable ledger model. The review output shows current actionable cases, while history preserves superseded and resolved cases without allowing them to reappear as if undecided. `mdm.explain(review_id => ...)` presents the evidence, prior decisions, impacted entities, and exact effect of each available action.


### 13.1 Decision precedence and contradiction detection

Decision precedence is explicit and is better modeled as a small lattice than as “the newest row wins.” At the top are non-overridable integrity conditions such as one active membership per source record, atomic publication, valid source identities, authorized operations, and resource limits required to complete a calculation safely. An entity lock prevents automatic or stewardship-driven change until a separately authorized unlock occurs. Active stewardship directives—pair decisions, merge, split, move-member, and scoped conflict overrides—take precedence over automatic evidence, but they must be mutually consistent and must name any entity invariant or authoritative-identity conflict they are permitted to override. Domain invariants and authoritative conflicts block automatic merging; an elevated steward may override only those declared overrideable, with a reason and a previewed impact. Automatic identity, strong, and supporting evidence is evaluated only after these higher layers have been applied.

`MATCH` and `NOT_MATCH` do not silently outrank one another. They are assertions that cannot both remain active for the same implied relationship. Before accepting a directive, the engine computes the relevant must-link closure and verifies that no active cannot-link lies inside it. Thus `A MATCH B`, `B MATCH C`, and `A NOT_MATCH C` is detected as a contradiction even though the negative decision is not attached to either positive pair. Entity-level merge, split, move, and lock directives are reduced to the same signed constraints and membership requirements for validation. A new decision that contradicts existing directives is rejected with the shortest available contradiction chain and the explicit directives that must be superseded or withdrawn.

Configuration or source changes can expose a contradiction that was not visible when a decision was made. For example, a new source mapping may reveal that two manually joined records carry incompatible non-overrideable national identifiers. The engine preserves the last valid publication, opens a high-severity decision-conflict review, and refuses to publish a state that silently discards either the directive or the invariant. Resolution requires an authorized steward to supersede a decision, correct the source semantics, or record a scoped override where policy permits it. Timestamps are history, not precedence.

Every directive has an active interval, reason, actor, privilege scope, observed configuration and evidence, optional expiry, and supersession link. `mdm.explain()` can therefore answer not only which rule won, but why another rule was inapplicable, blocked, expired, superseded, or contradicted. This formal hierarchy makes manual corrections durable without allowing them to become an opaque second matching engine.

### 13.2 Review prioritization, ergonomics, and impact

The review queue is ordered by deterministic risk and operational value rather than by similarity score alone. Priority considers the severity of the conflict, confidence interval, number and maturity of entities that would change, member count, authoritative-source involvement, identity-history impact, downstream event volume, age, service-level target, and whether a decision blocks freshness. The score is decomposable in `explain()` so a reviewer can see why one case appears before another. Installations can tune policy weights or use bounded custom prioritizers, but the default is transparent and stable across refreshes when its inputs have not changed.

A database contract cannot dictate a graphical interface, but it can make a safe interface straightforward to build. A review explanation returns side-by-side source records, normalized values and value states, differences and agreements, source authority, pair evidence, existing entity members, connecting paths, prior decisions, relevant invariants, golden values that would change, stable-ID consequences, and the allowed actions for the caller’s role. Values are masked, hashed, or fully shown according to field sensitivity and privilege. The result uses stable sections and reason codes so a SQL client, command-line tool, or dedicated review application can present the same semantics without querying internal tables.

Before committing a consequential decision, the application can call `mdm.preview()` with the proposed directive. The result shows the exact local merge, split, move, membership, golden-value, review, and change-feed effects whenever the affected closure is bounded. If the engine cannot prove a complete local impact, it says so and requires a broader exact preview or an elevated operation. Bulk review is allowed only for homogeneous cases selected by a reproducible predicate, with aggregate impact, representative explanations, and resource limits shown before commitment. A bulk action can never turn an unbounded or partially evaluated candidate population into automatic matches.

---

## 14. Incremental processing and local reclustering

Incremental processing is based on explicit dependency state rather than a collection of ad hoc shortcuts. The engine keeps durable source-record fingerprints, normalized-value fingerprints, block memberships, current accepted relationships, relevant review relationships, manual directives, current components, stable membership, golden values, and configuration dependency fingerprints. A refresh first determines which source records were inserted, changed, deleted, or reactivated. It then invalidates only the normalized values and candidate channels that depend on changed inputs.

For a changed record, the engine removes or recomputes its block memberships, invalidates its incident automatic relationships, and discovers new candidate neighbors using the compiled indexes. The affected closure contains the record’s old component, components of new candidate endpoints, components referenced by manual directives, and any component reachable through accepted relationships whose validity may have changed. Deleting an accepted edge can split a component, so the old component is always included when one of its relationships disappears. Adding a new accepted relationship can merge components, so both sides are included. The closure is expanded until the engine can prove that no relationship crossing its boundary is affected.

Local reclustering loads the still-valid accepted relationships inside that closure, re-evaluates changed relationships, reapplies manual directives and component rules, and produces replacement components only for the affected area. Stable-ID reconciliation compares those replacement components with the old entities in the same closure. Unaffected components, memberships, golden values, reviews, and histories remain untouched. If the closure grows beyond safe bounds, a candidate rule changed globally, a custom dependency cannot be proven, or a source cannot establish deletion completeness, the engine escalates automatically to a partition or entity rebuild.

Configuration changes use the same dependency graph. A cleaner change invalidates normalized values and all downstream matches using that field. A comparison or strength change invalidates its candidate/evidence channels and affected components. A clustering-rule change may require all components but does not necessarily require source renormalization. A golden policy change invalidates only the named golden field. A presentation-only output change may require only republishing a view. A changed fragment is equivalent to the expanded paths it changes; its name alone has no special invalidation behavior. `mdm.preview()` shows this dependency impact before the new configuration is activated.

Declared extension dependencies join this graph. A changed dictionary relation used by a cleaner invalidates that cleaner's outputs, even when source rows did not change. A dependency with a key-mapping contract can identify affected normalized values; a dependency that declares only relation-wide influence invalidates every use of the function. If the engine cannot prove that a mutable dependency is unchanged or cannot localize its effect, it chooses broad invalidation or rebuild. It never preserves incremental state on the assumption that an undeclared lookup table probably did not change.

After an entity publishes, its change events enter the same dependency machinery. Any downstream entity that uses the output as a source becomes pending at the new upstream publication boundary. Field-level events invalidate only mapped downstream fields where possible; merges, splits, retirement, source-key changes, or incomplete event retention trigger the broader source reconciliation required by the synthesized contract. Refresh scheduling follows the acyclic entity graph, but each entity keeps its own configuration version, source boundary, and atomic publication revision.

### 14.1 Atomic publication

Long-running normalization and matching work occurs in run-scoped staging or versioned work partitions. Published state is never updated piecemeal. Each entity has an active publication revision, and semantic rows carry validity revisions or belong to an atomically selected publication partition. The final publication transaction rechecks the expected configuration version, stewardship decision epoch, source snapshot boundary, and safety validations; applies or attaches the complete membership, master, golden, lineage, review, metric, and history delta; and advances the active revision. PostgreSQL readers see either the previous complete state or the new complete state under MVCC.

For an incremental run, the final transaction changes only the affected rows and closes their prior validity intervals. For a very large rebuild, the engine can bulk-build a new publication partition and switch the active pointer in the final transaction. These are internal strategies with the same output semantics. A failed run leaves its staging state available for bounded diagnostic retention but cannot become partially visible.

### 14.2 Idempotency

Running `refresh()` twice with no relevant source, configuration, function, or stewardship changes produces no semantic changes. Memberships, active `mdm_id` values, aliases, golden values, provenance, review states, and classifications remain identical. The second run may be recorded as an operational no-op with a new run identifier and timing, but it does not advance the active semantic revision. Deterministic content hashes allow the engine to verify this property and avoid rewriting unchanged rows.


### 14.3 Late events, valid time, and current source state

Incremental processing distinguishes the state that a source asserts is currently effective from the order in which PostgreSQL observed source changes. For an ordered change source, the engine keeps the source event ID, effective time, source sequence, observed time, lifecycle state, and payload fingerprint. Repeated delivery of the same event is a no-op. A late event is inserted into source history and evaluated under the source contract’s ordering rule. It changes current matching only when it is the effective successor or an explicit correction; otherwise it may change only a historical valid-time projection.

Publication history remains append-only even when a late event corrects the past. “What `pg_mdm` published at revision 500” is never rewritten. A separate valid-time query can answer “what the source now says was effective on that date,” using the latest knowledge available. `mdm.explain()` clearly distinguishes these two questions. If retained source history is insufficient to replay a late correction, the engine either performs the broader source reconciliation required by the contract or marks the historical result incomplete; it never invents an ordering from ingestion time.

Source watermarks express completeness, not merely maximum timestamp. An ordered event source may say that all events through sequence 8,000,000 are available. A complete snapshot may say that snapshot `2026-09-01T12:00Z` contains every in-scope key. A soft-delete table may prove current rows through a transaction or extraction boundary while requiring periodic key reconciliation for hard deletes. `refresh()` publishes the exact boundary it consumed, and `describe()` computes source-to-MDM lag from that boundary. Events beyond it remain pending for the next revision and cannot be accidentally mixed into the current one.

---

## 15. Configuration validation, semantic preview, and history

Validation runs during preview, creation, and refresh. Structural validation checks relations, columns, stable-ID uniqueness, field types, function signatures, permissions, value-state maps, source timestamps, and output names. Semantic validation checks that an entity has at least one safe candidate path, supporting evidence cannot become accidentally decisive, identity and source-authority rules are coherent, golden policies can operate on the available metadata, entity rules are satisfiable, and temporal retention is compatible with required provenance. Computational validation estimates block sizes, fuzzy-neighbor pressure, candidate volume, component size, expected temporary storage, custom-function cost, and index requirements.

Errors are expressed in MDM language. They identify the entity, source, field, or match at fault, explain the consequence, and give a concrete next action. A stable MDM error code is carried in structured detail in addition to the PostgreSQL error fields. For example:

```text
ERROR: customer.email would create an unbounded fuzzy candidate channel
DETAIL: The largest estimated block contains 1,240,818 records and the run
        would evaluate approximately 4.8 billion pairs.
HINT: Use an exact strong email match, add a safe composite match, mark known
      placeholders as unknown, or inspect mdm.preview('customer').
MDM-CODE: MDM_CONFIG_CANDIDATE_EXPLOSION
```

Configuration history is immutable and relational. Each version stores its canonical definition, composition manifest, compiler version, logical-plan digest, custom-function and declared-dependency fingerprints, creator, time, comment, parent version, and semantic diff. `mdm.describe(format => 'history')` shows the sequence and operational outcomes. Rollback creates a new version from an old exported definition so that history remains append-only and audit-friendly. `mdm.describe(format => 'definition')` exports a canonical portable representation with a format version, resolved fragment and preset versions, effective evidence semantics, logical candidate-plan policy, and qualified relation and function names; `mdm.create()` consumes the same representation. Physical index choices and batch sizes are deliberately omitted because they may adapt to each environment without changing the logical candidate set. Import resolves local relations, fragments, and functions, verifies their fingerprints, and preserves the semantic digest. This supports Git, code review, CI/CD, repeatable environments, and promotion without adding separate import and export concepts.

A configuration can include labeled test cases or reference a versioned set maintained by stewards. Pair labels state known `MATCH` and `NOT_MATCH` examples; entity fixtures can state expected groups, required separations, and expected golden values. Preview runs the proposed definition against those labels and reports regressions, precision, recall, conflict behavior, and changed explanations. Labeled cases never become automatic production decisions unless they are also recorded as stewardship directives; this separation permits test data to evaluate rules without unexpectedly changing live entities.


### 15.1 Configuration compatibility and invalidation classes

Every proposed definition change is assigned a compatibility class before it can become desired configuration. `identical` means the canonical semantic digest did not change. `metadata_only` changes comments or labels. `output_only` changes a non-identity projection. `golden_only` changes value selection or presentation without changing membership. `normalization_local` invalidates named field values and every dependency downstream of them. `match_local` changes a candidate or evidence rule and requires regeneration of its channels plus reclustering of the complete affected closure. `cluster_global` changes a component rule whose effect cannot be bounded by stored dependencies. `source_resnapshot` changes key, row-scope, deletion, or ordering semantics. `rebuild_required` applies when correctness cannot be proven through existing dependency state. `incompatible` means the proposed definition cannot be safely executed at all.

The classification is part of the compiled definition and includes the exact reasons, dependency edges, estimated scope, stable-ID risk, physical work, and safe fallback. It is used consistently by `create()`, `preview()`, and `refresh()`. A golden-policy change cannot accidentally regenerate a fuzzy candidate graph, while a stable-key or source-scope change cannot masquerade as a small incremental update. An administrator may request broader work than required for verification, but may not force narrower work than the compiler can prove correct.

### 15.2 Reproducible resolution and engine upgrade stability

Each semantic publication stores a resolution manifest. At minimum it contains the entity-definition semantic digest, composition manifest with expanded fragment and preset versions, source-contract versions, source schema fingerprints, source snapshot or event watermarks, consumed upstream entity revisions, a digest and retained references for the active source-record versions, the parent publication and durable identity-allocation ledger digest, custom-function signatures, body fingerprints, and declared dependency boundaries, PostgreSQL and relevant extension semantic versions, database encoding and collation dependencies, fixed-point evidence rules, the logical candidate-plan digest, component-rule version, stewardship-decision epoch, any deterministic seeds used only for non-semantic preview or diagnostic sampling, and the `pg_mdm` compiler and semantic-engine versions. Production matching and clustering never depend on sampled evidence. Operational details such as worker count may differ without changing the manifest’s semantic portion.

When retained source history and the durable identity-allocation ledger are sufficient, a rebuild under the same manifest must reproduce memberships, stable-ID decisions, golden values, lineage, review classifications, and change causes. When source history has been compacted, the engine can still verify the retained publication hashes and explain which inputs are no longer replayable. `describe()` reports the reproducibility horizon rather than implying indefinite replay. This makes a historical explanation auditable and prevents a function with the same SQL name but a different body from silently changing future results.

Extension upgrades are classified independently from entity-definition changes. A storage migration can rewrite internal representation while preserving publication hashes. A physical optimizer improvement can choose a cheaper index or batch strategy only if it proves the same logical candidate set and evidence. A semantic engine change—such as a new built-in normalizer, comparison formula, bridge policy, or stable-ID tie-break—receives a new semantic version and does not reinterpret active configurations automatically. The old publication remains valid, while `preview()` shows the effect of opting into the new semantics. Where practical, compatibility kernels keep existing configurations on their pinned behavior; where that is impossible, the upgrade blocks activation until an administrator previews and publishes the required migration.

A correctness repair may be mandatory when an old engine version is known to have produced invalid state. Even then, the extension records the defect identifier, previews the affected scope where possible, uses normal stable-ID reconciliation, publishes atomically, and emits explicit repair events. Upgrade tooling never resets identities merely because internal tables changed.

---

## 16. Quality metrics and matching diagnostics

Quality is measured at source, field, match, entity, golden-value, review, and operational levels. The standard entity scorecard includes source-record count, active and deleted records, entity count, singleton count and rate, cluster-size percentiles and maximum, entities by source composition, automatic match rate, review rate, conflict rate, manual-decision rate, merge and split counts, open and aging reviews, golden-field completeness, invalid and unknown value rates, provenance completeness, and currentness. Metrics are recorded by configuration and publication revision so that changes can be compared over time.

Matching diagnostics focus on whether each rule contributes useful evidence. For every match, the engine reports value coverage, candidate count, candidate yield, duplicate-channel overlap, automatic-match yield, review yield, conflict rate, acceptance rate of reviewed cases, labeled precision and recall, average marginal contribution, expensive blocks, and records for which the rule was the only connection. A rule that creates millions of candidates but no accepted matches is visible as noise. A field that is frequently missing or invalid is visible as weak coverage. A supporting rule that is present in many false-positive labeled cases is flagged even if its average similarity looks high.

Cluster diagnostics include size distribution, weakest accepted bridge, number of independent cross-component links, anchor compatibility, identity diversity, cannot-link pressure, manual-lock count, and entities affected by candidate truncation. Golden diagnostics show which sources win each field, tie frequency, stale-value selection, override rate, and fields whose authoritative source is often invalid. `mdm.describe()` presents the important summary and recommended actions; `mdm.explain()` provides the record-level details. The metrics relation supports dashboards without requiring direct access to engine tables.

Metrics are descriptive rather than a hidden self-tuning model. Automatic optimization may recommend a safer or cheaper compiled plan, but a change that could alter candidate semantics is previewed and versioned. This preserves determinism and user control while still allowing the product to identify ineffective configuration and propose concrete improvements.

---

## 17. PostgreSQL-native physical architecture

The extension owns five schemas with clear boundaries. `mdm` contains the five normal actions, the five typed noun constructors, and read-only catalog projections. `mdm_out` contains generated entity outputs. `mdm_steward` contains durable human-decision and correction operations. `mdm_admin` contains rebuild, repair, retention, and privileged maintenance operations. `mdm_internal` contains catalogs, current state, history, indexes, staging metadata, and implementation functions, with no direct access for ordinary roles. An optional `mdm_ext` schema can contain registration helpers for trusted custom PostgreSQL functions.

The logical internal model includes immutable configuration versions; source definitions; durable source records and tombstones; raw and normalized field values; value states and quality flags; candidate-plan channels and block memberships; current and historical relationship evidence; manual directives; component summaries; master entities and aliases; current and historical memberships; golden values and provenance; review items; labeled cases; run records; publication revisions; metrics; and dependency fingerprints. The exact physical layout is not a public contract. A small entity may use compact shared tables. A large entity may receive generated partitions, typed field columns, dedicated indexes, or run partitions. Output views hide that choice.

PostgreSQL performs the set-oriented work it already handles well: source reads, exact joins, index-assisted fuzzy searches, grouping, ranking, window-based golden selection, MVCC publication, permissions, and history queries. Performance-sensitive normalization, compact fixed-point comparison, component checks, and union-find clustering may be implemented in native extension code where benchmarks justify it. The design does not require a background worker, planner hook, custom storage engine, external queue, or external coordination service. Multiple workers may cooperate within an explicit refresh by claiming internal batches through ordinary PostgreSQL locking, but refresh remains a caller-initiated database operation.

Extension migrations preserve durable configuration, identities, decisions, aliases, memberships needed for continuity, golden overrides, and history. Ephemeral candidate work and run staging can be recreated. Backup and restore tests must prove that a restored database returns the same active mappings and can continue incrementally without assigning new IDs. Relation and function references are stored in portable qualified form in exported definitions and resolved safely on import, while internal object identifiers are repaired during extension updates.


### 17.1 Backup, restore, replication, and disaster recovery

The design distinguishes durable identity state from rebuildable working state. Durable state includes entity definitions and history, source contracts, source-record identities and lifecycle history needed for continuity, active manual directives and their supersession chains, the `mdm_id` registry, aliases and split relationships, active and historical memberships promised by retention, publication manifests and pointers, golden values and required provenance, review decisions, ordered downstream events, and enough run metadata to verify the active revision. Current accepted relationships and dependency indexes are durable when needed for incremental correctness, although they can be regenerated by a verified rebuild. Candidate scratch tables, temporary sort state, preview staging, and expired failed-run work are rebuildable.

A normal PostgreSQL physical backup or point-in-time recovery must restore the active publication exactly, including the same source-to-`mdm_id` mapping and identity aliases. After restore, `mdm_admin.verify()` resolves relation and function references, checks fingerprints and collation dependencies, verifies publication hashes and uniqueness invariants, confirms that no directive is contradictory, and establishes whether incremental state is trustworthy. If only rebuildable state is damaged, the system can reconstruct it without assigning new IDs except where the current source data genuinely requires a deterministic split or merge. If required source history is unavailable, verification states the limitation and the recovery plan explicitly.

Streaming replication and failover treat `pg_mdm` like other transactional PostgreSQL state. Read replicas can serve a transactionally consistent published revision. Refresh, review, and configuration writes occur on the writable primary, and a promoted replica continues from the replicated publication pointer and run state. Staging work that was in progress at failover is either reclaimable under its run manifest or discarded; it cannot be mistaken for published output. Disaster-recovery objectives for publication recovery, source replay, and review continuity are reported with the entity’s service objectives rather than left as undocumented operator assumptions.

---

## 18. Extensibility through PostgreSQL functions

Custom behavior uses PostgreSQL functions rather than a separate plugin framework. A field can name a custom cleaner. A match can name a custom comparator and, where necessary, a custom bounded candidate-key function. An entity can name a component-check function for domain-specific constraints. A golden value can name a custom selector. The functions use small, versioned contracts and return extension-defined composite types, so the engine can validate results, preserve explanation, and track dependencies.

Illustrative contracts are:

```sql
-- Raw input to a normalized value, state, quality, and safe diagnostics.
(raw_value anyelement, options jsonb) -> mdm.cleaned_value

-- Two cleaned values to fixed-point similarity and reason codes.
(left mdm.cleaned_value, right mdm.cleaned_value, options jsonb)
    -> mdm.match_evidence

-- One cleaned value to a bounded set of deterministic candidate keys.
(value mdm.cleaned_value, options jsonb) -> text[]

-- Two component summaries to allow, reject, or review a proposed union.
(left mdm.entity_summary, right mdm.entity_summary, options jsonb)
    -> mdm.entity_check

-- Ordered field candidates to one value and one or more provenance references.
(candidates mdm.golden_candidate[], options jsonb) -> mdm.golden_choice
```

Registration resolves a qualified `regprocedure`, verifies the exact signature and return type, checks volatility and parallel-safety declarations, rejects unsafe `SECURITY DEFINER` behavior by default, stores a function-definition fingerprint, records every semantic dependency, and executes a bounded preview sample. Cleaners, comparators, candidate-key functions, and component checks must be deterministic and side-effect free. A function body change is detected as a dependency change and requires a previewed configuration version before refresh. A custom comparator does not automatically solve candidate discovery; it must either reuse an existing complete channel or provide a bounded candidate-key contract.

The registration record is the deterministic extension contract. It declares null and value-state behavior, collation and locale assumptions, expected cost class, maximum output cardinality, parallel-safety, leakproof expectations where relevant, accepted option schema, semantic version, test vectors, and dependencies on relations, columns, functions, collations, extensions, or approved settings. A relation dependency also declares how change is observed and whether a changed key can be mapped to affected function inputs. For example, `company.clean_legal_name(text)` can depend on `company.company_suffix_dictionary`; a dictionary change then invalidates every dependent normalized name unless the registration provides a safe narrower mapping.

The engine stores the function definition fingerprint, registration digest, and dependency boundaries in each configuration and resolution manifest. Merely replacing a function under the same name cannot preserve semantic identity. Native or procedural implementations are held to the same contract. Static SQL dependencies are verified where PostgreSQL exposes them, and trusted registration must attest to dynamic lookups that cannot be discovered. A function whose output depends on clock time, random state, undeclared table contents, unapproved session settings, or network access is rejected for semantic use. If a declared relation has no trustworthy change contract, refresh treats its contents as a global dependency and proves it unchanged or invalidates all dependent results.

Resource controls apply to custom functions through per-call and per-run budgets, local statement and lock timeouts where applicable, sample benchmarks, output cardinality limits, memory limits in native code, and fail-safe classification. Custom-function failures never produce partial output or an automatic merge based on missing evidence. Administrators can restrict registration to approved schemas and owners, while ordinary configurators can use only registered functions.

Extension contracts stop at five constrained decisions: clean a value, generate bounded candidate keys, compare two values, check a proposed component union, or select a golden value. The engine retains ownership of relationship lifecycle, decision precedence, clustering order, stable-ID reconciliation, dependency invalidation, and atomic publication. New registered implementations, and later new implementation kinds that obey these boundaries, appear as declarative options inside entity definitions. They do not add normal public actions. This provides extensibility without making extension loading, service deployment, or a heavyweight plugin lifecycle part of the product.

---

## 19. Concurrency and correctness

Only one publication for the same entity can commit at a time, enforced by an entity-scoped advisory lock or equivalent row lock. Different entities can refresh concurrently. Work inside one refresh can be parallelized in bounded PostgreSQL batches, but every batch is associated with the same source snapshot boundary, configuration version, compiler plan, and decision epoch. Source writes that commit after the snapshot are not inconsistently mixed into the run; they remain pending for the next refresh and are shown by `describe()` when the source’s change contract can detect them.

Configuration changes use `expected_version`, and stewardship writes use an expected review or directive version. Refresh publication revalidates both the configuration version and decision epoch. If either changed, the engine determines whether the change is unrelated to the current closure; otherwise it recomputes the affected work or marks the run superseded. It does not publish a result calculated before a new `NOT_MATCH`, merge, split, move, or lock directive. Entity-level correction functions can initiate a targeted refresh, but they still use the same publication protocol.

Internal workers claim batches with `FOR UPDATE SKIP LOCKED` or equivalent ordinary PostgreSQL coordination, record heartbeats in run state, and make completed batches idempotent. A worker crash leaves reclaimable work rather than an ambiguous half-decision. The final publication transaction validates counts, uniqueness of active memberships, absence of violated cannot-links and invariants, completeness of golden provenance, and consistency of stable-ID aliases before advancing the active revision.

---

## 20. Security and PII-aware diagnostics

The installation does not assume one universal role model or create cluster-wide roles silently. It provides documented grant templates for four responsibilities: administration, configuration and refresh, stewardship, and read-only consumption. Administrators can install and migrate the extension, register trusted functions, rebuild entities, and manage retention. Configurators can create definitions, preview, refresh, and describe. Stewards can inspect authorized review evidence and record decisions or corrections. Readers can query selected golden and membership outputs, with lineage or review access granted separately.

All dynamic relation and column references are resolved through PostgreSQL catalog identifiers rather than string interpolation. Security-definer functions use a fixed safe `search_path`, schema-qualified internal objects, explicit privilege checks, and minimal execution rights. Custom functions normally execute as security invokers. The internal schema is revoked from `PUBLIC`, and output relations can be granted independently by entity. Explain operations enforce the same source and field sensitivity rules as outputs rather than becoming a back door to protected values.

PII is minimized in logs, diagnostics, and history. Server logs use entity IDs, source names, stable source IDs when permitted, reason codes, counts, and digests rather than raw names, emails, phones, or identifiers. Review and explain output supports `masked`, `hashed`, and `full` diagnostic classes per field and role. Redacted source values remain explicitly redacted. Evidence tables store scores, states, comparator versions, and provenance references; raw historical values are retained only under the entity’s retention policy and protected grants. Golden provenance retains enough information to identify the exact winning source value without copying all losing values indefinitely.

Resource safety is also part of the security model. Candidate explosions, pathological components, recursive history queries, untrusted custom functions, and accidental full rebuilds can otherwise become denial-of-service paths. Preview estimates work, refresh enforces budgets, large operations require explicit administrator privilege when they exceed configured limits, and every refusal includes a concrete next action.


### 20.1 Namespace and multi-tenant isolation

A single installation can host many independent entities and, where required, many administrative namespaces. Namespace is an operational scope comparable to a PostgreSQL schema, not an additional MDM modeling noun; users inside one namespace still define only sources, fields, matches, entities, and golden values. The default installation has one namespace. Hosted or shared installations can bind a namespace to owning roles, allowed source schemas, an output schema, trusted extension schemas, retention policy, encryption or masking policy, concurrency quota, temporary-storage budget, and service objectives.

Every internal key, catalog lookup, advisory lock, generated relation, run, review, event, and resource counter is scoped by namespace and entity. Cross-namespace candidate discovery and clustering are impossible by construction rather than merely discouraged by grants. Security-definer entry points determine namespace from an authorized catalog mapping, never from an untrusted `search_path`. Row-level security may provide additional defense, but explicit namespace predicates and role checks remain mandatory. Tests include attempts to explain, review, export, or consume another namespace’s records through guessed IDs, aliases, run IDs, and error paths.

Resource isolation is part of tenancy. One namespace cannot consume all candidate budget, workers, temporary storage, review retention, or publication lock time. Administrators can reserve capacity or establish weighted fairness while preserving each entity’s deterministic semantic plan. A resource delay may make an entity pending, but it cannot cause evidence to be truncated silently or permit one tenant’s load to alter another tenant’s matching decisions.

### 20.2 Data minimization as an architectural rule

The extension stores only the identity data needed for current operation, promised explanation, historical policy, and recovery. Exact candidate indexes may use namespace-keyed digests where the comparison semantics allow it. Rejected candidate pairs normally retain reason codes, scores, rule versions, and source references rather than duplicate raw PII. Non-winning raw values can expire earlier than winning golden provenance. Golden lineage can point to a stable source-record version and value digest, with a protected copy retained only when required to reproduce a historical published value. Fuzzy comparison may require normalized representations, but their retention, access, and diagnostic display are field-specific rather than universally permanent.

Every field has a sensitivity class and a retention class. These control raw-value storage, normalized-value storage, explain visibility, review visibility, change-feed payloads, log redaction, export, and history. Data minimization does not weaken correctness: a policy that would make promised provenance or deterministic replay impossible is rejected or shown as reducing the reproducibility horizon. Purge and compaction are transactional, auditable, and reflected by `describe()` so an operator knows which historical questions remain answerable.

---

## 21. Operational observability and excellent errors

Every refresh, preview, rebuild, configuration change, and stewardship action has a durable operation record. A run reports queued, running, validating, publishing, succeeded, failed, cancelled, no-op, or superseded state; source snapshot boundaries; configuration and decision versions; records inserted, changed, deleted, and reactivated; values normalized; candidates generated; pairs classified; components reclustered; entities affected; IDs created, merged, split, retired, or aliased; golden fields changed; review items opened and closed; rows published; temporary storage; and stage timings. Errors preserve a stable MDM code, PostgreSQL error information, affected subject, and safe diagnostic context.

`mdm.describe()` answers “Is MDM current?” without forcing the user to infer it from timestamps. For each source it reports the last processed watermark or reconciliation boundary, detected pending changes, whether deletion completeness is known, and the time of the active publication. The entity is `current` only when all configured source contracts prove that no eligible change precedes the stated boundary. It is `pending` when known changes exist, `stale` when a refresh failed after changes, and `unknown` when a source cannot establish currentness without a scan. This prevents a recent successful run timestamp from being mistaken for proof that the outputs include all source changes.

Errors follow a consistent shape: a concise message in MDM language, detailed cause and consequence, and a concrete hint. Examples include a non-unique `(source, source_id)`, an unsupported field mapping, an identity rule that contradicts a manual merge, a source timestamp that cannot support `latest`, a custom cleaner whose volatility is unsafe, a candidate channel expected to explode, a component exceeding the safe clustering limit, a stale review version, or a publication superseded by a new decision. `describe()` carries unresolved warnings and recommended actions after the immediate error has passed.

Operational state is queryable through ordinary relations for dashboards and alerting, but the common path remains one `describe()` call. No external monitoring service is required to establish correctness, although standard PostgreSQL monitoring systems can consume the run and metric views.


### 21.1 Freshness guarantees and service-level objectives

Freshness is expressed as a boundary and a lag, not just a completion timestamp. For each source, the active publication records the complete snapshot token, source sequence, transaction boundary, or event-time watermark through which changes are known to be included. It also records whether deletion completeness was proven. Entity currentness is the intersection of those source guarantees plus the active configuration and decision epochs. `current` means all eligible changes through the displayed boundaries are included; `pending` means later changes are known; `stale` means processing failed or exceeded policy; `unknown` means the source contract cannot prove completeness without additional reconciliation.

Service-level objectives are optional operational policy associated with an entity or namespace and shown through `describe()`. Standard objectives include maximum source-to-MDM lag, maximum duration of a normal incremental refresh, publication availability, maximum unresolved failed-run age, maximum high-risk review backlog and age, change-feed retention, explain latency for bounded subjects, configuration-preview turnaround, and recovery objectives after failover or restore. Objectives do not alter identity semantics. They set alert thresholds, prioritization inputs, workload scheduling, and operator expectations.

An objective is evaluated only from meaningful evidence. A source-to-MDM lag cannot be green when a source has no completeness watermark. A review backlog objective distinguishes cases that block safe publication from informational reviews. A refresh-duration objective does not encourage silent candidate truncation. Breaches are retained as operational events with their cause, affected revisions, and recommended action, allowing teams to distinguish inadequate capacity from a bad match rule, broken source contract, blocked decision, or pathological cluster.

---

## 22. Temporal and historical behavior

The current outputs are simple, but the internal model is revision-aware from the beginning. Configuration versions, decision epochs, source-record activity, relationships, memberships, entity status, aliases, golden values, lineage, reviews, and metrics all carry publication or validity ranges. Current relations project the rows valid at the active publication revision. History relations expose intervals and events without duplicating every unchanged row for every refresh.

This model supports point-in-time questions such as which `mdm_id` a source record belonged to, which golden phone was published, why two records were considered the same, and which old IDs were aliases at a specified revision or timestamp. `mdm.explain(..., as_of => ...)` uses retained source and evidence history when available and clearly states when retention prevents a complete historical explanation. Historical `mdm_id` values remain resolvable through merge aliases and split relationships rather than being overwritten.

Retention is configurable by data class. Identity lifecycle, manual decisions, configuration history, and winning golden provenance are durable by default. Non-winning raw PII and low-value rejected evidence can expire earlier. Compact event history makes temporal support compatible with large incremental systems while leaving room for future point-in-time golden and membership relations, audit exports, and temporal joins.


### 22.1 Storage lifecycle, retention, and compaction

Storage lifecycle is defined by data class rather than by a single global age. Identity IDs, merge aliases, split relationships, active manual decisions, decision supersession, configuration manifests, and the minimum publication history required to resolve old references are durable by default. Winning golden provenance and review audit history follow the entity’s regulatory and business requirements. Normalized values, non-winning raw values, rejected relationship evidence, candidate details, explain expansions, failed-run staging, previews, and operational samples can have shorter retention. The default favors retaining structural causes and digests while minimizing duplicated PII.

History uses validity intervals, append-only events, periodic checkpoints, and partitioned compaction so unchanged rows are not copied for every publication. A checkpoint can summarize a complete identity state at revision N while later revisions store deltas. Compaction must preserve current output, promised point-in-time queries, old-ID resolution, causal change explanations, downstream replay guarantees, manual-decision audit, and rebuild inputs within the stated reproducibility horizon. Before a policy change reduces one of those guarantees, `mdm.preview()` reports the lost capability and `mdm.create()` or the retention administrator requires explicit acceptance.

Retention work is bounded and concurrent with normal reads. It closes or detaches eligible partitions, verifies that no active publication, preview, consumer guarantee, legal hold, or recovery checkpoint references them, and then removes or anonymizes data according to policy. Storage metrics report current size by class, growth rate, compaction backlog, projected exhaustion, and the entities or rules responsible for disproportionate growth. A storage limit cannot silently delete evidence needed for a published explanation; the engine must compact safely, request more capacity, narrow an optional retention promise, or stop before correctness is compromised.

---

## 23. Scaling without changing the mental model

A definition that works for ten thousand records remains the definition used for one hundred million records. The user does not replace `refresh()` with a sharding workflow, hand-build a candidate graph, or rewrite strong and supporting matches as physical blocks. The compiler can change internal representation from shared tables to dedicated partitions, introduce per-source or hash partitioning, choose typed wide storage or compact value tables, build additional indexes, parallelize normalization and candidate generation, spill graph work to relations, and process micro-batches. Those choices are described and previewed but do not become application concepts.

Performance work is concentrated where it matters: normalizing changed values once, bounding candidate discovery, avoiding duplicate candidate work, using compact internal record IDs, persisting current accepted relationships, reclustering only affected closures, selecting golden values set-wise, and keeping publication transactions short. Memory is bounded by batch and component budgets. Pathological exact blocks, fuzzy neighborhoods, or giant connected components are detected before or during execution and handled through safer alternate channels, review, partition rebuild, or an actionable refusal.

The benchmark program should use the same public definitions at representative tiers such as 10,000, 1 million, 10 million, and 100 million source records. Results must report hardware, source distribution, candidate density, cluster distribution, changed-record fraction, exact and fuzzy mix, index build time, full rebuild time, incremental time, publication latency, temporary storage, and explain latency. The design does not promise that every arbitrary 100-million-row configuration fits every PostgreSQL server; it promises that scale is addressed through compiled physical mechanisms and resource validation rather than a different product model.

At very large scale, source contracts become especially important. A reliable change relation can avoid scanning all source values, while a source without deletion information may require periodic key reconciliation. The engine makes that operational cost visible in `describe()` and `preview()`. It never asks the user to understand dirty-component closure or micro-batch mechanics merely to keep the entity current.


### 23.1 Backpressure, workload control, and resumability

Large identity workloads must coexist with transactional applications and other analytical work in the same PostgreSQL installation. Each namespace and entity can therefore have operational budgets for concurrent workers, CPU-oriented function calls, candidate comparisons, temporary bytes, write-ahead log volume, lock wait, publication duration, and maintenance windows. The engine uses these budgets to size batches, choose worker count, yield between stages, or leave safe pending work. It does not lower match thresholds, drop candidate channels, skip component checks, or publish partial results to meet a runtime target.

Run-scoped work is checkpointed by deterministic input ranges and dependency digests. A completed batch can be replayed without semantic duplication, and an abandoned batch can be reclaimed after its worker heartbeat expires. Pausing or cancelling a run closes its workers and leaves staging isolated; resuming verifies that the source boundary, desired configuration, decision epoch, and function fingerprints are still valid. If they changed, the run is superseded or its affected batches are invalidated. The active publication remains available throughout.

Backpressure is observable. `describe()` reports whether currentness is delayed by source backlog, review blockage, configured maintenance policy, resource quota, competing entities, or a pathological data shape. Administrators can raise a budget, reschedule work, repair a rule, or authorize a broader rebuild with a clear understanding of the consequence. Normal users still invoke `refresh()`; workload control is an operational implementation of that action, not a second resolution workflow.

---

## 24. Lifecycle examples

### 24.1 A normal incremental change

A CRM customer changes its phone number. The source contract identifies one changed source record. The phone cleaner recomputes one normalized value and updates its block membership. Exact-phone relationships incident to the record are invalidated and regenerated; name, email, and tax-ID values remain untouched. If membership does not change, only the customer’s phone golden value and lineage may be recomputed. The final publication changes the affected golden row and history atomically, while all other entities and fields remain as they were.

### 24.2 A relationship disappears and an entity splits

A cleaner correction reveals that two previously equal tax IDs are different. The changed values invalidate an accepted identity relationship. The engine loads the old component because removing an accepted edge can split it, re-evaluates the component’s remaining relationships and rules, and produces two coherent components. Stable-ID reconciliation gives the old `mdm_id` to the child with the strongest deterministic continuity and gives the other child a new ID. A split event and review case are published with both memberships and golden records in one revision. Unrelated components are not reclustered.

### 24.3 A configuration change adds supporting evidence

An administrator proposes a fuzzy name match as supporting evidence. `preview()` validates that the new rule cannot cause automatic matches by itself, compiles a bounded candidate channel, estimates index work, runs labeled cases, and reports expected merges, splits, reviews, and affected entities. After `create()` stores the new version, `refresh()` rebuilds only the candidate/evidence dependencies required by that global rule; if the rule touches most records, the engine may choose a broad matching rebuild while reusing unchanged normalization. The public actions and outputs remain the same.

### 24.4 A steward rejects a dangerous bridge

Two established companies would be joined by one high-name-similarity bridge, but their supporting tax information is inconsistent. The engine sends the bridge to review rather than merging. The steward records `NOT_MATCH` with a reason. That durable cannot-link increments the decision epoch, locally reclusters the affected components if necessary, and remains active in future refreshes even if other records create a transitive path. `explain()` shows both the automatic evidence and the human decision.

### 24.5 Recovery through rebuild

An administrator suspects that a previous extension defect left stale candidate dependencies. `mdm_admin.rebuild()` recomputes all source records, candidates, evidence, components, golden values, and reviews under the current definition and decisions in run-scoped staging. Stable-ID reconciliation compares the result with the active state, preserving IDs wherever the deterministic continuity rules permit. Only after full validation does the publication pointer advance. Downstream readers never see a partially rebuilt state.


### 24.6 A late source event corrects history

An ERP event with effective time in June arrives in September after a newer July event. The source contract supplies a monotonic change sequence and marks the September delivery as a correction to the June version. The engine appends the event to source history, recomputes the valid-time projection, and verifies whether the July state remains current. If current matching is unchanged, no entity membership revision is published; the operation records a historical correction and updates any retained point-in-time projection. `explain()` distinguishes what was published in June from what the source is now known to have meant in June.

### 24.7 A source schema changes unexpectedly

A registry view changes `tax_id` from a jurisdictional identifier to a display string while preserving the column name. The mapped-column and view-definition fingerprints differ from the active source contract. Refresh fails before normalization and leaves the previous publication visible. `describe()` marks the source contract as violated and explains that identity semantics may have changed. An administrator updates the field mapping or cleaner in a proposed definition, previews the candidate and split impact, and publishes a new configuration only after labeled cases and authoritative-ID checks succeed.

### 24.8 A manual decision would contradict an existing split

A steward proposes `MATCH` between two records, but one belongs to each side of an active entity-level split directive. Preview returns the exact contradiction chain, the locked memberships, and the merge and golden-value consequences that would follow if the split were superseded. The write is rejected unless an authorized steward explicitly supersedes the older directive with a reason. No hidden “latest decision wins” rule is applied.

### 24.9 A downstream consumer processes a merge and split

A search index has processed customer changes through publication revision 1,200. Revision 1,201 merges `mdm_17` into `mdm_3`; the change feed provides the alias and changed golden fields, so the index redirects the old document and refreshes only the winner. Revision 1,205 splits `mdm_3`; the continuity child keeps `mdm_3`, while `mdm_91` is created for several moved members. The split event lists those memberships, allowing the index to create the new document and move dependent records without diffing the entire customer output.

### 24.10 An engine upgrade changes no identity semantics

An extension upgrade introduces a faster candidate index and a new storage layout. The migration verifies that the logical candidate-plan digest, fixed-point evidence, clustering semantics, and stable-ID rules are unchanged. Existing publication hashes remain valid, and no semantic refresh occurs. A later optional normalizer improvement has a new semantic version; `preview()` shows its expected matches, splits, reviews, and labeled-test results before an administrator opts in through a new configuration version.

---

## 25. Non-goals and boundary discipline

`pg_mdm` is not an ingestion platform, source-system connector suite, workflow engine, data catalog, product-information-management interface, machine-learning training service, or write-back synchronization system. It does not require a web application, although a review UI can be built against the stable review, explain, and stewardship contracts. It does not invent new SQL grammar or require users to maintain an external matching service. Optional learned comparators may be added later only if they satisfy the same deterministic versioning, explanation, candidate-bounding, preview, and safety contracts; they do not replace the transparent default semantics.

The product boundary is protected by a design rule: a new engine mechanism does not automatically justify a new public noun or action. Candidate graphs, definition fragments, composition manifests, entity-dependency graphs, edge persistence, dependency invalidation, publication revisions, adaptive indexing, and micro-batching are internal or advanced configuration details because they make `create`, `describe`, `preview`, `refresh`, and `explain` correct and fast. Advanced stewardship and administration are separated by namespace and privilege so they remain available without crowding the normal interface.

The boundary test is simple: users may extend what sources, fields, matches, entities, and golden values mean, but should almost never need to extend the five actions. Geographic comparison, an organization name cleaner, a candidate strategy, or a finance survivorship rule belongs in a definition and still runs through `preview()`, `refresh()`, and `explain()`. A custom `run_special_match()` or replacement clustering entry point does not.

---

## 26. Acceptance criteria

A v1 implementation is acceptable only when a user can define an entity from multiple PostgreSQL relations, map logical fields, normalize values while preserving originals, express safe intent-level matching, refresh without all-pairs comparison, receive deterministic constraint-aware entities with stable IDs, query the membership map and golden records, trace every golden value, review uncertainty, record durable human decisions, explain every important outcome, and repeat a no-change refresh without semantic changes. Incremental processing must handle inserts, updates, deletions, disappearing relationships, local splits, local merges, and golden-only changes, with a safe fallback to broader work. Publication must be atomic, configuration and decisions must survive backup and restore, and resource guards must fail safely.

Source correctness is also a v1 requirement. Every source must have a validated stable-key and lifecycle contract, absence must never be interpreted as deletion without proof, ordered sources must handle duplicated and late events deterministically, incompatible schema drift must stop before publication, and active output must carry meaningful source boundaries. Configuration changes must be classified by invalidation scope. Active manual constraints must be logically consistent and follow a documented precedence model. Every publication must carry a resolution manifest sufficient to identify the exact configuration, functions, engine semantics, decisions, and source boundaries that produced it.

The strong product is acceptable when the same model also provides versioned reusable fragments, built-in and organization presets through one canonical compilation path, shallow composition with explicit overrides, schema inference that emits ordinary definitions, independently composable candidate channels, automatic physical optimization, semantic previews, comprehensive validation, health metrics, rule diagnostics, labeled regression cases, configuration history and portable definitions, entity-level correction, identity history, source authority, entity invariants, explicit value states, operational currentness, concurrency safety, PII-aware diagnostics, temporal history, actionable errors, excellent describe and explain experiences, a rebuild escape hatch, and demonstrated benchmark behavior through the largest supported scale tier without changing the public mental model.

Long-term operation is acceptable when review queues are risk-prioritized and easy to render safely, every consequential action has exact or clearly bounded impact analysis, downstream consumers can advance through ordered change events and resolve old IDs, source-to-MDM freshness is provable, declared extension dependencies preserve incremental correctness, resolved outputs can act as sources with acyclic dependency propagation, heavy work yields and resumes without partial publication, history and evidence have explicit storage lifecycles, backup and failover preserve identity continuity, namespaces are isolated in data and resources, diagnostic storage follows data-minimization policy, and service-level objectives can be measured without weakening matching correctness. A change between two revisions must have a bounded explanation of both its cause and its consequences.

---

## Appendix A. Requirement traceability

The table below maps all 82 requested traits to the part of this proposal that supplies them. “Covered” means the architecture and public contract explicitly include the capability; implementation sequencing can still stage the work behind compatible interfaces.

| # | Trait | Coverage in this proposal |
|---:|---|---|
| 1 | Entity definition | First-class `mdm.entity` definition consumed by `mdm.create()`; Sections 3–5 |
| 2 | Multiple sources | First-class `mdm.source` values over multiple relations with stable IDs; Sections 3–4 |
| 3 | Logical fields | Per-source physical-to-logical mappings; Sections 3–4 |
| 4 | Normalization | Raw preservation, deterministic cleaned values, versioning, quality states; Section 7 |
| 5 | Simple matching semantics | Exact/fuzzy comparison plus strong/supporting/identity roles; Section 8 |
| 6 | Efficient candidate discovery | Compiled bounded channels, indexes, blocks, nearest-neighbor search; Section 9 |
| 7 | Deterministic matching | Fixed-point evidence, pinned logical plan, complete tie orders; Section 8 |
| 8 | Conflict rules | Identity disagreements, cannot-links, authority, entity rules; Sections 8 and 10 |
| 9 | Entity clustering | Constraint-aware graph clustering and conservative bridge policy; Section 10 |
| 10 | Stable `mdm_id` | Deterministic continuation, merge, split, retirement, aliases; Section 11 |
| 11 | Membership mapping | Stable `(source, source_id) -> mdm_id` output contract; Sections 6 and 11 |
| 12 | Golden records | One row per entity, selected independently by field; Sections 6 and 12 |
| 13 | Simple golden-value policies | Prefer source, latest, first non-null, most common, custom; Section 12 |
| 14 | Provenance | Mandatory field-level lineage to source record and value; Section 12 |
| 15 | Human review | First-class review output for ambiguity, conflict, bridges, splits, ties; Section 13 |
| 16 | Durable manual decisions | Versioned `MATCH`, `NOT_MATCH`, corrections, and overrides; Sections 11 and 13 |
| 17 | Explainability | One `mdm.explain()` for records, pairs, entities, fields, reviews; Section 5.5 |
| 18 | Incremental processing | Dependency-driven changed-record and affected-entity recomputation; Section 14 |
| 19 | Local reclustering | Edge invalidation and complete affected-closure reclustering; Section 14 |
| 20 | Atomic refresh | Run staging plus one MVCC publication revision; Section 14.1 |
| 21 | Idempotency | No semantic revision for unchanged inputs and decisions; Section 14.2 |
| 22 | Three primary outputs | Golden, members, and review relations; Section 6 |
| 23 | Radical API simplicity | Five normal actions and separate advanced namespaces; Sections 2 and 5 |
| 24 | Declarative configuration | Complete immutable desired-state definition; Section 3 |
| 25 | Progressive disclosure | Intent defaults first, compiled detail and expert options later; Sections 2, 5, and 9 |
| 26 | PostgreSQL-native operation | Relations, functions, transactions, indexes, MVCC, permissions, migrations; Section 17 |
| 27 | PostgreSQL-function extensibility | Cleaners, comparators, candidate keys, checks, selectors; Section 18 |
| 28 | Safe defaults | Precision-first matching, review, bounded work, no silent incompleteness; Throughout, especially Sections 8–10 |
| 29 | Presets | Versioned and expanded person/company/product-style presets; Section 4.1 |
| 30 | Schema inference | Catalog- and statistics-based suggestions through preview; Section 4.1 |
| 31 | Automatic optimization | Compiled logical plan and automatically selected physical structures; Section 9 |
| 32 | Preview changes | Semantic merge/split/review/impact and resource preview; Section 5.3 |
| 33 | Configuration validation | Structural, semantic, computational, and function validation; Section 15 |
| 34 | Quality metrics | Entity, cluster, match, review, field, golden, and run scorecards; Section 16 |
| 35 | Matching diagnostics | Coverage, candidate yield, noise, marginal contribution, label performance; Section 16 |
| 36 | Labeled test cases | Pair and entity fixtures executed during preview; Section 15 |
| 37 | Configuration history | Immutable versions, ancestry, diffs, rollback by new version; Section 15 |
| 38 | Configuration import/export | Canonical `describe` definition consumed by `create`; Section 15 |
| 39 | Entity-level correction | Merge, split, move-member, and lock stewardship directives; Section 11.1 |
| 40 | Identity history | Aliases, merge events, split relationships, temporal membership; Sections 11 and 22 |
| 41 | Source authority | Identity-wide and field-specific authority for matching and golden values; Sections 4 and 12 |
| 42 | Entity invariants | Field rules and custom component checks; Sections 8.1 and 10 |
| 43 | Missing-value semantics | Absent, empty, invalid, unknown, redacted, unsupported, value; Section 7 |
| 44 | Operational observability | Currentness, pending work, failures, stage timings, run metrics; Section 21 |
| 45 | Resource safeguards | Candidate, block, component, custom-function, time, and storage budgets; Sections 9 and 20 |
| 46 | Concurrency safety | Entity publication locks, epochs, expected versions, worker recovery; Section 19 |
| 47 | Security model | Separate privileges for administration, configuration, review, and read; Section 20 |
| 48 | PII-aware diagnostics | Masking classes, protected raw storage, safe logs and retention; Sections 7 and 20 |
| 49 | Temporal/history support | Revision validity, event history, point-in-time explain and history views; Section 22 |
| 50 | Excellent errors | Stable MDM codes, cause, consequence, and concrete next action; Sections 15 and 21 |
| 51 | Excellent `describe()` | Effective config, compiled plan, health, currentness, history, recommendations; Section 5.2 |
| 52 | Excellent `explain()` | One subject-polymorphic conceptual interface; Section 5.5 |
| 53 | Rebuild escape hatch | Explicit privileged `mdm_admin.rebuild()` using the same publication contract; Section 5.6 |
| 54 | Scale without changing the mental model | Same definition and actions across scale tiers; Section 23 |
| 55 | Source data contracts | Stable-key, completeness, ordering, lifecycle, and schema contracts; Section 4.2 |
| 56 | Deletion semantics | Deleted, deactivated, superseded, temporarily absent, and filtered states; Sections 4.2 and 14.3 |
| 57 | Late-arriving and out-of-order data | Declared total order, idempotent replay, valid time, and completeness watermarks; Sections 4.2 and 14.3 |
| 58 | Source schema evolution | Relation and view fingerprints, compatibility checks, and fail-before-publish behavior; Sections 4.2 and 15.1 |
| 59 | Configuration compatibility analysis | Explicit invalidation classes and safe work scope; Sections 5.1 and 15.1 |
| 60 | Reproducible resolution | Per-publication resolution manifests and stated reproducibility horizon; Section 15.2 |
| 61 | Engine upgrade stability | Storage, physical-plan, semantic, and correctness-repair upgrade classes; Section 15.2 |
| 62 | Deterministic extension contracts | Constrained cleaner, candidate, comparator, component-check, and selector contracts with declared dependencies, fingerprints, versions, cost, cardinality, and test vectors; Section 18 |
| 63 | Decision precedence | Explicit integrity, lock, stewardship, invariant, authority, and automatic-evidence lattice; Section 13.1 |
| 64 | Contradiction detection | Must-link/cannot-link closure validation and explicit supersession; Section 13.1 |
| 65 | Review prioritization | Deterministic risk, impact, age, authority, and SLO-based ordering; Sections 13 and 13.2 |
| 66 | Review ergonomics | Stable side-by-side explain sections, context, masking, consequences, and permitted actions; Section 13.2 |
| 67 | Impact analysis | Preview of configuration and stewardship effects on IDs, members, goldens, reviews, and events; Sections 5.3 and 13.2 |
| 68 | Downstream change feed | Atomic ordered relational publication events; Section 6.5 |
| 69 | Stable downstream references | Merge aliases, deterministic split continuity, identity history, and revision cursors; Sections 6.5 and 11 |
| 70 | Freshness guarantees | Per-source completeness boundaries, lag, and current/pending/stale/unknown states; Sections 21 and 21.1 |
| 71 | Backpressure and workload control | Resource budgets, yielding, checkpointing, cancellation, and safe resume; Sections 5.4 and 23.1 |
| 72 | Storage lifecycle management | Data-class retention, interval history, checkpoints, compaction, and storage safeguards; Section 22.1 |
| 73 | Disaster recovery semantics | Durable/rebuildable state, backup verification, replication, failover, and identity continuity; Section 17.1 |
| 74 | Multi-tenant and namespace isolation | Explicit catalog, permission, source, output, lock, and resource scoping; Section 20.1 |
| 75 | Data minimization | Sensitivity and retention classes, digest-based exact indexes, and limited PII duplication; Section 20.2 |
| 76 | Service-level objectives | Freshness, run, review, feed, explain, and recovery objectives; Section 21.1 |
| 77 | Reusable definition fragments | Versioned partial definitions for fields, source conventions, matches, entity rules, and golden policies; Section 3.1 |
| 78 | One canonical compilation model | Built-in presets, organization presets, fragments, inference, and direct definitions expand before one validation and compilation pipeline; Sections 3.1 and 4.1 |
| 79 | Shallow composition and explicit overrides | Ordered composition, conflict errors, pinned inputs, no entity inheritance, and fully expanded describe output; Section 3.1 |
| 80 | Extension dependency declarations | Registered relation, function, collation, extension, and setting dependencies drive safe invalidation; Sections 14 and 18 |
| 81 | Composable candidate channels | Independent bounded discovery channels feed unchanged evidence rules and remain expert match configuration; Section 9 |
| 82 | Resolved outputs as sources | Stable `mdm_out` relations, synthesized contracts, acyclic entity dependencies, and downstream dirty propagation; Sections 4.3 and 14 |

---

## Appendix B. Public namespace summary

```text
mdm
  Five noun constructors:
    source, field, match, entity, golden_value

  Five normal actions:
    create, describe, preview, refresh, explain

    Advanced definition composition:
        versioned fragments and presets, expanded before entity compilation

mdm_out
  Three primary outputs per entity:
    <entity>
    <entity>_members
    <entity>_review

  Optional relational detail:
    <entity>_lineage
    <entity>_metrics
    <entity>_history
    <entity>_members_history
    <entity>_identity_history
    <entity>_changes

mdm_steward
  Durable exceptional decisions:
    decide, merge, split, move_member, lock, unlock,
    override_golden, clear_golden_override, label

mdm_admin
  Privileged recovery and maintenance:
    rebuild, verify, repair, retention, pause_run, resume_run, cancel_run

mdm_internal
  No ordinary user access:
    catalogs, normalized values, candidate plans, edges, components,
        composition manifests, entity dependencies, dirty state, publications,
        histories, staging, dependency indexes
```

The namespace summary is not an invitation to expose engine internals. It is a reminder that normal use remains five nouns, five actions, and three outputs, while specialized stewardship and recovery capabilities stay available behind clear privilege and naming boundaries.

---

## Appendix C. System invariants

The implementation is governed by a small set of invariants that are stronger than any particular table layout or algorithm. These invariants are useful during design review because they reveal whether a proposed optimization preserves the product contract.

1. **No partial publication.** Memberships, entities, golden values, reviews, lineage, identity history, and downstream events from one semantic revision become visible together or not at all.
2. **No silent candidate incompleteness.** A bounded search that cannot complete may produce review or an actionable failure, but it cannot be treated as proof of non-match.
3. **No implicit deletion.** A missing row withdraws a source record only when the active source contract proves that the relevant snapshot or event boundary is complete.
4. **No unversioned semantic change.** A definition, preset, custom function, collation dependency, or engine behavior that can change identity produces a new previewable semantic version.
5. **No contradictory active stewardship state.** Must-link, cannot-link, split, move, and lock directives are validated as a coherent constraint set before they can affect publication.
6. **One active membership per source record.** Within an entity and namespace, each active `(source, source_id)` maps to exactly one active `mdm_id`.
7. **Stable identity follows deterministic continuity.** Merge and split survivors are selected by published tie-break rules, never by row order, worker timing, or incidental UUID ordering.
8. **Every golden value has provenance.** A current or retained historical golden value identifies the source value or deterministic computation that produced it.
9. **Currentness must be provable.** A successful run timestamp is not enough; currentness depends on source completeness boundaries, desired configuration, active decisions, and publication state.
10. **The same semantic manifest yields the same semantic result.** Worker count, batching, and physical indexes may change performance, but not memberships, identities, goldens, reviews, or explanations.
11. **Every semantic change has a bounded cause-and-consequence explanation.** The explanation names the initiating change, affected dependencies, identity result, output changes, and downstream events, with controlled expansion for large graphs.
12. **Resource pressure changes timing, not truth.** Backpressure may delay a publication or force a broader safe plan; it may not weaken evidence, skip constraints, or expose incomplete output.
13. **Every definition has one canonical expansion.** Fragments, presets, inference, and direct declarations resolve to one pinned effective definition before validation or compilation, and conflicts require explicit overrides.
14. **Every mutable extension input is declared.** A custom semantic function cannot preserve incremental results unless the engine can fingerprint or observe every relation, function, collation, extension, and setting that can change its output.
15. **Entity dependencies are acyclic and revision-bound.** An MDM output can be a source only through a cycle-free dependency graph, and every downstream publication records the exact upstream revisions it consumed.
16. **Extensions do not replace engine governance.** Custom behavior may clean, discover, compare, check, or select; the engine always owns decision precedence, relationship lifecycle, clustering, stable IDs, invalidation, and publication.

