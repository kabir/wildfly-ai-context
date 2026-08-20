# WildFly Ecosystem Topology

This document describes the component hierarchy, dependency relationships, and execution flows across the WildFly ecosystem's repositories.

## Component Hierarchy

The WildFly ecosystem is organized in a layered architecture where each layer builds on the one below it:

```
┌──────────────────────────────────────────────────────────────────┐
│                       Developer Tooling                          │
│     WildFly Maven Plugin · WildFly Glow · WildFly Operator       │
├────────────────────────────┬─────────────────────────────────────┤
│   Feature Pack Registry    │     Feature Pack Extensions          │
│   (wildfly-galleon-        │  Cloud · Datasources · gRPC         │
│    feature-packs)          │  MyFaces · GraphQL · Vault           │
├────────────────────────────┴─────────────────────────────────────┤
│                WildFly (Full Distribution)                        │
│    Jakarta EE Subsystems · Clustering · Elytron · BOMs           │
├──────────────────────────────────────────────────────────────────┤
│                      WildFly Core                                │
│    Controller · Management Model · CLI · Remoting · Boot         │
├──────────────────────────────────────────────────────────────────┤
│                   Galleon / Feature Packs                         │
│    Provisioning Engine · Layers · Feature Specs                   │
└──────────────────────────────────────────────────────────────────┘
```

### WildFly Core (Base Runtime and Management)

WildFly Core is the kernel of the application server. It provides:

- **Server Controller**: The central management kernel that owns the management resource tree, processes management operations, and orchestrates the service container lifecycle. All management operations — read-resource, write-attribute, add, remove — flow through the controller's operation step handler chain.
- **Management Model**: A tree-structured resource registry where each node defines attributes, operations, and child resources. Extensions register subsystem root resources into this tree at boot time. The model is the single source of truth for server configuration and runtime state.
- **CLI (JBoss CLI)**: A command-line management client that communicates with the controller over the Remoting-based native management interface. Supports scripting, batch operations, and tab-completion against the live management model.
- **Deployment Scanner**: Monitors a directory for application archives and triggers deployment through the controller. In domain mode, deployments are pushed from the domain controller to host controllers and then to individual servers.
- **Remoting**: Provides the transport layer for the native management interface and EJB-to-EJB invocation. Based on JBoss Remoting with SASL authentication.
- **Domain Mode Infrastructure**: Host controller, domain controller, server groups, and the two-phase domain-wide operation coordination protocol.

### WildFly (Full Distribution)

The `wildfly` repository extends `wildfly-core` with the full Jakarta EE platform:

- **Jakarta EE Subsystems**: Each specification (Servlet/Undertow, EJB, JPA/Hibernate, CDI/Weld, JAX-RS/RESTEasy, JMS/ActiveMQ Artemis, Batch, Concurrency, Mail, MicroProfile) is implemented as one or more subsystems that register into the management model at boot.
- **Elytron Security Integration**: Security domains, SASL/HTTP authentication factories, credential stores, and TLS/SSL context configuration. Elytron replaces the legacy PicketBox security subsystem.
- **Clustering and High Availability**: Infinispan-based distributed caching for HTTP sessions and stateful EJBs, JGroups-based cluster communication, mod_cluster load-balancer integration, and singleton services.
- **Distribution Packaging**: Galleon feature packs that compose wildfly-core and wildfly subsystems into installable server distributions. Bill of Materials (BOMs) for dependency management in application projects.
- **Test Suite**: Integration tests exercising the full server, including multi-node clustering tests, domain-mode tests, and compatibility tests.

### WildFly Glow (Provisioning Analysis)

WildFly Glow (`org.wildfly.glow`) is the provisioning analysis engine. It is both a standalone tool (with its own CLI) and a library consumed by the WildFly Maven Plugin. Its modules are organized into three areas:

