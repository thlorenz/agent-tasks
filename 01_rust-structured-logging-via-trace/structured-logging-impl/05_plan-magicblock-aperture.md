# Magicblock-Aperture Structured Logging Implementation Plan

Based on the deterministic pattern from `02_plan-general.md`, this document provides a concrete, step-by-step implementation guide specific to the `magicblock-aperture` crate.

---

## Overview

The `magicblock-aperture` crate is a JSON-RPC server handling both HTTP and WebSocket connections for Solana validator interaction. It processes account updates, transactions, and block events from the validator through Geyser plugin integration.

**Total files to update**: **8 files** with meaningful async logging.

### Architecture Overview

1. **Core Entry Points**: `lib.rs` - Server lifecycle management
2. **Event Processing**: `processor.rs` - Background event handling and distribution
3. **HTTP Server**: `server/http/mod.rs` - TCP listener and connection handling
4. **WebSocket Server**: `server/websocket/mod.rs` - WebSocket upgrade and connection lifecycle
5. **WebSocket Connection**: `server/websocket/connection.rs` - Per-connection event loop
6. **HTTP Request Handlers**: `requests/http/` - RPC method implementations (10+ files, minimal logging)
7. **Transaction Handling**: `requests/http/send_transaction.rs` - Transaction submission
8. **Account Fetching**: `requests/http/mod.rs` - Account resolution helpers

---

## File Categories

### Category 1: Server Lifecycle (2 files)
- `lib.rs` - Main entrypoint, server startup/shutdown
- `processor.rs` - Event processor startup and main loop

### Category 2: Network Servers (2 files)
- `server/http/mod.rs` - HTTP server
- `server/websocket/mod.rs` - WebSocket server

### Category 3: Connection Handling (1 file)
- `server/websocket/connection.rs` - Per-connection event loop

### Category 4: Request Processing (3 files)
- `requests/http/send_transaction.rs` - Transaction submission
- `requests/http/mod.rs` - Helper functions for account/transaction handling
- `requests/http/get_account_info.rs` - Account info retrieval

---

## Standard Field Naming for Magicblock-Aperture

| Field | Type | Format | Use Case |
|-------|------|--------|----------|
| `processor_id` | usize | bare | Event processor instance ID |
| `event_type` | &str | bare | Event type (slot, block, account, transaction) |
| `error` | Error | `?` | Error details |
| `pubkey` | Pubkey | `%` | Account public key |
| `signature` | Signature | `%` | Transaction signature |
| `slot` | u64 | bare | Block slot number |
| `duration_ms` | u64 | bare | Processing duration |

---

## Critical Files (Process First)

### File 1: `src/lib.rs`

**Priority**: 1 (Entry point, no dependencies)

**Async functions with logging**:
- `run()` - Main server loop

**Current log statements**:

```rust
// Line 58 - startup
info!("Running JSON-RPC server");

// Line 63 - shutdown
info!("JSON-RPC server has shutdown");
```

**Implementation**:

1. Add `#[instrument]` to `run()` with no fields (top-level server)
2. Rewrite:
   - Line 58: `info!("JSON-RPC server running");`
   - Line 63: `info!("JSON-RPC server shutdown");`

---

### File 2: `src/processor.rs`

**Priority**: 2 (Heavy logging, foundational for event distribution)

**Key async function**:
1. `run()` - Main event processing loop (async)

**Current log statements**:

```rust
// Line 108 - startup
info!("event processor {id} is running");

// Lines 119, 123, 137, 151 - error handling
warn!("Geyser slot update error: {e}");
warn!("Geyser block update error: {e}");
warn!("Geyser account update error: {e}");
warn!("Geyser transaction update error: {e}");

// Line 167 - shutdown
info!("event processor {id} has terminated");
```

**Implementation**:

1. Add `#[instrument(fields(processor_id = id))]` to `run()`
2. Rewrite all log statements:
   - Line 108: `info!("Event processor started");`
   - Lines 119, 123: `warn!(error = ?e, "Geyser slot/block update failed");`
   - Line 137: `warn!(error = ?e, "Geyser account update failed");`
   - Line 151: `warn!(error = ?e, "Geyser transaction update failed");`
   - Line 167: `info!("Event processor terminated");`

**Detailed before/after**:

