# Magicblock-Metrics Structured Logging Implementation Plan

Based on the deterministic pattern from `02_plan-general.md`, this document provides a concrete, step-by-step implementation guide specific to the `magicblock-metrics` crate.

---

## Overview

The `magicblock-metrics` crate is a lightweight metrics collection and HTTP server module. It has minimal async code with logging, making it a straightforward implementation target.

**Total files to update**: **2 files** with async functions containing logging statements.

### Crate Structure

- **`src/service.rs`** - HTTP metrics server (async I/O, logging)
- **`src/metrics/mod.rs`** - Metric definitions (sync, no async logging)
- **`src/metrics/types.rs`** - Type definitions (no logging)
- **`src/lib.rs`** - Public API exports (no logging)

---

## Files Analysis

### File 1: `src/service.rs` (Primary Target)

**Priority**: 1 (Only async functions with logging)

**Async functions with logging**:
1. `run()` - Main HTTP server loop
2. `metrics_service_router()` - HTTP request handler

#### `run()` - Lines 48-93

**Current log statements**:

```rust
// Line 55
info!("Serving {}/metrics", &addr);

// Line 59
error!("Failed to bind to {}: {:?}", &addr, err);

// Line 78
error!("Error: {:?}", err);

// Line 82-85
error!(
    "Accepting connection from {} failed: {:?}",
    addr, err
)

// Line 91
info!("MetricsService shutdown!");
```

**Context**: This is the main metrics HTTP server event loop. It binds to a socket address, accepts connections, and serves metrics. The function is nested inside `spawn()` via `tokio::task::spawn(Self::run(...))`.

#### `metrics_service_router()` - Lines 96-140

**Current log statements**:

```rust
// Lines 100-116 (trace with multiple context fields)
trace!(
    "[{}] {:?} from {} ({})",
    req.method(),
    req.uri()
        .path_and_query()
        .map(|x| x.as_str())
        .unwrap_or_default(),
    req.headers()
        .get("host")
        .map(|h| h.to_str().unwrap_or_default())
        .unwrap_or_default(),
    req.headers()
        .get("user-agent")
        .map(|h| h.to_str().unwrap_or_default())
        .unwrap_or_default(),
);

// Line 122
warn!("could not encode custom metrics: {}", error);
```

**Context**: This is the HTTP request router. It handles `GET /metrics` requests and returns Prometheus metrics. It also logs trace info for all requests to help with debugging.

---

## Field Naming Convention

For magicblock-metrics, use these standard fields:

| Field | Type | Format | Use Case |
|-------|------|--------|----------|
| `addr` | SocketAddr | `%` | Server bind address |
| `error` | Error | `?` | Error details |
| `method` | Method | `%` | HTTP method (GET, POST, etc.) |
| `path` | String | bare | URI path (already extracted string) |
| `host` | String | bare | Request host header |
| `user_agent` | String | bare | Request user-agent header |

---

## Before/After Examples

### Example 1: `run()` - Full Function

**Before**:
```rust
async fn run(
    addr: SocketAddr,
    cancellation_token: CancellationToken,
) -> tokio::task::JoinHandle<()> {
    tokio::task::spawn(async move {
        let listener = match TcpListener::bind(&addr).await {
            Ok(listener) => {
                info!("Serving {}/metrics", &addr);
                listener
            }
            Err(err) => {
                error!("Failed to bind to {}: {:?}", &addr, err);
                return;
            }
        };

        loop {
            select!(
                _ = cancellation_token.cancelled() => {
                    break;
                }
                result = listener.accept() => {
                    match result {
                        Ok((stream, _)) => {
                            let io = TokioIo::new(stream);
                            tokio::task::spawn(async move {
                                if let Err(err) = http1::Builder::new()
                                    .serve_connection(io, service_fn(metrics_service_router))
                                    .await
                                {
                                    error!("Error: {:?}", err);
                                }
                            });
                        }
                        Err(err) => error!(
                            "Accepting connection from {} failed: {:?}",
                            addr, err
                        ),
                    };
                }
            );
        }

        info!("MetricsService shutdown!");
    })
}
```

