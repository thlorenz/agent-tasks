# Magicblock-Account-Cloner Structured Logging Implementation Plan

Based on the deterministic pattern from `02_plan-general.md`, this document provides a concrete, step-by-step implementation guide specific to the `magicblock-account-cloner` crate.

---

## Overview

The `magicblock-account-cloner` crate is a **lightweight account cloning service** that handles:
- Regular account cloning with delegation tracking
- Program cloning with BPF Loader V1/V4 support
- Transaction scheduling and lookup table preparation
- Error handling and transaction diagnostics

**Total files to update**: **3 files** (very focused scope)
**Async functions with logging**: **6 functions**
**Log statements to update**: **5 statements**

### Crate Characteristics

- **Small, focused scope**: Only 4 source files
- **Minimal logging**: 5 total log statements (2 debug, 2 error, 1 trace)
- **Simple patterns**: Most logging is single-statement per function
- **Heavy async trait impl**: Core functionality in trait impl (Cloner)
- **No manual spans**: Clean codebase, no existing span management

---

## File Analysis

### File 1: `src/lib.rs` - Main Implementation

**Category**: Core cloner implementation with trait impl

**Async functions with logging**:

1. **`send_transaction(&self, tx: Transaction)` - Lines 78-85**
   - NO logging
   - Result: Skip this function
   
2. **`maybe_prepare_lookup_tables(&self, pubkey, owner)` - Lines 319-345 (contains spawned async task)**
   - Contains spawned tokio task with async closure
   - Logging: Lines 333-340
   - Two log statements: trace! (line 333) and error! (line 339)
   - Current patterns:
     ```rust
     trace!("Reserving lookup keys for {pubkey} took {:?}", initiated.elapsed());
     error!("Failed to reserve lookup keys for {pubkey}: {err:?}");
     ```

3. **`map_committor_request_result(&res, committor)` - Lines 347-389 (async fn)**
   - NO logging in this function
   - Complex error mapping
   - Result: Skip this function

4. **`clone_account(&self, request)` - Lines 394-413 (trait impl, async)**
   - NO logging
   - Delegates to other methods
   - Result: Skip this function

5. **`clone_program(&self, program)` - Lines 415-444 (trait impl, async)**
   - NO logging
   - Calls `try_transaction_to_clone_program()` which has logging
   - Result: Skip this function (logging is in sync helper)

**Sync functions with logging**:

1. **`try_transaction_to_clone_program(&self, program, blockhash)` - Lines 178-317**
   - SYNC function (not async)
   - Logging: Lines 191, 230-233, 236-239
   - Three debug statements
   - Current patterns:
     ```rust
     debug!("Loading V1 program {}", program.program_id);
     debug!("Program {} is retracted on chain, won't retract it...", program.program_id);
     debug!("Deploying program with V4 loader {}", program.program_id);
     ```
   - **NOTE**: Per the plan, sync functions do NOT get spans

---

### File 2: `src/account_cloner.rs` - Error Mapping Utilities

**Async functions with logging**:

1. **`map_committor_request_result<T, CC>(&res, &intent_committor)` - Lines 24-68**
   - NO logging
   - Generic utility for error mapping
   - Result: Skip this function

**Result**: No changes needed for this file.

---

### File 3: `src/util.rs` - Transaction Utilities

**Async functions with logging**:

1. **`get_tx_diagnostics<C>(&sig, &committor)` - Lines 7-18**
   - NO logging
   - Simple utility to fetch transaction diagnostics
   - Result: Skip this function

**Result**: No changes needed for this file.

---

### File 4: `src/bpf_loader_v1.rs` - BPF Program Handling

**Async functions with logging**: None

**Sync functions**: All sync utility functions

**Result**: No changes needed for this file.

---

## Functions Requiring Changes

Only **1 async function** requires `#[instrument]`:

### Function: `maybe_prepare_lookup_tables` (indirectly - the spawned task)

**Issue**: This function spawns a tokio task with an async closure. The async closure has logging but no span.

**Current code (lines 319-345)**:

```rust
fn maybe_prepare_lookup_tables(&self, pubkey: Pubkey, owner: Pubkey) {
    // Allow the committer service to reserve pubkeys in lookup tables
    // that could be needed when we commit this account
    if let Some(committor) = self.changeset_committor.as_ref() {
        if self.config.prepare_lookup_tables {
            let committor = committor.clone();
            tokio::spawn(async move {
                match Self::map_committor_request_result(
                    committor.reserve_pubkeys_for_committee(pubkey, owner),
                    &committor,
                )
                .await
                {
                    Ok(initiated) => {
                        trace!(
                            "Reserving lookup keys for {pubkey} took {:?}",
                            initiated.elapsed()
                        );
                    }
                    Err(err) => {
                        error!("Failed to reserve lookup keys for {pubkey}: {err:?}");
                    }
                };
            });
        }
    }
}
```

**Problem**: The closure doesn't have a span. We need to extract it to a proper async function with `#[instrument]`.

