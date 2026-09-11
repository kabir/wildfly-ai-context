# ADR 0002: Search-First, Evidence-Aware Source Intelligence

**Status**: Accepted

**Date**: 2026-09-11

## Context

The WildFly ecosystem spans repositories with different source layouts, release cycles, dependency relationships, and levels of AI-context adoption. Coding assistants need to locate declarations, callers, implementations, configuration, documentation, and cross-repository ownership without loading the entire ecosystem into every task.

The source-intelligence investigation evaluated ordinary search, declaration indexes, CBM-style graph navigation, SCIP generation and lookup, partial workspaces, immutable index registries, and several established source-navigation approaches. The findings are recorded in the [source intelligence architecture options](../architecture/source-intelligence-architecture-options.md) and Experiments 0001–0006.

The investigation established several constraints:

1. Ordinary search was fast, transparent, and sufficient for the bounded navigation tasks tested.
2. Generated indexes can provide useful structural or caller-navigation leads, but generation, storage, build preparation, and freshness validation can be expensive.
3. Unguarded generated state can silently become incorrect after branch switches, dirty edits, renames, deletions, generated-source changes, or dependency changes.
4. A feature-pack or partial workspace may provide useful routing, configuration, and version evidence while lacking the runtime source needed to answer implementation or caller questions.
5. SCIP and similar formats address interchange or navigation facts, not repository discovery, build execution, freshness, artifact distribution, or assistant integration as a whole.
6. A central immutable index registry is technically plausible, but its ownership, artifact trust, production backend, CI lifecycle, and assistant-level value are not yet established.

The POC AI-context files in satellite repositories are not yet stable APIs or prerequisites for all users. Source intelligence must therefore work with repositories that have no AI-context files, repositories that later adopt them, old maintenance branches, current development branches, released versions, SNAPSHOT dependencies, dirty worktrees, and multiple assistants.

This ADR complements [ADR 0001](0001-hub-and-spoke-ai-context.md). ADR 0001 defines the federated hub-and-spoke context model. This ADR defines how source-navigation and generated-index capabilities fit into that model; it does not change repository ownership or require immediate satellite adoption.

## Decision

Adopt a search-first, evidence-aware source-intelligence architecture.

### Hub responsibilities

The hub remains a routing, ownership, provenance, and evidence-classification layer. It may describe where source, documentation, artifacts, and related repositories are located, but it does not become a monolithic source database.

The hub should maintain repository and component profiles that identify, where known:

- repository identity and ownership;
- source and documentation locations;
- supported branches, releases, and tags;
- cross-repository relationships;
- available artifacts and their coordinates;
- evidence provenance and verification status.

These profiles must not imply that a linked repository or artifact proves a particular implementation revision unless that relationship is explicitly verified.

### Search baseline and fallback

Ordinary local search and bounded source reading are the mandatory baseline. They must remain available when generated indexes are absent, stale, partial, unavailable, too expensive, or incompatible with the current workspace.

No assistant workflow may interpret an incomplete index as proof that a caller, implementation, or reference does not exist. Results must distinguish exact source evidence, navigation hints, partial coverage, unavailable context, and inference.

### Optional generated indexes

Generated indexes may be introduced as optional accelerators for tasks where experiments demonstrate measurable value, such as repeated caller or impact exploration. They must not be a prerequisite for working in a satellite repository and must not require every satellite to commit permanent AI-context or index files.

The architecture does not select CBM, SCIP, MCP, a shared CLI, a remote query service, or a particular index storage format as a mandatory dependency. Existing interchange formats and indexing systems may be used where they satisfy the source-state and provenance requirements.

### Source and index identity

Every generated result must carry enough identity to determine whether it applies to the requested source state. At minimum, this includes:

- canonical repository identity and exact source commit;
- working-tree and generated-source state, or an explicit clean-state claim;
- dependency, build, compiler, and toolchain inputs that can affect the result;
- timestamped and checksum information for SNAPSHOT dependencies where relevant;
- generator, parser, configuration, and schema versions;
- included, excluded, and parse-partial coverage;
- artifact checksums and generation status.

A branch name, tag name, repository basename, nominal version, or local path is not sufficient index identity. Branches are mutable aliases. Tags may be treated as immutable only when the surrounding publication process verifies and enforces that property.

### Workspace and freshness behavior

The source-intelligence layer must handle the following explicitly:

