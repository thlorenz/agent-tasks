# Magicblock-Ledger Structured Logging Implementation Plan

Based on the deterministic pattern from `02_plan-general.md`, this document provides a concrete, step-by-step implementation guide specific to the `magicblock-ledger` crate.

---

## Overview

The `magicblock-ledger` crate is primarily a **synchronous, storage-focused library** with minimal async operations. It contains three async functions with logging statements:

1. **`blockstore_processor/mod.rs`**: Ledger block replay and processing
2. **`ledger_truncator.rs`**: Ledger size management and compaction

Total files to update: **3 files** with meaningful async logging (some files have sync logging only, which we defer to later passes).

### Crate Structure

- **Core Logic**: Synchronous database operations (get, put, delete)
- **Block Processing**: Async replay of blocks from ledger to transaction scheduler
- **Maintenance**: Async ledger truncation and compaction
- **Utilities**: File descriptor limits, conversions (mostly sync)

---

## Critical Files (Process First)

### File 1: `src/blockstore_processor/mod.rs`

**Priority**: 1 (Foundational, replay logic)

**Async functions with logging**:
1. `replay_blocks()` - Main async block replay loop (lines 32-142)
2. `process_ledger()` - Public entry point (lines 146-170)

**Current log statements**:

```rust
// Line 59-63: Progress reporting
info!(
    "Processing block: {}/{}",
    slot.to_formatted_string(&Locale::en),
    max_slot
);

// Line 131: Trace transaction result (conditional)
debug!("Result: {signature} - {result:?}");

// Line 156-158: Debug parameter logging
debug!(
    "Loaded accounts into bank from storage replaying blockhashes from {} and transactions from {}",
    blockhashes_only_starting_slot, full_process_starting_slot
);
```

**Implementation**:

1. Add `#[instrument(skip(params, transaction_scheduler))]` to `replay_blocks()` with fields:
   - `full_process_starting_slot = params.full_process_starting_slot`
   - `blockhashes_only_starting_slot = params.blockhashes_only_starting_slot`

2. Add `#[instrument(skip(ledger, transaction_scheduler))]` to `process_ledger()` with fields:
   - `full_process_starting_slot`
   - `max_age`

3. Rewrite log statements:
   - Line 59-63: `info!(slot = slot, "Processing block");`
   - Line 131: `debug!(signature = %signature, result = ?result, "Transaction replay result");`
   - Line 156: `debug!("Replay configuration");` (constants can go in span field)

**Before/After**:

```rust
// BEFORE
async fn replay_blocks(
    params: IterBlocksParams<'_>,
    transaction_scheduler: TransactionSchedulerHandle,
) -> LedgerResult<u64> {
    // ... setup ...
    loop {
        // ...
        if enabled!(Level::INFO) && slot.is_multiple_of(PROGRESS_REPORT_INTERVAL) {
            info!(
                "Processing block: {}/{}",
                slot.to_formatted_string(&Locale::en),
                max_slot
            );
        }
        
        // ... replay transactions ...
        if !enabled!(Level::TRACE) {
            debug!("Result: {signature} - {result:?}");
        }
    }
}

pub async fn process_ledger(
    ledger: &Ledger,
    full_process_starting_slot: Slot,
    transaction_scheduler: TransactionSchedulerHandle,
    max_age: u64,
) -> LedgerResult<u64> {
    let blockhashes_only_starting_slot =
        full_process_starting_slot.saturating_sub(max_age);
    debug!(
        "Loaded accounts into bank from storage replaying blockhashes from {} and transactions from {}",
        blockhashes_only_starting_slot, full_process_starting_slot
    );
    // ...
}
```

```rust
// AFTER
#[instrument(
    skip(params, transaction_scheduler),
    fields(
        full_process_starting_slot = params.full_process_starting_slot,
        blockhashes_only_starting_slot = params.blockhashes_only_starting_slot,
    )
)]
async fn replay_blocks(
    params: IterBlocksParams<'_>,
    transaction_scheduler: TransactionSchedulerHandle,
) -> LedgerResult<u64> {
    // ... setup ...
    loop {
        // ...
        if enabled!(Level::INFO) && slot.is_multiple_of(PROGRESS_REPORT_INTERVAL) {
            info!(slot, "Processing block");
        }
        
        // ... replay transactions ...
        if !enabled!(Level::TRACE) {
            debug!(signature = %signature, result = ?result, "Transaction replay result");
        }
    }
}

#[instrument(
    skip(ledger, transaction_scheduler),
    fields(
        full_process_starting_slot,
        max_age,
    )
)]
pub async fn process_ledger(
    ledger: &Ledger,
    full_process_starting_slot: Slot,
    transaction_scheduler: TransactionSchedulerHandle,
    max_age: u64,
) -> LedgerResult<u64> {
    let blockhashes_only_starting_slot =
        full_process_starting_slot.saturating_sub(max_age);
    debug!("Ledger replay starting");
    // ...
}
```

