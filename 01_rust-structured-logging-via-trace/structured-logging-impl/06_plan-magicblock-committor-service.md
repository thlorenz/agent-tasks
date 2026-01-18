# Magicblock-Committor-Service Structured Logging Implementation Plan

Based on the deterministic pattern from `02_plan-general.md`, this document provides a concrete, step-by-step implementation guide specific to the `magicblock-committor-service` crate.

---

## Overview

The `magicblock-committor-service` crate handles intent execution, scheduling, transaction preparation, and persistence for the Committor service. Total files to update: **14 files** with async functions and logging statements.

### Architecture Summary

1. **Core Service**: Main actor-based service handling messages
   - `service.rs` - CommittorActor message handler and event loop
   - `service_ext.rs` - Result dispatcher for intent execution

2. **Intent Execution Management**: Scheduling and execution orchestration
   - `intent_execution_manager.rs` - High-level manager
   - `intent_execution_manager/intent_execution_engine.rs` - Main event loop and executor orchestration
   - `intent_execution_manager/db.rs` - Database abstraction

3. **Intent Execution Strategies**: Single-stage and two-stage execution
   - `intent_executor/mod.rs` - Core executor interface and main logic
   - `intent_executor/single_stage_executor.rs` - Optimized single-stage flow
   - `intent_executor/two_stage_executor.rs` - Two-stage commit/finalize flow
   - `intent_executor/task_info_fetcher.rs` - Account and metadata fetching

4. **Transaction Processing**: Preparation and delivery
   - `transaction_preparator/mod.rs` - Strategy preparation and cleanup
   - `transaction_preparator/delivery_preparator.rs` - Buffer account and transaction delivery
   - `committor_processor.rs` - Main processor orchestrating all components

5. **Testing Support**: Test stubs
   - `stubs/changeset_committor_stub.rs` - Testing helper

---

## Critical Files (Process First)

### File 1: `src/service_ext.rs`

**Priority**: 1 (No dependencies, foundational service extension)

**Async functions with logging**:
- `dispatcher()` (async, static function spawned as task)

**Current log statements**:

```rust
// Line 74 - shutdown signal
info!("Committor service stopped, stopping Committor extension");

// Line 81 - channel closed
info!("Intent execution got shutdown, shutting down result Committor extension!");

// Line 87 - lag detected
error!("CommittorServiceExt lags behind Intent execution! skipped: {}", skipped);

// Line 105-107 - send failure
error!("Failed to send BaseIntent execution result to listener");
```

**Implementation**:

1. Add `#[instrument(skip(pending_message, results_subscription))]` to `dispatcher()`
2. Rewrite log statements:
   ```rust
   // Line 74
   info!("Shutting down extension");
   
   // Line 81
   info!("Intent execution shutdown");
   
   // Line 87
   error!(skipped, "Dispatcher lag detected");
   
   // Line 105-107
   error!("Failed to send execution result");
   ```

---

### File 2: `src/service.rs`

**Priority**: 2 (Main service, handles messages)

**Async functions with logging**:
- `handle_msg()` (async)
- `run()` (async main event loop)

**Current log statements** (lines 140, 151, 162, 173, 183, 194, 209, 221, 227, 249):

```rust
// Line 140 (in spawned task)
error!("Failed to send response {:?}", e);

// Line 151 (in spawned task)
error!("Failed to send response {:?}", e);

// Line 162 (in spawned task)
error!("Failed to send response {:?}", e);

// Line 173 (in handler)
error!("Failed to send response {:?}", e);

// Line 183 (in handler)
error!("Failed to send response {:?}", e);

// Line 194 (in handler)
error!("Failed to send response {:?}", e);

// Line 209 (in spawned task)
error!( "Failed to send response for GetTransactionLogs: {:?}", err);

// Line 221 (in handler)
error!("Failed to send response {:?}", e);

// Line 227 (in handler)
error!("Failed to send response {:?}", err);

// Line 249 (in run loop)
info!("CommittorActor shutdown!");
```

**Implementation**:

1. Add `#[instrument(skip(self))]` to `handle_msg()`
2. Add `#[instrument(skip(self, cancel_token))]` to `run()`
3. Extract message type as field in log statements
4. Rewrite all response failures:
   ```rust
   error!(message_type = "ReservePubkeysForCommittee", error = ?e, "Failed to send response");
   ```
5. Rewrite shutdown:
   ```rust
   info!("Actor shutdown");
   ```

---

### File 3: `src/intent_execution_manager/intent_execution_engine.rs`

**Priority**: 3 (Core scheduling engine, critical logging)

**Key async functions**:
1. `main_loop()` - Main event loop
2. `next_scheduled_intent()` - Intent scheduling logic
3. `get_new_intent()` - Channel/DB reading
4. `execute()` - Executor wrapper

**Current log statements** (lines 144, 148, 156, 206, 219, 280, 293, 314, 342-345, 347):

```rust
// Line 144
info!("Channel closed, exiting IntentExecutionEngine::main_loop");

// Line 148
error!("Failed to fetch intent from db: {:?}", err);

// Line 156
trace!("Could not schedule any intents");

// Line 206
warn!("Scheduler capacity exceeded: {}", num_blocked_intents);

// Line 219
error!("Executor failed to complete: {}", err);

// Line 280
error!("Failed to execute BaseIntent. id: {}. {}", intent.id, err)

// Line 293
warn!("No result listeners of intent execution: {}", err);

// Line 314
error!("Failed to cleanup after intent: {}", err);

// Line 342-345
info!("Intent took too long to execute: {}s. {:?}", intent_execution_secs, result);

// Line 347
trace!("Seconds took to execute intent: {}", intent_execution_secs);
```

**Implementation**:

1. Add `#[instrument(skip(self, result_sender))]` to `main_loop()`
2. Add `#[instrument(skip(self))]` to `next_scheduled_intent()`
3. Add `#[instrument(skip(receiver, db))]` to `get_new_intent()` (static method - no span needed per pattern, but can add if called from async context)
4. Add `#[instrument(skip(executor, persister, inner_scheduler, execution_permit, result_sender))]` to `execute()` with fields:
   - `intent_id = intent.id`
   - `trigger_type = ?intent.trigger_type`

5. Rewrite log statements:
   ```rust
   // Line 144
   info!("Channel closed, exiting");
   
   // Line 148
   error!(error = ?err, "Failed to fetch intent");
   
   // Line 156
   trace!("No intents available");
   
   // Line 206
   warn!(blocked_count = num_blocked_intents, "Capacity exceeded");
   
   // Line 219
   error!(error = ?err, "Executor failed");
   
   // Line 280
   error!(intent_id = intent.id, error = ?err, "Execution failed");
   
   // Line 293
   warn!(error = ?err, "No result listeners");
   
   // Line 314
   error!(error = ?err, "Cleanup failed");
   
   // Line 342-345
   info!(duration_secs = intent_execution_secs, "Execution slow");
   
   // Line 347
   trace!(duration_secs = intent_execution_secs, "Execution time");
   ```

---

### File 4: `src/committor_processor.rs`

**Priority**: 4 (Processor orchestrating all components)

**Async functions with logging**:
- `schedule_base_intents()` - Entry point for scheduling

**Current log statements** (lines ~110, ~114):

```rust
// In schedule_base_intents
error!(
    "DB EXCEPTION: Failed to persist changeset to be committed: {:?}",
    // ...
);

// Line ~114
error!("Failed to schedule base intent: {}", err);
```

**Implementation**:

1. Add `#[instrument(skip(self))]` to `schedule_base_intents()`
2. Rewrite:
   ```rust
   error!(error = ?err, "Failed to persist changeset");
   
   error!(error = ?err, "Failed to schedule intent");
   ```

---

### File 5: `src/service_ext.rs` - `schedule_base_intents_waiting()`

**Part of File 1 but separate function**

**Priority**: 1b (Same file as dispatcher)

**Async function**:
- `schedule_base_intents_waiting()` - Waits for execution results

**Current log statements**: None in this function itself.

**Implementation**:

1. Add `#[instrument(skip(self, base_intents))]` to `schedule_base_intents_waiting()`

---

### File 6: `src/intent_executor/two_stage_executor.rs`

