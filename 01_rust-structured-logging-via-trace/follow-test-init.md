The crate we are currently working on has the following issue.

The tests do not initialize the logger. This can be done simply by adding the following code to each test module/test:

```rust
use test_kit::init_logger;
```

Then inside a common setup function or at the start of each test function call:
```rust
init_logger!();
```

Handoff to a subagent to fix this across all tests in the crate we are currently working on.
Particularly magicblock-accounts-db/src/tests.rs seem to miss logger init.
