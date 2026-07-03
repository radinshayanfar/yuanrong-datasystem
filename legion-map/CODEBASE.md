# Codebase Map

**Analyzed:** 2026-07-03
**Generated At:** 2026-07-03T15:36:15Z
**Map Schema Version:** 2.0
**Analyzed Commit:** 3f18c3d609f3e3859009a41b6ec29591ae8435a9
**Source File Count:** 1902
**Source Fingerprint:** d8d6722cd603f46a
**Source Fingerprint Kind:** hash (sha256 of sorted path|size over code + build files, excl .git/third_party)
**Scope:** project-root
**Root:** /home/runner/work/yuanrong-datasystem/yuanrong-datasystem/target
**Confidence:** HIGH

<!--
map_schema_version: "2.0"
generated_at: "2026-07-03T15:36:15Z"
analyzed_commit: "3f18c3d609f3e3859009a41b6ec29591ae8435a9"
source_file_count: 1902
source_fingerprint: "d8d6722cd603f46a"
source_fingerprint_kind: "hash"
scope: "project-root"
-->

## Architecture Overview

`yuanrong-datasystem` (openYuanrong datasystem, v0.8.2) is the **data-system component of the
openYuanrong serverless compute engine**: a high-performance, highly-available distributed
multi-tier cache built in **C++17**. It pools HBM/DRAM/SSD across a compute cluster to provide
near-compute caching with KV, Object, Stream, and heterogeneous (NPU HBM) object semantics —
targeting LLM KVCache, model-parameter reshuffle, checkpoint save/load, and microservice state.

The runtime is a three-role distributed system: a **multi-language SDK/client** (C++ core with
Python, Java, and Go bindings) that talks to a co-located **worker** process over shared memory;
**worker** processes that own DRAM/SSD allocation, cache data services (object/stream/kv), and
inter-worker data transfer over TCP/RDMA(URMA); and a **master/coordinator** layer plus
**cluster management** built on ETCD/Metastore for node discovery, health, hash-ring routing,
failover, and online scale up/down. RPC contracts are protobuf-defined and carried over a ZMQ/RPC
layer; bulk data moves over shared memory, RDMA/URMA, or (for NPU) HCCS/RoCE via the separate
`transfer_engine` subsystem.

The codebase is **layered by role** under `src/datasystem/{client,worker,master,server,common}`,
with `common/` holding the shared infrastructure (rdma, l2cache/slot secondary storage, log,
metrics, rpc/zmq, kvstore/etcd, shared_memory). It builds with **both Bazel and CMake** (dual build
graph, `build.sh` is the unified entrypoint) and exposes public headers under `include/datasystem`.
This is infrastructure-grade code: hot paths, concurrency safety, crash/recovery correctness, and
operational availability are first-class concerns — see `.repo_context/` for the maintained,
source-backed module guidance that this map complements.

## Language Distribution

| Extension | File Count | % of Codebase |
|-----------|-----------|---------------|
| .cpp | 785 | 26% |
| .h | 622 | 20% |
| .py | 455 | 15% |
| .json | 201 | 7% |
| .md | 200 | 7% |
| .rst | 168 | 6% |
| .bazel | 148 | 5% |
| .txt | 83 | 3% |
| .cmake | 69 | 2% |
| .yml | 64 | 2% |
| .js | 58 | 2% |
| .sh | 45 | 1% |
| .ts | 35 | 1% |
| .yaml | 28 | 1% |
| .java | 27 | 1% |
| .proto | 23 | 1% |
| .go | 12 | <1% |

Total tracked code files (.cpp/.cc/.h/.hpp/.c/.go/.java/.py, excl. `third_party/`): **1902**.
Primary language is C++17; Python is the dominant scripting/SDK/test language; Go and Java provide
thin binding surfaces. `.js`/`.ts`/`.cedar` files belong to `.github/` agentic tooling, not the runtime.

## Detected Stack

