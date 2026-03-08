# Settlement Protocol and Usage Accounting
## AURIA Engineering Specification Book
## Document 15 of 22
## Version: 1.0
## Status: Draft for Production Implementation

---

# 1. Purpose

This document defines the Settlement Protocol and Usage Accounting subsystem of the AURIA Runtime Core (ARC).

The settlement subsystem is implemented as the **AURIA Settlement Client (ASC)**.

This subsystem is responsible for:

- recording shard usage
- attributing expert usage
- generating usage receipts
- submitting settlement records
- enabling royalty distribution

Settlement MUST be deterministic, verifiable, and tamper-resistant.

Settlement MUST NOT interfere with execution latency.

Settlement occurs asynchronously.

---

# 2. Definitions

## 2.1 Usage Event

A usage event represents execution of an expert during inference.

Usage events MUST be recorded.

---

## 2.2 Usage Receipt

A usage receipt is a cryptographically signed record of execution.

Receipt MUST be verifiable.

Receipt MUST include all execution details.

---

## 2.3 Settlement Authority

Settlement authority processes usage receipts.

Settlement authority MAY be:

- blockchain contract
- settlement server
- distributed network

---

# 3. Usage Event Structure

Usage event structure:

```rust
struct UsageEvent {
    event_id: [u8; 32],
    timestamp: u64,
    node_identity: [u8; 32],
    model_id: [u8; 32],
    expert_id: [u8; 32],
    shard_ids: Vec<[u8; 32]>,
    input_hash: [u8; 32],
    output_hash: [u8; 32],
}
```

Usage event MUST be immutable.

---

# 4. Event ID Generation

Event ID MUST be generated using:

```
event_id = SHA256(node_identity + expert_id + timestamp)
```

Event ID MUST be unique.

---

# 5. Usage Receipt Structure

Usage receipt structure:

```rust
struct UsageReceipt {
    receipt_id: [u8; 32],
    usage_events: Vec<UsageEvent>,
    node_identity: [u8; 32],
    timestamp: u64,
    signature: [u8; 64],
}
```

Receipt MUST be signed.

Receipt MUST be verifiable.

---

# 6. Receipt Signature

Receipt MUST be signed using Ed25519.

Signature MUST cover entire receipt.

Signature MUST be verified before settlement.

---

# 7. Usage Recording Procedure

Usage recording procedure:

```
Execution completed
Generate usage event
Append usage event to receipt buffer
```

Recording MUST be asynchronous.

Recording MUST NOT block execution.

---

# 8. Receipt Buffering

Usage receipts MUST be buffered.

Buffer structure:

```rust
struct ReceiptBuffer {
    receipts: Vec<UsageReceipt>,
}
```

Buffer MUST be flushed periodically.

---

# 9. Merkle Tree Construction

Usage receipts MUST be aggregated into Merkle tree.

Merkle tree structure:

```rust
struct MerkleNode {
    hash: [u8; 32],
}
```

Merkle root MUST represent receipt set.

Merkle root MUST be deterministic.

---

# 10. Merkle Root Calculation

Merkle root procedure:

```
Hash each receipt
Pair hashes
Hash pairs
Repeat until single root
```

Merkle root MUST be reproducible.

---

# 11. Settlement Submission

Settlement submission procedure:

```
Compute Merkle root
Submit Merkle root to settlement authority
Transmit receipt batch
```

Submission MUST be authenticated.

Submission MUST be verifiable.

---

# 12. Settlement Timing

Settlement MUST occur periodically.

Default interval:

```
60 seconds
```

Interval MUST be configurable.

Settlement MUST NOT delay execution.

---

# 13. Royalty Attribution

Royalty attribution MUST use usage receipts.

Attribution MUST include:

- shard usage
- expert usage
- execution frequency

Royalty calculation MUST be deterministic.

---

# 14. Settlement Verification

Settlement authority MUST verify:

- receipt signatures
- Merkle root correctness
- event integrity

Invalid receipts MUST be rejected.

---

# 15. Receipt Persistence

Receipts MUST be persisted locally.

Receipt directory:

```
~/.auria/receipts/
```

Receipts MUST be immutable.

---

# 16. Settlement Failure Handling

Settlement failure MUST trigger retry.

Retry procedure:

```
Retry submission after delay
Log failure
```

Settlement MUST eventually succeed.

---

# 17. Duplicate Receipt Prevention

Duplicate receipts MUST be prevented.

Duplicate detection using receipt_id.

Duplicate receipts MUST be rejected.

---

# 18. Privacy Requirements

Receipts MUST NOT expose sensitive input data.

Receipts MUST include only hashes.

Sensitive data MUST NOT be transmitted.

---

# 19. Settlement Client Interface

Settlement client interface:

```rust
trait SettlementClient {
    fn record_usage(event: UsageEvent);
    fn generate_receipt() -> Result<UsageReceipt>;
    fn submit_receipt(receipt: UsageReceipt) -> Result<()>;
}
```

Settlement client MUST enforce accounting integrity.

---

# 20. Settlement Security Requirements

Settlement MUST use cryptographic verification.

Receipts MUST be signed.

Submission MUST be authenticated.

Tampering MUST be detectable.

---

# 21. Performance Requirements

Settlement overhead MUST be minimal.

Settlement MUST NOT increase execution latency.

Settlement MUST operate asynchronously.

---

# 22. Summary

Settlement protocol ensures economic attribution.

Settlement subsystem provides:

- usage accounting
- royalty attribution
- cryptographic verification
- settlement submission

Settlement protocol enables decentralized economic participation in ARC.