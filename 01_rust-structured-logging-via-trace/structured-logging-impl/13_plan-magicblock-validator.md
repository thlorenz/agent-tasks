# Magicblock-Validator Structured Logging Implementation Plan

Based on the deterministic pattern from `02_plan-general.md`, this document provides a concrete, step-by-step implementation guide specific to the `magicblock-validator` crate (the main entry point binary).

---

## Overview

The `magicblock-validator` crate is the main entry point/binary for the MagicBlock validator. It is minimal and focused, with only **8 log statements** across **2 files**. The crate serves to:

1. Initialize the async runtime and logger
2. Load configuration
3. Create and start the validator API
4. Wait for shutdown signals
5. Perform graceful shutdown

Total files to update: **2 files**

### File Breakdown

1. **`src/main.rs`** - Main entry point (6 log statements)
   - Async `run()` function - Primary async entrypoint
   - Helper `print_info()` - Conditional log printing (sync, no instrument needed)

2. **`src/shutdown.rs`** - Signal handling (2 log statements)
   - Async `Shutdown::wait()` - Signal waiter

---

## Critical Files (Process First)

### File 1: `src/shutdown.rs`

**Priority**: 1 (No dependencies, simple foundational)

**Async functions with logging**:
- `Shutdown::wait()` - Main signal handler (async, lines 9-27)

**Current log statements**:

```rust
// Line 14 - SIGTERM
info!("SIGTERM has been received, initiating graceful shutdown");

// Line 17 - SIGINT
info!("SIGINT signal received, initiating graceful shutdown");
```

**Analysis**:
- Two information logs about shutdown signals
- Both are action-oriented (initiating graceful shutdown)
- No parameters to extract
- Message text is constant and appropriate
- Signal type can be captured as a field

**Implementation**:

1. Add `#[instrument]` to `Shutdown::wait()`
2. Rewrite log statements:
   - Line 14: `info!(signal = "SIGTERM", "Initiating graceful shutdown");`
   - Line 17: `info!(signal = "SIGINT", "Initiating graceful shutdown");`

Rationale: Make the signal type a field so it's queryable separately from the message. This allows filtering/counting by signal type in observability platforms.

---

### File 2: `src/main.rs`

**Priority**: 2 (Depends on shutdown via imports)

**Async functions with logging**:
- `async fn run()` - Main validator runtime (lines 36-99)

**Current log statements**:

```rust
// Line 48 - startup info
info!("Starting validator with config:\n{:#?}", config);

// Line 61 - debug after API creation
debug!("Created API .. starting things up");

// Line 94 - error on shutdown failure
error!("Failed to gracefully shutdown: {}", err);

// Line 33 - main shutdown (NOT in async, in main())
info!("main runtime shutdown!");

// Lines 113, 48 via print_info() - conditional logs
info!("{}", msg);  // Called from print_info()
```

**Analysis**:
- `run()` is the main async function and needs `#[instrument]`
- Line 48: Config object with debug formatting - consider extracting key fields
- Line 61: Simple debug message
- Line 94: Error with context - needs error field
- Line 33: Outside async context, no `#[instrument]` needed
- Lines 113: Inside sync helper `print_info()`, no `#[instrument]` needed

**Implementation**:

1. Add `#[instrument]` to `async fn run()` with:
   - Skip parameters: None needed (no self, config is passed as value)
   - Fields: Consider adding `rpc_port`, `ws_port`, `validator_identity` as they're important startup context
   - Actually, simpler approach: Don't add custom fields in instrument, extract in messages

2. Rewrite log statements within `run()`:
   - Line 48: Change from debug format of entire config to structured fields
   - Line 61: Keep simple message
   - Line 94: Extract error to field

---

## Before/After Examples

### Example 1: `src/shutdown.rs` - `Shutdown::wait()`

**Before**:
```rust
pub async fn wait() -> std::io::Result<()> {
    let mut terminate_signal =
        signal::unix::signal(SignalKind::terminate())?;
    tokio::select! {
        _ = terminate_signal.recv() => {
            info!("SIGTERM has been received, initiating graceful shutdown");
        },
        _ = tokio::signal::ctrl_c() => {
            info!("SIGINT signal received, initiating graceful shutdown");
        },
    }

    Ok(())
}
```

**After**:
```rust
#[instrument]
pub async fn wait() -> std::io::Result<()> {
    let mut terminate_signal =
        signal::unix::signal(SignalKind::terminate())?;
    tokio::select! {
        _ = terminate_signal.recv() => {
            info!(signal = "SIGTERM", "Initiating graceful shutdown");
        },
        _ = tokio::signal::ctrl_c() => {
            info!(signal = "SIGINT", "Initiating graceful shutdown");
        },
    }

    Ok(())
}
```

**Key changes**:
- Added `#[instrument]` - automatically creates span for the async function
- Extracted signal type to field: `signal = "SIGTERM"` (string literal, no formatting needed)
- Unified message text: "Initiating graceful shutdown" (same for both signals)
- Message is constant and action-oriented

---

### Example 2: `src/main.rs` - `async fn run()`

