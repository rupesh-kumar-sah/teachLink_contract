/* Authorized Protocol Quality Assurance & Formal Verification Test Suite */

# Smart Contract Security Hardening Guide

This document establishes the defensive security engineering standards, formal verification guidelines, and pre-deployment hardening protocols for TeachLink Soroban smart contracts.

---

## 1. Pre-Deployment Security Checklist

Every contract modification and pull request must satisfy this verification checklist before testnet or mainnet deployment:

- [ ] **Authorization Verification**: Every mutating entry point enforces `caller.require_auth()` or explicit multisig/role-based checks.
- [ ] **Checks-Effects-Interactions (CEI)**: Internal state updates are persisted to storage *before* invoking external contracts or token transfers.
- [ ] **Arithmetic Bound Checks**: All integer calculations utilize `checked_add`, `checked_sub`, `checked_mul`, or `checked_div` with explicit `Result<T, Error>` error mapping.
- [ ] **Storage TTL & Rent Management**: Critical persistent entries employ automated `extend_ttl` calls to guarantee record availability across active epochs.
- [ ] **Zero-Value & Boundary Validation**: Amounts, durations, thresholds, and array bounds are strictly asserted against zero and maximum limits.
- [ ] **Administrative Handover**: Privileged admin transfers follow a two-step propose-and-claim ceremony (`transfer_admin` -> `claim_admin`).
- [ ] **Event Emission**: All security-relevant state transitions publish structured Soroban events with indexed topics for audit observability.
- [ ] **Regression & Invariant Testing**: Automated unit and property-based tests achieve 100% branch coverage across all error variants.

---

## 2. Soroban Architectural Best Practices

### A. Strict Authorization Enforcement
Soroban executes transactions with granular cryptographic authorization.
```rust
// Guard every state-altering method with appropriate authorization
pub fn execute_action(env: Env, caller: Address, amount: i128) -> Result<(), Error> {
    caller.require_auth();
    // Validate inputs after verifying caller identity
    if amount <= 0 {
        return Err(Error::InvalidAmount);
    }
    // Proceed with state updates...
    Ok(())
}
```

### B. Checks-Effects-Interactions (CEI) Pattern
To prevent reentrancy and state desynchronization during cross-contract token transfers:
1. **Check**: Validate caller authorization, match/course state, and parameters.
2. **Effect**: Mutate contract storage, update balances, and extend TTL.
3. **Interact**: Invoke the external token client `transfer` or cross-contract call.

```rust
// CORRECT: Effects before interactions
m.state = CourseState::Completed;
env.storage().persistent().set(&DataKey::Course(id), &m);

// External interaction happens last
token_client.transfer(&contract_address, &instructor, &payout);
```

### C. Safe Arithmetic & Overflow Immunity
Native Rust debug builds panic on overflow, whereas release builds wrap. Always use checked arithmetic explicitly:
```rust
let total_payout = base_fee
    .checked_add(bonus)
    .ok_or(Error::ArithmeticOverflow)?;
```

---

## 3. Vulnerability Classifications & Mitigations

### 1. State Desynchronization (Cross-Contract Divergence)
- **Risk**: External call failure or intermediate revert leaving state partially committed.
- **Mitigation**: Rely on Soroban's atomic transaction rollback. Never suppress errors from external token clients using `let _ = client.transfer(...)`; always propagate errors with `?`.

### 2. Unauthorized Caller & Privilege Escalation
- **Risk**: Adverse callers invoking administrative methods (e.g. fee adjustment, pause, upgrade).
- **Mitigation**: Store admin in instance storage and assert `admin.require_auth()` on privileged entry points. Implement two-step admin transfer to prevent accidental bricking from typo addresses.

### 3. Replay Attacks in Delegated Signatures
- **Risk**: Reusing an authorization signature across multiple transactions or chains.
- **Mitigation**: Incorporate nonces, contract IDs, and chain network passphrases (`env.ledger().network_id()`) into every signed message envelope.

### 4. Storage Eviction & TTL Expiration
- **Risk**: Long-lived records (escrow balances, certificates, student registries) getting evicted due to expired storage rent.
- **Mitigation**: Automatically extend TTL on every read/write operation:
```rust
env.storage().persistent().extend_ttl(&key, MIN_PERSISTENT_TTL, MAX_PERSISTENT_TTL);
```

### 5. Adverse Front-Running / State Race Conditions
- **Risk**: External callers observing pending transactions and front-running price-sensitive actions.
- **Mitigation**: Implement commitment-reveal schemes, deadline checks (`deadline < env.ledger().sequence()`), and minimum output boundaries (`min_tokens_out`).

---

## 4. Formal Verification & Invariant Testing Standard

All protocol critical paths must be supported by Foundry/Soroban regression suites validating invariant consistency:

| Invariant ID | Description | Target Invariant |
| :--- | :--- | :--- |
| `INV_SOLVENCY` | Contract token reserve must always equal sum of all uncollected deposits | $\sum \text{escrow\_balances} \le \text{contract\_reserve}$ |
| `INV_SINGLE_CLAIM` | No participant can claim payout or certificate more than once | $\text{claim\_count}(id) \le 1$ |
| `INV_MONOTONIC_SEQ` | Nonce and sequence counters must strictly increase | $\text{nonce}_{t+1} > \text{nonce}_t$ |
| `INV_ADMIN_IMMUTABILITY` | Admin cannot be set to null or unowned address without claim verification | $\text{admin} \ne \text{Address::zero()}$ |

---

*This guide is maintained under the Protocol Quality Assurance standards. Review and update after every major architectural upgrade.*