- clean checkouts at matching commits may use matching generated state;
- branch, ref, or commit changes invalidate an index unless an exact matching entry exists;
- dirty, untracked, renamed, or deleted source requires search, refresh, or an explicit downgrade in result confidence;
- generated-source, build-input, dependency, and SNAPSHOT changes invalidate affected evidence unless their impact is proven otherwise;
- missing runtime source or unavailable indexes produce explicit unavailable-context results;
- a stale or mismatched index must never be silently reported as current or complete.

An identity check should be cheaper than a full source digest where possible. A provider may reject generated state using commit/status/build metadata without hashing the entire checkout, then perform a more detailed refresh only when policy permits.

### Evidence composition

Evidence providers may be combined only when their repository, source/artifact, version, build-input, and coverage identities are compatible for the question being answered.

For example, a feature-pack declaration may establish that a runtime artifact or repository is relevant, but it must not be silently combined with an unrelated or older runtime index to answer an implementation question. Source, source JARs, binaries, feature-pack artifacts, documentation, hub metadata, and generated indexes remain distinct evidence classes.

### Distribution and future registries

A secondary repository or artifact store containing manifests and immutable index artifacts remains a permitted future option. If pursued, it must be treated as distribution infrastructure rather than the source of truth for source code or repository ownership.

Registry entries must resolve to exact source and generator identities, expose lifecycle and coverage status, verify artifacts before use, and preserve search fallback. Branch pointers may provide convenience, but immutable commit entries remain the identity boundary. A registry is not required for the baseline architecture.

### Assistant interfaces

The underlying result model should be provenance-aware and interface-neutral. Plain files and structured CLI/JSON output are valid first consumers. IDE integrations, libraries, and MCP may be added as adapters when they provide measurable value, but no assistant transport is required by this ADR.

## Consequences

### Positive

- Every repository remains usable without adopting a particular index tool or permanent AI-context files.
- Search provides a cheap, transparent, offline fallback for clean, dirty, partial, and unsupported workspaces.
- Generated indexes can be evaluated and adopted incrementally without making experimental tooling part of the ecosystem contract.
- Exact source, build, dependency, generator, and coverage identity becomes part of the evidence model.
- The architecture supports several assistants without coupling the hub to MCP, an IDE, or one vendor-specific protocol.
- A future central registry can distribute exact released or committed indexes without becoming the owner of satellite source.
- WildFly-specific enrichment can be added only where generic source-navigation tools do not provide sufficient evidence.

### Negative

- Search-first behavior may leave repeated caller and impact tasks slower than a verified graph or precise index.
- Assistants and adapters must understand provenance, coverage, and unavailable-context states instead of treating every result as equally authoritative.
- Freshness validation adds implementation and operational complexity before any generated result can be trusted.
- A future index registry would require artifact retention, verification, access control, lifecycle, and cache policies.
- The hub will not provide one complete answer for every cross-repository question when source or compatible artifacts are unavailable.
- Quantitative assistant savings from generated indexes remain unproven and may not justify their setup or storage cost.

## Alternatives Considered

### Mandatory repository-local indexes

Rejected as the baseline. This would require immediate satellite changes, complicate old branches and dirty worktrees, and make experimental tooling a prerequisite. It may be reconsidered for a specific repository after demonstrated value.

### SCIP-first precise navigation

Rejected as the architecture decision. SCIP remains a possible interchange format, but the experiments did not establish a reliable task-facing lookup path or assistant-level savings. The architecture must not depend on a format whose generation, coverage, and query behavior are not yet validated for WildFly.

### CBM-first local graph navigation

Rejected as the default. CBM produced useful structural and caller-navigation leads, but stale-state behavior, cache cost, daemon setup, and incomplete semantic coverage require an external guard and search fallback. It remains an optional experimental provider.

### Central immutable index registry first

Rejected as the first architectural dependency. A manifest-first registry and compressed artifact distribution are plausible, especially for exact released revisions, but central ownership, artifact authenticity, CI publication, retention, and assistant value remain unresolved. Registry work should follow a demonstrated need for shared exact-revision indexes.

### Custom Bob-tags-style index as the primary solution

Rejected as the immediate direction. Bob-tags-style extraction may be useful for simple declarations or WildFly-specific facts, but generic source navigation already has established formats and systems. Custom extraction should be limited to relationships that generic tools repeatedly fail to expose.

### Monolithic remote source-intelligence service

Rejected as the baseline. A remote service introduces availability, authentication, source exposure, tenancy, cache invalidation, and operational costs before the assistant value of generated indexes has been established. It may be evaluated later as a distribution or query adapter.
