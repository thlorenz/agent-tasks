# Magicblock-RPC-Client Structured Logging Implementation Plan

Based on the deterministic pattern from `02_plan-general.md`, this document provides a concrete, step-by-step implementation guide specific to the `magicblock-rpc-client` crate.

---

## Overview

The `magicblock-rpc-client` crate is a thin wrapper around Solana's `RpcClient` providing improved transaction sending, confirmation, and status checking functionality. It contains **2 files** with logging statements.

### Crate Purpose
- **Primary**: Transaction sending and confirmation with configurable wait strategies
- **Secondary**: Helper utilities for error mapping and retry logic
- **Logging Pattern**: Focused on transaction lifecycle, status monitoring, and error conditions

### Files to Update
1. `src/lib.rs` - Core RPC client wrapper (main)
2. `src/utils.rs` - Error mapping and retry helpers (secondary)

---

## File Analysis

### File 1: `src/lib.rs`

**Priority**: 1 (Primary, contains most logging)

**Async functions with logging**:

1. **`get_latest_blockhash()` - Line 250**
   - Current: No logging detected
   - Status: Clean

2. **`get_slot()` - Line 267**
   - Current: No logging detected
   - Status: Clean

3. **`get_account()` - Line 283**
   - Current: No logging detected
   - Status: Clean

4. **`get_multiple_accounts()` - Line 298**
   - Current: No logging detected
   - Status: Clean

5. **`get_latest_signatures()` - Line 326**
   - Current: No logging detected
   - Status: Clean

6. **`get_address_lookup_table()` - Line 346**
   - Current: No logging detected
   - Status: Clean

7. **`send_transaction()` - Lines 421-491**
   - Current: No logging
   - Status: Clean

8. **`wait_for_processed_status()` - Lines 494-559**
   - Current logging:
     ```rust
     // Line 539-542
     trace!(
         "Waiting for blockhash {} to become valid",
         recent_blockhash
     );
     ```
   - **Issue**: Message contains format placeholder; needs extraction

9. **`wait_for_confirmed_status()` - Lines 562-595**
   - Current: No logging detected
   - Status: Clean

**Summary**: Only 1 log statement in the file, in `wait_for_processed_status()`.

---

### File 2: `src/utils.rs`

**Priority**: 2 (Secondary, contains utility functions)

**Async functions with logging**:

1. **`send_transaction_with_retries()` - Lines 34-71**
   - Type: Generic async function with retry loop
   - Current logging:
     ```rust
     // No direct logging in this function
     ```
   - Status: Clean

2. **`map_magicblock_client_error()` - Lines 93-141**
   - Type: Sync function (not async) - **No span needed**
   - Current logging:
     ```rust
     // Line 129
     error!("Unexpected error during send transaction: {:?}", err);
     ```
   - **Issue**: Message contains format placeholder; needs extraction

3. **`decide_rpc_native_flow()` - Lines 203-214**
   - Type: Sync function (not async) - **No span needed**
   - Current: No logging
   - Status: Clean

**Summary**: Only 1 log statement in async context, 1 in sync function.

---

## Critical Observation: Minimal Logging

The `magicblock-rpc-client` crate has **very minimal logging** compared to other crates:
- Only **2 log statements total** across both files
- Only **1 in an async function** (`wait_for_processed_status`)
- Only **1 in a sync function** (`map_magicblock_client_error`)

**Implementation is straightforward**: Apply the pattern to the single async function and fix the one sync-context error log.

---

## Standard Field Naming for Magicblock-RPC-Client

| Field | Type | Format | Use Case |
|-------|------|--------|----------|
| `signature` | Signature | `%` | Transaction signature |
| `blockhash` | Hash | `%` | Blockhash value |
| `commitment` | CommitmentLevel | `%` | RPC commitment level |
| `timeout_ms` | u64 | bare | Timeout in milliseconds |
| `elapsed_ms` | u64 | bare | Elapsed time in milliseconds |
| `slot` | Slot | bare | Block slot number |
| `error` | Error | `?` | Error details |
| `attempt_count` | usize | bare | Retry attempt number |
| `account_count` | usize | bare | Number of accounts |

---

## Before/After Examples

### Example 1: `wait_for_processed_status()` - Line 539

**Before**:
```rust
#[tokio::main]
pub async fn wait_for_processed_status(
    &self,
    signature: &Signature,
    recent_blockhash: &Hash,
    timeout: Duration,
    check_interval: Duration,
    blockhash_valid_timeout: &Option<Duration>,
) -> MagicBlockRpcClientResult<TransactionResult<()>> {
    let start = Instant::now();
    while start.elapsed() < timeout {
        // ... 
        trace!(
            "Waiting for blockhash {} to become valid",
            recent_blockhash
        );
        // ...
    }
}
```

