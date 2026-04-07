# Network Protocol and Node Communication
## AURIA Engineering Specification Book
## Document 13 of 22
## Version: 1.0
## Status: Draft for Production Implementation

---

# 1. Purpose

This document defines the network protocol and node communication interfaces for the AURIA Runtime Core (ARC).

The network subsystem enables:

- client-to-node communication
- node-to-node communication
- cluster coordination
- license acquisition
- shard discovery

This document specifies:

- transport protocols
- message formats
- authentication requirements
- API interfaces
- streaming protocol
- error handling

Network communication MUST be secure, deterministic, and verifiable.

---

# 2. Transport Protocols

ARC MUST support the following transport protocols:

Primary protocol:

```
gRPC over HTTP/2 with TLS
```

Secondary protocol:

```
HTTPS REST API
```

Cluster internal protocol:

```
gRPC over QUIC (optional)
```

All network communication MUST use encryption.

---

# 3. Security Requirements

All communication MUST use:

- TLS 1.3 or higher
- mutual authentication
- certificate validation

Nodes MUST authenticate each other.

Unauthorized connections MUST be rejected.

---

# 4. Node Identity and Authentication

Each node MUST possess:

```rust
struct NodeIdentity {
    public_key: [u8; 32],
    private_key: [u8; 64],
}
```

Authentication MUST use Ed25519 signatures.

Handshake procedure:

```
Client → Node: HandshakeRequest
Node → Client: HandshakeResponse
Client → Node: SignatureVerification
```

Connection MUST NOT proceed without authentication.

---

# 5. Message Format

All messages MUST use canonical binary encoding.

Message structure:

```rust
struct MessageHeader {
    version: u32,
    message_type: MessageType,
    message_id: u64,
    timestamp: u64,
    sender_identity: [u8; 32],
    signature: [u8; 64],
}
```

Message body follows header.

Message MUST be signed.

---

# 6. Message Types

Defined message types:

```rust
enum MessageType {
    ExecutionRequest,
    ExecutionResponse,
    ShardRequest,
    ShardResponse,
    LicenseRequest,
    LicenseResponse,
    RegistrationRequest,
    RegistrationResponse,
    Heartbeat,
    Error,
}
```

All message types MUST be supported.

---

# 7. Execution Request Protocol

Execution request message:

```rust
struct ExecutionRequestMessage {
    header: MessageHeader,
    model_id: [u8; 32],
    input_tensor: Tensor,
    tier: Tier,
}
```

Execution request MUST be authenticated.

Execution request MUST be verified before execution.

---

# 8. Execution Response Protocol

Execution response message:

```rust
struct ExecutionResponseMessage {
    header: MessageHeader,
    output_tensor: Tensor,
    execution_time_us: u64,
}
```

Response MUST be signed.

Response MUST include execution time.

---

# 9. Streaming Protocol

Streaming MUST support token-by-token output.

Streaming uses gRPC streaming.

Stream message structure:

```rust
struct StreamMessage {
    token: Vec<u8>,
    token_index: u64,
}
```

Streaming MUST preserve order.

Streaming MUST be deterministic.

---

# 10. REST API Specification

ARC MUST provide REST API.

Endpoints:

```
POST /v1/chat/completions
POST /v1/completions
POST /v1/embeddings
GET /v1/health
```

REST API MUST use HTTPS.

REST API MUST support JSON.

---

# 11. gRPC Service Definition

gRPC service structure:

```proto
service AuriaRuntime {
    rpc Execute(ExecutionRequestMessage)
        returns (ExecutionResponseMessage);

    rpc ExecuteStream(ExecutionRequestMessage)
        returns (stream StreamMessage);

    rpc RegisterNode(RegistrationRequest)
        returns (RegistrationResponse);
}
```

All services MUST be implemented.

---

# 12. Heartbeat Protocol

Heartbeat messages ensure node availability.

Heartbeat message:

```rust
struct HeartbeatMessage {
    header: MessageHeader,
    node_status: NodeStatus,
}
```

Heartbeat interval:

```
5 seconds
```

Missing heartbeat indicates failure.

---

# 13. Error Protocol

Error message structure:

```rust
struct ErrorMessage {
    header: MessageHeader,
    error_code: ErrorCode,
    error_message: String,
}
```

Error MUST be signed.

Error MUST include error code.

---

# 14. Shard Request Protocol

Shard request message:

```rust
struct ShardRequestMessage {
    header: MessageHeader,
    shard_id: [u8; 32],
}
```

Shard response message:

```rust
struct ShardResponseMessage {
    header: MessageHeader,
    shard_data: Vec<u8>,
}
```

Shard MUST be verified before use.

---

# 15. License Request Protocol

License request message:

```rust
struct LicenseRequestMessage {
    header: MessageHeader,
    shard_id: [u8; 32],
    node_identity: [u8; 32],
}
```

License response message:

```rust
struct LicenseResponseMessage {
    header: MessageHeader,
    license_token: LicenseToken,
}
```

License MUST be verified before execution.

---

# 16. Connection Lifecycle

Connection lifecycle:

```
Connection initiated
Handshake performed
Authentication completed
Message exchange begins
Connection maintained with heartbeats
Connection closed safely
```

Connection MUST be authenticated.

---

# 17. Connection Limits

Node MUST limit concurrent connections.

Default limit:

```
1000 connections
```

Connection limit MUST be configurable.

---

# 18. Message Integrity Requirements

All messages MUST be signed.

Signature MUST be verified.

Invalid signature MUST result in rejection.

---

# 19. Network Failure Handling

Network failures MUST be handled safely.

Failure handling procedure:

```
Detect failure
Retry connection
Fallback to alternative node
```

Failure MUST NOT corrupt runtime state.

---

# 20. Protocol Versioning

Protocol version MUST be included in message header.

Unsupported version MUST be rejected.

Versioning enables compatibility.

---

# 21. Network Interface

Network interface:

```rust
trait NetworkInterface {
    fn send(message: Message) -> Result<()>;
    fn receive() -> Result<Message>;
    fn connect(address: String) -> Result<Connection>;
}
```

Network interface MUST enforce protocol rules.

---

# 22. Summary

Network protocol enables secure communication.

Network subsystem provides:

- secure node communication
- client interface support
- cluster coordination
- license and shard retrieval

Network protocol is essential for distributed ARC operation.