**After**:
```rust
#[instrument(skip(cancellation_token), fields(addr = %addr))]
async fn run(
    addr: SocketAddr,
    cancellation_token: CancellationToken,
) -> tokio::task::JoinHandle<()> {
    tokio::task::spawn(async move {
        let listener = match TcpListener::bind(&addr).await {
            Ok(listener) => {
                info!("Metrics server started");
                listener
            }
            Err(err) => {
                error!(error = ?err, "Failed to bind");
                return;
            }
        };

        loop {
            select!(
                _ = cancellation_token.cancelled() => {
                    break;
                }
                result = listener.accept() => {
                    match result {
                        Ok((stream, _)) => {
                            let io = TokioIo::new(stream);
                            tokio::task::spawn(async move {
                                if let Err(err) = http1::Builder::new()
                                    .serve_connection(io, service_fn(metrics_service_router))
                                    .await
                                {
                                    error!(error = ?err, "Connection serving failed");
                                }
                            });
                        }
                        Err(err) => error!(
                            error = ?err,
                            "Failed to accept connection"
                        ),
                    };
                }
            );
        }

        info!("Metrics server shutdown");
    })
}
```

**Changes**:
1. Added `#[instrument(skip(cancellation_token), fields(addr = %addr))]` to function
2. Line 55: `info!("Metrics server started");` - constant message, addr in span
3. Line 59: `error!(error = ?err, "Failed to bind");` - extracted error, constant message
4. Line 78: `error!(error = ?err, "Connection serving failed");` - extracted error
5. Line 82-85: `error!(error = ?err, "Failed to accept connection");` - extracted error and addr
6. Line 91: `info!("Metrics server shutdown");` - constant message

---

### Example 2: `metrics_service_router()` - HTTP Handler

**Before**:
```rust
async fn metrics_service_router(
    req: Request<hyper::body::Incoming>,
) -> Result<Response<BoxBody<Bytes, hyper::Error>>, hyper::Error> {
    if enabled!(Level::TRACE) {
        trace!(
            "[{}] {:?} from {} ({})",
            req.method(),
            req.uri()
                .path_and_query()
                .map(|x| x.as_str())
                .unwrap_or_default(),
            req.headers()
                .get("host")
                .map(|h| h.to_str().unwrap_or_default())
                .unwrap_or_default(),
            req.headers()
                .get("user-agent")
                .map(|h| h.to_str().unwrap_or_default())
                .unwrap_or_default(),
        );
    }
    let result = match (req.method(), req.uri().path()) {
        (&Method::GET, "/metrics") => {
            let metrics = TextEncoder::new()
                .encode_to_string(&metrics::REGISTRY.gather())
                .unwrap_or_else(|error| {
                    warn!("could not encode custom metrics: {}", error);
                    String::new()
                });
            Ok(Response::new(full(metrics)))
        }
        _ => {
            let mut not_found = Response::new(empty());
            *not_found.status_mut() = StatusCode::NOT_FOUND;
            Ok(not_found)
        }
    };
    // ... rest of function ...
}
```

**After**:
```rust
#[instrument(skip(req), fields(
    method = %req.method(),
    path = req.uri().path(),
    host = tracing::field::Empty,
    user_agent = tracing::field::Empty,
))]
async fn metrics_service_router(
    req: Request<hyper::body::Incoming>,
) -> Result<Response<BoxBody<Bytes, hyper::Error>>, hyper::Error> {
    // Record optional headers
    if let Some(host) = req.headers()
        .get("host")
        .and_then(|h| h.to_str().ok())
    {
        tracing::Span::current().record("host", host);
    }
    if let Some(ua) = req.headers()
        .get("user-agent")
        .and_then(|h| h.to_str().ok())
    {
        tracing::Span::current().record("user_agent", ua);
    }

    let result = match (req.method(), req.uri().path()) {
        (&Method::GET, "/metrics") => {
            let metrics = TextEncoder::new()
                .encode_to_string(&metrics::REGISTRY.gather())
                .unwrap_or_else(|error| {
                    warn!(error = %error, "Failed to encode metrics");
                    String::new()
                });
            Ok(Response::new(full(metrics)))
        }
        _ => {
            let mut not_found = Response::new(empty());
            *not_found.status_mut() = StatusCode::NOT_FOUND;
            Ok(not_found)
        }
    };
    // ... rest of function ...
}
```

