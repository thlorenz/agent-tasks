# Structured Logging with `tracing` and `tracing-subscriber`

A comprehensive guide for leveraging structured logging in the MagicBlock validator codebase, focusing on the `magicblock-chainlink` module's account subscription and stream management operations.

---

## Table of Contents

1. [What is Structured Logging](#what-is-structured-logging)
2. [Current State: Unstructured Logging Examples](#current-state-unstructured-logging-examples)
3. [Structured Logging Examples](#structured-logging-examples)
4. [Best Practices](#best-practices)
5. [Benefits](#benefits)

---

## What is Structured Logging

### Traditional String Formatting vs. Structured Logging

**Traditional logging** (old approach with `log` + `env_logger`):
```rust
warn!("Failed to subscribe to account {}: {}", pubkey, error);
```

This produces a single **unstructured string**:
```
Failed to subscribe to account D7r... : connection timeout
```

**Structured logging** (new approach with `tracing` + `tracing-subscriber`):
```rust
warn!(pubkey = %pubkey, error = ?error, "Failed to subscribe to account");
```

This produces **key-value pairs that machines can parse**:
```
event: "Failed to subscribe to account"
pubkey: "D7r..."
error: "connection timeout"
timestamp: "2024-01-16T10:30:45Z"
target: "magicblock_chainlink"
level: "WARN"
```

### Why Does This Matter?

1. **Queryability**: You can filter logs by `pubkey=D7r*` or `error=timeout` instead of grepping for substrings
2. **Aggregation**: Tools like Datadog, Grafana Loki, or ELK can count "failed subscriptions by account" without parsing strings
3. **Debugging**: Related information stays together as fields, not scattered across log text
4. **Observability**: Metrics can be derived directly from structured fields

---

## Current State: Unstructured Logging Examples

Here are real examples from `magicblock-chainlink` showing the current logging patterns:

### Example 1: Subscription Failure with Client ID

**File**: [`chain_laser_actor.rs:202-205`](file:///Users/thlorenz/dev/mb/wizard/magicblock-validator/magicblock-chainlink/src/remote_account_provider/chain_laser_actor.rs#L202-L205)

```rust
fn shutdown(&mut self) {
    info!(
        "[client_id={}] Shutting down ChainLaserActor",
        self.client_id
    );
    // ...
}
```

**Current output**:
```
Shutting down ChainLaserActor [client_id=abc-123]
```

**Issues**:
- Client ID is embedded in the message string
- Can't easily filter/aggregate by client_id without parsing strings
- If message format changes, parsing breaks

---

### Example 2: Subscription Response Error

**File**: [`chain_pubsub_client.rs:269-271`](file:///Users/thlorenz/dev/mb/wizard/magicblock-validator/magicblock-chainlink/src/remote_account_provider/chain_pubsub_client.rs#L269-L271)

```rust
warn!("ChainPubsubClientImpl::subscribe - RecvError occurred while awaiting subscription response for {}: {err:?}. This indicates the actor sender was dropped without responding.", pubkey);
```

**Current output**:
```
ChainPubsubClientImpl::subscribe - RecvError occurred while awaiting subscription response for D7r...: connection timeout. This indicates the actor sender was dropped without responding.
```

**Issues**:
- Very long unstructured message
- Multiple pieces of information mixed together
- Can't distinguish between which account failed or what error type it was
- Hard to count "how many subscribe errors happened for pubkey X"

---

### Example 3: Subscription Update with Slot Information

**File**: [`mod.rs:541-542`](file:///Users/thlorenz/dev/mb/wizard/magicblock-validator/magicblock-chainlink/src/remote_account_provider/mod.rs#L541-L542)

```rust
warn!("Received stale subscription update for {} at slot {}. Fetch started at slot {}",
    update.pubkey, slot, fetch_start_slot);
```

**Current output**:
```
Received stale subscription update for D7r... at slot 1234567. Fetch started at slot 1234560
```

**Issues**:
- Positional parameters make code hard to maintain
- Can't filter "show me all stale updates with slot_diff > 100"
- No way to correlate with other events for the same account

---

### Example 4: Account Subscription Error

**File**: [`chain_pubsub_actor.rs:407`](file:///Users/thlorenz/dev/mb/wizard/magicblock-validator/magicblock-chainlink/src/remote_account_provider/chain_pubsub_actor.rs#L407)

```rust
error!("[client_id={client_id}] Failed to subscribe to account {pubkey} {err:?}");
```

**Current output**:
```
Failed to subscribe to account D7r... connection_refused. [client_id=xyz-789]
```

**Issues**:
- Format string approach is fragile
- Error details are debug-formatted, not separated as distinct fields
- Can't easily correlate subscription failures across multiple clients

---

## Structured Logging Examples

Now let's refactor each example using structured logging with the `tracing` crate:

### Example 1: Structured Shutdown Logging

**Before**:
```rust
info!(
    "[client_id={}] Shutting down ChainLaserActor",
    self.client_id
);
```

**After**:
```rust
info!(
    client_id = %self.client_id,
    "Shutting down ChainLaserActor"
);
```

**Benefits**:
- `client_id` is now a structured field
- Subscriber can filter on `client_id=abc-123`
- Message string is clean and constant (good for grouping similar events)

**Example output** (with a JSON subscriber):
```json
{
  "timestamp": "2024-01-16T10:30:45.123456Z",
  "level": "INFO",
  "message": "Shutting down ChainLaserActor",
  "client_id": "abc-123",
  "target": "magicblock_chainlink::remote_account_provider::chain_laser_actor",
  "module_path": "magicblock_chainlink::remote_account_provider::chain_laser_actor",
  "file": "chain_laser_actor.rs",
  "line": 202
}
```

---

### Example 2: Structured Subscription Error with Context

**Before**:
```rust
warn!("ChainPubsubClientImpl::subscribe - RecvError occurred while awaiting subscription response for {}: {err:?}. This indicates the actor sender was dropped without responding.", pubkey);
```

**After**:
```rust
warn!(
    pubkey = %pubkey,
    error = ?err,
    reason = "actor_sender_dropped",
    "Failed to await subscription response"
);
```

**Benefits**:
- Each piece of information is a separate field
- Reason is categorized (not buried in text)
- Can filter: `level=WARN reason=actor_sender_dropped` to find all sender-dropped errors
- Error details are properly formatted

**Example output**:
```json
{
  "timestamp": "2024-01-16T10:30:45.987654Z",
  "level": "WARN",
  "message": "Failed to await subscription response",
  "pubkey": "D7r7JZfKjx7dTuVAcWYjRyQMXYmKAP9EKVrgLjM1fzUP",
  "error": "RecvError: channel closed",
  "reason": "actor_sender_dropped",
  "target": "magicblock_chainlink::remote_account_provider::chain_pubsub_client"
}
```

---

### Example 3: Structured Stale Update Detection with Slot Tracking

**Before**:
```rust
warn!("Received stale subscription update for {} at slot {}. Fetch started at slot {}",
    update.pubkey, slot, fetch_start_slot);
```

**After**:
```rust
let slot_diff = slot.saturating_sub(fetch_start_slot);
warn!(
    pubkey = %update.pubkey,
    current_slot = slot,
    fetch_started_at_slot = fetch_start_slot,
    slot_diff = slot_diff,
    "Received stale subscription update"
);
```

**Benefits**:
- All slot information is queryable
- Can now filter: `slot_diff > 10` to find "very stale" updates
- Metrics can track average `slot_diff` for stale updates
- Easier to correlate with other events for the same `pubkey`

**Example output**:
```json
{
  "timestamp": "2024-01-16T10:30:46.111111Z",
  "level": "WARN",
  "message": "Received stale subscription update",
  "pubkey": "D7r7JZfKjx7dTuVAcWYjRyQMXYmKAP9EKVrgLjM1fzUP",
  "current_slot": 1234567,
  "fetch_started_at_slot": 1234560,
  "slot_diff": 7,
  "target": "magicblock_chainlink::remote_account_provider"
}
```

---

### Example 4: Structured Account Subscription Error with Span Context

**Before**:
```rust
error!("[client_id={client_id}] Failed to subscribe to account {pubkey} {err:?}");
```

**After**:
```rust
error!(
    client_id = %client_id,
    pubkey = %pubkey,
    error = ?err,
    "Failed to subscribe to account"
);
```

**With Span** (wrapping the entire subscription operation):
```rust
let span = tracing::info_span!(
    "account_subscription",
    client_id = %client_id,
    pubkey = %pubkey
);
let _enter = span.enter();

match pubsub_connection.account_subscribe(&pubkey, config.clone()).await {
    Ok((stream, unsubscribe)) => {
        info!("Subscription established");
        // Handle subscription...
    }
    Err(err) => {
        error!(
            error = ?err,
            "Failed to subscribe to account"
        );
    }
}
```

**Benefits**:
- Span wraps the entire operation
- All logs within the span inherit `client_id` and `pubkey`
- Error logs include context without re-stating it
- Related events are grouped together logically

**Example output** (span shows related logs grouped):
```
↳ account_subscription{client_id=xyz-789, pubkey=D7r...}
  ├─ event "Attempting to subscribe to account"
  ├─ event "Failed to subscribe to account" error=ConnectionRefused
  └─ event "Aborting subscription and signaling connection issue"
```

---

## Best Practices

### 1. When to Use Fields vs Spans

**Use Fields** for:
- Individual log events with metadata
- Error details, status codes, counts
- Event-specific information

```rust
// ✅ Good: error details as fields
error!(
    request_id = %request_id,
    status_code = 500,
    error = ?e,
    "Failed to process request"
);
```

**Use Spans** for:
- Logical operations with multiple related events
- Context that applies to a group of logs
- Track entry/exit of functions or workflows

```rust
// ✅ Good: operation context in span
let span = info_span!(
    "process_subscription_batch",
    batch_id = %batch_id,
    account_count = accounts.len()
);
let _guard = span.enter();

for account in accounts {
    // Each log automatically includes batch_id and account_count
    info!(account = %account.pubkey, "Processing account");
}
```

### 2. Naming Conventions for Fields and Spans

**Field Names**:
- Use **snake_case**: `client_id`, `slot_diff`, `fetch_started_at_slot`
- Be **specific**: `error` not `err`, `pubkey` not `pub_key`
- Include **units** when helpful: `duration_ms`, `size_bytes`, `slot_number`
- Use **suffixes** for clarity: `_count`, `_bytes`, `_ms`, `_slot`

```rust
// ✅ Good
info!(
    account_count = 42,
    duration_ms = 156,
    batch_id = %batch_id,
    "Batch processed"
);

// ❌ Avoid
info!(
    c = 42,
    d = 156,
    "Batch processed"
);
```

**Span Names**:
- Use **snake_case**, present tense or gerund: `account_subscription`, `processing_batch`
- Short but descriptive: `subscribe` not `subscribing_to_account`
- Avoid redundancy: `account_subscription` not `account_subscription_span`

```rust
// ✅ Good span names
let span = info_span!("account_subscription");
let span = debug_span!("fetch_remote_account");
let span = warn_span!("reconnection_attempt");

// ❌ Avoid
let span = info_span!("doing_subscription_stuff");
let span = debug_span!("account_subscription_span");
```

### 3. Performance Considerations

**Use `%` for Display formatting** (cheaper):
```rust
// ✅ Efficient: uses Display impl
info!(pubkey = %pubkey, "Account update");
```

**Use `?` for Debug formatting** (slower, only when necessary):
```rust
// ✅ Only for complex/debug types
error!(error = ?err, "Subscription failed");
```

**Avoid expensive computations in log statements**:
```rust
// ❌ Bad: field computation happens even if log is disabled
debug!(message_size = vec.len() * 32, "Processing vector");

// ✅ Good: only compute if level is enabled
if tracing::enabled!(tracing::Level::DEBUG) {
    let size = vec.len() * 32;
    debug!(message_size = size, "Processing vector");
}
```

**Lazy evaluation with tracing macros**:
```rust
// ✅ Good: string interpolation happens only if log is enabled
debug!(
    accounts = accounts.iter().map(|a| a.pubkey).collect::<Vec<_>>(),
    "Processing batch"
);
```

### 4. Integration with tracing-subscriber Filters

**Filter by level**:
```rust
// Show only WARN and ERROR logs
let filter = EnvFilter::new("warn");
```

**Filter by target**:
```rust
// Show DEBUG for chainlink, INFO for everything else
let filter = EnvFilter::new("info,magicblock_chainlink=debug");
```

**Filter by field values**:
```rust
// Custom filter - only show errors for specific client
use tracing_subscriber::filter::FilterFn;

let filter = FilterFn::new(|metadata| {
    // This is basic; tracing-filter supports more sophisticated filtering
    metadata.is_event()
});
```

**Layer composition**:
```rust
use tracing_subscriber::{fmt, prelude::*, EnvFilter};

let fmt_layer = fmt::layer()
    .with_target(true)
    .with_file(true)
    .with_line_number(true);

let filter_layer = EnvFilter::try_from_default_env()
    .unwrap_or_else(|_| EnvFilter::new("info"));

tracing_subscriber::registry()
    .with(filter_layer)
    .with(fmt_layer)
    .init();
```

### 5. Structured Logging Patterns for magicblock-chainlink

#### Pattern 1: Subscription Lifecycle

```rust
// Entry point with context
let span = info_span!(
    "account_subscription",
    client_id = %client_id,
    pubkey = %pubkey
);
let _enter = span.enter();

info!("Initiating subscription");

// On successful subscription
match pubsub_connection.account_subscribe(&pubkey, config).await {
    Ok((stream, unsub)) => {
        info!("Subscription established");
        
        // Inner loop with updates
        loop {
            match stream.next().await {
                Some(update) => {
                    debug!(
                        slot = update.context.slot,
                        lamports = update.value.lamports,
                        "Account update received"
                    );
                }
                None => {
                    warn!("Stream ended unexpectedly");
                    break;
                }
            }
        }
    }
    Err(e) => {
        error!(error = ?e, "Failed to subscribe");
    }
}
```

#### Pattern 2: Batch Operations with Progress

```rust
let span = info_span!(
    "resubscribe_accounts",
    total_accounts = accounts.len()
);
let _enter = span.enter();

let mut success_count = 0;
let mut failure_count = 0;

for account in accounts {
    match self.subscribe(&account.pubkey).await {
        Ok(_) => {
            success_count += 1;
            debug!(pubkey = %account.pubkey, "Resubscribed");
        }
        Err(e) => {
            failure_count += 1;
            warn!(
                pubkey = %account.pubkey,
                error = ?e,
                "Failed to resubscribe"
            );
        }
    }
}

info!(
    success_count = success_count,
    failure_count = failure_count,
    "Resubscription batch complete"
);
```

#### Pattern 3: State Transitions

```rust
let span = debug_span!(
    "connection_state",
    client_id = %self.client_id
);
let _enter = span.enter();

match (self.state, new_state) {
    (ConnectionState::Connected, ConnectionState::Disconnected) => {
        warn!(
            reason = ?disconnect_reason,
            "Connection lost"
        );
        self.state = new_state;
    }
    (ConnectionState::Disconnected, ConnectionState::Reconnecting) => {
        info!(
            attempt_number = self.reconnect_attempts,
            "Attempting reconnection"
        );
        self.state = new_state;
    }
    _ => {
        debug!(
            from_state = ?self.state,
            to_state = ?new_state,
            "Unexpected state transition"
        );
    }
}
```

---

## Benefits

### 1. Observability Improvements

**Before** (unstructured):
```
warn: Failed to subscribe to account D7r7... RecvError occurred. This indicates the actor sender was dropped.
warn: Failed to subscribe to account GHs2... ConnectionRefused occurred. This indicates the actor sender was dropped.
```

To find "how many subscription failures happened?", you must grep and count lines.

**After** (structured):
```json
{"level":"WARN","message":"Failed to await subscription response","pubkey":"D7r7...","error":"RecvError"}
{"level":"WARN","message":"Failed to await subscription response","pubkey":"GHs2...","error":"ConnectionRefused"}
```

Now you can:
- **Count**: `count(level=WARN AND message="Failed to await subscription response")`
- **Group**: `group_by(error)` → See 50 `RecvError`, 10 `ConnectionRefused`
- **Correlate**: Find all failures for `pubkey=D7r7*` across time

### 2. Monitoring & Alerting

With structured fields, you can:

**Create metrics**:
```
subscription_failures_total{client_id="abc-123", error="RecvError"} = 15
subscription_failures_total{client_id="abc-123", error="ConnectionRefused"} = 3
```

**Set alerts**:
```
alert: HighSubscriptionErrorRate
if: rate(subscription_failures_total[5m]) > 10 per minute
```

**Create dashboards**:
- Graph subscription success rate by client
- Show error distribution pie chart
- Track recovery time from disconnection

### 3. Debugging & Troubleshooting

Structured logs with spans make debugging easier:

```
↳ remote_account_update{pubkey=D7r..., client_id=abc-123}
  ├─ 10:30:45.123 INFO "Account update received" slot=1234567
  ├─ 10:30:46.456 WARN "Stale update detected" current_slot=1234500 fetch_slot=1234560
  ├─ 10:30:47.789 WARN "Received stale subscription update" slot_diff=67
  └─ 10:30:48.012 ERROR "Failed to forward update" error=ChannelClosed
```

Instead of:

```
10:30:45.123 INFO Account update for D7r... at slot 1234567
10:30:46.456 WARN Received stale subscription update for D7r... at slot 1234500. Fetch started at slot 1234560
10:30:47.789 WARN Failed to forward subscription update: channel closed
```

### 4. Code Maintainability

Structured logging encourages:
- **Consistent field naming**: Easy to search codebase for how people log errors
- **Clear intent**: Fields are explicit, not buried in strings
- **Better error handling**: Each piece of context is named and intentional
- **Easier refactoring**: Changing log format doesn't break your log processing

### 5. Compliance & Audit

Structured logs are easier to:
- **Parse and archive** in compliance systems
- **Redact sensitive data** (field-by-field filtering)
- **Correlate across systems** using consistent field names (e.g., `request_id`, `user_id`)
- **Generate audit trails** with queryable fields

---

## Quick Reference: Tracing Macro Syntax

```rust
// Basic log with message
info!("Something happened");

// Log with Display-formatted field (efficient)
info!(pubkey = %pubkey, "Account updated");

// Log with Debug-formatted field (use for complex types)
info!(error = ?err, "Operation failed");

// Log with value (debug format)
info!(count = 42, "Processed items");

// Multiple fields
info!(
    client_id = %client_id,
    pubkey = %pubkey,
    slot = slot,
    error = ?err,
    "Failed to process account update"
);

// Create span (wraps multiple related logs)
let span = info_span!("operation", client_id = %client_id);
let _guard = span.enter();
info!("Inside span"); // inherits client_id field

// Debug-level span
let span = debug_span!("fetch", account = %account);

// Span with computed field
let span = warn_span!(
    "reconnection",
    attempt = attempt_count,
    backoff_ms = 1000 * 2_u32.pow(attempt_count)
);
```

---

## Migration Checklist

When migrating existing logs to structured format:

- [ ] Identify all log statements in your module
- [ ] Extract string formatting into separate fields
- [ ] Define consistent field names (snake_case, specific)
- [ ] Add spans around logical operations
- [ ] Test that logging still works with your subscriber setup
- [ ] Verify logs appear with fields in your target output format
- [ ] Update documentation with your field naming conventions
- [ ] Train team on structured logging patterns

---

## Additional Resources

- [Tracing Crate Docs](https://docs.rs/tracing/)
- [Tracing-Subscriber Docs](https://docs.rs/tracing-subscriber/)
- [Structured Logging with Tracing](https://tokio.rs/tokio/tutorial/tracing)
- [Observability with Tracing](https://docs.rs/tracing-subscriber/latest/tracing_subscriber/)
