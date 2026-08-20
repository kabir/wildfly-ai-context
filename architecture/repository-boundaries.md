# Repository Boundaries and SPI Contracts

This document defines what belongs in each WildFly ecosystem repository, the SPI contracts between them, and the versioning discipline that keeps repositories independently releasable.

## Ownership Rules

### What belongs in `wildfly-core`

A component belongs in wildfly-core if the base server cannot boot, accept management operations, or process deployments without it. Specifically:

- **Controller and management kernel**: Operation handler chain, resource registration, capability/requirement system, management model persistence.
- **Boot-critical subsystems**: Logging (for server boot output), IO (XNIO worker pools), Remoting (management transport), Discovery, Core Security Realms (legacy), Elytron (core authentication/authorization), JMX.
- **Deployment infrastructure**: Deployment overlay support, deployment scanner, the deployment unit processor chain SPI.
- **CLI client**: The `jboss-cli` tool, its command framework, and Aesh-based terminal integration.
- **Domain mode kernel**: Host controller, domain controller, server group management, process controller, two-phase operation coordination.
- **Patching infrastructure**: The server patching mechanism for applying and rolling back patches.

**Test**: If removing the component prevents the server from starting in standalone mode with an empty configuration (no subsystems beyond core), it belongs in wildfly-core.

### What belongs in `wildfly`

A component belongs in the wildfly repository if it implements a Jakarta EE specification, provides enterprise integration, or is part of the full distribution packaging:

- **Jakarta EE subsystems**: Undertow (Servlet), EJB, JPA (Hibernate), CDI (Weld), JAX-RS (RESTEasy), JMS (ActiveMQ Artemis), Bean Validation, Batch (JBeret), Concurrency, JSON-P/JSON-B, Mail, MicroProfile specifications.
- **Clustering**: Infinispan cache containers, JGroups stacks, mod_cluster, singleton services, distributable web session management.
- **Elytron integration subsystems**: Application security domains, HTTP authentication factory configurations that depend on Jakarta EE (e.g., FORM/BASIC auth for web applications, EJB security domain mappings).
- **Distribution feature packs**: `wildfly-ee-galleon-pack`, `wildfly-galleon-pack`, `wildfly-preview-feature-pack`, layer definitions, BOMs.
- **Full test suite**: Integration tests, clustering tests, domain-mode tests, MicroProfile TCKs, basic and preview distribution smoke tests.

**Test**: If the component implements something defined by a Jakarta EE or MicroProfile specification, or if it only makes sense in the context of application workloads (not bare management), it belongs in wildfly.

### What belongs in provisioning tools (`wildfly-glow`, `wildfly-maven-plugin`)

These repositories contain build-time and analysis tooling. They never run inside the server process:

- **WildFly Glow**: Archive scanning rules, layer mapping databases, provisioning configuration generators.
- **WildFly Maven Plugin**: Maven Mojo implementations, Galleon API integration, dev-mode server lifecycle management, CLI script runner.
- These tools depend on Galleon APIs and WildFly feature pack metadata, but do not depend on wildfly-core or wildfly runtime classes.

## SPI Contracts Between Repositories

### wildfly-core → wildfly

The `wildfly` repository depends on `wildfly-core` and extends it. The contracts flow in one direction:

- **Extension SPI**: `wildfly-core` defines the `org.jboss.as.controller.Extension` interface. Subsystems in `wildfly` implement this interface to register their management resources, parsers, and transformers. This SPI is stable and versioned with the controller module.
- **Capability and Requirement System**: `wildfly-core` provides the capability registry. Subsystems in `wildfly` declare capabilities they provide and requirements they have on other capabilities. The controller resolves these at boot and reports unsatisfied requirements as boot errors.
- **Deployment Processor Chain**: `wildfly-core` defines the `DeploymentUnitProcessor` interface and the phase-ordered processing chain. Subsystems in `wildfly` contribute processors at specific phases to handle annotation scanning, descriptor parsing, component installation, and so on.
- **Management Resource SPI**: `ResourceDefinition`, `OperationStepHandler`, `AttributeDefinition`, and transformer interfaces defined in wildfly-core are implemented by every subsystem in wildfly.

