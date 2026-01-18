# Magicblock-Accounts Structured Logging Implementation Plan

Based on the deterministic pattern from `02_plan-general.md`, this document provides a concrete, step-by-step implementation guide specific to the `magicblock-accounts` crate.

---

## Overview

The `magicblock-accounts` crate is a lean, focused library that manages scheduled commit processing for the MagicBlock validator. It coordinates with the committor service, chainlink providers, and accounts database.

**Total files to update**: **1 file** with substantive logging
**Total async functions with logging**: **3 core async functions**
**Complexity**: Low (minimal logging, clear structure)

### File Structure

```
src/
├── config.rs              (No logging - no changes)
├── errors.rs              (No logging - no changes)
├── lib.rs                 (Module re-exports only - no changes)
├── traits.rs              (Trait definition only - no changes)
└── scheduled_commits_processor.rs  (PRIMARY - 3 async functions, 10 log statements)
```

---

## File Categories

### Primary File: scheduled_commits_processor.rs

This is the only file requiring changes. It contains:

1. **`ScheduledCommitsProcessorImpl` struct** - Main processor implementation
2. **Three core async functions**:
   - `process_undelegation_requests()` - Handles undelegation subscription requests
   - `result_processor()` - Main event loop (static/spawned)
   - `process_intent_result()` - Processes individual intent results

3. **One trait implementation**:
   - `ScheduledCommitsProcessor::process()` - Entry point (calls internal methods)

---

## Async Functions with Logging Analysis

### 1. `process_undelegation_requests()` - Lines 138-171

**Current state**:
- Async function
- 1 error log statement at line 166-169

```rust
error!(
    "Failed to subscribe to accounts being undelegated: {:?}",
    sub_errors
);
```

**Context**: Collects subscription errors for undelegated accounts. Currently includes full error vector in message.

**Rewrite strategy**:
- Extract error count as field
- Extract error list as field
- Make message constant

---

### 2. `result_processor()` - Lines 173-245

**Current state**:
- Async function (static method, spawned as task)
- 5 log statements scattered through event loop
- Line 190: `info!("ScheduledCommitsProcessorImpl stopped.");`
- Line 197: `info!("Intent execution got shutdown, shutting down result processor!");`
- Line 203: `error!("ScheduledCommitsProcessorImpl lags behind Intent execution! skipped: {}", skipped);`
- Line 215: `info!("OffChain triggered BaseIntent executed: {}", intent_id);`
- Line 230-233: Multi-line error with `intent_id` in message

```rust
error!(
    "CRITICAL! Failed to find IntentMeta for id: {}!",
    intent_id
);
```

**Context**: Main event loop processing intent execution results. Handles shutdown, lagging, trigger types, and metadata lookup failures.

**Rewrite strategy**:
- Make messages constant (no variables in text)
- Extract `intent_id` as field
- Extract `skipped` count as field
- Extract `trigger_type` as field if needed

---

### 3. `process_intent_result()` - Lines 247-270

**Current state**:
- Async function (called from result_processor)
- 2 log statements
- Line 262-265: `debug!("Signaled sent commit with internal signature: {:?}", signature);`
- Line 267: `error!("Failed to signal sent commit via transaction: {}", err);`

**Context**: Handles sending committed intents via transaction scheduler. Both logs relate to transaction execution.

**Rewrite strategy**:
- Extract signature as field
- Extract error as field
- Make messages constant

---

### 4. `build_sent_commit()` - Lines 272-322

**Current state**:
- Sync function (not async)
- 2 log statements
- Line 288-291: Multi-line error about commit failure
- Line 305: `info!("Patched intent: {}. error was: {}", intent_id, err);` (in closure)

