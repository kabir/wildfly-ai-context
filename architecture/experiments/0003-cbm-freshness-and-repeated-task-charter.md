# Experiment Charter 0003: CBM Freshness and Repeated-Task Value

**Status:** Proposed

**Purpose:** Determine whether Codebase Memory MCP (CBM) can be used as an optional local assistant backend without returning results from the wrong repository state, and whether it reduces end-to-end assistant cost for repeated caller and impact-analysis tasks.

This is a focused follow-up to Experiments 0001 and 0002. It does not reopen the broad comparison of source-index technologies. SCIP generation is now established but its available lookup clients were not task-usable; SCIP is deferred unless a reliable existing consumer is identified.

## Objective

Compare ordinary search with CBM for tasks where a graph may provide more value than text search:

- caller and inbound-reference discovery;
- change-impact exploration;
- repeated navigation across a large repository;
- structural discovery followed by source verification.

The primary question is:

> Can CBM provide repeatable assistant-cost savings while explicitly rejecting or refreshing indexes when repository identity or source state changes?

The experiment must separate two claims:

1. CBM can produce useful navigation evidence.
2. CBM can safely and economically provide that evidence to an assistant.

The first claim is already partially supported. This experiment concentrates on the second.

## Scope and non-goals

Use only:

- ordinary `rg`/file-read exploration as the baseline;
- CBM’s direct local interface as the primary indexed condition;
- one small CBM MCP smoke test after the direct interface is understood.

Do not:

- implement a WildFly indexer;
- implement a SCIP query client;
- modify satellite repositories or their build files;
- commit CBM databases or caches;
- decide the final MCP, cache, CI, or artifact architecture;
- treat CBM graph edges as proof of WildFly management, service, provisioning, or cross-repository semantics.

## Repositories and source identities

Use disposable clones or worktrees of:

- `wildfly-core` at the POC `ai-index` ref;
- `wildfly` at the POC `ai-index` ref;
- one older maintenance ref for at least one repository if this does not obscure the primary freshness test.

The primary index must use the Git repository root, not a module directory. Record for every run:

- canonical repository root and checkout path;
- remote identity where available;
- branch/ref and commit SHA;
- dirty/clean state and changed paths;
- a source-state digest or equivalent external identity record;
- CBM version, parser/schema version, cache root, and project identifier;
- index root, included/excluded paths, warnings, and parse-partial files.

The harness may calculate identity metadata outside CBM for evaluation. It must not silently infer that a project name or filesystem path uniquely identifies a source state.

## Conditions

### Baseline

Use `rg`, directory listing, bounded reads, and ordinary Git inspection. Run the baseline against the same checkout states used for CBM. Do not give the baseline access to generated graph output.

### CBM direct interface

Use the proven CBM setup from Experiment 0002. Measure:

- clean repository-root indexing;
- persistent-daemon warm queries;
- structural search and exact symbol discovery;
- caller/inbound tracing after qualified-name discovery;
- change detection and impact-oriented queries;
- explicit status/coverage information before returning results.

Record one-shot coordination overhead separately from warm daemon query time. Do not claim a warm-query benefit from a one-shot invocation.

### CBM MCP smoke test

After the direct condition is complete, run one equivalent search or trace through MCP if the stdio client can be started reproducibly. Record initialization, project selection, query, result-shaping, and shutdown costs. This is an interface smoke test, not a full MCP benchmark.

## Task corpus

Use the following task cards, with answer keys prepared before measurement:

1. Locate the `read-resource` implementation and registration in `wildfly-core`.
2. Locate Undertow subsystem and deployment model registration in `wildfly`.
3. Discover the qualified name of `UndertowExtension.getResolver`, then trace inbound callers.
4. Starting from a changed Undertow source file, identify likely impacted declarations, registrations, and tests.
5. Repeat tasks 2–4 enough times to measure whether CBM’s one-time indexing cost is amortized.

The answer key must classify results as exact, useful navigation evidence, approximate textual/structural evidence, or false positive. A WildFly-aware reviewer must inspect a sample of caller and impact results.

Do not require AI-context files in the satellite repositories. Record their presence or absence only as environmental context.

## Freshness and identity matrix

Use disposable clones/worktrees and record CBM status before and after each transition.

