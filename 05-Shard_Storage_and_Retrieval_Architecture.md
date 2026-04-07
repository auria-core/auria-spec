# Shard Storage and Retrieval Architecture
## AURIA Engineering Specification Book
## Document 05 of 22
## Version: 1.0
## Status: Draft for Production Implementation

---

# 1. Purpose

This document defines the storage, caching, retrieval, and persistence architecture for shards within the AURIA Runtime Core (ARC).

The shard storage subsystem is implemented as the **AURIA Model Store (AMS)**.

This document specifies:

- Storage hierarchy
- Disk layout
- Cache architecture
- Retrieval algorithms
- Prefetch strategy
- Distributed storage integration
- Integrity enforcement

This subsystem is critical to runtime performance and correctness.

---

# 2. Storage Architecture Overview

Shard storage is hierarchical:

```
Level 0: GPU VRAM Cache
Level 1: Pinned RAM Cache
Level 2: Disk Cache
Level 3: Distributed Storage
Level 4: Remote Archival Storage
```

Each level has different latency and capacity characteristics.

---

# 3. Storage Levels

## 3.1 Level 0 — VRAM Cache

Characteristics:

- fastest access
- smallest capacity
- volatile

Used for:

- active experts
- execution tensors

Managed by Expert Cache (Document 08).

---

## 3.2 Level 1 — RAM Cache

Characteristics:

- fast access
- medium capacity
- volatile

Used for:

- recently accessed shards
- prefetched shards

Pinned RAM MUST be used for GPU transfers.

---

## 3.3 Level 2 — Disk Cache

Characteristics:

- persistent
- large capacity
- medium latency

Primary local storage for shards.

Disk cache MUST be used as primary shard store.

---

## 3.4 Level 3 — Distributed Storage

Examples:

- IPFS
- Arweave
- Peer nodes

Used when shard not present locally.

---

## 3.5 Level 4 — Archival Storage

Examples:

- cold storage
- archival nodes

Used for long-term preservation.

High latency.

---

# 4. Disk Cache Directory Structure

Disk cache root:

```
~/.auria/shards/
```

Shard files organized by prefix:

```
~/.auria/shards/
├── ab/
│   ├── abcd1234... .auria-shard
├── cd/
│   ├── cdef5678... .auria-shard
```

Prefix directory derived from shard_id first two bytes.

This prevents excessive directory size.

---

# 5. Shard File Naming

Shard file name:

```
<shard_id>.auria-shard
```

Example:

```
a3f4c2d1... .auria-shard
```

Shard ID MUST match file name.

---

# 6. Shard Index

Model Store MUST maintain shard index.

Structure:

```rust
struct ShardIndexEntry {
    shard_id: [u8; 32],
    local_path: Option<PathBuf>,
    storage_location: StorageLocation,
    last_accessed: u64,
}
```

Shard index stored at:

```
~/.auria/shard_index.json
```

---

# 7. Storage Location Enum

```rust
enum StorageLocation {
    VRAM,
    RAM,
    Disk,
    Distributed,
    Remote,
}
```

---

# 8. Shard Retrieval Algorithm

Retrieval procedure:

```
Check VRAM cache
If not present:
    Check RAM cache
If not present:
    Check Disk cache
If not present:
    Check Distributed storage
If not present:
    Check Remote storage
```

First valid shard MUST be used.

---

# 9. Disk Retrieval Procedure

Disk retrieval steps:

```
Locate shard path
Verify file exists
Verify shard integrity
Load shard
Return shard
```

Integrity MUST be verified before use.

---

# 10. Distributed Storage Integration

Distributed storage MUST support:

- shard lookup by shard_id
- shard download
- shard verification

Supported protocols:

- IPFS CID lookup
- HTTP retrieval
- Peer-to-peer retrieval

Distributed storage MUST NOT be trusted without verification.

---

# 11. IPFS Integration

IPFS shard mapping:

```
shard_id → CID
```

CID stored in metadata.

Retrieval:

```
ipfs cat <CID>
```

Retrieved shard MUST be verified.

---

# 12. Shard Download Procedure

Download procedure:

```
Request shard from storage
Receive shard file
Verify shard integrity
Store shard in disk cache
Update shard index
Return shard
```

Invalid shard MUST be rejected.

---

# 13. Prefetch Strategy

Prefetch improves performance.

Prefetch MUST occur when:

- expert routing decision made
- expert likely to be used soon

Prefetch MUST NOT block execution.

Prefetch MUST be asynchronous.

---

# 14. Prefetch Algorithm

Prefetch procedure:

```
Predict next expert usage
Retrieve shards for predicted experts
Store shards in RAM cache
```

Prediction based on routing history.

---

# 15. Cache Promotion Policy

Promotion rules:

Disk → RAM:

```
When shard accessed frequently
```

RAM → VRAM:

```
When expert assembled
```

Promotion MUST improve performance.

---

# 16. Cache Eviction Policy

Eviction policy:

```
LRU-K (Least Recently Used with frequency)
```

Eviction order:

```
VRAM → RAM → Disk
```

Disk eviction only when storage full.

---

# 17. Disk Capacity Management

Disk cache size configurable.

Default:

```
100 GB
```

Disk usage MUST be monitored.

Old shards MAY be evicted.

---

# 18. Integrity Enforcement

All shards MUST be verified before use.

Verification includes:

- shard hash verification
- metadata verification
- signature verification

Unverified shards MUST NOT be used.

---

# 19. Concurrency Model

Shard retrieval MUST be thread-safe.

Multiple threads MAY retrieve shards.

Shard retrieval MUST avoid duplicate downloads.

Locking MUST be implemented.

---

# 20. Failure Handling

Shard retrieval failures MUST trigger fallback.

Fallback options:

- alternative storage source
- retry retrieval
- execution failure

Failure MUST NOT corrupt state.

---

# 21. Storage Interface

Model Store interface:

```rust
trait ModelStore {
    fn get_shard(shard_id: [u8; 32]) -> Result<Shard>;
    fn store_shard(shard: Shard);
    fn shard_exists(shard_id: [u8; 32]) -> bool;
}
```

All shard access MUST use Model Store.

---

# 22. Summary

Shard storage architecture provides:

- persistent shard storage
- fast shard retrieval
- distributed shard access
- integrity enforcement
- performance optimization

Model Store ensures shards are reliably available and verifiable.

This subsystem is essential for ARC execution performance and correctness.