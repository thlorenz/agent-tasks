# Structured Logging Implementation Plans

This directory contains deterministic implementation plans for migrating structured logging across the MagicBlock validator codebase.

## Files

### 01_samples.md
Sample outputs showing what properly structured logging looks like across different trace levels.

### 02_plan-general.md
**THE MASTER PATTERN** - One single pattern to follow consistently across all crates. All design decisions have been made. Read this first.

Contains:
- The single pattern: `#[instrument]` on all async functions + structured fields + error context
- Message rewriting rules
- Field naming and format specifications
- Span implementation rules
- Implementation procedure
- Checklist

### 02_plan-general-initial.md
Initial version of the general plan (archived reference).

### 03_plan-magicblock-chainlink.md
**TEMPLATE EXAMPLE** - Concrete implementation plan for magicblock-chainlink crate (19 files, 10 priority groups).

Use this as a reference for the level of detail and format to expect in crate-specific plans.

### 06_plan-magicblock-committor-service.md
**NEW** - Concrete implementation plan for magicblock-committor-service crate (14 files, 13 priority groups).

## Implementation Workflow

1. **Start with 02_plan-general.md**
   - Read the pattern
   - Understand the rules
   - Review the checklist

2. **Look at 03_plan-magicblock-chainlink.md**
   - See how the pattern is applied to a real crate
   - Understand the format and detail level
   - Review before/after examples

3. **Use 06_plan-magicblock-committor-service.md** (or similar)
   - Follow the priority order
   - Implement file by file
   - Use the before/after examples as a guide
   - Validate with the steps provided

## Key Principles

- **One Pattern**: All crates use the same pattern
- **Deterministic**: All design decisions are pre-made; no options to choose
- **Actionable**: Each file has specific line numbers and concrete examples
- **Testable**: Validation steps confirm correct implementation
- **Focused**: Each plan focuses on async functions with logging

## Pattern at a Glance

```rust
#[instrument(skip(self), fields(client_id = %self.client_id, pubkey = %pubkey))]
async fn subscribe(&mut self, pubkey: &Pubkey) -> Result<()> {
    info!("Initiating subscription");
    
    self.connect().await?;
    debug!("Connected");
    
    info!("Subscription established");
    Ok(())
}
```

**What you get**:
- **Message**: Constant text (enables aggregation)
- **Fields**: Context as key=value pairs (queryable)
- **Span**: Async function wrapped in trace span (automatic context inheritance)
- **Errors**: Automatically inherit span context

## Quick Start for New Crate Plan

1. Analyze the crate's architecture (5-6 major components)
2. Find all async functions with logging (use `rg "async fn" src/ -A 30 | rg "(info!|warn!|error!|debug!)`)
3. Organize files by priority/dependencies
4. For each file:
   - List current log statements with line numbers
   - Identify async functions
   - Provide implementation guidance
   - Show before/after examples
5. Create standard field naming table
6. Provide 2-3 representative before/after examples
7. List validation steps

## Helpful Commands

```bash
# Find all files with async functions
find src -name "*.rs" -type f -exec grep -l "async fn" {} \; | sort

# Find async functions with logging
cd <crate>
rg "async fn" src/ -l | while read f; do 
  echo "=== $f ==="; 
  rg -c "(info!|warn!|error!|debug!|trace!)" "$f"; 
done | grep -B1 -v "^0$"

# Count logging statements per file
rg "(info!|warn!|error!|debug!|trace!)" src/ --files-with-matches | wc -l

# Show logging in context
rg "(info!|warn!|error!|debug!|trace!)" src/ -B 3 -A 1
```

## Status

- [x] General pattern documented (02_plan-general.md)
- [x] Template example provided (03_plan-magicblock-chainlink.md - 19 files)
- [x] Magicblock-committor-service planned (06_plan-magicblock-committor-service.md - 14 files)
- [ ] Other crates (to be planned as needed)
