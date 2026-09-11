# Experiment Charter 0004: Partial Workspaces and Dependency Context

**Status:** Proposed

**Purpose:** Determine how an assistant should navigate a repository when important dependencies are not checked out locally, and how the hub should represent repositories and components that are controlled by the WildFly organization but are outside the main WildFly runtime repositories.

This experiment follows Experiments 0001–0003. It does not assume that a complete ecosystem source graph is available, that all repositories have AI-context files, or that a missing dependency can be solved by automatically cloning source.

## Objective

Test whether an assistant can provide useful, honest navigation from a partial workspace containing a feature-pack or ecosystem repository while `wildfly` and `wildfly-core` source checkouts are absent.

The primary question is:

> How should the assistant combine local source, Maven dependency metadata, binary/source artifacts, hub routing information, and unavailable context without making false completeness claims?

The secondary questions are:

- Can the assistant identify the exact dependency version or SNAPSHOT artifact involved?
- Can it distinguish source-backed findings from binary/API/documentation evidence?
- Can it route the user to the owning repository without requiring that repository to be cloned?
- What useful answers remain possible when dependency source is unavailable?
- Does the hub need a broader ecosystem catalog rather than a WildFly-runtime-only repository list?

## Scope and non-goals

The primary repository is `wildfly-galleon-feature-packs`, using a real feature-pack or layer-related task. The primary workspace must not contain source checkouts of `wildfly` or `wildfly-core`.

Optionally include one repository outside the WildFly runtime source umbrella, such as `wildfly-glow` or `wildfly-maven-plugin`, to test repository-profile and routing assumptions. This second repository is supplementary; it must not prevent completion of the feature-pack case.

Do not:

- implement a dependency indexer or remote service;
- clone missing dependency source automatically as part of the baseline;
- modify satellite repositories, build files, or AI-context files;
- commit generated indexes, downloaded artifacts, or caches;
- assume all controlled repositories use the same build system or branch policy;
- claim that an absent dependency has no callers, implementation, or relationship merely because its source is unavailable;
- decide whether source JARs, CI indexes, MCP, or remote services belong in the final architecture.

## Workspace states

Create disposable workspaces for the following states. Record exactly which files and artifacts are present.

### State A: partial source workspace

Checkout the feature-pack repository only. Do not checkout `wildfly` or `wildfly-core` source. Allow the normal build metadata to identify dependencies, but do not prefetch dependency source unless the build does so as an unavoidable side effect.

### State B: partial workspace with resolved binaries

Use the same feature-pack source with its normal Maven-resolved dependency JARs. Record group/artifact/version, repository URL, checksum, SNAPSHOT timestamp or metadata, and local cache path where available.

### State C: partial workspace with source JARs

If matching `-sources.jar` files are available, expose them without adding dependency Git checkouts. Record whether the source JAR version and checksum match the binary artifact and dependency resolution used by the feature pack.

### State D: optional hub/routing context

Provide the current hub `llms.txt`, repository-boundary documentation, topology documentation, and any relevant repository profile without adding dependency source. Measure whether this improves routing and version identification.

If a state cannot be reproduced because an artifact is unavailable, record the absence and continue. Do not substitute a different version silently.

## Candidate assistant conditions

### Local source and build inspection

Use ordinary search, directory listing, bounded reads, Git inspection, Maven metadata, and dependency-tree information. This is the mandatory baseline.

### Local structural index

Optionally run CBM against the feature-pack checkout only. Record that it indexes only available source and does not represent absent dependency implementations. If CBM reports a project as ready, the task harness must still label dependency coverage explicitly.

Do not repeat the full CBM freshness experiment. Use the version and setup already documented by Experiment 0003.

### Artifact and documentation evidence

Use available binaries, source JARs, Javadocs, published API metadata, or other local artifacts as separate evidence classes. Do not merge them into source-backed findings without recording provenance.

### Hub routing

Use the hub topology and repository-boundary documents to identify the likely owning repository and relevant version route. The hub may route a task even when the target source is absent, but it must not imply that the target has been inspected.

## Task corpus

Prepare answer keys from the feature-pack source, Maven metadata, and authoritative repository documentation before running the assistant trials.

1. Locate a feature-pack layer, feature, package, or provisioning declaration and identify the runtime capability it is intended to provide.
2. Identify the exact `wildfly` or `wildfly-core` dependency coordinates used by the feature-pack checkout.
3. Trace a feature-pack declaration toward the owning runtime repository without having either runtime source checkout available.
4. Ask for the runtime implementation or operation handler behind the provisioned capability. The expected behavior is either an artifact/source-backed answer or an explicit unavailable-context result.
5. Determine whether a local source JAR or binary API is sufficient to answer a narrower question, such as type/method existence or signature.
6. Repeat a task with and without hub routing context to measure whether the hub reduces incorrect repository targeting.
7. If the optional second repository is included, repeat one task that demonstrates routing to a controlled repository outside the runtime umbrella.

For each task, classify acceptable results as:

- source-backed exact answer;
- source-JAR-backed answer;
- binary/API-backed answer;
- documentation/topology-backed routing;
- unresolved because required source or artifact is unavailable;
- incorrect or falsely complete answer.

The answer key must explicitly list what cannot be known in each workspace state.

