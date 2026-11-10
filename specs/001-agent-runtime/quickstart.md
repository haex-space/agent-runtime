# Quickstart Validation Guide

This guide describes the planned end-to-end checks after the first implementation is available. The current project contains planning artifacts only.

## Prerequisites

- Rust toolchain meeting the project's declared MSRV.
- Cargo.
- No model API key is needed for fixture-backed checks.

## Validate a harness configuration

```sh
cargo run -p agent-runtime-cli -- harness validate examples/harnesses/default.toml
```

Expected result: the configuration validates, reports its pinned packages and effective contributions, and makes no network or code-execution attempt during activation.

## Run deterministic fixtures

```sh
cargo test --workspace
cargo run -p agent-runtime-cli -- eval specs/001-agent-runtime/fixtures/core-eval.jsonl
```

Expected result: the multi-step tool call, denied consequential call, child task tree, bounded parallel execution, and cancellation cases produce the expected per-case outcomes. Repeating the fixture evaluation produces identical tool decisions and task-tree outcomes.

For local inference, run the same provider/tool scenario once through a local OpenAI-compatible llama.cpp server and once through the optional in-process llama.cpp provider. Both paths must preserve the runtime's tool authorization and task limits. The in-process backend is opt-in and built with the selected platform backend feature.

## Compare harness versions

```sh
cargo run -p agent-runtime-cli -- eval specs/001-agent-runtime/fixtures/core-eval.jsonl --harness examples/harnesses/default.toml --report target/eval-default.json
cargo run -p agent-runtime-cli -- eval specs/001-agent-runtime/fixtures/core-eval.jsonl --harness examples/harnesses/compact.toml --report target/eval-compact.json
cargo run -p agent-runtime-cli -- eval compare target/eval-default.json target/eval-compact.json
```

Expected result: the comparison identifies both configuration digests and shows per-case completion, tool decisions, stop reason, and usage where available. The fixture suite requires no external service.

## Inspect task lifecycle

Run the orchestration fixture and inspect the resulting report. Every task must show its ID, parent ID, state, stop reason, effective limits, and declared result. Cancelling the root must leave no active child task.

The command names and example paths are proposed contract surface; implementation may refine their spelling while preserving these outcomes.