**Rule**: `wildfly-core` must never import or reference any class from `wildfly`. If a feature requires coordination between core and full subsystems, it must be mediated through the capability system or a new SPI defined in core.

### wildfly → Galleon Feature Packs

- WildFly defines Galleon feature packs (in `galleon-pack/` directories) that describe how to assemble a server from its modules, configurations, and scripts.
- Feature packs reference Galleon layers. Each layer declares the subsystem configurations it includes and the other layers it depends on.
- Feature pack metadata is consumed by the Maven plugin and Glow at build time.

### Glow → Feature Pack Metadata

- WildFly Glow reads feature pack metadata (layer descriptions, dependencies, configuration elements) to map application-level discoveries to provisioning layers.
- Glow ships its own rules database that maps annotations, descriptors, and API class references to layers. This database must be updated when layers are added, renamed, or restructured in the wildfly feature packs.

### Maven Plugin → Galleon API

- The WildFly Maven Plugin delegates provisioning to the Galleon API. It passes feature pack coordinates, layer names, and configuration options.
- The plugin also invokes the JBoss CLI (from wildfly-core) for post-provisioning script execution, using the embedded CLI API in admin-only mode.

## Schema Versioning Standards

WildFly subsystem configurations are defined by XML schemas. Maintaining schema compatibility is critical for configuration migration and domain-mode interoperability.

### Schema Naming Convention

Each subsystem schema follows the pattern:

```
wildfly-{subsystem-name}_{major}_{minor}.xsd
```

For wildfly-core subsystems, the historical pattern uses `jboss-as-` or `wildfly-` prefixes depending on when the subsystem was introduced.

### Schema Evolution Rules

1. **New minor version for additive changes**: Adding new optional attributes, new optional child elements, or new operations with optional parameters increments the minor version. Existing configurations must parse without modification under the new schema.

2. **New major version for breaking changes**: Removing attributes, changing attribute types, making previously optional elements required, or restructuring the element hierarchy requires a new major version. The subsystem parser must continue to support parsing all prior major versions and transforming them to the current internal model.

3. **Transformer support**: When a subsystem schema advances, the subsystem must register management model transformers for each prior version it needs to interoperate with in domain mode. Transformers convert operations and resource descriptions between model versions so that a newer domain controller can manage older host controllers and vice versa.

4. **Namespace registration**: Each schema version has a corresponding XML namespace. The subsystem parser registers all supported namespaces and routes to the appropriate parsing logic based on the namespace encountered in the configuration file.

### Domain-Mode Compatibility

Domain mode requires that the domain controller and host controllers can operate across a version skew of at most one major WildFly release:

- **Forward compatibility**: A newer host controller joining an older domain controller must transform its management model down to the version the domain controller understands. This is handled by transformers registered by subsystems.
- **Backward compatibility**: An older host controller managed by a newer domain controller must receive operations and resource descriptions in its own version. The domain controller applies transformers before sending operations to the older host.
- **Mixed-version domains**: Server groups can contain servers of different WildFly versions. The domain controller must track the minimum version in each server group and apply the appropriate transformers when pushing configuration or operations.

## Dependency Direction Enforcement

The dependency graph is strictly acyclic:

```
wildfly-maven-plugin ──→ Galleon API
                         ──→ Feature Pack metadata (from wildfly)
wildfly-glow ──→ Galleon API
                 ──→ Feature Pack metadata (from wildfly)
wildfly ──→ wildfly-core
wildfly-core ──→ (JBoss Modules, JBoss MSC, JBoss Remoting, XNIO)
```

**Violations to watch for**:

- A wildfly-core module importing a class from a wildfly subsystem module.
- A subsystem in wildfly directly instantiating a deployment processor from another subsystem instead of going through the capability system.
- WildFly Glow or the Maven plugin importing runtime server classes instead of using metadata and Galleon APIs.
- Circular layer dependencies in feature pack definitions.

These violations should be caught in code review and, where possible, enforced by module dependency declarations and build-time checks.
