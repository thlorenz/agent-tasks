# Magicblock-Chainlink Structured Logging Implementation Plan

Based on the deterministic pattern from `02_plan-general.md`, this document provides a concrete, step-by-step implementation guide specific to the `magicblock-chainlink` crate.

---

## Overview

The `magicblock-chainlink` crate contains multiple async components handling RPC subscriptions, account cloning, and delegation management. Total files to update: **19 files** with logging statements.

### File Categories

1. **Remote Account Provider**: Core subscription and update logic
   - `chain_laser_actor.rs` - gRPC laser streaming (Helius/Triton)
   - `chain_laser_client.rs` - Client interface for laser actor
   - `chain_pubsub_actor.rs` - WebSocket RPC subscriptions
   - `chain_pubsub_client.rs` - Client interface for pubsub
   - `subscription_reconciler.rs` - Subscription state reconciliation
   - Other provider modules

2. **Submux** (Subscription Multiplexer): Multi-client coordination
   - `mod.rs` - Main submux logic
   - `subscription_task.rs` - Individual subscription tasks

3. **Chainlink** (Fetch/Clone/Delegation): Account fetching and cloning
   - `fetch_cloner/mod.rs` - Main fetch cloner
   - `fetch_cloner/pipeline.rs` - Account fetch pipeline

---

## Critical Files (Process First)

### File 1: `src/remote_account_provider/subscription_reconciler.rs`

**Priority**: 1 (No dependencies, simple, foundational)

**Async functions with logging**:
- `unsubscribe_and_notify_removal()` (async)

**Current log statements**:

```rust
// Line 23 - warn on send failure
warn!("Failed to send removal update for {pubkey}: {err:?}");

// Line 28 - warn on unsubscribe failure  
warn!("Failed to unsubscribe from pubsub for {pubkey}: {err:?}");
```

**Implementation**:

1. Add `#[instrument(skip(pubsub_client, removed_account_tx), fields(pubkey = %pubkey))]`
2. Rewrite:
   - Line 23: `warn!(error = ?err, "Failed to send removal update");`
   - Line 28: `warn!(error = ?err, "Failed to unsubscribe");`

---

### File 2: `src/remote_account_provider/chain_laser_actor.rs`

**Priority**: 2 (Used by chain_laser_client)

**Key async functions**:
1. `run()` - Main event loop
2. `handle_account_update()` - Process account updates
3. `handle_program_update()` - Process program updates
4. `signal_connection_issue()` - Signal disconnect
5. `process_subscription_update()` - Core update processing

**Current patterns** (all have `[client_id=...]` prefix):

```rust
// Line 202-204
info!("[client_id={}] Shutting down ChainLaserActor", self.client_id);

// Line 240-243
debug!("[client_id={}] Account subscription stream ended...", self.client_id);

// Line 607-610
error!("[client_id={}] Error in account update stream for accounts with idx {}: {}...", self.client_id, idx, err);

// Line 636-639
error!("[client_id={}] Error in program subscription stream: {}...", self.client_id, err);

// Line 678
debug!("[client_id={client_id}] Signaling connection issue");

// Line 719-720
error!("[client_id={}] Failed to parse pubkey in subscription update", self.client_id);

// Line 739-743
trace!("[client_id={}] Received {source} subscription update for {}: slot {}",
       self.client_id, pubkey, slot);

// Line 753-756
error!("[client_id={}] Failed to parse owner pubkey in subscription update for account {}",
       self.client_id, pubkey);

// Line 801-804
error!("[client_id={}] Failed to send subscription update for account {}", self.client_id, pubkey);
```

**Implementation**:

1. Add `#[instrument(skip(self))]` to `run()`, `handle_account_update()`, `handle_program_update()`
2. Add `#[instrument(skip(...))]` to `signal_connection_issue()` with field `client_id`
3. Add `#[instrument(skip(self, update))]` to `process_subscription_update()` with fields:
   - `client_id = %self.client_id`
   - `pubkey` and `slot` as Empty initially, recorded when extracted
   - `source = %source`

**Specific rewrites**:

```rust
// Line 202 (in shutdown)
info!("Shutting down laser actor");

// Line 240
debug!("Account subscription stream ended");

// Line 607-610
error!(stream_index = idx, error = ?err, "Error in account update stream");

// Line 636-639
error!(error = ?err, "Error in program subscription stream");

// Line 678
debug!("Signaling connection issue");

// Line 719
error!("Failed to parse pubkey");

// Line 739 (update: use tracing::Span::current().record() for extracted fields)
trace!("Received subscription update");

// Line 753
error!(pubkey = %pubkey, "Failed to parse owner pubkey");

// Line 801
error!(pubkey = %pubkey, "Failed to send subscription update");
```

