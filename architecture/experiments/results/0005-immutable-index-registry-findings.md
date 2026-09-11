# Experiment 0005 Findings: Immutable Index Registry and Distribution

## Hypothesis

A manifest-first local registry can safely distribute generated indexes when immutable identity is the repository plus exact commit, generator/schema/configuration, and artifact checksum. Branches can remain mutable aliases and tags can remain immutable aliases, provided neither is used as the artifact identity.

## Input artifacts and identities

The previously disposable Experiment 0002/0003 artifacts were located under `/private/tmp`; no new source checkout or CBM generation was required. The registry and cache were created under `/private/tmp/wildfly-exp0005-k4cEIB/` and served only by a local HTTP server.

| Input | Identity and evidence class | Uncompressed size | SHA-256 |
|---|---|---:|---|
| CBM `wildfly-core.db` | `wildfly/wildfly-core`, `ai-index`, commit `d511a0d1e5c34305f8a6d2d8f917fab1180c65b8`; clean checkout recorded by Experiment 0003; CBM 0.10.8 root index | 400,556,032 B | `6e19cddc78e7b54eab2085a0a7c51974e8d5176b1ed970159fb1739797af72c` |
| CBM `wildfly.db` | `wildfly/wildfly`, `ai-index`, commit `39e7ce3cb5a23f354e3825bece6d4ccb15c1ab22`; clean checkout recorded by Experiment 0003; CBM 0.10.8 root index | 779,288,576 B (743 MiB on disk) | `f6877d5a20122754fa9c8822f6c8d66efd5df37b5edc22ee03cfb74f561f32a0` |
| SCIP `wildfly-core.scip` | `wildfly-core` commit identity recorded by Experiment 0002; opaque SCIP artifact, not used for task lookup | 184,801,762 B | `c542cbe023e484e0606474bb10cd23cb9e4bf87bba72082be9cc66776a74eaf7` |
| Feature-pack evidence | `wildfly/wildfly-galleon-feature-packs`, commit `ba872664f8f97f79c78a47e28afa4b4f3a7e5030`; Experiment 0004 feature-pack/declaration evidence | no artifact copied | no checksum available in this registry |

The full CBM artifact was measured at 779,288,576 bytes (743 MiB) in the located prior cache but was not copied into a new package. The CBM indexes contained 115,043 nodes/638,213 edges for core and 180,047 nodes/773,017 edges for full. Core had 11 parse-partial files; full had 10 and deliberately excluded 157 files/eight directories. No artifact was relabelled as an authoritative source implementation revision.

## Registry layout and manifest

The disposable registry used the charter layout:

```text
registry/wildfly-core/commits/<commit>/manifest.json
registry/wildfly-core/tags/ai-index/manifest.json
registry/wildfly-core/branches/ai-index/pointer.json
registry/wildfly-galleon-feature-packs/commits/<commit>/manifest.json
```

The CBM manifest recorded repository identity and URL, organization/component class, commit/ref, clean state, source-state identity note, CBM version, index schema/configuration, evidence class, coverage/exclusions/parse-partial files, build/dependency metadata, compressed and uncompressed sizes, both checksums, artifact location/access, lifecycle status, timestamps, owner, and checksum-only trust. The feature-pack entry was a separate `feature-pack-artifact` evidence class with no runtime artifact location. Failed and unavailable controls were represented explicitly rather than as empty successful entries.

The exact commit entry was immutable in the test. The `ai-index` branch pointer was advanced to a different target and the tag target remained unchanged. The branch then required a new commit manifest; it could not safely resolve by repository basename or branch name alone.

## Packaging and compression results

The real core CBM database was the primary packaging representative. The prior SCIP file was an opaque comparison.

| Artifact | Form | Size | Ratio | Compression time | Decompression/materialization |
|---|---|---:|---:|---:|---:|
| CBM core | original SQLite | 400,556,032 B | 100% | control | already queryable |
| CBM core | ZIP `-9` | 52,299,620 B | 13.1% | 10.36 s | 1.13 s |
| CBM core | Zstandard `-19` | 32,685,537 B | 8.16% | 20.58 s | 0.19 s |
| SCIP core | original | 184,801,762 B | 100% | control | opaque artifact |
| SCIP core | ZIP `-9` | 19,665,166 B | 10.6% | 1.60 s | not separately timed |
| SCIP core | Zstandard `-19` | 12,497,472 B | 6.76% | 1.60 s | not separately timed |