| Layer | Technology | Evidence |
|-------|-----------|----------|
| Language | C++17 | `.bazelrc`: `build --cxxopt=-std=c++17`; 785 `.cpp` + 622 `.h` |
| Build (primary) | Bazel | `WORKSPACE`, `BUILD.bazel`, 148 `*.bazel` files, `.bazelrc` |
| Build (secondary) | CMake | `CMakeLists.txt`, `cmake/`, per-module `CMakeLists.txt`; `build.sh -b cmake|bazel` |
| RPC / serialization | Protobuf + ZMQ | 20 `.proto` under `src/datasystem/protos`; `src/datasystem/common/rpc/zmq` |
| Transport | Shared memory, TCP, RDMA/URMA, HCCS/RoCE | `src/datasystem/common/{shared_memory,rdma}`, `transfer_engine` |
| Coordination | ETCD / Metastore | `src/datasystem/common/kvstore/etcd`, `third_party/protos/etcd` |
| Python SDK | pybind11 + CPython 3.9–3.11 | `src/datasystem/pybind_api`, `python/yr/datasystem`, `setup.py` |
| Java binding | JNI | `src/datasystem/java_api`, `java/pom.xml` |
| Go binding | cgo over C API | `src/datasystem/c_api`, `go/go.mod` (module `clients`, go 1.17) |
| Test | GoogleTest + CTest; pytest | `tests/ut`, `tests/st` (gtest), `tests/python`, CTest via `GoogleTestToCTest.cmake` |
| Lint/format | clang-format, clang-tidy | `.clang-format`, `.clang-tidy` |
| Deploy | Docker + Helm + CLI | `k8s/`, `k8s_deployment/`, `cli/` (start/stop/up/down, config gen) |
| Architecture | Layered by runtime role (client/worker/master/common) | `src/datasystem/` directory structure (inferred + doc-confirmed) |

## Conventions Detected

- **File naming**: `snake_case` for C++ and Python files (e.g. `worker_oc_service_impl.cpp`, `kv_client.py`).
- **Module structure**: layered by runtime role under `src/datasystem/`; shared infra centralized in `common/`; public API in `include/datasystem/`.
- **Config location**: gflags (`common/flags`, `worker_flags.cpp`) + JSON config (`cli/deploy/conf/worker_config.json`); no `.env` convention.
- **Test approach**: dedicated `tests/` tree — `ut` (unit, GoogleTest), `st` (system), `perf`, plus `tests/python` (pytest); C++ tests registered to CTest.
- **Import style**: C++ project-root-relative includes (`datasystem/...`); protobuf-generated headers flow outward from `protos/`.
- **Linting/formatting**: clang-format + clang-tidy configured at repo root.
- **RPC/contracts**: define `.proto` in `src/datasystem/protos`, wire through the ZMQ/RPC layer.

## Entry Points

| Type | Path | Evidence |
|------|------|----------|
| Worker daemon (main) | `src/datasystem/worker/worker_main.cpp` | `main()` for the worker process |
| Worker CLI wiring | `src/datasystem/worker/worker_cli.cpp` | flag/config parse and startup |
| Server assembly | `src/datasystem/server/common_server.cpp` | executable-side wiring |
| Unified build | `build.sh` | `Usage: bash build.sh [-h] [-r] [-d] [-b cmake|bazel] [-c off/on/html] [-t off|build|run] ...` |
| Python package | `setup.py` | builds `openyuanrong-datasystem` wheel (Python 3.9–3.11) |
| Python SDK root | `python/yr/datasystem/__init__.py` | `ds_client`, `kv_client`, `object_client`, `stream_client`, `hetero_client` |
| Operational CLI | `cli/command.py`, `cli/start.py`, `cli/up.py` | cluster start/stop/up/down, config & helm generation |
| C++ SDK header | `include/datasystem/datasystem.h` | public API aggregation header |
| Go binding | `src/datasystem/c_api/*` + `go/` (module `clients`) | cgo bridge over C API |

## Functionality Inventory