---

### File 3: `src/remote_account_provider/chain_laser_client.rs`

**Priority**: 3 (Thin wrapper, depends on actor)

**Async functions with logging**:
- `send_msg()` - Internal async helper
- Methods in `ReconnectableClient` trait impl

**Current log statements**:

```rust
// Line 160 (in inspect_err)
warn!("RecvError occurred while awaiting reconnect response: {err:?}.");
```

**Implementation**:

1. Add `#[instrument(skip(self))]` to `send_msg()`
2. Rewrite line 160:
   - `warn!(error = ?err, "RecvError while awaiting reconnect response");`

---

### File 4: `src/remote_account_provider/chain_pubsub_actor.rs`

**Priority**: 4 (Heavy logging, used by submux)

**Key async functions**:
1. `new()` / `new_from_url()` - Constructors
2. `shutdown()` - Shutdown handler (async)
3. `handle_msg()` - Main message dispatcher (async, large)
4. `add_sub()` - Add account subscription (async)
5. `add_program_sub()` - Add program subscription (async)
6. `try_reconnect()` - Reconnection handler (async)

**Current patterns** (line 124, 256, 284, 305, 369, 374, 407, 431, 438, 447, 450):

```rust
// Line 124 - shutdown
info!("[client_id={client_id}] Shutting down ChainPubsubActor");

// Line 256, 305 - warn disconnected subscribe
warn!("[client_id={client_id}] Ignoring subscribe request for {pubkey}...");

// Line 284 - trace disconnected unsubscribe
trace!("[client_id={client_id}] Ignoring unsubscribe request for {pubkey}...");

// Line 369 - trace subscription exists
trace!("[client_id={client_id}] Subscription for {pubkey} already exists...");

// Line 374 - trace adding subscription
trace!("[client_id={client_id}] Adding subscription for {pubkey} with commitment...");

// Line 407 - error on subscribe failure
error!("[client_id={client_id}] Failed to subscribe to account {pubkey} {err:?}");

// Line 431 - trace cancelled
trace!("[client_id={client_id}] Subscription for {pubkey} was cancelled");

// Line 438 - trace received update
trace!("[client_id={client_id}] Received update for {pubkey}: {rpc_response:?}");

// Line 447 - error on send failure
error!("[client_id={client_id}] Failed to send {pubkey} subscription update: {err:?}");

// Line 450 - debug ended
debug!("[client_id={client_id}] Subscription for {pubkey} ended (EOF)...");
```

**Implementation**:

1. Add `#[instrument]` to all async functions:
   - `shutdown()`: `#[instrument(skip(...))]` with field `client_id`
   - `handle_msg()`: `#[instrument(skip(...))]` with field `client_id`
   - `add_sub()`: `#[instrument(skip(subs, program_subs, pubsub_connection, subscription_updates_sender, abort_sender))]` with fields `client_id, pubkey, commitment = ?commitment_config`
   - `add_program_sub()`: Similar to `add_sub()` but `program_id = %pubkey`
   - `try_reconnect()`: `#[instrument(skip(...))]` with field `client_id`

2. Rewrite all log statements removing message parameters into fields

---

### File 5: `src/submux/subscription_task.rs`

**Priority**: 5 (Used by submux/mod.rs)

**Key async function**:
- `process()` - Main task executor (async, spawns work)

**Current log statements** (lines 197-199):

```rust
// In spawned task, around line 197
warn!(
    "Some clients failed to {}: {}",
    op_name.to_lowercase(),
    errors.join(", ")
);

// Timeout errors in async closures
"Subscribe timed out after {:?} for client {}"
```

**Implementation**:

1. Add `#[instrument(skip(clients))]` to `process()` with fields:
   - Derive from `self`: operation name, required confirmations, total clients
   
2. Rewrite warn statement:
   ```rust
   warn!(
       operation = %op_name,
       total_clients,
       required_confirmations,
       error_count = errors.len(),
       "Some clients failed"
   );
   ```

3. Timeout error messages become:
   ```rust
   error!(
       timeout_ms = SUBSCRIBE_TIMEOUT.as_millis() as u64,
       client_index = i,
       "Subscribe timed out"
   );
   ```

