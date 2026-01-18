# Deterministic Migration Plan: Structured Logging for MagicBlock Codebase

This plan provides **one single pattern** to follow consistently across all crates. All design decisions have been made. Follow the instructions without deviation.

---

## Table of Contents

1. [The Single Pattern](#the-single-pattern)
2. [Message Rewriting Rules](#message-rewriting-rules)
3. [Field Naming and Format Specifications](#field-naming-and-format-specifications)
4. [Span Implementation Rules](#span-implementation-rules)
5. [Implementation Procedure](#implementation-procedure)
6. [Checklist](#checklist)

---

## The Single Pattern

**Pattern: `#[instrument]` on all async functions + structured fields + error context**

```rust
#[instrument(skip(self), fields(client_id = %self.client_id, pubkey = %pubkey))]
async fn subscribe(&mut self, pubkey: &Pubkey) -> Result<()> {
    info!("Initiating subscription");

    // Step 1
    self.connect().await?;
    debug!("Connected to endpoint");

    // Step 2
    self.send_request().await?;
    debug!("Request sent");

    // Step 3
    self.await_confirmation().await?;

    info!("Subscription established");
    Ok(())
}
```

**What this gives you**:
- **Message**: Each log statement has a clear, constant message (e.g., "Subscription established")
- **Fields**: Context data is extracted as key=value pairs (pubkey, client_id, error details)
- **Span**: The entire async function is wrapped in a span. All logs within inherit the span context automatically.

**Errors automatically get span context** by virtue of being logged within an `#[instrument]`-wrapped async function. No additional work needed.

---

## Message Rewriting Rules

### Rule 1: Extract All Parameters to Fields

**Before**:
```rust
info!("Subscription activated for account {} on client {}", pubkey, client_id);
```

**After**:
```rust
info!(pubkey = %pubkey, client_id = %client_id, "Subscription activated");
```

**Rationale**: Constant message enables grouping. Parameters become queryable fields.

### Rule 2: Message Must Be Constant and Action-Oriented

**Rule**: The message string must be identical across all calls to the same log point.

**Good**:
- `"Subscription activated"` (same everywhere)
- `"Failed to subscribe"` (same everywhere)
- `"Account update received"` (same everywhere)

**Bad**:
- `"Subscription activated for {}"` (varies per call)
- `"Failed: {}"` (message changes based on error)

**Why**: Constant messages allow log aggregation and grouping in observability platforms.

### Rule 3: Message Length and Content

- **Length**: 3-6 words (e.g., "Account update received")
- **Content**: What happened, not how it happened
- **Action verb**: Start with verb when possible ("Initiating", "Processing", "Failed")
- **No implementation details**: Don't include "deserializing" or "validating" in message if not user-relevant

### Rule 4: Parameter Extraction Examples

| Scenario | Before | After |
|----------|--------|-------|
| IDs/Keys in message | `format!("Account {}", pk)` | `pubkey = %pk, "Account processed"` |
| Multiple params | `format!("{} {}", a, b)` | `field_a = a, field_b = b, "Operation complete"` |
| Status/state in message | `format!("Status: {}", s)` | `status = ?s, "State updated"` |
| Counts/numbers | `format!("Processed {} items", n)` | `item_count = n, "Batch complete"` |
| Error details | `format!("Error: {}", e)` | `error = ?e, "Operation failed"` |

---

## Field Naming and Format Specifications

### Field Naming Convention

Use **snake_case** exclusively. Pattern table:

| Pattern | Example | Use For |
|---------|---------|---------|
| `<noun>_id` | `client_id`, `pubkey`, `request_id` | Identifiers, keys |
| `<noun>_count` | `retry_count`, `success_count`, `total_count` | Counts and totals |
| `<noun>_ms` | `duration_ms`, `latency_ms`, `timeout_ms` | Millisecond durations |
| `<noun>_slot` | `current_slot`, `fetch_slot`, `start_slot` | Slot numbers |
| `<noun>_bytes` | `data_bytes`, `message_bytes`, `size_bytes` | Byte sizes |
| `from_<noun>` | `from_state`, `from_slot`, `from_value` | Source/original value |
| `to_<noun>` | `to_state`, `to_slot`, `to_value` | Target/new value |
| Simple noun | `error`, `reason`, `status`, `message`, `event` | Descriptions, enums |

### Format Specification Rules

**Rule: Always use exactly one of these formats**

| Format | Use Case | Example |
|--------|----------|---------|
| `%` (Display) | Types with good Display impl | `pubkey = %pubkey`, `addr = %socket_addr` |
| `?` (Debug) | Enums, errors, complex types | `error = ?err`, `state = ?connection_state` |
| bare (no prefix) | Primitives (u64, u32, i32, bool) | `count = 42`, `slot = 12345`, `success = true` |

**Examples**:
```rust
// Display formatting
info!(pubkey = %pubkey, "Subscription activated");
warn!(endpoint = %self.endpoint, "Connection failed");

// Debug formatting
error!(error = ?err, "Operation failed");
debug!(state = ?current_state, "State transition");

// Bare value
info!(slot = 12345, "Block processed");
warn!(attempt_count = 3, "Retrying");
debug!(success = true, "Validation passed");
```

---

## Span Implementation Rules

### Rule 1: Every Async Function Gets #[instrument]

**MANDATORY**: Every async function that does observable work (I/O, async operations, significant computation) must have `#[instrument]`.

```rust
#[instrument(skip(self), fields(client_id = %self.client_id))]
async fn subscribe(&mut self, pubkey: &Pubkey) -> Result<()> {
    // ... function body
}
```

### Rule 2: Use `skip()` for These Cases

**Skip from field capture** if:
- Type doesn't implement Display/Debug well
- Field is part of `self` and would create circular reference
- Field is very large (Vec, HashMap, etc.)
- Field is internal implementation detail

```rust
#[instrument(skip(self, large_vec), fields(item_count = large_vec.len()))]
async fn process(&self, large_vec: Vec<Item>) -> Result<()> {
    // self is skipped, but item_count is captured
}
```

### Rule 3: Span Name Comes From Function Name

**The span name is automatically derived from the function name.**

```rust
async fn subscribe(&mut self, pubkey: &Pubkey) {
    // Span name will be "subscribe"
}

async fn handle_account_update(&mut self, update: &Update) {
    // Span name will be "handle_account_update"
}
```

**Naming rules for functions** (which automatically become span names):
- Use `verb_noun` pattern: `subscribe_account`, `process_update`, `handle_error`
- Use lowercase with underscores
- Keep to 2-4 words
- Be specific: `subscribe_account` not `subscribe_operation`

### Rule 4: NO Manual Spans Needed

**You do NOT create manual spans.** The `#[instrument]` macro handles all span creation and scope.

❌ **Don't do this**:
```rust
#[instrument]
async fn foo() {
    let span = info_span!("manual_span");  // ← WRONG
    let _g = span.enter();
    // ...
}
```

✅ **Do this**:
```rust
#[instrument]
async fn foo() {
    info!("Starting operation");
    // Already wrapped in a span from #[instrument]
}
```

**Exception**: Only if a function is NOT async, then it gets no span (by design - sync code doesn't get spans).

### Rule 5: Errors Inherit Span Context Automatically

When you log an error within an `#[instrument]`-wrapped async function, it automatically inherits the span context. No special action needed.

```rust
#[instrument(skip(self), fields(pubkey = %pubkey))]
async fn subscribe(&mut self, pubkey: &Pubkey) -> Result<()> {
    match self.connect().await {
        Ok(_) => debug!("Connected"),
        Err(e) => {
            // This error log automatically has the span context from #[instrument]
            // and inherits pubkey field
            error!(error = ?e, "Failed to connect");
        }
    }
    Ok(())
}
```

---

## Implementation Procedure

Follow these steps for each crate:

### Step 1: Identify All Async Functions with Logging

Find every async function that contains a `tracing::info!`, `tracing::warn!`, `tracing::debug!`, or `tracing::error!` call.

```bash
cd <crate>
rg -A5 "async fn" src/ | rg -B5 "info!|warn!|error!|debug!"
```

### Step 2: Add #[instrument] to Each Async Function

For each async function identified:

1. Add `#[instrument]` attribute
2. Include `skip(self)` if function takes `&self` or `&mut self`
3. List fields that are meaningful context (see examples below)

**Example before**:
```rust
pub async fn subscribe(&mut self, pubkey: &Pubkey) -> Result<()> {
    info!("Subscribing to account {}", pubkey);
    // ...
}
```

**Example after**:
```rust
#[instrument(skip(self), fields(pubkey = %pubkey))]
pub async fn subscribe(&mut self, pubkey: &Pubkey) -> Result<()> {
    info!("Subscribing to account");  // Message constant now
    // ...
}
```

### Step 3: Convert Log Messages to Constant + Fields

For each log statement in the file:

1. Extract parameters from the format string into fields
2. Make the message constant (same text every time)
3. Use correct format specifiers (%, ?, or bare)

**Example transformation**:
```rust
// Before
warn!("Failed to subscribe to {} at slot {}: {:?}", pubkey, slot, error);

// After
warn!(
    pubkey = %pubkey,
    slot = slot,
    error = ?error,
    "Failed to subscribe"
);
```

### Step 4: Verify No Manual Spans

Search the file for `info_span!`, `debug_span!`, `warn_span!`, `trace_span!`.

**Result should be**: Zero manual spans (they're not needed with #[instrument]).

If you find manual spans, **convert to function calls instead**:

```rust
// Before (manual span)
#[instrument]
async fn process_items(&mut self, items: Vec<Item>) {
    for item in items {
        let span = debug_span!("process_item", id = item.id);
        let _g = span.enter();
        self.process(&item).await;
    }
}

// After (extracted function with #[instrument])
#[instrument(skip(self, item))]
async fn process_item(&mut self, item: &Item) {
    self.process(item).await;
}

#[instrument(skip(self))]
async fn process_items(&mut self, items: Vec<Item>) {
    for item in items {
        self.process_item(&item).await;
    }
}
```

### Step 5: Test and Verify

Run tests with logging enabled and verify output:

```bash
RUST_LOG=debug cargo test --package <crate-name> 2>&1 | head -50
```

Check:
- [ ] Each async function creates one top-level span
- [ ] Log messages are constant (same text each time)
- [ ] Fields appear with correct values
- [ ] No duplicate information (in message AND field)

---

## Checklist

Use this for each crate being converted:

### Pre-Implementation
- [ ] Identified all async functions with logging
- [ ] Chose which fields to include in #[instrument] annotations
- [ ] Reviewed field naming conventions

### During Implementation
- [ ] Added #[instrument] to every async function with logs
- [ ] Used correct skip() parameters
- [ ] Converted all log messages to constant text
- [ ] Extracted parameters to fields
- [ ] Used correct format specifiers (%, ?, bare)
- [ ] Deleted all manual span creations
- [ ] Verified no loops with per-item spans

### Post-Implementation

#### You must run these commands and verify:

- [ ] `cargo check` passes
- [ ] `make lint` passes
- [ ] `cargo nextest run -p <crate-name> -j16` passes
- [ ] All log messages are constant
- [ ] All context is in fields
- [ ] No manual spans remain
- [ ] Code review: patterns match this plan exactly

#### I will manually verify:

- [ ] Logging output verified with RUST_LOG=debug

##### Example Log Verification

Run test and look for patterns like:

```
// ✅ Correct: constant message, structured fields
2024-01-16T10:30:45Z WARN   pubkey=D7r7..., client_id=abc-123, error=connection_timeout  Failed to subscribe

// ❌ Wrong: varying message
2024-01-16T10:30:45Z WARN   Failed to subscribe to D7r7... for client abc-123: connection_timeout

// ❌ Wrong: message has variables
2024-01-16T10:30:45Z WARN   Failed to subscribe to {} at slot {}
```

---

## Summary

**Follow this ONE pattern for every crate**:

1. Put `#[instrument]` on every async function with logging
2. List relevant fields in the attribute (extracted from parameters)
3. Extract all parameters from log messages into fields
4. Make messages constant and action-oriented
5. Use correct format specifiers
6. Delete all manual span creations
7. Test and verify

**That's it.** This is deterministic, consistent, and requires no decisions.