**Changes**:
1. Added `#[instrument]` with early-populated fields `method` and `path`
2. Used `tracing::field::Empty` for optional headers that may not be present
3. Recorded optional headers with `Span::current().record()` after extraction
4. Line 122: `warn!(error = %error, "Failed to encode metrics");` - extracted error, constant message

---

## Implementation Details

### Step 1: Add Imports

Add to `src/service.rs` if not already present:

```rust
use tracing::instrument;  // Add to existing use tracing::*;
```

The `#[instrument]` macro comes from the `tracing` crate which is already imported via `use tracing::*;`.

### Step 2: Modify `run()` function

File: `src/service.rs`, starting at line 48

1. Add `#[instrument(skip(cancellation_token), fields(addr = %addr))]` above `async fn run`
2. Update log messages:
   - Line 55: Change to `info!("Metrics server started");`
   - Line 59: Change to `error!(error = ?err, "Failed to bind");`
   - Line 78: Change to `error!(error = ?err, "Connection serving failed");`
   - Lines 82-85: Change to `error!(error = ?err, "Failed to accept connection"),`
   - Line 91: Change to `info!("Metrics server shutdown");`

### Step 3: Modify `metrics_service_router()` function

File: `src/service.rs`, starting at line 96

1. Add `#[instrument(skip(req), fields(method = %req.method(), path = req.uri().path(), host = tracing::field::Empty, user_agent = tracing::field::Empty))]` above `async fn metrics_service_router`
2. After the function start, add header extraction with `Span::current().record()` calls
3. Remove the `if enabled!(Level::TRACE)` block (not needed with `#[instrument]`)
4. Change line 122: `warn!(error = %error, "Failed to encode metrics");`

### Step 4: Verify No Manual Spans

Search for manual span creation:

```bash
cd magicblock-metrics
rg "info_span!|debug_span!|warn_span!|trace_span!"
```

