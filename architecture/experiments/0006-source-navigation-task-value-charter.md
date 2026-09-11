# Experiment Charter 0006: Source-Navigation Task Value Comparison

**Status:** Proposed

**Purpose:** Determine whether established source-navigation and code-search approaches reduce assistant work on representative WildFly tasks enough to justify an optional index layer.

This experiment follows the prior-art review and is deliberately separate from the immutable-registry experiment. It evaluates task value and correctness, not production packaging or central ownership.

## Objective

Compare the following evidence providers against the same tasks and source revisions:

1. ordinary search and bounded source reading;
2. cheap declaration indexing, such as Universal Ctags or the existing Bob-style output;
3. a graph-oriented local index, such as CBM;
4. a precise source-index format, preferably SCIP if a usable lookup path can be established;
5. one established searchable-source alternative, such as OpenGrok or Zoekt, if it can be installed without disproportionate setup.

The primary question is:

> Does any index-backed approach reduce total assistant effort while preserving or improving correctness and evidence quality on real WildFly navigation tasks?

Secondary questions:

- Which tasks benefit: declarations, callers, implementations, repeated navigation, or cross-repository routing?
- Does the benefit survive index generation, startup, refresh, and provenance checks?
- Are generic indexes sufficient, or do WildFly-specific facts justify enrichment?
- Does a common structured result model allow multiple assistants to consume the same evidence?
- When a precise index is unavailable or incomplete, does search fallback preserve correctness?

## Scope and non-goals

Use one or two real WildFly repositories, preferably `wildfly-core` and `wildfly`, with exact recorded commits. Include one feature-pack or dependency-context task where runtime source is absent if the fixtures are available.

Do not:

- select a permanent index technology;
- implement a production CLI, service, registry, MCP server, or cache;
- publish artifacts or modify satellite repositories;
- treat a generated index as authoritative without checking identity and coverage;
- claim an index is useful solely because it is faster to generate or query;
- repeat the full freshness investigation from Experiment 0003 except for the minimum checks needed to prevent stale results from contaminating task measurements.

The experiment may use disposable scripts, virtual environments, local servers, and caches outside this repository. Commit only the concise findings document.

## Required reading and prior evidence

Read repository instructions and:

- `architecture/source-intelligence-architecture-options.md`;
- Experiments 0001–0005 charters and findings;
- the relevant satellite `llms.txt` files where available;
- the prior-art links in the architecture-options document.

Treat the following as prior evidence, not as conclusions:

- ordinary search was strongest for the small 0001 task corpus;
- CBM produced useful structural/caller leads but had stale-state hazards;
- SCIP generation was possible but task-facing lookup was not established;
- central artifact distribution and compression are separate from task value.

## Experimental corpus and source identity

Record for every checkout:

- canonical repository identity and URL;
- local root and exact commit;
- branch/ref and whether the checkout is shallow;
- working-tree status;
- Java/Maven/toolchain versions;
- relevant dependency and SNAPSHOT resolution;
- generated-source availability and exclusions;
- indexer, parser, schema, and configuration versions.

Use clean checkouts for the principal comparison. Run a small contamination check by changing branch or source state and confirming that the provider is rejected or refreshed before results are used.

Do not compare an index built from one revision with source answers taken from another revision. If a provider cannot prove its source identity, label its result non-authoritative and exclude it from correctness claims.

## Task set

Create at least six task cards. Each task must have a short question, an expected evidence set, and a source-reviewed answer key. Use concrete WildFly code rather than synthetic examples.

Required task categories:

1. **Declaration:** locate a class, method, field, or configuration type and report exact source location.
2. **Caller/reference:** identify callers or references and distinguish direct evidence from search leads.
3. **Implementation:** find an implementation of an interface or abstract contract and report relevant alternatives.
4. **Repeated navigation:** answer two related questions about the same subsystem or call path so warm index/query effects can be observed.
5. **Cross-repository routing:** start from one repository and identify the owning repository or dependency needed for the answer.
6. **Partial workspace:** answer a feature-pack or dependency question while the runtime source repository is absent; the expected result must include an explicit unavailable-context statement where implementation evidence cannot be established.

At least three tasks must require caller, impact, implementation, or repeated-navigation evidence. At least one task must cross a repository boundary. If a category cannot be made fair for a provider, record that as an applicability limitation rather than replacing the task with an easier one.

