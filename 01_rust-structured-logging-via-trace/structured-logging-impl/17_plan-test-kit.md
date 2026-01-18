# Test-Kit Structured Logging Implementation Plan

Based on the deterministic pattern from `02_plan-general.md`, this document provides a concrete, step-by-step implementation guide specific to the `test-kit` crate.

---

## Overview

The `test-kit` crate is a lightweight helper module for integration tests. It contains async functions for transaction execution, simulation, and replay. Total files to update: **2 files** with logging statements.

### File Categories

1. **Core test utilities** (`lib.rs`): Main `ExecutionTestEnv` struct with async transaction methods
2. **Test macros and helpers** (`macros.rs`): Logger initialization and devnet availability check

---

## Current State Analysis

### Async Functions with Logging

**In `lib.rs`**:
1. `execute_transaction()` - Line 267-275
2. `schedule_transaction()` - Line 278-283
3. `simulate_transaction()` - Line 286-299
4. `replay_transaction()` - Line 302-310

**In `macros.rs`**:
1. `is_devnet_up()` - Line 91-96 (async, no logging)
2. Macro `skip_if_devnet_down!` - Line 99-106 (has warn! call)

### Current Log Statements

**`lib.rs`**:
```rust
// Line 274 (in execute_transaction)
.inspect_err(|err| error!("failed to execute transaction: {err}"))

// Line 296 (in simulate_transaction)
error!("failed to simulate transaction: {err}")

// Line 309 (in replay_transaction)
.inspect_err(|err| error!("failed to replay transaction: {err}"))
```

**`macros.rs`**:
```rust
// Line 102 (in skip_if_devnet_down! macro)
::tracing::warn!("Devnet is down, skipping test");
```

---

## Standard Field Naming for Test-Kit

| Field | Type | Format | Use Case |
|-------|------|--------|----------|
| `error` | Error | `?` | Transaction execution/simulation error |
| `operation` | &str | bare | Operation type (execute, simulate, replay) |
| `result_error` | &str | bare | Error description from result |

---

## File-by-File Implementation Plan

### File 1: `src/lib.rs`

**Priority**: 1 (Main file, all async functions have logging)

**Async functions to instrument**:

1. **`execute_transaction()`** (Lines 267-275)
2. **`schedule_transaction()`** (Lines 278-283)
3. **`simulate_transaction()`** (Lines 286-299)
4. **`replay_transaction()`** (Lines 302-310)

#### Implementation for `execute_transaction()`

**Current code** (Lines 267-275):
```rust
pub async fn execute_transaction(
    &self,
    txn: impl SanitizeableTransaction,
) -> TransactionResult {
    self.transaction_scheduler
        .execute(txn)
        .await
        .inspect_err(|err| error!("failed to execute transaction: {err}"))
}
```

**After**:
```rust
#[instrument(skip(self, txn))]
pub async fn execute_transaction(
    &self,
    txn: impl SanitizeableTransaction,
) -> TransactionResult {
    self.transaction_scheduler
        .execute(txn)
        .await
        .inspect_err(|err| error!(error = ?err, "Transaction execution failed"))
}
```

**Rationale**:
- `skip(self, txn)` because both are test implementation details
- Constant message: "Transaction execution failed"
- Extract error to field for queryability
- No context fields needed in this span since this is a thin delegation

---

#### Implementation for `schedule_transaction()`

**Current code** (Lines 278-283):
```rust
pub async fn schedule_transaction(
    &self,
    txn: impl SanitizeableTransaction,
) {
    self.transaction_scheduler.schedule(txn).await.unwrap();
}
```

**After**:
```rust
#[instrument(skip(self, txn))]
pub async fn schedule_transaction(
    &self,
    txn: impl SanitizeableTransaction,
) {
    self.transaction_scheduler.schedule(txn).await.unwrap();
}
```

**Rationale**:
- Add `#[instrument]` for consistency with other transaction methods
- No logging currently, but span wraps the async work
- If the `.unwrap()` panics, the span context will help identify which test failed
- `skip(self, txn)` to avoid capturing test implementation details

---

#### Implementation for `simulate_transaction()`

**Current code** (Lines 286-299):
```rust
pub async fn simulate_transaction(
    &self,
    txn: impl SanitizeableTransaction,
) -> TransactionSimulationResult {
    let result = self
        .transaction_scheduler
        .simulate(txn)
        .await
        .expect("transaction executor has shutdown during test");
    if let Err(ref err) = result.result {
        error!("failed to simulate transaction: {err}")
    }
    result
}
```