**Before**:
```rust
async fn run() {
    init_logger();
    #[cfg(feature = "tokio-console")]
    console_subscriber::init();
    let args = std::env::args_os();
    let config = match ValidatorParams::try_new(args) {
        Ok(c) => c,
        Err(err) => {
            eprintln!("Failed to read validator config: {err}");
            std::process::exit(1);
        }
    };
    info!("Starting validator with config:\n{:#?}", config);
    // ... rest of function ...
    
    if let Err(err) = Shutdown::wait().await {
        error!("Failed to gracefully shutdown: {}", err);
    }

    api.prepare_ledger_for_shutdown();
    api.stop().await;
}
```

**After**:
```rust
#[instrument(skip(ledger_lock))]
async fn run() {
    init_logger();
    #[cfg(feature = "tokio-console")]
    console_subscriber::init();
    let args = std::env::args_os();
    let config = match ValidatorParams::try_new(args) {
        Ok(c) => c,
        Err(err) => {
            eprintln!("Failed to read validator config: {err}");
            std::process::exit(1);
        }
    };
    info!("Starting validator");
    debug!(rpc_port = config.aperture.listen.port(), ws_port = config.aperture.listen.port() + 1, "Validator configured");
    // ... rest of function ...
    
    if let Err(err) = Shutdown::wait().await {
        error!(error = ?err, "Failed to gracefully shutdown");
    }

    api.prepare_ledger_for_shutdown();
    api.stop().await;
}
```

**Key changes**:
- Added `#[instrument]` with `skip()` (no self, but if large params should skip)
- Line 48: Removed debug format of entire config object
  - Before: `info!("Starting validator with config:\n{:#?}", config);`
  - After: `info!("Starting validator");` (config is loaded, no need to dump it all)
  - Added `debug!()` with extracted fields: `rpc_port`, `ws_port`
- Line 94: Extracted error to field
  - Before: `error!("Failed to gracefully shutdown: {}", err);`
  - After: `error!(error = ?err, "Failed to gracefully shutdown");`
- Message is constant: "Failed to gracefully shutdown"

---

### Example 3: Optional - Enhanced Version with More Context

If we want to capture more startup context, we could enhance Example 2:

```rust
#[instrument]
async fn run() {
    // ... initialization ...
    
    let rpc_port = config.aperture.listen.port();
    let ws_port = rpc_port + WS_PORT_OFFSET;
    let rpc_host = config.aperture.listen.ip();
    let validator_identity = config.validator.keypair.pubkey();
    
    info!(rpc_port, ws_port, "Starting validator");
    
    // ... rest of function ...
}
```

This approach:
- Captures key startup parameters as fields
- Makes them queryable and filterable
- Doesn't pollute the message text
- All logs in this span inherit these fields

---

## Standard Field Naming for Magicblock-Validator

| Field | Type | Format | Use Case |
|-------|------|--------|----------|
| `signal` | &str | bare | OS signal type (SIGTERM, SIGINT) |
| `error` | Error | `?` | Error details |
| `rpc_port` | u16 | bare | RPC listening port |
| `ws_port` | u16 | bare | WebSocket listening port |
| `rpc_host` | IpAddr | `%` | RPC listening IP address |
| `validator_identity` | Pubkey | `%` | Validator public key |

---

## Implementation Order

Process files in this exact order:

1. **`src/shutdown.rs`** - No dependencies, simple, foundational
   - Add `#[instrument]` to `Shutdown::wait()`
   - Extract signal type to field
   - Unify message text

2. **`src/main.rs`** - Depends on shutdown
   - Add `#[instrument]` to `async fn run()`
   - Extract config values to fields
   - Convert debug format of config to debug-level fields
   - Extract error parameter to field
   - Note: `main()` function is NOT async, so no `#[instrument]` needed
   - Note: `print_info()` is sync helper, no `#[instrument]` needed

---

## Validation Steps

### 1. Syntax Check
```bash
cd magicblock-validator
cargo check
```

### 2. Lint
```bash
make lint
```

### 3. Tests (if any exist)
```bash
cargo nextest run -p magicblock-validator
```

### 4. Build
```bash
cargo build --release
```

### 5. Log Inspection (Running the validator)
```bash
RUST_LOG=debug ./target/release/magicblock-validator --config configs/ephem-devnet.toml 2>&1 | head -50
```

**Verify**:
- [ ] `run` span appears at startup
- [ ] `wait` span appears when awaiting shutdown
- [ ] Log messages are constant (no variables embedded)
- [ ] Fields appear before the colon with key=value format
- [ ] No `{:#?}` config dumps in log output
- [ ] Signal type appears as field when shutdown signal received
- [ ] Error field appears when shutdown fails

---

## Summary

The `magicblock-validator` crate is minimal with only **8 log statements** across **2 files**. This is a straightforward implementation:

1. **`src/shutdown.rs`**: Add `#[instrument]` to `Shutdown::wait()`, extract signal type to field
2. **`src/main.rs`**: Add `#[instrument]` to `async fn run()`, extract config/error values to fields

Both files follow the single deterministic pattern:
- `#[instrument]` on every async function with logging
- Constant message text
- Parameters extracted to fields
- Correct format specifiers

The result: Clean, structured, queryable logging at the validator's main entry point.
