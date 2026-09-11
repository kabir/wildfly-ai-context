# Experiment 0001 Findings: Smallest Runnable Source-Navigation Subset

## Hypothesis

For two representative WildFly navigation tasks, ordinary `rg` plus bounded file reads can locate the relevant implementation and registration relationships with less setup than generated indexes. Generated semantic or graph indexes are useful only if their setup, freshness, and correctness costs are acceptable.

## Environment

The experiment used temporary clones; no satellite checkout was modified.

| Repository | Ref | Commit | Module |
|---|---|---|---|
| `wildfly-core` | `ai-index` | `d511a0d1e5c34305f8a6d2d8f917fab1180c65b8` | `controller` |
| `wildfly` | `ai-index` | `39e7ce3cb5a23f354e3825bece6d4ccb15c1ab22` | `undertow` |

Java was 17.0.18 initially and Java 21.0.10 from SDKMAN for JDTLS. Maven was 3.9.14; the platform was macOS, aarch64. Both source repositories were subsequently built with `mvn clean install`, enabling the successful SCIP rerun.

`wildfly-core` supplied `AGENTS.md` and `llms.txt`. The adjacent `wildfly` checkout had no `AGENTS.md`, `CLAUDE.md`, or local `llms.txt`; this was recorded as setup evidence.

## Conditions tested

Two task cards were run against one representative module from each repository:

1. In `wildfly-core/controller`, locate the `read-resource` management-operation implementation and its registration/call sites. Expected concepts: `ReadResourceHandler` and `GlobalOperationHandlers`.
2. In `wildfly/undertow`, locate the subsystem extension and its subsystem/deployment model registrations. Expected concept: `UndertowExtension`.

Conditions were ordinary search, disposable BSD ctags lookup, SCIP, JDTLS, and `codebase-memory-mcp`. All generated indexes, caches, workspaces, and transcripts stayed outside the repository.

## Setup procedures and findings

### Ordinary search

No installation was required. The temporary checkout was queried directly with `rg`, directory listing, and bounded `sed` reads. This was the only condition with no generated state, dependency resolution, daemon, runtime mismatch, or cache coordination.

### BSD ctags / Bob-style lookup

The available `/usr/bin/ctags` was Apple/BSD ctags. Its recursive form failed:

```text
ctags -R -f tags src/main/java
ctags: illegal option -- R
```

A disposable file-list equivalent worked:

```bash
find src/main/java -type f -name '*.java' -print0 \
  | xargs -0 ctags -f /tmp/module.tags
```

This produced declaration tags only. Universal Ctags or the actual Bob Tags setup should be tested separately before treating this as a Bob implementation result. Caller lookup remained textual `rg`, and the pre-edit tags did not see a newly added dirty-file symbol.

### SCIP

The successful setup required all of the following:

1. Java 17 or newer and Maven.
2. Coursier, installed here with Homebrew:

   ```bash
   brew install coursier/formulas/coursier
   ```

3. The explicit SCIP main class. The artifact has no default main class, so this form is required:

   ```bash
   coursier launch org.scip-code:scip-java:0.13.0 \
     --main org.scip_code.scip_java.ScipJava -- --help
   ```

4. Completed Maven reactor installation. For WildFly Core, indexing `controller` standalone initially failed because Maven could not resolve the SNAPSHOT parent, test BOM, and sibling artifacts. The source checkout had to be built with `mvn clean install`; installing the temporary core parent and test BOM was also needed while diagnosing the missing reactor state.
5. A module-local invocation with output outside the checkout:

   ```bash
   coursier launch org.scip-code:scip-java:0.13.0 \
     --main org.scip_code.scip_java.ScipJava -- index \
     --build-tool=maven --no-cleanup \
     --output=/tmp/scip/controller.scip
   ```

The first corrected attempt still failed because `--cleanup=false` is invalid syntax; the CLI requires the boolean flag `--no-cleanup`. Once the Maven reactor artifacts were installed, both modules indexed successfully. The main setup difficulty is that SCIP is not just a source parser: it depends on a resolvable, buildable Maven state and may compile the module.

### JDTLS

JDTLS 1.60.0 required Java 21. Java 21.0.10 was already installed through SDKMAN and was passed explicitly:

```bash
jdtls \
  --java-executable /Users/kabir/.sdkman/candidates/java/21.0.10-tem/bin/java \
  --jvm-arg=-Duser.home=/tmp/jdt-home \
  --jvm-arg=-Dosgi.configuration.area=/tmp/jdt-config \
  -data /tmp/jdt-workspace
```