---

### File 6: `src/submux/mod.rs`

**Priority**: 6 (Depends on subscription_task)

**Key async functions**:
- `try_reconnect()` method with debug logging
- Functions that spawn tasks

**Current log statements** (lines 253-269, 298-301):

```rust
// Line 253-269
debug!(
    "Failed to reconnect for client {}: {err:?}"
);

// Line 298-301
warn!("Program subscription already exists for {}", program_id);
```

**Implementation**:

1. Add `#[instrument]` where appropriate to async functions
2. Rewrite:
   - Line 253: `debug!(client_index = i, error = ?err, "Failed to reconnect");`
   - Line 298: `warn!(program_id = %program_id, "Program subscription already exists");`

---

### File 7: `src/chainlink/fetch_cloner/pipeline.rs`

**Priority**: 7 (Mostly sync, limited logging)

**Note**: Most functions are NOT async, so minimal #[instrument] needed.

**Current log statements** (lines 150, 172-173):

```rust
// Line 150 - sync function
error!("We should not be fetching accounts that are already in bank: {pubkey}:{slot}");

// Line 172-173 - sync function
warn!(
    "Not cloning native loader program account: {pubkey} (should have been blacklisted)",
);
```

**Implementation**:

1. No async functions to instrument
2. Just improve message format (no prefixes):
   - Line 150: `error!(pubkey = %pubkey, slot, "Unexpected: account already in bank");`
   - Line 172: `warn!(pubkey = %pubkey, "Not cloning: native loader program (should be blacklisted)");`

---

### File 8: `src/chainlink/fetch_cloner/mod.rs`

**Priority**: 8 (Depends on pipeline)

**Key async functions**:
- `start_subscription_listener()` - Spawned listener
- Helper functions with debug logging

**Current log statements**:

```rust
// Line ~448
debug!("Fetching undelegating account {}. delegated={}, undelegating={}", pubkey, ...);
```

**Implementation**:

1. Add `#[instrument]` to async functions
2. Rewrite: `debug!(pubkey = %pubkey, delegated, undelegating, "Fetching undelegating account");`

---

### File 9: `src/chainlink/mod.rs`

**Priority**: 9 (Main chainlink logic)

**Current log statements**:

```rust
// Line ~
debug!("Undelegation requested for account: {pubkey}");
```

**Implementation**:

1. Add `#[instrument]` to async functions
2. Extract pubkey into field: `debug!(pubkey = %pubkey, "Undelegation requested");`

---

### File 10: `src/remote_account_provider/mod.rs`

**Priority**: 10 (Main provider, depends on actors)

Multiple async functions with various logging. Follow pattern:
1. Add `#[instrument]` to each async function with logging
2. Extract all parameters from message text to fields
3. Use consistent field names from standard table (below)

---

## Standard Field Naming for Magicblock-Chainlink

| Field | Type | Format | Use Case |
|-------|------|--------|----------|
| `client_id` | String | `%` | RPC/laser client identifier |
| `pubkey` | Pubkey | `%` | Account address |
| `program_id` | Pubkey | `%` | Program address |
| `slot` | u64 | bare | Block slot |
| `source` | enum | `%` | Account vs Program |
| `stream_index` | usize | bare | Stream map index |
| `error` | Error | `?` | Error details |
| `operation` | &str | bare | Operation type |
| `required_confirmations` | usize | bare | Client confirmations needed |
| `total_clients` | usize | bare | Total clients available |
| `commitment` | CommitmentConfig | `?` | RPC commitment level |
| `timeout_ms` | u64 | bare | Timeout milliseconds |
| `error_count` | usize | bare | Number of errors |
| `delegated` | bool | bare | Is delegated |
| `undelegating` | bool | bare | Is undelegating |

---

## Before/After Examples

### Example 1: `chain_laser_actor.rs` - `process_subscription_update()`

**Before**:
```rust
async fn process_subscription_update(
    &mut self,
    update: SubscribeUpdate,
    source: AccountUpdateSource,
) {
    // ... extract pubkey ...
    if let Ok(pubkey) = Pubkey::try_from(account.pubkey) {
        // OK
    } else {
        error!(
            "[client_id={}] Failed to parse pubkey in subscription update",
            self.client_id
        );
        return;
    }
    
    trace!(
        "[client_id={}] Received {source} subscription update for {}: slot {}",
        self.client_id, pubkey, slot
    );
}
```

