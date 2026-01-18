# Magicblock-Validator-Admin Structured Logging Implementation Plan

Based on the deterministic pattern from `02_plan-general.md`, this document provides a concrete, step-by-step implementation guide specific to the `magicblock-validator-admin` crate.

---

## Overview

The `magicblock-validator-admin` crate is a small, focused crate that manages administrative tasks for the validator. It contains a single primary module with async functions for claiming validator fees. Total files to update: **1 file** with logging statements.

### File Categories

1. **Core Task Management**: Fee claiming task lifecycle
   - `claim_fees.rs` - Fee claim scheduling and execution

---

## Critical Files (Process First)

### File 1: `src/claim_fees.rs`

**Priority**: 1 (Only file in crate, no dependencies on other crate modules)

**Async functions with logging**:
1. `start()` - Task initialization (spawns async block)
2. `claim_fees()` - Main fee claiming operation

**Current log statements**:

```rust
// Line 29 - error if already started
error!("Claim fees task already started");

// Line 35 - task startup
info!("Starting claim fees task");

// Line 43 - fee claim failure
error!("Failed to claim fees: {:?}", err);

// Line 49 - task shutdown
info!("Claim fees task stopped");

// Line 56 - stop request
info!("Stopping claim fees task");

// Line 70 - operation start
info!("Claiming validator fees");

// Line 97 - operation success
info!("Successfully claimed validator fees");
```

**Async context**:
- The `start()` method is NOT async, but it spawns an async task via `tokio::spawn()`
- The spawned async block contains logging (lines 35, 43, 49)
- `claim_fees()` is async and contains logging (lines 70, 97)

**Implementation**:

1. Add `#[instrument]` to `claim_fees()` async function:
   - `#[instrument(fields(validator = %validator_authority().pubkey()))]`
   - Note: We capture the validator pubkey since it's derived inside the function

2. For the spawned task in `start()`:
   - Extract the async block into a separate `async fn run_claim_fees_loop()`
   - Add `#[instrument(skip(token), fields(tick_period_ms = tick_period.as_millis() as u64, url = %url))]`
   - Call this from `tokio::spawn()`

3. Keep `start()` and `stop()` as sync functions (no spans needed per design)
   - These manage lifecycle only, minimal context needed

4. Rewrite all log statements removing parameters into fields

---

## Standard Field Naming for Magicblock-Validator-Admin

| Field | Type | Format | Use Case |
|-------|------|--------|----------|
| `validator` | Pubkey | `%` | Validator authority pubkey |
| `url` | String | `%` | RPC endpoint URL |
| `tick_period_ms` | u64 | bare | Task tick period in milliseconds |
| `error` | Error | `?` | Error details |
| `task_state` | &str | bare | Task lifecycle state |

---

## Before/After Examples

### Example 1: `claim_fees.rs` - `start()` with spawned task

**Before**:
```rust
pub fn start(&mut self, tick_period: Duration, url: String) {
    if self.handle.is_some() {
        error!("Claim fees task already started");
        return;
    }

    let token = self.token.clone();
    let handle = tokio::spawn(async move {
        info!("Starting claim fees task");
        let start_time = Instant::now() + tick_period;
        let mut interval = tokio::time::interval_at(start_time, tick_period);
        loop {
            tokio::select! {
                _ = interval.tick() => {
                    if let Err(err) = claim_fees(url.clone()).await {
                        error!("Failed to claim fees: {:?}", err);
                    }
                },
                _ = token.cancelled() => break,
            }
        }
        info!("Claim fees task stopped");
    });
    self.handle = Some(handle);
}
```

**After**:
```rust
pub fn start(&mut self, tick_period: Duration, url: String) {
    if self.handle.is_some() {
        error!("Claim fees task already started");
        return;
    }

    let token = self.token.clone();
    let handle = tokio::spawn(run_claim_fees_loop(token, tick_period, url));
    self.handle = Some(handle);
}

// New extracted async function with #[instrument]
#[instrument(skip(token), fields(tick_period_ms = tick_period.as_millis() as u64, url = %url))]
async fn run_claim_fees_loop(token: CancellationToken, tick_period: Duration, url: String) {
    info!("Starting claim fees task");
    let start_time = Instant::now() + tick_period;
    let mut interval = tokio::time::interval_at(start_time, tick_period);
    loop {
        tokio::select! {
            _ = interval.tick() => {
                if let Err(err) = claim_fees(url.clone()).await {
                    error!(error = ?err, "Failed to claim fees");
                }
            },
            _ = token.cancelled() => break,
        }
    }
    info!("Claim fees task stopped");
}
```

**Key changes**:
- Extracted spawned task into separate `run_claim_fees_loop()` function
- Added `#[instrument]` to capture `tick_period_ms` and `url`
- Rewrote line 43: `error!(error = ?err, "Failed to claim fees");` (removed `:?` interpolation)
- Kept `start()` and `stop()` as sync (no spans needed)

### Example 2: `claim_fees.rs` - `claim_fees()` async function

