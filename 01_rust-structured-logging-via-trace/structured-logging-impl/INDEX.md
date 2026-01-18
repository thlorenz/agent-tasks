# Structured Logging Implementation Plans Index

Complete set of implementation plans for migrating MagicBlock codebase to structured logging with `tracing` and `#[instrument]`.

---

## Foundation Documents

1. **[01_samples.md](01_samples.md)** - Examples and Best Practices
   - What is structured logging
   - Before/after examples from actual codebase
   - Best practices and patterns
   - Benefits and use cases

2. **[02_plan-general.md](02_plan-general.md)** - Deterministic Migration Pattern
   - The single pattern: `#[instrument]` + structured fields + error context
   - Message rewriting rules
   - Field naming conventions and format specifiers
   - Span implementation rules (no manual spans)
   - Implementation procedure
   - Complete checklist

---

## Crate-Specific Implementation Plans

### Priority 1 (Core Infrastructure)

3. **[03_plan-magicblock-chainlink.md](03_plan-magicblock-chainlink.md)** - Proof of Concept
   - 19 files across 3 subsystems
   - Remote Account Provider (subscription/update logic)
   - Submux (multi-client coordination)
   - Chainlink (fetch/clone/delegation)
   - **Status**: Most complex, best example for other crates

### Priority 2 (High-Impact Systems)

4. **[06_plan-magicblock-committor-service.md](06_plan-magicblock-committor-service.md)** - Transaction Commitment
   - 14 files, 5 major components
   - Actor-based service architecture
   - Intent execution pipeline
   - Error handling with categorization

5. **[10_plan-magicblock-api.md](10_plan-magicblock-api.md)** - Magic Validator API
   - 6 files, 8+ async functions
   - Slot synchronization and committed block tracking
   - Dynamic field recording patterns
   - Complex spawned task handling

6. **[07_plan-magicblock-ledger.md](07_plan-magicblock-ledger.md)** - Ledger and Storage
   - 6 files (async focus), 4 files (deferred for pass 2)
   - Block processing and ledger operations
   - Data persistence and state management

### Priority 3 (Moderate Scope)

7. **[12_plan-magicblock-table-mania.md](12_plan-magicblock-table-mania.md)** - Lookup Table Management
   - 2 files, 12 async functions
   - Table lifecycle (creation, extension, deactivation, closure)
   - Dynamic field recording patterns
   - Batch operation tracking

8. **[05_plan-magicblock-aperture.md](05_plan-magicblock-aperture.md)** - RPC Server
   - 8 files, HTTP + WebSocket server
   - Event processor with multiple error conditions
   - Account and transaction request handling
   - Dynamic signature field recording

9. **[04_plan-magicblock-accounts.md](04_plan-magicblock-accounts.md)** - Account Management
   - 1 file (focused scope)
   - Scheduled commits processor
   - Intent result processing
   - Dynamic field recording with `Span::current().record()`

### Priority 4 (Smaller Scope)

10. **[09_plan-magicblock-processor.md](09_plan-magicblock-processor.md)** - Transaction Processing
    - 4 files, minimal logging
    - Executor and scheduler event loops
    - Error categorization

11. **[13_plan-magicblock-validator.md](13_plan-magicblock-validator.md)** - Main Validator
    - 2 files, very minimal scope
    - Shutdown signal handling
    - Startup context capture

12. **[08_plan-magicblock-rpc-client.md](08_plan-magicblock-rpc-client.md)** - RPC Client Library
    - 2 files, minimal logging (library)
    - Single async function with logging
    - One sync error handler

### Priority 5 (Minimal Scope)

13. **[11_plan-magicblock-account-cloner.md](11_plan-magicblock-account-cloner.md)** - Account Cloning
    - 1 file, 2 functions
    - Spawned task extraction pattern
    - Minimal refactoring needed

14. **[15_plan-magicblock-metrics.md](15_plan-magicblock-metrics.md)** - Metrics Server
    - 1 file, 2 async functions
    - HTTP server loop
    - Lightweight infrastructure crate

15. **[16_plan-magicblock-validator-admin.md](16_plan-magicblock-validator-admin.md)** - Admin Tools
    - 1 file, 2 async functions
    - Fee claiming loop
    - Simple refactoring

16. **[17_plan-test-kit.md](17_plan-test-kit.md)** - Test Infrastructure
    - 2 files, 5 async functions
    - Transaction helpers (execute, schedule, simulate, replay)
    - Error handling patterns

### No Changes Required

17. **[18_plan-magicblock-core.md](18_plan-magicblock-core.md)** - Core Infrastructure
    - Logger initialization only (no app logging)
    - Channel/link definitions
    - No migration needed

18. **[14_plan-magicblock-accounts-db.md](14_plan-magicblock-accounts-db.md)** - Database Storage
    - All sync code + 1 thread spawn
    - Extract fields and make messages constant
    - No `#[instrument]` needed

---

## Implementation Statistics