| Capability | Primary Files | Summary | Confidence |
|------------|---------------|---------|------------|
| Object cache (client) | `src/datasystem/client/object_cache/object_client_impl.cpp` | Distributed-futures object cache: put/get/ref, local replica of hot data | HIGH |
| Object cache (worker) | `src/datasystem/worker/object_cache/service/worker_oc_service_get_impl.cpp`, `worker_oc_service_batch_get_impl.cpp` | Worker-side object get/batch-get, eviction, spill, slot recovery | HIGH |
| Object metadata (master) | `src/datasystem/master/object_cache/oc_metadata_manager.cpp` | Master metadata ownership, replica management, cleanup-on-eviction | HIGH |
| KV cache | `src/datasystem/client/kv_cache/*`, `src/datasystem/worker/*` | Zero-copy shared-memory KV with reliability (write_through/back/none) | MEDIUM |
| Stream cache (pub/sub) | `src/datasystem/worker/stream_cache/client_worker_sc_service_impl.cpp`, `client/stream_cache/*` | Producer/consumer stream pub-sub, decoupled data transfer | HIGH |
| Heterogeneous (NPU) object | `src/datasystem/client/hetero_cache/*`, `transfer_engine/` | HBM object abstraction, H2D/D2H swap, inter-NPU P2P transfer | MEDIUM |
| Cluster management | `src/datasystem/worker/cluster_manager/*`, `worker/hash_ring/*` | Node discovery/health, hash-ring routing, scale up/down, failover | HIGH |
| ETCD/Metastore metadata | `src/datasystem/common/kvstore/etcd/*` | Distributed metadata: watch, keepalive, CAS, lease | HIGH |
| L2 cache / slot storage | `src/datasystem/common/l2cache/*`, `worker/object_cache/slot_recovery*` | Secondary SSD storage: slot store, replay, compaction, takeover recovery | HIGH |
| RDMA/URMA transport | `src/datasystem/common/rdma/urma_manager.cpp` | RDMA/URMA fast inter-worker data path | MEDIUM |
| Observability | `src/datasystem/common/log/*`, `common/metrics/*` | Structured logs, trace/context, access recorder, metrics exporters | HIGH |

## Module Ownership

| Area | Paths | Responsibilities | Downstream Consumers |
|------|-------|------------------|----------------------|
| Client SDK | `src/datasystem/client`, `include/datasystem`, `python/yr/datasystem` | Public KV/Object/Stream/Hetero API, shared-memory data path, service discovery | Python/Java/Go bindings, user apps |
| Worker runtime | `src/datasystem/worker` | Resource/metadata ownership, cache data services, cluster membership | Clients, master, cluster peers |
| Master/coordinator | `src/datasystem/master` | Object/stream metadata, replica management, redirect helpers | Workers |
| Shared infra | `src/datasystem/common` | rdma, l2cache/slot, log, metrics, rpc/zmq, kvstore/etcd, shared_memory, flags | All of client/worker/master |
| Protos/contracts | `src/datasystem/protos` (20 `.proto`) | RPC/message contracts | client, worker, master, server |
| Bindings | `src/datasystem/{c_api,java_api,pybind_api}` | C/Go, Java/JNI, Python/pybind11 surfaces | `go/`, `java/`, `python/` |
| Transfer engine | `transfer_engine` | NPU HBM transfer subsystem (ACL/HCCL, P2P) | Hetero object path |
| Ops / deploy | `cli`, `k8s`, `k8s_deployment`, `scripts` | Cluster lifecycle, config/helm generation, packaging | Operators |
| Tests | `tests` (`ut`/`st`/`perf`/`python`), `dsbench` | Unit/system/perf validation | CI |

## Risk Areas

| Area | Risk Level | Why | Recommendation |
|------|-----------|-----|----------------|
| Object-cache hot path | HIGH | `object_client_impl.cpp` (4132 LOC, 36 changes/90d) and `worker_oc_service_get_impl.cpp` (3147 LOC, 32 changes/90d) are large **and** highest-churn — the core get path | Treat as hot path: assess latency, lock contention, copies, cache locality before edits; avoid parallel edits; add targeted tests |
| Master metadata manager | HIGH | `oc_metadata_manager.cpp` = 4819 LOC, single largest source file; owns eviction-driven cleanup + replica metadata | Split responsibilities cautiously; crash-consistency and cleanup idempotency are critical |
| RDMA/URMA transport | MEDIUM | `urma_manager.cpp` = 31 changes/90d; memory-safety and lifetime sensitive | Verify buffer ownership/lifetime and registration under `common/rdma`; use `rdma-ucx-perf-debug` skill |
| Slot / recovery path | MEDIUM | `slot_end2end_test.cpp` high churn; persistence, replay, compaction, takeover on restart | Assess partial writes, startup rebuild, idempotency, failover (`.repo_context/modules/infra/slot/design.md`) |
| Large system tests | MEDIUM | Multiple `tests/st` files 2500–4300 LOC (producer/kv scale/eviction) | Keep flakiness in check; slow, stateful multi-process tests |
| Technical debt | LOW | 18 TODO/FIXME/HACK/XXX markers across 9 files (~0.01/file over ~1900 files) | Well-maintained; address debt only in files you touch |

