# Experiment Charter 0005: Immutable Index Registry and Distribution

**Status:** Proposed

**Purpose:** Test whether generated source indexes can be published as immutable, exact-commit artifacts with a central registry/catalog, while keeping branch switches, tags, partial workspaces, local caching, and multiple assistant clients safe and practical.

This experiment evaluates the distribution and identity model proposed after Experiments 0001–0004. It does not select Git LFS, GitHub Pages, GitHub Releases, object storage, CBM, SCIP, MCP, or a final registry implementation.

## Objective

Determine whether a secondary repository containing manifests and immutable index references, with compressed index artifacts stored separately or through large-file storage, can provide a useful shared source of generated context for exact repository revisions.

The primary question is:

> Can an assistant resolve and consume an exact index for a repository commit or immutable tag without confusing it with a different branch, source state, generator version, or incomplete artifact?

The secondary questions are:

- What compression and download costs do representative indexes have?
- Should artifacts be monolithic or split into independently downloadable components?
- Can branch pointers remain convenient without becoming identity keys?
- Can local caches reuse exact commit artifacts and evict old entries safely?
- What metadata is required for partial workspaces and dependency indexes?
- Which distribution form is appropriate: normal Git, Git LFS, release assets, or object storage?

## Scope and non-goals

Use existing generated artifacts from Experiments 0002/0003 where they are still available. If they are unavailable, generate only the smallest representative CBM root artifact needed for packaging measurements; do not repeat the full source-navigation comparison.

The test registry must be local or disposable. Do not create, push to, or modify a real GitHub repository, release, Pages site, LFS store, or external object store.

Do not:

- implement a production index generator;
- decide that the registry is the source of truth for source code;
- make generated indexes mandatory for satellite repositories;
- automatically clone missing source repositories;
- expose incomplete indexes as complete;
- benchmark assistant task savings again except for a minimal query/download smoke test;
- use branch names as immutable index identity;
- place multi-hundred-megabyte generated artifacts directly into normal Git history.

## Input artifacts

Prefer these existing artifacts, if present:

- one `wildfly-core` CBM root index;
- one `wildfly` CBM root index;
- one feature-pack or dependency-related artifact from Experiment 0004;
- optionally one SCIP artifact as an opaque packaging comparison, without making it task-usable.

Record for each input:

- original repository, canonical root, ref, commit, and dirty state;
- generator/tool and schema versions;
- source/build/dependency identity;
- original format and size;
- coverage, exclusions, parse-partial files, and warnings;
- whether it represents source, binary/API, feature-pack configuration, or another evidence class.

If an artifact was generated from a dirty checkout, it must not be published under a clean commit or tag identity.

## Registry model

Build a disposable local registry with this conceptual layout:

```text
registry/
  <org>/
    <repo>/
      commits/<commit-sha>/manifest.json
      tags/<tag>/manifest.json
      branches/<branch>/pointer.json
```

The tag and branch entries may point to the commit manifest rather than duplicate the artifact. A branch pointer is mutable convenience metadata. A commit manifest and its artifact reference are immutable.

The registry must support repository identities that are not limited to the WildFly runtime repositories. The `<org>` path is a routing namespace, not proof of ownership or source provenance.

## Manifest requirements

Each commit manifest must include or reference:

- canonical repository URL and stable repository identity;
- organization/component classification;
- exact commit SHA and ref/tag information where applicable;
- dirty/clean status and source-state digest;
- generator name/version and index schema version;
- index format and compression format;
- compressed and uncompressed sizes;
- checksums for compressed and, where practical, uncompressed content;
- source roots, included/excluded paths, parse-partial files, and coverage;
- build toolchain, dependency coordinates, and SNAPSHOT metadata where relevant;
- source-to-artifact mapping or an explicit statement that no source commit attestation exists;
- artifact location and access requirements;
- generation status: complete, partial, failed, or unavailable;
- creation time, verification time, owner, and trust/status metadata.

An artifact version or checksum must not be labelled as an exact source implementation revision unless an authoritative source-to-artifact mapping exists.

## Packaging and compression comparison

Package each representative artifact using at least two practical forms, subject to tool availability:

- ZIP;
- Zstandard-compressed data, such as `.zst` or `tar.zst`;
- the original uncompressed form as the control.

Measure:

- compressed size and ratio;
- compression and decompression time;
- checksum calculation time;
- first-query readiness time after download;
- memory and temporary disk requirements;
- whether the format supports useful streaming or chunking;
- whether a partial query can avoid downloading the complete artifact.