**Solution**: Create a helper async function `reserve_lookup_tables` with `#[instrument]` and call it from the spawn.

---

## Standard Field Naming for Magicblock-Account-Cloner

| Field | Type | Format | Use Case |
|-------|------|--------|----------|
| `pubkey` | Pubkey | `%` | Account being cloned |
| `program_id` | Pubkey | `%` | Program being cloned |
| `owner` | Pubkey | `%` | Account owner |
| `error` | Error | `?` | Error details |
| `duration_ms` | u64 | bare | Elapsed time in milliseconds |
| `slot` | u64 | bare | Slot number |
| `action` | &str | bare | Operation (e.g., "reserve_lookup_tables") |

---

## Before/After Examples

### Example 1: Spawned Task with Logging - `maybe_prepare_lookup_tables()`

**Before**:
```rust
fn maybe_prepare_lookup_tables(&self, pubkey: Pubkey, owner: Pubkey) {
    if let Some(committor) = self.changeset_committor.as_ref() {
        if self.config.prepare_lookup_tables {
            let committor = committor.clone();
            tokio::spawn(async move {
                match Self::map_committor_request_result(
                    committor.reserve_pubkeys_for_committee(pubkey, owner),
                    &committor,
                )
                .await
                {
                    Ok(initiated) => {
                        trace!(
                            "Reserving lookup keys for {pubkey} took {:?}",
                            initiated.elapsed()
                        );
                    }
                    Err(err) => {
                        error!("Failed to reserve lookup keys for {pubkey}: {err:?}");
                    }
                };
            });
        }
    }
}
```

**After**:
```rust
#[instrument(skip(committor), fields(pubkey = %pubkey, owner = %owner))]
async fn reserve_lookup_tables(
    &self,
    pubkey: Pubkey,
    owner: Pubkey,
    committor: Arc<CommittorService>,
) {
    match Self::map_committor_request_result(
        committor.reserve_pubkeys_for_committee(pubkey, owner),
        &committor,
    )
    .await
    {
        Ok(initiated) => {
            trace!(
                duration_ms = initiated.elapsed().as_millis() as u64,
                "Lookup table reservation completed"
            );
        }
        Err(err) => {
            error!(error = ?err, "Failed to reserve lookup tables");
        }
    };
}

fn maybe_prepare_lookup_tables(&self, pubkey: Pubkey, owner: Pubkey) {
    if let Some(committor) = self.changeset_committor.as_ref() {
        if self.config.prepare_lookup_tables {
            let committor = committor.clone();
            // Detach the spawned task - it will still run independently
            tokio::spawn(async move {
                self.reserve_lookup_tables(pubkey, owner, committor).await;
            });
        }
    }
}
```

**Wait - Issue**: The spawned closure can't call `&self` on a moved value. Let me revise:

**Correct After**:
```rust
#[instrument(fields(pubkey = %pubkey, owner = %owner))]
async fn reserve_lookup_tables(
    pubkey: Pubkey,
    owner: Pubkey,
    committor: Arc<CommittorService>,
) {
    match ChainlinkCloner::map_committor_request_result(
        committor.reserve_pubkeys_for_committee(pubkey, owner),
        &committor,
    )
    .await
    {
        Ok(initiated) => {
            trace!(
                duration_ms = initiated.elapsed().as_millis() as u64,
                "Lookup table reservation completed"
            );
        }
        Err(err) => {
            error!(error = ?err, "Failed to reserve lookup tables");
        }
    };
}

fn maybe_prepare_lookup_tables(&self, pubkey: Pubkey, owner: Pubkey) {
    if let Some(committor) = self.changeset_committor.as_ref() {
        if self.config.prepare_lookup_tables {
            let committor = committor.clone();
            tokio::spawn(reserve_lookup_tables(pubkey, owner, committor));
        }
    }
}
```

### Example 2: Debug Logging in Sync Function (No Span) - `try_transaction_to_clone_program()`

**Before**:
```rust
fn try_transaction_to_clone_program(
    &self,
    program: LoadedProgram,
    recent_blockhash: Hash,
) -> ClonerResult<Option<Transaction>> {
    use RemoteProgramLoader::*;
    match program.loader {
        V1 => {
            debug!("Loading V1 program {}", program.program_id);
            // ... rest of code ...
        }
        _ => {
            let program_id = program.program_id;
            
            if matches!(program.loader_status, LoaderV4Status::Retracted) {
                debug!(
                    "Program {} is retracted on chain, won't retract it. When it is deployed on chain we deploy the new version.",
                    program.program_id
                );
                return Ok(None);
            }
            debug!(
                "Deploying program with V4 loader {}",
                program.program_id
            );
            // ... rest of code ...
        }
    }
}
```

