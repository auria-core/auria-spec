# Expert Assembly and Tensor Construction
## AURIA Engineering Specification Book
## Document 07 of 22
## Version: 1.0
## Status: Draft for Production Implementation

---

# 1. Purpose

This document defines the deterministic assembly process for constructing executable expert tensors from shards within the AURIA Runtime Core (ARC).

Expert assembly is implemented by the **AURIA Expert Assembler (AEA)**.

This document specifies:

- shard validation prior to assembly
- shard ordering rules
- tensor construction algorithms
- memory layout requirements
- GPU upload procedures
- watermark injection procedures
- deterministic assembly guarantees

Expert assembly MUST be deterministic, verifiable, and hardware-safe.

---

# 2. Definitions

## 2.1 Expert

An expert is a deterministic tensor constructed from one or more shards.

Experts are execution units.

Experts are constructed at runtime.

Experts are never permanently stored in assembled form.

---

## 2.2 Assembly

Assembly is the process of constructing an expert tensor from shards.

Assembly includes:

- shard retrieval
- shard validation
- tensor concatenation
- memory allocation
- tensor upload

---

## 2.3 Tensor

A tensor is a multi-dimensional numeric array.

Tensors represent model parameters.

---

# 3. Assembly Architecture

Expert assembly consists of six stages:

```
Stage 1: Shard Retrieval
Stage 2: Shard Validation
Stage 3: Assembly Planning
Stage 4: Tensor Construction
Stage 5: Watermark Injection
Stage 6: Device Upload
```

Each stage MUST complete successfully.

---

# 4. Assembly Inputs

Required inputs:

```rust
struct AssemblyRequest {
    expert_id: [u8; 32],
    shard_ids: Vec<[u8; 32]>,
    target_device: Device,
}
```

Shard IDs MUST match expert definition.

---

# 5. Shard Retrieval

Assembler MUST retrieve shards via Model Store.

Assembler MUST NOT bypass Model Store.

Retrieved shards MUST be verified before use.

---

# 6. Shard Validation

Each shard MUST be validated.

Validation includes:

- shard hash verification
- metadata verification
- license verification

Assembler MUST reject invalid shards.

Assembler MUST NOT continue assembly if validation fails.

---

# 7. Assembly Planning

Assembler MUST construct assembly plan.

Assembly plan defines:

```rust
struct AssemblyPlan {
    shard_order: Vec<[u8; 32]>,
    total_elements: u64,
    tensor_shape: Vec<u32>,
    dtype: DataType,
}
```

Shard order MUST be deterministic.

Shard order MUST follow expert definition.

---

# 8. Tensor Construction

Assembler MUST allocate contiguous memory.

Memory allocation size:

```
total_elements × element_size
```

Element size determined by dtype.

Supported sizes:

FP32: 4 bytes  
FP16: 2 bytes  
BF16: 2 bytes  
INT8: 1 byte  
INT4: 0.5 bytes  

Memory MUST be aligned to 64 bytes.

---

# 9. Tensor Concatenation Algorithm

Concatenation procedure:

```
Allocate output buffer
For each shard in shard_order:
    Copy shard tensor data into output buffer
```

Copy MUST preserve order.

Copy MUST be byte-exact.

Assembler MUST NOT modify shard tensor data.

---

# 10. Memory Allocation Requirements

Memory MUST be allocated in one of:

- pinned RAM
- GPU VRAM
- unified memory

Preferred allocation:

Pinned RAM for GPU execution.

Memory allocation MUST succeed before copying.

Allocation failure MUST abort assembly.

---

# 11. Watermark Injection

Watermark injection MAY be required.

Watermark provides leak traceability.

Watermark injection MUST be deterministic per node.

Watermark generation:

```
watermark = SHA256(node_id + shard_id)
```

Watermark applied as minimal noise.

Noise MUST NOT affect model correctness.

Noise MUST be reversible or detectable.

Watermark MUST be applied after tensor construction.

---

# 12. Watermark Injection Algorithm

Procedure:

```
Generate watermark seed
Initialize deterministic RNG
Apply minimal perturbation to selected tensor elements
```

Perturbation MUST remain below numerical tolerance.

Perturbation MUST NOT exceed 1 ULP.

---

# 13. Device Upload

Tensor MUST be uploaded to target device.

Supported devices:

- CPU
- CUDA GPU
- ROCm GPU
- Metal GPU

Upload procedure:

```
Allocate device memory
Copy tensor to device memory
Verify transfer success
```

Upload MUST be verified.

Upload failure MUST abort assembly.

---

# 14. Device Memory Layout

Device tensor MUST use contiguous layout.

Layout MUST match host layout.

No implicit transposition allowed.

Tensor stride MUST be preserved.

---

# 15. Assembly Determinism Requirements

Assembly MUST be deterministic.

Given identical:

- shard inputs
- node identity
- assembly plan

Output tensor MUST be identical.

Determinism MUST NOT depend on:

- thread scheduling
- hardware timing
- memory location

---

# 16. Assembly Output

Output structure:

```rust
struct ExpertTensor {
    expert_id: [u8; 32],
    device_pointer: DevicePointer,
    shape: Vec<u32>,
    dtype: DataType,
    watermark_applied: bool,
}
```

ExpertTensor MUST be ready for execution.

---

# 17. Expert Cache Integration

Assembler MUST store assembled expert in Expert Cache.

Expert Cache MUST manage lifecycle.

Assembler MUST NOT duplicate cached experts.

---

# 18. Assembly Failure Handling

Assembly failure conditions:

- missing shard
- invalid shard
- license invalid
- memory allocation failure
- device upload failure

Failure MUST abort assembly.

Failure MUST NOT corrupt cache.

---

# 19. Thread Safety

Assembly MUST be thread-safe.

Multiple experts MAY be assembled concurrently.

Shared state MUST be protected.

Assembler MUST avoid race conditions.

---

# 20. Assembly Performance Requirements

Assembly time targets:

Nano tier:

≤ 10 ms per expert

Standard tier:

≤ 5 ms per expert

Pro tier:

≤ 3 ms per expert

Max tier:

≤ 1 ms per expert

Assembler MUST optimize for performance.

---

# 21. Expert Assembler Interface

Assembler interface:

```rust
trait ExpertAssembler {
    fn assemble_expert(request: AssemblyRequest) -> Result<ExpertTensor>;
}
```

Assembler MUST be deterministic.

Assembler MUST validate all shards.

---

# 22. Summary

Expert assembly constructs executable tensors from shards.

Expert assembly ensures:

- deterministic assembly
- cryptographic integrity
- license compliance
- hardware compatibility

Expert Assembler is essential for transforming shards into executable intelligence within ARC.