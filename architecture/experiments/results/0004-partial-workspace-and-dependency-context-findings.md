# Experiment 0004: Partial Workspaces and Dependency Context — Findings

## Hypothesis

A feature-pack-only workspace can answer registry, layer, version, and routing questions usefully, but it cannot answer runtime implementation questions without overstating the evidence. Resolved Maven artifacts and matching source JARs should extend the answerable set when their provenance is explicit. Hub documentation should improve repository routing, but should not be treated as inspected runtime source.

The hypothesis was tested as an evidence-boundary question, not as confirmation of Experiment 0001.

## Environment and workspace states

The disposable experiment root was `/tmp/wildfly-exp0004-bPU1dm/`. The repository checkout used for the primary case was:

| Item | Value |
|---|---|
| Repository | `wildfly-galleon-feature-packs` |
| Checkout | `/tmp/wildfly-exp0004-bPU1dm/feature-packs` |
| Branch/ref | `ai-index` |
| Commit | `ba872664f8f97f79c78a47e28afa4b4f3a7e5030` |
| Working tree | clean at experiment start |
| Runtime source | no `wildfly` or `wildfly-core` source checkout |
| Maven cache | `/tmp/wildfly-exp0004-bPU1dm/m2` |
| Raw evidence | `/tmp/wildfly-exp0004-bPU1dm/evidence/` |

The checkout contains the registry's own small Maven tool under `maven/`; that is feature-pack-registry source, not a runtime checkout. No satellite repository was modified or cloned. Existing user checkouts outside the disposable experiment workspace were not used as runtime source evidence.

State A contained only the feature-pack checkout and its repository instructions. The local registry identifies `41.0.0.Final` as latest and declares these relevant coordinates in its provisioning/catalog files:

- `org.wildfly:wildfly-galleon-pack:41.0.0.Final`
- `org.wildfly.cloud:wildfly-cloud-galleon-pack:9.2.3.Final`
- `org.wildfly:wildfly-datasources-galleon-pack:11.4.0.Final`

State B added Maven-resolved feature-pack ZIPs and their POM/parent metadata to the disposable cache. State C added a WildFly Core binary and matching source JAR for the narrower API question below. State D added the hub `llms.txt`, boundary document, topology document, and the feature-pack repository profile as routing context; it did not add runtime source.

## Conditions tested

The mandatory comparison used ordinary bounded search/read/Git inspection first, followed by Maven artifact inspection, source-JAR inspection, and hub routing inspection. This was not a repeat of the CBM freshness experiment. CBM was not run for this experiment because it is optional in the charter and Experiment 0003 already established that a feature-pack-only index cannot supply absent runtime implementation source. No MCP or remote source index was used.

Maven resolution used an isolated local repository and `dependency:get` with `-Dtransitive=false` for the three feature-pack ZIPs. The first narrow runtime lookup failed because the sandbox could not resolve `repository.jboss.org`; the same exact lookup was then retried with explicit network approval. No build of a satellite repository was run.

## Task corpus and answer-key method

Answer keys were prepared from the feature-pack checkout, downloaded POMs/ZIPs, the controller source JAR, and hub documents before comparison.

| Task | Answer-key result and evidence boundary |
|---|---|
| Locate a declaration and identify capability | `cloud-default-config` in the cloud ZIP depends on `web-server`, `datasources`, `management`, CDI, JAX-RS, messaging, clustering, web services, and other layers; it also declares an H2 example datasource. `postgresql-datasource` in the datasource ZIP depends on `datasources` and `postgresql-driver` and declares a PostgreSQL datasource. This is feature-pack artifact evidence, not implementation source. |
| Identify exact dependency coordinates | The registry source gives the exact coordinates and versions above. The resolved artifacts are release versions, not SNAPSHOTs. |
| Route toward the owner without runtime source | Local feature-pack instructions route core/kernel/controller/CLI work to `wildfly-core`, full/Jakarta EE/clustering/Elytron work to `wildfly`, and feature-pack content to the named extension repository. The hub adds the boundary and topology explanation. This is documentation/topology-backed routing. |
| Ask for the runtime implementation or operation handler | The feature-pack-only states cannot establish the implementation, caller graph, or operation handler behind a provisioned layer. The correct result is explicit unavailable context. A plausible implementation answer from only the layer XML is a safety failure. |
| Answer a narrower type/API question | `org.wildfly.core:wildfly-controller:33.0.0.Final` binary and source JAR were resolved. The source JAR contains `OperationStepHandler`, `OperationDefinition`, `ResourceDefinition`, and related types. This establishes type existence at that artifact version; it does not establish which runtime subsystem implements the selected feature-pack layer. |
| Repeat routing with and without hub context | Without hub context, the local profile supplies direct route hints but little ecosystem relationship detail. With hub context, the route is more precise: the registry owns compatibility declarations, `wildfly` owns distribution feature-pack definitions, extension packs own independently released optional content, and `wildfly-core` owns controller/resource/operation SPI. |
| Optional outside-runtime repository task | Not run. The primary partial-workspace case was completed without introducing another repository. |

