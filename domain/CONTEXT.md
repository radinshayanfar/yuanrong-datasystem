# YuanRong DataSystem

Domain glossary for the openYuanrong DataSystem: a distributed, multi-level cache that pools HBM/DRAM/SSD across a compute cluster and serves data to application processes through a shared-memory SDK. Terms below are the canonical vocabulary derived from the public API (`include/datasystem`) and the `src/datasystem` source tree.

## Runtime Roles

**Worker**:
The DataSystem runtime process deployed on each node; it owns the node's DRAM/SSD memory, serves cache operations over shared memory, and participates in cluster routing. One worker per node.
_Avoid_: server, agent, daemon, node (a node hosts a worker but is not the worker)

**Master**:
The metadata and coordination side of the runtime, responsible for object metadata, global references, and cluster-wide bookkeeping. In distributed mode the master/worker boundary is a code-level distinction, not two separate deployables.
_Avoid_: coordinator, metaserver, controller

**Embedded worker**:
A worker run in-process inside the client via `InitEmbedded`, so SDK and worker share the same process rather than communicating across a socket.
_Avoid_: local worker, in-proc server

**Node**:
A cluster host that runs one worker and any co-located application processes.
_Avoid_: machine, host, instance

## Client SDK

**DsClient**:
The aggregate client that bundles the KV, Object, and Hetero cache clients behind one connection and lifecycle. Note: it does **not** include the Stream client, which is constructed separately.
_Avoid_: Client, DataSystemClient, SDK (SDK is the whole binding layer, not this class)

**KV cache**:
The key-value cache family (`KVClient`) offering zero-copy `Set`/`Get`-style access over shared memory. This is the canonical name even though the header prose calls it the "state cache".
_Avoid_: state cache, KVStore, kv_client

**Object cache**:
The object cache family (`ObjectClient`) for near-compute local object storage, built around a `Buffer`/`Publish` create flow and global reference counting; underpins the Distributed Futures programming model.
_Avoid_: blob store, object store

**Stream cache**:
The producer/consumer streaming family (`StreamClient`, `Producer`, `Consumer`) for element-based pub/sub over named streams.
_Avoid_: queue, message cache, pubsub

**Hetero cache**:
The heterogeneous-object family (`HeteroClient`) that abstracts device (NPU) HBM as DataSystem objects and moves data host↔device and device↔device.
_Avoid_: device cache, GPU cache, hetero client

## Data & Memory Handles

**Object key**:
The string identifier for a cached object or KV entry. Constrained to letters, digits, and `~!@#$%^&*.-_`, length < 256.
_Avoid_: id, name, objectId (use "object key" for the API-facing term)

**Buffer**:
An object-cache handle over a shared-memory region into which the client writes object data before `Publish` makes it visible cluster-wide.
_Avoid_: blob, shm buffer, memory block

**Nested object**:
An object referenced by another object at `Publish` time (via `nestedKeys`), so its lifetime is tied to the parent.
_Avoid_: child object, sub-object

**Blob**:
A `{pointer, size}` descriptor for a contiguous region of device HBM in the hetero path.
_Avoid_: buffer (reserve "Buffer" for the object-cache handle), chunk

**DeviceBlobList**:
A group of `Blob`s on a single device card, tagged with `deviceIdx`, used as the unit of an H2D/D2H/D2D transfer.
_Avoid_: blob group, tensor list

**Element**:
The unit of data flowing through a stream: a `{ptr, size, id}` record sent by a Producer and received by a Consumer.
_Avoid_: message, record, item

**HBM**:
High-Bandwidth Memory on an NPU device card; the fastest tier and the native residence of heterogeneous objects.
_Avoid_: device memory, VRAM, GPU memory

## Object Lifecycle & References

**Global reference count**:
The cluster-wide reference count on an object that governs its lifetime, adjusted via `GIncreaseRef`/`GDecreaseRef` and read via `QueryGlobalRefNum`.
_Avoid_: refcount, GRef, global refs (fine informally; "global reference count" is canonical)

**Out-cloud reference**:
A reference held by a client outside the cluster ("out of cloud"), keyed by `remoteClientId` so gateway/ELB forwarding and client crashes don't corrupt the count.
_Avoid_: remote ref, external ref, out-of-cluster ref

**TTL**:
Time-to-live in seconds after which a key expires; `ttlSecond = 0` means keep-alive until explicit delete.
_Avoid_: expiry, timeout