**Priority**: 5 (Heavy error handling, complex logic)

**Key async functions**:
1. `commit()` - Commit stage
2. `patch_commit_strategy()` - Error handling in commit
3. `finalize()` - Finalize stage
4. `patch_finalize_strategy()` - Error handling in finalize

**Current log statements** (lines ~103, ~120, ~149, ~166, ~190, ~220, ~278, ~307):

```rust
// Line ~103
error!("CRITICAL! Recursion ceiling reached in intent execution.");

// Line ~120
error!("RPC returned unexpected task index: {}. optimized_tasks_len: {}", err, optimized_tasks.len());

// Line ~149 (commit flow errors)
error!("Unexpected error in two stage commit flow: {}", err);

// Line ~166 (multiple places)
error!("Unexpected error in two stage commit flow: {}", err);

// Line ~190 (finalize errors)
error!("Commit tasks exceeded CpiLimitError: {}", err);

// Line ~220 (finalize errors)
error!("Unexpected error in two stage finalize flow: {}", err);

// Line ~278
warn!("Finalization tasks exceeded CpiLimitError: {}", err);
```

**Implementation**:

1. Add `#[instrument(skip(self, persister))]` to `commit()` with fields:
   - `stage = "commit"`

2. Add `#[instrument(skip(executor, execution_err), fields(stage = "commit"))]` to `patch_commit_strategy()`

3. Add `#[instrument(skip(self, persister))]` to `finalize()` with fields:
   - `stage = "finalize"`

4. Add `#[instrument(skip(executor, execution_err), fields(stage = "finalize"))]` to `patch_finalize_strategy()`

5. Rewrite log statements:
   ```rust
   // Recursion ceiling
   error!("Recursion ceiling reached");
   
   // Unexpected task index
   error!(task_index = err, total_tasks = optimized_tasks.len(), "Invalid task index");
   
   // Unexpected error patterns
   error!(error_type = "actions", "Unexpected error in flow");
   error!(error_type = "delegation", "Unexpected error in flow");
   error!(error_type = "cpi_limit", "Unexpected error in flow");
   
   // CPI limit exceeded
   warn!(error_type = "cpi_limit", "Limit exceeded");
   ```

---

### File 7: `src/intent_executor/task_info_fetcher.rs`

**Priority**: 6 (Account fetching, moderate logging)

**Async functions with logging**:
- Various `fetch_*` functions with retry logic

**Current log statements**:

```rust
// Unexpected error case
error!("Unexpected error: {:?}", err);
```

**Implementation**:

1. Add `#[instrument(skip(...))]` to all public async functions
2. Rewrite error:
   ```rust
   error!(error = ?err, "Unexpected error");
   ```

---

### File 8: `src/intent_executor/single_stage_executor.rs`

**Priority**: 7 (Single-stage executor)

**Async functions**: `execute()`, `patch_strategy()`, internal handlers

**Implementation**:

1. Add `#[instrument(skip(self, base_intent, persister))]` to async functions
2. Follow pattern from two-stage executor for error handling

---

### File 9: `src/intent_executor/mod.rs`

**Priority**: 8 (Core executor logic)

**Async functions**:
- `execute_inner()` - Main execution dispatcher
- `single_stage_execution_flow()` - Single-stage flow
- `two_stage_execution_flow()` - Two-stage flow
- `handle_commit_id_error()` - Error handler
- `send_prepared_message()` - Transaction sending

**Current log statements**:

```rust
// Line ~
warn!("TransactionPreparator v1 does not use Legacy message");
```

**Implementation**:

1. Add `#[instrument]` to all async functions
2. Rewrite warn:
   ```rust
   warn!("Legacy message detected");
   ```

---

### File 10: `src/transaction_preparator/delivery_preparator.rs`

**Priority**: 9 (Transaction delivery, complex error handling)

**Async functions**:
- `prepare_for_delivery()` - Main entry point
- `prepare_task()` - Task preparation
- `prepare_task_handling_errors()` - Error handling wrapper
- `initialize_buffer_account()` - Buffer initialization
- `write_buffer_with_retries()` - Retryable write
- `write_missing_chunks()` - Chunk writing
- `send_ixs_with_retry()` - Instruction retry
- `try_send_ixs()` - Send attempt
- `prepare_lookup_tables()` - Lookup table prep
- `cleanup()` - Cleanup handler

