# Magicblock-Accounts-DB Structured Logging Implementation Plan

Based on the deterministic pattern from `02_plan-general.md`, this document provides a concrete, step-by-step implementation guide specific to the `magicblock-accounts-db` crate.

---

## Overview

The `magicblock-accounts-db` crate manages the core account storage system with three major components:
1. **Storage**: Memory-mapped append-only log (`AccountsStorage`)
2. **Indexing**: LMDB-based key-value index (`AccountsDbIndex`)
3. **Snapshotting**: Persistence and recovery (`SnapshotManager`)

**Total files to update: 5 files** with logging statements.

### Current Logging Summary

The crate has **minimal but critical logging**:
- `lib.rs`: 5 log statements (info, warn, error)
- `snapshot.rs`: 5 log statements (info, warn, error)
- `storage.rs`: 1 log statement (error)
- `error.rs`: 1 log statement (error, in extension trait)
- `index.rs`: 1 log statement (warn)

**Total: 13 log statements across all modules**

---

## Standard Field Naming for Magicblock-Accounts-DB

| Field | Type | Format | Use Case |
|-------|------|--------|----------|
| `pubkey` | Pubkey | `%` | Account public key |
| `slot` | u64 | bare | Block slot number |
| `error` | Error | `?` | Error details |
| `db_path` | Path | `%` | Database directory path |
| `target_slot` | u64 | bare | Target slot for restoration |
| `current_slot` | u64 | bare | Current slot |
| `restored_slot` | u64 | bare | Slot restored to |
| `blocks_count` | u32 | bare | Number of blocks |
| `size_bytes` | usize | bare | Byte size |
| `operation` | &str | bare | Operation type (insert, batch, restore) |
| `account_count` | usize | bare | Number of accounts processed |
| `processed_count` | usize | bare | Number of accounts successfully processed |
| `strategy` | &str | bare | Strategy name (Reflink, LegacyCopy) |
| `invalidated_count` | usize | bare | Number of invalidated snapshots |

---

## File Analysis & Implementation Order

### Priority 1: `src/error.rs` (Extension Trait)

**Status**: Minimal logging (1 call)

**Function with logging**:
- `LogErr::log_err()` - Extension trait method (sync, not async)

**Current log statements**:
```rust
// Line 48 - in LogErr::log_err()
error!("{}: {}", msg(), e);
```

**Implementation**:

This is a sync utility method. Per the general plan, sync code does NOT get `#[instrument]` spans. However, we should still apply structured field extraction when the trait is used via the `.log_err()` helper.

1. **Status**: No changes needed. The `LogErr` trait already captures error context through `.log_err(msg_fn)`. Users already structure this when calling:
   ```rust
   .log_err(|| "Failed to create accounts db")
   ```

2. **Rationale**: The message is already constant (provided by closure). This is acceptable as-is.

---

### Priority 2: `src/storage.rs`

**Status**: Minimal logging (1 call)

**Function with logging**:
- `validate_header()` - Validation method (sync, not async)

**Current log statements**:
```rust
// Line 198-200
error!(
    "AccountsDB corrupt: Block Size: {}, Total Blocks: {}",
    header.block_size, header.capacity_blocks
);
```

**Implementation**:

1. Add extraction of parameters to fields:
   ```rust
   error!(
       block_size = header.block_size,
       capacity_blocks = header.capacity_blocks,
       "AccountsDB corruption detected"
   );
   ```

2. **Rationale**: This is a sync validation function, so no `#[instrument]` needed. But we still extract parameters to queryable fields.

---

### Priority 3: `src/index.rs`

**Status**: Minimal logging (1 call)

**Function with logging**:
- `remove_program_index_entry()` - Cleanup method (sync, not async)

**Current log statements**:
```rust
// Line 296
warn!("account {pubkey} missing from owners index during cleanup");
```

**Implementation**:

1. Extract pubkey to field:
   ```rust
   warn!(pubkey = %pubkey, "Account missing from owners index");
   ```

2. **Message change**: Shortened and made action-oriented ("Account missing" vs "account missing").

---

### Priority 4: `src/snapshot.rs`

**Status**: 5 log statements, mostly in sync code

**Functions/Methods with logging**:
1. `SnapshotStrategy::detect()` - (sync, validation)
2. `SnapshotManager::create_snapshot()` - (async context, spawned thread)
3. `SnapshotManager::restore_from_snapshot()` - (not async, internal sync)
4. `prune_registry()` - (sync helper)

**Current log statements**:

```rust
// Line 36 - in detect()
info!("Snapshot Strategy: Reflink (Fast/CoW)");

// Line 39 - in detect()
warn!("Snapshot Strategy: Deep Copy (Slow)");

// Line 180 - in restore_from_snapshot()
info!("Restoring snapshot {} (req: {})", chosen_slot, target_slot);

// Line 185 - in restore_from_snapshot()
warn!("Pruning invalidated snapshot: {}", invalidated.display());

// Line 200-202 - in restore_from_snapshot()
error!(
    "Restore failed during promote: {}. Attempting rollback.",
    e
);

// Line 243 - in prune_registry()
warn!("Failed to prune {}: {}", path.display(), e);
```