**After**:
```rust
#[instrument(
    skip(self, update),
    fields(
        client_id = %self.client_id,
        pubkey = tracing::field::Empty,
        slot = tracing::field::Empty,
        source = %source,
    )
)]
async fn process_subscription_update(
    &mut self,
    update: SubscribeUpdate,
    source: AccountUpdateSource,
) {
    // ... extract pubkey ...
    if let Ok(pubkey) = Pubkey::try_from(account.pubkey) {
        tracing::Span::current()
            .record("pubkey", tracing::field::display(pubkey));
    } else {
        error!("Failed to parse pubkey");
        return;
    }
    
    tracing::Span::current()
        .record("slot", slot);
    
    trace!("Received subscription update");
}
```

### Example 2: `chain_pubsub_actor.rs` - `add_sub()`

**Before**:
```rust
async fn add_sub(
    pubkey: Pubkey,
    // ... params ...
    client_id: &str,
) {
    if subs.lock().unwrap().contains_key(&pubkey) {
        trace!("[client_id={client_id}] Subscription for {pubkey} already exists");
        return;
    }
    
    match pubsub_connection.account_subscribe(&pubkey, config).await {
        Ok(_) => { /* ... */ }
        Err(err) => {
            error!("[client_id={client_id}] Failed to subscribe to account {pubkey} {err:?}");
        }
    }
}
```

**After**:
```rust
#[instrument(
    skip(subs, program_subs, pubsub_connection, subscription_updates_sender, abort_sender),
    fields(client_id = client_id, pubkey = %pubkey, commitment = ?commitment_config)
)]
async fn add_sub(
    pubkey: Pubkey,
    // ... params ...
    client_id: &str,
) {
    if subs.lock().unwrap().contains_key(&pubkey) {
        trace!("Subscription already exists");
        return;
    }
    
    match pubsub_connection.account_subscribe(&pubkey, config).await {
        Ok(_) => { /* ... */ }
        Err(err) => {
            error!(error = ?err, "Failed to subscribe");
        }
    }
}
```

### Example 3: `subscription_task.rs` - `process()`

**Before**:
```rust
pub async fn process<T>(self, clients: Vec<Arc<T>>) -> RemoteAccountProviderResult<()> {
    // ...
    // In spawned task:
    warn!(
        "Some clients failed to {}: {}",
        op_name.to_lowercase(),
        errors.join(", ")
    );
}
```

**After**:
```rust
#[instrument(skip(clients))]
pub async fn process<T>(self, clients: Vec<Arc<T>>) -> RemoteAccountProviderResult<()> {
    let op_name = self.op_name();
    let total_clients = clients.len();
    let required_confirmations = /* ... */;
    
    // ...
    // In spawned task:
    warn!(
        operation = %op_name,
        total_clients,
        required_confirmations,
        error_count = errors.len(),
        "Some clients failed"
    );
}
```

---

## Implementation Order

Process files in this exact order:

1. `subscription_reconciler.rs` - No deps, simple start
2. `chain_laser_actor.rs` - Depended on by client
3. `chain_laser_client.rs` - Thin wrapper
4. `chain_pubsub_actor.rs` - Heavy logging, foundational
5. `subscription_task.rs` - Used by submux
6. `submux/mod.rs` - Depends on task
7. `fetch_cloner/pipeline.rs` - Mostly sync
8. `fetch_cloner/mod.rs` - Depends on pipeline
9. `chainlink/mod.rs` - Top level
10. `remote_account_provider/mod.rs` - Main provider

Then remaining files with minimal logging.

---

## Validation Steps

### 1. Syntax Check
```bash
cd magicblock-chainlink
cargo check
```

### 2. Lint
```bash
make lint
```

### 3. Tests
```bash
cargo nextest run -p magicblock-chainlink
```

### 4. Log Inspection
```bash
RUST_LOG=debug cargo test --package magicblock-chainlink --lib 2>&1 | head -100
```

**Verify**:
- Constant message text (no variables in message)
- Fields in span context (before the colon)
- No `[client_id=...]` prefixes in messages
- No duplicate info (in both message and field)

---

## Summary

This plan provides **19 files to update**, **10 priority groups**, **concrete before/after examples**, and **standard field naming**. Follow the implementation order, apply the pattern to each file, and validate with the provided steps.

The result: Clean, queryable, aggregatable structured logging across magicblock-chainlink.