**Eviction**:
Worker-side reclaiming of cache entries under memory pressure (LRU-based), distinct from explicit delete and from TTL expiry.
_Avoid_: purge, cleanup, reclaim

## Storage & Cache Tiers

**Multi-level cache**:
The tiered arrangement of HBM/DRAM/SSD that DataSystem presents as one distributed cache.
_Avoid_: multi-tier cache, hierarchical cache, cache hierarchy

**L2 cache**:
The secondary (durable/backend) cache tier that object/KV writes can flow to, selected per-write via `WriteMode` (e.g. write-through, write-back).
_Avoid_: secondary cache, L2, backend cache, persistence layer

**Spill**:
Worker-side movement of cache data out of DRAM down to a lower tier (disk / L2) to free memory.
_Avoid_: flush, offload, page-out

**Slot**:
The distributed-disk storage unit backing on-disk caching, with its own replay, compaction, takeover, and restart-recovery lifecycle (`slot_client`, `slot_recovery`).
_Avoid_: shard, segment, partition

**Shared memory**:
The zero-copy transport between an application process (SDK) and its local worker; data is read/written in place without copying across the boundary.
_Avoid_: shm (fine in code), IPC memory, mmap region

**CacheType**:
Per-operation choice of where an entry primarily lives: `MEMORY` or `DISK`.
_Avoid_: storage type, tier type

## Write & Consistency Semantics

**WriteMode**:
Per-write policy controlling L2-cache interaction: `NONE_L2_CACHE`, `WRITE_THROUGH_L2_CACHE` (sync), `WRITE_BACK_L2_CACHE` (async), and their `_EVICT` (evictable) variants.
_Avoid_: write policy, durability mode, sync mode

**ConsistencyType**:
The consistency model requested when creating an object: `PRAM` or `CAUSAL`.
_Avoid_: consistency level, consistency mode

**ExistenceOpt**:
The set-time existence guard: `NONE` (overwrite) or `NX` (set only if the key is absent).
_Avoid_: set condition, upsert flag

## Streaming Concepts

**Producer**:
A stream writer created via `CreateProducer` that sends Elements to a named stream.
_Avoid_: publisher, writer, sender

**Consumer**:
A stream reader created via `Subscribe` that receives Elements and (optionally auto-) acknowledges them.
_Avoid_: subscriber, reader, receiver

**Subscription**:
The binding of a Consumer to a stream, configured by `SubscriptionConfig` (name, type, cache capacity, prefetch low-water-mark).
_Avoid_: subscribe relation, binding, channel

**SubscriptionType**:
The delivery discipline of a subscription: `STREAM`, `ROUND_ROBIN`, or `KEY_PARTITIONS`.
_Avoid_: mode, delivery mode, queue mode

**Acknowledgement (Ack)**:
A Consumer's confirmation that an Element has been processed, gating stream progress; may be automatic (`autoAck`).
_Avoid_: commit, confirm, receipt

## Cluster Coordination

**Cluster management**:
The subsystem handling node discovery, health detection, fault recovery, and online scale-up/scale-down.
_Avoid_: cluster control, orchestration, membership service

**Hash ring**:
The worker-side consistent-hashing structure used for data routing and membership as nodes are added, removed, or restarted.
_Avoid_: consistent hash, ring, routing table

**ETCD**:
The external metadata/coordination backend used for node registration, health, and cluster state.
_Avoid_: registry, config store

**Metastore**:
The built-in metadata service intended to replace an external ETCD deployment in some cluster modes.
_Avoid_: meta service, internal etcd, metadata store

## Transfer & Movement

**H2D / D2H / D2D**:
Directional data-movement operations in the hetero path — host-to-device, device-to-host, and device-to-device (`MGetH2D`, `MSetD2H`, etc.). H2D and D2H are used as a matched pair.
_Avoid_: upload/download, copy-in/copy-out

**Heterogeneous object**:
An NPU HBM region published as a first-class DataSystem object, enabling direct inter-card transfer and P2P distribution.
_Avoid_: device object, tensor object, remote tensor

**P2P transmission**:
Direct inter-card (NPU-to-NPU) data distribution over HCCS/RoCE, with load-balancing across card links, used to fan model data out across nodes.
_Avoid_: broadcast, direct copy, peer transfer