**Before**:
```rust
async fn run(self, id: usize, cancel: CancellationToken) {
    info!("event processor {id} is running");
    loop {
        tokio::select! {
            Ok(latest) = self.block_update_rx.recv_async() => {
                let _ = self.geyser.notify_slot(latest.meta.slot).inspect_err(|e| {
                    warn!("Geyser slot update error: {e}");
                });
                let _ = self.geyser.notify_block(&latest).inspect_err(|e| {
                    warn!("Geyser block update error: {e}");
                });
            }
            Ok(state) = self.account_update_rx.recv_async() => {
                let _ = self.geyser.notify_account(&state).inspect_err(|e| {
                    warn!("Geyser account update error: {e}");
                });
            }
            Ok(status) = self.transaction_status_rx.recv_async() => {
                let _ = self.geyser.notify_transaction(&status).inspect_err(|e| {
                    warn!("Geyser transaction update error: {e}");
                });
            }
            _ = cancel.cancelled() => break,
        }
    }
    info!("event processor {id} has terminated");
}
```

**After**:
```rust
#[instrument(fields(processor_id = id))]
async fn run(self, id: usize, cancel: CancellationToken) {
    info!("Event processor started");
    loop {
        tokio::select! {
            Ok(latest) = self.block_update_rx.recv_async() => {
                let _ = self.geyser.notify_slot(latest.meta.slot).inspect_err(|e| {
                    warn!(error = ?e, "Geyser slot update failed");
                });
                let _ = self.geyser.notify_block(&latest).inspect_err(|e| {
                    warn!(error = ?e, "Geyser block update failed");
                });
            }
            Ok(state) = self.account_update_rx.recv_async() => {
                let _ = self.geyser.notify_account(&state).inspect_err(|e| {
                    warn!(error = ?e, "Geyser account update failed");
                });
            }
            Ok(status) = self.transaction_status_rx.recv_async() => {
                let _ = self.geyser.notify_transaction(&status).inspect_err(|e| {
                    warn!(error = ?e, "Geyser transaction update failed");
                });
            }
            _ = cancel.cancelled() => break,
        }
    }
    info!("Event processor terminated");
}
```

---

### File 3: `src/server/http/mod.rs`

**Priority**: 3 (HTTP server, depends on nothing)

**Key async functions**:
1. `run()` - Main server loop (async)

**Current log statements**:

```rust
// Line 79 - shutdown signal received
info!("HTTP server shutdown signal has been received");

// Line 91 - full shutdown
info!("HTTP server has shutdown");
```

**Implementation**:

1. Add `#[instrument]` to `run()` with no fields (server-level)
2. Rewrite:
   - Line 79: `info!("HTTP server shutdown signal received");`
   - Line 91: `info!("HTTP server shutdown");`

---

### File 4: `src/server/websocket/mod.rs`

**Priority**: 4 (WebSocket server, depends on connection handler)

**Key async functions**:
1. `run()` - Main server loop (async)

**Current log statements**:

```rust
// Line 98 - shutdown signal
info!("Websocket server shutdown signal has been received");

// Line 107 - full shutdown
info!("Websocket server has shutdown");

// Line 128 - connection error (in spawned task closure, not instrumented)
warn!("websocket connection terminated with error: {error}");

// Line 149 - upgrade error (in spawned task closure)
warn!("failed http upgrade to ws connection");
```

**Implementation**:

1. Add `#[instrument]` to `run()` with no fields
2. Rewrite log statements:
   - Line 98: `info!("WebSocket server shutdown signal received");`
   - Line 107: `info!("WebSocket server shutdown");`
   - Line 128: `warn!(error = ?error, "WebSocket connection terminated");`
   - Line 149: `warn!("HTTP upgrade to WebSocket failed");`

**Note**: Lines 128 and 149 are in spawned closures without access to span context. This is acceptable - the logs will still be visible without span inheritance. Consider if context is needed (e.g., connection ID).

---

### File 5: `src/server/websocket/connection.rs`

**Priority**: 5 (Per-connection handler, depends on server)

**Key async functions**:
1. `run()` - Main connection event loop (async)
2. `send()` - WebSocket frame sending (async)

**Current log statements**:

```rust
// Line 209 - send failure (in inspect_err)
debug!("failed to send websocket frame to the client: {e}")
```

**Implementation**:

1. Add `#[instrument]` to `run()` - no parameters, connection is self-contained
2. Rewrite line 209:
   - `debug!(error = ?e, "Failed to send WebSocket frame");`

**Detailed before/after**:

