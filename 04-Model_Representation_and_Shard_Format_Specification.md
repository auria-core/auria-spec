# Model Representation and Shard Format Specification
## AURIA Engineering Specification Book
## Document 04 of 22
## Version: 1.0
## Status: Draft for Production Implementation

---

# 1. Purpose

This document defines the canonical representation of intelligence models within AURIA Runtime Core (ARC), including the exact binary format, cryptographic structure, and metadata schema of shards.

This specification ensures:

- deterministic assembly
- cryptographic verifiability
- cross-platform compatibility
- efficient storage and retrieval
- forward compatibility

This document is normative.

All ARC implementations MUST comply.

---

# 2. Definitions

## 2.1 Model

A model is a logical collection of experts and routing parameters.

A model consists of:

- shard set
- expert definitions
- routing configuration
- metadata

Models are assembled dynamically.

Models are never stored as monolithic objects.

---

## 2.2 Shard

A shard is the atomic storage unit.

A shard contains:

- tensor data
- shard metadata
- cryptographic hash
- license binding

Shards are immutable.

---

## 2.3 Expert

An expert is a deterministic assembly of shards.

Experts are execution units.

Experts are not stored directly.

Experts are assembled at runtime.

---

# 3. Shard Format Overview

Shard file format consists of five sections:

```
Shard File
├── Header
├── Metadata
├── Tensor Descriptor
├── Tensor Data
└── Cryptographic Footer
```

All sections MUST be present.

---

# 4. File Encoding

Shard files MUST use binary encoding.

File extension:

```
.auria-shard
```

Optional compressed extension:

```
.auria-shard.zst
```

Compression algorithm:

```
Zstandard (Zstd)
```

Compression MUST NOT alter tensor semantics.

---

# 5. Header Specification

Header structure:

```rust
struct ShardHeader {
    magic: [u8; 8],
    version: u32,
    shard_id: [u8; 32],
    metadata_offset: u64,
    tensor_offset: u64,
    footer_offset: u64,
}
```

---

## 5.1 Header Field Definitions

magic:

```
"AURSHARD"
```

Used to identify valid shard file.

version:

```
1
```

Defines shard format version.

shard_id:

32-byte unique identifier.

metadata_offset:

Byte offset to metadata section.

tensor_offset:

Byte offset to tensor section.

footer_offset:

Byte offset to cryptographic footer.

---

# 6. Metadata Specification

Metadata contains descriptive information.

Structure:

```rust
struct ShardMetadata {
    model_id: [u8; 32],
    expert_id: [u8; 32],
    shard_index: u32,
    shard_count: u32,
    created_timestamp: u64,
    tensor_hash: [u8; 32],
}
```

---

# 7. Tensor Descriptor Specification

Tensor descriptor defines tensor structure.

Structure:

```rust
struct TensorDescriptor {
    dtype: DataType,
    dimensions: Vec<u32>,
    element_count: u64,
}
```

---

# 8. Supported Data Types

Supported data types:

```
FP32
FP16
BF16
FP8
INT8
INT4
```

Data type enum:

```rust
enum DataType {
    FP32,
    FP16,
    BF16,
    FP8,
    INT8,
    INT4,
}
```

---

# 9. Tensor Data Specification

Tensor data is stored as raw binary array.

Layout:

```
Row-major order
```

Element order MUST be deterministic.

Tensor data MUST NOT include padding.

---

# 10. Tensor Alignment

Tensor data MUST be aligned to:

```
64-byte boundary
```

Alignment improves performance.

---

# 11. Cryptographic Footer Specification

Footer contains integrity verification.

Structure:

```rust
struct ShardFooter {
    shard_hash: [u8; 32],
    metadata_hash: [u8; 32],
    tensor_hash: [u8; 32],
    signature: Option<[u8; 64]>,
}
```

---

# 12. Hash Algorithm

All hashes MUST use:

```
SHA-256
```

Hash inputs:

tensor_hash:

```
SHA256(tensor_data)
```

metadata_hash:

```
SHA256(metadata)
```

shard_hash:

```
SHA256(header + metadata + tensor + footer_without_signature)
```

---

# 13. Signature Specification

Optional signature MAY be present.

Signature algorithm:

```
Ed25519
```

Signature verifies shard authenticity.

Signature covers shard_hash.

---

# 14. Shard Identifier

Shard identifier MUST be:

```
SHA256(shard_hash)
```

Shard identifier MUST be globally unique.

---

# 15. Expert Assembly Structure

Expert definition:

```rust
struct ExpertDefinition {
    expert_id: [u8; 32],
    shard_ids: Vec<[u8; 32]>,
    tensor_layout: TensorLayout,
}
```

Expert assembled by concatenating shard tensors.

Assembly MUST be deterministic.

---

# 16. Tensor Layout Rules

Tensor layout MUST define:

- ordering of shards
- concatenation method
- dimension mapping

Tensor layout MUST be deterministic.

---

# 17. Shard Immutability

Shard files MUST NOT be modified after creation.

Modification invalidates hash.

Modified shards MUST be rejected.

---

# 18. Shard Verification Algorithm

Verification procedure:

```
Load shard
Verify magic
Verify version
Verify metadata hash
Verify tensor hash
Verify shard hash
Verify signature (if present)
```

If any verification fails, shard MUST be rejected.

---

# 19. Compression Rules

Compression MUST be lossless.

Supported compression:

```
Zstandard only
```

Compression MUST preserve binary equality after decompression.

---

# 20. Shard Storage Requirements

Shard files MAY be stored on:

- local disk
- network storage
- distributed storage

Shard integrity MUST be verified before use.

---

# 21. Forward Compatibility

Future versions MAY introduce new metadata fields.

Unknown metadata fields MUST be ignored.

Version field MUST be respected.

---

# 22. Summary

Shard format defines the atomic storage unit of intelligence.

Shard format ensures:

- deterministic assembly
- cryptographic integrity
- efficient storage
- cross-platform compatibility

All ARC implementations MUST conform to this shard format specification.