Without the temporary `user.home` and Eclipse configuration paths, startup attempted to write under `~/.eclipse`, which was unavailable in the sandbox. A disposable LSP client was required to send `initialize`, open a Java file, and issue definition/reference requests. JDTLS also attempted a Gradle metadata download during import. Same-file definitions worked, but the minimal module-scoped workspace did not produce cross-file reference results.

### `codebase-memory-mcp`

The binary-only setup avoided automatic agent configuration. The official Apple Silicon binary was downloaded and checksum-verified into `/tmp` with configuration changes disabled:

```bash
curl -fsSL https://raw.githubusercontent.com/DeusData/codebase-memory-mcp/main/install.sh \
  | bash -s -- --dir=/tmp/codebase-memory-mcp/bin --skip-config
```

Indexing used one shared external cache and disabled repository persistence:

```bash
CBM_CACHE_DIR=/tmp/codebase-memory-cache \
  /tmp/codebase-memory-mcp/bin/codebase-memory-mcp cli --progress \
  index_repository --repo-path=/tmp/wildfly-core/controller \
  --name=experiment-core --mode=full --persistence=false
```

The one-shot CLI still starts a temporary daemon and requires local IPC. The sandbox blocked that listener, so the successful run required externally approved execution. Separate cache roots also conflicted with the account daemon; both projects had to use one shared `CBM_CACHE_DIR`. Indexing a module directory rather than the parent Git root caused dirty-file detection to find the changed file but produce zero seed symbols, so repository-root indexing is required for a valid freshness test.

## Measured results

| Condition | Core task | Undertow task | Generation/import | Generated state | Result |
|---|---:|---:|---:|---:|---|
| Ordinary `rg` + bounded reads | 16.8 ms | 11.1 ms | none | none | Complete for both tasks |
| BSD ctags file-list lookup | Declaration tags in 89.0 ms | Declaration tags in 29.2 ms | 118.2 ms total | 6,438 + 902 bytes | Declarations worked; callers still required search |
| SCIP 0.13.0 | 14.7 s; 797 shards | 7.7 s; 212 shards | Maven `BUILD SUCCESS`, tests skipped | 43,559,863 + 7,778,566 bytes | Index generation succeeded for both modules |
| JDTLS 1.60.0 | Same-file definition in 57.9 s cold startup | Same-file definition in 31.0 s cold startup | Java 21 and Maven workspace startup | Temporary Eclipse workspace | Cross-file definition/reference queries remained empty |
| `codebase-memory-mcp` 0.10.8 full mode | 6.9 s; 20,890 nodes / 123,295 edges | 5.8 s; 5,871 nodes / 17,824 edges | Temporary daemon and IPC | About 100 MB shared cache | Exact declarations and a 36-caller trace succeeded |

The baseline found the core registration in `GlobalOperationHandlers.java`, the implementation in `ReadResourceHandler.java`, and the Undertow extension’s `registerSubsystemModel` and `registerDeploymentModel` calls. Ctags supplied declarations but no semantic callers.

CBM structural search found `ReadResourceHandler` at lines 73–657 and `UndertowExtension` at lines 24–54. Its inbound trace for `UndertowExtension.getResolver` returned 36 callers across Undertow resource-definition classes. Its architecture query identified the controller’s management-registration cluster without opening source files. A guessed `ReadResourceHandler.execute` qualified name failed, so exact qualified names must be discovered before tracing.

Raw evidence remains outside the repository:

- `/tmp/wildfly-source-nav-VxAuH0/` — initial baseline and ctags run;
- `/tmp/wildfly-source-nav-rerun-KA1KwU/` — SCIP, JDTLS, and CBM transcripts, indexes, workspaces, and queries;
- `/tmp/codebase-memory-mcp-exp-BHnRHR/` — checksum-verified CBM binary and cache.

## Setup difficulty and operational issues

| Tool/condition | Setup difficulty | Issues encountered |
|---|---|---|
| Ordinary search | None | Requires agent knowledge to interpret relationships; no persistent state |
| BSD ctags / Bob-style equivalent | Low for a single module; portability concern | macOS ctags rejected recursive `-R`; a file-list fallback was required. It provides declarations, not semantic callers or freshness handling |
| SCIP | High | Initially absent; required Coursier and Maven downloads. Controller indexing required installed core parent, test BOM, and sibling SNAPSHOT artifacts. Build coupling is significant |
| JDTLS | High | Required Java 21, a disposable LSP client, writable Eclipse configuration, and temporary user-home redirection. It attempted Gradle metadata access and had 31–58 s cold imports |
| `codebase-memory-mcp` | Medium | A 295 MB verified binary was downloaded. Even one-shot CLI mode starts a daemon and needs IPC; sandbox execution failed until externally approved. Separate cache roots conflicted with the account daemon, so one shared cache was required |

