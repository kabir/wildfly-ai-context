# wildfly-ai-context

Cross-repository AI context hub for the WildFly ecosystem. This repository provides a central index and architectural documentation that helps coding agents (LLMs, AI-assisted development tools) navigate the multi-repository WildFly codebase.

This repository does not contain application server source code. It contains:

- **`llms.txt`** — A top-level index following the [llmstxt.org](https://llmstxt.org) specification that routes agents to the correct satellite repository for any given task.
- **`architecture/`** — Ecosystem-level documentation covering component topology, repository boundaries, and SPI contracts.
- **`adrs/`** — Architectural Decision Records for ecosystem-wide decisions.

## How it works

The WildFly ecosystem spans multiple repositories (`wildfly-core`, `wildfly`, `wildfly-glow`, `wildfly-maven-plugin`, `wildfly-operator`, etc.), each with its own codebase and release cycle. Each satellite repository maintains its own `llms.txt` on an `ai-index` branch with component-level detail.

This hub connects them with a federated approach:

1. An agent starting a task reads `llms.txt` here to find the relevant satellite repository.
2. The satellite's own `llms.txt` provides subsystem-level navigation within that repository.
3. For cross-repository tasks, `architecture/repository-boundaries.md` resolves ownership questions and `architecture/ecosystem-topology.md` explains how components interact.

See [ADR 0001](./adrs/0001-hub-and-spoke-ai-context.md) for the rationale behind this architecture.

## Onboarding a new repository

Onboarding is a two-step process: first prepare the satellite repository, then register it in the hub.

### Step 1: Prepare the satellite repository

Run the `prepare-repository` skill with the local path to the target repository checkout. The skill will:
- Create `llms.txt` indexing the repository's components and documentation
- Create `AGENTS.md` with cross-repo routing back to the hub (and a `CLAUDE.md` symlink)
- Determine the raw GitHub URL for the repository's `llms.txt`

### Step 2: Register in the hub

Run the `onboard-repository` skill with the URL of the repository's root `llms.txt`. The skill will:
- Fetch and analyze the remote `llms.txt`
- Add the repository to the hub's `llms.txt` index
- Update `architecture/ecosystem-topology.md` with the repository's components and interactions
- Update `architecture/repository-boundaries.md` with ownership rules and SPI contracts
- Update the satellite repositories table in this README

Review the generated changes after each step. Both skills flag areas where they made assumptions or didn't have enough information.

If you are not using an agent with skill support, you can follow the same steps manually — the skill files at `.agents/skills/prepare-repository.md` and `.agents/skills/onboard-repository.md` document the full process.

## Satellite repositories

| Repository | Status | Description |
|---|---|---|
| [wildfly-core](https://github.com/wildfly/wildfly-core) | Indexed | Core server management model, controller, CLI, and kernel runtime |
| [wildfly](https://github.com/wildfly/wildfly) | Indexed | Full application server — Jakarta EE subsystems, clustering, Elytron, distribution packaging |
| wildfly-glow | Planned | Provisioning analysis engine for determining minimal server configuration |
| wildfly-maven-plugin | Planned | Developer-facing build tooling for packaging, provisioning, and running WildFly |
| wildfly-operator | Planned | Kubernetes operator for managing WildFly deployments |
