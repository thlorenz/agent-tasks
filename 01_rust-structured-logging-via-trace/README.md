## Adapting a Codebase to Use Structured Logging Features provided by the tracing crate

### General Preparation

#### How to use Tracing for Structured Logging

I first asked the agent to provide me insight into how the _tracing_ crate can be used to
implement structured logging in a Rust codebase.
This resulted in the following document:
- [./structured-logging-impl/01_samples.md](./structured-logging-impl/01_samples.md)

#### Planning the Adaptation to Structured Logging in General

Then I asked the agent to help me plan how I could adapt a specific codebase to use structured
logging which resulted in the following document:
- [./structured-logging-impl/02_plan-general-initial.md](./structured-logging-impl/02_plan-general-initial.md)

However that one had multiple options/decision points in it so I told the agents which of the
given options I preferered and asked it to create a revised plan based on my preferences
This plan should be deterministic (without any further options/decision points) and another
agent should be able to follow it step by step to adapt the codebase.

- [./structured-logging-impl/02_plan-general.md](./structured-logging-impl/02_plan-general.md)

#### Creating Plans for Each Crate

Next I asked the agent to create a plan for each crate in the codebase based on the general
plan. This resulted in the following documents:

- [structured-logging-impl/03_plan-magicblock-chainlink.md](./structured-logging-impl/03_plan-magicblock-chainlink.md)
- [structured-logging-impl/04_plan-magicblock-accounts.md](./structured-logging-impl/04_plan-magicblock-accounts.md)
- [structured-logging-impl/05_plan-magicblock-aperture.md](./structured-logging-impl/05_plan-magicblock-aperture.md)
- [structured-logging-impl/06_plan-magicblock-committor-service.md](./structured-logging-impl/06_plan-magicblock-committor-service.md)
- [structured-logging-impl/07_plan-magicblock-ledger.md](./structured-logging-impl/07_plan-magicblock-ledger.md)
- [structured-logging-impl/08_plan-magicblock-rpc-client.md](./structured-logging-impl/08_plan-magicblock-rpc-client.md)
- [structured-logging-impl/09_plan-magicblock-processor.md](./structured-logging-impl/09_plan-magicblock-processor.md)
- [structured-logging-impl/10_plan-magicblock-api.md](./structured-logging-impl/10_plan-magicblock-api.md)
- [structured-logging-impl/11_plan-magicblock-account-cloner.md](./structured-logging-impl/11_plan-magicblock-account-cloner.md)
- [structured-logging-impl/12_plan-magicblock-table-mania.md](./structured-logging-impl/12_plan-magicblock-table-mania.md)
- [structured-logging-impl/13_plan-magicblock-validator.md](./structured-logging-impl/13_plan-magicblock-validator.md)
- [structured-logging-impl/14_plan-magicblock-accounts-db.md](./structured-logging-impl/14_plan-magicblock-accounts-db.md)
- [structured-logging-impl/15_plan-magicblock-metrics.md](./structured-logging-impl/15_plan-magicblock-metrics.md)
- [structured-logging-impl/16_plan-magicblock-validator-admin.md](./structured-logging-impl/16_plan-magicblock-validator-admin.md)
- [structured-logging-impl/17_plan-test-kit.md](./structured-logging-impl/17_plan-test-kit.md)
- [structured-logging-impl/18_plan-magicblock-core.md](./structured-logging-impl/18_plan-magicblock-core.md)

I specifically asked the agent to **not implement anything yet**. This also allowed me to
handoff each document to another subagent and **run them in parallel**. I expressed that in the
prompt as well.

## Applying the Plans to the Codebase

I then created a prompt very similar to what is now in [prompt.md](./prompt.md) and ran it.
I opted not to use worktrees and run each task in sequence as there were lots of things to
check and fix for each crate before a PR could be created.

### Follow Ups

As I went along I noticed that some tests were missing logger initialization and used the
following prompt to fix it so I could also verify the logging output:

- [follow-test-init.md](./follow-test-init.md)

I also noticed that the agent missed some logs which remained unstructured and asked it to
fix that with the following prompt:

- [follow-unstructured.md](./follow-unstructured.md)

I then incoporated the following into the main prompt:

- check for test logs by using the test-init prompt
- read the thread in which I discovered missing logs to learn from it and not repeat the same
mistake (you may have to remove that part since you don't have that thread)

### Handoff vs. Restart

Even though each crate was done in a separate agent via _handoff_, I noticed that overtime the
context window was growing and the cost of each step was increasing. Even though I used _rush_
mode with cheaper models.

So I decided to mark in the prompt which steps had already been done and restarted the agent
with the same prompt after about 2-3 crates. This way the context window was smaller and the
cost per step was lower.
