# Magicblock-API Structured Logging Implementation Plan

Based on the deterministic pattern from `02_plan-general.md`, this document provides a concrete, step-by-step implementation guide specific to the `magicblock-api` crate.

---

## Overview

The `magicblock-api` crate is the main entry point for the MagicBlock validator, coordinating initialization, ledger processing, and service lifecycle management. It contains **9 source files** with **11 async functions** that require instrumentation.

### Key Characteristics

- **Initialization-heavy**: Most logging occurs during `MagicValidator::try_from_config()` and related setup functions
- **Lifecycle management**: Logging around `start()`, `stop()`, and service teardown
- **Async function count**: 11 total (mostly concentrated in `magic_validator.rs` and `tickers.rs`)
- **Message patterns**: Currently use simple log calls with parameters embedded; minimal structured logging
- **Service coordination**: Logs relate to ledger state, account initialization, and service readiness

---

## Async Functions with Logging

### `magic_validator.rs`

1. **`try_from_config()` (line 131)** - Main initialization function
   - Current logs: 1 structured (ledger slot), 2 debug multi-parameter
   - Status: Partial structure, needs #[instrument]

2. **`init_committor_service()` (line 335)** - Async helper, private
   - Current logs: 1 debug statement
   - Status: Needs #[instrument]

3. **`init_chainlink()` (line 371)** - Async helper, private
   - Current logs: 1 debug statement
   - Status: Needs #[instrument]

4. **`maybe_process_ledger()` (line 473)** - Ledger processing, async
   - Current logs: 1 warn, 1 debug, 1 info
   - Status: Needs #[instrument]

5. **`register_validator_on_chain()` (line 533)** - Chain registration, async
   - Current logs: 0 logging statements (pure logic)
   - Status: May not need #[instrument] if no logs, but add for consistency

6. **`ensure_validator_funded_on_chain()` (line 573)** - Balance check, async
   - Current logs: 0 logging statements
   - Status: No instrumentation needed

7. **`start()` (line 599)** - Main startup orchestrator, async
   - Current logs: 2 error statements (inside spawned task)
   - Status: Needs #[instrument]

8. **`stop()` (line 678)** - Shutdown handler, async
   - Current logs: 2 error, 1 info statements
   - Status: Needs #[instrument]

### `tickers.rs`

9. **`handle_scheduled_commits()` (line 82)** - Async internal helper
   - Current logs: 2 error statements
   - Status: Needs #[instrument]

10. **Tokio spawned closure in `init_slot_ticker()` (line 37)** - Anonymous async closure
    - Current logs: 2 error, 2 debug statements
    - Status: This is not a named function; cannot add #[instrument]. Log rewrites only.

11. **Tokio spawned closure in `init_system_metrics_ticker()` (line 129)** - Anonymous async closure
    - Current logs: 1 warn (in sync fn), 1 info
    - Status: sync function; no #[instrument] needed

---

## Current Log Patterns

### Pattern 1: Parameter Embedding (Most Common)
```rust
// Example from magic_validator.rs:148
info!("Latest ledger slot: {}", last_slot);

// Example from magic_validator.rs:507-509
debug!(
    "Found {} scheduled commits while processing ledger, clearing them",
    scheduled_commits
);

// Example from tickers.rs:53
debug!("Advanced to slot {}", next_slot);
```

### Pattern 2: Error Context
```rust
// Example from tickers.rs:49
error!("Failed to write block: {:?}", err);

// Example from tickers.rs:93
error!("Failed to accept scheduled commits: {:?}", err);

// Example from magic_validator.rs:707
error!("Failed to shutdown ledger: {:?}", err);
```

### Pattern 3: Info/Shutdown Messages
```rust
// Example from magic_validator.rs:718
info!("MagicValidator shutdown!");

// Example from tickers.rs:143
info!("System metrics ticker shutdown!");
```

---

## Standard Field Naming for Magicblock-API

