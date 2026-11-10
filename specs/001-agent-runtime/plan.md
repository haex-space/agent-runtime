# Implementation Plan: Extensible Agent Runtime

**Branch**: `001-agent-runtime` | **Date**: 2026-11-10 | **Spec**: [spec.md](spec.md)

**Input**: Feature specification from `specs/001-agent-runtime/spec.md`

## Summary

Build an embeddable Rust agent runtime with a provider-neutral tool loop, declarative and version-pinned harness packages, bounded parent-child task orchestration, and deterministic fixture-based evaluation. A host application owns the async runtime, tool implementations, credentials, authorization, and approval decisions. Harness packages can shape behavior but cannot grant authority.

## Technical Context

**Language/Version**: Rust 2024 edition; MSRV 1.85 or newer

**Primary Dependencies**: Tokio for async execution and task tracking; `tokio-util` cancellation tokens; Serde and `serde_json` for provider-neutral data and run records; TOML for human-edited harness manifests; `async-trait` for object-safe asynchronous provider and tool interfaces; `reqwest` for the streaming HTTP provider adapter; optional `llama-cpp-2` bindings for in-process GGUF inference. Exact versions are selected at implementation start.

**Storage**: No database required by the runtime core. Provide an in-memory run store for tests and a host-pluggable event/run-store interface. Package content is cached locally by immutable identity and digest.

**Testing**: `cargo test --workspace` for unit, contract, and integration tests; deterministic fixture evaluations; separate Criterion or equivalent benchmarks for runtime overhead.

**Target Platform**: Linux, macOS, and Windows desktop/server processes.

**Project Type**: Cargo workspace with library crates and a CLI for configuration validation and evaluation.

**Performance Goals**: Runtime scheduling overhead under 20 ms p95 for a fixture-backed no-op model/tool cycle, excluding model and provider time. Native inference, model loading, and token generation MUST run on dedicated blocking workers, never an async executor thread. Cancellation stops admission of new descendant work promptly and joins or reports all descendants before the parent reaches a terminal state.

**Constraints**: Host owns the Tokio runtime and supplies credentials, tools, permissions, approvals, and the run's provider/model. V1 child tasks inherit their parent's provider/model and cannot override it. V1 packages are declarative and never execute code during resolution or activation. Parent-child authority is monotonic: a child cannot have more access than its parent. Event payloads redact prompt and tool data by default. The optional in-process native inference adapter may terminate the host process on a native backend fault; hosts choose this mode explicitly.

**Scale/Scope**: One root task tree per run in one process. Defaults: at most 4 concurrently running children, depth 3, and 32 total tasks in a tree. Hosts may configure different finite per-run ceilings; package defaults cannot exceed them. V1 includes an OpenAI-compatible streaming HTTP adapter usable with local llama.cpp servers and compatible hosted endpoints, plus an optional in-process llama.cpp adapter for GGUF models; custom providers implement the host provider contract. Native vendor SDK adapters, multimodal inputs, distributed workers, a hosted package registry, and executable plugins are out of scope for v1.

## Constitution Check

No `.specify/memory/constitution.md` has been established for this new project. For this initial plan, the feature specification supplies the design gates:

- **Least authority**: PASS — the host owns authorization, approval, credentials, and final capability grants; packages and child agents cannot expand them.
- **Bounded work and cleanup**: PASS — every task has limits, cancellation, a visible terminal state, and parent-to-child cancellation propagation.
- **Reproducibility and provenance**: PASS — package revisions and digests are recorded; fixture evaluations avoid live provider dependencies.
- **Host independence**: PASS — the library does not own a UI, model credential store, or Tokio runtime.
- **Safe package loading**: PASS — v1 loads declarative data only; no package install hooks or executable plugin code run.

No gate failure requires a waiver.

**Post-design recheck**: PASS — the contracts keep executable tool authority with the host, the data model bounds child work by the run budget, and package provenance plus redacted events support reproducibility without making user data part of default telemetry.

## Project Structure

### Documentation (this feature)

```text
specs/001-agent-runtime/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── runtime-api.md
│   ├── provider-tool.md
│   ├── harness-packages.md
│   └── task-events-eval.md
└── tasks.md              # created by speckit-tasks
```

### Source Code (repository root)

```text
crates/
├── agent-runtime/        # public facade and host integration surface
├── agent-runtime-core/   # task state machine, loop, policies, event contracts
├── agent-runtime-providers/ # HTTP and optional in-process local inference adapters
├── agent-runtime-harness/  # package parsing, pinning, composition, validation
├── agent-runtime-eval/   # fixtures, evaluation runner, reports and replay
└── agent-runtime-cli/    # validate, inspect and evaluate commands

tests/
├── contract/             # public provider, tool and host-policy contracts
├── integration/          # task loops, delegation, cancellation and package composition
└── fixtures/             # deterministic model, tool and package fixtures
```

**Structure Decision**: Use a Cargo workspace to separate the stable runtime contracts from package resolution, provider adapters, evaluation, and the CLI. The initial implementation should avoid creating a crate for every internal module; split only at boundaries with distinct consumers or dependency needs.

## Delivery Order

1. Establish the runtime contracts, fixture provider/tool, task lifecycle, limits, cancellation, and host policy boundary.
2. Add the OpenAI-compatible HTTP adapter, optional in-process llama.cpp adapter, and tool execution integration without moving host authorization into the model loop.
3. Add declarative harness package resolution, provenance, composition, and validation.
4. Add child task orchestration, parallel limits, inherited grants, cancellation propagation, and run inspection.
5. Add evaluation and replay workflows, then the CLI packaging and host integration examples.

The data model and public contracts below are the planning baseline. `tasks.md` will break this order into implementation-ready work.
