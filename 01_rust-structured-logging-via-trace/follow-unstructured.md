I still see lots of unstructured logs from this crate.

Handoff to a subagent and do the following to identify them:

1. Run the following commands (substitute <crate-name> accordingly):

Run application:

```sh
RUST_LOG=warn,<crate_name>=trace cargo run
```

Example:

```sh
RUST_LOG=warn,magicblock_ledger=trace cargo run
```

Run tests:

```sh
RUST_LOG=warn,<crate_name>=trace cargo test --package <crate-name> --no-capture
```

Example:

```sh
RUST_LOG=warn,magicblock_ledger=trace cargo nextest run -p magicblock-ledger --no-capture
```

The subagent should analyze the output of the command and identify all unstructured log
statements.

Then handoff to another subagent to do the following:

2. Provide the list of unstructured log statements found in step 1

The subagent should then find the source code locations of the unstructured log statements, and
fix them.
