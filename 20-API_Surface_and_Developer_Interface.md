# API Surface and Developer Interface
## AURIA Engineering Specification Book
## Document 20 of 22
## Version: 1.0
## Status: Draft for Production Implementation

---

# 1. Purpose

This document defines the complete public API surface and developer interfaces of the AURIA Runtime Core (ARC).

This includes:

- HTTP REST APIs
- gRPC APIs
- Rust runtime interfaces
- plugin interfaces
- execution interfaces

These interfaces allow:

- client applications to use ARC
- developers to extend ARC
- nodes to interoperate

All APIs MUST be deterministic, secure, and versioned.

---

# 2. API Architecture Overview

ARC exposes three primary interface layers:

```
Interface Layers
├── Client API Layer (HTTP REST)
├── Runtime API Layer (Rust Interfaces)
└── Node API Layer (gRPC Protocol)
```

Each layer serves a different purpose.

---

# 3. HTTP REST API Overview

HTTP REST API provides client-facing interface.

All endpoints MUST use HTTPS.

All endpoints MUST support JSON.

Base URL:

```
https://<node_address>/v1/
```

---

# 4. Health Endpoint

Health endpoint provides runtime status.

Endpoint:

```
GET /v1/health
```

Response structure:

```json
{
  "status": "ok",
  "version": "1.0.0"
}
```

---

# 5. Chat Completions Endpoint

Endpoint:

```
POST /v1/chat/completions
```

Request structure:

```json
{
  "model": "auria-standard",
  "messages": [
    {"role": "user", "content": "Hello"}
  ],
  "stream": false
}
```

Response structure:

```json
{
  "id": "completion_id",
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "Hello."
      }
    }
  ]
}
```

Endpoint MUST be OpenAI-compatible.

---

# 6. Completions Endpoint

Endpoint:

```
POST /v1/completions
```

Request structure:

```json
{
  "model": "auria-standard",
  "prompt": "Hello"
}
```

Response structure:

```json
{
  "text": "Hello."
}
```

---

# 7. Embeddings Endpoint

Endpoint:

```
POST /v1/embeddings
```

Request structure:

```json
{
  "model": "auria-standard",
  "input": "Hello"
}
```

Response structure:

```json
{
  "embedding": [0.123, 0.456]
}
```

---

# 8. Version Endpoint

Endpoint:

```
GET /v1/version
```

Response:

```json
{
  "version": "1.0.0"
}
```

---

# 9. gRPC Service Overview

gRPC provides node-to-node communication.

Service definition:

```proto
service AuriaRuntime {
    rpc Execute(ExecutionRequest)
        returns (ExecutionResponse);

    rpc ExecuteStream(ExecutionRequest)
        returns (stream ExecutionStreamResponse);

    rpc RegisterNode(NodeRegistration)
        returns (RegistrationResponse);

    rpc Heartbeat(HeartbeatRequest)
        returns (HeartbeatResponse);
}
```

All services MUST be implemented.

---

# 10. ExecutionRequest Message

Structure:

```proto
message ExecutionRequest {
    bytes model_id = 1;
    bytes input_tensor = 2;
    uint32 tier = 3;
}
```

ExecutionRequest MUST be authenticated.

---

# 11. ExecutionResponse Message

Structure:

```proto
message ExecutionResponse {
    bytes output_tensor = 1;
    uint64 execution_time_us = 2;
}
```

ExecutionResponse MUST be signed.

---

# 12. Streaming Response Message

Structure:

```proto
message ExecutionStreamResponse {
    bytes token = 1;
    uint64 index = 2;
}
```

Streaming MUST preserve order.

---

# 13. Rust Runtime API Overview

Rust API provides internal developer interface.

Rust API MUST be stable.

Rust API MUST enforce determinism.

---

# 14. Execution Core Interface

Rust interface:

```rust
trait ExecutionCore {
    fn execute(
        request: ExecutionRequest
    ) -> Result<ExecutionResult>;
}
```

ExecutionCore MUST be authoritative execution interface.

---

# 15. Router Interface

Router interface:

```rust
trait Router {
    fn route(
        input: RoutingInput
    ) -> Result<RoutingDecision>;
}
```

Router MUST be deterministic.

---

# 16. Model Store Interface

Model store interface:

```rust
trait ModelStore {
    fn get_shard(
        shard_id: [u8; 32]
    ) -> Result<Shard>;
}
```

ModelStore MUST enforce shard integrity.

---

# 17. License Manager Interface

License manager interface:

```rust
trait LicenseManager {
    fn verify_license(
        shard_id: [u8; 32]
    ) -> Result<LicenseToken>;
}
```

LicenseManager MUST enforce license authorization.

---

# 18. Plugin Interface

Plugin interface enables extension.

Plugin interface:

```rust
trait AuriaPlugin {
    fn initialize();
    fn execute();
    fn shutdown();
}
```

Plugins MUST NOT violate determinism.

---

# 19. Authentication Requirements

All API calls MUST be authenticated.

Authentication methods:

- API key
- signature-based authentication
- mutual TLS

Unauthorized access MUST be rejected.

---

# 20. Error Handling

Error response structure:

```json
{
  "error": {
    "code": "execution_failed",
    "message": "Execution failed"
  }
}
```

Errors MUST be deterministic.

Errors MUST include error code.

---

# 21. API Versioning

API version MUST be included in path.

Example:

```
/v1/chat/completions
```

Future versions MUST use new version number.

---

# 22. Summary

API surface provides complete interface for ARC.

API subsystem enables:

- client interaction
- node communication
- developer integration

API surface ensures ARC is accessible and extensible.