---

### File 2: `src/ledger_truncator.rs`

**Priority**: 2 (Async background service)

**Key async function**:
- `run()` (lines 57-87) - Main async event loop for truncation worker

**Current log statements** (in `run()` and related functions):

```rust
// Line 70: Error checking truncation
error!("Failed to check truncation condition: {err}");

// Line 77: Skip truncation
info!("Skipping truncation, ledger size: {}", current_size);

// Line 81: Ledger size report
info!("Ledger size: {current_size}");

// Line 83: Truncation failure
error!("Failed to truncate ledger!: {:?}", err);

// Line 101: Warning on missing range
warn!("Could not estimate truncation range");

// Line 114: Fat truncation start
info!("Fat truncation");

// Line 123-126: Fat truncation progress
info!(
    "Fat truncation. lowest slot: {}, hightest slot: {}",
    lowest_slot, highest_slot
);

// Line 128: Edge case warning
warn!("Nani3?");

// Line 140-143: Fat truncation complete
info!(
    "Fat truncation: truncating up to(inclusive): {}",
    truncate_to_slot
);

// Line 150: Flush error
error!("Failed to flush: {}", err);

// Line 287: Slot range truncation
info!(
    "LedgerTruncator: truncating slot range [{from_slot}; {to_slot}]"
);

// Line 325-327: Compaction start
info!(
    "LedgerTruncator: compacting slot range [{from_slot}; {to_slot}]"
);

// Lines 495-499, 506-510 (macro): Shutdown during compaction
info!(
    "Validator shutting down - stopping compaction after {}",
    <$cf as $crate::database::columns::ColumnName>::NAME
);
```

**Implementation**:

1. Add `#[instrument(skip(self))]` to `run()`

2. Rewrite all log statements following constant message pattern:
   - Line 70: `error!(error = ?err, "Truncation check failed");`
   - Line 77: `info!(current_size, "Skipping truncation");`
   - Line 81: `info!(current_size, "Checking ledger size");`
   - Line 83: `error!(error = ?err, "Truncation failed");`
   - Line 101: `warn!("No truncation range available");`
   - Line 114: `info!("Starting fat truncation");`
   - Line 123: `info!(lowest_slot, highest_slot, "Fat truncation bounds");`
   - Line 128: `warn!("Invalid slot range");`
   - Line 140: `info!(truncate_to_slot, "Fat truncation complete");`
   - Line 150: `error!(error = ?err, "Flush failed");`
   - Line 287: `info!(from_slot, to_slot, "Truncating slot range");`
   - Line 325: `info!(from_slot, to_slot, "Compacting slot range");`
   - Macro: `info!(column_name = %<$cf as ColumnName>::NAME, "Shutdown during compaction");`

**Before/After**:

```rust
// BEFORE
pub async fn run(self) {
    let mut interval = interval(self.truncation_time_interval);
    loop {
        tokio::select! {
            _ = self.cancellation_token.cancelled() => {
                return;
            }
            _ = interval.tick() => {
                let current_size = match self.ledger.storage_size() {
                    Ok(value) => value,
                    Err(err) => {
                        error!("Failed to check truncation condition: {err}");
                        continue;
                    }
                };

                if current_size < (self.ledger_size / 100) * FILLED_PERCENTAGE_LIMIT as u64 {
                    info!("Skipping truncation, ledger size: {}", current_size);
                    continue;
                }

                info!("Ledger size: {current_size}");
                if let Err(err) = self.truncate(current_size) {
                    error!("Failed to truncate ledger!: {:?}", err);
                }
            }
        }
    }
}

pub fn truncate_fat_ledger(&self, current_ledger_size: u64) -> LedgerResult<()> {
    info!("Fat truncation");
    
    let (highest_slot, _) = self.ledger.get_max_blockhash()?;
    let lowest_slot = self.ledger.get_lowest_slot()?.unwrap_or(0);
    info!(
        "Fat truncation. lowest slot: {}, hightest slot: {}",
        lowest_slot, highest_slot
    );
    if lowest_slot == highest_slot {
        warn!("Nani3?");
        return Ok(());
    }
    
    let truncate_to_slot = lowest_slot + num_slots_to_truncate - 1;
    info!(
        "Fat truncation: truncating up to(inclusive): {}",
        truncate_to_slot
    );
    // ...
}
```