## Technical Debt Signals

- **TODO/FIXME count**: 18 markers across 9 files (density ~0.01/file — LOW).
- **Large files (>500 lines)**: `oc_metadata_manager.cpp` (4819), `object_client_impl.cpp` (4132), `worker_oc_service_get_impl.cpp` (3147), `client_worker_sc_service_impl.cpp` (2664); several `tests/st` files 2400–4300. (`tests/kvtest/src/vendor/httplib.h` at 9370 is vendored — exclude.)
- **Files without tests**: N/A at file granularity — the project ships a large dedicated `tests/` tree (ut/st/perf/python) rather than co-located tests.
- **Git hotspots (90d)**: `object_client_impl.cpp` (36), `worker_oc_service_get_impl.cpp` (32), `common/rdma/urma_manager.cpp` (31); 674 commits in the last 90 days (very active).

## Dependency Risk

**Ecosystem**: C/C++ (Bazel + CMake, vendored `third_party/`) with Python packaging (`setup.py`).

Dependency currency cannot be assessed with a package-manager `outdated` command: C++ dependencies
are vendored under `third_party/` and pinned via Bazel `WORKSPACE` / CMake `cmake/external_libs`,
not a lockfile-driven registry. The Python side (`setup.py`) has an empty root `requirements.txt`
(build/runtime deps are wheel-bundled). See `Third_Party_Open_Source_Software_Notice` for the full
vendored-dependency inventory.

### Dependency Risk Summary
| Metric | Value | Risk Level |
|--------|-------|-----------|
| Outdated packages | Not measurable (vendored/pinned) | _Unknown_ |
| Major version behind | Not measurable | _Unknown_ |
| Heavy dependencies | Vendored `third_party/` (many, intentional for a systems project) | MEDIUM |
| Potentially unmaintained | Not measurable without per-vendor audit | _Unknown_ |

_Automated dependency-currency tooling does not apply cleanly to this vendored C++ layout; audit
`third_party/` and `WORKSPACE`/`cmake/external_libs` manually when a specific dependency is in scope._

## Agent Guidance

Distilled advice for agents working on this codebase:

- **Preferred**: Follow `.repo_context/` routing (start at `index.md`) before touching a module; C++17 with `snake_case`; match existing style; add/extend tests under `tests/ut` (unit) or `tests/st` (system); define RPC changes as `.proto` in `src/datasystem/protos`; treat performance, concurrency, and recovery as requirements.
- **Avoid**: Broad refactors or formatting churn; editing vendored `third_party/`; introducing new build systems (respect the dual Bazel/CMake graph); adding `.env`-style config (use gflags/JSON config); assuming a single build path — changes must work under both Bazel and CMake where applicable.
- **Touch with care**: `master/object_cache/oc_metadata_manager.cpp`, `client/object_cache/object_client_impl.cpp`, `worker/object_cache/service/worker_oc_service_get_impl.cpp`, `common/rdma/urma_manager.cpp`, and the slot/recovery path — all large and/or high-churn on the hot data path. Assess latency, lock ordering, memory lifetime, and crash consistency; coordinate to avoid parallel edits.

## Dependency Graph

**Files analyzed**: sampled from entry points, largest files, and 90d hotspots (import analysis is
approximate for a C++/protobuf project where coupling flows through generated headers).

Structural coupling (from directory + proto layout, not line-level import parse):