**Before**:
```rust
async fn claim_fees(url: String) -> Result<(), MagicBlockRpcClientError> {
    info!("Claiming validator fees");

    let rpc_client = RpcClient::new_with_commitment(url, CommitmentConfig::confirmed());

    let keypair_ref = &validator_authority();
    let validator = keypair_ref.pubkey();

    let ix = validator_claim_fees(validator, None);

    let latest_blockhash = rpc_client.get_latest_blockhash().await.map_err(|e| {
        MagicBlockRpcClientError::GetLatestBlockhash(Box::new(e))
    })?;

    let tx = Transaction::new_signed_with_payer(
        &[ix],
        Some(&validator),
        &[keypair_ref],
        latest_blockhash,
    );

    rpc_client
        .send_and_confirm_transaction(&tx)
        .await
        .map_err(|e| MagicBlockRpcClientError::SendTransaction(Box::new(e)))?;

    info!("Successfully claimed validator fees");

    Ok(())
}
```

**After**:
```rust
#[instrument(fields(validator = %validator_authority().pubkey()))]
async fn claim_fees(url: String) -> Result<(), MagicBlockRpcClientError> {
    info!("Claiming validator fees");

    let rpc_client = RpcClient::new_with_commitment(url, CommitmentConfig::confirmed());

    let keypair_ref = &validator_authority();
    let validator = keypair_ref.pubkey();

    let ix = validator_claim_fees(validator, None);

    let latest_blockhash = rpc_client
        .get_latest_blockhash()
        .await
        .map_err(|e| {
            error!(error = ?e, "Failed to get latest blockhash");
            MagicBlockRpcClientError::GetLatestBlockhash(Box::new(e))
        })?;

    let tx = Transaction::new_signed_with_payer(
        &[ix],
        Some(&validator),
        &[keypair_ref],
        latest_blockhash,
    );

    rpc_client
        .send_and_confirm_transaction(&tx)
        .await
        .map_err(|e| {
            error!(error = ?e, "Failed to send transaction");
            MagicBlockRpcClientError::SendTransaction(Box::new(e))
        })?;

    info!("Successfully claimed validator fees");

    Ok(())
}
```

**Key changes**:
- Added `#[instrument(fields(validator = %validator_authority().pubkey()))]`
  - Captures validator pubkey from `validator_authority()` call
  - This provides span context for all operations within the function
- Added error logging at map_err points to provide context for RPC failures
- All info messages remain constant (no parameters in message text)
- Errors are logged with `error = ?e` field format

---

## Implementation Order

Process files in this exact order:

1. `src/claim_fees.rs` - Single file, no dependencies

---

## Detailed Implementation Steps

### Step 1: Extract Spawned Task

In `src/claim_fees.rs`, create a new async function `run_claim_fees_loop()`:

```rust
#[instrument(skip(token), fields(tick_period_ms = tick_period.as_millis() as u64, url = %url))]
async fn run_claim_fees_loop(
    token: CancellationToken,
    tick_period: Duration,
    url: String,
) {
    info!("Starting claim fees task");
    let start_time = Instant::now() + tick_period;
    let mut interval = tokio::time::interval_at(start_time, tick_period);
    loop {
        tokio::select! {
            _ = interval.tick() => {
                if let Err(err) = claim_fees(url.clone()).await {
                    error!(error = ?err, "Failed to claim fees");
                }
            },
            _ = token.cancelled() => break,
        }
    }
    info!("Claim fees task stopped");
}
```

### Step 2: Update `start()` Method

Replace the async block spawn with a call to the extracted function:

```rust
pub fn start(&mut self, tick_period: Duration, url: String) {
    if self.handle.is_some() {
        error!("Claim fees task already started");
        return;
    }

    let token = self.token.clone();
    let handle = tokio::spawn(run_claim_fees_loop(token, tick_period, url));
    self.handle = Some(handle);
}
```

### Step 3: Add `#[instrument]` to `claim_fees()`

```rust
#[instrument(fields(validator = %validator_authority().pubkey()))]
async fn claim_fees(url: String) -> Result<(), MagicBlockRpcClientError> {
    // ... body unchanged ...
}
```

### Step 4: Add Error Context to Error Paths

In `claim_fees()`, add logging when errors occur:

```rust
let latest_blockhash = rpc_client
    .get_latest_blockhash()
    .await
    .map_err(|e| {
        error!(error = ?e, "Failed to get latest blockhash");
        MagicBlockRpcClientError::GetLatestBlockhash(Box::new(e))
    })?;
```

And similarly for the send_and_confirm_transaction call.

### Step 5: Add Imports

Add to imports (if not already present):

```rust
use tracing::{error, info, instrument};
```

---

## Validation Steps

### 1. Syntax Check
```bash
cd magicblock-validator-admin
cargo check
```

### 2. Lint
```bash
make lint
```

### 3. Tests
```bash
cargo nextest run -p magicblock-validator-admin
```

### 4. Log Inspection (if applicable)
Since this crate doesn't have direct tests, verify by:
```bash
RUST_LOG=debug cargo build -p magicblock-validator-admin 2>&1 | head -50
```

**Verify**:
- No compile errors after adding `#[instrument]`
- All log messages are constant (no variables in message)
- Fields properly captured in span attributes
- No duplicate information (in message AND field)

---

## Summary

This plan provides:

1. **1 file to update**: `src/claim_fees.rs`
2. **2 async functions to instrument**:
   - `run_claim_fees_loop()` (newly extracted)
   - `claim_fees()`
3. **Concrete before/after examples** for both functions
4. **Standard field naming** specific to validator administration
5. **Clear implementation order** (sequential, single file)

**Key insight**: The main complexity is extracting the spawned async block from `start()` into its own function so we can add `#[instrument]`. This makes the task's execution observable with proper span context and field capture.

The result: Clean, queryable logging for validator administrative tasks with proper context for fee claim operations and task lifecycle events.