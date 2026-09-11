# Experiment 0003 Findings: CBM Freshness and Repeated-Task Value

## Hypothesis

CBM can provide useful caller and impact-navigation evidence, but it is safe for assistant use only if repository identity and source-state changes are detected before results are returned. Any repeated-task savings must include indexing, refresh, daemon, cache, and interpretation costs.

## Environment and source identities

The run used fresh `--no-local` clones and a new CBM cache under `/tmp/wildfly-exp0003-kznGx4/`. No satellite checkout or production document was modified.

| Repository | Initial ref and commit | Canonical repository root | CBM project |
|---|---|---|---|
| `wildfly-core` | `ai-index`, `d511a0d1e5c34305f8a6d2d8f917fab1180c65b8` | `/private/tmp/wildfly-exp0003-kznGx4/wildfly-core` | `private-tmp-wildfly-exp0003-kznGx4-wildfly-core` |
| `wildfly` | `ai-index`, `39e7ce3cb5a23f354e3825bece6d4ccb15c1ab22` | `/private/tmp/wildfly-exp0003-kznGx4/wildfly` | `private-tmp-wildfly-exp0003-kznGx4-wildfly` |

The initial clones were clean. Java was 17.0.18, Maven 3.9.14, macOS aarch64, and CBM was 0.10.8. The core clone was later switched to `2.x`, commit `21d2c0968e9f2a47e93503e3cffdb992a86a446d`. A second clean core checkout with the same basename received a distinct project, `private-tmp-wildfly-exp0003-other-wildfly-core`, rather than colliding with the first cache entry.

The fresh root indexes contained 115,043 nodes / 638,213 edges for core and 180,047 nodes / 773,017 edges for full. The shared cache reached approximately 1.7 GB, including 382 MB, 743 MB, and 581 MB project databases. CBM reported 11 parse-partial core files and 10 parse-partial full files; the full index deliberately excluded 157 files and eight directories.

## Conditions tested

- Ordinary `rg` searches and bounded source/Git inspection, with three repeated runs.
- CBM direct interface: fresh root indexing, symbol discovery, caller tracing, change detection, status/coverage, persistent-daemon warm queries, explicit refresh, and repeated tasks.
- Freshness transitions: method-body edit, new declaration, dirty-file state, rename, deletion, branch switch, explicit refresh, and a second checkout with the same repository basename.
- CBM MCP: one stdio initialization and real `search_graph` smoke query after direct behavior was understood.
- SCIP was not rerun, as Experiment 0003 explicitly defers it after the 0002 recovery showed generation success but no reliable task-facing lookup.

## Task corpus and answer-key method

The answer key used direct source inspection before measurement:

1. Core `ReadResourceHandler` implementation and `GlobalOperationHandlers` registration.
2. Full `UndertowExtension`, `registerSubsystemModel`, and `registerDeploymentModel` registration.
3. Qualified-name discovery for `UndertowExtension.getResolver`, followed by inbound tracing.
4. A changed Undertow source file and likely impacted declarations, registrations, and tests.

Exact files and declarations were classified as exact. CBM caller lists and impact lists were classified as useful structural navigation evidence requiring source verification. File-level nodes, generic method matches, and unrelated modules in the deletion impact list were treated as approximate or false-positive candidates, not semantic truth. A WildFly-aware review of representative results found the expected Undertow resource-definition callers, but the deletion impact list also contained unrelated-looking clustering, EJB, messaging, POJO, and test entries that require filtering.

## Measured results