**Implementation**:

1. **detect()**: Extract strategy to field
   ```rust
   // Line 36
   info!(strategy = "Reflink", "Snapshot strategy detected");
   
   // Line 39
   warn!(strategy = "LegacyCopy", "Snapshot strategy detected");
   ```

2. **restore_from_snapshot()**: Extract slots to fields
   ```rust
   // Line 180
   info!(
       chosen_slot = chosen_slot,
       target_slot = target_slot,
       "Restoring snapshot"
   );
   
   // Line 185
   warn!(
       invalidated_path = %invalidated.display(),
       "Pruning invalidated snapshot"
   );
   
   // Line 200-202
   error!(
       error = ?e,
       "Restore failed during promote"
   );
   ```

3. **prune_registry()**: Extract path and error to fields
   ```rust
   // Line 243
   warn!(
       path = %path.display(),
       error = ?e,
       "Failed to prune snapshot"
   );
   ```

---

### Priority 5: `src/lib.rs`

**Status**: 5 log statements, in sync and potential async contexts

**Functions with logging**:
1. `new()` - Constructor (sync)
2. `insert_batch()` - Batch insert (sync, but critical operation)
3. `trigger_background_snapshot()` - Spawns thread (sync wrapper)
4. `restore_state_if_needed()` - State restoration (sync, but critical)

**Current log statements**:

```rust
// Line 64 - in new()
info!("Resetting AccountsDb at {}", db_dir.display());

// Line 139 - in insert_batch(), on error
error!("Batch insert failed at {pubkey}: {e}");

// Line 149 - in insert_batch(), on commit error
error!("Batch insert commit failed {e}");

// Line 329 - in trigger_background_snapshot(), in spawned thread
warn!("Snapshot failed for slot {}: {}", slot, err);

// Line 347-350 - in restore_state_if_needed()
warn!(
    "Current slot {} is ahead of target {}. Rolling back...",
    self.slot(),
    target_slot
);

// Line 371 - in restore_state_if_needed()
info!("Successfully rolled back to slot {}", restored_slot);
```

**Implementation**:

1. **new()**: Extract path to field
   ```rust
   // Line 64
   info!(db_path = %db_dir.display(), "Resetting accounts database");
   ```

2. **insert_batch()**: Extract pubkey and error to fields
   ```rust
   // Line 139
   error!(pubkey = %pubkey, error = ?e, "Batch insert failed");
   
   // Line 149
   error!(error = ?e, "Batch insert commit failed");
   ```

3. **trigger_background_snapshot()**: Extract slot and error to fields
   ```rust
   // Line 329
   warn!(slot = slot, error = ?err, "Snapshot failed");
   ```

4. **restore_state_if_needed()**: Extract slots to fields
   ```rust
   // Line 347-350
   warn!(
       current_slot = self.slot(),
       target_slot = target_slot,
       "Current slot ahead of target, rolling back"
   );
   
   // Line 371
   info!(restored_slot = restored_slot, "Rolled back to snapshot");
   ```

---

## Before/After Examples

### Example 1: `snapshot.rs` - `SnapshotStrategy::detect()`

**Before**:
```rust
fn detect(dir: &Path) -> AccountsDbResult<Self> {
    if fs_backend::supports_reflink(dir)
        .log_err(|| "CoW check failed".to_string())?
    {
        info!("Snapshot Strategy: Reflink (Fast/CoW)");
        Ok(Self::Reflink)
    } else {
        warn!("Snapshot Strategy: Deep Copy (Slow)");
        Ok(Self::LegacyCopy)
    }
}
```

**After**:
```rust
fn detect(dir: &Path) -> AccountsDbResult<Self> {
    if fs_backend::supports_reflink(dir)
        .log_err(|| "CoW check failed".to_string())?
    {
        info!(strategy = "Reflink", "Snapshot strategy detected");
        Ok(Self::Reflink)
    } else {
        warn!(strategy = "LegacyCopy", "Snapshot strategy detected");
        Ok(Self::LegacyCopy)
    }
}
```

**Changes**:
- Extracted `strategy` to queryable field
- Made message constant: "Snapshot strategy detected" (same for both branches)
- Enables grouping logs by strategy in observability platform

---

### Example 2: `lib.rs` - `restore_state_if_needed()`

**Before**:
```rust
pub fn restore_state_if_needed(
    &mut self,
    target_slot: u64,
) -> AccountsDbResult<u64> {
    if target_slot >= self.slot().saturating_sub(1) {
        return Ok(self.slot());
    }

    warn!(
        "Current slot {} is ahead of target {}. Rolling back...",
        self.slot(),
        target_slot
    );

    let _guard = self.write_lock.write();
    let restored_slot = self
        .snapshot_manager
        .restore_from_snapshot(target_slot)
        .log_err(|| {
            format!(
                "Snapshot restoration failed for target {}",
                target_slot
            )
        })?;

    self.storage.reload(path)?;
    self.index.reload(path)?;

    info!("Successfully rolled back to slot {}", restored_slot);
    Ok(restored_slot)
}
```