**Current log statements**:

```rust
// In error handler (info for account initialization)
info!("Account already initialized, setting up delegation");

// Other potential logs in error paths
```

**Implementation**:

1. Add `#[instrument(skip(...))]` to all async functions with meaningful parameters
2. Extract account addresses and signatures as fields
3. Rewrite any existing logs

---

### File 11: `src/transaction_preparator/mod.rs`

**Priority**: 10 (Strategy preparation wrapper)

**Async functions**:
- `prepare_for_strategy()` - Preparation dispatcher
- `cleanup_for_strategy()` - Cleanup dispatcher

**Implementation**:

1. Add `#[instrument(skip(...), fields(strategy_type = ?strategy))]` to both functions

---

### File 12: `src/intent_execution_manager.rs`

**Priority**: 11 (High-level manager)

**Async functions**:
- `schedule()` - Public API

**Current log statements**: Possible logs in error paths

**Implementation**:

1. Add `#[instrument(skip(...), fields(intent_count = base_intents.len()))]` if exists

---

### File 13: `src/intent_execution_manager/db.rs`

**Priority**: 12 (Database abstraction, minimal logging)

**Async functions**:
- Various DB trait methods

**Implementation**:

1. Add `#[instrument(skip(...))]` to concrete implementations
2. Extract identifiers as fields (intent IDs, etc.)

---

### File 14: `src/stubs/changeset_committor_stub.rs`

**Priority**: 13 (Test stub, low priority)

**Async functions**:
- Test stub implementations

**Implementation**:

1. Add `#[instrument]` to stub implementations for consistency

---

## Standard Field Naming for Magicblock-Committor-Service

| Field | Type | Format | Use Case |
|-------|------|--------|----------|
| `intent_id` | u64 | bare | Intent identifier |
| `trigger_type` | TriggerType | `?` | Trigger type (enum) |
| `message_type` | &str | bare | CommittorMessage variant |
| `stage` | &str | bare | Execution stage (commit/finalize) |
| `error` | Error | `?` | Error details |
| `error_type` | &str | bare | Error category |
| `blocked_count` | usize | bare | Number of blocked intents |
| `duration_secs` | f64 | bare | Duration in seconds |
| `duration_ms` | u64 | bare | Duration in milliseconds |
| `task_index` | usize | bare | Task index |
| `total_tasks` | usize | bare | Total task count |
| `intent_count` | usize | bare | Number of intents |
| `skipped` | u64 | bare | Number of skipped items |
| `strategy_type` | TransactionStrategy | `?` | Strategy enum |

---

## Before/After Examples

### Example 1: `intent_execution_manager/intent_execution_engine.rs` - `main_loop()`

**Before**:
```rust
async fn main_loop(
    mut self,
    result_sender: broadcast::Sender<BroadcastedIntentExecutionResult>,
) {
    loop {
        let intent = match self.next_scheduled_intent().await {
            Ok(value) => value,
            Err(IntentExecutionManagerError::ChannelClosed) => {
                info!("Channel closed, exiting IntentExecutionEngine::main_loop");
                break;
            }
            Err(IntentExecutionManagerError::DBError(err)) => {
                error!("Failed to fetch intent from db: {:?}", err);
                break;
            }
        };
        let Some(intent) = intent else {
            trace!("Could not schedule any intents");
            continue;
        };
        // ...
    }
}
```

**After**:
```rust
#[instrument(skip(self, result_sender))]
async fn main_loop(
    mut self,
    result_sender: broadcast::Sender<BroadcastedIntentExecutionResult>,
) {
    loop {
        let intent = match self.next_scheduled_intent().await {
            Ok(value) => value,
            Err(IntentExecutionManagerError::ChannelClosed) => {
                info!("Channel closed");
                break;
            }
            Err(IntentExecutionManagerError::DBError(err)) => {
                error!(error = ?err, "Failed to fetch intent");
                break;
            }
        };
        let Some(intent) = intent else {
            trace!("No intents available");
            continue;
        };
        // ...
    }
}
```

