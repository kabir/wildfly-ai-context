# WildFly AI Context Hub — Agent Instructions

This repository is the cross-repository context manager for the WildFly ecosystem. It does not contain application server source code. It contains architectural documentation, boundary definitions, and an `llms.txt` index that routes agents to the correct satellite repository for any given task.

## Navigating the Ecosystem

When working on a task that spans multiple WildFly repositories, or when you need to determine which repository owns a component:

1. Read `llms.txt` in this repository root. It indexes every satellite repository and links to their own `llms.txt` files with component-level detail.
2. Follow the satellite link for the repository relevant to your task. Each satellite `llms.txt` provides subsystem-level navigation, key file pointers, and domain-specific guidance.
3. Consult `architecture/repository-boundaries.md` when you are uncertain whether a change belongs in `wildfly-core` or `wildfly`. The boundary rules there are authoritative.
4. Consult `architecture/ecosystem-topology.md` to understand how components interact across repository boundaries, especially for provisioning and build-time flows.

## Repository Scope Rules

- **wildfly-core**: Kernel runtime, management model, controller, CLI, JMX, logging, patching, remoting, deployment scanner, core security realms, and all subsystems required for the base server to boot and accept management operations.
- **wildfly** (full distribution): Jakarta EE subsystems (EJB, JPA, Servlet, CDI, JAX-RS, JMS, MicroProfile), clustering/HA, Elytron security integration, distribution packaging (feature packs, BOMs), and the full test suite.
- **wildfly-ai-context** (this repo): Ecosystem-level architecture docs, ADRs, topology diagrams, and the hub `llms.txt`. No server source code lives here.

## Contributing New ADRs

Architectural Decision Records live in `adrs/` and follow this convention:

- Filename: `NNNN-short-slug.md` where NNNN is the next sequential number, zero-padded to four digits.
- Use the template structure from `adrs/0001-hub-and-spoke-ai-context.md`: Title, Status, Context, Decision, Consequences.
- Status values: `Proposed`, `Accepted`, `Deprecated`, `Superseded by ADR NNNN`.
- When an ADR is superseded, update its status and link to the replacement. Do not delete old ADRs.
- After adding or modifying an ADR, update the link in `llms.txt` under the "Global Architecture & Contracts" section if the ADR is significant enough to surface at the hub level.

## Updating Topology and Boundary Documents

- `architecture/ecosystem-topology.md` should be updated when new repositories are added to the ecosystem, when a significant component moves between repositories, or when the provisioning/build flow changes materially.
- `architecture/repository-boundaries.md` should be updated when SPI contracts between repositories change, when schema versioning rules are modified, or when new boundary enforcement mechanisms are introduced.
- Keep both documents factual and current. They are consumed by agents making routing decisions; stale information causes incorrect repository targeting.

## Cross-Repository Task Workflow

When an agent receives a task that touches multiple repositories:

1. Start here. Read `llms.txt` to identify which repositories are involved.
2. Read the boundary document to confirm the task decomposition respects repository ownership.
3. Navigate to each satellite repository's `llms.txt` for component-level context.
4. Execute changes in each repository according to its own `AGENTS.md` or `CLAUDE.md` conventions.
5. If the task reveals a boundary ambiguity or a missing topology relationship, update the relevant document in this hub as part of the task.