- **Core Scanning Engine** (`core/`): `GlowSession` orchestrates the end-to-end scan of application archives (WAR, JAR, EAR). `ScanArguments` configures scan parameters (execution context, profile, add-ons, version, server variant). `ScanResults` holds discovered layers and feature packs. `LayerMapping` and `LayerMetadata` implement the rule-matching system that maps deployment content (Java API types, XML descriptors, properties files, annotations) to Galleon layers. `ProvisioningUtils` generates Galleon provisioning configuration from scan results. `DockerSupport` generates container images.
- **CLI and Integration** (`cli/`, `cli-support/`): Picocli-based command-line interface with `ScanCommand` (deployment scanning), `ShowAddOnsCommand`, `ShowServerVersionsCommand`, `ShowConfigurationCommand`, and `GoOfflineCommand`. The `cli-support/` module provides shared infrastructure (`AbstractCommand`, `CLIConfigurationResolver`). JBang integration allows running single Java files in WildFly with Glow-based provisioning.
- **Build and Test Tooling** (`arquillian-plugin/`, `doc-plugin/`, `maven-resolver/`): The Arquillian Maven plugin (`ScanMojo`) scans `@Deployment` methods in test classes to generate `provisioning.xml` for test server provisioning. The doc plugin generates documentation. The maven-resolver module handles Galleon feature pack resolution from Maven repositories.
- **OpenShift Deployment** (`openshift-deployment/`): Automatic service deployers for PostgreSQL, MySQL, MariaDB, AMQ Broker, and Keycloak. When deploying to OpenShift via the CLI, Glow detects the need for these services and deploys them alongside the application, binding them automatically.
- **Add-on Discovery** (`Suggestions`): Based on discovered layers, Glow suggests optional server features (e.g., SSL, OpenAPI, database drivers) that the developer can enable. Add-ons are grouped by family and context (bare-metal vs cloud).

### WildFly Maven Plugin (Developer Tooling)

The Maven plugin (`org.wildfly.plugins:wildfly-maven-plugin`) is the primary developer interface for building and running WildFly-based applications. It depends on WildFly Glow as a library for deployment scanning and on Galleon for server provisioning. It provides 15 Maven goals organized into four areas:

- **Provisioning and Packaging** (`provision`, `package`, `image`): Invokes Galleon to provision a trimmed server based on declared feature packs and layers, or delegates to WildFly Glow for automatic discovery via `<discover-provisioning-info/>`. The `package` goal can produce a directory-based server installation or a Bootable JAR (an executable fat JAR containing server + application). The `image` goal builds and optionally pushes Docker/Podman container images.
- **Server Lifecycle** (`dev`, `run`, `start`, `start-jar`, `shutdown`): The `dev` goal provides hot-reload development mode — it watches source directories, recompiles on change, and redeploys. The `run` goal starts the server with the application deployed. The `start`/`start-jar` goals launch the server as a background process.
- **Deployment Operations** (`deploy`, `deploy-only`, `deploy-artifact`, `redeploy`, `redeploy-only`, `undeploy`): Deploy, redeploy, and undeploy applications and artifacts to a running WildFly server via the management API.
- **CLI and Resource Management** (`execute-commands`, `add-resource`): Executes JBoss CLI commands against a running or embedded server. Packaging scripts run CLI commands in an embedded server during build to fine-tune configuration beyond what layers provide.
- **Channel Integration**: Resolves feature pack versions through WildFly Channels (YAML-based version manifests), providing controlled update streams for reproducible provisioning.

### WildFly Operator (Kubernetes Deployment)

The WildFly Operator manages WildFly application lifecycle on Kubernetes and OpenShift:

- Creates and manages StatefulSets for WildFly application pods.
- Coordinates graceful shutdown with session draining and transaction recovery.
- Integrates with OpenShift build infrastructure for S2I (Source-to-Image) workflows.
- Manages server configuration through WildFlyServer custom resources.

### WildFly Galleon Feature Packs (Feature Pack Registry)

The `wildfly-galleon-feature-packs` repository (`org.wildfly.galleon.feature-packs`) is the central registry of Galleon feature packs that can be provisioned with each WildFly server version. It is structured as a versioned catalog:

- **Per-Version Provisioning Files**: Each WildFly version (e.g., `41.0.0.Final/`) has `provisioning-bare-metal.xml` and `provisioning-cloud.xml` listing compatible feature packs with their Maven coordinates and versions. Tech-preview variants are in `tech-preview/` subdirectories; EE 10 variants are in `ee-10/` subdirectories (versions 40+).
- **JSON Catalogs**: The `catalog/` directory provides machine-readable JSON metadata per version (feature pack lists, variant definitions) consumed by tooling.
- **Spaces**: The `spaces/` directory defines feature-pack spaces (e.g., incubating) with their own version metadata, allowing feature packs to graduate from incubating to stable.
- **Glow Integration**: WildFly Glow reads this registry from the `release` branch to resolve feature pack coordinates for server provisioning. The registry is the authoritative source of which feature packs are compatible with each WildFly version.
- **Release Automation**: Shell scripts (`add-wildfly-release.sh`, `release-publish-maven-metadata.sh`) automate adding new WildFly releases and publishing Maven metadata.

