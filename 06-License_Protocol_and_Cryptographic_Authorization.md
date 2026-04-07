# License Protocol and Cryptographic Authorization
## AURIA Engineering Specification Book
## Document 06 of 22
## Version: 1.0
## Status: Draft for Production Implementation

---

# 1. Purpose

This document defines the License Protocol and Cryptographic Authorization system for the AURIA Runtime Core (ARC).

The License Protocol ensures that shard execution is cryptographically authorized, verifiable, and economically enforceable.

This document specifies:

- License token format
- License issuance
- License verification
- License expiration and renewal
- License revocation
- Cluster license propagation
- Cryptographic enforcement rules

All shard execution MUST be license-authorized.

---

# 2. Definitions

## 2.1 License

A license is a cryptographically signed authorization permitting a node to execute a specific shard.

A license binds:

- shard identity
- node identity
- authorization conditions
- expiration

---

## 2.2 License Authority

A License Authority is an entity capable of issuing licenses.

Examples:

- shard owner
- DAO-controlled license contract
- delegated authority

---

## 2.3 Node Identity

Each node MUST possess a unique cryptographic identity.

Node identity is an Ed25519 public/private key pair.

Structure:

```rust
struct NodeIdentity {
    public_key: [u8; 32],
    private_key: [u8; 64],
}
```

---

# 3. License Token Format

License token structure:

```rust
struct LicenseToken {
    version: u32,
    license_id: [u8; 32],
    shard_id: [u8; 32],
    node_public_key: [u8; 32],
    issued_at: u64,
    expires_at: u64,
    issuer_public_key: [u8; 32],
    signature: [u8; 64],
}
```

LicenseToken MUST be serialized using canonical binary encoding.

---

# 4. License ID

License ID MUST be defined as:

```
SHA256(shard_id + node_public_key + issued_at)
```

License ID MUST be globally unique.

---

# 5. License Signature

Signature algorithm:

```
Ed25519
```

Signature covers:

```
version
license_id
shard_id
node_public_key
issued_at
expires_at
issuer_public_key
```

Signature verification MUST be performed before shard execution.

---

# 6. License Storage

License tokens MUST be stored locally.

License directory:

```
~/.auria/licenses/
```

License file name:

```
<license_id>.auria-license
```

---

# 7. License Verification Procedure

Verification steps:

```
Load license token
Verify signature
Verify shard_id matches shard
Verify node_public_key matches node identity
Verify current_time < expires_at
Verify issuer is trusted authority
```

If any step fails, license MUST be rejected.

---

# 8. Trusted License Authorities

Trusted authorities MUST be configured.

Trusted authorities file:

```
~/.auria/trusted_issuers.json
```

Structure:

```json
{
  "trusted_issuers": [
    "<issuer_public_key>"
  ]
}
```

Only trusted issuers may issue valid licenses.

---

# 9. License Expiration

License expiration enforced using expires_at field.

License MUST NOT be used if:

```
current_time ≥ expires_at
```

Expired license MUST be rejected.

---

# 10. License Renewal

Expired license MAY be renewed.

Renewal requires new license token.

Old license MUST NOT be reused.

---

# 11. License Revocation

License MAY be revoked.

Revocation requires:

- revocation list
- or expiration enforcement

Revocation list file:

```
~/.auria/revoked_licenses.json
```

Structure:

```json
{
  "revoked_licenses": [
    "<license_id>"
  ]
}
```

Revoked license MUST be rejected.

---

# 12. License Binding Rules

License MUST bind to:

- specific shard
- specific node

License MUST NOT be transferable.

License MUST NOT be reused by other nodes.

---

# 13. License Enforcement Point

License verification MUST occur before:

- shard assembly
- shard execution

License MUST be verified in License Manager.

Execution without valid license MUST be prohibited.

---

# 14. License Manager Interface

License Manager interface:

```rust
trait LicenseManager {
    fn verify_license(shard_id: [u8; 32]) -> Result<LicenseToken>;
    fn license_valid(shard_id: [u8; 32]) -> bool;
}
```

License Manager MUST be authoritative.

---

# 15. Cluster License Propagation

In cluster mode:

Coordinator MUST verify licenses.

Worker nodes MUST NOT execute shard without license authorization.

License propagation MUST occur securely.

License MUST be validated at worker node.

---

# 16. License Caching

Valid licenses MAY be cached.

Cache location:

```
~/.auria/license_cache/
```

Cache MUST respect expiration.

Cache MUST NOT bypass verification.

---

# 17. License Request Protocol

License request procedure:

```
Node requests license from authority
Authority verifies ownership conditions
Authority signs license
Node receives license
Node stores license
```

Transport may be:

- HTTP
- gRPC
- blockchain-based retrieval

---

# 18. License Integrity Requirements

License tokens MUST be immutable.

License modification invalidates signature.

Modified licenses MUST be rejected.

---

# 19. Failure Handling

License verification failure MUST result in:

- shard rejection
- execution abort

Runtime MUST NOT execute unauthorized shard.

---

# 20. Security Considerations

License protocol prevents:

- unauthorized shard execution
- shard theft
- license forgery

Security relies on cryptographic signature integrity.

Private keys MUST be protected.

---

# 21. Cryptographic Requirements

Required cryptographic primitives:

- SHA256
- Ed25519

No alternative algorithms permitted in version 1.

---

# 22. Summary

License protocol ensures shard execution authorization.

License protocol provides:

- cryptographic authorization
- execution control
- ownership enforcement

License Manager MUST enforce all license validation.

This subsystem guarantees authorized execution in AURIA Runtime Core.