```rust
// AFTER
#[instrument(skip(self))]
pub async fn run(self) {
    let mut interval = interval(self.truncation_time_interval);
    loop {
        tokio::select! {
            _ = self.cancellation_token.cancelled() => {
                return;
            }
            _ = interval.tick() => {
                let current_size = match self.ledger.storage_size() {
                    Ok(value) => value,
                    Err(err) => {
                        error!(error = ?err, "Truncation check failed");
                        continue;
                    }
                };

                if current_size < (self.ledger_size / 100) * FILLED_PERCENTAGE_LIMIT as u64 {
                    info!(current_size, "Skipping truncation");
                    continue;
                }

                info!(current_size, "Checking ledger size");
                if let Err(err) = self.truncate(current_size) {
                    error!(error = ?err, "Truncation failed");
                }
            }
        }
    }
}

pub fn truncate_fat_ledger(&self, current_ledger_size: u64) -> LedgerResult<()> {
    info!("Starting fat truncation");
    
    let (highest_slot, _) = self.ledger.get_max_blockhash()?;
    let lowest_slot = self.ledger.get_lowest_slot()?.unwrap_or(0);
    info!(lowest_slot, highest_slot, "Fat truncation bounds");
    if lowest_slot == highest_slot {
        warn!("Invalid slot range");
        return Ok(());
    }
    
    let truncate_to_slot = lowest_slot + num_slots_to_truncate - 1;
    info!(truncate_to_slot, "Fat truncation complete");
    // ...
}
```

---

## Standard Field Naming for Magicblock-Ledger

| Field | Type | Format | Use Case |
|-------|------|--------|----------|
| `slot` | u64 | bare | Block slot number |
| `full_process_starting_slot` | u64 | bare | Replay start for transactions |
| `blockhashes_only_starting_slot` | u64 | bare | Replay start for hashes only |
| `max_age` | u64 | bare | Maximum block age |
| `from_slot` | u64 | bare | Range start (truncation) |
| `to_slot` | u64 | bare | Range end (truncation) |
| `lowest_slot` | u64 | bare | Lowest slot in ledger |
| `highest_slot` | u64 | bare | Highest slot in ledger |
| `current_size` | u64 | bare | Current storage size in bytes |
| `truncate_to_slot` | u64 | bare | Target slot for truncation |
| `signature` | Signature | `%` | Transaction signature |
| `result` | Result/Error | `?` | Transaction execution result |
| `error` | Error | `?` | Error details |
| `column_name` | String | `%` | RocksDB column family name |

---

## Deferred: Sync Code and Minor Logging

The following files contain logging but are **not async** and thus do **not get spans**. Defer these to a second pass:

### Non-Async Files (No Spans Needed)

**File**: `src/store/utils.rs`
- `adjust_ulimit_nofile()` - Sync function
- Lines 32, 42-51, 60 - Logging for file descriptor setup
- These are sync so they do NOT get `#[instrument]`
- Just rewrite messages to constant format without adding attributes

**File**: `src/conversions/transaction.rs`
- Multiple sync helper functions
- Lines 154, 173, 222, 267, 344 - Validation warnings
- No `#[instrument]` needed (sync code)
- Rewrite: `warn!(error = ?err, "Invalid pubkey");` etc.

**File**: `src/database/cf_descriptors.rs`
- Sync column descriptor detection
- Lines 63, 78 - Column family setup logging
- No spans needed

**File**: `src/database/ledger_column.rs`
- Line 654: Sync utility function
- `warn!("Negative entry counter!");`
- No spans needed

---

## Implementation Order

Process files in this exact order:

1. **`src/blockstore_processor/mod.rs`** - Two async functions, foundational
2. **`src/ledger_truncator.rs`** - One main async function, background service

Then for follow-up pass (sync code):
3. `src/store/utils.rs` - File descriptor setup (sync)
4. `src/conversions/transaction.rs` - Validation helpers (sync)
5. `src/database/cf_descriptors.rs` - Column setup (sync)
6. `src/database/ledger_column.rs` - Utilities (sync)