### Feature Pack Extensions

The following Galleon feature packs extend WildFly with additional capabilities. Each defines one or more Galleon layers that can be provisioned alongside the WildFly base feature packs. They share a common pattern: a Galleon layer definition, JBoss Module descriptors, and typically a WildFly subsystem extension.

#### WildFly Cloud Galleon Pack

The cloud feature pack (`org.wildfly.cloud:wildfly-cloud-galleon-pack`) adjusts WildFly configuration for OpenShift and Kubernetes:

- **Cloud Configuration Layers**: The `cloud-default-config` layer and related adjustment layers configure JGroups for TCP-based clustering (replacing UDP multicast), route logs to console output, enable health probes, and tune subsystems for container environments.
- **Launch Scripts**: Shell scripts provisioned into the server's `bin/launch/` directory configure the server via environment variables at startup — datasource URLs/credentials, JGroups parameters, Elytron security, MicroProfile Config, messaging endpoints, and more.
- **Bootable JAR Configurator**: A runtime extension (`CloudConfigurator`) applies cloud-specific configuration at bootable JAR boot time, including node naming and JGroups `KUBE_PING` discovery protocol setup.
- **Preview Variant**: `wildfly-preview-cloud-galleon-pack` targets Jakarta EE 11+ preview servers.

#### WildFly Datasources Galleon Pack

The datasources feature pack (`org.wildfly:wildfly-datasources-galleon-pack`) provides database connectivity layers:

- **Three-Layer Pattern**: Each supported database gets three decorator layers — a driver layer (installs the JDBC driver module), a datasource layer (configures a datasource with environment-variable-driven connection properties), and a default-datasource layer (marks it as the Jakarta EE default datasource).
- **Supported Databases**: PostgreSQL, MySQL, MariaDB, Oracle, DB2, MS SQL Server, H2, and AWS Advanced JDBC Wrapper (8 total).
- **Environment Variable Configuration**: All datasource properties (URL, user, password, connection pool min/max, validation) are configurable via environment variables, making layers suitable for cloud deployments.
- **Driver Version Override**: JDBC driver versions can be overridden via system properties during provisioning.

#### WildFly gRPC Feature Pack

The gRPC feature pack (`org.wildfly.grpc:wildfly-grpc-feature-pack`) adds gRPC subsystem support:

- **Subsystem Extension**: The `org.wildfly.extension.grpc` subsystem manages a Netty-based gRPC server (default port 9555) with configurable flow control, keep-alive, message size limits, and TLS settings.
- **Auto-Discovery**: At deployment time, a `DeploymentUnitProcessor` scans applications using Jandex for `BindableService` and `ServerInterceptor` implementations and registers them with the gRPC server.
- **TLS Support**: Supports plaintext, one-way TLS, and mutual TLS configurations via key/trust store paths or Elytron SSL context references.
- **Per-Deployment Config**: Deployments can include `META-INF/grpc-deployment.xml` for deployment-specific gRPC configuration.
- **Galleon Layer**: The `grpc` layer depends on `cdi` and is registered as an RPC add-on for WildFly Glow discovery.

#### WildFly MyFaces Feature Pack

The MyFaces feature pack (`org.wildfly:wildfly-myfaces-feature-pack`) provides an alternative Jakarta Faces implementation:

- **Galleon Layer**: The `myfaces` layer depends on `cdi` and `jsf`, sets `default-jsf-impl-slot` to `myfaces`, and declares itself as an add-on alternative to the default JSF implementation.
- **Injection Bridge**: Java code in the `myfaces-injection` module bridges MyFaces managed beans to WildFly's CDI container, delegates annotation scanning to WildFly's deployment infrastructure, and bootstraps MyFaces via a `ServletContainerInitializer`.
- **JBoss Modules**: Module descriptors for the MyFaces API JAR, implementation JAR, and injection bridge in the `myfaces` slot of the JSF module path.

#### WildFly MicroProfile GraphQL Feature Pack

The GraphQL feature pack (`org.wildfly.extras.graphql:wildfly-microprofile-graphql-feature-pack`) adds MicroProfile GraphQL support:

- **Subsystem Extension**: The `microprofile-graphql-smallrye` subsystem integrates SmallRye GraphQL into WildFly, processing deployments to expose GraphQL endpoints and optionally enabling the GraphiQL developer UI.
- **Galleon Layer**: The `microprofile-graphql` layer depends on `cdi`, `microprofile-config`, and `microprofile-context-propagation`. It also requires the WildFly Reactive feature pack.
- **GraphQL Client**: Includes SmallRye GraphQL Client modules with both standard and Vert.x-based client implementations.
- **TCK Compliance**: Runs the MicroProfile GraphQL Technology Compatibility Kit for specification compliance.