## Measured results

The experiment measured artifact resolution and evidence coverage, not model latency or token usage. No assistant transcript runner was available, so no claim is made about model-call timing.

### State A: local source

Ordinary search completed the registry/version/routing tasks. It located the provisioning XML, catalog JSON, `versions.yaml`, local `AGENTS.md`, and `llms.txt`. It could not answer the runtime implementation task. The local source was sufficient for exact registry coordinates but not for runtime class ownership, handler implementation, or callers.

### State B: resolved feature-pack binaries

All three exact release artifacts resolved successfully into the isolated cache. The ZIPs are feature-pack content, not runtime classpath JARs:

| Coordinate | Repository provenance | Size | SHA-256 | SHA-1 |
|---|---|---:|---|---|
| `org.wildfly:wildfly-galleon-pack:41.0.0.Final:zip` | Maven Central (`repo.maven.apache.org`) | 527,576 bytes | `4b425b0ff305d663d47a5ae6421b8ad1dc535a00e54e2748946b9a5360bc1fdf` | `b376ed8c12e2414c1c9dfded09d40ebc336c132f` |
| `org.wildfly.cloud:wildfly-cloud-galleon-pack:9.2.3.Final:zip` | JBoss public repository group (`repository.jboss.org`) | 115,834 bytes | `a3bf1a3ee2b274a479fb17f6be7cbbc64a92e31ba38b58254676f868e07315c8` | `2699c0ed6d45c6daded172a806205d02ba457ff3` |
| `org.wildfly:wildfly-datasources-galleon-pack:11.4.0.Final:zip` | JBoss public repository group (`repository.jboss.org`) | 47,393 bytes | `f19c09790f2b71f8ba8477421acfc8057f2aef1ea6c0b09f8b00cb6291a34be5` | `d5154006673b65d1d0181c3e957c11e4b17de802` |

The first Maven resolution of the three feature-pack artifacts took approximately 20.7 seconds for the command and fetched POM/parent/BOM metadata in addition to the ZIPs. The feature-pack ZIPs supplied useful layer, package, module, and configuration declarations. They supplied no Java implementation source for the runtime task.

### State C: binary and matching source JAR

The exact coordinate `org.wildfly.core:wildfly-controller:33.0.0.Final` was evidenced by the resolved WildFly Core parent/dependency metadata and was fetched as both binary and sources:

| Artifact | Size | SHA-256 | SHA-1 |
|---|---:|---|---|
| `wildfly-controller-33.0.0.Final.jar` | 2,453,484 bytes | `37ec6d195142cdfd80f015ed4bcab7a61d985eb5896b85ed03a47907a788cb84` | `d2fd01d6836034e0518d6e1b849286f1faeb06cf` |
| `wildfly-controller-33.0.0.Final-sources.jar` | 1,199,326 bytes | `be13a3c8d975a9e62b75c50e0ab13f4bc051f8b045f356cbbec0e796a0d3f26f` | `07920b2ead808e8fde54f193aaf41e7ea58a549e` |

The binary/source pair is version-matched, and the source JAR contains the controller SPI types named by the hub boundary document. It enabled a source-JAR-backed answer to type existence, but not a source-backed answer about the implementation of `cloud-default-config` or `postgresql-datasource`.

### State D: hub routing

Hub context materially improved routing and reduced ambiguity. The hub states that the registry contains versioned compatibility metadata, that `wildfly` owns distribution feature packs and full Jakarta EE capabilities, that extension packs own independent optional content, and that `wildfly-core` owns management/resource/operation SPI. It also states that Glow consumes registry metadata and does not need runtime source for metadata routing.

The hub did not identify an exact runtime source revision for the feature-pack declarations. Its repository links point to `ai-index` documentation routes, not a source commit proven to implement the resolved release artifacts. Therefore the hub result is documentation/topology-backed routing, not source-backed inspection.

## Provenance and completeness review

Every positive result can be labeled as follows:

- Registry coordinates, versions, and compatibility declarations: local-source-backed at feature-pack commit `ba872664f8f97f79c78a47e28afa4b4f3a7e5030`.
- Layer dependencies and feature parameters: binary/feature-pack-artifact-backed, with the ZIP coordinate and checksum recorded above.
- Controller SPI type existence: matching source-JAR-backed and binary-backed at `33.0.0.Final`, with checksums recorded above.
- Repository ownership and cross-repository route: documentation/topology-backed by the hub and feature-pack repository profiles.
- Runtime implementation, operation handler, caller graph, and exact implementation commit for the provisioned layers: unavailable in this workspace.

The correct incomplete answer is therefore: “The declaration provisions or configures this capability and routes to the relevant repository; the runtime implementation is not inspected and cannot be asserted from this workspace.” Any answer naming a concrete subsystem handler or caller without source-JAR or runtime-source evidence would be incorrectly complete.

