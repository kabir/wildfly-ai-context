# ADR 0001: Federated Hub-and-Spoke Context Architecture for Coding Agents

**Status**: Accepted

**Date**: 2026-08-20

## Context

The WildFly ecosystem spans multiple repositories — `wildfly-core`, `wildfly`, `wildfly-glow`, `wildfly-maven-plugin`, `wildfly-operator`, and others — each with distinct release cycles, codebases, and domain concerns. Coding agents (LLMs, AI-assisted development tools) working on WildFly-related tasks need architectural context to make correct decisions: what goes where, how components interact, what SPI contracts exist, and where to find implementation details.

Three problems arise with naive approaches:

1. **Token window decay**: Dumping the full context of multiple repositories into a single agent session exceeds practical context window limits and degrades agent reasoning quality. Large monolithic context files become stale, repetitive, and difficult for agents to navigate efficiently.

2. **Context fragmentation**: Without a central index, agents working on cross-repository tasks (e.g., adding a new subsystem that requires changes in both wildfly-core and wildfly) have no reliable way to discover which repository owns a component or where to find related documentation.

3. **Maintenance burden**: Centralized documentation that duplicates information from satellite repositories becomes stale as the underlying code evolves. Each repository's maintainers know their code best and are in the best position to maintain context documentation alongside it.

## Decision

Adopt a federated hub-and-spoke architecture for AI context distribution:

### Hub (`wildfly-ai-context`)

A dedicated repository serves as the ecosystem-wide context router:

- Maintains an `llms.txt` file following the llmstxt.org specification, acting as the top-level index into the ecosystem.
- Hosts cross-cutting architectural documentation that no single satellite repository owns: ecosystem topology, repository boundary definitions, and architectural decision records.
- Provides `AGENTS.md` with navigation instructions for coding agents performing cross-repository tasks.
- Does not duplicate subsystem-level or code-level documentation from satellite repositories.

### Spokes (Satellite Repositories)

Each satellite repository (`wildfly-core`, `wildfly`, etc.) maintains its own AI context files on a dedicated `ai-index` branch:

- Each satellite has its own `llms.txt` that indexes its subsystems, key directories, configuration schemas, and domain-specific guidance.
- Each satellite may include its own `AGENTS.md` or `CLAUDE.md` with repository-specific conventions for coding agents.
- Satellite context is maintained by the repository's own contributors alongside the code it describes.
- The `ai-index` branch is used rather than `main` to avoid polluting the primary development branch with AI-specific documentation files that are not part of the build.

### Navigation Protocol

1. An agent starting a task begins at the hub's `llms.txt`.
2. The hub routes the agent to the appropriate satellite repository or repositories based on the task domain.
3. The satellite's `llms.txt` provides component-level navigation within that repository.
4. For cross-repository tasks, the hub's boundary and topology documents resolve ownership questions.

This protocol keeps each agent session focused on the context it needs, avoids loading irrelevant subsystem documentation, and provides a deterministic path from "I need to work on X" to "here is the relevant code and documentation."

## Consequences

### Positive

- **Scalable context loading**: Agents load only the hub index and the satellite context relevant to their task. The full ecosystem context is never loaded into a single session.
- **Decentralized maintenance**: Each repository team maintains its own context files. There is no central bottleneck for keeping documentation current.
- **Consistent navigation**: The `llms.txt` specification provides a standard format that any LLM-aware tool can consume. The hub-and-spoke routing is predictable and documented.
- **Incremental adoption**: New repositories can be added to the ecosystem by creating their own `llms.txt` and adding a link to the hub. Existing repositories are not affected.
- **Cross-cutting concerns have a home**: Architectural decisions, boundary rules, and topology diagrams that span repositories have a dedicated, maintained location rather than being duplicated or lost.

### Negative

- **Link maintenance**: The hub must keep its links to satellite repositories' `llms.txt` files current. If a satellite restructures or changes its branch naming, the hub index breaks.
- **Branch coordination**: Using `ai-index` branches in satellites introduces a branch that must be kept reasonably in sync with the default branch. Stale `ai-index` branches produce misleading context.
- **Discovery dependency**: Agents that do not start at the hub (e.g., agents invoked directly in a satellite repository) may miss ecosystem-wide context. Satellite `AGENTS.md` files should include a pointer back to the hub for cross-repository tasks.
- **Additional repository**: The hub repository itself must be maintained. However, its content changes infrequently (primarily when repositories are added or architectural decisions change) and the maintenance cost is low relative to the benefit.

## Alternatives Considered

### Monolithic context dump

A single large context file containing all ecosystem documentation. Rejected because it exceeds practical context windows, becomes stale quickly, and creates a single point of maintenance failure.

### Per-query repository crawling

Agents dynamically crawl repository structures on each task. Rejected because it is slow, unreliable (agents may not know where to look), and consumes context window space with directory listings and file probing.

### Git submodules

Embedding satellite context into the hub via git submodules. Rejected because it reintroduces the monolithic loading problem and adds git submodule management complexity without clear benefit over URL-based linking.