**After**:
```rust
#[instrument(
    skip(self),
    fields(
        signature = %signature,
        blockhash = %recent_blockhash,
        timeout_ms = timeout.as_millis() as u64,
    )
)]
pub async fn wait_for_processed_status(
    &self,
    signature: &Signature,
    recent_blockhash: &Hash,
    timeout: Duration,
    check_interval: Duration,
    blockhash_valid_timeout: &Option<Duration>,
) -> MagicBlockRpcClientResult<TransactionResult<()>> {
    let start = Instant::now();
    while start.elapsed() < timeout {
        // ...
        trace!(
            elapsed_ms = start.elapsed().as_millis() as u64,
            "Waiting for blockhash validity"
        );
        // ...
    }
}
```

**Changes**:
- Line 494: Add `#[instrument(skip(self), fields(signature = %signature, blockhash = %recent_blockhash, timeout_ms = timeout.as_millis() as u64))]`
- Line 539: Convert `trace!("Waiting for blockhash {} to become valid", recent_blockhash)` to `trace!(elapsed_ms = start.elapsed().as_millis() as u64, "Waiting for blockhash validity")`
- Message becomes constant: "Waiting for blockhash validity"
- Blockhash extracted to span field (captured at function entry)
- Elapsed time added as field at log point

---

### Example 2: `map_magicblock_client_error()` - Line 129

**Before**:
```rust
pub fn map_magicblock_client_error<TxMap, ExecErr>(
    transaction_error_mapper: &TxMap,
    error: MagicBlockRpcClientError,
) -> ExecErr
where
    TxMap: TransactionErrorMapper<ExecutionError = ExecErr>,
    ExecErr: From<MagicBlockRpcClientError>,
{
    match error {
        // ... cases ...
        err @ (MagicBlockRpcClientError::GetSlot(_)
        | MagicBlockRpcClientError::LookupTableDeserialize(_)) => {
            error!("Unexpected error during send transaction: {:?}", err);
            err.into()
        }
        // ...
    }
}
```

**After**:
```rust
pub fn map_magicblock_client_error<TxMap, ExecErr>(
    transaction_error_mapper: &TxMap,
    error: MagicBlockRpcClientError,
) -> ExecErr
where
    TxMap: TransactionErrorMapper<ExecutionError = ExecErr>,
    ExecErr: From<MagicBlockRpcClientError>,
{
    match error {
        // ... cases ...
        err @ (MagicBlockRpcClientError::GetSlot(_)
        | MagicBlockRpcClientError::LookupTableDeserialize(_)) => {
            error!(error = ?err, "Unexpected error during send transaction");
            err.into()
        }
        // ...
    }
}
```

**Changes**:
- Line 129: Convert `error!("Unexpected error during send transaction: {:?}", err)` to `error!(error = ?err, "Unexpected error during send transaction")`
- Error extracted to field (using `?` for Debug formatting)
- Message becomes constant: "Unexpected error during send transaction"
- **Note**: This is a sync function, so no `#[instrument]` needed

---

## Implementation Plan

### Step 1: Add #[instrument] to `wait_for_processed_status()`

**File**: `src/lib.rs`

**Location**: Line 494

**Action**:
```rust
// BEFORE
pub async fn wait_for_processed_status(
    &self,
    signature: &Signature,
    recent_blockhash: &Hash,
    timeout: Duration,
    check_interval: Duration,
    blockhash_valid_timeout: &Option<Duration>,
) -> MagicBlockRpcClientResult<TransactionResult<()>> {

// AFTER
#[instrument(
    skip(self),
    fields(
        signature = %signature,
        blockhash = %recent_blockhash,
        timeout_ms = timeout.as_millis() as u64,
    )
)]
pub async fn wait_for_processed_status(
    &self,
    signature: &Signature,
    recent_blockhash: &Hash,
    timeout: Duration,
    check_interval: Duration,
    blockhash_valid_timeout: &Option<Duration>,
) -> MagicBlockRpcClientResult<TransactionResult<()>> {
```

**Rationale**:
- `skip(self)`: Avoids circular reference with Arc<RpcClient>
- `signature = %signature`: Important context for transaction tracking
- `blockhash = %recent_blockhash`: Identifies which blockhash we're waiting for
- `timeout_ms`: Helps correlate with actual timeout behavior

---

### Step 2: Rewrite trace! in `wait_for_processed_status()`

**File**: `src/lib.rs`

