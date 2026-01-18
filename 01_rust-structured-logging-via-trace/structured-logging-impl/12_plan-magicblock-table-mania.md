# Magicblock-Table-Mania Structured Logging Implementation Plan

Based on the deterministic pattern from `02_plan-general.md`, this document provides a concrete, step-by-step implementation guide specific to the `magicblock-table-mania` crate.

---

## Overview

The `magicblock-table-mania` crate manages lookup table lifecycle (creation, extension, deactivation, closure) on the Solana blockchain. Total files to update: **2 files** with async functions and logging statements.

The crate is relatively compact with focused responsibilities:
- **lookup_table_rc.rs** - Lookup table operations (init, extend, deactivate, close)
- **manager.rs** - Table manager coordinating table operations
- **find_tables.rs** - No async logging (only one async fn, no logs)
- **compute_budget.rs** - Purely sync, no logging
- **error.rs** - Error types only
- **derive_keypair.rs** - Sync utilities
- **lib.rs** - Module exports

---

## Async Functions with Logging by File

### File 1: `src/manager.rs`

**Async functions with logging**:

1. **`reserve_pubkey()` (line 206)**
   - Current logs (lines 209-212, 217):
     ```rust
     trace!("Added reservation for pubkey {} to table {}", pubkey, table.table_address());
     trace!("No table found for which we can reserve pubkey {}", pubkey);
     ```
   - Should capture: `pubkey`

2. **`reserve_new_pubkeys()` (line 224)**
   - Current logs (lines 258, 324, 345, 350, 406):
     - Line 258: `error!("Error extending table {}: {:?}", table.table_address(), err)`
     - Line 324: `trace!("Initializing new table...")`
     - Line 345: `trace!("Stored {}", stored.len())`
     - Line 350: `trace!("Stored {}, remaining: {}", stored_count, remaining.len())`
     - Line 406: `trace!("Successfully stored all {} pubkeys", pubkeys.len())`
   - Should capture: `pubkey_count = pubkeys.len()`, `stored_count`, `remaining_count`

3. **`wait_for_remote_tables_to_match()` (line 467)**
   - Current logs (lines 586-589, 600-603):
     - Line 586-589: `error!("Timed out waiting for remote tables...{:?}...{:#?}...{:#?}", wait_duration, local_tables, remote_tables)`
     - Line 600-603: `debug!("Waiting for remote tables {} to match local tables.", table_keys_str)`
   - Should capture: `table_count`, `timeout_ms`, `elapsed_ms`, `current_slot`

4. **`release_pubkeys()` (line 418)**
   - Current logs: None directly, but calls `release_pubkey()`

5. **`release_pubkey()` (line 438)**
   - Current logs (lines 441, 449):
     - Line 441: `trace!("Released reservation for pubkey {} from table {}", pubkey, table.table_address())`
     - Line 449: `trace!("No table found for which we can release pubkey {}", pubkey)`
   - Should capture: `pubkey`

6. **`launch_garbage_collector()` (line 630)** - Spawned async
   - Spawns `deactivate_tables()` and `close_tables()` repeatedly

7. **`deactivate_tables()` (line 692)**
   - Current logs (lines 710-714):
     - `error!("Error deactivating table {}: {:?}", table.table_address(), err)`
   - Should capture: `table_address`, `error`, `table_count`

8. **`close_tables()` (line 720)**
   - Current logs (lines 740, 772-776):
     - Line 740: `error!("Error getting latest slot: {:?}", err)`
     - Line 772-776: `error!("Error closing table {}: {:?}", deactivated_table.table_address(), err)`
   - Should capture: `error`, `slot`, `table_address`, `deactivated_table_count`

### File 2: `src/lookup_table_rc.rs`

**Async functions with logging**:

1. **`extend()` (line 491)** - Async
   - Current logs (lines 538):
     - `error!("Error extending lookup table: {:?} ({})", error, signature)`
   - Should capture: `error`, `signature`, `table_address`, `pubkey_count`

