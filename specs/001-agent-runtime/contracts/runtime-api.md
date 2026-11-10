# Runtime API Contract

## Purpose

Define the host-facing behavior of the embeddable runtime. Concrete Rust type names may evolve during implementation; the behavioral contract is stable.

## Host responsibilities

The host supplies:

- A model provider implementation and credential access.
- Tool definitions and tool execution functions.
- The set of capabilities granted to the root task.
- The provider/model selected for the run's root task.
- Authorization and approval decisions for consequential actions.
- Optional event/run persistence and user interface.
- Runtime configuration and limits.

The runtime does not create an async runtime, acquire host credentials, or decide to expand host grants.

## Run lifecycle

1. The host creates a validated runtime configuration, provider, tools, policy, and limits.
2. The host starts a run with a request and an explicit root capability set.
3. The runtime emits ordered events, obtains provider responses, validates tool proposals, requests host authorization/approval when needed, and invokes only approved host tools.
4. The host may request cancellation through a run/task control handle; the runtime propagates it to descendants and reports final states.
5. The runtime returns a final result and stop reason, after active descendants have stopped or have a reported shutdown failure.
6. The host may inspect or persist the event stream and run record.

## Failure behavior

- Provider and tool failures are typed runtime outcomes with safe diagnostics; they are not represented as successful model answers.
- A denied proposal is returned to the model as a denial result and never invokes the tool.
- A limit or cancellation stops further work and produces a terminal task state.
- Configuration validation failure prevents activation; the last valid configuration remains available to the host.
- Event-store failure is surfaced to the host and cannot be interpreted as approval or authorization.

## Runtime invariants

- Child capability grants are no broader than the parent grants.
- In v1, every child inherits its parent's provider/model and cannot select an override.
- Package contributions cannot alter host authorization, approval, credentials, or guardrails.
- Every child task has one parent and a separate conversation state.
- Every task reaches an observable terminal state, including children stopped by parent cancellation.
- Defaults redact request and tool payload content in retained events.
