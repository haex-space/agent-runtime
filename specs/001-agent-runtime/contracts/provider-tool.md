# Provider and Tool Contracts

## Provider boundary

A provider accepts a request containing conversation state, generation controls, advertised host tools, and any opaque provider continuation data. It emits zero or more streaming events and a terminal outcome.

Common events include:

- Text delta.
- Tool-call proposal with provider call ID, host tool name, and raw arguments.
- Usage update when supplied by the provider.
- Provider-specific metadata that must survive a round trip when required for continuation.
- Terminal finish/stop reason or typed provider error.

The common contract does not assume every provider supports streaming, parallel tool proposals, structured output, or the same finish reasons. Capabilities are declared and unsupported options are rejected or explicitly degraded, never silently misrepresented.

The initial built-in adapters provide two local-model paths: a streaming OpenAI-compatible HTTP chat API for local servers or compatible hosted endpoints, and optional in-process GGUF inference through llama.cpp. They share the same provider contract. The in-process backend is opt-in because native faults can terminate the host process and acceleration builds vary by platform. Neither adapter claims every provider-specific API feature is portable; hosts may supply custom adapters for native APIs.

## Tool boundary

A host tool declares a stable ID, description, input contract, and effect classification. Its executor receives validated arguments plus task/run context and returns a bounded result or typed error.

Execution order:

1. Resolve the proposed tool ID against tools offered to that task.
2. Validate the proposal shape and arguments.
3. Apply host authorization and any required approval.
4. Invoke the host executor only when approved.
5. Return a bounded result and emit correlated events.

Provider-managed server tools are provider capabilities. They are not routed through this host-tool executor unless the provider explicitly returns them as a host-executable proposal.

## Approval

The host policy receives the task identity, tool effect, validated arguments, and request context allowed by host privacy policy. It returns allow, deny, or require-user-approval. The runtime must not bypass an approval denial, and a child inherits no stronger approval setting than its parent.
