# Hypothesis

Established navigation indexes will reduce assistant effort on caller, implementation, and repeated-navigation tasks enough to justify optional local acceleration, while ordinary search remains the correctness fallback. The tested evidence does not support making any index mandatory.

# Corpus and source identities

The principal corpus used the POC `ai-index` branches and existing local checkouts:

| Repository | Ref/commit | Working tree | Tool/build state |
|---|---|---|---|
| `wildfly-core` | `ai-index`, `d511a0d1e5c34305f8a6d2d8f917fab1180c65b8` | clean | Java 17.0.18, Maven 3.9.14; parent is `34.0.0.Beta4-SNAPSHOT` |
| `wildfly` | `ai-index`, `39e7ce3cb5a23f354e3825bece6d4ccb15c1ab22` | `?? .serena/` only; preserved | Java 17.0.18, Maven 3.9.14; parent is `42.0.0.Beta1-SNAPSHOT` |
| `wildfly-galleon-feature-packs` | `ai-index`, `ba872664f8f97f79c78a47e28afa4b4f3a7e5030` | clean | registry/configuration source; no runtime source supplied |

The checkouts are not shallow. Generated-source availability was not established; results are source-tree results and do not claim complete generated or dependency coverage. Existing CBM and SCIP artifacts were from the same recorded commits and were used only with identity caveats. Raw indexes, tags, logs, caches, and transcripts remain under `/private/tmp` or `/tmp`.

# Task cards and answer keys

Six concrete cards were prepared from source before provider comparison. T2–T4 are caller, implementation, or repeated-navigation cards.

| Card | Question and source-reviewed answer key |
|---|---|
| T1 Declaration | Locate `ReadResourceHandler`: `wildfly-core/controller/.../ReadResourceHandler.java:73`; `GlobalOperationHandlers` references it at lines 102 and 106. |
| T2 Caller/reference | Identify registration references for `ReadResourceHandler`: `GlobalOperationHandlers.java:102,106`, plus its handler fields/constructors in the implementation. These are source-verified references, not every textual occurrence. |
| T3 Implementation | Find implementations of core `OperationStepHandler`; examples include `AbstractAddStepHandler.java:35`, `NoopOperationStepHandler.java:16`, and `CompositeOperationHandler.java:32`. An exhaustive answer requires declared coverage. |
| T4 Repeated navigation | Starting at `UndertowExtension.java`, answer both subsystem and deployment registration: `:45` registers `UndertowRootDefinition`; `:48` registers `DeploymentDefinition`. |
| T5 Cross-repository routing | `wildfly-core/controller/.../Extension.java:19` defines `Extension`; `wildfly/undertow/.../UndertowExtension.java:24` implements it. The relationship crosses Core and Full. |
| T6 Partial workspace | The feature-pack registry declares `org.wildfly:wildfly-galleon-pack:41.0.0.Final` at `41.0.0.Final/provisioning-bare-metal.xml:4`. With runtime source absent, the answer may route to `wildfly`; it must not claim the concrete runtime implementation or callers. |

# Providers and setup

`rg` plus bounded `sed`/`nl` reads was the control and was run live three times per card. It required no setup and saw the current checkout.

The only installed ctags was Apple/BSD `/usr/bin/ctags`; it has no Java parser and no recursive `-R` option. File-list invocation took about 0.12 s for the Core source set and 0.03 s for Undertow, but emitted only a few Java enum/annotation-like tags, not T1–T5 declarations. Universal Ctags and Bob-style declaration output were unavailable; this is a setup failure, not evidence that a Java-aware declaration index has no value.

CBM 0.10.8 was available as a prior disposable binary/cache. A guarded invocation checked repository root and exact commit before querying. It rejected reuse of the prior path-derived cache for the current checkout; a direct CLI attempt then timed out during local daemon/IPC startup after 12 s. Historical same-commit CBM evidence from Experiment 0003 is retained for comparison: root indexing took about 21 s per repository, warm graph calls about 1.25 s, and a qualified Undertow method trace returned 43 caller candidates. Those results were navigation evidence requiring source review, not exhaustive semantic truth.

