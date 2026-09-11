# Source Intelligence Architecture Options

**Status:** Draft for review

**Purpose:** Synthesize Experiments 0001–0005 into architecture options for making assistant work across the WildFly ecosystem cheaper, faster, and safer. This document is an investigation result and review artifact. It is not an ADR, implementation specification, or implementation plan.

## Executive summary

The experiments do not support building a single complete WildFly source graph or making generated indexes a prerequisite for repository work.

The current architecture direction supported by the evidence is a search-first, evidence-aware baseline:

- the hub acts as a repository, component, version, ownership, and provenance catalog;
- ordinary local search remains the default and mandatory fallback;
- local code-intelligence indexes and published indexes remain optional accelerator hypotheses;
- any generated state is usable only after an external identity/freshness gate validates it;
- binaries, source JARs, feature-pack artifacts, documentation, and hub routing are separate evidence classes;
- unavailable dependency source is reported explicitly rather than inferred;
- MCP, CI indexes, remote services, committed indexes, and satellite-repository changes are not prerequisites for the first prototype.

The hybrid model described later is a possible composition of these capabilities, not a current decision. No generated-index option has yet demonstrated enough end-to-end value to be selected.

## Evidence base

### Experiment 0001: source navigation and assistant cost

[Findings](experiments/results/0001-source-navigation-findings.md)

- Ordinary `rg` plus bounded reads was fastest and complete for small navigation tasks.
- Ctags/Bob-style declarations were cheap but did not demonstrate caller or task savings.
- SCIP generation succeeded only after substantial Maven reactor and SNAPSHOT preparation; task-facing lookup was not established at that stage.
- JDTLS setup was expensive and did not demonstrate cross-file results in the harness.
- CBM produced useful structural and caller results, but its generic graph did not model WildFly-specific relationships.

### Experiment 0002: end-to-end code intelligence and freshness

[Findings](experiments/results/0002-code-intelligence-follow-up-findings.md)

- CBM repository-root generation worked, but root databases were large and warm-query savings were not demonstrated.
- CBM did not automatically create Core-to-Full relationships.
- SCIP repository-root generation succeeded in the recovery run, but the available lookup clients did not expose reliable source locations or references.
- MCP was a viable access path but added daemon/IPC/setup concerns.

### Experiment 0003: CBM freshness and repeated-task value

[Findings](experiments/results/0003-cbm-freshness-and-repeated-task-findings.md)

- CBM was useful for caller and structural navigation evidence.
- Unguarded CBM was unsafe: it reused old state after branch switches and remained `ready` with dirty files.
- Explicit refresh worked, including after declaration, rename, deletion, and branch changes, but the tested full refresh took about 15 seconds.
- Repeated tasks did not demonstrate end-to-end savings over ordinary search for the small task corpus.
- A guard must validate repository identity and source state before returning results.

### Experiment 0004: partial workspaces and dependency context

[Findings](experiments/results/0004-partial-workspace-and-dependency-context-findings.md)

- A feature-pack-only checkout could answer registry, layer, version, and routing questions.
- Feature-pack ZIPs provided declarations and configuration, not runtime implementation source.
- Matching binary/source JARs enabled narrow type/API answers, but did not prove the implementation behind a feature-pack capability.
- Hub documents improved routing and ownership understanding but did not prove an exact implementation revision.
- The correct response to absent runtime source is explicit unavailable context, not a guessed implementation or negative claim.

### Experiment 0005: immutable index registry and distribution

[Findings](experiments/results/0005-immutable-index-registry-findings.md)

- A local manifest-first registry resolved exact commit entries, a tag alias that remained pinned during the test, and mutable branch pointers without basename collisions.
- Checksums, explicit lifecycle statuses, cache keys, LRU eviction, tag pinning, and offline reuse behaved safely in the disposable simulation.
- Zstandard reduced the representative CBM core database from 400 MB to 32.7 MB; ZIP reduced it to 52.3 MB. Both still required full materialization before SQLite queries.
- The experiment supports separating compact manifests/pointers from large compressed artifacts, but does not select Git LFS, release assets, object storage, or a remote service.
- Partial feature-pack evidence remained separate from runtime index evidence.

### Experiment 0006: source-navigation task value comparison

[Findings](experiments/results/0006-source-navigation-task-value-findings.md)

- Ordinary search reached the source-reviewed answers for all six bounded task cards and remained the only provider with transparent live coverage.
- CBM supplied useful caller and structural-navigation leads in prior exact-commit runs, but its stale-state behavior requires an external identity guard and source verification.
- SCIP generation produced symbol/document metadata, but the tested lookup path exposed no usable occurrences, locations, or references. SCIP therefore remains a format/generation possibility rather than a selected assistant backend.
- Apple/BSD Ctags was not a meaningful Java declaration provider; this is a setup limitation and does not evaluate Universal Ctags or Bob-style output.
- No assistant/model harness was available, so the experiment did not establish model-token, tool-call, or end-to-end assistant savings. Generated-source, dependency-input, and SNAPSHOT freshness were also not tested.
- The result supports search-first with optional guarded accelerators, but does not justify a mandatory index or another broad provider comparison.