## Version and SNAPSHOT review

The tested declarations use `41.0.0.Final`, `9.2.3.Final`, and `11.4.0.Final`; all are release versions. The registry also lists unavailable future/working versions including `41.0.1.Final-SNAPSHOT`, `42.0.0.Beta1-SNAPSHOT`, and older `*.Final-SNAPSHOT` entries, but none was silently substituted or resolved for this experiment. The controller API artifact used for the narrow source-JAR task is explicitly `33.0.0.Final`; it is not claimed to be the implementation revision of the `41.0.0.Final` feature pack.

No SNAPSHOT timestamp, snapshot metadata, or checksum was needed for the tested path. A future SNAPSHOT evaluation must record timestamped metadata and must not map it to a default branch or to the nearest Final release.

## Security and operational observations

Maven `dependency:get` was used with an isolated cache and `-Dtransitive=false`; the operation resolved metadata and downloaded declared artifacts rather than executing a satellite build. Artifact URLs, sizes, local cache paths, and checksums were captured outside the repository. The first network attempt failed under sandbox DNS restrictions; the retry required explicit network approval. This makes network access visible and controllable.

No source clone, remote index, generated index, binary, cache, transcript, or downloaded artifact was written into the hub repository. The feature-pack checkout's `origin` is a local source path inherited from the supplied checkout; it was not used to fetch or mutate a satellite. The isolated cache prevents this experiment's artifacts from being confused with another repository's cache.

The main operational risk is provenance collapse: a feature-pack layer declaration can look sufficiently specific to tempt an assistant to invent the runtime implementation. The evidence contract must keep feature-pack content, runtime binary/API evidence, source-JAR evidence, and hub routing separate.

## Failures and limitations

- No model-call harness was available, so time-to-first-answer, total assistant task time, calls, and tokens were not measured.
- CBM was not run; this experiment records the partial-workspace evidence boundary established by Experiment 0003 rather than repeating its freshness protocol.
- The resolved feature-pack ZIPs are metadata/content artifacts, not the runtime implementation source.
- The controller source JAR answers a deliberately narrow API/type question and does not prove the implementation or operation handler for a selected feature-pack layer.
- The hub routes repositories and relationships but does not provide an exact source revision for the release artifacts.
- The optional non-runtime repository case was not included.
- Maven repository availability is environment-dependent; the first exact runtime lookup failed before the approved retry. An offline run would work only for artifacts already present in the isolated cache.

## Evidence confidence

High confidence: feature-pack checkout identity, clean state, exact registry declarations, release versions, ZIP sizes/checksums, ZIP layer contents, controller binary/source-JAR identity and checksums, and absence of runtime source in the disposable checkout.

Medium confidence: repository ownership and cross-repository routing, because these are supported by the current hub and repository profiles but are documentation evidence rather than implementation inspection.

Low or unavailable confidence: any claim about the concrete runtime implementation, operation handler, caller graph, or exact source revision behind a feature-pack capability. Those claims require additional runtime source or a clearly versioned artifact/source mapping and were intentionally not inferred.

## Decision matrix

| Question | State A local source | State B feature-pack binaries | State C matching source JAR | State D hub context |
|---|---|---|---|---|
| Exact registry/version answer | Yes, source-backed | Yes | Yes | Yes |
| Layer/package/configuration declaration | Partial local declarations | Yes, artifact-backed | Yes | Yes |
| Runtime type/API existence | No | No | Yes, for controller 33.0.0.Final | Yes only as documented route |
| Runtime implementation/operation handler | No; unavailable | No; unavailable | No for the selected feature-pack capability | No; route only |
| Owning repository route | Partial local profile | Partial | Partial | Yes, documentation/topology-backed |
| Exact runtime source revision | No | No | No; source JAR has artifact version only | No |
| False-completeness risk | High unless labeled | High unless labeled | Medium and scope-limited | Medium if routing is mistaken for inspection |

## Architectural consequence

The experiment supports a provenance-aware partial-workspace contract, not an implementation or architecture change. A context system should expose evidence classes and completeness limits as first-class result metadata: local source, feature-pack artifact, binary/API, source JAR, hub routing, inference, and unavailable. A repository being routable must not imply that it was inspected.

The result also supports treating the hub as an ecosystem routing/provenance catalog for repositories outside the runtime source trees. It does not justify implementation of an indexer, remote service, automatic source retrieval, or a production architecture decision in this experiment.

## Recommended next step or stop condition

Stop this experiment before implementation and architectural decisions, as required by the charter. The next evaluation, if authorized, should use a fixed task runner with the same prompts across States A–D and record model timing/tokens plus explicit provenance labels. Any retrieval of missing runtime source should require a separate user-approved workflow and should preserve the exact release/ref mapping; it must not be introduced as an automatic completion step.