- `worker/*` and `client/*` and `master/*` → depend on `common/*` (rpc, log, metrics, shared_memory, kvstore) and on generated headers from `protos/*`.
- `protos/*` is a high fan-in hub — 20 `.proto` files define contracts consumed across client/worker/master/server.
- `common/rdma`, `common/l2cache`, `common/shared_memory` are shared low-level transport/storage consumed by worker object/stream cache.
- Bindings (`c_api`, `java_api`, `pybind_api`) → depend on `client/*` and `include/datasystem/*`.

### Fan-in (most depended-on, inferred)
| Area | Depended on by |
|------|----------------|
| `src/datasystem/common` | client, worker, master, server, bindings |
| `src/datasystem/protos` | client, worker, master, server |
| `include/datasystem` | all bindings + external SDK consumers |

_Line-level import graph not fully resolved — C++ include + protobuf codegen coupling requires build-graph analysis. Use Bazel/CMake target graphs (`modules/quality/cmake-build/design.md`) for precise edges._

## Test Coverage Map

**Test convention**: dedicated `tests/` tree — GoogleTest for C++ (`tests/ut`, `tests/st`), pytest for Python (`tests/python`), plus `tests/perf` and `tests/benchmark`. Registered to CTest via `cmake/scripts/GoogleTestToCTest.cmake`.
**Coverage**: Not measured — no coverage report file present in the tree (build.sh exposes `-c off/on/html` to generate one on demand).
**Source**: Estimated from test-tree structure; no `coverage-summary.json`/`lcov.info`/`coverage.xml` found.

### Critical Untested Files
_No untested critical files detected at map granularity_ — core hot-path modules (`object_client_impl`, `worker_oc_service_get_impl`, `oc_metadata_manager`, slot recovery) all have corresponding `tests/st`/`tests/ut` suites (e.g. `object_client_test.cpp`, `slot_end2end_test.cpp`, `slot_store_test.cpp`). Run `build.sh -c on` for real line coverage before asserting gaps.

## API Surface

No HTTP/REST web framework — this is an RPC system. The API surface is **protobuf service
contracts** (20 `.proto` files in `src/datasystem/protos`) carried over the ZMQ/RPC layer, plus the
public SDK headers in `include/datasystem`.

| Contract group | Proto files | Purpose |
|--------|------|---------|
| Object | `master_object.proto`, `worker_object.proto`, `object_posix.proto`, `ut_object.proto` | Object cache metadata + data RPCs |
| Stream | `master_stream.proto`, `worker_stream.proto`, `stream_posix.proto`, `p2p_subscribe.proto` | Stream pub/sub RPCs |
| Cluster/meta | `master_heartbeat.proto`, `hash_ring.proto`, `meta_transport.proto`, `meta_zmq.proto`, `slot_recovery.proto` | Heartbeat, routing, metadata transport, recovery |
| Transport/generic | `generic_service.proto`, `rpc_option.proto`, `share_memory.proto`, `perf_posix.proto`, `zmq_perf.proto`, `zmq_test.proto`, `utils.proto` | Generic service, shared-memory, perf, options |

Public SDK API surface: `include/datasystem/{kv_client.h, object_client.h, stream_client.h, hetero_client.h, perf_client.h, router_client.h, datasystem.h}`.

## Config & Environment

**Config style**: gflags (compile/runtime flags) + JSON config files; no `.env` convention.

### Config Files
| File | Category | Notes |
|------|----------|-------|
| `cli/deploy/conf/worker_config.json` | Runtime | Worker deployment config (high-churn, 21 changes/90d) |
| `src/datasystem/worker/worker_flags.cpp`, `common/flags/*` | Runtime | gflags definitions for worker/common |
| `.bazelrc` | Build | Bazel C++17 build options |
| `CMakeLists.txt`, `cmake/*` | Build | CMake build graph + external libs |
| `k8s/helm_chart/datasystem/values.yaml`, `k8s_deployment/helm_chart` | Deploy | Helm chart values (high-churn, 24 changes/90d) |
| `WORKSPACE`, `version.bzl`, `VERSION` (0.8.2) | Build/version | Bazel workspace + version pinning |

### Environment / Flags
Configuration is driven primarily through **gflags** and JSON config, not process environment
variables. `DS_*` symbols in source are mostly status-code / gflag-declaration macros
(`DS_DEFINE_*`, `DS_ETCD_*`), not runtime env vars. ETCD endpoints, ports, resource sizes, and
reliability/consistency modes are set via worker flags/config. Secret material (ak/sk, tokens, IAM)
is handled under `src/datasystem/common/{ak_sk,token,iam,encrypt}`.