## Correctness review

Both baseline tasks were complete against their expected files and concepts. CBM’s Undertow caller trace was substantially richer than ctags, but expert review is still required for semantic correctness. Caller results from ctags were treated as approximate textual matches, not resolved call graphs.

JDTLS same-file definitions were correct. Cross-file use-site definition and reference queries returned empty results both immediately and after a 30-second warm-up in the module-scoped harness. This does not establish that JDTLS cannot resolve them; it establishes that this workspace/import setup did not demonstrate them.

CBM dirty-file detection found the changed Undertow file but reported zero seed and impacted symbols. The index root was `wildfly/undertow`, while Git’s repository root was its parent `wildfly`; this path mismatch is a concrete freshness integration hazard. A repository-root index is required before judging CBM freshness support.

## Failures and limitations

- The first SCIP invocation used an invalid boolean syntax and failed before compilation; the corrected invocation succeeded after the completed Maven builds supplied the reactor artifacts.
- JDTLS’s lightweight module-scoped client did not demonstrate cross-file references, call hierarchy, or type hierarchy.
- CBM’s graph does not automatically represent WildFly-specific management registration, deployment-processor ordering, MSC service dependencies, Galleon layers, or cross-repository SPI relationships. Its Quarkus evaluation similarly warns that build-item relationships are not ordinary `CALLS` edges and that some Java edges need source verification.
- No warm-trial matrix, branch-switch matrix, deletion/rename checks, generated-source checks, full-reactor index, or cross-repository task was run.
- The small task corpus and one primary trial per condition make these directional results, not a final tool selection.

## Evidence confidence

Moderate for baseline, ctags, SCIP generation, and CBM indexing observations. Low-to-moderate for JDTLS semantic-query conclusions because the harness and workspace import were deliberately minimal. Low for claims about WildFly-specific semantic correctness until a larger task corpus is reviewed by a WildFly-aware expert.

## Decision matrix

| Approach | Task usefulness | Operational viability | Current decision |
|---|---|---|---|
| Ordinary search | Sufficient for both representative tasks | Immediate, no generated state, dirty-file aware | Use as baseline and first prototype |
| Bob-style materialized declarations | Fast declarations; no demonstrated task saving | Simple but ctags portability and freshness need attention | Follow up with Universal Ctags/Bob-compatible setup |
| SCIP | High-quality index generation completed; task lookup not yet measured | Viable after Maven reactor SNAPSHOT artifacts are installed; large artifacts and build coupling | Continue task-level lookup measurement |
| JDTLS | Same-file definitions worked; cross-file semantic queries unproven | Java 21, workspace import, LSP client, and cold-start costs are substantial | Do not select yet; rerun at repository-root scope |
| `codebase-memory-mcp` | Strong structural lookup and a 36-caller trace | Fast indexing, but requires daemon IPC, cache coordination, and correct Git root | Add to the next controlled comparison |

## Architectural consequence

The experiment does not justify a production artifact, assistant interface, remote service, or final storage model. Direct search has no generation or cache cost. SCIP and CBM provide useful generated state but introduce build, cache, daemon, and freshness requirements. JDTLS provides interactive language-server semantics but has notable workspace startup and integration costs.

The observations inform later questions as follows:

- **Generation location:** local or CI generation must account for tool/runtime availability and Maven reactor state.
- **Storage:** generated artifacts should remain disposable until task-level savings and freshness are demonstrated.
- **Versioning:** retained artifacts must identify repository ref, tool version, parser/index schema, and build inputs.
- **Freshness:** dirty edits and Git-root mismatches can produce stale or empty impact results without explicit validation.
- **Assistant interface:** CBM’s MCP graph interface is promising; SCIP and JDTLS still need a task-facing lookup layer.
- **WildFly-specific enrichment:** generic indexes must be supplemented or reviewed for management, deployment, provisioning, service, and cross-repository relationships.

## Follow-up experiment

Run the same tasks plus one cross-repository task using repository-root indexes for SCIP, JDTLS, and CBM. Add warm trials, exact qualified-name discovery, dirty-file and branch-switch checks, and expert review of caller/impact results. Test whether WildFly-specific enrichment is needed for management registration, deployment processors, MSC services, Galleon layers, and Core-to-Full SPI edges.