## Prior art and established patterns

The source-navigation problem is established, but it is split across several layers. Existing projects should inform the interfaces and operating model; they do not by themselves answer which WildFly-specific relationships are worth indexing.

### Interchange formats and graph schemas

- [SCIP](https://scip-code.org/) is a language-agnostic protocol for source indexes used for definitions, references, implementations, and related navigation. It is the closest established interchange format for generic precise source navigation in this investigation.
- [LSIF](https://github.com/microsoft/lsif-node) serializes language-server knowledge so navigation requests can be served later without running a language server for every request.
- [Kythe](https://kythe.io/docs/schema/) provides a graph schema with explicit corpus/path/signature identity. Its documentation also makes clear that symbol identity can vary with indexer versions, which reinforces the need to include generator and schema identity in the WildFly contract.

These formats describe facts or navigation edges. They do not, by themselves, provide repository discovery, build execution, freshness validation, artifact retention, permissions, or an assistant workflow.

### Indexing and serving systems

- [Sourcegraph precise code navigation](https://sourcegraph.com/docs/code-navigation/precise-code-navigation) combines SCIP indexes with search-based navigation. Its [CI guidance](https://sourcegraph.com/docs/code-navigation/how-to/adding-scip-to-workflows) treats exact-commit CI generation as the reliable path for complex builds, while allowing lower-frequency indexing where indexing every commit is too expensive.
- Sourcegraph's [auto-indexing policies](https://sourcegraph.com/docs/code-navigation/auto-indexing) are prior art for selecting branches, tags, commits, schedules, dependency indexes, and failure states. Its documented fallback to search is especially relevant when a precise index is absent or failed.
- [Glean](https://engineering.fb.com/2024/12/19/developer-tools/glean-open-source-code-indexing/) demonstrates a much larger central architecture: distributed indexing, revision-aware facts, cross-references, and a query service. It validates the central-service pattern but is not evidence that WildFly should adopt a comparable platform.
- [OpenGrok](https://github.com/oracle/opengrok) demonstrates the value of a central searchable source corpus with definitions, cross-references, source browsing, and incremental updates, without requiring full compiler-level precision.

The main architectural lesson is to separate the index format, generator, artifact catalog, query layer, and assistant adapter. A secondary repository can own manifests and pointers without owning the source repositories or being the index generator.

### Adjacent tools

- Universal Ctags remains useful as a low-cost declaration baseline.
- [CodeQL database extraction](https://docs.github.com/en/code-security/reference/code-scanning/codeql/codeql-cli-manual/database-create) is relevant for custom semantic analysis, but its database/query model is not automatically a replacement for ordinary source navigation. This is an inference from its documented analysis-oriented interface and should be tested only for specific WildFly relationships.

### Consequences for this investigation

Experiment 0006 did not ask whether WildFly can reproduce every Bob-tags feature. It compared whether established layers reduce work on representative tasks:

1. search and bounded source reading;
2. cheap declaration indexing;
3. precise or graph-based navigation;
4. WildFly-specific evidence where generic navigation is insufficient.

The comparison must measure task correctness, tool calls, source lines opened, model tokens, latency, index generation/refresh cost, and stale or false-completeness failures. A central registry should be evaluated as distribution infrastructure independently from the question of which index facts are valuable.

## Evidence, assumptions, and unresolved questions

### Evidence

| Claim | Source | Confidence and scope | Status |
|---|---|---|---|
| Small navigation tasks do not justify mandatory index generation | 0001, 0002, 0003 | High for the small task corpus; not a claim about larger impact tasks | Observed |
| Generic indexes can provide useful caller and structural navigation leads | 0001, 0002, 0003 | Moderate; caller/impact precision was sampled, not exhaustively reviewed | Observed |
| Generated state can be expensive, large, build-coupled, and stale | 0001, 0002, 0003 | High for tested tools and environments | Observed |
| Branch/ref changes are a hard freshness boundary unless an exact matching index is available | 0002, 0003 | High for CBM; other tools remain untested | Observed |
| Partial workspaces can answer declarations and routing but not absent runtime implementation questions | 0004 | High for the feature-pack case; broader ecosystem scope is untested | Observed |
| Repository routing and provenance remain valuable when source is unavailable | 0004 | Medium; measured in one feature-pack case | Observed |
| A manifest-first registry can resolve exact commit artifacts while keeping tags immutable and branches as aliases | 0005 | High in a disposable local simulation; remote distribution was not tested | Observed |
| Compression can materially reduce index transfer/storage size; compressed SQLite still requires full materialization for normal queries | 0005 | High for CBM compression/decompression and SQLite integrity; SCIP compression was measured but not queried | Observed |
| Large generated artifacts should be separated from normal Git history | 0005 | High for the measured CBM artifacts; backend choice remains unresolved | Observed |
| Search reached all six bounded task answers while index-backed providers remained partial or setup-limited | 0006 | High for the selected task cards and source revisions; not a broad workload benchmark | Observed |
| CBM can provide caller/structural navigation leads but must be guarded against stale state | 0003, 0006 | Moderate; caller value was sampled and assistant savings were not measured | Observed |
| The tested SCIP lookup path did not expose usable occurrences or source locations | 0002, 0006 | High for the generated artifacts and conversion path tested; not evidence that all SCIP consumers fail | Observed |
| Index-backed approaches reduce end-to-end assistant effort | None | No model/tool-call harness was available | Unresolved |
| Multiple assistants can consume one provenance-aware result model | None | Not measured | Assumption |
| The model generalizes to all controlled repositories outside the runtime umbrella | None | Optional non-runtime case was not run | Untested |

### Assumptions to validate

- A source-state guard can reject stale indexes cheaply enough for interactive use.
- Repeated caller/impact tasks on a larger corpus may amortize optional index generation.
- A structured provenance result can be consumed by several assistants without requiring MCP.
- A fixed assistant harness will show measurable savings for at least some caller/impact or repeated-navigation tasks.
- A Java-aware declaration provider will offer useful value beyond ordinary search at acceptable setup cost.
- Source JARs and Maven metadata are useful enough to include in the first partial-workspace prototype.
- A signed manifest plus a separately stored compressed artifact can provide sufficient trust and operational control for shared distribution.

### Unresolved questions

- Can CBM perform a genuinely incremental/path-limited refresh, or only a full refresh?
- How much disk and startup cost is acceptable for a developer checkout?
- How should generated sources and build/dependency inputs participate in source identity?
- Should selected released versions receive CI-published indexes later?
- How should the catalog represent repositories outside the main WildFly runtime repositories?
- Which WildFly-specific relationships justify a separate enrichment layer?
- Does MCP materially improve assistant outcomes after freshness/provenance guards, or only provide transport portability?
- Which catalog metadata is needed for repositories with different build systems, release policies, or documentation conventions?
- Which artifact backend provides acceptable cost, access control, retention, and offline behavior for compressed indexes?
- Who owns and governs a shared index registry, and how does it remain distinct from repository compatibility/feature-pack registries?
- What happens when an exact commit artifact is missing, expired, failed, or cannot be authenticated?

### Terms requiring explicit interpretation

- **Evidence class:** the origin of a result, such as local source, source JAR, binary/API metadata, feature-pack artifact, hub documentation, or inference.
- **Completeness:** the scope within which a result is exhaustive; it is not implied by an index being marked `ready`.
- **Coverage:** the files, artifacts, repositories, versions, and generated/excluded paths represented by a provider.
- **Navigation hint:** a lead that helps locate code but has not been verified as an exact semantic relationship.
- **Source state:** the combination of repository identity, checkout/ref, commit, working-tree changes, generated inputs, dependency/build inputs, and tool/index configuration that can affect results.
- **Matching index:** an index whose recorded identity and coverage are valid for the requested source state and query.

## Architectural principles

1. **Evidence before inference.** Every result should identify whether it came from local source, a source JAR, a binary, a feature-pack artifact, documentation, a hub profile, or inference.
2. **Partial context is normal.** Missing repositories and artifacts are expected states, not errors that should be concealed.
3. **No false negatives from incomplete coverage.** An incomplete index must not support claims such as “no callers” or “no implementation.”
4. **Identity is part of the data.** Repository root, repository identity, ref/commit, dirty state, source digest, tool/schema version, and relevant build inputs must accompany generated results.
5. **Search is the safety net.** Generated state may accelerate work, but ordinary search remains available whenever indexes are absent, stale, partial, or too expensive.
6. **The hub routes; it does not pretend to own all source.** The hub should describe ownership and relationships without becoming a monolithic cross-repository source database.
7. **Assistant interfaces are adapters.** Plain files, CLI/JSON, IDE integrations, and MCP should consume the same provenance-aware result model where practical.
8. **Catalog entries have provenance too.** Repository profiles and topology claims should carry their source/revision, owner, verification date, and trust/status metadata; a hub link is not proof that a target source revision was inspected.

## Proposed source-state policy

This policy is a design hypothesis for review, not an accepted contract.

| Source state | Minimum detection | Proposed behavior |
|---|---|---|
| Clean checkout at matching commit | Canonical root, repository identity, exact commit, tool/schema/configuration match | Index results may be used within recorded coverage |
| Detached HEAD | Exact commit; branch label is optional metadata | Treat the commit as the identity; do not rely on a branch name |
| Shallow or incomplete checkout | Git completeness and missing-object checks | Warn about unavailable history and avoid claims requiring it |
| Staged or unstaged tracked edits | Git status and changed-path classification | Use search for changed files; refresh or downgrade graph results according to change type |
| Untracked source files | Untracked-path detection and content identity | Treat as uncovered until indexed or searched directly |
| Ignored/generated files changed | Build/generated-input policy and path classification | Invalidate affected generated or dependent evidence; do not assume ignored means irrelevant |
| Rename or deletion | Path/status transition | Hard-invalidate old paths until refresh |
| Branch/ref/commit change | Exact commit comparison, not branch label alone | Reject the previous index unless an exact matching index is available |
| Maven/SNAPSHOT/dependency/build-input change | Resolved coordinates, timestamped metadata, checksums, toolchain/build configuration | Refresh or downgrade results whose classpath/build inputs may differ |
| Force-pushed or ambiguous remote branch | Local commit and remote identity, where available | Use the local commit as identity; never treat a branch name as immutable |
| Parse-partial or excluded coverage | Provider status and coverage output | Permit navigation hints, but prohibit exhaustive or negative claims |

The external guard must distinguish a cheap identity check from a potentially expensive source digest. A commit/status mismatch can reject an index without hashing the entire tree. A digest should only be computed when the policy needs to decide whether a dirty state can reuse or partially refresh an index.

### Index identity contract

A commit SHA alone is not sufficient when generated sources, dependency resolution, compiler/toolchain settings, build configuration, or SNAPSHOT artifacts can affect the generated result. An authoritative index identity should therefore include:

- repository identity and exact source commit;
- working-tree/generated-source state, or an explicit clean-state claim;
- dependency and build-input fingerprints, including timestamped SNAPSHOT metadata where applicable;
- compiler/toolchain and relevant build configuration;
- generator, parser, schema, and configuration versions;
- coverage, exclusions, and parse-partial status;
- compressed artifact and, where practical, uncompressed-content checksums.

If these inputs cannot be proven, the result must be labelled partial or non-authoritative for the affected questions. An artifact version or checksum must not be presented as the source commit that produced the implementation unless an authoritative source-to-artifact attestation exists.

## Cross-cutting evaluation criteria

Every option and prototype should be evaluated against the same criteria:

- **Correctness:** exact results, approximate leads, false positives, and false-completeness claims.
- **Coverage:** source roots, generated sources, dependency artifacts, excluded files, parse-partial files, and cross-repository scope.
- **Freshness:** clean, dirty, renamed, deleted, branch-switched, detached, shallow, and SNAPSHOT states.
- **Provenance:** repository/ref/artifact/checksum/source mapping, catalog revision, owner, verification date, and trust/status metadata.
- **Assistant cost:** model tokens, tool calls, files/lines opened, time to first useful result, and total task time.
- **Operational cost:** generation, refresh, startup, disk, memory, eviction, schema migration, failure recovery, and concurrency.
- **Security and privacy:** artifact verification, untrusted build execution, cache isolation, daemon/socket permissions, MCP authentication, secrets/prompt-injection content, and source-path exposure.
- **Distribution:** installation, offline operation, network requirements, and compatibility with several assistants.

## Common conceptual architecture

```text
assistant task
    |
    v
workspace/context resolver
    |-- repository root, branch, commit, dirty paths
    |-- build/dependency coordinates and available artifacts
    |-- hub repository/profile/topology metadata
    v
evidence providers
    |-- local search and source files
    |-- optional local structural index
    |-- source JARs, binaries, Javadocs
    |-- feature-pack/configuration artifacts
    |-- hub routing and ownership documents
    v
freshness + provenance + completeness gate
    |
    v
assistant-facing result
    |-- evidence and version identity
    |-- confidence/coverage limitations
    |-- source locations or routing links
    |-- fallback or unavailable-context explanation
```

The architecture does not require all providers to exist. The resolver should select the strongest available evidence and make its limitations visible.

## Options

### Option A: Search-first hub catalog

#### Components

- the existing hub `llms.txt` and architecture documents;
- versioned repository profiles and ownership/topology metadata;
- local search and bounded source reading;
- Maven/dependency inspection for partial workspaces;
- optional source JAR and documentation discovery.

#### Data flow

The assistant starts with the current repository and hub route. It inspects local source and build metadata, then follows exact dependency coordinates or repository links when deeper context is needed. Missing source is reported explicitly.

#### Version and source-state handling

The current checkout and resolved artifacts provide identity. No generated graph needs to be retained. SNAPSHOTs require timestamp/checksum evidence; branch/source mappings remain explicit rather than inferred.

#### Assistant integration

Plain files and ordinary CLI tools work immediately. JSON/TSV result conventions could be added later. MCP is optional.

#### Strengths

- smallest and safest first prototype;
- no index generation or cache lifecycle;
- naturally handles dirty worktrees and partial workspaces;
- no satellite changes required;
- can be applied to repositories outside the WildFly runtime umbrella, subject to profile validation.

#### Weaknesses and risks

- repeated caller/impact tasks remain expensive for assistants;
- routing does not provide semantic source relationships;
- quality depends on repository profiles and documentation;
- each assistant may rediscover similar context.

### Option B: Guarded local code-intelligence backend

#### Components

- Option A as the baseline;
- an optional local structural/semantic index such as CBM;
- an external identity and freshness guard;
- explicit refresh and cache lifecycle;
- a plain/structured query interface, with MCP as an optional adapter.

#### Data flow

The resolver validates the checkout before querying the index. If identity matches, the assistant may use graph results. If the checkout is dirty or structurally changed, the guard either refreshes, downgrades the result to a navigation hint, or falls back to search. A branch/ref mismatch is a hard invalidation.

#### Version and source-state handling

Index entries are associated with repository identity, canonical root, ref/commit, dirty/source digest, generator/schema version, configuration, and relevant build inputs. Exact clean commits may be reused; dirty states should be treated as separate or temporary.

#### Assistant integration

The first interface should be plain structured output or a CLI. MCP can expose the same guarded operations to multiple assistants but must not bypass the guard.

#### Strengths

- can provide callers, structural neighborhoods, and impact leads;
- does not require all repositories to adopt permanent files;
- could support queries from verified local state after setup; offline refresh remains untested;
- can remain optional for tasks that benefit from it.

#### Weaknesses and risks

- CBM freshness is unsafe without an external guard;
- cache size and daemon/permission costs are material;
- semantic precision is incomplete for WildFly-specific relationships;
- no end-to-end cost win has yet been demonstrated;
- a wrapper becomes an important architectural component.

### Option C: Central immutable index registry

#### Components

- a secondary Git repository containing compact manifests and branch/tag pointers;
- index generation in controlled CI environments;
- immutable compressed artifacts stored through LFS, release assets, object storage, or an equivalent artifact store;
- artifacts keyed by repository/ref/commit/tool/schema/build inputs;
- local download/cache and assistant adapters;
- Option A fallback.

This proposed registry is distinct from `wildfly-galleon-feature-packs`, whose existing role is feature-pack compatibility and provisioning metadata. Ownership, governance, and whether the registry belongs in or alongside the hub remain open decisions.

#### Data flow

After a merge or tag event, CI would generate an index for the exact commit. A scheduled reconciliation job would detect missed events. CI would publish an immutable commit manifest and compressed artifact, then advance a branch alias only after successful verification. An assistant would first retrieve a small manifest, then download the exact artifact only when a matching commit, generator/schema, coverage, and checksum are acceptable. If no match exists, it uses local search or other evidence. Experiment 0005 demonstrated these mechanics only in a disposable local registry and localhost simulation; CI event handling, central ownership, WAN distribution, and shared-service governance remain unvalidated.

#### Version and source-state handling

Commit manifests would be immutable and keyed by repository identity, exact commit, dependency/build fingerprints, generator/schema/configuration, coverage, and checksums. Branch names would be mutable aliases and never artifact identity. A tag alias may be treated as immutable only when registry policy rejects attempted mutation; the Git tag name alone is not an enforcement mechanism. Clean commits and release artifacts are easier to identify than dirty worktrees. SNAPSHOTs require timestamped artifact metadata, exact dependency/source mapping where available, and a clear retention policy. Local uncommitted changes cannot be represented without local overlay or regeneration.

#### Assistant integration

Plain downloadable artifacts, a CLI, library, or MCP could consume the indexes. The manifest would expose provenance and coverage before the artifact is downloaded. Any assistant adapter would need to preserve complete/partial/failed/unavailable status and checksum verification; MCP could not bypass the same checks.

#### Strengths

- moves expensive generation away from developers;
- reusable and traceable indexes for stable releases;
- potentially useful when source is not checked out locally;
- could support multiple assistants through a shared manifest/artifact contract;
- permits exact historical commit lookup while keeping branch aliases convenient.

#### Weaknesses and risks

- operational and storage cost;
- stale or missing coverage for branches, SNAPSHOTs, and private changes;
- source paths and metadata may be exposed through artifact storage;
- requires CI adoption and artifact retention policy;
- does not solve dirty-worktree behavior;
- signatures, resumable downloads, multi-writer recovery, and backend access control still require validation;
- compressed SQLite still requires complete local materialization before normal queries, so compression does not provide remote or incremental query access.

#### Deferred variant: live remote query service

A live remote service is not equivalent to immutable artifact publication. It introduces separate availability, authentication, tenancy, source-path exposure, request logging, cache invalidation, and service-version risks. It should be evaluated later only if immutable artifacts cannot provide the required assistant experience.

### Option D: Hybrid progressive evidence composition

#### Components

- Option A as the mandatory base;
- hub-owned repository/component/version catalog;
- local resolver for source, build metadata, binaries, source JARs, and documentation;
- optional guarded local indexes;
- optional central immutable index registry for selected exact revisions;
- common provenance/completeness result model;
- assistant adapters including CLI, plain files, IDE integration, or MCP.

#### Data flow

The resolver chooses the strongest available evidence in this order:

1. current local source and clean matching local index;
2. current local source with ordinary search;
3. exact source JAR or released index for the resolved artifact;
4. binary/API and feature-pack metadata;
5. hub routing/topology documentation;
6. explicit unavailable-context result.

No layer claims completeness beyond its coverage. The resolver can use a generated index for a hard graph task while still using ordinary search to verify changed files and current line ranges.

Evidence providers may be combined only when their repository, source/artifact, version, and build-input identities are compatible for the question being answered. Otherwise the result must expose separate evidence branches and the identity conflict. A feature-pack declaration must never be silently combined with an older or unrelated runtime index to answer an implementation question.

#### Version and source-state handling

Local source states are keyed by checkout identity and source state. Released artifacts and CI indexes are keyed by exact coordinates, checksums, refs, generator/schema versions, and build inputs. SNAPSHOTs are never mapped implicitly to a branch or Final release.

#### Assistant integration

The underlying result model is structured and provenance-aware. Plain files and CLI output are the first integration targets. MCP and IDE integrations are adapters, not core dependencies.

#### Strengths

- handles complete and partial workspaces;
- keeps search available for every failure mode;
- allows local and CI generation to coexist;
- is designed to support repositories outside the runtime umbrella, subject to catalog validation;
- permits gradual adoption without requiring every satellite to change;
- makes evidence boundaries visible to several assistants.

#### Weaknesses and risks

- largest conceptual and operational scope;
- requires a resolver and provenance contract;
- multiple providers can create confusing precedence rules;
- CI/remote indexes may still be unnecessary if local search is adequate;
- WildFly-specific enrichment remains a separate problem.

Option D is a composition hypothesis, not a mutually exclusive alternative to Options A–C. It should not be considered selected merely because it contains the other options.

## Comparison

Options A–C represent a baseline and independent accelerator choices. Option D describes a possible composition of them; the table is not a ranking in which D automatically dominates the others.

| Dimension | A: Search-first baseline | B: Local accelerator | C: Central registry accelerator | D: Possible composition |
|---|---|---|---|---|
| First prototype effort | Lowest | Medium | High | Composition-dependent; not independently evaluated |
| Small one-off tasks | Excellent | Usually worse due setup | Usually worse due retrieval | Search path remains excellent |
| Repeated caller/impact tasks | Limited | Potentially strong | Potentially strong | Optional strong path |
| Dirty worktrees | Natural | Requires guard/overlay | Poor without local regeneration | Search fallback plus guarded index |
| Branch switching | Natural | Hard invalidation needed | Exact artifact selection | Exact identity plus fallback |
| Missing dependency source | Routing/artifact metadata | Only indexes present source | Can provide selected released indexes | Layered evidence and explicit gaps |
| SNAPSHOT support | Build metadata | Local setup required | Timestamped artifact policy | Both local and exact artifact paths |
| Multiple assistants | Plain files/tools | CLI plus optional adapters | Shared artifacts/service | Shared result model plus adapters |
| Satellite changes required | None | None initially | CI/integration changes eventually | None initially, optional later |
| Operational cost | Low | Medium/high | Medium/high | Starts low, grows selectively |
| Main risk | Limited acceleration | Stale/incorrect graph | Coverage and infrastructure | Complexity and precedence |

## Provisional recommendation

The current decision supported by the evidence is Option A: a search-first hub catalog with explicit provenance, coverage, and freshness requirements.

Option B remains an unapproved local accelerator hypothesis. Option C has locally demonstrated registry mechanics from Experiment 0005, but its central ownership, CI publication, production artifact backend, authenticity, and assistant value remain unvalidated. Option D is a possible composition, not a selected architecture. The experiments have not demonstrated repeatable end-to-end assistant cost savings sufficient to make any generated index mandatory.

Experiment 0006 strengthens the search-first conclusion. It found that search reached all six bounded task answers, while CBM remained useful only as guarded navigation evidence and the tested SCIP lookup path remained incomplete. Because no assistant/model harness was available, the experiment does not prove that indexes fail to reduce model work; it leaves that as a narrow optional question rather than a reason to broaden the architecture.

The next sequence should be:

1. define the hub catalog, repository profile, dependency identity, evidence, and completeness concepts;
2. build or prototype search-first partial-workspace routing without requiring satellite AI-context files;
3. treat the 0006 provider comparison as sufficient evidence for the baseline and defer another broad tool comparison;
4. run a narrow assistant-harness benchmark only if quantitative proof of model/tool-call savings is needed;
5. evaluate a guarded local index only if that benchmark demonstrates value and safe freshness behavior;
6. consider the central immutable registry for exact released versions only after artifact trust, distribution, and assistant value are proven;
7. add WildFly-specific enrichment only for relationships that generic tools repeatedly fail to expose.

This current direction deliberately does not select CBM as a permanent dependency, SCIP as a backend, MCP as a required interface, a remote service, or a committed index format.

## Prototype boundaries

The experiments and review indicate that the prototype should be split into three independent gates: a search-first capability gate, a guarded local-index gate, and a distribution/registry mechanics gate. Combining them would make it difficult to tell which assumption failed.

### Prototype 1: search-first provenance and routing

This is the smallest prototype supported by current evidence.

#### Inputs

- one feature-pack-only checkout with `wildfly` and `wildfly-core` absent;
- one ordinary source checkout with a dirty-worktree state;
- hub repository profiles, topology, and boundary documents;
- Maven dependency coordinates and already-available binary/source artifacts.

#### Behavior

- identify repository root, exact commit where available, dirty state, and dependency coordinates;
- route component questions to the owning repository/profile;
- report locally available evidence and explicit unavailable dependencies;
- use ordinary search for local source;
- inspect exact binary/source artifacts when present;
- emit structured results with provenance and completeness labels.

#### Acceptance criteria

- no automatic source cloning or build execution;
- exact dependency versions and checksums are preserved where available;
- feature-pack declarations are not presented as runtime implementation evidence;
- absent runtime source produces an explicit limitation rather than an invented or negative answer;
- the same result can be rendered as plain text and structured data.

This gate establishes the baseline provenance/routing contract. It does not establish assistant savings from generated indexes or production registry distribution.

### Prototype 2: guarded local-index experiment

This is a separate evaluation gate, not part of Prototype 1’s minimum implementation.

#### Inputs and behavior

- the same source identity and provenance data as Prototype 1;
- an optional CBM index for one source checkout;
- clean, dirty, branch-switched, rename/delete, generated-source, dependency/build-input, SNAPSHOT, and partial-workspace states;
- ordinary search as the mandatory fallback;
- one structured interface and one second consumer, such as MCP or an independent CLI client.

#### Acceptance criteria

- no index result is returned for a mismatched commit/ref or unclassified dirty state;
- refresh and fallback behavior are explicit and measurable;
- caller/impact results are labeled as exact or navigation evidence and reviewed against source;
- total assistant cost, refresh cost, storage, and permission friction are measured;
- at least five tasks, including at least three caller/impact or repeated-navigation tasks, are run in at least three comparable repetitions;
- the candidate reduces median assistant calls, source lines opened, or model tokens by at least 20% on at least three tasks without reducing correctness or introducing a false-completeness result;
- branch/ref changes and structural file changes have zero silent-stale-result failures;
- generated-source, dependency/build-input, and SNAPSHOT transitions are either tested explicitly or recorded as incomplete guarantees that exclude the backend from the default path;
- indexing/refresh time and cache size are reported against provisional usability budgets of 30 seconds and 2 GiB per tested repository; exceeding either excludes the backend from the default path but does not automatically exclude an optional path.

This gate establishes whether a guarded local accelerator is worth considering. It does not establish shared registry distribution.

### Prototype 3: immutable registry distribution pilot

This is a packaging and distribution gate for exact clean commits, independent of whether CBM becomes the selected local backend.

#### Inputs and behavior

- one or two real compressed indexes and their manifests;
- a disposable manifest repository and local artifact store/HTTP server;
- immutable commit entries, registry-pinned tag aliases whose mutation policy is explicit, and mutable branch pointers;
- manifest-first retrieval, checksum verification, local cache reuse, and eviction;
- one partial feature-pack workspace with runtime source absent.

#### Acceptance criteria

- commit identity is never confused with branch name, tag name, repository basename, or nominal version;
- failed, partial, unavailable, and checksum-invalid artifacts are refused or reported explicitly;
- large artifacts are not placed in normal Git history;
- cache entries are reusable offline only after verification;
- compressed-artifact decompression/materialization time, peak disk usage, and startup cost are measured;
- artifact backend, retention, signing, resumable download, atomic publication, cache isolation, and multi-writer recovery are tested or explicitly excluded from shared-distribution approval;
- checksum verification is treated as corruption detection only; signed manifests or an equivalent trusted publication mechanism are required before authenticity is claimed;
- feature-pack evidence cannot be silently combined with an older or unrelated runtime index.

This gate establishes registry mechanics and checksum-validated reads/offline reuse. It does not establish central ownership, CI/event reliability, WAN performance, artifact authenticity, or assistant task savings unless those additional controls are separately validated.

### Explicit non-goals for all prototypes

- no full ecosystem graph;
- no automatic source cloning;
- no live remote service;
- no committed generated indexes;
- no external GitHub repository, release, Pages site, LFS store, or object-store publication;
- no requirement for satellite repositories to adopt permanent files;
- no production MCP integration;
- no WildFly-specific semantic graph until a concrete repeated need is demonstrated.

## Follow-up work decomposition

| Task | Objective | Repositories/files to inspect | Deliverable | Dependencies | Parallel? | Decision informed |
|---|---|---|---|---|---|---|
| Identity and freshness | Define the minimum source/index identity and stale-state policy | 0002/0003 findings; Git worktree behavior; CBM status/refresh output | Identity and freshness contract with measured digest/refresh costs | 0003 evidence | Yes | Whether guarded local indexes are safe |
| Partial-workspace resolver | Specify evidence precedence for source, artifacts, source JARs, and hub routing | 0004 findings; feature-pack `pom.xml`, registry/catalog files; hub `llms.txt` and topology | Provenance/completeness matrix and resolver behavior | 0004 evidence | Yes | Whether the hub can support absent dependencies |
| Repository catalog | Generalize profiles beyond `wildfly`/`wildfly-core` | Hub `llms.txt`; topology; boundaries; `wildfly-galleon-feature-packs`; optionally `wildfly-glow` and `wildfly-maven-plugin` profiles | Catalog fields and onboarding/version mapping proposal | Partial-workspace resolver | Yes | Whether the hub scope supports controlled non-runtime repositories |
| Assistant interfaces | Compare plain files, structured CLI/JSON, IDE, and MCP as adapters | Existing hub/satellite `llms.txt`; 0002/0003 MCP observations; assistant configuration conventions | Interface-neutral result model and adapter comparison | Identity and resolver contracts | Yes | Whether MCP is required, optional, or deferred |
| WildFly enrichment | Identify relationships generic indexes cannot expose | `architecture/ecosystem-topology.md`, `architecture/repository-boundaries.md`; management, provisioning, service, and feature-pack docs | Prioritized enrichment candidates with evidence sources | Partial-workspace resolver | Yes | Whether domain-specific indexes are justified |
| Prototype 1 evaluation | Run the search-first provenance/routing prototype across complete and partial workspaces | Feature-pack checkout plus one source checkout; existing experiment fixtures | Provenance, routing, unavailable-context, and cost/correctness findings | Partial-workspace resolver | After resolver/catalog tasks | Whether the baseline contract is useful |
| Prototype 2 evaluation | Separately evaluate a guarded local index against search | Fresh source checkout, CBM, branch/dirty fixtures, two structured consumers | Freshness, fallback, task-value, storage, and permission findings | Identity/freshness contract; Prototype 1 result model | After Prototype 1; independent of catalog polish | Whether any local index accelerator is justified |
| Prototype 3 evaluation | Validate immutable manifests, compressed artifacts, aliases, verification, and cache behavior | 0005 findings; disposable registry; representative compressed indexes; feature-pack partial workspace | Registry manifest/artifact contract and backend requirements | Provenance contract; 0005 evidence | Yes, independent of Prototype 2 | Whether shared exact-commit distribution is viable |
| Task-value comparison (completed) | Compare search, declarations, graph navigation, and precise-index providers on real assistant tasks | 0006 charter; one or two real WildFly repositories; existing 0001–0005 fixtures; prior-art tools where setup is bounded | [0006 findings](experiments/results/0006-source-navigation-task-value-findings.md) | Source identities and result model; 0001–0005 evidence | Completed | Whether any index-backed approach merits an optional accelerator or WildFly-specific enrichment |
| Assistant-harness benchmark | Measure model/tool-call savings for the already-compared providers without repeating broad setup work | 0006 findings; fixed assistant/model harness; same six task cards and source identities; CBM guard and SCIP limitations | Narrow end-to-end cost/correctness benchmark, or explicit stop decision if harness cost exceeds expected value | 0006 findings; stable task cards and result model | Yes, but only if quantitative savings are required | Whether to authorize an optional guarded accelerator |

## Questions requiring experiments or benchmarks

These should not be settled by discussion alone:

- CBM path-limited versus full refresh cost;
- source-state digest calculation time and false-stale rate;
- repeated caller/impact assistant savings on a larger task corpus;
- whether source JARs materially reduce assistant calls and source reading;
- cache size and eviction behavior across branches and repositories;
- exact-commit index reuse after branch switching;
- MCP versus plain structured output after freshness guards;
- false-completeness rate when dependency source is absent;
- value of CI-published indexes for released versions;
- whether one non-runtime controlled repository requires different catalog/profile fields.
- whether search, declaration, graph, or precise indexes reduce assistant effort on representative tasks without false-completeness or stale-result failures;
- whether a fixed assistant harness changes the 0006 conclusion by demonstrating measurable model/tool-call savings;
- whether a Java-aware declaration provider changes the conclusion reached from Apple/BSD Ctags;
- whether signed manifests, resumable downloads, and multi-writer cache recovery are required before shared distribution;
- which artifact backend provides acceptable cost, access control, retention, and offline behavior.

## Proposed architecture-spec outline

The eventual reviewed architecture specification should contain:

1. Status, goals, non-goals, and terminology.
2. Ecosystem scope and repository/component catalog.
3. Workspace and dependency discovery.
4. Evidence classes and provenance model.
5. Source/index identity and versioning.
6. Freshness, dirty-worktree, and branch-switch behavior.
7. Generation and storage options.
8. Partial-workspace and unavailable-context behavior.
9. Assistant-facing interfaces and adapter responsibilities.
10. WildFly-specific enrichment boundaries.
11. Security, privacy, artifact verification, and untrusted-repository behavior.
12. Offline and failure behavior.
13. Operational lifecycle, cache retention, and CI publication.
14. Prototype scope and acceptance criteria.
15. Open decisions and follow-up experiments.

## Review gate

This draft should be reviewed before:

- adding an ADR;
- implementing a resolver, index wrapper, or cache;
- requiring AI-context files from satellite repositories;
- publishing or committing generated indexes;
- selecting CBM, SCIP, MCP, or a remote service as a project dependency.
