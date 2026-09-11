# Experiment Charter 0002: End-to-End Code Intelligence and Freshness

**Status:** Proposed

**Purpose:** Determine whether a ready-made local code-intelligence tool materially reduces the end-to-end cost of assistant work on WildFly, and whether its generated state remains trustworthy across repository roots, repeated tasks, dirty worktrees, and branch changes.

This is a follow-up to [Experiment 0001](0001-source-navigation-charter.md), not a production design. The previous experiment showed that ordinary search is excellent for small one-off lookups, while `codebase-memory-mcp` (CBM) produced the strongest agent-facing structural results. It did not yet establish whether CBM or SCIP saves enough total assistant cost to justify their setup, cache, build, and freshness requirements.

## Objective

Measure the total cost and correctness of the following conditions:

1. ordinary repository exploration using `rg`, directory listing, and bounded reads;
2. CBM through its direct CLI or plain structured output;
3. CBM through MCP, if the harness can expose it reproducibly;
4. SCIP through a task-facing lookup path;
5. JDT Language Server only as a secondary condition if a repository-root workspace can be made reproducible within the time budget.

The primary question is:

> For repeated, cross-file, and cross-repository tasks, does generated code intelligence reduce total assistant time, tokens, tool calls, and irrelevant source reading enough to justify its operational cost?

The secondary questions are:

- Does indexing the repository root avoid the freshness and identity problems seen in Experiment 0001?
- Is CBM’s graph useful beyond declaration lookup, especially for callers and impact analysis?
- Does SCIP provide a task-level benefit once its build and reactor requirements are included?
- Can an assistant recognize and safely handle stale, missing, partial, or unavailable indexes?

## Relationship to the first experiment

Experiment 0001 remains the baseline evidence. Do not silently reuse its generated state, caches, or conclusions. Re-run the relevant tasks under the conditions below, while preserving the same task wording where possible so that results remain comparable.

Treat these as hypotheses, not decisions:

- CBM may provide the best practical agent-facing experience.
- SCIP may provide useful semantic data but may be too build-coupled for routine use.
- Ordinary search may remain preferable for small or infrequent tasks.
- A generic graph may need WildFly-specific enrichment for management, provisioning, services, and cross-repository relationships.

## Repositories and identity

Use temporary clones or disposable worktrees of:

- `wildfly-core`;
- `wildfly`.

Use the POC `ai-index` branches as the current development branches for this experiment, as agreed for the POC. Also include one older maintenance ref or commit where practical. The older ref need not contain AI-context files; their absence is itself a supported condition.

For every indexed or queried state, record:

- repository name and absolute checkout path;
- Git repository root and index root;
- branch, ref, and commit SHA;
- worktree dirty/clean state;
- changed paths and a source-state identifier or digest where available;
- Java, Maven, and tool versions;
- index schema/generator version;
- build inputs and installed SNAPSHOT artifacts;
- cache location and whether it was shared with another checkout.

Do not index only `wildfly/undertow` or `wildfly-core/controller` for the primary comparison. Module-scoped indexing may be included as a diagnostic, but the primary CBM and SCIP runs must use the Git repository root so that freshness and path identity can be evaluated correctly.

## Conditions

### Baseline: ordinary exploration

The assistant may use repository files, `rg`, directory listing, bounded file reads, and ordinary Git inspection. It receives no generated source index. The baseline must be run from the same checkout state as each indexed condition.

### CBM direct interface

Use the installed CBM version from the experiment environment. Measure full indexing, incremental update, structural search, architecture query, caller/reference tracing, and impact-oriented queries where supported. Record the exact commands and output format used by the assistant.

The direct interface is a separate condition from MCP. Do not assume that MCP overhead, natural-language translation, or client configuration is representative of the underlying graph.

### CBM through MCP

Run only if the daemon, cache, and MCP client can be started reproducibly. Measure the same logical tasks as the direct-interface condition. Record daemon startup, IPC, client setup, cache sharing, and any query translation or result truncation.

Do not make MCP a prerequisite for using the underlying index. The experiment is comparing assistant access paths, not deciding that MCP must be part of the final architecture.

### SCIP

Generate SCIP from the repository root where the build supports it. Use a documented, read-only task-facing query path. If SCIP generation succeeds but no practical lookup path can answer the task, record generation as evidence but do not call it an assistant-successful condition.

Record build commands, reactor scope, downloaded dependencies, installed parent/BOM/SNAPSHOT artifacts, compilation side effects, unresolved symbols, artifact size, and lookup latency. Include the cost of preparing the build state in the total-cost calculation.

#### Reproducible SCIP setup

