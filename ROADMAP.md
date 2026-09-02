# `pg_mdm` V1 implementation roadmap

## Purpose

This roadmap divides the V1 design into testable development releases implemented sequentially by a coding agent with human review. Only one release is in progress at a time, and the next starts after the current release meets its exit evidence. The estimates are rough-order-of-magnitude ranges, not commitments, and include implementation, tests, documentation, review, and defect fixing. Their confidence is low until working code establishes delivery velocity.

The roadmap follows [`DESIGN_V1.md`](DESIGN_V1.md). Each release should leave its completed behavior runnable and tested, while features assigned to [`DESIGN_V2.md`](DESIGN_V2.md) remain out of scope. In particular, V1 uses full-entity resolution over complete terminal evidence relations and does not wait for affected-set resolution or durable output deltas.

This repository currently contains designs only. Implementation does not start until a released `pg_trickle` capability advertises stable `external_graph_refresh` major 1 and passes its conformance gate. Before attaching calendar dates, v0.1 and v0.2 must then establish the extension toolchain and test harness, and the remaining estimates must be revised from measured delivery. A milestone is complete only when its behavior is installed, runnable, and covered by its stated tests; design or partially wired code does not count. If a milestone exceeds its upper range, re-estimate the remaining work and move optional behavior to V2 rather than silently extending V1.

## Development releases

### v0.1 — Extension foundation (3–5 person-weeks)

Establish the extension package, installation and upgrade scripts, internal and public schemas, roles, privileges, fixed security-definer search paths, and durable operation records. This release provides the smallest reliable base on which later SQL APIs and state can be built, but it does not yet create or resolve entities.

Exit evidence: clean install, upgrade, privilege, hostile-`search_path`, and rollback tests run on the single PostgreSQL major inherited from the selected `pg_trickle` release. Additional PostgreSQL majors are post-V1 scope.

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

As of 2026-09-02, the latest `pg_trickle` release is v0.91.0. Its roadmap places backup, restore, upgrade, and clone correctness in v0.92.0, graph contracts and external ownership in v0.93.0, and strict transactional graph refresh in v0.94.0; all three are planned, sequential, large releases without committed dates. Do not start v0.1 against an experimental substitute. Once a released capability advertises stable `external_graph_refresh` major 1, run a two-week compatibility spike before v0.1 and revise or stop the roadmap before committing to dates.

### v0.9 — Transactional refresh (6–9 person-weeks)

Implement `preview`, `refresh`, and administrative rebuild around strict graph refresh, complete source boundaries, full terminal-relation scans, full-entity resolution, and atomic publication. A failure after graph maintenance must roll back graph contents, consumed frontiers, MDM state, and public outputs in the caller's outer PostgreSQL transaction.

Exit evidence: success, no-op, injected failure, concurrent source-write, concurrent lifecycle, and fallback tests prove that graph and MDM state commit or roll back together.

### v0.10 — Operational hardening (8–12 person-weeks)

Complete security, authorization, concurrency, lifecycle locking, crash recovery, clone isolation, backup and restore behavior, extension upgrades, full-fallback handling, and documented resource limits. Failures must use stable, actionable results and must never leave a partial publication or silently weaken candidate completeness.

### v0.11 — Release qualification (5–8 person-weeks)

Run the shared `pg_trickle` conformance suite and the generated differential-versus-full reference suite across inserts, updates, deletes, stewardship changes, merges, splits, golden-only changes, rollback, concurrency, fallback, rebuild, and upgrade. Publish the tested operating envelope and complete the installation, administration, recovery, and user documentation required for a supported release.

## v1.0 release gate

V1.0 freezes the public compatibility contract only after every V1 acceptance criterion passes. The supported `pg_trickle` release must advertise `external_graph_refresh` major 1, and the shared suite must prove canonical graph contracts, durable external orchestration, strict transactional refresh, complete source boundaries, rollback, concurrency, clone isolation, recovery, and supported upgrades. `output_delta_consumer` is not a V1 prerequisite.

The milestone ranges total 65–98 engineering person-weeks before cross-cutting rework. Because there is no implementation evidence and the critical integration API does not yet exist, use an initial funding envelope of 90–150 person-weeks rather than a single target. For one engineer sustaining 40–45 project weeks per year, that is roughly 24–45 calendar months after implementation starts. Upstream waiting time and unrelated maintenance extend that range.

All implementation waits for the stable `pg_trickle` extension contract. If upstream delivery is delayed, keep the design current rather than building code or a temporary integration layer. After the compatibility spike, deliver v0.1 through v0.11 in order with a separate reviewed change for each release. Publish a calendar forecast only after v0.2 establishes measured coding-agent delivery velocity.