**After** (no span, but extract fields):
```rust
fn try_transaction_to_clone_program(
    &self,
    program: LoadedProgram,
    recent_blockhash: Hash,
) -> ClonerResult<Option<Transaction>> {
    use RemoteProgramLoader::*;
    match program.loader {
        V1 => {
            debug!(program_id = %program.program_id, "Loading V1 program");
            // ... rest of code ...
        }
        _ => {
            let program_id = program.program_id;
            
            if matches!(program.loader_status, LoaderV4Status::Retracted) {
                debug!(program_id = %program.program_id, "Program is retracted on chain");
                return Ok(None);
            }
            debug!(program_id = %program.program_id, "Deploying program with V4 loader");
            // ... rest of code ...
        }
    }
}
```

**Why no `#[instrument]`?**: Sync functions do not get spans per the deterministic plan.

---

## Implementation Strategy

### Why This Crate Is Special

The `magicblock-account-cloner` crate is **minimal in logging requirements**:
- Only 1 async function has logging that needs span instrumentation
- The logging statements are simple and clear
- No complex error propagation patterns
- Most functions delegate work elsewhere

### Refactoring Required

**Extract spawned task function**:
```
Before:  tokio::spawn(async move { ... logging ... })
After:   Extract to #[instrument] async fn, spawn it
```

This is the **only significant refactoring** needed.

---

## Detailed Implementation Plan

### Step 1: Modify `src/lib.rs`

**Change 1: Extract async function from spawn**

1. Add new async function `reserve_lookup_tables` (lines ~320-345) before `maybe_prepare_lookup_tables`
2. Move logging from spawn closure into the new function
3. Add `#[instrument(fields(pubkey = %pubkey, owner = %owner))]`
4. Rewrite log statements:
   - Line 333: `trace!(duration_ms = initiated.elapsed().as_millis() as u64, "Lookup table reservation completed");`
   - Line 339: `error!(error = ?err, "Failed to reserve lookup tables");`
5. Update `maybe_prepare_lookup_tables` to call the new function via `tokio::spawn`

**Change 2: Update sync logging (no spans)**

1. Line 191: `debug!(program_id = %program.program_id, "Loading V1 program");`
2. Line 230-233: `debug!(program_id = %program.program_id, "Program is retracted on chain");`
3. Line 236-239: `debug!(program_id = %program.program_id, "Deploying program with V4 loader");`

### Step 2: No changes to other files

- `src/account_cloner.rs` - No logging
- `src/util.rs` - No logging
- `src/bpf_loader_v1.rs` - No logging

### Step 3: Verify

Run tests and linting to ensure changes compile and follow the pattern.

---

## Implementation Order

**Single file, simple execution**:

1. **`src/lib.rs`** - Extract spawned task function + update logging
   - Add `#[instrument]` to new `reserve_lookup_tables` function
   - Rewrite 2 log statements in spawned task
   - Rewrite 3 log statements in sync function (no spans)
   - Update `maybe_prepare_lookup_tables` to call extracted function

**Expected result**: All 5 log statements updated, 1 new async function with `#[instrument]`.

---

## Validation Steps

### 1. Syntax Check
```bash
cd magicblock-account-cloner
cargo check
```

### 2. Lint
```bash
make lint
```

### 3. Tests
```bash
cargo nextest run -p magicblock-account-cloner
```

### 4. Log Inspection
```bash
RUST_LOG=debug cargo test --package magicblock-account-cloner --lib 2>&1 | head -100
```

**Verify**:
- New `reserve_lookup_tables` span appears with constant message
- Fields captured: `pubkey`, `owner`, `duration_ms`
- Sync function logs have constant messages (no span)
- All parameters extracted to fields
- No duplicate info in message and field

---

## Checklist

### Pre-Implementation
- [x] Identified all async functions with logging (1 in spawned task)
- [x] Identified sync functions with logging (1 function, 3 statements)
- [x] Chose fields to include in #[instrument]
- [x] Reviewed field naming conventions
- [x] Planned refactoring (extract spawned task)

### During Implementation
- [ ] Extract `reserve_lookup_tables` async function
- [ ] Add `#[instrument(fields(pubkey = %pubkey, owner = %owner))]` to new function
- [ ] Update 2 log statements in spawned task
  - [ ] Convert trace! message to constant + field
  - [ ] Convert error! message to constant + field
- [ ] Update 3 log statements in sync function
  - [ ] Convert debug! statements to extract program_id field
- [ ] Update `maybe_prepare_lookup_tables` to call new function via spawn
- [ ] Verify no manual spans exist

### Post-Implementation
- [ ] `cargo check` passes
- [ ] `make lint` passes
- [ ] `cargo nextest run -p magicblock-account-cloner` passes
- [ ] All log messages are constant
- [ ] All context is in fields
- [ ] New function has correct span
- [ ] Sync function logs follow pattern (no span)

---

## Summary

This plan converts the minimal logging in `magicblock-account-cloner` from inline spawned tasks to proper instrumented async functions. The crate is small and focused, requiring:

1. **One new async function** with `#[instrument]` annotation
2. **Five log statement rewrites** (2 in spawned task, 3 in sync function)
3. **One refactoring** (extract spawned task closure to function)

Result: Clean, queryable, aggregatable structured logging for account and program cloning operations.
