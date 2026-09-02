# `pg_mdm` V1 implementation roadmap

## Purpose

This roadmap divides the V1 design into testable development releases. The estimates assume one experienced engineer working full-time who is already familiar with PostgreSQL extension development, Rust, and `pg_trickle`. They are planning ranges, not commitments, and measure engineering effort rather than elapsed calendar time.

The roadmap follows [`DESIGN_V1.md`](DESIGN_V1.md). Each release should leave its completed behavior runnable and tested, while features assigned to [`DESIGN_V2.md`](DESIGN_V2.md) remain out of scope. In particular, V1 uses full-entity resolution over complete terminal evidence relations and does not wait for affected-set resolution or durable output deltas.

This repository currently contains designs only. Before attaching calendar dates, v0.1 and v0.2 must establish the extension toolchain and test harness, and the remaining estimates must be revised from measured delivery. A milestone is complete only when its behavior is installed, runnable, and covered by its stated tests; design or partially wired code does not count.

## Development releases

### v0.1 — Extension foundation (3–5 person-weeks)

Establish the extension package, installation and upgrade scripts, internal and public schemas, roles, privileges, fixed security-definer search paths, and durable operation records. This release provides the smallest reliable base on which later SQL APIs and state can be built, but it does not yet create or resolve entities.

Exit evidence: clean install, upgrade, privilege, hostile-`search_path`, and rollback tests run in the supported PostgreSQL versions.

### v0.2 — Definitions and validation (5–7 person-weeks)

Implement the entity, source, field, match, and golden-value definition model together with immutable definition versions, semantic digests, source-key validation, optimistic concurrency, and the initial `create` and `describe` surfaces. Invalid or unsupported definitions must fail without leaving partially installed state.

Exit evidence: definitions round-trip through `describe`, semantic no-ops remain no-ops, and invalid or concurrent updates leave no partial state. Re-estimate v0.3–v0.7 from the effort measured through this release.

### v0.3 — Normalization (4–6 person-weeks)

Implement the V1 built-in cleaners, typed normalized-value states, deterministic ordering rules, and source-record identity handling. Tests must cover supported scalar and composite keys, cleaner versioning, invalid and absent values, authoritative fields, and equivalent results across clean rebuilds.

### v0.4 — Candidate generation (6–9 person-weeks)

Compile exact, composite, prefix, token, and other bounded V1 candidate channels into complete candidate blocks and canonical pairs. Enforce per-block and aggregate completeness limits so resource pressure fails the operation rather than truncating required work or treating an unexamined pair as a non-match.

Exit evidence: generated small datasets match an all-pairs reference, while oversized blocks and aggregate limits fail without emitting partial results.

### v0.5 — Evidence and pair decisions (7–10 person-weeks)

Evaluate built-in exact and fuzzy comparisons for discovered pairs, group correlated evidence, apply authority conflicts, and produce deterministic automatic pair decisions. Add durable steward `MATCH` and `NOT_MATCH` decisions with precedence, optimistic concurrency, and contradiction checks.

### v0.6 — Conservative clustering (7–10 person-weeks)

Implement the full-reference resolver, deterministic edge order, must-link closure, cannot-link enforcement, and the V1 component-admission rule that prevents weak chain accretion. Generated tests must exercise edge appearance and disappearance, conflicting constraints, large components, and stable results under input and plan reordering.

### v0.7 — Identity and publication model (8–12 person-weeks)

Implement stable `mdm_id` allocation and reconciliation, merge and split continuity, aliases, golden-value selection, anchored overrides, provenance, reviews, and bounded machine-readable explanation. Publish the three V1 output-table shapes in resolver tests, without yet advancing a live `pg_trickle` graph.

Exit evidence: generated histories preserve the specified identities and produce identical memberships, goldens, reviews, and explanations under replay and input reordering. Re-estimate integration and qualification after this release.

### v0.8 — `pg_trickle` graph integration (6–10 person-weeks after upstream availability)

Compile definitions into private stream-table graphs and integrate capability negotiation, member and graph contract digests, durable `EXTERNAL` orchestration, and supported lifecycle operations. This release depends on a usable `external_graph_refresh` major 1 contract from `pg_trickle`; private catalogs or provisional internal APIs must not be used as substitutes.

As of 2026-09-02, the latest `pg_trickle` release is v0.91.0 and the required graph contract and strict transactional refresh are planned for its v0.93.0 and v0.94.0 releases. Work through v0.7 can continue, but v0.8 must not start against an experimental substitute. Once the stable contract exists, begin with a short compatibility spike and revise the v0.8–v0.11 estimates before committing to dates.

### v0.9 — Transactional refresh (6–9 person-weeks)

Implement `preview`, `refresh`, and administrative rebuild around strict graph refresh, complete source boundaries, full terminal-relation scans, full-entity resolution, and atomic publication. A failure after graph maintenance must roll back graph contents, consumed frontiers, MDM state, and public outputs in the caller's outer PostgreSQL transaction.

Exit evidence: success, no-op, injected failure, concurrent source-write, concurrent lifecycle, and fallback tests prove that graph and MDM state commit or roll back together.

### v0.10 — Operational hardening (8–12 person-weeks)

Complete security, authorization, concurrency, lifecycle locking, crash recovery, clone isolation, backup and restore behavior, extension upgrades, full-fallback handling, and documented resource limits. Failures must use stable, actionable results and must never leave a partial publication or silently weaken candidate completeness.

### v0.11 — Release qualification (5–8 person-weeks)

Run the shared `pg_trickle` conformance suite and the generated differential-versus-full reference suite across inserts, updates, deletes, stewardship changes, merges, splits, golden-only changes, rollback, concurrency, fallback, rebuild, and upgrade. Publish the tested operating envelope and complete the installation, administration, recovery, and user documentation required for a supported release.

## v1.0 release gate

V1.0 freezes the public compatibility contract only after every V1 acceptance criterion passes. The supported `pg_trickle` release must advertise `external_graph_refresh` major 1, and the shared suite must prove canonical graph contracts, durable external orchestration, strict transactional refresh, complete source boundaries, rollback, concurrency, clone isolation, recovery, and supported upgrades. `output_delta_consumer` is not a V1 prerequisite.

The base estimate is 65–98 person-weeks. Plan on 80–125 person-weeks after a 25% contingency for extension integration, correctness failures found by generated tests, and release qualification. For one full-time engineer that is roughly 18–29 months after implementation starts, excluding upstream waiting time, review latency, and maintenance interruptions.

Work through v0.7 can proceed independently of the final `pg_trickle` extension points; v0.8 through v1.0 remain gated by their stable implementation and conformance behavior. If upstream delivery is delayed, continue the resolver and its reference tests rather than building a temporary integration layer. Calendar forecasts should be published only after v0.2 establishes delivery velocity and again after the post-v0.7 compatibility spike.