**Before**:
```rust
pub(super) async fn run(mut self) {
    const MAX_INACTIVE_INTERVAL: Duration = Duration::from_secs(60);
    // ... setup ...
    loop {
        tokio::select! {
            // ... event handling ...
            Some(update) = self.updates_rx.recv() => {
                if self.send(update.as_ref()).await.is_err() {
                    break;
                }
            }
            // ...
        }
    }
}

async fn send(
    &mut self,
    payload: impl Into<Payload<'_>>,
) -> Result<(), WebSocketError> {
    let frame = Frame::text(payload.into());
    self.ws.write_frame(frame).await.inspect_err(|e| {
        debug!("failed to send websocket frame to the client: {e}")
    })
}
```

**After**:
```rust
#[instrument(skip(self))]
pub(super) async fn run(mut self) {
    const MAX_INACTIVE_INTERVAL: Duration = Duration::from_secs(60);
    // ... setup ...
    loop {
        tokio::select! {
            // ... event handling ...
            Some(update) = self.updates_rx.recv() => {
                if self.send(update.as_ref()).await.is_err() {
                    break;
                }
            }
            // ...
        }
    }
}

async fn send(
    &mut self,
    payload: impl Into<Payload<'_>>,
) -> Result<(), WebSocketError> {
    let frame = Frame::text(payload.into());
    self.ws.write_frame(frame).await.inspect_err(|e| {
        debug!(error = ?e, "Failed to send WebSocket frame");
    })
}
```

---

### File 6: `src/requests/http/send_transaction.rs`

**Priority**: 6 (Transaction submission, uses helpers)

**Key async function**:
- `send_transaction()` - Transaction submission handler (async)

**Current log statements**:

```rust
// Line 32 - transaction prep failure (in inspect_err)
debug!("Failed to prepare transaction: {err}");

// Line 50 - scheduling
trace!("Scheduling transaction {signature}");

// Line 52 - executing
trace!("Executing transaction {signature}");
```

**Implementation**:

1. Add `#[instrument(skip(self, request))]` to `send_transaction()` with field:
   - `signature = tracing::field::Empty` (recorded after extraction)
2. Rewrite:
   - Line 32: `debug!(error = ?err, "Failed to prepare transaction");`
   - Lines 50, 52: Rewrite to record signature in span, then log constant message

**Detailed before/after**:

**Before**:
```rust
pub(crate) async fn send_transaction(
    &self,
    request: &mut JsonRequest,
) -> HandlerResult {
    let transaction = self
        .prepare_transaction(&transaction_str, encoding, true, false)
        .inspect_err(|err| {
            debug!("Failed to prepare transaction: {err}")
        })?;
    let signature = *transaction.signature();

    if config.skip_preflight {
        TRANSACTION_SKIP_PREFLIGHT.inc();
        self.transactions_scheduler.schedule(transaction).await?;
        trace!("Scheduling transaction {signature}");
    } else {
        trace!("Executing transaction {signature}");
        self.transactions_scheduler.execute(transaction).await?;
    }

    let signature = SerdeSignature(signature);
    Ok(ResponsePayload::encode_no_context(&request.id, signature))
}
```

**After**:
```rust
#[instrument(skip(self, request), fields(signature = tracing::field::Empty))]
pub(crate) async fn send_transaction(
    &self,
    request: &mut JsonRequest,
) -> HandlerResult {
    let transaction = self
        .prepare_transaction(&transaction_str, encoding, true, false)
        .inspect_err(|err| {
            debug!(error = ?err, "Failed to prepare transaction");
        })?;
    let signature = *transaction.signature();

    tracing::Span::current()
        .record("signature", tracing::field::display(signature));

    if config.skip_preflight {
        TRANSACTION_SKIP_PREFLIGHT.inc();
        self.transactions_scheduler.schedule(transaction).await?;
        trace!("Transaction scheduled");
    } else {
        trace!("Transaction executing");
        self.transactions_scheduler.execute(transaction).await?;
    }

    let signature = SerdeSignature(signature);
    Ok(ResponsePayload::encode_no_context(&request.id, signature))
}
```

---

### File 7: `src/requests/http/mod.rs`

**Priority**: 7 (Helper functions for account resolution)

**Key async functions**:
1. `read_account_with_ensure()` - Single account fetch
2. `read_accounts_with_ensure()` - Multiple account fetch
3. `ensure_transaction_accounts()` - Transaction account resolution