| Condition | Measurement | Result |
|---|---:|---|
| Ordinary baseline | 3 runs per task, about 0.00–0.01 s per search | Complete for the small implementation/registration tasks and caller text search; source interpretation required |
| CBM cold core index | About 21 s; 115,043 nodes / 638,213 edges | Root index created successfully |
| CBM cold full index | About 21 s; 180,047 nodes / 773,017 edges | Root index created successfully |
| CBM cold direct queries | About 3.73–3.78 s per one-shot query | Found expected classes; qualified-name discovery plus trace returned 43 inbound callers |
| CBM warm repeated queries | First daemon-backed class query 3.75 s; subsequent queries 1.24–1.27 s | Three repetitions of class discovery and caller tracing remained correct; no model-token or source-reading reduction was measured |
| CBM declaration refresh | 15.21 s for explicit full-root refresh | New declaration appeared only after refresh; node count rose to 180,048 |
| CBM MCP smoke test | Process completed in about 4.3 s | Initialize plus real search succeeded; result shaping was readable and no MCP error was returned |

CBM’s direct graph was useful for the 43-caller trace after exact-name discovery. It was not a complete replacement for search: broad core discovery returned 30 matches with pagination, while ordinary search immediately showed the relevant registration and implementation paths.

## Cold/warm and amortization analysis

The two fresh indexes cost about 42 seconds combined before assistant queries, and the combined cache was approximately 1.7 GB. Warm direct calls after the first daemon-coordination cost were about 1.25 seconds each. Ordinary search remained below 0.01 seconds for these task cards.

For the repeated tasks actually measured, CBM did not amortize against ordinary search. Even excluding interpretation, the approximately 42-second initial index cost and approximately 1.25-second warm queries are much larger than the baseline’s milliseconds. CBM’s value is therefore capability-based for caller/impact exploration, not demonstrated end-to-end cost savings. A numeric break-even cannot be justified without a task corpus where ordinary search requires materially more source opening and assistant calls.

## Freshness and identity review

| Transition | Observed behavior | Safety assessment |
|---|---|---|
| Clean initial checkout | Root path and project were recorded; index status was ready | Correct initial association, but ref/commit identity was not exposed by CBM status |
| Method-body edit | `detect_changes` found the file and two seed symbols, with zero impacted symbols | Change detectable; stale graph remains usable only with an external warning/refresh policy |
| Add declaration | New method was absent before refresh; explicit refresh took 15.21 s and exposed it at lines 37–39 | Refresh works; automatic safety was not demonstrated |
| Dirty file | Git status showed the edit; CBM status still reported `ready` | Not safe to return without external dirty-state validation |
| Rename | Before refresh, the old path/node remained; after refresh, the new path was returned and old path coverage became `freshness: missing` | Explicit refresh removes stale path, but pre-refresh reuse is unsafe |
| Delete | Before refresh, the deleted class remained; after refresh, only the unrelated test extension remained | Explicit refresh removes stale nodes |
| Branch switch | Switching core to `2.x` left the old 115,043-node index reported as `ready`; old `ReadResourceHandler` results were returned | Failed safety criterion: silent stale reuse |
| Explicit branch refresh | Refresh replaced the index with 77,089 nodes / 442,185 edges | Works, but requires an external identity gate to force it |
| Same repository basename, second checkout | CBM created a distinct path-derived project and database | Collision avoided in this setup; path-derived identity is not a sufficient source/ref identity |
| Network removal | Not run as a destructive environment change; all queries used local generated state | Offline query remains plausible but unverified; refresh behavior remains unspecified |

The required Experiment 0002 stale-branch observation reproduced exactly. CBM itself did not reject or mark the old index stale. The minimum safe procedure therefore requires a wrapper that records canonical root, remote identity, ref, commit, dirty paths, and a source digest; validates them before every result; and forces refresh or falls back to `rg` on mismatch.

## Correctness review

The core and Undertow implementation/registration tasks were complete with CBM plus source verification. The qualified-name discovery and inbound trace were useful: CBM returned 43 callers concentrated in Undertow resource-definition classes, including registration-related methods. The graph did not prove management registration semantics or cross-repository SPI relationships.

The change detector correctly identified edited, renamed, and deleted paths, but impact output is not reliably semantic. The deletion transition reported 18 impacted symbols across unrelated-looking modules, including clustering/common, EJB, messaging, POJO, tests, and Undertow. Those results are leads for review, not a safe automatic impact set.

## Security and operational observations