---

## Validation Steps

### 1. Syntax Check
```bash
cd magicblock-ledger
cargo check
```

### 2. Lint
```bash
make lint
```

### 3. Tests
```bash
cargo nextest run -p magicblock-ledger
```

### 4. Log Inspection
```bash
RUST_LOG=debug cargo test --package magicblock-ledger --lib 2>&1 | head -100
```

**Verify**:
- [ ] Each async function creates one top-level span
- [ ] Log messages are constant (no variables in message)
- [ ] Fields appear with correct values in span context
- [ ] No duplicate information (in both message and field)
- [ ] Slot/block info is in span fields, not message

---

## Specific Rewrite Checklist for blockstore_processor/mod.rs

- [ ] Line 59-63: Rewrite progress log to constant message + field
  - Extract: `slot`
  - Message: `"Processing block"` (constant)
  - Remove: max_slot calculation when not at log level (it's debug-only)

- [ ] Line 131: Rewrite trace/debug log
  - Extract: `signature`, `result`
  - Message: `"Transaction replay result"`
  - Update condition: Keep enabled check or simplify

- [ ] Line 156-158: Rewrite debug parameter log
  - Extract: `full_process_starting_slot`, `blockhashes_only_starting_slot`
  - Message: `"Ledger replay starting"` (or similar constant)
  - These values should be in `#[instrument]` fields already

- [ ] Add `#[instrument]` to `replay_blocks()` (line 32)
- [ ] Add `#[instrument]` to `process_ledger()` (line 146)

---

## Specific Rewrite Checklist for ledger_truncator.rs

- [ ] Line 70: `error!("Failed to check truncation condition: {err}");`
  - Rewrite: `error!(error = ?err, "Truncation check failed");`

- [ ] Line 77: `info!("Skipping truncation, ledger size: {}", current_size);`
  - Rewrite: `info!(current_size, "Skipping truncation");`

- [ ] Line 81: `info!("Ledger size: {current_size}");`
  - Rewrite: `info!(current_size, "Checking ledger size");`

- [ ] Line 83: `error!("Failed to truncate ledger!: {:?}", err);`
  - Rewrite: `error!(error = ?err, "Truncation failed");`

- [ ] Line 101: `warn!("Could not estimate truncation range");`
  - Already constant! No change needed (or keep as is)

- [ ] Line 114: `info!("Fat truncation");`
  - Rewrite: `info!("Starting fat truncation");`

- [ ] Lines 123-126: Multi-line fat truncation log
  - Rewrite: `info!(lowest_slot, highest_slot, "Fat truncation bounds");`

- [ ] Line 128: `warn!("Nani3?");`
  - Rewrite: `warn!("Invalid slot range");`

- [ ] Lines 140-143: Multi-line completion log
  - Rewrite: `info!(truncate_to_slot, "Fat truncation complete");`

- [ ] Line 150: `error!("Failed to flush: {}", err);`
  - Rewrite: `error!(error = ?err, "Flush failed");`

- [ ] Line 287: `info!("LedgerTruncator: truncating slot range [{from_slot}; {to_slot}]");`
  - Rewrite: `info!(from_slot, to_slot, "Truncating slot range");`

- [ ] Line 325: `info!("LedgerTruncator: compacting slot range [{from_slot}; {to_slot}]");`
  - Rewrite: `info!(from_slot, to_slot, "Compacting slot range");`

- [ ] Macro `compact_cf_or_return!` lines 495-510
  - Rewrite: `info!(column_name = %<$cf as ColumnName>::NAME, "Shutdown during compaction");`

- [ ] Add `#[instrument(skip(self))]` to `run()` (line 57)

---

## Summary

This plan provides **2 critical async files to update** with concrete before/after examples. The magicblock-ledger crate is storage-focused with minimal async operations:

1. **blockstore_processor/mod.rs**: Two async functions for block replay with progress logging
2. **ledger_truncator.rs**: One main async function for background ledger maintenance

**Key characteristics**:
- Minimal logging in async paths
- Most logging is in synchronous utility functions (deferred to pass 2)
- Focus on slot/block numbers and error conditions
- Background service pattern with progress reporting

**Next steps**:
1. Apply `#[instrument]` to the two async functions
2. Rewrite all log messages in these files to constant format
3. Extract parameters to structured fields
4. Run validation checks (cargo check, lint, test)
5. Proceed to sync code pass if needed