---

### Example 2: `service.rs` - `handle_msg()` and `run()`

**Before**:
```rust
async fn handle_msg(&self, msg: CommittorMessage) {
    use CommittorMessage::*;
    match msg {
        ReservePubkeysForCommittee { ... } => {
            tokio::task::spawn(async move {
                // ...
                if let Err(e) = respond_to.send(result) {
                    error!("Failed to send response {:?}", e);
                }
            });
        }
        // ... other variants with same error handling ...
    }
}

pub async fn run(&mut self, cancel_token: CancellationToken) {
    loop {
        select! {
            msg = self.receiver.recv() => {
                if let Some(msg) = msg {
                    self.handle_msg(msg).await;
                } else {
                    break;
                }
            }
            _ = cancel_token.cancelled() => {
                break;
            }
        }
    }
    info!("CommittorActor shutdown!");
}
```

**After**:
```rust
#[instrument(skip(self))]
async fn handle_msg(&self, msg: CommittorMessage) {
    use CommittorMessage::*;
    match msg {
        ReservePubkeysForCommittee { ... } => {
            tokio::task::spawn(async move {
                // ...
                if let Err(e) = respond_to.send(result) {
                    error!(message_type = "ReservePubkeys", error = ?e, "Failed to send response");
                }
            });
        }
        ScheduleBaseIntents { ... } => {
            // ...
            if let Err(e) = respond_to.send(result) {
                error!(message_type = "ScheduleIntents", error = ?e, "Failed to send response");
            }
        }
        // ... other variants ...
    }
}

#[instrument(skip(self, cancel_token))]
pub async fn run(&mut self, cancel_token: CancellationToken) {
    loop {
        select! {
            msg = self.receiver.recv() => {
                if let Some(msg) = msg {
                    self.handle_msg(msg).await;
                } else {
                    break;
                }
            }
            _ = cancel_token.cancelled() => {
                break;
            }
        }
    }
    info!("Actor shutdown");
}
```

---

### Example 3: `service_ext.rs` - `dispatcher()`

**Before**:
```rust
async fn dispatcher(
    committor_stopped: WaitForCancellationFutureOwned,
    results_subscription: oneshot::Receiver<
        broadcast::Receiver<BroadcastedIntentExecutionResult>,
    >,
    pending_message: Arc<Mutex<HashMap<u64, MessageResultListener>>>,
) {
    let mut results_subscription = results_subscription.await.unwrap();

    tokio::pin!(committor_stopped);
    loop {
        let execution_result = tokio::select! {
            biased;
            _ = &mut committor_stopped => {
                info!("Committor service stopped, stopping Committor extension");
                return;
            }
            execution_result = results_subscription.recv() => {
                match execution_result {
                    Ok(result) => result,
                    Err(broadcast::error::RecvError::Closed) => {
                        info!("Intent execution got shutdown, shutting down result Committor extension!");
                        break;
                    }
                    Err(broadcast::error::RecvError::Lagged(skipped)) => {
                        error!("CommittorServiceExt lags behind Intent execution! skipped: {}", skipped);
                        continue;
                    }
                }
            }
        };
        
        if sender.send(execution_result).is_err() {
            error!("Failed to send BaseIntent execution result to listener");
        }
    }
}
```

**After**:
```rust
#[instrument(skip(pending_message, results_subscription))]
async fn dispatcher(
    committor_stopped: WaitForCancellationFutureOwned,
    results_subscription: oneshot::Receiver<
        broadcast::Receiver<BroadcastedIntentExecutionResult>,
    >,
    pending_message: Arc<Mutex<HashMap<u64, MessageResultListener>>>,
) {
    let mut results_subscription = results_subscription.await.unwrap();

    tokio::pin!(committor_stopped);
    loop {
        let execution_result = tokio::select! {
            biased;
            _ = &mut committor_stopped => {
                info!("Shutting down extension");
                return;
            }
            execution_result = results_subscription.recv() => {
                match execution_result {
                    Ok(result) => result,
                    Err(broadcast::error::RecvError::Closed) => {
                        info!("Intent execution shutdown");
                        break;
                    }
                    Err(broadcast::error::RecvError::Lagged(skipped)) => {
                        error!(skipped, "Dispatcher lag detected");
                        continue;
                    }
                }
            }
        };
        
        if sender.send(execution_result).is_err() {
            error!("Failed to send result");
        }
    }
}
```