SHA-256 for the CBM compressed outputs was `34a4267ca83ddf2a0ae99f33d790fcc3fefada94dc3e4803fcf180037291449f` (Zstandard) and `2ac032d54af6a82d6af4b67234d037019a73433b4112ada2a597312c10fa167c` (ZIP). For SCIP Zstandard and ZIP it was `3435156fdb860a0748f40f548f44eccd5853cf8963667586fb6e39f71606b01e` and `dc88deac20e9da8102dad506c2550682339a18af210fbf36d72e2ee31b5833f9`.

The compressed SQLite database still required full materialization before normal SQLite queries: both decompressed copies passed `PRAGMA integrity_check` and returned 77,114 nodes, but a compressed stream was not independently queryable. ZIP and Zstandard therefore reduce transfer/storage cost, not random-access download cost. Splitting CBM tables or using a query-native chunk format would be a separate design, not a consequence of compression alone.

## Distribution simulations

- File lookup succeeded for the exact commit manifest and artifact.
- A bounded localhost HTTP server returned the manifest first: HTTP 200, 2,024 bytes, 8.8 ms; the artifact then returned HTTP 200, 32,685,537 bytes, 28.2 ms on the local server. The downloaded checksum matched the manifest.
- A missing artifact returned HTTP 404. A truncated artifact returned HTTP 200 but its checksum differed, so the consumer rejected it after transfer; expected-size validation would allow earlier rejection.
- The branch pointer advanced while the immutable tag continued to target the original commit.
- `complete`, `failed`, and `unavailable` entries were distinguishable. A failed/unavailable entry did not fall back to a different commit.
- A generator/schema mismatch was rejected conceptually by the cache key and manifest comparison; the key includes repository identity, commit, generator, schema, configuration, and compressed checksum.
- No partial HTTP range/query optimization was attempted: the SQLite consumer requires the complete decompressed file.

## Cache behavior

The disposable cache keyed entries by repository identity, commit, generator, schema, configuration, and compressed checksum. Re-reading the exact core artifact was a hit. A 100-byte truncation was rejected. Switching back to the original commit can reuse the verified entry; the changed branch alias cannot overwrite that entry.

The scripted cache checks also showed:

- an LRU size limit evicted unpinned entries while retaining a pinned tag entry;
- eight concurrent readers of one verified key all observed the same checksum-valid content;
- permissions were set to `0600` for a local cache entry;
- an already verified entry remained usable offline;
- a cache key does not depend on local path basename, nominal version, or branch name alone.

The interrupted-download case was represented by a truncated file and recovery-by-reject; a production downloader would need an atomic temporary name and retry/resume policy before replacing the verified cache entry. Concurrent writers need the same atomic publish rule; the read-only concurrent test does not prove a multi-writer implementation.

## Partial-workspace behavior

The feature-pack manifest remained separate from the CBM runtime indexes. It identified feature-pack source/declaration evidence at commit `ba872664f8f97f79c78a47e28afa4b4f3a7e5030`, with the Experiment 0004 release coordinates, but had no runtime artifact location. The registry therefore reported `feature-pack-artifact` evidence and unavailable runtime implementation context rather than merging an older `wildfly` or `wildfly-core` index.

This preserves the Experiment 0004 contract: feature-pack layer/package declarations, binary/source-JAR evidence, and runtime source indexes are different evidence classes. An exact dependency index, when available, must carry its own artifact/version/checksum identity; a released index must not silently answer for current feature-pack source or a SNAPSHOT.

## Security and operational observations

Checksums verified both compressed downloads and decompressed SQLite integrity. The registry used no signatures, so checksum-only trust does not protect against a compromised registry that changes both manifest and artifact. A signed manifest or trusted transport would be required for a shared service.

Manifest-declared size should be checked before allocation and after download. Archives/index contents must be treated as untrusted data; the test only decompressed and queried SQLite and did not execute repository build logic. The local registry exposed repository paths, commit identity, parser coverage, and dependency metadata, which can reveal more workspace information than a short index pointer. Cache permissions and per-user separation therefore matter.

