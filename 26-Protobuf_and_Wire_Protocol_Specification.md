# Protobuf and Wire Protocol Specification
## AURIA Engineering Specification Book
## Document 26 of 28
## Version: 1.0
## Status: Mandatory for Production Implementation

---

# 1. Purpose

This document defines the canonical Protocol Buffers (.proto) schemas and wire-level rules for AURIA Runtime Core (ARC) node communication.

This specification covers:

- gRPC service definitions
- message schemas
- authentication and signatures at the message level
- streaming semantics
- error codes
- version negotiation

All ARC implementations MUST use these schemas verbatim unless a MAJOR protocol version is introduced.

---

# 2. Protocol Versioning

## 2.1 Protocol Version Field

All messages MUST carry:

- `protocol_major`
- `protocol_minor`

Compatibility rules:

- `protocol_major` mismatch ⇒ connection MUST be rejected
- `protocol_minor` lower/higher ⇒ allowed if fields are forward/backward compatible

Current protocol version:

```
protocol_major = 1
protocol_minor = 0
```

---

# 3. Canonical Encoding Rules

- Protobuf encoding MUST use proto3
- All byte fields are raw bytes (no base64 in gRPC)
- Hashes are 32 bytes (SHA-256)
- Signatures are 64 bytes (Ed25519)
- Public keys are 32 bytes (Ed25519)

---

# 4. Error Code Registry

Error codes MUST be stable.

```proto
enum AuriaErrorCode {
  AURIA_ERROR_CODE_UNSPECIFIED = 0;

  // Auth / Identity
  AUTH_FAILED = 100;
  SIGNATURE_INVALID = 101;
  NONCE_INVALID = 102;
  VERSION_UNSUPPORTED = 103;

  // Shards / Storage
  SHARD_NOT_FOUND = 200;
  SHARD_CORRUPT = 201;
  SHARD_HASH_MISMATCH = 202;
  SHARD_DOWNLOAD_FAILED = 203;

  // License
  LICENSE_REQUIRED = 300;
  LICENSE_EXPIRED = 301;
  LICENSE_INVALID = 302;
  LICENSE_ISSUER_UNTRUSTED = 303;
  LICENSE_REVOKED = 304;

  // Routing / Execution
  ROUTING_FAILED = 400;
  TIER_NOT_SUPPORTED = 401;
  INSUFFICIENT_RESOURCES = 402;
  EXECUTION_FAILED = 403;
  BACKEND_UNAVAILABLE = 404;
  DETERMINISM_VIOLATION = 405;

  // Cluster
  WORKER_UNAVAILABLE = 500;
  ASSIGNMENT_TIMEOUT = 501;
  COORDINATOR_UNAVAILABLE = 502;

  // Settlement
  RECEIPT_REJECTED = 600;
  SETTLEMENT_SUBMISSION_FAILED = 601;

  // Internal
  INTERNAL_ERROR = 900;
}
```

---

# 5. Canonical Proto Package

Package name MUST be:

```
auria.runtime.v1
```

Language options:

- `go_package` optional
- Rust uses `tonic-build`

---

# 6. File: auria_runtime_v1.proto

```proto
syntax = "proto3";

package auria.runtime.v1;

option java_multiple_files = true;

message ProtocolVersion {
  uint32 protocol_major = 1;
  uint32 protocol_minor = 2;
}

message Signature {
  bytes public_key = 1;   // 32 bytes
  bytes signature = 2;    // 64 bytes
}

message SignedEnvelope {
  ProtocolVersion version = 1;

  // Anti-replay
  bytes nonce = 2;        // 32 bytes random
  uint64 timestamp_ms = 3;

  // Sender identity (public key) + signature over the canonical payload hash
  Signature signer = 4;

  // Hashing rule:
  // payload_hash = SHA256(payload_type || payload_bytes)
  // signer.signature signs payload_hash
  string payload_type = 5;
  bytes payload_bytes = 6;
}

message AuriaError {
  AuriaErrorCode code = 1;
  string message = 2;
  bytes details = 3; // optional structured bytes
}

enum Tier {
  TIER_UNSPECIFIED = 0;
  NANO = 1;
  STANDARD = 2;
  PRO = 3;
  MAX = 4;
}

message ModelSelector {
  string model_name = 1;    // e.g., "auria-standard"
  bytes model_id = 2;       // 32 bytes
  Tier tier = 3;
}

message TokenChunk {
  uint64 index = 1;
  bytes token = 2;          // token bytes (UTF-8 fragment or codec bytes)
  string text = 3;          // optional decoded text
  bool is_final = 4;
}

message ExecutionRequest {
  ModelSelector model = 1;

  // Client request id (opaque)
  string request_id = 2;

  // Canonical input representation:
  // For chat/completions use JSON payload (OpenAI-compatible) serialized as UTF-8 bytes.
  bytes openai_json = 3;

  // Optional: routing override controls
  uint32 top_k = 4;         // if 0, tier default
  bool stream = 5;

  // Determinism: client may provide seed; if 0, node chooses
  uint64 seed = 6;
}

message ExecutionResponse {
  string request_id = 1;
  uint64 created_ms = 2;

  // OpenAI-compatible JSON response bytes (UTF-8)
  bytes openai_json = 3;

  // Optional internal timings
  uint64 routing_time_us = 4;
  uint64 assembly_time_us = 5;
  uint64 execution_time_us = 6;
}

message HealthRequest {}

message HealthResponse {
  string status = 1;          // "ok"
  string runtime_version = 2; // "1.0.0"
  ProtocolVersion protocol = 3;
  repeated Tier enabled_tiers = 4;
}

service AuriaRuntimeService {
  // Unary execution (non-streaming)
  rpc Execute(SignedEnvelope) returns (SignedEnvelope);

  // Token streaming execution
  rpc ExecuteStream(SignedEnvelope) returns (stream SignedEnvelope);

  // Health/status
  rpc Health(HealthRequest) returns (HealthResponse);
}
```