**After**:
```rust
#[instrument(skip(self, txn))]
pub async fn simulate_transaction(
    &self,
    txn: impl SanitizeableTransaction,
) -> TransactionSimulationResult {
    let result = self
        .transaction_scheduler
        .simulate(txn)
        .await
        .expect("transaction executor has shutdown during test");
    if let Err(ref err) = result.result {
        error!(error = ?err, "Transaction simulation failed");
    }
    result
}
```

**Rationale**:
- `skip(self, txn)` same reasoning as above
- Rewrite error log to constant message with error as field
- Remove inline error interpolation, extract to `error` field

---

#### Implementation for `replay_transaction()`

**Current code** (Lines 302-310):
```rust
pub async fn replay_transaction(
    &self,
    txn: impl SanitizeableTransaction,
) -> TransactionResult {
    self.transaction_scheduler
        .replay(txn)
        .await
        .inspect_err(|err| error!("failed to replay transaction: {err}"))
}
```

**After**:
```rust
#[instrument(skip(self, txn))]
pub async fn replay_transaction(
    &self,
    txn: impl SanitizeableTransaction,
) -> TransactionResult {
    self.transaction_scheduler
        .replay(txn)
        .await
        .inspect_err(|err| error!(error = ?err, "Transaction replay failed"))
}
```

**Rationale**:
- Same pattern as `execute_transaction()`
- Constant message: "Transaction replay failed"
- Extract error to field

---

### File 2: `src/macros.rs`

**Priority**: 2 (After lib.rs, contains helper functions)

**Async functions to instrument**:

1. **`is_devnet_up()`** (Lines 91-96) - No logging in function itself
2. **`skip_if_devnet_down!` macro** (Lines 99-106) - Has warn! call

#### Implementation for `is_devnet_up()`

**Current code** (Lines 91-96):
```rust
pub async fn is_devnet_up() -> bool {
    RpcClient::new("https://api.devnet.solana.com".to_string())
        .get_version()
        .await
        .is_ok()
}
```

**After**:
```rust
#[instrument]
pub async fn is_devnet_up() -> bool {
    RpcClient::new("https://api.devnet.solana.com".to_string())
        .get_version()
        .await
        .is_ok()
}
```

**Rationale**:
- Add `#[instrument]` for span context around async operation
- No parameters to skip (function takes no self/references)
- No logging in function, but span helps trace devnet connectivity checks
- Minimal attribute needed

---

#### Implementation for `skip_if_devnet_down!` macro

**Current code** (Lines 99-106):
```rust
#[macro_export]
macro_rules! skip_if_devnet_down {
    () => {
        if !$crate::macros::is_devnet_up().await {
            ::tracing::warn!("Devnet is down, skipping test");
            return;
        }
    };
}
```

**After**:
```rust
#[macro_export]
macro_rules! skip_if_devnet_down {
    () => {
        if !$crate::macros::is_devnet_up().await {
            ::tracing::warn!("Devnet is down, skipping test");
            return;
        }
    };
}
```

**Rationale**:
- **Message is already constant**: "Devnet is down, skipping test"
- No parameters in the message
- No field extraction needed
- Already in correct format
- **NO CHANGES NEEDED** - This is already structured correctly

The warn! call in the macro is already properly structured:
- Constant message (no interpolation)
- No parameters to extract
- Appropriate log level (warn for skipped test)

---

## Before/After Examples Summary

### Example 1: `execute_transaction()` - Simplest case

**Before**:
```rust
pub async fn execute_transaction(
    &self,
    txn: impl SanitizeableTransaction,
) -> TransactionResult {
    self.transaction_scheduler
        .execute(txn)
        .await
        .inspect_err(|err| error!("failed to execute transaction: {err}"))
}
```

**After**:
```rust
#[instrument(skip(self, txn))]
pub async fn execute_transaction(
    &self,
    txn: impl SanitizeableTransaction,
) -> TransactionResult {
    self.transaction_scheduler
        .execute(txn)
        .await
        .inspect_err(|err| error!(error = ?err, "Transaction execution failed"))
}
```

**Changes**:
- Add `#[instrument(skip(self, txn))]` attribute
- Extract error from message format to field: `error = ?err`
- Make message constant: "Transaction execution failed"

---

### Example 2: `simulate_transaction()` - Error logging in body

**Before**:
```rust
pub async fn simulate_transaction(
    &self,
    txn: impl SanitizeableTransaction,
) -> TransactionSimulationResult {
    let result = self
        .transaction_scheduler
        .simulate(txn)
        .await
        .expect("transaction executor has shutdown during test");
    if let Err(ref err) = result.result {
        error!("failed to simulate transaction: {err}")
    }
    result
}
```

