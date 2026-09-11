# Experiment 0002 Findings: End-to-End Code Intelligence and Freshness

## Hypothesis

Repository-root CBM may reduce repeated and cross-file navigation cost, but its generated state must be checked against the repository identity and source state. Ordinary search remains a strong low-setup baseline. SCIP is useful only if a task-facing lookup path can be run without treating index generation as the task interface.

## Environment and source identities

All work was performed against disposable `--no-local` clones under `/tmp`; the user checkouts and this repository's production documents were not modified. Raw outputs, CBM databases, and clone state remain outside the repository at `/tmp/wildfly-exp0002-KhbQog/`.

| Repository | Primary ref and commit | Repository root used for indexing |
|---|---|---|
| `wildfly-core` | `ai-index`, `d511a0d1e5c34305f8a6d2d8f917fab1180c65b8` | `/private/tmp/wildfly-exp0002-KhbQog/wildfly-core` |
| `wildfly` | `ai-index`, `39e7ce3cb5a23f354e3825bece6d4ccb15c1ab22` | `/private/tmp/wildfly-exp0002-KhbQog/wildfly` |

The source clones were initially clean. Java was 17.0.18, Maven 3.9.14, macOS aarch64, and CBM was `0.10.8`. The full clone was later switched, in the disposable copy only, to `25.x` at `5178df9f4fdd841ca7e166f3a00317f7a6d49f3a`.

CBM used one cache root shared by the two projects. The resulting database files were approximately 586 MB for core and 734 MB for full. This is generated state, not a proposed storage format.

## Conditions tested

- Ordinary `rg` search and bounded interpretation of source files.
- CBM direct CLI: repository-root indexing, graph search, caller tracing, coverage/status, change detection, and one-shot versus daemon-backed invocation.
- CBM MCP: a stdio initialize handshake, project listing, and a real `search_graph` call against the repository-root index.
- SCIP: the recovery run used the 0001 setup procedure and successfully generated repository-root indexes after Maven reactor/SNAPSHOT preparation. The available snapshot and SQLite lookup paths were then tested; neither provided reliable task-facing locations or references.
- JDTLS was not extended; the charter makes it secondary and 0001 had already failed to demonstrate the minimum cross-file result.

## Task corpus and answer-key method

The unchanged 0001 tasks were rerun as the smallest comparison: find core `ReadResourceHandler` and its registration, and find the full-repository Undertow extension and subsystem/deployment registration. The answer key was checked directly in source: `ReadResourceHandler.java` and `GlobalOperationHandlers.java`; `UndertowExtension.java`, `UndertowRootDefinition`, and `DeploymentDefinition`.

The added checks were: discover and trace `UndertowExtension.getResolver`; identify a changed Undertow source file and likely impacts; and follow the Core-to-Full `Extension` SPI relationship. Generic graph edges were treated as navigation evidence, not proof of WildFly management or cross-repository semantics.

## Measured results

| Condition | Cold/setup observation | Task-facing result |
|---|---:|---|
| Ordinary search | 0.00–0.03 s per bounded search | Complete for both 0001 tasks and the Core-to-Full `Extension` relationship; source interpretation required |
| CBM core root index | 21.7 s; 115,043 nodes / 638,342 edges | Found controller symbols and registrations; broad class search returned 30 matches and required narrowing |
| CBM full root index | 21.4 s; 180,047 nodes / 773,017 edges | Found `UndertowExtension` at lines 24–54 and its graph neighborhood |
| CBM direct query | About 4.0 s per one-shot CLI call, including coordination/daemon startup | Exact class discovery worked; `getResolver` trace returned 43 inbound callers after qualified-name discovery |
| CBM MCP | About 5.6 s for initialize, project listing, and one search call | Reproducible stdio path; returned both projects and the expected Undertow class |
| SCIP | 200.8 s for root core generation including Maven preparation; 580.4 s for root full generation including Maven preparation | Root generation succeeded; snapshot lookup crashed and SQLite exposed no usable chunks, mentions, or ranges |

CBM's full-root graph was materially richer than the module-scoped 0001 graph for the tested repository identity, but root scope also increased generated databases to roughly 1.3 GB combined. A persistent daemon started successfully; the measured CLI query still took about 4.0 s, so the available CLI path did not demonstrate a large warm-query reduction.

The cross-repository task was partially useful: ordinary search verified that `wildfly-core` defines `org.jboss.as.controller.Extension` and that Undertow implements it. CBM could answer each repository-local half, but its separate project graphs did not create a Core-to-Full relationship.

## Cold/warm and amortization analysis

The two cold CBM indexes took about 43 seconds combined. The tested one-shot direct queries cost about 4 seconds each; the MCP process completed its handshake and lookup in about 5.6 seconds. Ordinary search completed the representative lookups in milliseconds and had no setup cost.

