# Magicblock-Processor Structured Logging Implementation Plan

Based on the deterministic pattern from `02_plan-general.md`, this document provides a concrete, step-by-step implementation guide specific to the `magicblock-processor` crate.

---

## Overview

The `magicblock-processor` crate is responsible for transaction execution and scheduling. It is a relatively small, focused crate with **2 main async functions** and **7 log statements** distributed across **4 files**.

### File Categories

1. **Core Components**:
   - `scheduler/mod.rs` - Main scheduler event loop
   - `executor/mod.rs` - Transaction executor event loop
   - `executor/processing.rs` - Transaction processing logic
   - `loader.rs` - Program loader utility

The crate has minimal logging compared to other magicblock components, making it a straightforward migration.

---

## Critical Files (Process In Order)

### File 1: `src/loader.rs`

**Priority**: 1 (No async functions, simple sync logging)

**Async functions with logging**: None (synchronous only)

**Current log statements**:

```rust
// Line 19 - debug on load start
debug!("Loading programs from files: {:#?}", progs);
```

**Implementation**:

1. This is a **synchronous function**, so no `#[instrument]` is added (by design - sync code doesn't get spans per the pattern).
2. Rewrite line 19:
   - `debug!(program_count = progs.len(), "Loading programs");`

**Before/After**:

```rust
// Before
pub fn load_upgradeable_programs(
    accountsdb: &AccountsDb,
    progs: &[(Pubkey, PathBuf)],
) -> Result<(), Box<dyn Error>> {
    debug!("Loading programs from files: {:#?}", progs);
    // ...
}

// After
pub fn load_upgradeable_programs(
    accountsdb: &AccountsDb,
    progs: &[(Pubkey, PathBuf)],
) -> Result<(), Box<dyn Error>> {
    debug!(program_count = progs.len(), "Loading programs");
    // ...
}
```

---

### File 2: `src/executor/mod.rs`

**Priority**: 2 (First async function, no dependencies)

**Key async functions**:
1. `run()` - Main executor event loop (async, critical)

**Current log statements**:

```rust
// Line 166 - info on termination
info!("Transaction executor {} terminated", self.id);

// Line 198 - warn on serialization failure  
warn!("Failed to serialize slot hashes: {e}");
```

**Implementation**:

1. Add `#[instrument(skip(self))]` to `run()` with field `executor_id = self.id`
2. Rewrite line 166:
   - `info!("Executor terminated");`
3. Rewrite line 198:
   - `warn!(error = ?e, "Failed to serialize slot hashes");`

**Before/After - `run()` function**:

```rust
// Before
async fn run(mut self) {
    let mut guard = self.sync.read();
    let mut block_updated = self.block.subscribe();

    loop {
        tokio::select! {
            // ... select arms ...
        }
    }
    info!("Transaction executor {} terminated", self.id);
}

// After
#[instrument(skip(self), fields(executor_id = self.id))]
async fn run(mut self) {
    let mut guard = self.sync.read();
    let mut block_updated = self.block.subscribe();

    loop {
        tokio::select! {
            // ... select arms ...
        }
    }
    info!("Executor terminated");
}
```

**Note on `set_sysvars()` function**:

The `set_sysvars()` method at line 177 contains the serialization error on line 198. This is a synchronous helper function, so only the error log statement needs conversion (no `#[instrument]` needed).

---

### File 3: `src/executor/processing.rs`

**Priority**: 3 (Synchronous methods in executor, depends on executor being defined)

**Async functions with logging**: None (all methods are synchronous)

**Current log statements**:

```rust
// Line 173 - error on task send failure
error!("Scheduled tasks service disconnected: {e}");

// Line 250 - error on ledger commit failure
error!("failed to commit transaction to the ledger: {error}");
```

**Implementation**:

1. These are in synchronous methods (`process_scheduled_tasks()` and `write_to_ledger()`), so no `#[instrument]` is added.
2. Rewrite line 173:
   - `error!(error = ?e, "Scheduled tasks service disconnected");`
3. Rewrite line 250:
   - `error!(error = ?error, "Failed to commit transaction to ledger");`

**Before/After - `process_scheduled_tasks()` method**:

```rust
// Before
fn process_scheduled_tasks(&self) {
    while let Some(task) = ExecutionTlsStash::next_task() {
        if let Err(e) = self.tasks_tx.send(task) {
            error!("Scheduled tasks service disconnected: {e}");
        }
    }
}

// After
fn process_scheduled_tasks(&self) {
    while let Some(task) = ExecutionTlsStash::next_task() {
        if let Err(e) = self.tasks_tx.send(task) {
            error!(error = ?e, "Scheduled tasks service disconnected");
        }
    }
}
```

**Before/After - `write_to_ledger()` method**:

```rust
// Before
fn write_to_ledger(
    &self,
    txn: SanitizedTransaction,
    meta: TransactionStatusMeta,
) {
    let signature = *txn.signature();
    let index = match self.ledger.write_transaction(
        signature,
        self.processor.slot,
        &txn,
        meta.clone(),
    ) {
        Ok(i) => i,
        Err(error) => {
            error!("failed to commit transaction to the ledger: {error}");
            return;
        }
    };
    // ...
}

// After
fn write_to_ledger(
    &self,
    txn: SanitizedTransaction,
    meta: TransactionStatusMeta,
) {
    let signature = *txn.signature();
    let index = match self.ledger.write_transaction(
        signature,
        self.processor.slot,
        &txn,
        meta.clone(),
    ) {
        Ok(i) => i,
        Err(error) => {
            error!(error = ?error, "Failed to commit transaction to ledger");
            return;
        }
    };
    // ...
}
```

---

### File 4: `src/scheduler/mod.rs`

**Priority**: 4 (Second async function, main scheduler loop)

**Key async functions**:
1. `run()` - Main scheduler event loop (async, critical)

**Current log statements**:

```rust
// Line 153 - info on termination
info!("Transaction scheduler has terminated");

// Line 224 - error on executor send failure
error!("Executor {executor} has shutdown or crashed, should not be possible: {e}")
```

**Implementation**:

1. Add `#[instrument(skip(self))]` to `run()` (no additional fields needed beyond the span name)
2. Rewrite line 153:
   - `info!("Scheduler terminated");`
3. Rewrite line 224:
   - `error!(executor = executor, error = ?e, "Executor channel send failed");`

**Before/After - `run()` function**:

```rust
// Before
async fn run(mut self) {
    let mut block_produced = self.latest_block.subscribe();
    loop {
        tokio::select! {
            biased;
            // ... select arms ...
        }
    }
    drop(self.executors);
    self.ready_rx.recv().await;
    info!("Transaction scheduler has terminated");
}

// After
#[instrument(skip(self))]
async fn run(mut self) {
    let mut block_produced = self.latest_block.subscribe();
    loop {
        tokio::select! {
            biased;
            // ... select arms ...
        }
    }
    drop(self.executors);
    self.ready_rx.recv().await;
    info!("Scheduler terminated");
}
```

**Note on `schedule_transaction()` method**:

The error log at line 224 is in the synchronous method `schedule_transaction()`, which uses `inspect_err()`. This is a simple conversion that doesn't require `#[instrument]`.

---

## Standard Field Naming for Magicblock-Processor

| Field | Type | Format | Use Case |
|-------|------|--------|----------|
| `executor_id` | u32 | bare | Transaction executor identifier |
| `program_count` | usize | bare | Number of programs |
| `error` | Error | `?` | Error details |
| `slot` | u64 | bare | Block slot |

---

## Summary of Changes

| File | Functions | Log Statements | Changes |
|------|-----------|-----------------|---------|
| `loader.rs` | 0 async | 1 | Convert 1 log to structured fields |
| `executor/mod.rs` | 1 async | 2 | Add `#[instrument]` + convert 2 logs |
| `executor/processing.rs` | 0 async | 2 | Convert 2 logs to structured fields |
| `scheduler/mod.rs` | 1 async | 2 | Add `#[instrument]` + convert 2 logs |
| **TOTAL** | **2 async** | **7 logs** | **7 conversions** |

---

## Implementation Order

Process files in this exact order:

1. **`loader.rs`** - No async, simple debug statement
2. **`executor/mod.rs`** - First async function (`run()`)
3. **`executor/processing.rs`** - Synchronous error logs in processing module
4. **`scheduler/mod.rs`** - Second async function (`run()`)

---

## Validation Steps

### 1. Syntax Check
```bash
cd magicblock-processor
cargo check
```

### 2. Lint
```bash
make lint
```

### 3. Tests
```bash
cargo nextest run -p magicblock-processor
```

### 4. Log Inspection
```bash
RUST_LOG=info cargo test --package magicblock-processor --lib 2>&1 | grep -E "info|warn|error|debug"
```

**Verify**:
- Constant message text (no variables in message)
- Fields in span context (before the colon)
- No parameters embedded in message strings
- Executor ID and error details are in fields, not in message

---

## Detailed Implementation Steps

### Step 1: Update `loader.rs`

**Changes**:
- Line 19: Convert debug statement
- No async functions, so no `#[instrument]` needed

**Task**:
```rust
// BEFORE
debug!("Loading programs from files: {:#?}", progs);

// AFTER
debug!(program_count = progs.len(), "Loading programs");
```

---

### Step 2: Update `executor/mod.rs`

**Changes**:
- Line 26: Import `instrument` from tracing (if not present)
- Line 135: Add `#[instrument]` attribute to `run()`
- Line 166: Convert info log
- Line 198: Convert warn log

**Task**:
```rust
// Add to imports (if not already there)
use tracing::{info, warn, instrument};

// BEFORE run() function
async fn run(mut self) {

// AFTER run() function
#[instrument(skip(self), fields(executor_id = self.id))]
async fn run(mut self) {

// Line 166: BEFORE
info!("Transaction executor {} terminated", self.id);

// Line 166: AFTER
info!("Executor terminated");

// Line 198: BEFORE
warn!("Failed to serialize slot hashes: {e}");

// Line 198: AFTER
warn!(error = ?e, "Failed to serialize slot hashes");
```

---

### Step 3: Update `executor/processing.rs`

**Changes**:
- Line 173: Convert error log in `process_scheduled_tasks()`
- Line 250: Convert error log in `write_to_ledger()`

**Task**:
```rust
// Line 173: BEFORE
error!("Scheduled tasks service disconnected: {e}");

// Line 173: AFTER
error!(error = ?e, "Scheduled tasks service disconnected");

// Line 250: BEFORE
error!("failed to commit transaction to the ledger: {error}");

// Line 250: AFTER
error!(error = ?error, "Failed to commit transaction to ledger");
```

---

### Step 4: Update `scheduler/mod.rs`

**Changes**:
- Line 25: Import `instrument` from tracing (if not present)
- Line 126: Add `#[instrument]` attribute to `run()`
- Line 153: Convert info log
- Line 224: Convert error log in `schedule_transaction()`

**Task**:
```rust
// Add to imports (if not already there)
use tracing::{error, info, instrument};

// BEFORE run() function
async fn run(mut self) {

// AFTER run() function
#[instrument(skip(self))]
async fn run(mut self) {

// Line 153: BEFORE
info!("Transaction scheduler has terminated");

// Line 153: AFTER
info!("Scheduler terminated");

// Line 224: BEFORE (in schedule_transaction method)
error!("Executor {executor} has shutdown or crashed, should not be possible: {e}")

// Line 224: AFTER (in schedule_transaction method)
error!(executor = executor, error = ?e, "Executor channel send failed")
```

---

## Notes and Considerations

### Why So Few Logs?

The `magicblock-processor` crate is intentionally lean on logging. It focuses on:
- Execution and scheduling machinery
- Account and transaction state changes

This is appropriate—transaction processing is fast and critical path. Detailed logging is delegated to higher-level orchestration (scheduler/executor creation, coordination).

### Sync vs Async Logging

**Key insight**: Synchronous functions do NOT get `#[instrument]` by design. Only the two main async event loops get spans:
- `executor::TransactionExecutor::run()`
- `scheduler::TransactionScheduler::run()`

All other logging (in synchronous methods) will inherit the span context from the async function they're called from.

### Field Values

Most logs here carry error details or executor identifiers. Fields are minimal:
- `executor_id` - Identifies which executor
- `program_count` - How many programs loaded
- `error` - Error details (only field with `?` format)

---

## Checklist

Use this for verification:

### Pre-Implementation
- [ ] Identified 2 async functions with logging
- [ ] Identified 7 log statements total
- [ ] Reviewed field naming (executor_id, program_count, error)
- [ ] Reviewed all before/after examples above

### During Implementation
- [ ] Added `#[instrument]` to both `run()` functions
- [ ] Converted all 7 log statements to structured fields
- [ ] Ensured log messages are constant
- [ ] Extracted all parameters to fields
- [ ] Used correct format specifiers (`?` for error, bare for counts/ids)

### Post-Implementation
- [ ] `cargo check` passes
- [ ] `make lint` passes
- [ ] `cargo nextest run -p magicblock-processor` passes
- [ ] All log messages are constant (no format variables in message text)
- [ ] All context is in fields (pubkey, slot, error, etc.)
- [ ] Code patterns match the examples exactly

---

## Summary

This plan updates **4 files** with **7 log conversions** and **2 `#[instrument]` annotations**.

The crate is small and focused, making this a straightforward migration that follows the deterministic pattern exactly. After implementation, all logging in magicblock-processor will be structured, queryable, and aggregatable across all execution paths.