**After**:
```rust
#[instrument(skip(self, txn))]
pub async fn simulate_transaction(
    &self,
    txn: impl SanitizeableTransaction,
) -> TransactionSimulationResult {
    let result = self
        .transaction_scheduler
        .simulate(txn)
        .await
        .expect("transaction executor has shutdown during test");
    if let Err(ref err) = result.result {
        error!(error = ?err, "Transaction simulation failed");
    }
    result
}
```

**Changes**:
- Add `#[instrument(skip(self, txn))]` attribute
- Extract error to field in error! call: `error = ?err`
- Make message constant: "Transaction simulation failed"

---

### Example 3: `schedule_transaction()` - No logging (only span needed)

**Before**:
```rust
pub async fn schedule_transaction(
    &self,
    txn: impl SanitizeableTransaction,
) {
    self.transaction_scheduler.schedule(txn).await.unwrap();
}
```

**After**:
```rust
#[instrument(skip(self, txn))]
pub async fn schedule_transaction(
    &self,
    txn: impl SanitizeableTransaction,
) {
    self.transaction_scheduler.schedule(txn).await.unwrap();
}
```

**Changes**:
- Add `#[instrument(skip(self, txn))]` attribute for consistency
- No logging changes needed
- Span wraps the async work for context

---

## Implementation Order

Process files in this exact order:

1. **`src/lib.rs`** - 4 async functions to instrument
2. **`src/macros.rs`** - 1 async function to instrument, 1 macro that's already correct

---

## Validation Steps

### 1. Syntax Check
```bash
cd test-kit
cargo check
```

### 2. Lint
```bash
make lint
```

### 3. Tests
```bash
cargo nextest run -p test-kit
```

### 4. Log Inspection (Optional)
```bash
RUST_LOG=debug cargo test --package test-kit --lib 2>&1 | head -100
```

**Verify**:
- Each async function creates one top-level span (name derived from function name)
- Log messages are constant (no variables in message text)
- Fields appear in span context
- No duplicate information (in both message AND field)

---

## Checklist

### Pre-Implementation
- [ ] Read this plan fully
- [ ] Understand the 4 async functions in lib.rs
- [ ] Understand skip_if_devnet_down! macro pattern

### During Implementation
- [ ] Add `#[instrument(skip(self, txn))]` to `execute_transaction()`
- [ ] Rewrite error message: `error!(error = ?err, "Transaction execution failed")`
- [ ] Add `#[instrument(skip(self, txn))]` to `schedule_transaction()`
- [ ] Add `#[instrument(skip(self, txn))]` to `simulate_transaction()`
- [ ] Rewrite error message: `error!(error = ?err, "Transaction simulation failed")`
- [ ] Add `#[instrument(skip(self, txn))]` to `replay_transaction()`
- [ ] Rewrite error message: `error!(error = ?err, "Transaction replay failed")`
- [ ] Add `#[instrument]` to `is_devnet_up()`
- [ ] Verify `skip_if_devnet_down!` macro needs no changes

### Post-Implementation

#### You must run these commands and verify:

- [ ] `cargo check` passes with no errors
- [ ] `make lint` passes with no clippy warnings
- [ ] `cargo nextest run -p test-kit` passes (all tests pass)
- [ ] All error messages are constant (no format strings)
- [ ] All error details are in fields (not message text)
- [ ] All async functions have `#[instrument]` attribute
- [ ] No manual spans exist in code

#### Verification Checklist:

- [ ] `src/lib.rs`: All 4 transaction methods have `#[instrument]`
- [ ] `src/macros.rs`: `is_devnet_up()` has `#[instrument]`
- [ ] `src/lib.rs`: 3 error! calls converted to structured format
- [ ] `src/macros.rs`: warn! call in macro unchanged (already correct)
- [ ] Build succeeds: `cargo build`
- [ ] Lint succeeds: `make lint`
- [ ] Tests pass: `cargo nextest run -p test-kit`

---

## Summary

The test-kit crate is small and focused:

- **2 files** with async functions
- **5 async functions** total (4 in lib.rs, 1 in macros.rs)
- **3 error! log statements** to rewrite
- **1 warn! in macro** already correct (no changes)

**Work required**:

1. Add `#[instrument]` to all 5 async functions
2. Rewrite 3 error! calls with structured fields
3. Verify macro logging is already correct
4. Run tests and lint

**Expected outcome**: Structured, queryable logging for all transaction execution paths in test environment.