2. **`deactivate()` (line 602)**
   - Current logs (lines 632-635):
     - `error!("Error deactivating lookup table: {:?} ({})", error, signature)`
   - Should capture: `error`, `signature`, `table_address`

3. **`close()` (line 710)**
   - Current logs (lines 747-750):
     - `debug!("Error closing lookup table: {:?} ({}) - may need longer deactivation time", error, signature)`
   - Should capture: `error`, `signature`, `table_address`, `deactivation_slot`, `current_slot`

4. **`is_deactivated_on_chain()` (line 661)**
   - Current logs (lines 687-691):
     - `trace!("'{}' deactivates in {} slots", self.table_address(), deactivated_slot.saturating_sub(slot))`
   - Should capture: `table_address`, `deactivation_slot`, `current_slot`, `slots_remaining`

5. **`is_closed()` (line 695)**
   - Current logs: None

6. **`get_chain_pubkeys()` (line 765)**
   - Current logs: None

7. **`get_chain_pubkeys_for()` (line 772)**
   - Current logs: None

---

## Standard Field Naming for Magicblock-Table-Mania

| Field | Type | Format | Use Case |
|-------|------|--------|----------|
| `pubkey` | Pubkey | `%` | Account address to reserve/release |
| `table_address` | Pubkey | `%` | Lookup table address |
| `pubkey_count` | usize | bare | Number of pubkeys being processed |
| `stored_count` | usize | bare | Number of pubkeys stored |
| `remaining_count` | usize | bare | Pubkeys still to be stored |
| `table_count` | usize | bare | Number of tables involved |
| `deactivated_table_count` | usize | bare | Count of deactivated tables |
| `signature` | Signature | `%` | Transaction signature |
| `error` | Error | `?` | Error details |
| `deactivation_slot` | u64 | bare | Slot when table deactivation occurs |
| `current_slot` | u64 | bare | Current blockchain slot |
| `slots_remaining` | u64 | bare | Slots until table deactivates |
| `timeout_ms` | u64 | bare | Timeout in milliseconds |
| `elapsed_ms` | u64 | bare | Elapsed time in milliseconds |

---

## Before/After Examples

### Example 1: `manager.rs` - `reserve_pubkey()`

**Before**:
```rust
async fn reserve_pubkey(&self, pubkey: &Pubkey) -> bool {
    for table in self.active_tables.read().await.iter() {
        if table.reserve_pubkey(pubkey) {
            trace!(
                "Added reservation for pubkey {} to table {}",
                pubkey,
                table.table_address()
            );
            return true;
        }
    }
    trace!("No table found for which we can reserve pubkey {}", pubkey);
    false
}
```

**After**:
```rust
#[instrument(skip(self), fields(pubkey = %pubkey))]
async fn reserve_pubkey(&self, pubkey: &Pubkey) -> bool {
    for table in self.active_tables.read().await.iter() {
        if table.reserve_pubkey(pubkey) {
            trace!("Added reservation to table");
            return true;
        }
    }
    trace!("No table found for reservation");
    false
}
```

---

### Example 2: `manager.rs` - `reserve_new_pubkeys()`

**Before**:
```rust
async fn reserve_new_pubkeys(
    &self,
    authority: &Keypair,
    pubkeys: &HashSet<Pubkey>,
) -> TableManiaResult<()> {
    // ... setup ...
    
    while !remaining.is_empty() {
        // ... try to extend ...
        if let Err(err) = self.extend_table(...) {
            error!(
                "Error extending table {}: {:?}",
                table.table_address(),
                err
            );
        }
        
        // ...
        trace!("Initializing new table...");
        
        if let Some(stored) = self.initialize_table(...).await {
            trace!("Stored {}", stored.len());
            // ...
        }
        trace!("Stored {}, remaining: {}", stored_count, remaining.len());
    }
    
    trace!("Successfully stored all {} pubkeys", pubkeys.len());
    Ok(())
}
```