**After**:
```rust
pub fn restore_state_if_needed(
    &mut self,
    target_slot: u64,
) -> AccountsDbResult<u64> {
    if target_slot >= self.slot().saturating_sub(1) {
        return Ok(self.slot());
    }

    warn!(
        current_slot = self.slot(),
        target_slot = target_slot,
        "Current slot ahead of target, rolling back"
    );

    let _guard = self.write_lock.write();
    let restored_slot = self
        .snapshot_manager
        .restore_from_snapshot(target_slot)
        .log_err(|| {
            format!(
                "Snapshot restoration failed for target {}",
                target_slot
            )
        })?;

    self.storage.reload(path)?;
    self.index.reload(path)?;

    info!(restored_slot = restored_slot, "Rolled back to snapshot");
    Ok(restored_slot)
}
```

**Changes**:
- Line 347: Extracted `current_slot` and `target_slot` to fields
- Message shortened: "Current slot ahead of target, rolling back" (constant)
- Line 371: Extracted `restored_slot` to field
- Message shortened: "Rolled back to snapshot" (constant)
- Enables drilling down: filter by target_slot, group by current_slot vs target_slot

---

### Example 3: `lib.rs` - `insert_batch()`

**Before**:
```rust
pub fn insert_batch<'a>(
    &self,
    accounts: impl Iterator<Item = &'a (Pubkey, AccountSharedData)> + Clone,
) -> AccountsDbResult<()> {
    let mut txn = self.index.rwtxn()?;
    let mut processed_count = 0;

    for (pubkey, account) in accounts.clone() {
        let result = self.perform_account_upsert(pubkey, account, &mut txn);
        if let Err(e) = result {
            error!("Batch insert failed at {pubkey}: {e}");
            self.rollback_borrowed_accounts(accounts, processed_count);
            return Err(e);
        }
        processed_count += 1;
    }

    if let Err(e) = txn.commit() {
        error!("Batch insert commit failed {e}");
        self.rollback_borrowed_accounts(accounts, processed_count);
        return Err(e.into());
    }
    Ok(())
}
```

**After**:
```rust
pub fn insert_batch<'a>(
    &self,
    accounts: impl Iterator<Item = &'a (Pubkey, AccountSharedData)> + Clone,
) -> AccountsDbResult<()> {
    let mut txn = self.index.rwtxn()?;
    let mut processed_count = 0;

    for (pubkey, account) in accounts.clone() {
        let result = self.perform_account_upsert(pubkey, account, &mut txn);
        if let Err(e) = result {
            error!(pubkey = %pubkey, error = ?e, "Batch insert failed");
            self.rollback_borrowed_accounts(accounts, processed_count);
            return Err(e);
        }
        processed_count += 1;
    }

    if let Err(e) = txn.commit() {
        error!(
            processed_count = processed_count,
            error = ?e,
            "Batch insert commit failed"
        );
        self.rollback_borrowed_accounts(accounts, processed_count);
        return Err(e.into());
    }
    Ok(())
}
```

**Changes**:
- Line 139: Extracted `pubkey` and `error` to fields
- Message constant: "Batch insert failed" (no pubkey interpolation)
- Line 149: Extracted `processed_count` and `error` to fields
- Message constant: "Batch insert commit failed" (no error interpolation)
- Enables: Filter errors by specific pubkey, aggregate by processed_count ranges

---

## Implementation Order

Process files in this exact order (considering dependencies and complexity):

1. **`src/error.rs`** - No changes needed (already optimal)
2. **`src/storage.rs`** - Simple validation error extraction
3. **`src/index.rs`** - Simple warning extraction
4. **`src/snapshot.rs`** - Multiple log points, moderate complexity
5. **`src/lib.rs`** - Critical operations, final polish

---

## Validation Steps

### 1. Syntax Check
```bash
cd magicblock-accounts-db
cargo check
```

### 2. Lint
```bash
make lint
```

### 3. Tests
```bash
cargo nextest run -p magicblock-accounts-db -j16
```

### 4. Log Inspection (Optional - not async, won't produce spans)
```bash
RUST_LOG=debug cargo test --package magicblock-accounts-db --lib 2>&1 | grep -E "info!|warn!|error!" | head -50
```

**Verify**:
- [ ] No parameters in message strings
- [ ] All context extracted to fields
- [ ] Constant message text (same every call)
- [ ] Correct format specifiers (%, ?, or bare)
- [ ] No manual spans (not expected - sync code)

---

## Summary

This plan covers **5 files with 13 log statements total**. The crate is primarily sync code (except thread spawning), so no `#[instrument]` attributes are needed. Focus is on:

1. Extracting all parameters from messages to fields
2. Making all messages constant and action-oriented
3. Using correct format specifiers

**Result**: Clean, queryable structured logging with zero spans (appropriate for sync code).
