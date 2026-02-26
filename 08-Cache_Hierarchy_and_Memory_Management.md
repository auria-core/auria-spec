# Cache Hierarchy and Memory Management
## AURIA Engineering Specification Book
## Document 08 of 22
## Version: 1.0
## Status: Draft for Production Implementation

---

# 1. Purpose

This document defines the cache hierarchy, memory allocation strategy, and memory lifecycle management for the AURIA Runtime Core (ARC).

The cache subsystem is implemented as the **AURIA Expert Cache (AEC)**.

This subsystem is responsible for:

- minimizing shard assembly latency
- minimizing device transfer overhead
- maximizing hardware utilization
- ensuring deterministic and safe memory use

Cache hierarchy is critical for performance.

---

# 2. Cache Hierarchy Overview

ARC implements a four-level cache hierarchy:

```
Level 0: Device Cache (VRAM)
Level 1: Pinned Host Cache (RAM)
Level 2: General Host Cache (RAM)
Level 3: Persistent Disk Cache
```

Each level serves different performance and persistence goals.

---

# 3. Cache Levels

## 3.1 Level 0: Device Cache (VRAM)

Characteristics:

- fastest access
- smallest capacity
- volatile

Stores:

- assembled expert tensors
- active execution tensors

VRAM cache is primary execution cache.

---

## 3.2 Level 1: Pinned Host Cache (Pinned RAM)

Characteristics:

- fast access
- DMA-capable
- volatile

Stores:

- prefetched experts
- staging tensors

Pinned RAM enables fast GPU transfers.

---

## 3.3 Level 2: General Host Cache (RAM)

Characteristics:

- moderate access speed
- larger capacity
- volatile

Stores:

- recently assembled experts
- frequently used tensors

---

## 3.4 Level 3: Disk Cache

Characteristics:

- persistent
- large capacity
- slowest access

Stores:

- shard files
- optional serialized experts

Managed by Model Store (Document 05).

---

# 4. Cache Object Structure

Expert cache object structure:

```rust
struct CachedExpert {
    expert_id: [u8; 32],
    device_pointer: Option<DevicePointer>,
    host_pointer: Option<HostPointer>,
    pinned_pointer: Option<PinnedPointer>,
    last_accessed: u64,
    access_count: u64,
    memory_size: u64,
}
```

---

# 5. Cache Lookup Procedure

Cache lookup procedure:

```
Check Device Cache
If found: return expert
Else check Pinned RAM Cache
If found: promote to Device Cache
Else check Host RAM Cache
If found: promote to Device Cache
Else assemble expert
```

Cache lookup MUST be deterministic.

---

# 6. Cache Promotion Rules

Promotion hierarchy:

```
Disk → Host RAM → Pinned RAM → Device VRAM
```

Promotion MUST occur when expert is used.

Promotion MUST improve execution speed.

Promotion MUST preserve deterministic behavior.

---

# 7. Cache Eviction Policy

Eviction policy MUST use:

```
LRU-K (Least Recently Used with Frequency)
```

Eviction priority:

```
Device Cache → Pinned RAM → Host RAM → Disk Cache
```

Eviction MUST preserve system stability.

Eviction MUST NOT remove in-use tensors.

---

# 8. VRAM Cache Management

VRAM cache MUST enforce capacity limit.

Capacity defined by tier:

Nano:

No VRAM cache

Standard:

≤ 80% VRAM

Pro:

≤ 90% VRAM

Max:

cluster-managed

VRAM usage MUST be tracked.

---

# 9. RAM Cache Management

RAM cache MUST enforce capacity limits.

Pinned RAM limit:

≤ 25% total RAM

Host RAM limit:

≤ 50% total RAM

Cache MUST NOT exhaust system memory.

---

# 10. Cache Entry Lifecycle

Cache entry lifecycle:

```
Created → Active → Idle → Evicted → Destroyed
```

Idle entries MAY be evicted.

Active entries MUST NOT be evicted.

Destroyed entries MUST release memory.

---

# 11. Memory Allocation Types

Supported memory types:

```rust
enum MemoryType {
    DeviceVRAM,
    PinnedRAM,
    HostRAM,
}
```

Allocation MUST match execution target.

---

# 12. Device Memory Allocation

Device memory MUST be allocated via backend API.

Examples:

CUDA:

```
cudaMalloc()
```

ROCm:

```
hipMalloc()
```

Metal:

```
MTLBuffer allocation
```

Device allocation MUST be verified.

---

# 13. Pinned Memory Allocation

Pinned memory MUST be allocated using OS-specific APIs.

Examples:

Linux:

```
mlock()
```

CUDA:

```
cudaHostAlloc()
```

Pinned memory enables faster transfers.

---

# 14. Memory Alignment Requirements

All expert tensors MUST be aligned to:

```
64 bytes minimum
```

Preferred alignment:

```
256 bytes
```

Alignment improves execution performance.

---

# 15. Memory Isolation Requirements

Memory isolation MUST be enforced.

Expert tensors MUST NOT overlap memory regions.

Memory corruption MUST be prevented.

Memory bounds MUST be enforced.

---

# 16. Memory Zeroization

Sensitive memory MUST be cleared when released.

Zeroization procedure:

```
Overwrite memory with zeros
Release memory
```

Zeroization prevents data leakage.

---

# 17. Cache Concurrency Model

Cache MUST be thread-safe.

Concurrent reads allowed.

Writes MUST use locking.

Cache consistency MUST be preserved.

---

# 18. Cache Locking Mechanism

Cache entries MUST support locking.

Lock types:

```
Read Lock
Write Lock
Exclusive Lock
```

Locks prevent concurrent modification.

---

# 19. Prefetch Integration

Cache MUST support prefetch.

Prefetch MUST store experts in pinned RAM.

Prefetch MUST NOT evict active experts.

Prefetch MUST be asynchronous.

---

# 20. Cache Monitoring

Cache subsystem MUST monitor:

- memory usage
- hit rate
- eviction rate
- allocation failures

Metrics MUST be available.

Metrics defined in Document 16.

---

# 21. Cache Interface

Expert Cache interface:

```rust
trait ExpertCache {
    fn get(expert_id: [u8; 32]) -> Option<CachedExpert>;
    fn insert(expert: CachedExpert);
    fn evict(expert_id: [u8; 32]);
    fn exists(expert_id: [u8; 32]) -> bool;
}
```

Cache MUST be authoritative for assembled experts.

---

# 22. Summary

Cache hierarchy ensures efficient expert reuse.

Cache subsystem provides:

- low latency execution
- efficient memory usage
- deterministic caching behavior
- safe memory management

Expert Cache is essential for high-performance ARC execution.