**After**:
```rust
#[instrument(skip(self, authority, pubkeys), fields(pubkey_count = pubkeys.len()))]
async fn reserve_new_pubkeys(
    &self,
    authority: &Keypair,
    pubkeys: &HashSet<Pubkey>,
) -> TableManiaResult<()> {
    // ... setup ...
    let initial_count = pubkeys.len();
    
    while !remaining.is_empty() {
        // ... try to extend ...
        if let Err(err) = self.extend_table(...) {
            error!(
                error = ?err,
                table_address = %table.table_address(),
                "Failed to extend table"
            );
        }
        
        // ...
        trace!("Initializing new table");
        
        if let Some(stored) = self.initialize_table(...).await {
            let stored_count = stored.len();
            tracing::Span::current().record("stored_count", stored_count);
            trace!("Pubkeys stored");
            // ...
        }
        let remaining = remaining.len();
        tracing::Span::current().record("remaining_count", remaining);
        trace!("Progress in reservation");
    }
    
    info!("Successfully stored all pubkeys");
    Ok(())
}
```

---

### Example 3: `lookup_table_rc.rs` - `extend()`

**Before**:
```rust
pub async fn extend(
    &self,
    rpc_client: &MagicblockRpcClient,
    authority: &Keypair,
    pubkeys: &[Pubkey],
    compute_budget: &TableManiaComputeBudget,
) -> TableManiaResult<()> {
    check_max_pubkeys(pubkeys)?;
    
    // ... transaction setup ...
    
    let outcome = rpc_client
        .send_transaction(...)
        .await?;
    let (signature, error) = outcome.into_signature_and_error();
    if let Some(error) = &error {
        error!(
            "Error extending lookup table: {:?} ({})",
            error, signature
        );
    }
    
    // ... process signature ...
    Ok(())
}
```

**After**:
```rust
#[instrument(
    skip(self, rpc_client, authority, pubkeys, compute_budget),
    fields(
        table_address = %self.table_address(),
        pubkey_count = pubkeys.len(),
        signature = tracing::field::Empty,
        error = tracing::field::Empty,
    )
)]
pub async fn extend(
    &self,
    rpc_client: &MagicblockRpcClient,
    authority: &Keypair,
    pubkeys: &[Pubkey],
    compute_budget: &TableManiaComputeBudget,
) -> TableManiaResult<()> {
    check_max_pubkeys(pubkeys)?;
    
    // ... transaction setup ...
    
    let outcome = rpc_client
        .send_transaction(...)
        .await?;
    let (signature, error) = outcome.into_signature_and_error();
    
    tracing::Span::current().record("signature", %signature);
    
    if let Some(error) = &error {
        tracing::Span::current().record("error", ?error);
        error!("Failed to extend lookup table");
    }
    
    // ... process signature ...
    Ok(())
}
```

---

### Example 4: `lookup_table_rc.rs` - `is_deactivated_on_chain()`

**Before**:
```rust
pub async fn is_deactivated_on_chain(
    &self,
    rpc_client: &MagicblockRpcClient,
    current_slot: Option<Slot>,
) -> bool {
    let Self::Deactivated {
        deactivation_slot, ..
    } = self
    else {
        return false;
    };
    let slot = {
        if let Some(slot) = current_slot {
            slot
        } else {
            let Ok(slot) = rpc_client.get_slot().await else {
                return false;
            };
            slot
        }
    };
    // ...
    let deactivated_slot = deactivation_slot + MAX_ENTRIES as u64;
    trace!(
        "'{}' deactivates in {} slots",
        self.table_address(),
        deactivated_slot.saturating_sub(slot),
    );
    deactivated_slot <= slot
}
```