**Current log statements**:

```rust
// Line 128 - account fetch failure
debug!("Failed to ensure account {pubkey}: {e}");

// Line 139 - trace accounts being ensured
trace!("Ensuring accounts {pubkeys:?}");

// Line 155 - multi-account fetch failure
warn!("Failed to ensure accounts: {e}");

// Line 225 - transaction account resolution issues
debug!(
    "Transaction {} account resolution encountered issues:\n{res}",
    transaction.signature());

// Line 235 - transaction account resolution failure
warn!("Failed to ensure transaction accounts: {:?}", err);
```

**Implementation**:

1. Add `#[instrument(skip(self))]` to each async function with appropriate fields
2. Rewrite:
   - Line 128: `debug!(pubkey = %pubkey, error = ?e, "Failed to ensure account");`
   - Line 139: `trace!(pubkey_count = pubkeys.len(), "Ensuring accounts");`
   - Line 155: `warn!(error = ?e, "Failed to ensure accounts");`
   - Line 225: `debug!(signature = %transaction.signature(), "Account resolution issues");`
   - Line 235: `warn!(error = ?err, "Failed to ensure transaction accounts");`

**Detailed before/after** (showing read_account_with_ensure):

**Before**:
```rust
async fn read_account_with_ensure(
    &self,
    pubkey: &Pubkey,
) -> Option<AccountSharedData> {
    let _timer = ENSURE_ACCOUNTS_TIME
        .with_label_values(&["account"])
        .start_timer();
    let _ = self
        .chainlink
        .ensure_accounts(
            &[*pubkey],
            None,
            AccountFetchOrigin::GetAccount,
            None,
        )
        .await
        .inspect_err(|e| {
            debug!("Failed to ensure account {pubkey}: {e}");
        });
    self.accountsdb.get_account(pubkey)
}
```

**After**:
```rust
#[instrument(skip(self), fields(pubkey = %pubkey))]
async fn read_account_with_ensure(
    &self,
    pubkey: &Pubkey,
) -> Option<AccountSharedData> {
    let _timer = ENSURE_ACCOUNTS_TIME
        .with_label_values(&["account"])
        .start_timer();
    let _ = self
        .chainlink
        .ensure_accounts(
            &[*pubkey],
            None,
            AccountFetchOrigin::GetAccount,
            None,
        )
        .await
        .inspect_err(|e| {
            debug!(error = ?e, "Failed to ensure account");
        });
    self.accountsdb.get_account(pubkey)
}
```

---

### File 8: `src/requests/http/get_account_info.rs`

**Priority**: 8 (Simple handler, minimal changes)

**Key async function**:
- `get_account_info()` - Account info RPC handler

**Current log statements**:

```rust
// Line 27 - trace
debug!("get_account_info: '{}'", pubkey);
```

**Implementation**:

1. Add `#[instrument(skip(self, request))]` to `get_account_info()` with field:
   - `pubkey = %pubkey`
2. Rewrite line 27:
   - `debug!("Getting account info");`

---

## Implementation Order

Process files in this exact order:

1. `src/lib.rs` - Entry point, no dependencies
2. `src/processor.rs` - Event processor, independent
3. `src/server/http/mod.rs` - HTTP server, independent
4. `src/server/websocket/mod.rs` - WebSocket server, independent
5. `src/server/websocket/connection.rs` - Per-connection handler
6. `src/requests/http/send_transaction.rs` - Transaction handler
7. `src/requests/http/mod.rs` - Helper functions
8. `src/requests/http/get_account_info.rs` - Account handler

---

## Validation Steps

### 1. Syntax Check
```bash
cd magicblock-aperture
cargo check
```

### 2. Lint
```bash
make lint
```

### 3. Tests
```bash
cargo nextest run -p magicblock-aperture
```

### 4. Log Inspection
```bash
RUST_LOG=debug cargo test --package magicblock-aperture --lib 2>&1 | head -100
```

**Verify**:
- Constant message text (no variables in message)
- Fields in span context (before the colon)
- No format string placeholders in messages
- No duplicate info (in both message and field)

---

## Summary

This plan provides **8 files to update**, **specific before/after examples for 4 representative functions**, and **concrete logging patterns** for a JSON-RPC server crate.

Follow the implementation order, apply the pattern to each file systematically, and validate with the provided steps. The result: queryable, structured logging across magicblock-aperture with constant messages and extracted context fields.