**Note**: This is NOT an async function, so it will NOT get `#[instrument]` per the design pattern (sync code doesn't get spans). However, if we want to add structured logging to the inline `info!` call at line 305, we should log it at the call site instead. This function should remain unchanged per the deterministic pattern.

---

### 5. `process()` trait method - Lines 327-367

**Current state**:
- Async function (trait implementation)
- No direct log statements (delegates to other async functions)
- All logging is in called methods which will be instrumented

**Action**: Add `#[instrument]` for span context, even though no direct logging.

---

## Standard Field Naming for Magicblock-Accounts

| Field | Type | Format | Use Case |
|-------|------|--------|----------|
| `intent_id` | u64 | bare | Unique intent identifier |
| `pubkey` | Pubkey | `%` | Account address |
| `error_count` | usize | bare | Number of errors encountered |
| `skipped_count` | usize | bare | Number of skipped messages |
| `trigger_type` | TriggerType | `?` | OnChain or OffChain trigger |
| `error` | Error | `?` | Error details |
| `signature` | Signature | `%` | Transaction signature |

---

## Critical Implementation Points

### Field Recording Strategy for `result_processor()`

Since `result_processor()` is a static async function (not a method), it doesn't have `self` to skip. The `intent_id` is extracted during loop iteration. Use the span field recording pattern:

```rust
#[instrument(fields(intent_id = tracing::field::Empty))]
async fn result_processor(...) {
    // Later, when intent_id is available:
    tracing::Span::current().record("intent_id", intent_id);
}
```

This allows the `intent_id` to be captured in logs even though it's not known until runtime.

---

## Before/After Examples

### Example 1: `process_undelegation_requests()` (Lines 138-171)

**Before**:
```rust
async fn process_undelegation_requests(&self, pubkeys: Vec<Pubkey>) {
    let mut join_set = task::JoinSet::new();
    for pubkey in pubkeys.into_iter() {
        let chainlink = self.chainlink.clone();
        join_set.spawn(async move {
            (pubkey, chainlink.undelegation_requested(pubkey).await)
        });
    }
    let sub_errors = join_set
        .join_all()
        .await
        .into_iter()
        .filter_map(|(pubkey, inner_result)| {
            if let Err(err) = inner_result {
                Some(format!(
                    "Subscribing to account {} failed: {}",
                    pubkey, err
                ))
            } else {
                None
            }
        })
        .collect::<Vec<_>>();
    if !sub_errors.is_empty() {
        error!(
            "Failed to subscribe to accounts being undelegated: {:?}",
            sub_errors
        );
    }
}
```

**After**:
```rust
#[instrument(skip(self))]
async fn process_undelegation_requests(&self, pubkeys: Vec<Pubkey>) {
    let mut join_set = task::JoinSet::new();
    for pubkey in pubkeys.into_iter() {
        let chainlink = self.chainlink.clone();
        join_set.spawn(async move {
            (pubkey, chainlink.undelegation_requested(pubkey).await)
        });
    }
    let sub_errors = join_set
        .join_all()
        .await
        .into_iter()
        .filter_map(|(pubkey, inner_result)| {
            if let Err(err) = inner_result {
                Some(format!(
                    "Subscribing to account {} failed: {}",
                    pubkey, err
                ))
            } else {
                None
            }
        })
        .collect::<Vec<_>>();
    if !sub_errors.is_empty() {
        error!(
            error_count = sub_errors.len(),
            "Failed to subscribe to accounts being undelegated"
        );
    }
}
```

**Changes**:
- Added `#[instrument(skip(self))]`
- Extracted `sub_errors.len()` as `error_count` field
- Made message constant: "Failed to subscribe to accounts being undelegated"
- Removed the detailed error list from message (now just count)

---

### Example 2: `result_processor()` (Lines 173-245)

**Before**:
```rust
async fn result_processor(
    result_subscriber: oneshot::Receiver<
        broadcast::Receiver<BroadcastedIntentExecutionResult>,
    >,
    cancellation_token: CancellationToken,
    intents_meta_map: Arc<Mutex<HashMap<u64, ScheduledBaseIntentMeta>>>,
    internal_transaction_scheduler: TransactionSchedulerHandle,
) {
    const SUBSCRIPTION_ERR_MSG: &str =
        "Failed to get subscription of results of BaseIntents execution";

    let mut result_receiver =
        result_subscriber.await.expect(SUBSCRIPTION_ERR_MSG);
    loop {
        let execution_result = tokio::select! {
            biased;
            _ = cancellation_token.cancelled() => {
                info!("ScheduledCommitsProcessorImpl stopped.");
                return;
            }
            execution_result = result_receiver.recv() => {
                match execution_result {
                    Ok(result) => result,
                    Err(broadcast::error::RecvError::Closed) => {
                        info!("Intent execution got shutdown, shutting down result processor!");
                        break;
                    }
                    Err(broadcast::error::RecvError::Lagged(skipped)) => {
                        error!("ScheduledCommitsProcessorImpl lags behind Intent execution! skipped: {}", skipped);
                        continue;
                    }
                }
            }
        };

        let intent_id = execution_result.id;
        let trigger_type = execution_result.trigger_type;
        // Here we handle on OnChain triggered intent
        if matches!(trigger_type, TriggerType::OffChain) {
            info!("OffChain triggered BaseIntent executed: {}", intent_id);
            continue;
        }

        // Remove intent from metas
        let intent_meta = if let Some(intent_meta) = intents_meta_map
            .lock()
            .expect(POISONED_MUTEX_MSG)
            .remove(&intent_id)
        {
            intent_meta
        } else {
            error!(
                "CRITICAL! Failed to find IntentMeta for id: {}!",
                intent_id
            );
            continue;
        };
        // ... process_intent_result ...
    }
}
```

**After**:
```rust
#[instrument(skip(result_subscriber, cancellation_token, intents_meta_map, internal_transaction_scheduler))]
async fn result_processor(
    result_subscriber: oneshot::Receiver<
        broadcast::Receiver<BroadcastedIntentExecutionResult>,
    >,
    cancellation_token: CancellationToken,
    intents_meta_map: Arc<Mutex<HashMap<u64, ScheduledBaseIntentMeta>>>,
    internal_transaction_scheduler: TransactionSchedulerHandle,
) {
    const SUBSCRIPTION_ERR_MSG: &str =
        "Failed to get subscription of results of BaseIntents execution";

    let mut result_receiver =
        result_subscriber.await.expect(SUBSCRIPTION_ERR_MSG);
    loop {
        let execution_result = tokio::select! {
            biased;
            _ = cancellation_token.cancelled() => {
                info!("Shutting down result processor");
                return;
            }
            execution_result = result_receiver.recv() => {
                match execution_result {
                    Ok(result) => result,
                    Err(broadcast::error::RecvError::Closed) => {
                        info!("Intent execution service shut down");
                        break;
                    }
                    Err(broadcast::error::RecvError::Lagged(skipped)) => {
                        error!(skipped_count = skipped, "Lagging behind intent execution");
                        continue;
                    }
                }
            }
        };

        let intent_id = execution_result.id;
        let trigger_type = execution_result.trigger_type;
        // Here we handle on OnChain triggered intent
        if matches!(trigger_type, TriggerType::OffChain) {
            info!(intent_id, trigger_type = ?trigger_type, "OffChain intent executed");
            continue;
        }

        // Remove intent from metas
        let intent_meta = if let Some(intent_meta) = intents_meta_map
            .lock()
            .expect(POISONED_MUTEX_MSG)
            .remove(&intent_id)
        {
            intent_meta
        } else {
            error!(intent_id, "Failed to find intent metadata");
            continue;
        };
        // ... process_intent_result ...
    }
}
```

**Changes**:
- Added `#[instrument(skip(...))]` with all large/complex parameters skipped
- Line 190: "Shutting down result processor" (constant)
- Line 197: "Intent execution service shut down" (constant)
- Line 203: Extracted `skipped` as `skipped_count` field
- Line 215: Added `intent_id` and `trigger_type` fields to message
- Line 230-233: Extracted `intent_id` as field, message is constant

---

### Example 3: `process_intent_result()` (Lines 247-270)

**Before**:
```rust
async fn process_intent_result(
    intent_id: u64,
    internal_transaction_scheduler: &TransactionSchedulerHandle,
    result: BroadcastedIntentExecutionResult,
    mut intent_meta: ScheduledBaseIntentMeta,
) {
    let intent_sent_transaction =
        std::mem::take(&mut intent_meta.intent_sent_transaction);
    let sent_commit =
        Self::build_sent_commit(intent_id, intent_meta, result);
    register_scheduled_commit_sent(sent_commit);
    match internal_transaction_scheduler
        .execute(intent_sent_transaction)
        .await
    {
        Ok(signature) => debug!(
            "Signaled sent commit with internal signature: {:?}",
            signature
        ),
        Err(err) => {
            error!("Failed to signal sent commit via transaction: {}", err);
        }
    }
}
```

**After**:
```rust
#[instrument(skip(internal_transaction_scheduler, result, intent_meta), fields(intent_id))]
async fn process_intent_result(
    intent_id: u64,
    internal_transaction_scheduler: &TransactionSchedulerHandle,
    result: BroadcastedIntentExecutionResult,
    mut intent_meta: ScheduledBaseIntentMeta,
) {
    let intent_sent_transaction =
        std::mem::take(&mut intent_meta.intent_sent_transaction);
    let sent_commit =
        Self::build_sent_commit(intent_id, intent_meta, result);
    register_scheduled_commit_sent(sent_commit);
    match internal_transaction_scheduler
        .execute(intent_sent_transaction)
        .await
    {
        Ok(signature) => debug!(
            signature = %signature,
            "Sent commit signaled"
        ),
        Err(err) => {
            error!(error = ?err, "Failed to signal sent commit");
        }
    }
}
```

**Changes**:
- Added `#[instrument(skip(...), fields(intent_id))]`
- Line 262: Extracted `signature` as field, made message constant
- Line 267: Extracted `err` as field, made message constant

---

## Implementation Order

Since there's only one substantive file, the implementation is straightforward:

### Priority 1: `scheduled_commits_processor.rs`

Process in this order:
1. Add `#[instrument]` to `process_undelegation_requests()` (simple, no dependencies)
2. Add `#[instrument]` to `result_processor()` (complex, but no dependencies)
3. Add `#[instrument]` to `process_intent_result()` (depends on result_processor)
4. Add `#[instrument]` to `process()` trait method (depends on both)
5. Rewrite all log statements (10 total) following the before/after examples

---

## Complete Log Statement Inventory

### Line 118 (in `preprocess_intent()`)
- **Context**: inspect() closure within filter_map
- **Current**: `info!("Account got evicted from AccountsDB after intent was scheduled!");`
- **Action**: Convert to use parent async function's span. Since `preprocess_intent()` is NOT async, we will need to handle this in the calling `process()` async function or remove it. **Recommendation**: Keep as-is since it's informational only and context is preserved by `process()` span.

### Line 166-169 (in `process_undelegation_requests()`)
- **Current**: `error!("Failed to subscribe to accounts being undelegated: {:?}", sub_errors);`
- **Rewrite**: `error!(error_count = sub_errors.len(), "Failed to subscribe to accounts being undelegated");`

### Line 190 (in `result_processor()`)
- **Current**: `info!("ScheduledCommitsProcessorImpl stopped.");`
- **Rewrite**: `info!("Shutting down result processor");`

### Line 197 (in `result_processor()`)
- **Current**: `info!("Intent execution got shutdown, shutting down result processor!");`
- **Rewrite**: `info!("Intent execution service shut down");`

### Line 203 (in `result_processor()`)
- **Current**: `error!("ScheduledCommitsProcessorImpl lags behind Intent execution! skipped: {}", skipped);`
- **Rewrite**: `error!(skipped_count = skipped, "Lagging behind intent execution");`

### Line 215 (in `result_processor()`)
- **Current**: `info!("OffChain triggered BaseIntent executed: {}", intent_id);`
- **Rewrite**: `info!(intent_id, trigger_type = ?trigger_type, "OffChain intent executed");`

### Line 230-233 (in `result_processor()`)
- **Current**: `error!("CRITICAL! Failed to find IntentMeta for id: {}!", intent_id);`
- **Rewrite**: `error!(intent_id, "Failed to find intent metadata");`

### Line 262-265 (in `process_intent_result()`)
- **Current**: `debug!("Signaled sent commit with internal signature: {:?}", signature);`
- **Rewrite**: `debug!(signature = %signature, "Sent commit signaled");`

### Line 267 (in `process_intent_result()`)
- **Current**: `error!("Failed to signal sent commit via transaction: {}", err);`
- **Rewrite**: `error!(error = ?err, "Failed to signal sent commit");`

### Line 288-291 (in `build_sent_commit()`)
- **Note**: This is a sync function. Per the deterministic pattern, sync functions do NOT get spans.
- **Action**: Leave as-is (no change)

### Line 305 (in `build_sent_commit()`)
- **Note**: In closure within sync function
- **Action**: Leave as-is (no change)

---

## Validation Steps

### 1. Syntax Check
```bash
cd magicblock-accounts
cargo check
```

### 2. Lint
```bash
make lint
```

### 3. Tests
```bash
cargo nextest run -p magicblock-accounts
```

### 4. Log Inspection
```bash
RUST_LOG=debug cargo test --package magicblock-accounts --lib 2>&1 | grep -E "ScheduledCommitsProcessor|process_undelegation|result_processor"
```

**Verify**:
- Constant message text (no variables in message)
- Fields appear in span context
- No `[...]` prefixes in messages
- No duplicate information (in both message and field)

---

## Summary

**magicblock-accounts** is a minimal implementation with clear responsibilities:

| Aspect | Details |
|--------|---------|
| **Files to modify** | 1 (scheduled_commits_processor.rs) |
| **Async functions to instrument** | 4 (process_undelegation_requests, result_processor, process_intent_result, process) |
| **Log statements to rewrite** | 10 |
| **Complexity** | Low (straightforward pattern application) |
| **Key challenge** | Field recording in result_processor (use tracing::Span::current().record()) |

The crate follows a clean separation: one trait definition, one implementation. All instrumentation happens in `scheduled_commits_processor.rs`. The pattern is deterministic and requires no special decisions beyond standard field naming and proper span wrapping.