**Expected result**: Zero matches (magicblock-metrics doesn't use manual spans)

---

## File Implementation Order

Since there are only 2 files with async functions containing logging:

1. **`src/service.rs`** - Primary implementation
   - Contains both `run()` and `metrics_service_router()`
   - All logging is in these two functions
   - No dependencies on other files in this crate

2. **Remaining files** (`metrics/mod.rs`, `metrics/types.rs`, `lib.rs`)
   - No async functions with logging
   - No changes needed

---

## Detailed Implementation Checklist

### Pre-Implementation
- [ ] Read and understand `02_plan-general.md`
- [ ] Review before/after examples in this document (Examples 1-2 above)
- [ ] Identify the two async functions in `src/service.rs`
- [ ] Understand field naming conventions (table above)

### During Implementation

#### `src/service.rs` - `run()` function

- [ ] Add `#[instrument(skip(cancellation_token), fields(addr = %addr))]` attribute
- [ ] Verify `addr` is captured in span (used via `self.addr` or parameter)
- [ ] Update line 55: Make message constant `"Metrics server started"`
- [ ] Update line 59: Extract error to field `error = ?err, "Failed to bind"`
- [ ] Update line 78: Extract error to field `error = ?err, "Connection serving failed"`
- [ ] Update line 82-85: Extract error to field `error = ?err, "Failed to accept connection"`
- [ ] Update line 91: Make message constant `"Metrics server shutdown"`
- [ ] Verify all changes compile: `cargo check`

#### `src/service.rs` - `metrics_service_router()` function

- [ ] Add `#[instrument(...)]` with early fields:
  - `method = %req.method()`
  - `path = req.uri().path()`
  - `host = tracing::field::Empty`
  - `user_agent = tracing::field::Empty`
- [ ] Add header extraction after function start:
  - Extract `host` header with `Span::current().record("host", ...)`
  - Extract `user-agent` header with `Span::current().record("user_agent", ...)`
- [ ] Remove entire `if enabled!(Level::TRACE)` block (lines 99-116)
- [ ] Update line 122: Change to `warn!(error = %error, "Failed to encode metrics");`
- [ ] Verify all changes compile: `cargo check`

### Post-Implementation

#### Syntax and Compilation
- [ ] `cargo check` passes for magicblock-metrics
- [ ] No compilation errors or warnings

#### Linting
- [ ] `make lint` passes (clippy with all warnings denied)

#### Tests
- [ ] `cargo nextest run -p magicblock-metrics` passes
- [ ] All unit tests pass without errors

#### Code Quality Verification
- [ ] All log messages are constant text (no format variables)
- [ ] All contextual information extracted to fields
- [ ] No duplicate information (in both message and field)
- [ ] Correct format specifiers used:
  - `%` for types with good Display impl (addr, method)
  - `?` for error types
  - bare for primitive values
- [ ] No manual span creations remain (`info_span!`, `debug_span!`, etc.)

#### Visual Inspection
- [ ] Review changes match the before/after examples
- [ ] Verify `#[instrument]` attributes are correctly formatted
- [ ] Verify field names follow snake_case convention

---

## Testing & Verification

### 1. Compile Check
```bash
cd magicblock-metrics
cargo check
```

**Expected**: Should pass without errors.

### 2. Lint Check
```bash
make lint
```

**Expected**: Should pass (all clippy warnings must be fixed).

### 3. Run Tests
```bash
cargo nextest run -p magicblock-metrics
```

**Expected**: All tests pass.

### 4. Verify Logging Output

Start a test server and inspect logs:

```bash
RUST_LOG=debug cargo test --package magicblock-metrics --lib -- --nocapture 2>&1 | grep -A5 "metrics"
```

**What to look for**:
- Constant message text (e.g., "Metrics server started")
- Fields appear before the colon (e.g., `addr=127.0.0.1:8080`)
- No `[param=value]` prefixes in the message itself
- No variable interpolation in messages

**Example good output**:
```
2024-01-16T10:30:45Z INFO mbv addr=127.0.0.1:8080  Metrics server started
2024-01-16T10:30:46Z ERROR mbv error=bind_failed   Failed to bind
```

**Example bad output** (what NOT to see):
```
2024-01-16T10:30:45Z INFO mbv  Serving 127.0.0.1:8080/metrics
2024-01-16T10:30:46Z ERROR mbv  Failed to bind to 127.0.0.1:8080: error_details
```

---

## Summary

**Implementation scope**: 2 files, 2 async functions, 6 log statements

**Key points**:
1. Add `#[instrument]` to both async functions in `src/service.rs`
2. Extract all parameters from format strings into fields
3. Make all messages constant and action-oriented
4. Use correct format specifiers (%, ?, bare)
5. Remove trace logging conditional (replaced by span)
6. No other files need changes

**Expected outcome**: Clean, queryable, aggregatable structured logging for magicblock-metrics HTTP server and metrics collection.

---

## Pattern Consistency

This implementation follows the exact pattern from `02_plan-general.md`:

✅ Every async function gets `#[instrument]`
✅ Messages are constant, action-oriented  
✅ Parameters extracted to structured fields
✅ Correct format specifiers applied
✅ No manual span creations
✅ Error context automatically inherited via span

All changes are deterministic and require no design decisions.