Use the successful setup sequence from Experiment 0001 before judging SCIP unavailable. Perform it only in disposable clones or worktrees, and record the exact commands and versions:

1. Use Java 17 or newer and Maven.
2. Install Coursier if it is not already available. The 0001 macOS setup used:

   ```bash
   brew install coursier/formulas/coursier
   ```

3. Validate the launcher with the explicit main class. `scip-java` does not provide a default main class:

   ```bash
   coursier launch org.scip-code:scip-java:0.13.0 \
     --main org.scip_code.scip_java.ScipJava -- --help
   ```

4. Prepare a resolvable Maven reactor in the disposable checkout. The 0001 run required `mvn clean install` before indexing; `wildfly-core/controller` also required the temporary core parent, test BOM, and sibling SNAPSHOT artifacts to be installed while resolving the reactor. Do not hide this preparation in the reported generation time.
5. Run the indexer from the Git repository root, with output outside the checkout:

   ```bash
   coursier launch org.scip-code:scip-java:0.13.0 \
     --main org.scip_code.scip_java.ScipJava -- index \
     --build-tool=maven --no-cleanup \
     --output=/tmp/scip/wildfly-core.scip
   ```

   Use `/tmp/scip/wildfly.scip` when indexing the `wildfly` checkout.

   The CLI uses `--no-cleanup`; `--cleanup=false` is invalid syntax. Do not replace a failed repository-root run with a module-scoped run without labelling it as a diagnostic comparison.
6. If macOS requests privacy or execution approval for the launcher, record the prompt and obtain approval only for the disposable experiment environment. A launcher failure caused by the host environment is setup evidence, not evidence that SCIP cannot index the source.

The setup sequence establishes generation viability only. The agent must still provide a real read-only lookup path over the resulting SCIP data for at least symbol definition, references/callers, or equivalent task-facing navigation. If no suitable existing client is available, report the SCIP recovery as successful generation but leave task usefulness unresolved; do not write a custom query implementation as part of this experiment.

### JDT Language Server (secondary)

Attempt this only if the repository-root workspace can be imported and queried within a bounded setup budget. The minimum useful demonstration is a cross-file definition, references, or call hierarchy result. If the harness again produces only same-file results or empty cross-file results, stop the condition and report it rather than expanding the harness indefinitely.

## Task corpus

Use the two Experiment 0001 tasks unchanged where possible:

1. In `wildfly-core`, locate the `read-resource` management-operation implementation, registration, and relevant callers.
2. In `wildfly`, locate the Undertow subsystem extension and its subsystem/deployment model registrations.

Add these tasks:

3. Trace callers of a method with a known qualified name, and separately discover the qualified name when the task begins. Compare precision, completeness, and irrelevant results.
4. Starting from a changed source file, identify likely impacted declarations, tests, and registrations.
5. Follow one real Core-to-Full or other cross-repository relationship selected from the hub topology and boundary documents.
6. Repeat a mixed set of the preceding tasks enough times to measure amortization of one-time generation/import cost.

For every task, prepare an expert-reviewed answer key containing relevant files, symbols, concepts, and acceptable partial answers. The answer key must distinguish:

- exact semantic relationships;
- useful navigation hints;
- plausible but unverified textual matches;
- false positives.

Do not require any AI-context file in the satellite repository for a task to be valid. If a repository has adopted the POC files, record that as context rather than as a condition required by the tool.

## Source-state and freshness matrix

Run the minimum matrix below against at least one repository. Use disposable worktrees or clones so that state transitions cannot damage the user’s checkouts.

| State change | Required observation |
|---|---|
| Clean initial checkout | Index identity and complete task results |
| Edit a method body without changing its signature | Whether locations/results remain valid and whether updates are needed |
| Add a method or declaration | Detection and visibility after refresh |
| Edit an uncommitted source file | Dirty-state detection, inclusion, and warning behavior |
| Rename or delete a source file | Removal of stale nodes and paths |
| Switch to another branch or older ref | Whether the index is rejected, rebuilt, or silently reused |
| Use a different checkout with the same repository name | Cache collision and identity behavior |
| Change a generated-source or dependency input | Whether affected results are marked stale |
| Remove network access after setup | Offline query and update behavior |

At least one test must reproduce the Experiment 0001 CBM root mismatch deliberately, then verify that the repository-root procedure avoids or clearly reports it.

## Execution protocol

Use the same assistant model, task wording, checkout contents, and answer keys for each comparable condition. A fresh experiment agent should perform the rerun to reduce confirmation bias; it must read the previous charter and findings but should not treat the previous decision matrix as an expected result.

For each condition:

1. Run a cold trial with no pre-existing index, workspace, or relevant cache.
2. Run a warm trial with generated state already available.
3. Run repeated tasks to calculate amortization and break-even.
4. Randomize condition order where practical.
5. Give the assistant a concise, equivalent usage guide for each interface.
6. Capture complete transcripts and raw outputs outside the repository.
7. Have a WildFly-aware reviewer assess correctness without assuming that graph edges are semantic truth.

If a condition cannot run, record the failure, setup cost, and reason. Do not replace it with a hand-built approximation without labelling the approximation as a different condition.

### SCIP recovery run

Because Experiment 0002 already produced the CBM and ordinary-search evidence, a rerun may be limited to the SCIP condition and its directly comparable task cards. Preserve the existing 0002 findings and append the SCIP measurements, lookup results, and any changed decision-matrix row. Do not repeat CBM solely to obtain a new SCIP result.

## Measurements

Capture per task:

- completed correctly: yes/no/partial;
- total wall-clock time, including generation/import and setup;
- time to first relevant file or symbol;
- model input/output tokens, if available;
- assistant tool calls and index queries;
- files and source lines opened;
- irrelevant files, lines, and graph results opened;
- exact, approximate, missing, and misleading results;
- index/update time and generated state size;
- memory, daemon, and cache requirements where observable;
- network access, downloads, and verification steps;
- source checkout changes and build side effects;
- offline behavior.

Use these derived comparisons:

```text
total cost per task = setup + generation/import/update + assistant task execution
amortized cost = one-time setup + repeated task costs / number of tasks
break-even tasks = one-time generation cost / baseline per-task savings
```

Report both raw tool latency and end-to-end assistant cost. A fast graph query is not a win if its generation or interpretation cost dominates.

## Security and operational observations

Record, but do not broaden the experiment into a security audit:

- whether indexing reads only source or executes repository build logic;
- whether SCIP requires compiling or resolving untrusted build configuration;
- downloaded binaries, checksums, signatures, and installation paths;
- cache permissions and cross-user or cross-checkout sharing;
- source paths and metadata exposed through output or MCP;
- behavior when a repository contains malformed or hostile source/build files;
- reproducibility from a clean machine or offline cache.

## Decision rules

Do not recommend CBM, SCIP, MCP, committed indexes, local caches, or CI publication based on indexing speed alone. A condition is a candidate for the first prototype only if it demonstrates:

1. a repeatable task-level reduction in total assistant cost on more than one task;
2. acceptable correctness for the relationships it claims to expose;
3. explicit behavior for dirty, switched, missing, and stale source states;
4. an installation and offline story appropriate for the intended users;
5. a fallback to ordinary search when generated state is unavailable.

Possible conclusions include:

- search remains the first prototype and CBM is worth a targeted optional experiment;
- CBM is suitable as an optional local assistant backend, with search fallback;
- CBM’s interaction model is useful but its generic graph needs WildFly-specific enrichment;
- SCIP is valuable for a separate CI or analysis workflow but not the default assistant path;
- MCP adds useful portability across assistants, or merely adds avoidable operational overhead;
- no generated index has demonstrated enough benefit yet.

## Deliverable

Write one concise findings document:

`architecture/experiments/results/0002-code-intelligence-follow-up-findings.md`

Use these sections:

```text
Hypothesis
Environment and source identities
Conditions tested
Task corpus and answer-key method
Measured results
Cold/warm and amortization analysis
Correctness and freshness review
Security and operational observations
Failures and limitations
Evidence confidence
Decision matrix
Architectural consequence
Recommended next experiment or stop condition
```

Keep raw transcripts, indexes, caches, binaries, and large benchmark outputs outside the repository. Do not modify satellite source, build files, AI-context files, or production architecture documents as part of this experiment.

## Handoff

The experiment agent should:

1. read this charter, Experiment 0001, the repository `AGENTS.md`, `README.md`, `llms.txt`, and the two architecture documents;
2. inspect the relevant satellite repository instructions and topology links;
3. create temporary clones or worktrees and verify repository roots before indexing;
4. if resuming the SCIP portion, read the setup procedure above and the 0001 setup findings before attempting launcher or Maven commands;
5. run the smallest complete comparison first, then extend only where the result is ambiguous;
6. preserve the same task wording and answer keys across conditions;
7. write only the concise findings document under `architecture/experiments/results/`;
8. stop before implementation, production architecture changes, or committing generated artifacts.

The findings must explicitly state which later architecture questions they inform: generation location, storage, repository ownership, version identity, freshness, assistant interface, offline behavior, security, and WildFly-specific enrichment.