**After**:
```rust
#[instrument(
    skip(self, rpc_client),
    fields(
        table_address = %self.table_address(),
        deactivation_slot = tracing::field::Empty,
        current_slot = tracing::field::Empty,
        slots_remaining = tracing::field::Empty,
    )
)]
pub async fn is_deactivated_on_chain(
    &self,
    rpc_client: &MagicblockRpcClient,
    current_slot: Option<Slot>,
) -> bool {
    let Self::Deactivated {
        deactivation_slot, ..
    } = self
    else {
        return false;
    };
    let slot = {
        if let Some(slot) = current_slot {
            slot
        } else {
            let Ok(slot) = rpc_client.get_slot().await else {
                return false;
            };
            slot
        }
    };
    
    tracing::Span::current().record("current_slot", slot);
    tracing::Span::current().record("deactivation_slot", deactivation_slot);
    
    // ...
    let deactivated_slot = deactivation_slot + MAX_ENTRIES as u64;
    let slots_remaining = deactivated_slot.saturating_sub(slot);
    tracing::Span::current().record("slots_remaining", slots_remaining);
    
    trace!("Table deactivation in progress");
    deactivated_slot <= slot
}
```

---

## Implementation Order

Process files in this exact order:

### Priority 1: `src/lookup_table_rc.rs`
**Reason**: Core table operations, no external async dependencies. Foundational for manager.rs.

Functions to update:
1. `extend()` (line 491)
2. `deactivate()` (line 602)
3. `close()` (line 710)
4. `is_deactivated_on_chain()` (line 661)

### Priority 2: `src/manager.rs`
**Reason**: Depends on lookup_table_rc operations.

Functions to update:
1. `reserve_pubkey()` (line 206)
2. `reserve_new_pubkeys()` (line 224)
3. `release_pubkey()` (line 438)
4. `release_pubkeys()` (line 418) - if needed
5. `deactivate_tables()` (line 692)
6. `close_tables()` (line 720)
7. `wait_for_remote_tables_to_match()` (line 467)

---

## Implementation Checklist

### Phase 1: `lookup_table_rc.rs`

- [ ] Add `#[instrument]` to `extend()`
  - Fields: `table_address`, `pubkey_count`, `signature`, `error`
  - Use `tracing::field::Empty` for fields recorded dynamically
- [ ] Add `#[instrument]` to `deactivate()`
  - Fields: `table_address`, `signature`, `error`
- [ ] Add `#[instrument]` to `close()`
  - Fields: `table_address`, `signature`, `error`, `deactivation_slot`, `current_slot`
- [ ] Add `#[instrument]` to `is_deactivated_on_chain()`
  - Fields: `table_address`, `deactivation_slot`, `current_slot`, `slots_remaining`

### Phase 2: `manager.rs`

- [ ] Add `#[instrument]` to `reserve_pubkey()`
  - Fields: `pubkey`
- [ ] Add `#[instrument]` to `reserve_new_pubkeys()`
  - Fields: `pubkey_count`, `stored_count`, `remaining_count`
- [ ] Add `#[instrument]` to `release_pubkey()`
  - Fields: `pubkey`
- [ ] Add `#[instrument]` to `deactivate_tables()`
  - Fields: `table_count`, `error`
- [ ] Add `#[instrument]` to `close_tables()`
  - Fields: `deactivated_table_count`, `slot`, `error`
- [ ] Add `#[instrument]` to `wait_for_remote_tables_to_match()`
  - Fields: `table_count`, `timeout_ms`, `elapsed_ms`, `current_slot`

### Phase 3: Message Rewriting

For each file:
- [ ] Remove all message parameters from log statements
- [ ] Extract parameters to fields with correct format specifiers
- [ ] Ensure messages are constant and action-oriented
- [ ] Verify no duplicate information (message + field)

### Phase 4: Validation

- [ ] `cargo check` passes
- [ ] `make lint` passes
- [ ] `cargo nextest run -p magicblock-table-mania -j16` passes
- [ ] All log messages are constant (no variables in strings)
- [ ] All context is in structured fields
- [ ] No manual spans remain

---

## Specific Changes by Function

### `lookup_table_rc.rs::extend()`

**Lines to change**:
- Line 538: `error!("Error extending lookup table: {:?} ({})", error, signature);`