SCIP generation artifacts from the same commits were inspected through the official converted SQLite path. The artifacts were generated with `scip-java 0.13.0`; prior generation including Maven preparation was about 200.8 s for Core and 580.4 s for Full. Read-only queries took about 0.01 s and found global symbols/documents, but both databases had zero `chunks`, zero `mentions`, and zero definition-enclosing ranges. This made declarations/document identity inspectable but callers, locations, and implementations unusable through that path.

OpenGrok and Zoekt were not attempted: neither was installed, and setup would not have been bounded relative to the required providers.

# Measurement protocol

Each card used the same repository state and answer key. Search and declaration probes were repeated three times; ctags generation and queries were measured three times where the provider existed. Measurements captured wall-clock query/generation time, output lines, source files/line locations exposed, setup and cache state, and correctness class. No assistant/model harness was available, so model calls, model tokens, and end-to-end assistant latency were not observable; provider invocations and opened source lines are reported as lower-level proxies, not assistant savings.

The control used direct `rg` and bounded source reads. Index providers were queried before consulting the source key. An index result was classified as exact only after source verification; graph candidates were hints. No negative claim was made from incomplete CBM or SCIP coverage.

# Task-level results

| Provider | T1–T6 comparable result | Repetition/latency observation | Evidence quality |
|---|---|---|---|
| `rg` + bounded reads | All six answer keys were reachable; T6 correctly stopped at routing/unavailable runtime context | Three repetitions per card; representative probes were below timer resolution (`<1 ms` in the shell loop, roughly 0.2–0.4 s including macOS filesystem startup in the earlier module run) | Exact after source read; transparent coverage |
| Apple/BSD ctags | T1–T5 not answerable as Java declarations; T6 is configuration text, outside ctags scope | Generation 0.12/0.03 s, three query repetitions; generated files contained only 54 Core and 5 Undertow tags | Incorrect/inapplicable for Java; no callers or freshness model |
| Guarded CBM | Current invocation refused mismatched cached state; historical same-commit run supplied T2/T4 caller leads and structural hints | Historical cold index about 21 s/repository; warm calls about 1.25 s; current guard/IPC path did not return task data | Partial navigation evidence; unsafe without the guard and source verification |
| SCIP SQLite inspection | T1 symbol identity/document lookup partly possible; T2–T5 locations/references unavailable because chunks/mentions/ranges were empty; T6 not represented | Three read-only inspections were effectively 0.01 s each after generation; generation cost remained 200.8/580.4 s | Partial symbol/document metadata; not a task-facing reference index |

Search used 1–2 focused files for T1, T2, T4, and T5, bounded result sets for T3, and 6–10 configuration/source hits for T6. CBM exposed graph candidates without proving WildFly registration semantics. No provider demonstrated a 20% reduction in assistant calls/tokens on three tasks because those assistant measurements were unavailable; therefore the charter’s optional-accelerator threshold was not met.

# Freshness and fallback results

The clean matching-commit case was valid for the source checkouts and the matching historical index artifacts. Search reflected the live state immediately.

The required branch/ref and source-edit/rename/delete checks were reproduced by the prior CBM experiment: CBM remained `ready` after a branch switch and dirty edit, retained old symbols before refresh, and required explicit refresh for declaration, rename, and deletion changes. This is a silent-stale-result failure unless an external guard rejects the result and falls back to search. The guarded invocation in this experiment rejected a cache/root mismatch before task use.

An unavailable-index case was exercised by the guarded CBM path and by the empty SCIP lookup path. Neither was converted into “no callers” or “no implementation.” The correct result was fallback or unavailable context.

Generated-source and dependency-input changes were not exercised in this comparison. SNAPSHOT coordinates were recorded, but no new Maven resolution or timestamped SNAPSHOT mapping was performed. Those cases remain freshness gaps.

# Cost model

For one-off small tasks, ordinary search dominates: zero generation, negligible storage, no daemon, and immediate fallback. BSD ctags generation was cheap but not applicable to Java. SCIP’s generation cost is prohibitive for an interactive one-off task. CBM’s prior root generation was cheaper than SCIP but still about 42 s for two repositories and approximately 1.7 GB of cache; warm queries did not amortize against millisecond search on these cards.