| Crate | Files | Async Functions | Log Statements | Complexity |
|-------|-------|-----------------|----------------|-----------|
| magicblock-chainlink | 19 | 30+ | 80+ | High |
| magicblock-committor-service | 14 | 25+ | 60+ | High |
| magicblock-api | 6 | 8+ | 20+ | Medium |
| magicblock-ledger | 6 | 2 | 10+ | Low-Medium |
| magicblock-table-mania | 2 | 12 | 25+ | Medium |
| magicblock-aperture | 8 | 8+ | 20+ | Medium |
| magicblock-accounts | 1 | 4 | 10 | Low |
| magicblock-processor | 4 | 2 | 7 | Low |
| magicblock-validator | 2 | 2 | 8 | Low |
| magicblock-rpc-client | 2 | 1 | 1 | Low |
| magicblock-account-cloner | 1 | 2 | 5 | Low |
| magicblock-metrics | 1 | 2 | 7 | Low |
| magicblock-validator-admin | 1 | 2 | 6 | Low |
| test-kit | 2 | 5 | 3 | Low |
| magicblock-accounts-db | 5 | 0 | 13 | Very Low |
| magicblock-core | 9 | 0 | 0 | N/A |
| **Total** | **97** | **125+** | **280+** | - |

---

## How to Use These Plans

### Sequential Implementation Path

**Recommended order** (respects dependencies and complexity):

1. **Phase 1: Foundation**
   - Start with `magicblock-chainlink` (most comprehensive, best patterns)

2. **Phase 2: Core Systems**
   - `magicblock-committor-service` (actor pattern)
   - `magicblock-api` (dynamic fields)
   - `magicblock-ledger` (block processing)

3. **Phase 3: Supporting Systems**
   - `magicblock-table-mania` (lookup tables)
   - `magicblock-aperture` (RPC server)
   - `magicblock-accounts` (account processing)

4. **Phase 4: Utilities**
   - `magicblock-processor` (executor/scheduler)
   - `magicblock-validator` (main validator)
   - `magicblock-rpc-client` (client library)
   - `magicblock-account-cloner` (account cloning)
   - `magicblock-metrics` (metrics server)
   - `magicblock-validator-admin` (admin tools)
   - `test-kit` (test helpers)

5. **Phase 5: Database Layer**
   - `magicblock-accounts-db` (sync-only, simple)

6. **Skip**
   - `magicblock-core` (infrastructure only)

### Per-Plan Structure

Each crate-specific plan includes:

- **Overview**: File count, async function count, complexity assessment
- **Standard Fields**: Domain-specific field naming and format specifications
- **Implementation Order**: Prioritized list respecting dependencies
- **File-by-File Breakdown**: Per-file implementation instructions
- **Concrete Examples**: 2-4 before/after code examples
- **Validation Steps**: Build, lint, test, and log verification commands
- **Implementation Checklist**: Pre/during/post tasks

### Using a Plan

1. Read the overview to understand scope
2. Review the standard fields table for your domain
3. Follow the implementation order
4. Use file-by-file instructions as a checklist
5. Reference before/after examples for pattern guidance
6. Run validation steps after completion

---

## Key Patterns Across All Plans

### The Single Pattern (From 02_plan-general.md)

```rust
#[instrument(skip(self), fields(client_id = %self.client_id, pubkey = %pubkey))]
async fn operation(&mut self, pubkey: &Pubkey) -> Result<()> {
    info!("Starting operation");
    
    match self.do_work().await {
        Ok(_) => info!("Operation succeeded"),
        Err(e) => error!(error = ?e, "Operation failed"),
    }
    
    Ok(())
}
```

**Components**:
- `#[instrument]`: Wraps entire async function with span
- `skip(self)`: Prevents circular data in span fields
- `fields(...)`: Explicit context fields captured
- Constant messages: Same text every time
- Structured fields: Parameters extracted from messages
- Error context: Errors inherit span context automatically

### No Manual Spans

Every plan specifies: **Do NOT create manual spans**. Use `#[instrument]` only.

### Deterministic Field Naming

All plans follow naming conventions from 02_plan-general.md:
- `<noun>_id` for identifiers
- `<noun>_count` for counts
- `<noun>_ms` for milliseconds
- `error`, `reason`, `status` for enums/descriptions
- Bare values for primitives (u64, i32, bool)

---

## Validation Across All Plans

Each plan includes validation steps:

```bash
# Syntax check
cargo check --package <crate>

# Linting
make lint

# Testing
cargo nextest run --package <crate>

# Log verification
RUST_LOG=debug cargo test --package <crate> 2>&1 | head -50
```

Expected output characteristics:
- ✅ Constant log messages (same text each time)
- ✅ Fields appear with correct values
- ✅ No duplicate information (in message AND field)
- ✅ Span nesting visible in indentation
- ✅ Zero manual spans

---

## Questions During Implementation?

Refer back to:
- **General principles**: 02_plan-general.md
- **Pattern examples**: 01_samples.md
- **Your crate's specific guidance**: The numbered plan for that crate

All ambiguity has been removed. Each plan is deterministic and actionable.