For two small tasks, CBM does not amortize against ordinary search. Its potential break-even depends on repeated caller/impact tasks that ordinary search cannot answer as directly; this run did not establish a numeric break-even because no comparable model-token or full repeated-task execution series was available. The observed graph query advantage is therefore a capability advantage, not yet an end-to-end cost win.

## Correctness and freshness review

CBM found the expected Undertow class and reported 43 inbound callers for `UndertowExtension.getResolver` after exact-name discovery. This is useful structural evidence, but the result includes file-level and registration-related entries and still requires source review. The Core-to-Full SPI edge was not inferred automatically.

After adding an uncommitted comment to `UndertowExtension.java`, `detect_changes` correctly reported the changed path and five seed symbols, but reported zero impacted symbols. This demonstrates dirty-file detection, not complete impact analysis. Coverage for the clean full-root index reported `metadata_match` for the Undertow file, while also reporting ten parse-partial files and 157 deliberately ignored files for the repository.

After switching the disposable full clone from `ai-index` to `25.x`, CBM continued to report the old 180,047-node index as `ready` with the original root path and returned the old Undertow symbols. It did not reject, rebuild, or visibly mark the index stale. This is a concrete branch/ref identity hazard. The earlier 0001 module-root mismatch was avoided by indexing the Git repository root, but repository-root scope alone does not solve ref freshness.

## Security and operational observations

CBM read the source tree and built a local daemon/database; no repository build was required. The binary was already available in the experiment environment and ran from `/tmp`. macOS privacy approval was required for the daemon/IPC invocation, which is a user-facing installation friction. One cache root had to be shared by both CBM projects.

The full-root index intentionally excluded `.git`, build/distribution directories, binary assets, and other ignored paths. Parse-partial files were indexed with warnings. These exclusions and warnings must be surfaced to an assistant before it makes negative or exhaustive claims.

SCIP recovery required Coursier and Maven downloads, then succeeded after the required Maven reactor/SNAPSHOT artifacts were installed. Root generation repeated Maven reactor work after preparation. The official snapshot formatter crashed on these indexes, while SQLite conversion succeeded but exposed only incomplete symbol/document metadata. No source checkout or build file was changed.

## Failures and limitations

- SCIP repository-root generation succeeded, but the available task-facing lookup paths failed: the snapshot formatter crashed, and SQLite conversion exposed no chunks, mentions, or definition-enclosing ranges.
- The MCP test covered a reproducible stdio handshake and one real lookup, not a full MCP task matrix.
- The CBM CLI used deprecated raw-JSON argument syntax and one-shot coordination added several seconds to each invocation.
- The warm daemon started, but the CLI measurement did not show a meaningful latency reduction.
- No generated-source/dependency-input transition or offline-after-setup transition was run.
- The branch-switch test observed stale reuse after the clone changed refs; it did not test CBM's behavior after a forced explicit re-index on the new ref.
- Model token counts and reviewer-independent semantic precision were not available; caller and impact results remain approximate until checked against source.

## Evidence confidence

High for repository-root identity, CBM indexing counts, direct/MCP availability, dirty-file detection, stale branch reuse, and SCIP root generation cost. Moderate for CBM caller usefulness because the 43 callers were not exhaustively reviewed. High that ordinary search is the lowest-cost baseline for the two small tasks. Low for SCIP task usefulness: generation succeeded, but the available lookup clients did not expose reliable locations or references. This does not disprove SCIP’s semantic model; it demonstrates that the tested toolchain is not currently task-usable.

## Decision matrix

| Approach | Task usefulness | Operational viability | Finding for this experiment |
|---|---|---|---|
| Ordinary search | Complete for small navigation and cross-repository SPI verification | Immediate, dirty/ref-aware through Git and source inspection | Retain as mandatory fallback and first path for small tasks |
| CBM direct | Strong structural discovery and caller tracing after qualified-name discovery | Root indexing worked, but databases were large and ref freshness was unsafe | Worth an optional local experiment/backend with explicit identity validation |
| CBM MCP | Same graph is portable through a reproducible stdio path | Daemon/IPC and macOS privacy approval add setup friction | Useful access path for further testing; not a production decision |
| SCIP | Repository-root generation succeeded in this recovery run; exact symbol/document lookup worked, but reference/occurrence lookup remained unresolved | Requires a full Maven reactor/SNAPSHOT preparation and repeats substantial Maven work during indexing; the available snapshot and SQLite inspection paths were incomplete | Keep generation as viable evidence, but do not treat SCIP as task-useful until a lookup client returns locations and references reliably |

## Architectural consequence

These findings inform later architecture questions without selecting an implementation:

- **Generation location:** local generation is practical for CBM, but native daemon permissions and multi-gigabyte root caches matter; SCIP would need controlled build/tool environments.
- **Storage:** CBM state is disposable and large; do not commit or publish it based on this run.
- **Repository ownership and version identity:** project names and root paths are not sufficient; indexes must carry repository identity, commit/ref, and source digest, and must reject branch/ref reuse.
- **Freshness:** root indexing fixes the 0001 path mismatch but did not prevent stale reuse after a branch switch. Dirty detection and explicit stale warnings are required.
- **Assistant interface:** direct CBM and MCP both expose useful structural lookup; MCP adds portability but also daemon/IPC setup.
- **Offline behavior:** CBM queried from an existing local cache; offline update/rebuild behavior was not established. SCIP installation and Maven preparation were network-dependent in this environment.
- **Security:** binary provenance, daemon permissions, cache permissions, and build execution for SCIP remain operational concerns.
- **WildFly-specific enrichment:** generic Java edges did not represent the Core-to-Full SPI relationship, management registration semantics, or provisioning/deployment relationships without source-guided interpretation.

## SCIP recovery measurements

This recovery run used fresh disposable clones under `/tmp/wildfly-exp0002-scip-pJfOfa/`, indexed from each Git repository root, and used the proven `scip-java 0.13.0` launcher from Experiment 0001. The clone identities were unchanged from the earlier 0002 run: `wildfly-core` `ai-index` at `d511a0d1e5c34305f8a6d2d8f917fab1180c65b8` and `wildfly` `ai-index` at `39e7ce3cb5a23f354e3825bece6d4ccb15c1ab22`. No source or implementation files were modified.

### Setup and generation

The Maven reactor/SNAPSHOT preparation was measured explicitly from each repository root:

```text
mvn -B clean install -DskipTests
```

| Repository | Maven preparation | Root SCIP generation | Artifact |
|---|---:|---:|---:|
| `wildfly-core` | 68.249 s, exit 0 | 132.572 s, exit 0 | 184,801,762 bytes |
| `wildfly` | 229.160 s, exit 0 | 351.240 s, exit 0 | 131,475,873 bytes |

The root generation command was:

```text
coursier launch org.scip-code:scip-java:0.13.0 \
  --main org.scip_code.scip_java.ScipJava -- index \
  --build-tool=maven --no-cleanup --output=/tmp/.../<repository>.scip
```

The indexer invoked Maven compilation/reactor work again after the preparation step. This repeated work is part of the measured generation cost. All logs and `.scip` files remained outside the repository. The successful exit codes and large artifacts establish repository-root generation, not yet task-level usefulness.

### Read-only lookup attempts

The built-in `scip-java snapshot` command was first attempted, but it requires a separate `scip` executable. Installing the official disposable SCIP CLI (`v0.7.1`) into `/tmp` resolved that setup dependency. Both snapshot runs then failed with a runtime panic in SCIP's snapshot formatter (`index out of range [0] with length 0`) while processing an empty SCIP range. This is a concrete lookup-client failure, not a generation failure.

The official CLI's experimental SQLite conversion was then used as a real read-only lookup path:

```text
scip expt-convert --output /tmp/.../<repository>.db /tmp/.../<repository>.scip
sqlite3 -readonly <repository>.db
```

Conversion succeeded in 0.859 s for `wildfly-core` (46,723,072-byte database) and 0.658 s for `wildfly` (35,139,584-byte database). Read-only SQL located the expected indexed symbols and source documents: `ReadResourceHandler` and `GlobalOperationHandlers` in `controller/src/main/java/.../operations/global/`, and `UndertowExtension` in `undertow/src/main/java/org/wildfly/extension/undertow/UndertowExtension.java`. However, both converted databases contained documents and global symbols but zero chunks, mentions, and definition-enclosing ranges. Consequently the lookup could not return reliable source locations, callers, or registration relationships. The representative tasks therefore remain unresolved from the SCIP output.

### Recovery decision

SCIP recovery is now a generation success with substantial build/tool setup cost. Task usefulness remains unresolved: the available official read-only clients either crashed on these indexes or exposed only symbol/document metadata without occurrences or locations. Retain SCIP as an experimental generated artifact, but do not promote it to an assistant-facing navigation backend or make production architecture changes on this evidence. The raw logs, indexes, SQLite databases, and disposable CLI are under `/tmp/wildfly-exp0002-scip-pJfOfa/`.

## Recommended next experiment or stop condition

Do not make an implementation or production architecture decision from this run. The next bounded experiment should add explicit ref/source-digest validation around CBM, force a branch-change refresh, and run a larger repeated task set with independent correctness review. Do not build a custom SCIP lookup client solely to continue this comparison; revisit SCIP only if an existing, reliable consumer becomes available. Stop the generated-index investigation if CBM does not demonstrate repeatable total-cost savings over ordinary search on repeated caller/impact tasks while maintaining safe freshness behavior.
