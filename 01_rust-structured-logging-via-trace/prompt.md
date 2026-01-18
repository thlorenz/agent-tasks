Now inside <current-dir>/structured-logging-impl/ you find the following plans each containing clear
steps to implement structured logging in it.

- 03_plan-magicblock-chainlink.md (WE ALREADY DID THIS ONE, SO SKIP IT)
- 04_plan-magicblock-accounts.md (WE ALREADY DID THIS ONE, SO SKIP IT)
- 05_plan-magicblock-aperture.md (WE ALREADY DID THIS ONE, SO SKIP IT)
- 06_plan-magicblock-committor-service.md (WE ALREADY DID THIS ONE, SO SKIP IT)
- 07_plan-magicblock-ledger.md (WE ALREADY DID THIS ONE, SO SKIP IT)
- 08_plan-magicblock-rpc-client.md (WE ALREADY DID THIS ONE, SO SKIP IT)
- 09_plan-magicblock-processor.md (WE ALREADY DID THIS ONE, SO SKIP IT)
- 10_plan-magicblock-api.md (WE ALREADY DID THIS ONE, SO SKIP IT)
- 11_plan-magicblock-account-cloner.md (WE ALREADY DID THIS ONE, SO SKIP IT)
- 12_plan-magicblock-table-mania.md (WE ALREADY DID THIS ONE, SO SKIP IT)
- 13_plan-magicblock-validator.md (WE ALREADY DID THIS ONE, SO SKIP IT)
- 14_plan-magicblock-accounts-db.md (WE ALREADY DID THIS ONE, SO SKIP IT)
- 15_plan-magicblock-metrics.md (WE ALREADY DID THIS ONE, SO SKIP IT)
- 16_plan-magicblock-validator-admin.md (WE ALREADY DID THIS ONE, SO SKIP IT)
- 17_plan-test-kit.md (WE ALREADY DID THIS ONE, SO SKIP IT)
- 18_plan-magicblock-core.md

For each plan handoff to a subagent to do the following:

## 1. Prepare and Checkout a new branch

1. `git checkout -b thlorenz/structured-logging-<crate-name> master`, for instance `git checkout -b thlorenz/structured-logging-magicblock-chainlink master`

## 2. Implement structured logging as per the plan

0. First check that all tests are initializing the logger properly, otherwise handoff to a
   subagent to fix that first as per the instructions in <current-dir>/follow-test-init.md.
1. Read the plan file.
2. Implement the structured logging as per the steps in the plan file and verify its
   correctness as explained in the plan file.
3. Recommend a commit message following this pattern: "chore: apply structured logging to
   <crate-name>".
4. Wait for me to stage the changed files and perform my own checks
5. Once I confirm via `ok` commit the changes with the recommended commit message. YOU ARE
   ALLOWED TO COMMIT IN THIS CASE BUT DO NOT ADD ANYTHING TO THE STAGING AREA YOURSELF.
5. After committing, report back to me that the changes have been committed and wait for me to
   type `ok` to continue to the final step of creating the pull request.

### Important Notes

DO NOT ADAPT LOG STATEMENTS WITH HARDCODED NUMBERS OR LABELS, especially in test code, unless.
For instance do not change:

```rust
debug!("Sending update 3");
```
to
```rust
debug!(update_number = 3, "Sending update");
```
Or
```rust
info!("1. Initially the account does not exist");
```
to
```rust
info!(step = 1, "Initially the account does not exist");
```

DO NOT ADAPT MEASUREMENTS OR METRICS LOG STATEMENTS.

For instance the `info` statement in this code block should not be changed:

```rust
let mut measure = Measure::start("compaction");
self.db.column::<C>().compact_range(from, to);
measure.stop();

info!("Compaction of column {} took: {}", C::NAME, measure);
```

Double check that all files are modified, ALL LOGS IN THE CRATE SHOULD BE STRUCTURED AND USING SPANS.
Some files may not be directly mentioned in the plan but if they belong to the crate and
contain logging statements they should be modified as well.
You should grep through all files in all folders and subfolders of the crate to find any
logging statements you might have missed.

NOTE: that `Grep tracing::(info|debug|warn|error|trace` does NOT catch `info!` or similar macros
without the `tracing::` prefix.

You can also run the following command to find log statements you may have missed:

```sh
RUST_LOG=trace cargo nextest run -p <crate-name> --no-capture
```

There are places where we collect a list of pubkeys and stringify them for logging. In such
cases, you need to make sure to keep the behavior since otherwise we'll log the unreadable vec
representation of each pubkey instead of the base58 representation.

ALL LOG STATEMENTS IN THE CRATE MUST BE MODIFIED TO USE STRUCTURED LOGGING.

So before starting analyse this thread @T-019bc8d5-e486-77db-a33e-7482da8fe175 carefully and avoid making the same mistakes that you
made in there.

DO NOT RUN MULTIPLE AGENTS IN PARALLEL.

USE A SEPARATE SUBAGENT FOR EACH PLAN.

## 3. Create Pull Request

1. First you create a body that looks as follows:

```
## Summary

Adapting crate <crate-name> to use structured logging.

## Description

This PR implements structured logging for the crate <crate-name> as per the plan included below
which was derived from a _master_ plan with detailed structured logging practices in order to
beconsistent other crates in the project.

<details>
<summary>
Plan file: <plan-file-name>
</summary>
<inline the content of the plan file>
</details>
```

Save that body to `<current-dir>/pr-body.md`

2. Then create the pull request with the following command:

```sh
gh pr create --title "chore: apply structured logging to <crate-name>" --body-file <current-dir>/pr-body.md --base master
```

NOTE the PR number and URL and add the URL to `<current-dir>/prs.md` as follows:

```markdown
- [<crate-name>-<pr-number>](<pr-url>)
```

After creating the pull request report back to me with the URL of the created pull request and
wait for me to type `ok` to continue with the next plan in a new subagent.
