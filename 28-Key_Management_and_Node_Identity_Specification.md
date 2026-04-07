# Key Management and Node Identity Specification
## AURIA Engineering Specification Book
## Document 28 of 28
## Version: 1.0
## Status: Mandatory for Production Implementation

---

# 1. Purpose

This document defines the cryptographic identity, key management, storage, rotation, and recovery requirements for nodes operating the AURIA Runtime Core (ARC).

Each ARC node MUST possess a unique cryptographic identity.

Node identity enables:

- authentication
- license authorization
- settlement attribution
- execution attestation
- cluster membership

Node identity MUST be secure and persistent.

---

# 2. Identity Model Overview

Each ARC node has exactly one primary identity.

Identity consists of:

```
Ed25519 keypair
```

Structure:

```rust
struct NodeIdentity {
    public_key: [u8; 32],
    private_key: [u8; 64],
}
```

Public key uniquely identifies node.

Private key MUST remain secret.

---

# 3. Identity Generation Requirements

Identity MUST be generated using cryptographically secure RNG.

Required RNG:

```
Operating system secure RNG
```

Examples:

Linux:

```
/dev/urandom
```

Rust example:

```rust
use ed25519_dalek::Keypair;
use rand::rngs::OsRng;

let keypair = Keypair::generate(&mut OsRng);
```

Identity MUST be generated once.

Identity MUST persist across restarts.

---

# 4. Identity Storage Location

Identity MUST be stored at:

```
~/.auria/identity.key
```

File MUST contain:

private key  
public key  

File format:

binary or base64

Example JSON format:

```json
{
  "public_key": "...",
  "private_key": "..."
}
```

---

# 5. File Permission Requirements

Identity file MUST use secure permissions.

Required permission:

```
600
```

Owner read/write only.

Unauthorized access MUST be prevented.

---

# 6. Identity Loading Procedure

Runtime MUST load identity at startup.

Procedure:

```
Check identity file exists
If exists → load identity
If not exists → generate new identity
Save identity
```

Identity MUST NOT change automatically.

---

# 7. Identity Usage Requirements

Identity MUST be used for:

message signing  
license authorization  
settlement signing  
cluster authentication  

Identity MUST be used consistently.

---

# 8. Message Signing Requirements

All protocol messages MUST be signed.

Signing procedure:

```
payload_hash = SHA256(payload)
signature = Ed25519_sign(private_key, payload_hash)
```

Signature MUST accompany message.

---

# 9. Signature Verification Requirements

Verification procedure:

```
Verify signature using public key
Reject message if invalid
```

Invalid signatures MUST be rejected.

---

# 10. Identity Persistence Requirements

Identity MUST remain stable.

Changing identity results in:

loss of licenses  
loss of settlement attribution  

Identity persistence is critical.

---

# 11. Identity Backup Requirements

Node operators SHOULD backup identity file.

Backup MUST be stored securely.

Loss of identity results in loss of node ownership.

---

# 12. Identity Rotation Rules

Identity rotation MUST NOT occur automatically.

Identity rotation MAY occur manually.

Rotation procedure:

```
Generate new identity
Register new identity with network
```

Old identity MUST be retired.

---

# 13. Identity Revocation Model

Identity MAY be revoked by network authority.

Revoked identity MUST NOT be allowed to operate.

Revocation list MUST be enforced.

---

# 14. Key Storage Encryption (Optional)

Private key MAY be encrypted at rest.

Encryption recommended for:

enterprise deployments

Example encryption:

AES-256

---

# 15. Hardware Security Module (Optional)

Private keys MAY be stored in HSM.

HSM improves security.

HSM integration optional.

---

# 16. Identity Derivation Rules

Identity MUST NOT be derived from predictable input.

Forbidden:

password-derived identity  
machine-derived identity  

Identity MUST use secure randomness.

---

# 17. Cluster Identity Requirements

Cluster nodes MUST authenticate identity.

Cluster communication MUST verify identity.

Unauthorized nodes MUST be rejected.

---

# 18. License Binding Requirements

Licenses MUST bind to node identity.

License structure includes:

```
node_public_key
```

License valid only for specific node.

---

# 19. Settlement Binding Requirements

Settlement receipts MUST be signed by node identity.

Settlement attribution MUST use public key.

Public key identifies node ownership.

---

# 20. Identity Recovery Model

Identity recovery requires backup file.

Lost identity CANNOT be recovered without backup.

Protocol MUST NOT allow identity recovery without private key.

---

# 21. Identity Interface

Identity interface:

```rust
trait IdentityManager {
    fn load_identity() -> Result<NodeIdentity>;
    fn generate_identity() -> Result<NodeIdentity>;
    fn sign(data: &[u8]) -> Signature;
    fn verify(data: &[u8], signature: Signature, public_key: [u8; 32]) -> bool;
}
```

---

# 22. Identity Isolation Requirements

Private key MUST NOT be exposed to plugins.

Private key MUST NOT be exposed to external processes.

Private key access MUST be restricted.

---

# 23. Identity Validation Requirements

Identity MUST be validated at startup.

Invalid identity MUST prevent runtime startup.

---

# 24. Identity Uniqueness Requirements

Each node MUST have unique identity.

Duplicate identities MUST be rejected by network.

---

# 25. Identity Security Requirements

Private key MUST NEVER be transmitted.

Private key MUST NEVER be logged.

Private key MUST NEVER be exposed.

---

# 26. Identity Testing Requirements

Identity system MUST pass:

key generation tests  
signature verification tests  
identity persistence tests  

---

# 27. Identity Failure Handling

Identity loading failure MUST prevent startup.

Runtime MUST NOT operate without valid identity.

---

# 28. Summary

Identity system provides secure node authentication.

This specification ensures:

secure identity generation  
secure identity storage  
secure identity usage  

Identity is foundational to ARC security and operation.