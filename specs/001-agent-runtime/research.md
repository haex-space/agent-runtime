# Research: Extensible Agent Runtime

**Date**: 2026-11-10

This research resolves the technology and boundary choices in the initial implementation plan. The project has no separate constitution yet; the feature spec's safety and reproducibility requirements are treated as design gates.

## Async execution and subagent lifecycle

**Decision**: Use Tokio in the initial workspace. The library exposes async operations but never creates or owns a Tokio runtime. Track dynamic child tasks with a task set, use parent-derived cancellation tokens, and join children during graceful shutdown. Keep abort as a final fallback for a child that does not stop after cancellation.

**Rationale**: Child tasks are dynamic and finish independently; each needs a stable identity, its own limits and lifecycle, and a correlated result. Tokio's `JoinSet` supports dynamically spawned tasks and completion-order collection; `CancellationToken::child_token` supports parent-to-child cancellation. The embedding host remains in control of runtime lifecycle.

**Alternatives considered**: A runtime-neutral scheduler could avoid Tokio in the public dependency graph, but would add an abstraction layer before there is a second supported execution backend. A fixed `join!` group is less suitable for dynamic children. Hard abort alone is simpler but weakens cleanup and observability.

Sources: [Tokio JoinSet](https://docs.rs/tokio/latest/tokio/task/struct.JoinSet.html), [Tokio CancellationToken](https://docs.rs/tokio-util/latest/tokio_util/sync/struct.CancellationToken.html), [Tokio JoinSet shutdown](https://docs.rs/tokio/latest/tokio/task/struct.JoinSet.html#method.shutdown).

## Provider and tool contracts

**Decision**: Define a small provider-neutral request/response/event contract. Preserve provider-specific continuation metadata as opaque round-trippable data. Represent model tool proposals separately from host-side tool execution. The host validates proposals and applies authorization and approval before invoking a tool. Server-side provider tools remain provider capabilities; they are not treated as host tools.

**Rationale**: Providers differ in tool-call representation, stop reasons, streaming events, and continuation metadata. Some require provider-specific data to be returned unchanged on the next request. A strict lowest-common-denominator format risks breaking these flows; exposing provider SDK types from core would couple the runtime to vendors.

**Alternatives considered**: Vendor-native types in core preserve every feature but couple hosts and tests to each provider. Normalizing all responses to one minimal schema is simpler but can lose data needed for tool continuation. The chosen contract has a small common surface with an opaque extension field.

Sources: [OpenAI function calling](https://developers.openai.com/api/docs/guides/function-calling), [Anthropic tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview), [Gemini function calling](https://ai.google.dev/gemini-api/docs/function-calling), [Gemini tools](https://ai.google.dev/gemini-api/docs/tools).

**Initial adapter decision**: Ship a streaming OpenAI-compatible HTTP adapter for local inference servers such as llama.cpp and compatible hosted endpoints. Also ship an optional in-process llama.cpp adapter for local GGUF inference, as clarified for v1. Custom provider adapters remain possible through the provider contract.

**Rationale**: This provides a practical local-model path without embedding hardware-specific inference dependencies into the runtime and keeps the same event loop usable with remote compatible endpoints. Compatibility is not assumed to mean exact feature parity; the adapter advertises supported capabilities and reports unsupported options.

**Alternatives considered**: HTTP-only local inference has simpler deployment and process isolation but does not satisfy the in-process v1 requirement; ship vendor SDKs for every hosted provider (richer provider-specific features, but broader dependency and maintenance surface); support only user-written adapters (small core, but poor first-run usability).

Sources: [llama.cpp server API](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md), [Reqwest async HTTP client](https://docs.rs/reqwest/latest/reqwest/).

**In-process local adapter decision**: Use an optional `llama-cpp-2`-based adapter for GGUF models in v1, alongside the HTTP adapter. Keep the native dependency behind a Cargo feature so hosts that do not want native inference avoid its build and link requirements. Run model loading and generation on dedicated blocking workers.

**Rationale**: The project needs to run the user's current Qwen and Gemma GGUF models both with and without a separate inference process. `llama-cpp-2` provides Rust bindings to llama.cpp and feature-gated acceleration backends; upstream llama.cpp supports local GGUF models. The optional boundary keeps the generic agent runtime independent from native inference and allows host applications to choose HTTP for process isolation.

**Alternatives considered**: HTTP-only cannot meet the clarified in-process requirement. A pure Rust engine would reduce C++ FFI/build reliance, but current GGUF architecture coverage for the exact target models needs validation and may diverge from the existing llama.cpp model path. A native adapter has a material risk: an FFI or backend fault can terminate the host process, and acceleration backends impose platform-specific build dependencies.

Sources: [`llama-cpp-2` crate](https://docs.rs/llama-cpp-2/latest/llama_cpp_2/), [llama.cpp model formats](https://github.com/ggml-org/llama.cpp/blob/master/docs/models.md), [llama.cpp supported server backends](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md).

## Child model inheritance

**Decision**: The host selects the provider/model for the root run. In v1, every child inherits the provider/model and resolved harness configuration of its parent; child-level provider/model overrides are not supported. Effective capability and resource limits still intersect with the parent task.

**Rationale**: This keeps v1 orchestration predictable and avoids introducing model-selection policy and credential boundaries for each child. Hosts can still select the run's model before starting work.

**Alternatives considered**: Allow hosts to assign child profiles (enables routing work to faster or stronger models but adds profile policy and budget/credential checks); let the parent name arbitrary models (bypasses host selection and makes budget/credential policy harder to audit). Defer child-level routing until there is a concrete need.

## Rust async interface shape

**Decision**: Keep provider and tool implementations dynamically composable at host boundaries. Use object-safe async interfaces, with boxed futures or `async-trait`, rather than requiring hosts to monomorphize the complete runtime over every tool and provider type.

**Rationale**: Harness configuration and host applications assemble providers and tools at runtime. Native `async fn` in traits is not dyn-compatible in the general trait-object case; an explicit object-safe form makes the boundary predictable.

**Alternatives considered**: Generic-only interfaces can avoid dynamic dispatch and allocations but make heterogeneous, runtime-registered tools and adapters cumbersome. The small per-call dispatch cost is expected to be negligible relative to model and tool latency and will be measured.

Sources: [Rust Reference: dyn compatibility](https://doc.rust-lang.org/reference/items/traits.html#dyn-compatibility), [`async-trait`](https://docs.rs/async-trait/latest/async_trait/).

## Harness package format, composition, and trust

**Decision**: V1 packages are declarative manifests and content only. Use TOML for user-edited configuration, with package identity, version, runtime compatibility, dependencies, and immutable source revision plus content digest. Resolve in an explicit order, record provenance for each contributed setting, and reject conflicts unless the consuming configuration names an override. Packages may set non-authority task defaults within host-enforced ceilings and reference host-registered tool IDs, but cannot implement or grant tools.

**Rationale**: This carries spaex's useful composition, source pinning, provenance, and drift-inspection ideas into a runtime format without importing repository install semantics. User customization remains fast and shareable while package trust stays separate from host authorization.

**Alternatives considered**: Arbitrary in-process plugins provide ergonomic extension points but execute with the host process's authority and are not a sandbox. A subprocess/MCP boundary is more isolatable and interoperable but adds process lifecycle and protocol scope; defer executable extensions until there is a concrete need. WASM may offer a constrained execution boundary but adds host and capability design work. A hosted registry is also deferred; local packages and immutable source references suffice for the initial project.

Sources: [spaex README](https://github.com/haexmas/spaex/blob/main/README.md), [Pi README](https://github.com/earendil-works/pi), [Pi security and containerization notes](https://github.com/earendil-works/pi/blob/main/README.md#permissions--containerization), [MCP tools specification](https://github.com/modelcontextprotocol/modelcontextprotocol/blob/main/docs/specification/2026-07-28/server/tools.mdx).

## Task records, observability, and privacy

**Decision**: The canonical record is a runtime-owned, ordered event stream with run, task, and parent IDs, sequence numbers, timestamps, event kind, and redacted metadata. Persistence is behind a host-pluggable store. Export to OpenTelemetry may be an adapter, not the canonical schema. Full prompts, tool arguments, and results are omitted or redacted by default; a host may opt in to retained payloads.

**Rationale**: Parent-child inspection, cancellation, replay, and evaluation require correlated lifecycle events. Vendor-neutral traces support multiple hosts, while default redaction avoids turning observability into accidental data retention.

**Alternatives considered**: Persisting only final answers loses the explanation of tool and delegation behavior. Making OpenTelemetry the canonical storage format couples core persistence to an evolving semantic-convention set and its privacy decisions. Persisting full inputs by default improves replay fidelity but risks capturing credentials or personal data.

Sources: [OpenTelemetry GenAI semantic conventions](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/), [OpenTelemetry events](https://opentelemetry.io/docs/specs/semconv/general/events/).

## Evaluation and replay

**Decision**: Use scripted providers and fake tools for deterministic conformance and regression evaluations. Reports pin the harness package set and task-set revision and separate fixture replay from live-model inspection. Live provider behavior is not claimed to be bit-for-bit deterministic.

**Rationale**: Runtime safety, tool routing, event ordering, limits, and delegation can be validated without network services or model variance. Live runs remain useful for quality measurements but need their exact inputs, provider/model identity, and usage recorded.

**Alternatives considered**: Only live-model evaluation exercises production behavior but makes regressions noisy and expensive. Exact replay of all live responses requires storing sensitive payloads; fixture replay gives deterministic coverage without default full-payload retention.

Sources: Provider-specific tool and continuation behavior in the links above; [OpenTelemetry GenAI attributes](https://opentelemetry.io/docs/specs/semconv/registry/attributes/gen-ai/).
