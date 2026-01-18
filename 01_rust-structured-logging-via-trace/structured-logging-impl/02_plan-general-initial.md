# General Migration Plan: Structured Logging Across MagicBlock Codebase

Based on the structured logging examples in `01_samples.md`, this document provides a strategic plan for migrating the MagicBlock validator codebase to use structured fields and spans effectively.

---

## Table of Contents

1. [Message Rewriting Strategy](#message-rewriting-strategy)
2. [Span Implementation Strategy](#span-implementation-strategy)
3. [Span Naming Conventions](#span-naming-conventions)
4. [Span Nesting and Scoping](#span-nesting-and-scoping)
5. [Implementation Phases](#implementation-phases)
6. [Validation Checklist](#validation-checklist)
7. [Common Patterns](#common-patterns)
8. [Decision Matrix](#decision-matrix)

---

## Message Rewriting Strategy

### Principle: From Strings to Fields

The goal is to move context information from message strings into structured fields, making the core message concise and machine-groupable.

### Step 1: Identify What Can Be Extracted

**Pattern 1: Parameters in formatted strings**
```rust
// Before
warn!("Failed to subscribe to account {} at slot {}", pubkey, slot);

// After
warn!(
    pubkey = %pubkey,
    slot = slot,
    "Failed to subscribe to account"
);
```

**Pattern 2: State values in brackets or parentheses**
```rust
// Before
info!("[client_id={}] Subscription activated", client_id);

// After
info!(
    client_id = %client_id,
    "Subscription activated"
);
```

**Pattern 3: Multiple error contexts**
```rust
// Before
error!("Error processing account {}: {} (reason: {})", pubkey, error, reason);

// After
error!(
    pubkey = %pubkey,
    error = ?error,
    reason = reason,
    "Error processing account"
);
```

### Step 2: Field Naming Conventions

Use **snake_case** for all field names, following these patterns:

| Pattern | Example | Use Case |
|---------|---------|----------|
| `<noun>_id` | `client_id`, `request_id`, `subscription_id` | Identifiers |
| `<noun>_count` | `subscription_count`, `error_count`, `retry_count` | Counts/quantities |
| `<noun>_ms` | `latency_ms`, `duration_ms`, `timeout_ms` | Millisecond durations |
| `<noun>_slot` | `current_slot`, `fetch_started_at_slot` | Slot numbers |
| `<noun>_bytes` | `data_bytes`, `message_bytes` | Byte sizes |
| `from_<noun>` | `from_state`, `from_slot` | Source/original values |
| `to_<noun>` | `to_state`, `to_slot` | Destination/new values |
| Simple nouns | `error`, `reason`, `source`, `status` | Values, enums, descriptions |

### Step 3: Format Specifiers

Use appropriate format specifiers for different field types:

```rust
// Display formatting (% prefix) - for types with good Display impl
info!(pubkey = %pubkey, "...");              // Pubkey
info!(socket = %socket_addr, "...");        // SocketAddr
info!(error_msg = %error, "...");           // Custom Display

// Debug formatting (? prefix) - for complex types or enums
warn!(error = ?err, "...");                 // Error types
debug!(state = ?connection_state, "...");   // Enums
debug!(details = ?response, "...");         // Complex structs

// Bare value (no prefix) - for primitives
info!(slot = 12345, "...");                 // u64
info!(count = 42, "...");                   // i32
info!(success = true, "...");               // bool
```

### Step 4: Rewriting Messages

The log message itself should be:
- **Concise**: 5-10 words typically
- **Constant**: Same message for the same event type (enables grouping)
- **Action-oriented**: Describe what happened, not debug details
- **Descriptive**: Clear enough to stand alone

**Examples**:
- ✅ `"Failed to subscribe to account"` → grouped across all failures
- ❌ `"Failed to subscribe to {} for reason {}"` → message varies, no grouping
- ✅ `"Account update received"` → constant, clean
- ❌ `"Account update at slot {} lamports {}"` → message includes data

---

## Span Implementation Strategy

Spans provide **hierarchical context** around related log events. They track which operation a log belongs to.

### When to Add Spans

Add spans when:

1. **Async functions** that do significant work (subscription handling, remote requests)
2. **Request/response cycles** (HTTP endpoints, RPC handlers)
3. **State transitions** (connection lifecycle, sync progress)
4. **Batch operations** (processing multiple items in a loop)
5. **Error paths** (when handling errors with recovery logic)
6. **Cross-subsystem calls** (when calling into another module)

**Don't add spans** for:
- Trivial operations (simple helper functions)
- Inline math/computations (no observable work)
- Frequently called synchronous functions (overhead)

### Implementation Patterns

#### Pattern A: Using `#[instrument]` on async functions

For async functions (preferred when possible):

```rust
use tracing::instrument;

#[instrument(skip(self), fields(client_id = %self.client_id))]
async fn subscribe(&mut self, pubkey: &Pubkey) -> Result<()> {
    info!("Initiating subscription");
    // ... function body
    info!("Subscription established");
    Ok(())
}
```

**Advantages**:
- Automatic span creation/exit at function boundaries
- Automatic field capture at call entry
- Cleaner code
- Compiler checks field references

**Key points**:
- Use `skip()` for unprintable types or to avoid circular data structures
- Specify fields explicitly if function doesn't have simple field access
- The function name becomes the span name

#### Pattern B: Manual span creation for granular control

When you need spans within a function or for synchronous code:

```rust
fn process_update(&mut self, update: AccountUpdate) {
    let span = info_span!(
        "process_account_update",
        pubkey = %update.pubkey,
        slot = update.slot
    );
    let _guard = span.enter();

    if let Err(e) = self.validate(&update) {
        warn!(error = ?e, "Validation failed");
        return;
    }

    info!("Update valid, forwarding");
    // Rest of function inherits span context
}
```

**Key patterns**:
- Create span with `info_span!()`, `debug_span!()`, etc.
- Call `.enter()` to activate it
- Store result in `_guard` or `_enter` variable (keeps guard in scope)
- All logs within the guard inherit span fields

#### Pattern C: Nested spans for multi-level operations

```rust
#[instrument(skip(self))]
async fn resubscribe_all(&mut self) -> Result<()> {
    info!("Starting resubscription");

    for (idx, account) in self.accounts.iter().enumerate() {
        let span = debug_span!(
            "resubscribe_account",
            account_index = idx,
            pubkey = %account.pubkey
        );
        let _guard = span.enter();

        match self.subscribe(&account.pubkey).await {
            Ok(_) => debug!("Success"),
            Err(e) => warn!(error = ?e, "Failed"),
        }
    }

    info!("Resubscription complete");
    Ok(())
}
```

### Context Preservation in Async Code

When spawning tasks or using message-passing, preserve span context:

```rust
// ✅ Correct: Spawn with inherited span context
#[instrument]
async fn main_loop(&self) {
    for item in items {
        let handle = tokio::spawn(async move {
            // This task automatically inherits the parent span
            info!("Processing");
            process_item(item).await
        });
    }
}

// ✅ Also correct: Use instrument on spawned function
#[instrument(skip(item))]
async fn process_item(item: Item) {
    info!("Starting");
    // Has its own span, inherits parent context
}
```

---

## Span Naming Conventions

### Naming Pattern

Use this hierarchical structure:

```
<verb>_<noun>
or
<domain>_<operation>
```

**Examples**:
- `subscribe_account` - subscribing to an account
- `sync_state` - syncing state
- `handle_update` - handling an incoming update
- `fetch_accounts` - fetching accounts from remote
- `reconnect_client` - client reconnection attempt

### Domain-Specific Examples

**For subscription operations** (magicblock-chainlink):
```rust
info_span!("subscribe_account", pubkey = %pubkey)
debug_span!("process_subscription_update", slot = slot)
warn_span!("resubscribe_batch", total = accounts.len())
```

**For connection lifecycle**:
```rust
info_span!("establish_connection", endpoint = endpoint)
debug_span!("maintain_connection", client_id = %client_id)
warn_span!("handle_disconnection", reason = "timeout")
```

**For transaction processing**:
```rust
info_span!("process_transaction", signature = %tx_sig)
debug_span!("validate_transaction", lamports = amount)
error_span!("retry_transaction", attempt = 3)
```

**For error recovery**:
```rust
warn_span!("reconnection_attempt", backoff_ms = 5000)
error_span!("error_recovery", error_type = "network_timeout")
```

### Naming Guidelines

- **Verb-first**: Start with action (subscribe, process, handle, fetch, sync)
- **Specific nouns**: Prefer `subscribe_account` over `subscribe_operation`
- **Lowercase with underscores**: Follow Rust conventions
- **Consistent across codebase**: Same operation → same span name
- **Length**: Keep 2-4 words (readability in traces)

---

## Span Nesting and Scoping

### Nesting Levels

Spans form a tree hierarchy. Structure them logically:

```
request_context (top level)
├─ validate_input
├─ process_subscription
│   ├─ connect_to_endpoint
│   ├─ send_subscribe_message
│   └─ await_confirmation
└─ send_response
```

**Maximum nesting depth**: 3-4 levels typical
- Level 1: Request/operation entry point
- Level 2: Major phases (connect, subscribe, process)
- Level 3: Detailed steps (within a phase)
- Level 4+: Only for complex nested operations (rare)

### Span Scope Management

**Rule: Span scope = logical operation boundary**

```rust
// ✅ Correct: Span covers the subscription operation
{
    let span = info_span!("subscribe_account", pubkey = %pubkey);
    let _guard = span.enter();

    // All work for this subscription
    connect().await?;
    send_request().await?;
    receive_confirmation().await?;

    info!("Subscription established");
} // Span exited here, operation complete
```

```rust
// ❌ Wrong: Span leaks beyond operation
let span = info_span!("subscribe_account", pubkey = %pubkey);
let _guard = span.enter();
// ... lots of other code ...
// Span stays active for unrelated operations!
```

### Handling Concurrent Operations

For concurrent operations (StreamMap, tokio::spawn), spans should be per-operation:

```rust
#[instrument(skip(self))]
async fn manage_multiple_subscriptions(&mut self) {
    let mut handles = vec![];

    for pubkey in &self.pubkeys {
        let pubkey = *pubkey;
        let handle = tokio::spawn(async move {
            // Each task has inherited context + its own span from #[instrument]
            self.subscribe(&pubkey).await
        });
        handles.push(handle);
    }

    for handle in handles {
        let _ = handle.await;
    }
}
```

### Span Hierarchy Rules

1. **Parent-child relationships are automatic**: Child spans inherit parent fields
2. **No field hiding**: If parent has `client_id=abc`, child automatically has it
3. **Override cautiously**: Child can add fields but shouldn't re-define parent fields
4. **Guard scope**: `_guard` variable must stay in scope for entire operation

---

## Implementation Phases

Roll out structured logging systematically to avoid overwhelming developers and catch issues early.

### Phase 0: Preparation (Foundation)

**Goals**: Set up infrastructure and define standards

**Tasks**:
- [ ] Review and document field naming standards (per crate)
- [ ] Create shared constants for common field names (e.g., `const CLIENT_ID_FIELD = "client_id"`)
- [ ] Update logging initialization in `magicblock-core/src/logger/mod.rs` if needed
- [ ] Create logging guide document for team
- [ ] Add linting rules if desired (custom clippy checks for consistent field names)

**Deliverables**:
- Naming conventions documented per domain
- Example PR showing patterns
- Team training/review of standards

### Phase 1: High-Traffic Paths (Proof of Concept)

**Duration**: 1-2 weeks
**Scope**: `magicblock-chainlink` (subscription and account update handling)

**Focus areas**:
1. **Subscription lifecycle** (subscribe, unsubscribe, resubscribe)
   - Files: `remote_account_provider/mod.rs`, `chain_laser_actor.rs`, `chain_pubsub_client.rs`
   - Add spans around core operations
   - Extract ID/key parameters to fields

2. **Account update processing**
   - File: `remote_account_provider/mod.rs` (update handler)
   - Add fields for account info (pubkey, slot, lamports)
   - Detect stale updates with structured fields

3. **Error handling**
   - File: `remote_account_provider/mod.rs`, `chain_pubsub_actor.rs`
   - Structured error context
   - Categorize errors with reason field

**Success criteria**:
- [ ] All subscription operations have spans
- [ ] Update events include structured fields (pubkey, slot, lamports)
- [ ] Error logs include reason/error_type field
- [ ] Logs appear correctly in test output
- [ ] No performance degradation

### Phase 2: Error Handling & Edge Cases

**Duration**: 1-2 weeks
**Scope**: All error paths and recovery mechanisms

**Focus areas**:
1. **Reconnection logic**
   - Span around reconnection attempts
   - Field for attempt number, backoff time
   - Duration of reconnection process

2. **Timeout/failure scenarios**
   - Stream errors (connection_lost, timeout)
   - Recovery mechanisms
   - Categorize failures

3. **Validation failures**
   - Account validation errors
   - Stale data detection
   - Invalid subscription states

**Success criteria**:
- [ ] All error paths have structured context
- [ ] Recovery operations tracked with spans
- [ ] Error metrics can be derived from fields

### Phase 3: Integration Spans (Cross-Subsystem)

**Duration**: 1-2 weeks
**Scope**: Spans crossing subsystem boundaries

**Focus areas**:
1. **RPC request/response** (magicblock-rpc)
   - Span for each RPC method
   - Request/response fields

2. **Commitment service** (magicblock-committor-service)
   - Transaction processing spans
   - Intent execution spans

3. **Ledger/storage operations** (magicblock-ledger)
   - Block processing spans
   - Data persistence spans

4. **API requests** (magicblock-aperture)
   - HTTP endpoint spans
   - Request context propagation

**Success criteria**:
- [ ] Request IDs/correlation IDs tracked across subsystems
- [ ] Parent-child span relationships visible in traces
- [ ] Cross-system debugging easier with span context

### Phase 4: Remaining Crates

**Duration**: Ongoing
**Scope**: All other crates with logging

Apply the patterns from Phases 1-3 to:
- `magicblock-accounts`
- `magicblock-processor`
- `magicblock-api`
- `magicblock-table-mania`
- Others with logging

---

## Validation Checklist

Use this checklist after making logging changes:

### Message Format Validation

- [ ] Message is 5-10 words (concise)
- [ ] Message is constant (doesn't vary per instance)
- [ ] Message is action-oriented (describe what happened)
- [ ] Parameter data is in fields, not message
- [ ] No debug output (`{:?}`) in message text

### Field Validation

- [ ] Field names use snake_case
- [ ] Field names are consistent with naming conventions
- [ ] Field values don't contain duplicated message text
- [ ] Format specifiers are appropriate (%, ?, bare)
- [ ] No sensitive data in fields (passwords, keys, tokens)

### Span Validation

- [ ] Span boundaries match logical operation
- [ ] Span names follow conventions (verb_noun)
- [ ] No field hiding between parent/child
- [ ] Guard variable stays in scope for operation
- [ ] Parent-child relationships make sense

### Testing Validation

Run in test mode and verify:

```bash
RUST_LOG=debug cargo test --package magicblock-chainlink 2>&1 | less
```

Check:
- [ ] Spans appear in log output (indentation shows nesting)
- [ ] Fields appear with correct values
- [ ] No duplicate information (in message and field)
- [ ] No malformed output

### Performance Validation

- [ ] No excessive span creation per-item in loops
- [ ] High-frequency operations don't have debug_span()
- [ ] Guard variables don't cause overhead
- [ ] No regressions vs. old logging (benchmark if needed)

### Output Format Validation

For JSON output, verify:

```bash
RUST_LOG=debug cargo test 2>&1 | jq '.message' | sort | uniq
```

Check:
- [ ] Constant messages group well
- [ ] Field names are queryable
- [ ] All expected fields are present

---

## Common Patterns

### Pattern: Subscription Lifecycle

```rust
#[instrument(skip(self), fields(pubkey = %pubkey))]
async fn subscribe(&mut self, pubkey: &Pubkey) -> Result<()> {
    info!("Requesting subscription");

    {
        let span = debug_span!("connect_to_endpoint", endpoint = self.endpoint);
        let _g = span.enter();
        self.connect().await?;
        debug!("Connected");
    }

    {
        let span = debug_span!("send_subscribe");
        let _g = span.enter();
        self.send_subscribe_request().await?;
        debug!("Request sent");
    }

    {
        let span = debug_span!("await_confirmation", timeout_ms = 5000);
        let _g = span.enter();
        self.wait_confirmation().await?;
        debug!("Confirmation received");
    }

    info!("Subscription established");
    Ok(())
}
```

### Pattern: Error With Context

```rust
match operation.await {
    Ok(result) => {
        debug!(
            operation_ms = elapsed_ms,
            items_processed = result.count,
            "Operation succeeded"
        );
    }
    Err(e) => {
        warn!(
            error = ?e,
            error_type = std::any::type_name_of_val(&e),
            operation_ms = elapsed_ms,
            "Operation failed"
        );
    }
}
```

### Pattern: State Transition

```rust
if self.state != new_state {
    let span = warn_span!(
        "state_transition",
        from_state = ?self.state,
        to_state = ?new_state
    );
    let _g = span.enter();

    warn!("Transitioning");
    self.state = new_state;
}
```

### Pattern: Batch Processing

```rust
let total = items.len();
let span = info_span!("batch_processing", total_items = total);
let _g = span.enter();

let mut success_count = 0;
let mut failure_count = 0;

for (idx, item) in items.iter().enumerate() {
    debug!(item_index = idx, item_id = %item.id, "Processing");

    match process_item(item).await {
        Ok(_) => success_count += 1,
        Err(e) => {
            failure_count += 1;
            warn!(error = ?e, item_id = %item.id, "Failed");
        }
    }
}

info!(
    success_count = success_count,
    failure_count = failure_count,
    "Batch complete"
);
```

---

## Decision Matrix

Use this matrix to decide between logging approaches:

| Scenario | Message | Field | Span | Example |
|----------|---------|-------|------|---------|
| Error with context | ✅ | ✅ | ❌ | `warn!(error=?e, pubkey=%pk, "Failed")` |
| Multi-step operation | ✅ | ✅ | ✅ | Subscription with connect/request/confirm |
| Async function | ✅ | ✅ | ✅ | Use `#[instrument]` on function |
| Loop iteration (high frequency) | ✅ | ✅ | ❌ | Just fields, no per-item span |
| State change | ✅ | ✅ | ❌ | `warn!(from=s1, to=s2, "State change")` |
| Progress tracking | ❌ | ✅ | ✅ | Count successes/failures in batch |
| Debug output | ✅ | ❌ | ❌ | `debug!("Entering function {}", name)` |
| Performance-critical path | ✅ | ❌ | ❌ | No structured logging if unneeded |

---

## Next Steps

1. **Review Phase 0 tasks** with the team
2. **Identify Phase 1 files** (magicblock-chainlink focus)
3. **Create example PR** showing patterns for review
4. **Pilot Phase 1** on a single file to validate approach
5. **Roll out Phase 1** across magicblock-chainlink
6. **Measure impact** (observability, performance)
7. **Plan Phase 2** based on learnings

---

## Appendix: Quick Reference

### Field Formatting Quick Guide

```rust
// Display formatting (%)
pubkey = %pubkey              // Pubkey Display
addr = %socket_addr          // SocketAddr Display
reason = %error.to_string()  // Custom Display

// Debug formatting (?)
error = ?err                 // Error types
state = ?ConnectionState::Connected
details = ?serde_json::json!({...})

// Bare value
count = 42                   // Numbers
slot = block.slot           // Simple types
success = true              // Booleans
```

### Common Field Names (Standard)

```
client_id, request_id, subscription_id, account_id
pubkey, address, endpoint, socket_addr
slot, block_height, timestamp
error, error_type, reason, status
count, total, success_count, failure_count
duration_ms, latency_ms, timeout_ms
attempt, retry_count, backoff_ms
from_state, to_state, old_value, new_value
```
