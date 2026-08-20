# WildFly AI Context: Progressive Disclosure Architecture

The `wildfly-ai-context` repository is the central knowledge hub and context router for coding agents working across the WildFly ecosystem. It enables AI-assisted development across the ecosystem's many tightly coupled repositories without overwhelming prompt context windows or duplicating documentation.

## Contents

- [The Progressive Disclosure Framework](#the-progressive-disclosure-framework)
- [Hub-and-Spoke Ecosystem Integration](#hub-and-spoke-ecosystem-integration)
- [Repository Contents](#repository-contents)
- [Satellite Repositories](#satellite-repositories)
- [Onboarding a New Repository](#onboarding-a-new-repository)
- [Maintainer Guidelines](#maintainer-guidelines)

## The Progressive Disclosure Framework

Rather than dumping full documentation suites into an agent's context window, this ecosystem uses **progressive disclosure**. Information is fetched in lightweight tiers, resolving context on demand:

```
[ Tier 1: Ecosystem Router ] ──> Central llms.txt / Local AGENTS.md
                                       │
[ Tier 2: Satellite Index  ] ──> Satellite llms.txt / Deep dives / Architecture docs
                                       │
[ Tier 3: Source Documents ] ──> Raw .adoc guides / Source code
```

| Tier | Component | Purpose | Format |
|------|-----------|---------|--------|
| **1: Router** | Central `llms.txt` and local `AGENTS.md` | Ecosystem routing, build rules, component boundaries | Short Markdown |
| **2: Index** | Satellite `llms.txt`, `.agents/deep-dives/`, and `architecture/` docs | Subsystem navigation, internal mechanics, SPI boundaries | Markdown |
| **3: Source** | Raw documentation and source code | Single source of truth for end-user and admin configuration | AsciiDoc / Java |

An agent starting a task reads Tier 1 to find the relevant repository, follows links to Tier 2 for subsystem-level navigation, and only dips into Tier 3 when it needs specific implementation details or canonical documentation.

### Deep dives

Some contributor-facing technical knowledge is too detailed for `llms.txt` summaries but doesn't belong in end-user `.adoc` documentation — internal pipeline mechanics, rule-engine internals, SPI implementation patterns. These live in `.agents/deep-dives/` within the satellite repository, linked from its `llms.txt`. Deep dives give agents a concise, LLM-friendly explanation of complex internals without requiring them to reverse-engineer the source code or wade through user-oriented guides that cover a different audience.

## Hub-and-Spoke Ecosystem Integration

Every repository in the WildFly ecosystem maintains a lean, local `AGENTS.md` file (symlinked as `CLAUDE.md` for Claude Code). Local files handle repository-specific build commands and conventions, and delegate cross-repository or deep-domain context to this central hub.

```
                  ┌──────────────────────────────┐
                  │   wildfly-ai-context Hub     │
                  │   (Central Ecosystem Index)  │
                  └──────────────┬───────────────┘
                                 │
      ┌──────────────┬───────────┼───────────┬──────────────┐
      ▼              ▼           ▼           ▼              ▼
┌───────────┐ ┌───────────┐ ┌─────────┐ ┌─────────┐ ┌────────────┐
│ wildfly-  │ │ wildfly   │ │ wildfly │ │ wildfly │ │ feature    │
│ core      │ │ (full     │ │ -glow   │ │ -maven- │ │ pack       │
│ (kernel)  │ │  server)  │ │         │ │ plugin  │ │ extensions │
└───────────┘ └───────────┘ └─────────┘ └─────────┘ └────────────┘
```

### What stays local vs. what routes to the hub

Each satellite `AGENTS.md` covers repository-specific concerns: build commands, test conventions, module structure, and coding patterns. When an agent encounters a cross-repository question — "which repository owns this SPI?", "how does provisioning interact with the runtime?" — the local `AGENTS.md` routes it to this hub's `llms.txt` and architecture documents.

## Repository Contents

This repository does not contain application server source code. It contains:

- **`llms.txt`** — A top-level index following the [llmstxt.org](https://llmstxt.org) specification that routes agents to the correct satellite repository for any given task.
- **`architecture/`** — Ecosystem-level documentation covering component topology, repository boundaries, and SPI contracts.
- **`adrs/`** — Architectural Decision Records for ecosystem-wide decisions.

See [ADR 0001](./adrs/0001-hub-and-spoke-ai-context.md) for the rationale behind this architecture.

## Satellite Repositories

| Repository | Status | Description |
|---|---|---|
| [wildfly-core](https://github.com/wildfly/wildfly-core) | Indexed | Core server management model, controller, CLI, and kernel runtime |
| [wildfly](https://github.com/wildfly/wildfly) | Indexed | Full application server — Jakarta EE subsystems, clustering, Elytron, distribution packaging |
| [wildfly-glow](https://github.com/wildfly/wildfly-glow) | Indexed | Provisioning analysis engine — scans deployments to discover required Galleon feature packs and layers |
| [wildfly-maven-plugin](https://github.com/wildfly/wildfly-maven-plugin) | Indexed | Developer-facing build tooling for provisioning, packaging, deploying, and running WildFly servers from Maven |
| [wildfly-galleon-feature-packs](https://github.com/wildfly/wildfly-galleon-feature-packs) | Indexed | Central registry of Galleon feature packs for WildFly server provisioning |
| [wildfly-cloud-galleon-pack](https://github.com/wildfly/wildfly-cloud-galleon-pack) | Indexed | Cloud-optimized Galleon feature pack for OpenShift/Kubernetes environments |
| [wildfly-datasources-galleon-pack](https://github.com/wildfly/wildfly-datasources-galleon-pack) | Indexed | JDBC database driver and datasource Galleon layers for 8 databases |
| [wildfly-grpc-feature-pack](https://github.com/wildfly-extras/wildfly-grpc-feature-pack) | Indexed | gRPC subsystem support as a Galleon feature pack |
| [wildfly-myfaces-feature-pack](https://github.com/wildfly/wildfly-myfaces-feature-pack) | Indexed | Apache MyFaces (Jakarta Faces) alternative implementation feature pack |
| [wildfly-graphql-feature-pack](https://github.com/wildfly-extras/wildfly-graphql-feature-pack) | Indexed | MicroProfile GraphQL support via SmallRye GraphQL feature pack |
| [wildfly-vault-feature-pack](https://github.com/wildfly-security/wildfly-vault-feature-pack) | Indexed | HashiCorp Vault credential management integration feature pack |
| wildfly-operator | Planned | Kubernetes operator for managing WildFly deployments |

## Onboarding a New Repository

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

## Maintainer Guidelines

### Single source of truth for end-user docs

Never duplicate end-user documentation into `llms.txt` or architecture files. Admin guides, user manuals, and configuration references live in their canonical `.adoc` files in each repository. Use `llms.txt` entries to link directly to raw `.adoc` source files on GitHub rather than copying content.

### What belongs where

- **`llms.txt`**: One-paragraph summaries per repository and component. Enough for an agent to decide whether to follow the link, not enough to act on. Think "table of contents", not "chapter".
- **`architecture/`**: Cross-repository relationships that no single repository owns — SPI contracts, dependency rules, provisioning flows. Update these when repositories are added, components move between repositories, or boundary contracts change.
- **`.agents/deep-dives/`** (in satellite repos): Contributor-facing technical content that an agent needs to work effectively but that isn't covered by end-user documentation. Add a deep dive when the internal mechanics are complex enough that an agent would otherwise have to reverse-engineer them from source code. Examples: how a rule engine evaluates scan rules, the step-by-step lifecycle of a subsystem registration.

### Keep indexes current

When a satellite repository adds, renames, or removes components, the corresponding `llms.txt` entry and any references in `architecture/ecosystem-topology.md` or `architecture/repository-boundaries.md` should be updated. Stale routing information causes agents to target the wrong repository.