Normal Git is suitable for compact manifests and pointers, but the 400 MB uncompressed/33 MB compressed CBM artifact and the 743 MiB full CBM database are poor normal-Git-history candidates. This run provides no evidence selecting LFS, release assets, Pages, or object storage; it only shows that the artifact layer should be separate from the manifest layer. Retention must protect immutable tags, while failed generations should remain visible as failed or be tombstoned without reusing a prior artifact.

## Failures and limitations

- No prior CBM artifacts were initially visible in the repository, but the preserved `/private/tmp` Experiment 0002/0003 directories contained the real databases, so no new representative generation was needed.
- The full CBM database was not recompressed in this run; its size/checksum and index counts are carried from the located prior artifact and findings.
- The HTTP server could only be run with localhost-only elevated execution because the sandbox denied socket binding. No external host, GitHub state, release, Pages site, LFS store, or object store was used.
- HTTP timing is local-server timing, not network download performance.
- ZIP versus Zstandard measurements use one CBM SQLite database and one SCIP file; they are not a broad corpus benchmark.
- No digital signature, resumable range protocol, multi-writer lock, or production eviction implementation was tested.
- The concurrent test used verified readers, not simultaneous competing writers.
- No assistant model calls, token savings, or task-time savings were measured, as prohibited by the charter.

## Evidence confidence

High confidence: located artifact identities, checksums, SQLite integrity, package sizes/timings, manifest-first HTTP behavior, exact commit/tag distinction, branch-pointer mutation, checksum rejection, explicit lifecycle statuses, cache-key behavior, and separation of feature-pack evidence.

Moderate confidence: cache concurrency and LRU conclusions, because they were scripted behavioral simulations rather than a production cache implementation; and distribution cost conclusions, because the server was local.

Low confidence/unavailable: signature/trust guarantees, multi-writer recovery, WAN performance, and whether a split/chunked index would materially improve query cost. Those require a separate implementation and service evaluation.

## Decision matrix

| Question | Result | Finding |
|---|---|---|
| Exact commit lookup | Pass | Commit identity plus checksum resolves without basename/branch collision |
| Immutable tag and mutable branch | Pass | Tag stayed pinned while branch advanced; branch is an alias only |
| Manifest-first retrieval | Pass | Small manifest lookup preceded exact artifact download |
| Checksum/failure handling | Pass | Truncated, missing, failed, and unavailable states were not accepted as complete |
| Compression cost | Pass for transfer reduction | Zstandard was 8.16% of core SQLite size versus ZIP at 13.1%, but both require full materialization |
| Cache reuse/eviction | Pass in disposable simulation | Composite identity, checksum validation, LRU, pinning, permissions, offline reuse, and same-key concurrency behaved safely |
| Partial workspace | Pass | Feature-pack declarations remained separate from unavailable runtime implementation indexes |
| Normal Git artifact storage | Fail for CBM databases | Multi-hundred-megabyte generated state should not be placed in normal Git history |
| Production distribution choice | Unresolved | This experiment does not select LFS, releases, Pages, or object storage |

## Architectural consequence

The registry model is viable as a prototype contract: manifests/pointers can be compact, immutable commit entries can be exact, and large generated artifacts can be distributed separately with checksum verification. The essential boundary is identity and provenance, not the particular artifact transport.

The minimum consumer contract is manifest-first resolution, exact composite cache keys, immutable commit entries, mutable alias semantics, expected-size/checksum validation, atomic cache publication, explicit complete/partial/failed/unavailable status, and ordinary-search/source fallback when an exact artifact is absent or stale. Compression should be treated as a storage/transfer optimization; it does not make a SQLite graph incrementally queryable.

## Recommended next step or stop condition

Stop before production implementation or external publication. If another bounded experiment is authorized, compare a signed manifest plus one selected artifact backend using a real downloader with atomic resumable recovery and multi-writer locking. Keep manifests in normal Git-sized storage and keep large CBM/SCIP artifacts outside normal Git history. Do not make a branch pointer, artifact version, or dependency coordinate stand in for an exact source commit.