**Rewrite to**:
```rust
error!(error = ?error, signature = %signature, "Failed to extend lookup table");
```

**Add to function signature**:
```rust
#[instrument(
    skip(self, rpc_client, authority, pubkeys, compute_budget),
    fields(
        table_address = %self.table_address(),
        pubkey_count = pubkeys.len(),
    )
)]
```

---

### `lookup_table_rc.rs::deactivate()`

**Lines to change**:
- Line 632: `error!("Error deactivating lookup table: {:?} ({})", error, signature);`

**Rewrite to**:
```rust
error!(error = ?error, signature = %signature, "Failed to deactivate lookup table");
```

**Add to function signature**:
```rust
#[instrument(
    skip(self, rpc_client, authority, compute_budget),
    fields(table_address = %self.table_address())
)]
```

---

### `lookup_table_rc.rs::close()`

**Lines to change**:
- Line 747: `debug!("Error closing lookup table: {:?} ({}) - may need longer deactivation time", error, signature);`

**Rewrite to**:
```rust
debug!(
    error = ?error,
    signature = %signature,
    "Error closing lookup table; may need longer deactivation time"
);
```

**Add to function signature**:
```rust
#[instrument(
    skip(self, rpc_client, authority, compute_budget),
    fields(table_address = %self.table_address())
)]
```

---

### `lookup_table_rc.rs::is_deactivated_on_chain()`

**Lines to change**:
- Line 687: `trace!("'{}' deactivates in {} slots", self.table_address(), deactivated_slot.saturating_sub(slot),);`

**Rewrite to**:
```rust
trace!("Table deactivation in progress");
```

**Add dynamic field recording before trace**:
```rust
tracing::Span::current().record("current_slot", slot);
tracing::Span::current().record("deactivation_slot", deactivation_slot);
tracing::Span::current().record("slots_remaining", deactivated_slot.saturating_sub(slot));
```

**Add to function signature**:
```rust
#[instrument(
    skip(self, rpc_client),
    fields(
        table_address = %self.table_address(),
        deactivation_slot = tracing::field::Empty,
        current_slot = tracing::field::Empty,
        slots_remaining = tracing::field::Empty,
    )
)]
```

---

### `manager.rs::reserve_pubkey()`

**Lines to change**:
- Line 209: `trace!("Added reservation for pubkey {} to table {}", pubkey, table.table_address());`
- Line 217: `trace!("No table found for which we can reserve pubkey {}", pubkey);`

**Rewrite to**:
```rust
trace!("Added reservation to table");
trace!("No table found for reservation");
```

**Add to function signature**:
```rust
#[instrument(skip(self), fields(pubkey = %pubkey))]
```

---

### `manager.rs::reserve_new_pubkeys()`

**Lines to change**:
- Line 258: `error!("Error extending table {}: {:?}", table.table_address(), err)`
- Line 324: `trace!("Initializing new table...")`
- Line 345: `trace!("Stored {}", stored.len())`
- Line 350: `trace!("Stored {}, remaining: {}", stored_count, remaining.len())`
- Line 406: `trace!("Successfully stored all {} pubkeys", pubkeys.len())`

**Rewrite to**:
```rust
error!(error = ?err, table_address = %table.table_address(), "Failed to extend table");
trace!("Initializing new table");
trace!("Pubkeys stored");
trace!("Progress update in reservation");
info!("Successfully stored all pubkeys");
```

**Add to function signature**:
```rust
#[instrument(
    skip(self, authority, pubkeys),
    fields(pubkey_count = pubkeys.len(), stored_count = tracing::field::Empty, remaining_count = tracing::field::Empty)
)]
```

**Record dynamic fields**:
- After computing `stored.len()`: `tracing::Span::current().record("stored_count", stored.len());`
- After computing `remaining.len()`: `tracing::Span::current().record("remaining_count", remaining.len());`

---

### `manager.rs::release_pubkey()`