**Location**: Line 539

**Action**:
```rust
// BEFORE (Lines 539-542)
trace!(
    "Waiting for blockhash {} to become valid",
    recent_blockhash
);

// AFTER
trace!(
    elapsed_ms = start.elapsed().as_millis() as u64,
    "Waiting for blockhash validity"
);
```

**Rationale**:
- Message constant: "Waiting for blockhash validity"
- Extract elapsed time as field (more useful than iteration count)
- Blockhash already in span context, no need to repeat
- Trace level is appropriate for polling loops

---

### Step 3: Rewrite error! in `map_magicblock_client_error()`

**File**: `src/utils.rs`

**Location**: Line 129

**Action**:
```rust
// BEFORE (Line 129)
error!("Unexpected error during send transaction: {:?}", err);

// AFTER
error!(error = ?err, "Unexpected error during send transaction");
```

**Rationale**:
- Message constant: "Unexpected error during send transaction"
- Error extracted to field with Debug formatting
- This is a sync function, so no span wrapper needed
- Error log will have access to all call context via the error object itself

---

### Step 4: Verify imports

**Action**: Ensure `#[instrument]` macro is imported from tracing

**File**: `src/lib.rs`

**Current**: Line 34 has `use tracing::*;` ✅

No changes needed - wildcard import already covers `#[instrument]`.

---

## Files in Implementation Order

Process in this exact order:

1. **`src/lib.rs`** (async function with logging)
   - Add `#[instrument]` to `wait_for_processed_status()`
   - Rewrite trace! at line 539

2. **`src/utils.rs`** (sync function with logging)
   - Rewrite error! at line 129 (no span needed)

---

## Checklist

### Pre-Implementation
- [ ] Identified all async functions: Only 9 total, 1 with logging
- [ ] Identified sync functions with logging: 1 found
- [ ] Chose fields to capture: signature, blockhash, timeout_ms, elapsed_ms
- [ ] Reviewed field naming conventions: All follow standard table

### During Implementation

#### File: src/lib.rs
- [ ] Added `#[instrument]` to `wait_for_processed_status()` (line 494)
- [ ] Used `skip(self)` parameter (Arc<RpcClient> not Display)
- [ ] Captured fields: signature, blockhash, timeout_ms
- [ ] Converted trace! message to constant (line 539)
- [ ] Extracted elapsed_ms as field at log point
- [ ] No manual spans created

#### File: src/utils.rs
- [ ] Converted error! message to constant (line 129)
- [ ] Extracted error to field using `?` format
- [ ] Verified sync function (no #[instrument] needed)

### Post-Implementation

#### You must run these commands and verify:

```bash
cd /Users/thlorenz/dev/mb/wizard/magicblock-validator
cargo check -p magicblock-rpc-client
make lint
cargo nextest run -p magicblock-rpc-client
RUST_LOG=trace cargo test -p magicblock-rpc-client 2>&1 | head -100
```

- [ ] `cargo check` passes
- [ ] `make lint` passes (no clippy warnings)
- [ ] `cargo nextest run` passes all tests
- [ ] Log message is constant: "Waiting for blockhash validity"
- [ ] Log message is constant: "Unexpected error during send transaction"
- [ ] Fields appear in trace output
- [ ] No format placeholders in message text

#### Expected log output (when running tests with RUST_LOG=trace):

```
trace span: wait_for_processed_status{signature=<sig>, blockhash=<hash>, timeout_ms=50000}
    elapsed_ms=120 Waiting for blockhash validity

error message: error=<error_detail> Unexpected error during send transaction
```

#### I will manually verify:

- [ ] Logging output matches constant message pattern
- [ ] All context extracted to fields
- [ ] Span names derived from function names
- [ ] No duplicate information (message AND field)

---

## Summary

This plan provides:
- **2 files to update** (very small crate)
- **1 async function** needing `#[instrument]` 
- **2 log statements total** to fix (1 async, 1 sync)
- **Concrete before/after examples** for both cases
- **Deterministic implementation order**

The `magicblock-rpc-client` crate is straightforward: minimal logging, focused on transaction lifecycle. Implementation should take ~15 minutes.

### Key Differences from magicblock-chainlink

| Aspect | Chainlink | RPC-Client |
|--------|-----------|-----------|
| Files | 19 | 2 |
| Async functions | 50+ | 9 |
| Log statements | 100+ | 2 |
| Complexity | High (heavy logging) | Low (minimal logging) |
| Instrumentation needed | High density | Sparse (1 function) |

The RPC-Client crate demonstrates the "light touch" scenario: only critical paths are logged.