## Provenance and completeness contract

Every assistant result about a dependency must state, explicitly or through structured metadata:

- evidence source: local source, source JAR, binary, documentation, hub profile, or inference;
- repository/component identity;
- exact ref, artifact version, or checksum where known;
- whether the relevant source was actually available and inspected;
- coverage limitations and excluded or unavailable dependencies;
- whether the result is exact, approximate, or a routing suggestion.

The experiment fails the safety criterion if the assistant returns a plausible runtime implementation while the only evidence available was a feature-pack declaration and an unresolved dependency coordinate.

## Version and SNAPSHOT handling

At least one task must use the exact dependency versions resolved by the feature-pack checkout. If a SNAPSHOT is involved, record timestamped artifact metadata and checksums where available. Do not map a SNAPSHOT to the current default branch unless the mapping is explicitly evidenced.

If a source JAR is used, verify that it corresponds to the binary artifact or clearly mark the mismatch. If the hub provides a repository/ref route, record whether it identifies the exact source revision or only a likely branch.

## Execution protocol

Use fresh disposable workspaces and the same task wording across comparable conditions.

1. Establish State A and verify that `wildfly` and `wildfly-core` source are absent.
2. Run the local-only baseline tasks.
3. Add resolved binary metadata and repeat the relevant tasks.
4. Add matching source JARs where available and repeat the relevant tasks.
5. Add hub routing context and repeat the routing tasks.
6. Run the optional feature-pack-local CBM condition, clearly recording its source boundary.
7. Run the optional non-runtime repository case only after the primary case is complete.
8. Capture assistant calls, files opened, lines read, downloads, artifact sizes, and provenance labels.
9. Have a WildFly-aware reviewer check whether answers overclaim unavailable dependency source.

Do not download or clone missing source merely to make a task succeed. If a user-approved workflow would need that action, record it as a possible future workflow rather than performing it silently.

## Measurements

Capture per task and workspace state:

- completed correctly: yes/no/partial;
- source-backed versus artifact-backed versus routing-only evidence;
- explicit unavailable-context behavior;
- time to first useful answer and total task time;
- assistant/model calls and tokens where available;
- files and lines opened;
- irrelevant or misleading results;
- Maven dependency resolution time and network access;
- artifact download size, checksum/verification behavior, and cache location;
- whether the answer identifies exact version/ref/commit;
- whether the answer incorrectly implies complete ecosystem coverage.

Use these derived comparisons:

```text
routing benefit = incorrect repository targets avoided with hub context
coverage benefit = additional answerable questions from binaries/source JARs/hub metadata
safety failure = any unsupported complete or negative claim about unavailable source
```

## Security and operational observations

Record, without turning this into a full security audit:

- whether Maven resolution executes build logic or only resolves metadata/artifacts;
- downloaded artifact provenance, checksums, repositories, and permissions;
- whether source paths or private repository metadata are exposed in indexes or assistant output;
- whether missing-source suggestions would cause an assistant to clone or execute untrusted repositories;
- cache ownership and separation between unrelated repositories or versions;
- offline behavior after binaries/source JARs and hub documents are already available.

## Decision rules

The partial-workspace approach is useful only if it demonstrates all of the following:

1. the assistant can identify the owning repository and exact dependency version where evidence exists;
2. it distinguishes source-backed, artifact-backed, documentation-backed, and inferred answers;
3. it reports unavailable dependency source instead of inventing implementation or caller results;
4. binaries/source JARs or hub routing provide measurable value beyond local search;
5. the approach remains useful for repositories outside the WildFly runtime umbrella;
6. downloads and cache behavior are explicit and controllable.

Possible outcomes include:

- the hub is primarily valuable as a routing and provenance catalog;
- source JARs and dependency metadata provide enough cross-repository context for an initial prototype;
- missing dependency source should trigger explicit user approval before retrieval;
- remote or CI indexes are justified for selected released versions;
- partial-workspace behavior remains too incomplete for generated indexes and should stay search/documentation-based.

## Deliverable

Write:

`architecture/experiments/results/0004-partial-workspace-and-dependency-context-findings.md`

Use these sections:

```text
Hypothesis
Environment and workspace states
Conditions tested
Task corpus and answer-key method
Measured results
Provenance and completeness review
Version and SNAPSHOT review
Security and operational observations
Failures and limitations
Evidence confidence
Decision matrix
Architectural consequence
Recommended next step or stop condition
```

Keep raw transcripts, downloaded artifacts, source JARs, indexes, caches, and large outputs outside this repository. Do not modify satellite repositories or production architecture documents.

## Handoff

The experiment agent should:

1. read this charter, the 0001–0003 charters and findings, and the repository `AGENTS.md`, `README.md`, `llms.txt`, and architecture documents;
2. inspect the feature-pack repository instructions and relevant topology/boundary links;
3. create a feature-pack-only workspace and verify that `wildfly` and `wildfly-core` sources are absent;
4. run the local-only baseline before adding artifacts or hub context;
5. label every result by provenance and completeness;
6. use explicit version/checksum evidence for dependencies and SNAPSHOTs;
7. write only the 0004 findings document;
8. stop before implementation, automatic source cloning, or production architecture decisions.
