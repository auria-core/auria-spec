# Rust Crate Architecture Specification
## AURIA Engineering Specification Book
## Document 23 of 28
## Version: 1.0
## Status: Mandatory for Production Implementation

---

# 1. Purpose

This document defines the canonical Rust crate architecture for the AURIA Runtime Core (ARC).

This specification ensures:

- deterministic module boundaries
- compile-time safety
- strict ownership control
- maintainable architecture
- predictable dependency graph

All Rust implementations of ARC MUST conform to this crate architecture.

This crate architecture is normative.

---

# 2. Design Principles

The crate architecture is designed according to the following principles:

Isolation  
Each crate has single responsibility.

Determinism  
Execution-critical crates have minimal dependencies.

Security  
Sensitive components are isolated.

Testability  
Each crate can be tested independently.

Extensibility  
Plugin and backend crates integrate cleanly.

---

# 3. Root Workspace Structure

Canonical workspace structure:

```
auria-runtime/
├── Cargo.toml
├── auria-core/
├── auria-config/
├── auria-execution/
├── auria-router/
├── auria-storage/
├── auria-license/
├── auria-security/
├── auria-network/
├── auria-settlement/
├── auria-observability/
├── auria-cluster/
├── auria-plugin/
├── auria-backend-cpu/
├── auria-backend-gpu/
├── auria-node/
└── auria-cli/
```

This structure MAY be preserved but MAY not necessarily exist.

---

# 4. Workspace Cargo.toml

Root Cargo.toml MAY define workspace but MAY not necessarily exist:

```toml
[workspace]
resolver = "2"

members = [
    "auria-core",
    "auria-config",
    "auria-execution",
    "auria-router",
    "auria-storage",
    "auria-license",
    "auria-security",
    "auria-network",
    "auria-settlement",
    "auria-observability",
    "auria-cluster",
    "auria-plugin",
    "auria-backend-cpu",
    "auria-backend-gpu",
    "auria-node",
    "auria-cli"
]
```

---

# 5. Crate Responsibilities

## 5.1 auria-core

Purpose:

Defines fundamental types and traits.

Contains:

- Tensor
- Shard
- Expert
- RuntimeVersion
- Error types

Dependencies:

NONE (foundational crate)

All other crates depend on auria-core.

---

## 5.2 auria-config

Purpose:

Manages configuration loading and validation.

Contains:

- config loader
- environment parser
- runtime parameter manager

Depends on:

auria-core

---

## 5.3 auria-execution

Purpose:

Implements expert assembly and execution pipeline.

Contains:

- execution engine
- tensor execution pipeline
- expert assembly logic

Depends on:

auria-core  
auria-router  
auria-storage  
auria-license  

---

## 5.4 auria-router

Purpose:

Implements deterministic routing.

Contains:

- routing logic
- expert selection

Depends on:

auria-core

---

## 5.5 auria-storage

Purpose:

Manages shard storage and retrieval.

Contains:

- shard loading
- shard verification
- cache integration

Depends on:

auria-core  
auria-security  

---

## 5.6 auria-license

Purpose:

Handles license verification.

Contains:

- license validation
- license cache

Depends on:

auria-core  
auria-security  

---

## 5.7 auria-security

Purpose:

Implements cryptographic verification.

Contains:

- hash verification
- signature verification
- attestation

Depends on:

auria-core

---

## 5.8 auria-network

Purpose:

Implements gRPC and HTTP networking.

Contains:

- HTTP server
- gRPC server
- protocol handlers

Depends on:

auria-core  
auria-execution  
auria-config  

---

## 5.9 auria-settlement

Purpose:

Handles usage accounting and settlement.

Contains:

- receipt generation
- Merkle tree builder
- settlement submission

Depends on:

auria-core  
auria-security  

---

## 5.10 auria-observability

Purpose:

Handles metrics and logging.

Contains:

- metrics collector
- logging system
- tracing

Depends on:

auria-core

---

## 5.11 auria-cluster

Purpose:

Handles cluster coordination.

Contains:

- worker management
- coordinator logic

Depends on:

auria-core  
auria-network  

---

## 5.12 auria-plugin

Purpose:

Handles plugin loading and management.

Contains:

- plugin loader
- plugin registry

Depends on:

auria-core

---

## 5.13 auria-backend-cpu

Purpose:

Implements CPU execution backend.

Depends on:

auria-core  
auria-execution  

---

## 5.14 auria-backend-gpu

Purpose:

Implements GPU execution backend.

Depends on:

auria-core  
auria-execution  

---

## 5.15 auria-node

Purpose:

Implements full runtime node.

Combines all subsystems.

Depends on:

ALL runtime crates.

---

## 5.16 auria-cli

Purpose:

Implements command-line interface.

Depends on:

auria-node  

---

# 6. Dependency Graph Rules

Mandatory dependency direction:

```
auria-core
   ↑
all subsystem crates
   ↑
auria-node
   ↑
auria-cli
```

Forbidden:

Subsystem crates MUST NOT depend on auria-node.

Circular dependencies are forbidden.

---

# 7. Public vs Private APIs

Each crate MUST define:

Public API:

```
pub trait ExecutionBackend
pub struct Tensor
```

Private implementation:

```
pub(crate)
```

Internal APIs MUST NOT be public.

---

# 8. Error Handling Architecture

All crates MUST use unified error type:

```rust
pub enum AuriaError
```

Defined in auria-core.

---

# 9. Feature Flags

GPU support MUST use feature flags:

```toml
[features]
gpu = []
cpu = []
```

---

# 10. Build Targets

Supported targets:

Linux x86_64  
Linux ARM64  
macOS  
Windows  

---

# 11. Binary Crates

Binary crates:

auria-node  
auria-cli  

All others MUST be library crates.

---

# 12. Plugin ABI Boundary

Plugin boundary MUST exist at auria-plugin crate.

Plugin crates MUST NOT access internal state.

---

# 13. Test Structure

Each crate MUST include:

```
tests/
src/
```

---

# 14. Security Isolation Requirements

Private keys MUST exist ONLY in:

auria-security

---

# 15. Determinism Isolation

Execution logic MUST exist ONLY in:

auria-execution  
auria-backend-cpu  
auria-backend-gpu  

---

# 16. Summary

This crate architecture ensures:

- deterministic execution
- modular structure
- secure implementation
- maintainable codebase

This structure is mandatory.