| Field | Type | Format | Use Case |
|-------|------|--------|----------|
| `slot` | u64 | bare | Block slot number |
| `committed_count` | u64 | bare | Number of scheduled commits |
| `error` | Error | `?` | Error details |
| `ledger_path` | Path | `%` | Ledger directory path |
| `identity` | Pubkey | `%` | Validator identity pubkey |
| `reason` | &str | bare | Operation reason/status |

---

## Files to Update (In Order)

### Priority 1: Utility Functions (No Dependencies)

**File 1: `src/tickers.rs`**
- Priority: 1 (No dependencies, utility functions)
- Async functions: `handle_scheduled_commits()` (needs #[instrument])
- Closures: 2 spawned closures (rewrite logs only, no #[instrument])

**File 2: `src/domain_registry_manager.rs`**
- Priority: 2 (Sync functions, minimal logging)
- Async functions: None (all sync)
- Status: Minimal logging - rewrite info statements only

### Priority 2: Initialization Chain

**File 3: `src/magic_validator.rs` - Helper Functions**
- Priority: 3 (Private async helpers)
- Process in order:
  1. `init_committor_service()` 
  2. `init_chainlink()`
  3. `maybe_process_ledger()`
  4. `register_validator_on_chain()` (likely no logs)
  5. `ensure_validator_funded_on_chain()` (no logs)
  6. `try_from_config()` (depends on above)
  7. `start()` (top-level orchestrator)
  8. `stop()` (shutdown handler)

---

## Before/After Examples

### Example 1: `tickers.rs` - `handle_scheduled_commits()`

**Before**:
```rust
async fn handle_scheduled_commits<C: ScheduledCommitsProcessor>(
    committor_processor: &Arc<C>,
    transaction_scheduler: &TransactionSchedulerHandle,
    latest_block: &LatestBlock,
) {
    let tx = InstructionUtils::accept_scheduled_commits(
        latest_block.load().blockhash,
    );
    if let Err(err) = transaction_scheduler.execute(tx).await {
        error!("Failed to accept scheduled commits: {:?}", err);
        return;
    }

    if let Err(err) = committor_processor.process().await {
        error!("Failed to process scheduled commits: {:?}", err);
    }
}
```

**After**:
```rust
#[instrument(skip(committor_processor, transaction_scheduler, latest_block))]
async fn handle_scheduled_commits<C: ScheduledCommitsProcessor>(
    committor_processor: &Arc<C>,
    transaction_scheduler: &TransactionSchedulerHandle,
    latest_block: &LatestBlock,
) {
    let tx = InstructionUtils::accept_scheduled_commits(
        latest_block.load().blockhash,
    );
    if let Err(err) = transaction_scheduler.execute(tx).await {
        error!(error = ?err, "Failed to accept scheduled commits");
        return;
    }

    if let Err(err) = committor_processor.process().await {
        error!(error = ?err, "Failed to process scheduled commits");
    }
}
```

**Changes**:
- Added `#[instrument(skip(...))]` - All params skipped (internal helpers, no user-facing context)
- Line 93: Extracted `err` to field, made message constant
- Line 101: Extracted `err` to field, made message constant

---

### Example 2: `magic_validator.rs` - `maybe_process_ledger()`

**Before**:
```rust
async fn maybe_process_ledger(&self) -> ApiResult<()> {
    // ... validation logic ...
    
    warn!("Skipping ledger keypair verification due to configuration");
    
    // ... processing ...
    
    debug!(
        "Found {} scheduled commits while processing ledger, clearing them",
        scheduled_commits
    );
    
    // ... more processing ...
    
    info!(
        "Processed ledger, validator continues at slot {}",
        slot_to_continue_at
    );

    Ok(())
}
```

**After**:
```rust
#[instrument(skip(self), fields(slot_to_continue = tracing::field::Empty))]
async fn maybe_process_ledger(&self) -> ApiResult<()> {
    // ... validation logic ...
    
    warn!("Skipping ledger keypair verification");
    
    // ... processing ...
    
    debug!(committed_count = scheduled_commits, "Cleared scheduled commits");
    
    // ... more processing ...
    
    tracing::Span::current().record("slot_to_continue", slot_to_continue_at);
    info!("Processed ledger");

    Ok(())
}
```

**Changes**:
- Added `#[instrument(skip(self))]` with empty field for slot (populated when known)
- Line 454: Removed "due to configuration" reason, made message simpler
- Line 507-509: Extracted `scheduled_commits` to field `committed_count`, constant message
- Line 525-527: Extracted `slot_to_continue_at` to field (record during execution), constant message
- Used `Span::current().record()` for values discovered during execution

---

### Example 3: `magic_validator.rs` - `start()`

**Before**:
```rust
pub async fn start(&mut self) -> ApiResult<()> {
    // ... setup ...
    
    let task_scheduler = self
        .task_scheduler
        .take()
        .expect("task_scheduler should be initialized");
    tokio::spawn(async move {
        let join_handle = match task_scheduler.start().await {
            Ok(join_handle) => join_handle,
            Err(err) => {
                error!("Failed to start task scheduler: {:?}", err);
                error!("Exiting process...");
                std::process::exit(1);
            }
        };
        match join_handle.await {
            Ok(Ok(())) => {}
            Ok(Err(err)) => {
                error!("An error occurred while running the task scheduler: {:?}", err);
                error!("Exiting process...");
                std::process::exit(1);
            }
            Err(err) => {
                error!("Failed to start task scheduler: {:?}", err);
                error!("Exiting process...");
                std::process::exit(1);
            }
        }
    });
    
    validator::finished_starting_up();
    Ok(())
}
```

**After**:
```rust
#[instrument(skip(self))]
pub async fn start(&mut self) -> ApiResult<()> {
    // ... setup ...
    
    let task_scheduler = self
        .task_scheduler
        .take()
        .expect("task_scheduler should be initialized");
    tokio::spawn(async move {
        let join_handle = match task_scheduler.start().await {
            Ok(join_handle) => join_handle,
            Err(err) => {
                error!(error = ?err, "Failed to start task scheduler");
                error!("Exiting process");
                std::process::exit(1);
            }
        };
        match join_handle.await {
            Ok(Ok(())) => {}
            Ok(Err(err)) => {
                error!(error = ?err, "Task scheduler failed");
                error!("Exiting process");
                std::process::exit(1);
            }
            Err(err) => {
                error!(error = ?err, "Task scheduler join failed");
                error!("Exiting process");
                std::process::exit(1);
            }
        }
    });
    
    validator::finished_starting_up();
    Ok(())
}
```

**Changes**:
- Added `#[instrument(skip(self))]` - self is internal state
- Line 654: Extracted `err` to field, simplified message
- Line 662: Extracted `err` to field, more specific message
- Line 667: Extracted `err` to field, distinguishes from startup error
- Removed "ellipsis" pattern in messages ("Exiting process..." → "Exiting process")

---

### Example 4: `magic_validator.rs` - `try_from_config()`

**Before**:
```rust
pub async fn try_from_config(config: ValidatorParams) -> ApiResult<Self> {
    // ... setup ...
    
    let (ledger, last_slot) =
        Self::init_ledger(&config.ledger, &config.storage)?;
    info!("Latest ledger slot: {}", last_slot);
    
    // ... more setup ...
    
    debug!(
        "Found {} scheduled commits while processing ledger, clearing them",
        scheduled_commits
    );
    
    // ... final setup ...
}
```

**After**:
```rust
#[instrument(skip_all, fields(last_slot))]
pub async fn try_from_config(config: ValidatorParams) -> ApiResult<Self> {
    // ... setup ...
    
    let (ledger, last_slot) =
        Self::init_ledger(&config.ledger, &config.storage)?;
    tracing::Span::current().record("last_slot", last_slot);
    info!("Initialization complete");
    
    // ... more setup ...
    
    debug!(committed_count = scheduled_commits, "Cleared scheduled commits");
    
    // ... final setup ...
}
```

**Changes**:
- Added `#[instrument(skip_all)]` - Skip all params (config is large, not useful in logs)
- Line 148: Record discovered slot, replace specific message with generic one
- Line 507-509: Use field extraction pattern for counts

---

## Implementation Order

Process files in this exact order:

1. **`src/tickers.rs`** - Utility async functions, no dependencies
2. **`src/domain_registry_manager.rs`** - Minimal logging, sync functions
3. **`src/magic_validator.rs` - Helpers**:
   - `init_committor_service()`
   - `init_chainlink()`
   - `maybe_process_ledger()`
   - `register_validator_on_chain()`
4. **`src/magic_validator.rs` - Main Functions**:
   - `try_from_config()`
   - `start()`
   - `stop()`
5. **`src/ledger.rs`** - 1 log statement, sync function
6. **`src/genesis_utils.rs`** - No logging
7. **`src/slot.rs`** - No logging
8. **`src/errors.rs`** - No logging
9. **`src/fund_account.rs`** - No logging
10. **`src/lib.rs`** - Check for any module-level logging

---

## Detailed File-by-File Instructions

### File: `src/tickers.rs`

**Total changes**: 3 functions, 5 log statements

#### Function 1: `handle_scheduled_commits()` (line 82)

Add `#[instrument]`:
```rust
#[instrument(skip(committor_processor, transaction_scheduler, latest_block))]
```

Rewrite logs:
- Line 93: `error!(error = ?err, "Failed to accept scheduled commits");`
- Line 101: `error!(error = ?err, "Failed to process scheduled commits");`

#### Function 2: `init_slot_ticker()` spawned closure (line 37)

This is NOT a named async function; it's a spawned closure. Skip #[instrument], but rewrite logs:

In the spawned async closure:
- Line 49: `error!(error = ?err, "Failed to write block");`
- Line 53: `debug!(slot = next_slot, "Advanced to slot");`
- Line 75: `debug!(slot = next_slot, "Advanced to slot");` (duplicate, same rewrite)

#### Function 3: `init_system_metrics_ticker()` spawned closure (line 129)

This is a sync function with spawned async closure. The spawned closure cannot get #[instrument].

Rewrite logs in spawned closure:
- Line 114: `warn!(error = ?err, "Failed to get ledger storage size");`
- Line 143: `info!("System metrics ticker shutdown");` (already good, just constant)

---

### File: `src/magic_validator.rs`

**Total changes**: 8 functions, 14 log statements

#### Function 1: `init_committor_service()` (line 335)

Add `#[instrument]`:
```rust
#[instrument(skip(config))]
```

Rewrite logs:
- Line 340: `debug!("Initializing committor service");` (or similar - check actual message)

#### Function 2: `init_chainlink()` (line 371)

Add `#[instrument]`:
```rust
#[instrument(skip(config, committor_service, transaction_scheduler, latest_block, accountsdb, faucet_pubkey))]
```

Rewrite logs:
- Line 359: `debug!("Reserved common pubkeys for committor service");`

#### Function 3: `maybe_process_ledger()` (line 473)

Add `#[instrument]`:
```rust
#[instrument(skip(self), fields(ledger_slot = tracing::field::Empty))]
```

Rewrite logs:
- Line 454: `warn!("Skipping ledger keypair verification");`
- Line 507-509: `debug!(committed_count = scheduled_commits, "Cleared scheduled commits");`
- Line 525-527: Record slot and use constant message:
  ```rust
  tracing::Span::current().record("ledger_slot", slot_to_continue_at);
  info!("Ledger processing complete");
  ```

#### Function 4: `register_validator_on_chain()` (line 533)

Add `#[instrument]` for consistency (no current logs):
```rust
#[instrument(skip(self, config), fields(identity = %self.identity))]
```

#### Function 5: `ensure_validator_funded_on_chain()` (line 573)

No logging - skip #[instrument]

#### Function 6: `try_from_config()` (line 131)

Add `#[instrument]`:
```rust
#[instrument(skip_all, fields(last_slot = tracing::field::Empty))]
```

Rewrite logs:
- Line 148: Extract and record slot:
  ```rust
  tracing::Span::current().record("last_slot", last_slot);
  info!("Ledger initialized");
  ```
- Line 262: `info!("Running execution backend");` (if "with N threads" exists, extract to field)
- Line 289: `info!("RPC runtime shutdown");`
- Line 296: Check context and rewrite if needed

#### Function 7: `start()` (line 599)

Add `#[instrument]`:
```rust
#[instrument(skip(self))]
```

Rewrite logs in spawned task (around lines 651-671):
- Line 654: `error!(error = ?err, "Failed to start task scheduler");`
- Line 655: `error!("Exiting process");` (keep as-is)
- Line 662: `error!(error = ?err, "Task scheduler failed");`
- Line 663: `error!("Exiting process");` (keep as-is)
- Line 667: `error!(error = ?err, "Task scheduler join failed");`
- Line 668: `error!("Exiting process");` (keep as-is)

#### Function 8: `stop()` (line 678)

Add `#[instrument]`:
```rust
#[instrument(skip(self))]
```

Rewrite logs:
- Line 699: `error!(error = ?err, "Failed to unregister");`
- Line 707: `error!(error = ?err, "Failed to shutdown ledger");`
- Line 714: `error!(error = ?err, "Ledger truncator did not gracefully exit");`
- Line 718: `info!("MagicValidator shutdown");`

---

### File: `src/domain_registry_manager.rs`

**Total changes**: 3 log statements (all in sync functions)

No async functions to instrument. Rewrite sync function logs:
- Line 135: `info!("Domain registry record up to date");`
- Line 138: `info!("Domain registry record requires update");`
- Line 143: `info!("Domain registry record absent, registering");`
- Line 193: `info!("Unregistering validator from domain registry");`

---

### File: `src/ledger.rs`

**Total changes**: 1 log statement (in sync function)

No async functions. Rewrite sync log:
- Line 26: `error!(error = ?err, "Unable to remove ledger");`

---

### Files with No Logging

No changes needed:
- `src/genesis_utils.rs` - Pure utility functions
- `src/slot.rs` - No logging
- `src/errors.rs` - Error definitions only
- `src/fund_account.rs` - No logging
- `src/lib.rs` - Verify no module-level logging

---

## Validation Steps

### 1. Syntax Check
```bash
cd magicblock-api
cargo check
```

### 2. Lint
```bash
make lint
```

### 3. Tests
```bash
cargo nextest run -p magicblock-api
```

### 4. Log Inspection
```bash
RUST_LOG=debug cargo test --package magicblock-api --lib 2>&1 | head -100
```

**Verify**:
- [ ] Each async function with logs has one `#[instrument]` annotation
- [ ] All log messages are constant (no variables in message)
- [ ] Parameters extracted to fields
- [ ] Correct format specifiers (%, ?, bare)
- [ ] No duplicate info (both message and field)
- [ ] Spawned closures have logs rewritten but no #[instrument]

---

## Summary

**magicblock-api implementation summary**:

| Item | Count |
|------|-------|
| Async functions requiring #[instrument] | 8 |
| Sync functions with logging needing rewrites | 5 |
| Spawned closures needing log rewrites | 2 |
| Total log statements to rewrite | 20+ |
| Files to update | 6 |
| High-priority files | 2 (`magic_validator.rs`, `tickers.rs`) |

**Key points**:
1. Most logging concentrated in `magic_validator.rs` (initialization and shutdown)
2. Simple patterns - mostly parameter embedding, easily extracted to fields
3. Few dependencies between files - can be processed in order
4. Spawned closures cannot get #[instrument] - rewrite logs only
5. Some functions have no logging - add #[instrument] for consistency in main initialization functions

Follow the implementation order, apply patterns from examples, and validate with provided steps.