### Secret Exposure Warnings
No `.env` files tracked. Auth/secret handling is code-managed under `common/{ak_sk,token,iam,encrypt}`; audit those modules (and cert-gen helpers `tests/ut/gen_cert*.sh`) rather than a dotenv surface.

## Setup / Runbook

| Task | Command or File | Notes |
|------|-----------------|-------|
| Build (bazel or cmake) | `bash build.sh -b bazel` or `bash build.sh -b cmake` | Unified entrypoint; `-r` release, `-d` debug, `-j <n>` parallelism |
| Build + run tests | `bash build.sh -t run` | `-t off|build|run` controls the test stage |
| Coverage | `bash build.sh -c on` (or `html`) | Generates coverage on demand (off by default) |
| Install (Python) | `pip install openyuanrong-datasystem` | Published to PyPI; Python 3.9–3.11, Linux x86-64 |
| Build Python wheel | `python setup.py bdist_wheel` | See `setup.py` |
| Deploy cluster | `python cli/command.py` (`up`/`start`/`stop`/`down`) | Requires ETCD; per-node worker registration |
| Format / lint | clang-format / clang-tidy | Config at repo root (`.clang-format`, `.clang-tidy`) |

## Pattern Library

1. **Role-layered services** — `*_service_impl.{h,cpp}` pairs implement RPC service handlers per role (`worker_service_impl`, `master_service_impl`, `worker_oc_service_get_impl`). Follow this split when adding an RPC.
2. **`common/` shared infra** — cross-cutting concerns (log, metrics, rpc, shared_memory, kvstore, rdma) live in `src/datasystem/common`; reuse before adding new helpers (per CLAUDE.md).
3. **Protobuf-first contracts** — define/modify `.proto` in `src/datasystem/protos`, then wire handlers; generated headers are the coupling surface.
4. **gflags + `.def` registration** — flags and metric/access-point families use `.def` include-lists (`res_metrics.def`, `access_point.def`) and gflag macros.
5. **GoogleTest → CTest** — C++ tests use gtest and are registered to CTest via `cmake/scripts/GoogleTestToCTest.cmake`; place unit tests in `tests/ut`, system tests in `tests/st`.

## Directory Mappings

Standard locations for different file categories:

| Category | Primary Location | Priority | Pattern |
|----------|-----------------|----------|---------|
| core source | `src/datasystem` | explicit | `**/*.{cpp,h}` |
| public headers | `include/datasystem` | explicit | `**/*.h` |
| contracts | `src/datasystem/protos` | explicit | `**/*.proto` |
| shared infra | `src/datasystem/common` | explicit | `**/*.{cpp,h}` |
| tests | `tests` (`ut`/`st`/`perf`/`python`) | explicit | `**/*_test.cpp`, `test_*.py` |
| python sdk | `python/yr/datasystem` | explicit | `**/*.py` |
| cli / ops | `cli`, `scripts` | explicit | `**/*.py` |
| config | `cli/deploy/conf`, `common/flags` | inferred | `*.json`, `*_flags.cpp` |
| deploy | `k8s`, `k8s_deployment` | explicit | `**/*.{yaml,yml}` |
| docs | `docs/source_zh_cn` | explicit | `**/*.{md,rst}` |
| examples | `example` (cpp/go/java/python) | explicit | `**/*` |

### Path Enforcement Rules
- **Strictness**: warn
- New files should follow these mappings and match the dual Bazel/CMake build graph (add `BUILD.bazel` + `CMakeLists.txt` entries).
- Exceptions require explicit override.

## Retrieval Artifacts

- **Index**: `.planning/codebase/index.jsonl`
- **Symbols**: `.planning/codebase/symbols.json`
- **Search protocol**: `.planning/codebase/search.md`

> Note: This repository maintains a rich, source-backed context set under `.repo_context/`. That
> tree is the authoritative, human-curated guidance (module designs, playbooks, routing). This map
> is the Legion-consumable retrieval layer — cross-reference `.repo_context/index.md` for depth.
