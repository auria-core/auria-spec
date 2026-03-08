# Router and Sparse Expert Selection
## AURIA Engineering Specification Book
## Document 09 of 22
## Version: 1.0
## Status: Draft for Production Implementation

---

# 1. Purpose

This document defines the routing subsystem responsible for selecting experts for execution in the AURIA Runtime Core (ARC).

The routing subsystem is implemented as the **AURIA Router (AR)**.

Routing determines which experts participate in execution.

Routing MUST be:

- deterministic
- efficient
- tier-compliant
- verifiable

Routing directly impacts correctness and performance.

---

# 2. Definitions

## 2.1 Router

The router selects experts based on input state.

Router outputs expert IDs.

---

## 2.2 Expert Selection

Expert selection is the process of choosing experts from the expert pool.

Selection is sparse.

Only selected experts participate in execution.

---

## 2.3 Top-K Selection

Top-K selection chooses the K highest scoring experts.

K depends on tier.

---

## 2.4 Routing Function

Routing function maps input state to expert scores.

Structure:

```
RoutingFunction(input) → ExpertScores
```

---

# 3. Routing Architecture

Router consists of five components:

```
Router
├── Routing Function
├── Score Calculator
├── Top-K Selector
├── Load Balancer
└── Routing Verifier
```

Each component has defined responsibilities.

---

# 4. Routing Input

Routing input structure:

```rust
struct RoutingInput {
    model_id: [u8; 32],
    token_index: u64,
    input_vector: Tensor,
}
```

Input vector MUST be deterministic.

---

# 5. Expert Score Calculation

Router MUST compute score for each expert.

Score function:

```
score = dot(input_vector, expert_gate_vector)
```

Gate vector stored in routing shard.

Score MUST be deterministic.

Score MUST use consistent precision.

---

# 6. Score Representation

Scores represented as:

```rust
struct ExpertScore {
    expert_id: [u8; 32],
    score: f32,
}
```

Floating-point precision MUST be consistent across hardware.

FP32 required for routing.

---

# 7. Top-K Selection

Top-K selection procedure:

```
Sort experts by score descending
Select top K experts
```

K determined by tier:

Nano: 4  
Standard: 8  
Pro: 16  
Max: 32 or greater

---

# 8. Top-K Determinism Requirements

Sorting MUST be deterministic.

Tie-breaking MUST use expert_id.

Tie-break rule:

```
Lower expert_id wins tie
```

Sorting MUST NOT depend on:

- hardware
- thread timing
- floating-point nondeterminism

---

# 9. Routing Output

Routing output structure:

```rust
struct RoutingDecision {
    selected_experts: Vec<[u8; 32]>,
    scores: Vec<f32>,
}
```

Output MUST be deterministic.

---

# 10. Tier Enforcement

Router MUST enforce tier limits.

Router MUST NOT select more experts than allowed.

Router MUST query Tier Manager.

Violation MUST cause routing failure.

---

# 11. Routing Shard Format

Routing shard contains gate vectors.

Structure:

```rust
struct RoutingShard {
    model_id: [u8; 32],
    expert_ids: Vec<[u8; 32]>,
    gate_vectors: Tensor,
}
```

Routing shard MUST be verified before use.

---

# 12. Routing Determinism

Routing MUST produce identical results given identical inputs.

Determinism MUST NOT depend on:

- thread scheduling
- hardware type
- execution timing

Routing MUST use fixed precision math.

---

# 13. Load Balancing

Load balancing MAY be applied.

Load balancing modifies scores:

```
adjusted_score = score − load_penalty
```

Load penalty based on expert utilization.

Load balancing MUST be deterministic.

---

# 14. Load Balancing State

Load state structure:

```rust
struct LoadState {
    expert_usage_count: HashMap<[u8; 32], u64>,
}
```

Load state MUST be updated atomically.

---

# 15. Routing Verification

Routing verification ensures correctness.

Verification procedure:

```
Verify routing shard integrity
Verify score computation
Verify Top-K selection
Verify tier compliance
```

Verification MUST occur before execution.

---

# 16. Routing Cache

Routing results MAY be cached.

Cache structure:

```rust
struct RoutingCacheEntry {
    input_hash: [u8; 32],
    routing_decision: RoutingDecision,
}
```

Cache MUST preserve determinism.

---

# 17. Routing Failure Handling

Routing failure conditions:

- routing shard missing
- routing shard invalid
- insufficient experts
- tier violation

Failure MUST abort execution.

---

# 18. Routing Performance Requirements

Routing time targets:

Nano:

≤ 100 microseconds

Standard:

≤ 50 microseconds

Pro:

≤ 25 microseconds

Max:

≤ 10 microseconds

Routing MUST be highly optimized.

---

# 19. Router Interface

Router interface:

```rust
trait Router {
    fn route(input: RoutingInput) -> Result<RoutingDecision>;
}
```

Router MUST be deterministic.

Router MUST enforce tier limits.

---

# 20. Routing State Persistence

Routing state MAY be persisted.

File:

```
~/.auria/routing_state.json
```

State persistence improves performance.

---

# 21. Security Requirements

Routing MUST NOT be modifiable without authorization.

Routing shards MUST be verified.

Routing integrity MUST be enforced.

---

# 22. Summary

Router determines expert selection.

Router ensures:

- deterministic expert selection
- tier compliance
- efficient execution
- verifiable routing

Router is essential for sparse execution in ARC.