## Providers and comparison protocol

### Required providers

- `rg` or equivalent ordinary search plus bounded file reads;
- Universal Ctags or the existing Bob-style declaration output;
- CBM using the safest available guarded invocation;
- SCIP generation and a lookup/inspection client if a usable path can be established.

### Optional provider

Attempt OpenGrok or Zoekt only if setup is bounded and reproducible. Record setup effort separately. Do not let an optional provider delay the required comparison.

For each provider:

1. prepare the source state and verify identity;
2. perform any one-time generation/setup;
3. run each task using the provider's normal assistant-facing interface;
4. capture the evidence returned before consulting another provider;
5. record whether fallback search was needed;
6. compare the answer with the source-reviewed key;
7. repeat enough times to expose warm/cold and task-order effects.

Use the same assistant model, task wording, and time budget for each provider where possible. If an assistant cannot consume a provider directly, use a minimal neutral adapter and report the adapter cost. Do not give one provider extra manually curated context without counting it.

Run at least three comparable repetitions per task/provider. Randomize task order or use separate fresh sessions where practical to reduce learning effects. Keep setup and generation timings separate from per-task timings, but include them in total-cost calculations for realistic one-off and repeated-use scenarios.

## Measurements

Record at least:

- answer correctness and completeness within the task scope;
- exact versus approximate/navigation-hint evidence;
- false-completeness and stale-result failures;
- assistant tool calls and provider queries;
- files and source lines opened;
- input/output/model-token estimates where available;
- time to first useful evidence and total task time;
- cold startup, warm query, generation, and refresh time;
- disk, memory, and network use where practical;
- fallback frequency and reason;
- setup complexity, dependencies, permissions, and failure modes.

Report medians and ranges per task and provider. Separate one-off tasks from repeated-task scenarios. Do not average away a correctness failure: a provider with lower latency but an incorrect or falsely complete answer must be marked unsafe for that task.

## Freshness and fallback checks

Before accepting task results, perform the minimum checks below:

- clean matching commit;
- branch switch or checkout change;
- one source edit or rename/delete;
- one generated-source or dependency-input change if the provider includes those inputs;
- one unavailable-index case.

For each check, verify that the provider either rejects the old result, refreshes it, or clearly downgrades it and falls back to search. A provider that returns stale data while reporting a healthy/ready state must not be counted as correct even if the original task answer happens to remain unchanged.

## Analysis rules

Classify each result as:

- **exact:** source location or relationship verified for the requested source state;
- **navigation evidence:** useful lead requiring source verification;
- **partial:** useful only within declared coverage;
- **unavailable:** the provider lacks the required source or relationship;
- **incorrect/stale:** contradicts the source key or current source state.

Compare providers on total assistant effort, not index query latency alone. A provider is a candidate optional accelerator only if it meets all of these provisional conditions:

- at least three caller/impact/repeated-navigation tasks are included;
- median assistant calls, opened source, or model tokens fall by at least 20% on at least three tasks;
- no correctness regression against the search baseline;
- no false-completeness result;
- no silent stale result in the freshness checks;
- setup, storage, and refresh costs are reported separately and are acceptable for the task frequency being claimed.

Failure to meet the thresholds does not prove that indexes are useless. It means the tested provider/task combination should remain optional and should not become a default prerequisite.

## Deliverable

Write:

`architecture/experiments/results/0006-source-navigation-task-value-findings.md`

Use these sections:

```text
Hypothesis
Corpus and source identities
Task cards and answer keys
Providers and setup
Measurement protocol
Task-level results
Freshness and fallback results
Cost model
Correctness and coverage analysis
Failures and limitations
Evidence confidence
Decision matrix
Architectural consequence
Recommended next step or stop condition
```

Include raw commands, large indexes, caches, transcripts, and generated reports only outside this repository or in ignored temporary storage. The findings document should be concise enough to review.

## Handoff

The experiment agent should:

1. verify the exact source and dependency state before generating or querying any index;
2. reuse existing setup instructions and artifacts where safe, but never reuse an artifact without identity validation;
3. keep ordinary search as the control and fallback;
4. distinguish tool/setup failure from evidence that the approach has no value;
5. record when a provider is not applicable to a task;
6. stop before implementing production integration or modifying external repositories;
7. write only the 0006 findings document and report any untested claims explicitly.