CBM read source files and built local SQLite databases; no WildFly build was required. The binary ran from `/tmp`. The daemon required local IPC and caused macOS privacy prompts in this environment, which is an installation and developer-experience cost. The cache was shared among projects and grew to approximately 1.7 GB.

CBM’s status and coverage output surfaced ignored files and parse-partial ranges, but `status: ready` did not encode the Git ref or dirty state. Coverage reported missing renamed/deleted paths only after explicit refresh. An assistant must read these warnings and retain ordinary-search fallback.

## Failures and limitations

- CBM silently reused an index from the old branch after a branch switch.
- CBM detected dirty changes but did not itself force refresh or mark the project unusable.
- Impact results included broad and potentially misleading cross-module candidates.
- Warm CLI latency improved after daemon startup, but no full assistant transcript, token count, source-line count, or reduced tool-call count was measured.
- The second-checkout test used distinct path-derived project names; it did not test two checkouts deliberately forced to the same project identifier.
- Network removal and generated-source/dependency-input changes were not run.
- The MCP result was a smoke test, not a full freshness-aware MCP benchmark.

## Evidence confidence

High for fresh root indexing, cache size, branch-switch stale reuse, explicit refresh behavior, declaration/rename/delete observations, and MCP reproducibility. Moderate for caller usefulness and correctness because graph results were sampled rather than exhaustively reviewed. High that ordinary search is cheaper for the small task cards. High that CBM fails the unguarded freshness safety criterion. Low-to-moderate for total assistant-cost conclusions because model tokens and source-reading reductions were not captured.

## Decision matrix

| Approach | Task usefulness | Safety and operational viability | Finding |
|---|---|---|---|
| Ordinary search | Complete for small navigation; transparent for dirty/ref changes | Immediate, tiny state, no daemon or cache | Mandatory fallback and preferred first path for small tasks |
| CBM direct, unguarded | Stronger caller/impact navigation evidence than text search | Unsafe across branch, rename, delete, and dirty transitions because stale state can be returned as ready | Reject as direct assistant backend |
| CBM direct, guarded | Potentially useful for repeated caller/impact tasks | Requires external identity/digest gate, explicit refresh, coverage checks, and search fallback; large cache and daemon permissions remain | Optional guarded backend only; no demonstrated economic win yet |
| CBM MCP | Reproducible portable query surface | Inherits CBM freshness hazards and adds IPC/startup/privacy overhead | Smoke-tested only; do not expose without the same guard |

## Architectural consequence

This experiment informs, but does not decide, later architecture questions:

- **Generation location:** local generation is feasible, but roughly 42 seconds of cold indexing and daemon permissions are material developer costs.
- **Storage:** root caches are large and disposable; retention requires lifecycle and eviction rules.
- **Repository ownership:** separate projects for separate roots help avoid accidental collisions, but the assistant must not infer ownership from a project name alone.
- **Version identity:** canonical root, remote identity, ref, commit, dirty state, and source digest must be external validation inputs.
- **Freshness:** CBM requires a guard that rejects mismatches and forces explicit refresh; `ready` is not equivalent to source-current.
- **Assistant interface:** direct CBM and MCP can expose useful structural evidence, but both need the same guard and ordinary-search fallback.
- **Offline behavior:** local queries worked from generated state; offline refresh was not tested.
- **Security:** daemon IPC, cache permissions, binary provenance, and macOS privacy prompts are operational concerns.
- **WildFly-specific enrichment:** generic graph edges remain insufficient for management, deployment, service, provisioning, and Core-to-Full semantics.

## Recommended next step or stop condition

Stop unguarded CBM evaluation. If further work is justified, implement only an external validation harness in a future experiment—not a production indexer—that records source identity, checks CBM coverage/status, forces refresh on mismatch, and falls back to `rg`. Re-run the repeated caller/impact tasks through that guard with measured assistant calls, files, lines, and tokens. If guarded CBM still shows no repeatable total-cost savings, stop the generated-index investigation and retain ordinary search as the default.