---

# 7. File: auria_cluster_v1.proto

```proto
syntax = "proto3";

package auria.runtime.v1;

message CapabilityProfile {
  Tier max_tier = 1;
  uint64 total_vram_bytes = 2;
  uint64 total_ram_bytes = 3;
  uint32 cpu_cores = 4;
  bool cuda = 5;
  bool rocm = 6;
  bool metal = 7;
}

message NodeRegistration {
  string node_name = 1;
  bytes node_public_key = 2; // 32 bytes
  CapabilityProfile capabilities = 3;
}

message RegistrationResponse {
  bool accepted = 1;
  string reason = 2;
  bytes cluster_id = 3;       // 32 bytes
}

message HeartbeatRequest {
  bytes node_public_key = 1;
  uint64 timestamp_ms = 2;
}

message HeartbeatResponse {
  bool ok = 1;
}

message Assignment {
  uint64 assignment_id = 1;
  string request_id = 2;
  ModelSelector model = 3;

  // Experts the worker must execute
  repeated bytes expert_ids = 4;   // each 32 bytes

  // Input for the worker: opaque bytes (typically intermediate tensor encoding)
  bytes input_payload = 5;

  // Required: license bundle hash (worker verifies licenses locally)
  bytes license_bundle_hash = 6;   // 32 bytes
}

message AssignmentResult {
  uint64 assignment_id = 1;
  string request_id = 2;
  bytes output_payload = 3;
  uint64 execution_time_us = 4;
}

service AuriaClusterService {
  rpc RegisterWorker(SignedEnvelope) returns (SignedEnvelope);
  rpc Heartbeat(SignedEnvelope) returns (SignedEnvelope);
  rpc Assign(SignedEnvelope) returns (SignedEnvelope);
}
```

---

# 8. File: auria_license_v1.proto

```proto
syntax = "proto3";

package auria.runtime.v1;

message LicenseTokenProto {
  uint32 version = 1;
  bytes license_id = 2;        // 32 bytes
  bytes shard_id = 3;          // 32 bytes
  bytes node_public_key = 4;   // 32 bytes
  uint64 issued_at_ms = 5;
  uint64 expires_at_ms = 6;
  bytes issuer_public_key = 7; // 32 bytes
  bytes signature = 8;         // 64 bytes
}

message LicenseRequestProto {
  bytes shard_id = 1;
  bytes node_public_key = 2;
  bytes nonce = 3;             // anti-replay
}

message LicenseResponseProto {
  LicenseTokenProto license = 1;
}

service AuriaLicenseService {
  rpc RequestLicense(SignedEnvelope) returns (SignedEnvelope);
}
```

---

# 9. File: auria_settlement_v1.proto

```proto
syntax = "proto3";

package auria.runtime.v1;

message UsageEventProto {
  bytes event_id = 1;          // 32 bytes
  uint64 timestamp_ms = 2;
  bytes node_public_key = 3;   // 32 bytes
  bytes model_id = 4;          // 32 bytes
  bytes expert_id = 5;         // 32 bytes
  repeated bytes shard_ids = 6;// each 32 bytes
  bytes input_hash = 7;        // 32 bytes
  bytes output_hash = 8;       // 32 bytes
}

message UsageReceiptProto {
  bytes receipt_id = 1;         // 32 bytes
  repeated UsageEventProto events = 2;
  bytes node_public_key = 3;    // 32 bytes
  uint64 timestamp_ms = 4;
  bytes signature = 5;          // 64 bytes
}

message MerkleBatchProto {
  bytes batch_id = 1;           // 32 bytes
  repeated bytes receipt_hashes = 2; // each 32 bytes
  bytes merkle_root = 3;        // 32 bytes
  uint64 created_ms = 4;
}

message SettlementSubmitRequest {
  MerkleBatchProto batch = 1;
  repeated UsageReceiptProto receipts = 2;
}

message SettlementSubmitResponse {
  bool accepted = 1;
  string reason = 2;
}

service AuriaSettlementService {
  rpc Submit(SignedEnvelope) returns (SignedEnvelope);
}
```

---

# 10. SignedEnvelope Canonical Hashing Rules

All SignedEnvelope payloads MUST be verified as follows:

1. Parse SignedEnvelope.
2. Ensure `version.protocol_major == 1`.
3. Ensure `timestamp_ms` is within allowed clock skew (default ±120s).
4. Ensure `nonce` has not been seen before for the signer within window (default 10 minutes).
5. Compute:

```
payload_hash = SHA256(UTF8(payload_type) || payload_bytes)
```

6. Verify Ed25519 signature using `signer.public_key`.

---

# 11. Streaming Semantics

For `ExecuteStream`:

- each streamed `SignedEnvelope` MUST wrap a `TokenChunk` payload
- `TokenChunk.index` MUST be strictly increasing
- exactly one chunk MUST have `is_final = true`

---

# 12. Implementation Notes (Rust)

Recommended implementation stack:

- `tonic` for gRPC
- `prost` for protobuf
- `axum` or `warp` for HTTP
- `ed25519-dalek` for signatures
- `sha2` for hashing

---

# 13. Summary

This specification provides complete `.proto` schemas and wire rules for:

- runtime execution (unary + streaming)
- cluster coordination
- licensing
- settlement submission

All ARC nodes MUST implement these interfaces for interoperability.