#### WildFly Vault Feature Pack

The vault feature pack (`org.wildfly.security.vault:wildfly-vault-feature-pack`) integrates HashiCorp Vault:

- **Subsystem Extension**: The `hashicorp-vault` subsystem connects to HashiCorp Vault servers to retrieve credentials and secrets, with HTTPS and mutual TLS connectivity via Elytron authentication contexts.
- **Credential Store Integration**: Provides Vault-backed Elytron credential stores that make Vault secrets available to any WildFly resource supporting `credential-reference` attributes (datasources, messaging, etc.).
- **Expression Resolution**: The `VaultExpressionResolver` resolves `${HC_VAULT::store:alias}` expressions anywhere in WildFly configuration, allowing secrets to be referenced declaratively without embedding them.
- **Galleon Layer**: The `hashicorp-vault` layer depends on `elytron`, `management`, and `core-tools`, registered as a security add-on for WildFly Glow.

## Build and Provisioning Flow

The following describes the end-to-end flow from application source to a running, trimmed WildFly server:

```
Developer Source Code
        │
        ▼
┌──────────────────┐
│  Maven Build      │  Application compilation and packaging
│  (wildfly-maven-  │
│   plugin)         │
└────────┬─────────┘
         │
         ▼
┌──────────────────┐
│  WildFly Glow     │  Archive scanning → layer discovery
│  Analysis         │  (optional, can specify layers manually)
└────────┬─────────┘
         │  Required layers list
         ▼
┌──────────────────┐
│  Galleon          │  Feature pack resolution → trimmed
│  Provisioning     │  server assembly
└────────┬─────────┘
         │  Provisioned server
         ▼
┌──────────────────┐
│  CLI Script       │  Post-provisioning configuration
│  Execution        │  customization
└────────┬─────────┘
         │  Configured server + deployed application
         ▼
┌──────────────────┐
│  Runtime          │  WildFly Core boots, extensions load,
│  (WildFly Core +  │  subsystems activate, application
│   WildFly Full)   │  deploys
└──────────────────┘
```

### Detailed Provisioning Steps

1. **Layer Resolution**: The Maven plugin (or Galleon CLI) takes a list of Galleon layers — either declared explicitly by the developer or produced by WildFly Glow's analysis. Each layer maps to a set of subsystem configurations and their transitive dependencies.

2. **Feature Pack Assembly**: Galleon resolves the declared feature packs (wildfly-ee-galleon-pack, wildfly-galleon-pack, etc.) and filters them down to only the packages required by the selected layers.

3. **Server Directory Construction**: Galleon writes out the server directory structure: module JARs in `modules/`, configuration XML in `standalone/configuration/` or `domain/configuration/`, and launch scripts in `bin/`.

4. **Configuration Generation**: Each layer contributes XML fragments to the server configuration. Galleon merges these fragments into the final `standalone.xml` or `domain.xml`, respecting dependency ordering so that subsystems appear in the correct initialization sequence.

5. **Post-Provisioning Customization**: The Maven plugin executes any declared CLI scripts against the provisioned server in admin-only mode. This allows configuration changes that go beyond what layers express — for example, configuring specific datasource connection URLs, adjusting thread pool sizes, or adding deployment overlays.

6. **Packaging**: The final output is either a directory-based server installation, a bootable JAR (server + application in a single executable JAR), or a container image layer depending on the packaging goal used.

## Runtime Boot Sequence

When the server starts:

1. **Module Loading**: JBoss Modules initializes the modular classloader hierarchy from the `modules/` directory.
2. **Controller Boot**: The server controller reads the XML configuration and builds the initial management model.
3. **Extension Loading**: Each `<extension>` element in the configuration triggers the loading of a module and the registration of its subsystem parsers.
4. **Subsystem Parsing**: The controller invokes each subsystem's XML parser to read its configuration block and produce management resource operations.
5. **Service Installation**: The controller executes the parsed operations, which install services into the MSC (Modular Service Container). Services declare dependencies on other services, and MSC resolves the dependency graph to determine activation order.
6. **Deployment Processing**: Once all subsystems are active, the deployment scanner (or the management API) processes application deployments through the deployment unit processing chain — a series of deployment processors contributed by subsystems.