Do not assume that compressing a SQLite or graph database makes it incrementally queryable. Record whether the full artifact must be materialized locally.

## Distribution simulations

Simulate the following without external writes:

1. local file-based registry lookup;
2. local HTTP serving of manifests and artifacts;
3. shallow retrieval of only a manifest followed by exact artifact download;
4. an unavailable artifact;
5. a checksum mismatch or truncated download;
6. a branch pointer advancing to a new commit;
7. an immutable tag pointing to one commit;
8. a generator/schema version mismatch;
9. a partial or failed generation entry.

If Git LFS is installed, a local pointer-file demonstration may be recorded, but it must not be treated as evidence of GitHub-hosted operational behavior. Use documented GitHub limits and billing constraints as external evidence rather than modifying a real remote repository.

## Cache behavior

Implement only a disposable test cache or a scripted cache simulation. Test:

- cache key based on repository identity, commit, generator, schema, configuration, and artifact checksum;
- tag and branch alias resolution to immutable commit entries;
- reuse after switching back to a previously indexed commit;
- refusal to reuse an artifact with a mismatched manifest or checksum;
- manifest-only lookup before artifact download;
- LRU or size-based eviction;
- tag pinning or retention policy;
- concurrent requests for the same artifact;
- interrupted download and recovery;
- cache permissions and separation between repositories/users where applicable.

The cache must never use only repository name, local path basename, branch name, or nominal version as its identity key.

## Partial-workspace and dependency behavior

Use the feature-pack evidence from Experiment 0004 as a lookup scenario:

- the feature-pack source is present;
- `wildfly` and `wildfly-core` source are absent;
- dependency coordinates are available;
- an exact dependency index may or may not exist;
- a source JAR or binary may be available separately.

Verify that the registry can return separate evidence classes and does not merge an older released index into current feature-pack source without reporting the identity difference.

## Security and operational observations

Record:

- artifact checksum/signature verification behavior;
- whether downloads are bounded by expected size and manifest data;
- permissions for manifests, artifacts, and local cache;
- behavior with malformed archives or untrusted index contents;
- whether consuming an index executes repository build logic;
- source paths, repository metadata, and private dependency information exposed by manifests;
- cache poisoning and concurrent-writer risks;
- retention, deletion, schema migration, and failed-generation recovery;
- offline use after manifests and artifacts are already cached.

## Decision rules

The registry model is viable for a prototype only if it demonstrates:

1. exact commit/tag lookup without branch or basename collisions;
2. immutable historical entries and mutable branch pointers with clear semantics;
3. manifest-first discovery and checksum-verified artifact retrieval;
4. explicit complete/partial/failed/unavailable status;
5. practical compression and download costs for representative artifacts;
6. safe cache reuse and eviction;
7. no mixing of incompatible source, artifact, or generator identities;
8. useful behavior for a partial feature-pack workspace;
9. an acceptable distribution and offline story without requiring every satellite repository to change.

Possible conclusions include:

- a secondary manifest repository plus release/LFS/object artifacts is viable;
- manifests should be in Git but large artifacts should use a separate store;
- the artifacts are too large and should be split or reduced;
- normal Git is adequate for compact indexes but not large graph databases;
- the registry is useful for exact released tags but not branches or SNAPSHOTs;
- the registry model is operationally too costly and search-first remains sufficient.

## Deliverable

Write:

`architecture/experiments/results/0005-immutable-index-registry-findings.md`

Use these sections:

```text
Hypothesis
Input artifacts and identities
Registry layout and manifest
Packaging and compression results
Distribution simulations
Cache behavior
Partial-workspace behavior
Security and operational observations
Failures and limitations
Evidence confidence
Decision matrix
Architectural consequence
Recommended next step or stop condition
```

Keep generated artifacts, compressed packages, local registry, cache, and raw evidence outside this repository. Do not modify external GitHub state.

## Handoff

The experiment agent should:

1. read this charter, the 0001–0004 charters and findings, the architecture options draft, and repository instructions;
2. locate existing 0002/0003 artifacts before generating anything new;
3. use a disposable local registry and local HTTP server only;
4. preserve exact source/index identity and evidence-class boundaries;
5. measure packaging and cache behavior rather than repeating assistant navigation trials;
6. write only the 0005 findings document;
7. stop before creating external repositories, releases, Pages sites, LFS objects, or production implementation files.