For repeated caller/impact work, CBM has a capability advantage because it produced caller candidates that text search alone does not resolve cleanly. The experiment did not measure model-token or source-reading reduction, and its graph candidates require validation. SCIP did not expose usable caller data. No economic break-even is justified.

# Correctness and coverage analysis

Search was the only provider with transparent live coverage and no silent stale state. CBM produced useful structural/caller hints but not guaranteed semantic callers, WildFly management relationships, generated-source relationships, or Core-to-Full edges. SCIP symbol metadata was exact only within its recorded generated/build identity; its converted lookup database had no occurrences or locations. The feature-pack task demonstrated that routing and version evidence are useful, while absent runtime source remains unavailable.

The evidence classes remain separate: local source, structural navigation hint, SCIP symbol/document metadata, feature-pack declaration, hub routing, and unavailable context. No index result was used to assert a negative relationship outside declared coverage.

# Failures and limitations

- No Universal Ctags, Bob output, OpenGrok, or Zoekt was available.
- CBM’s current guarded invocation could not use the prior path-derived cache and its daemon path timed out; historical CBM evidence was not relabelled as a current successful query.
- SCIP generation was inherited from the prior exact-commit run; task-facing lookup remained incomplete.
- No model-call harness measured assistant calls, opened files/lines attributable to the model, token estimates, or time to first useful answer.
- Generated-source changes, dependency/SNAPSHOT refresh, offline refresh, and a fresh branch-switch run for SCIP/ctags were untested.
- The feature-pack partial workspace did not include a runtime source checkout or an exact runtime source index.
- The task set is six cards, not a statistically broad WildFly workload; Universal Ctags failure must not be generalized to Java-aware declaration tools.

# Evidence confidence

High confidence: repository identities, working-tree states, source answer keys, search results, Apple/BSD ctags limitations, SCIP schema counts, and the explicit partial-workspace boundary. Moderate confidence: CBM caller usefulness and repeated-task capability, based on prior exact-commit runs and sampled source review. Low confidence: any claim about total assistant effort reduction, because the model harness and token measurements were unavailable.

# Decision matrix

| Approach | Task value | Safety/operational result | Decision |
|---|---|---|---|
| Ordinary search | Complete for all six bounded cards | Fresh, cheap, interpretable | Mandatory control and fallback; retain as default |
| Java-aware declarations | Not tested: Universal Ctags/Bob unavailable | Setup gap only | Revisit only with a real Java-aware provider |
| CBM, unguarded | Caller/impact leads and repeated structural navigation | Silent stale branch/dirty/rename/delete state; large cache | Do not expose unguarded |
| CBM, guarded | Potential capability for caller tasks | Current guarded path correctly refused mismatched state; economic benefit unproven | Optional experiment only, with identity guard and search fallback |
| SCIP | Generated artifacts and symbol metadata | High build/setup cost; no usable occurrences/locations in tested lookup path | Do not select as assistant backend on this evidence |
| OpenGrok/Zoekt | Untested | Setup not bounded | No decision |

# Architectural consequence

The experiment supports the existing search-first, evidence-aware direction. An optional accelerator may be useful for repeated caller/impact exploration, but only behind repository/ref/commit/dirty-state/build-input validation, explicit coverage, and automatic search fallback. Index generation, storage, and distribution remain separate costs; neither fast queries nor successful generation establishes assistant value.

The hub remains useful for cross-repository routing and provenance. A feature-pack declaration may identify `wildfly` as the owning source repository and exact released coordinate, but it must not be presented as runtime implementation evidence. No production CLI, service, MCP contract, satellite change, or index registry is justified by this experiment.

# Recommended next step or stop condition

Stop before implementation. Keep ordinary search as the default. If another experiment is authorized, provide a fixed assistant harness and a Java-aware declaration provider, then rerun at least three caller/impact/repeated-navigation cards with model calls, source lines, tokens, and freshness checks including generated/dependency inputs. Treat CBM as optional only if guarded trials show the charter’s required measurable effort reduction with no correctness or stale-result regression. SCIP should remain deferred unless an existing lookup client exposes verified locations and references.