**Lines to change**:
- Line 441: `trace!("Released reservation for pubkey {} from table {}", pubkey, table.table_address())`
- Line 449: `trace!("No table found for which we can release pubkey {}", pubkey)`

**Rewrite to**:
```rust
trace!("Released reservation from table");
trace!("No table found for release");
```

**Add to function signature**:
```rust
#[instrument(skip(self), fields(pubkey = %pubkey))]
```

---

### `manager.rs::deactivate_tables()`

**Lines to change**:
- Line 710: `error!("Error deactivating table {}: {:?}", table.table_address(), err)`

**Rewrite to**:
```rust
error!(
    error = ?err,
    table_address = %table.table_address(),
    "Failed to deactivate table"
);
```

**Add to function signature**:
```rust
#[instrument(skip(rpc_client, authority, released_tables, compute_budget))]
```

**Add field before loop**:
```rust
let table_count = released_tables.lock().await.len();
// Then include in #[instrument]: fields(table_count)
```

---

### `manager.rs::close_tables()`

**Lines to change**:
- Line 740: `error!("Error getting latest slot: {:?}", err)`
- Line 772: `error!("Error closing table {}: {:?}", deactivated_table.table_address(), err)`

**Rewrite to**:
```rust
error!(error = ?err, "Failed to get latest slot");
error!(
    error = ?err,
    table_address = %deactivated_table.table_address(),
    "Failed to close table"
);
```

**Add to function signature**:
```rust
#[instrument(
    skip(rpc_client, authority, released_tables, compute_budget),
    fields(
        deactivated_table_count = tracing::field::Empty,
        current_slot = tracing::field::Empty,
    )
)]
```

**Record dynamic fields**:
- Before processing: `let deactivated_count = released_tables.lock().await.iter().filter(|x| x.deactivate_triggered()).count(); tracing::Span::current().record("deactivated_table_count", deactivated_count);`
- After `get_slot()`: `tracing::Span::current().record("current_slot", latest_slot);`

---

### `manager.rs::wait_for_remote_tables_to_match()`

**Lines to change**:
- Line 586: Large error with formatted parameters
- Line 600: Debug message with table keys

**Rewrite to**:
```rust
error!(
    timeout_ms = wait_for_remote_table_match.as_millis() as u64,
    elapsed_ms = start.elapsed().as_millis() as u64,
    "Timed out waiting for remote tables to match"
);
debug!("Still waiting for remote tables");
```

**Add to function signature**:
```rust
#[instrument(
    skip(self),
    fields(
        table_count = tracing::field::Empty,
        timeout_ms = tracing::field::Empty,
        current_slot = tracing::field::Empty,
    )
)]
```

**Record dynamic fields**:
- Before loop: `let table_count = matching_tables.len(); tracing::Span::current().record("table_count", table_count); tracing::Span::current().record("timeout_ms", wait_for_remote_table_match.as_millis() as u64);`
- In loop after `wait_for_next_slot()`: `tracing::Span::current().record("current_slot", slot);`

---

## Summary

**Total functions to update**: 11 across 2 files

**Total log statements to rewrite**: ~15

**Changes required**:
1. Add `#[instrument]` attribute to all 11 async functions with logging
2. Rewrite ~15 log statements to use structured fields
3. Use dynamic field recording for values computed during execution
4. Ensure all messages are constant and action-oriented

**Expected outcome**: Complete structured logging coverage for the magicblock-table-mania crate, enabling queryable and aggregatable logs across all table lifecycle operations.

---

## Validation Commands

After completing implementation:

```bash
# 1. Check compilation
cd magicblock-table-mania
cargo check

# 2. Lint
make lint

# 3. Test
cargo nextest run -p magicblock-table-mania -j16

# 4. Inspect logs
RUST_LOG=debug cargo test --package magicblock-table-mania --lib 2>&1 | head -100
```

Verify in output:
- ✅ Constant message text (no variables in messages)
- ✅ Structured fields with key=value format
- ✅ No `{}` style formatting in log messages
- ✅ Table addresses, pubkey counts, errors as separate fields