| Transition | Required result |
|---|---|
| Clean initial checkout | Index is associated with the correct root, ref, commit, and source state |
| Edit a method body without changing its signature | Change is detected or the index is explicitly marked stale |
| Add a declaration | New symbol is absent until refresh, then present after refresh |
| Edit an uncommitted file | Results include the edit or clearly warn that the index excludes it |
| Rename or delete a source file | Old paths/nodes are removed or the index is rejected as stale |
| Switch from `ai-index` to an older or newer ref | Old results are never returned as ready for the new ref |
| Reuse the same repository name in another checkout | Cache collision is prevented or clearly reported |
| Force an explicit refresh after a ref switch | New symbols and paths replace old state |
| Remove network after setup | Queries work offline; refresh behavior is explicit |

The branch-switch test must reproduce the Experiment 0002 stale-index observation, then test the strongest available explicit refresh or identity-validation procedure. If safe behavior requires an external wrapper, document that as an architectural requirement rather than hiding it in the test harness.

## Execution protocol

Use fresh disposable indexes and caches. The same task wording and answer keys must be used for baseline and CBM comparisons.

1. Run the clean baseline tasks.
2. Cold-index each repository with CBM from the repository root.
3. Run the same tasks through the direct interface.
4. Keep the daemon alive and run warm repeated tasks.
5. Apply the freshness transitions one at a time and record status/results.
6. Run the optional MCP smoke test only after direct behavior is understood.
7. Capture raw transcripts, status output, and timing data outside this repository.
8. Have an independent WildFly-aware reviewer assess caller and impact precision.

If CBM cannot safely distinguish two source states, record that as a failed safety criterion even if the graph results are otherwise useful.

## Measurements

Capture per task and per source state:

- correctness: complete/partial/incorrect;
- total elapsed time, split into index/update, daemon startup, query, and assistant interpretation;
- model tokens, assistant calls, graph queries, files opened, and lines read where available;
- irrelevant or misleading results;
- cold and warm cache size;
- memory, daemon, permission, and network requirements;
- status/coverage warnings and excluded or parse-partial files;
- behavior after dirty edits, branch switches, renames, deletions, and explicit refresh.

Use:

```text
total cost = setup + generation/update + assistant task execution
amortized cost = one-time cost + repeated task costs
break-even tasks = one-time CBM cost / baseline per-task savings
```

Do not treat raw graph-query latency as an assistant-cost reduction unless the assistant actually opens fewer irrelevant files, makes fewer calls, or completes tasks more reliably.

## Decision rules

CBM is a candidate for the first prototype only if it demonstrates all of the following:

1. repeatable savings on caller or impact tasks, not merely declaration lookup;
2. acceptable reviewed precision for the relationships it exposes;
3. no silent reuse of an index from another ref, checkout, or dirty source state;
4. a bounded setup and cache cost appropriate for developers;
5. ordinary search fallback when CBM is missing, stale, partial, or unavailable.

If the graph remains useful but freshness is unsafe, the result should recommend a guarded optional backend rather than direct assistant access. If freshness can only be repaired by a wrapper, specify the wrapper’s required identity and validation contract without implementing it.

Stop the generated-index investigation if CBM fails to show repeatable total-cost savings after safe freshness handling, or if reviewed caller/impact results are too approximate to guide source navigation.

## Deliverable

Write:

`architecture/experiments/results/0003-cbm-freshness-and-repeated-task-findings.md`

Use these sections:

```text
Hypothesis
Environment and source identities
Conditions tested
Task corpus and answer-key method
Measured results
Cold/warm and amortization analysis
Freshness and identity review
Correctness review
Security and operational observations
Failures and limitations
Evidence confidence
Decision matrix
Architectural consequence
Recommended next step or stop condition
```

Keep generated state and raw evidence outside the repository. Do not modify satellite repositories or production architecture documents.

## Handoff

The experiment agent should:

1. read this charter, the 0001 and 0002 charters, both findings documents, and the repository instructions;
2. use the exact CBM version and setup notes from 0002 unless a newer version is deliberately compared;
3. create fresh disposable clones and verify canonical Git roots before indexing;
4. run the baseline and CBM direct comparison before attempting MCP;
5. preserve the stale-branch reproduction as a required control;
6. write only the 0003 findings document;
7. stop before implementation or production architecture decisions.
