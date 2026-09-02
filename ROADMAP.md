# `pg_mdm` V1 implementation roadmap

## Purpose

This roadmap divides the V1 design into small, testable development releases of approximately four to five person-weeks each. The estimates assume one experienced engineer familiar with PostgreSQL extension development, Rust, and `pg_trickle`; they measure engineering effort rather than elapsed calendar time. Releases before V1 are implementation milestones, not compatibility promises, and may change as the design is exercised.

The roadmap follows [`DESIGN_V1.md`](DESIGN_V1.md). Each release should leave its completed behavior runnable and tested, while features assigned to [`DESIGN_V2.md`](DESIGN_V2.md) remain out of scope. In particular, V1 uses full-entity resolution over complete terminal evidence relations and does not wait for affected-set resolution or durable output deltas.

## Development releases

### v0.1 — Extension foundation

Establish the extension package, installation and upgrade scripts, internal and public schemas, roles, privileges, fixed security-definer search paths, and durable operation records. This release provides the smallest reliable base on which later SQL APIs and state can be built, but it does not yet create or resolve entities.

### v0.2 — Definitions and validation

Implement the entity, source, field, match, and golden-value definition model together with immutable definition versions, semantic digests, source-key validation, optimistic concurrency, and the initial `create` and `describe` surfaces. Invalid or unsupported definitions must fail without leaving partially installed state.

### v0.3 — Normalization

Implement the V1 built-in cleaners, typed normalized-value states, deterministic ordering rules, and source-record identity handling. Tests must cover supported scalar and composite keys, cleaner versioning, invalid and absent values, authoritative fields, and equivalent results across clean rebuilds.

### v0.4 — Candidate generation

Compile exact, composite, prefix, token, and other bounded V1 candidate channels into complete candidate blocks and canonical pairs. Enforce per-block and aggregate completeness limits so resource pressure fails the operation rather than truncating required work or treating an unexamined pair as a non-match.

### v0.5 — Evidence and pair decisions

Evaluate built-in exact and fuzzy comparisons for discovered pairs, group correlated evidence, apply authority conflicts, and produce deterministic automatic pair decisions. Add durable steward `MATCH` and `NOT_MATCH` decisions with precedence, optimistic concurrency, and contradiction checks.

### v0.6 — Conservative clustering

Implement the full-reference resolver, deterministic edge order, must-link closure, cannot-link enforcement, and the V1 component-admission rule that prevents weak chain accretion. Generated tests must exercise edge appearance and disappearance, conflicting constraints, large components, and stable results under input and plan reordering.

### v0.7 — Identity and publication model

Implement stable `mdm_id` allocation and reconciliation, merge and split continuity, aliases, golden-value selection, anchored overrides, provenance, reviews, and bounded machine-readable explanation. Publish the three V1 output-table shapes in resolver tests, without yet advancing a live `pg_trickle` graph.

### v0.8 — `pg_trickle` graph integration

Compile definitions into private stream-table graphs and integrate capability negotiation, member and graph contract digests, durable `EXTERNAL` orchestration, and supported lifecycle operations. This release depends on a usable `external_graph_refresh` major 1 contract from `pg_trickle`; private catalogs or provisional internal APIs must not be used as substitutes.

### v0.9 — Transactional refresh

Implement `preview`, `refresh`, and administrative rebuild around strict graph refresh, complete source boundaries, full terminal-relation scans, full-entity resolution, and atomic publication. A failure after graph maintenance must roll back graph contents, consumed frontiers, MDM state, and public outputs in the caller's outer PostgreSQL transaction.

### v0.10 — Operational hardening

Complete security, authorization, concurrency, lifecycle locking, crash recovery, clone isolation, backup and restore behavior, extension upgrades, full-fallback handling, and documented resource limits. Failures must use stable, actionable results and must never leave a partial publication or silently weaken candidate completeness.

### v0.11 — Release qualification

Run the shared `pg_trickle` conformance suite and the generated differential-versus-full reference suite across inserts, updates, deletes, stewardship changes, merges, splits, golden-only changes, rollback, concurrency, fallback, rebuild, and upgrade. Publish the tested operating envelope and complete the installation, administration, recovery, and user documentation required for a supported release.

## v1.0 release gate

V1.0 freezes the public compatibility contract only after every V1 acceptance criterion passes. The supported `pg_trickle` release must advertise `external_graph_refresh` major 1, and the shared suite must prove canonical graph contracts, durable external orchestration, strict transactional refresh, complete source boundaries, rollback, concurrency, clone isolation, recovery, and supported upgrades. `output_delta_consumer` is not a V1 prerequisite.

The expected total is 44–55 person-weeks. Work through v0.7 can proceed independently of the final `pg_trickle` extension points; v0.8 through v1.0 remain gated by their stable implementation and conformance behavior. If upstream delivery is delayed, the resolver and its reference tests should continue rather than introducing a temporary integration layer that would later need to be removed.