---

## Implementation Order

Process files in this exact order (respecting dependencies):

1. `service_ext.rs` - No dependencies, simple dispatcher
2. `service.rs` - Depends on nothing, main service actor
3. `committor_processor.rs` - Simple orchestrator
4. `intent_execution_manager/intent_execution_engine.rs` - Core scheduling engine
5. `intent_execution_manager.rs` - Wrapper around engine
6. `intent_execution_manager/db.rs` - Database abstraction
7. `intent_executor/task_info_fetcher.rs` - Account fetching
8. `intent_executor/two_stage_executor.rs` - Complex error handling
9. `intent_executor/single_stage_executor.rs` - Similar to two-stage
10. `intent_executor/mod.rs` - Core executor logic
11. `transaction_preparator/delivery_preparator.rs` - Transaction delivery
12. `transaction_preparator/mod.rs` - Preparation wrapper
13. `stubs/changeset_committor_stub.rs` - Test stubs (last)

---

## Validation Steps

### 1. Syntax Check
```bash
cd magicblock-committor-service
cargo check
```

### 2. Lint
```bash
make lint
```

### 3. Tests
```bash
cargo nextest run -p magicblock-committor-service -j16
```

### 4. Log Inspection
```bash
RUST_LOG=debug cargo test --package magicblock-committor-service --lib 2>&1 | head -150
```

**Verify**:
- Constant message text (no variables in message)
- Fields in span context (before the colon)
- No duplicate info (in both message and field)
- All async functions have `#[instrument]` or are test functions
- Intent IDs and error types visible in field context

---

## Key Implementation Notes

### Span Naming Conventions

Function names automatically become span names via `#[instrument]`:
- `main_loop` → span "main_loop"
- `handle_msg` → span "handle_msg"
- `dispatcher` → span "dispatcher"

### Field Recording for Late-Binding Values

For fields not available at function entry, use `tracing::Span::current().record()`:

```rust
#[instrument(
    skip(self, update),
    fields(
        intent_id = tracing::field::Empty,
        pubkey = tracing::field::Empty,
    )
)]
async fn process_update(&self, update: Update) {
    let intent_id = extract_intent_id(&update);
    tracing::Span::current().record("intent_id", intent_id);
    
    let pubkey = extract_pubkey(&update);
    tracing::Span::current().record("pubkey", tracing::field::display(pubkey));
}
```

### Message Types as Fields

When handling different message variants in `CommittorActor::handle_msg()`, include the message type as a field to distinguish which operation failed:

```rust
match msg {
    CommittorMessage::ReservePubkeysForCommittee { ... } => {
        if let Err(e) = respond_to.send(result) {
            error!(message_type = "ReservePubkeys", error = ?e, "Failed to send response");
        }
    }
    CommittorMessage::ScheduleBaseIntents { ... } => {
        if let Err(e) = respond_to.send(result) {
            error!(message_type = "ScheduleIntents", error = ?e, "Failed to send response");
        }
    }
}
```

### Execution Stages

For two-stage and single-stage executors, always include the `stage` field:

```rust
#[instrument(skip(self, persister), fields(stage = "commit"))]
async fn commit<P: IntentPersister>(&mut self, ...) {
    // ...
}

#[instrument(skip(self, persister), fields(stage = "finalize"))]
async fn finalize<P: IntentPersister>(&mut self, ...) {
    // ...
}
```

---

## Summary

This plan provides:
- **14 files to update**
- **13 priority groups** with clear dependencies
- **3 concrete before/after examples** from representative functions
- **Standard field naming** specific to committor-service domain
- **Implementation order** respecting dependencies and complexity

Follow the implementation order, apply the pattern to each file, and validate with the provided steps. Result: Clean, queryable, aggregatable structured logging across magicblock-committor-service.
