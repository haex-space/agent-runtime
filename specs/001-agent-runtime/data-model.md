# Data Model: Extensible Agent Runtime

## Run

One execution started by a host for a user request.

- `run_id`: stable unique identifier.
- `root_task_id`: task at the top of the task tree.
- `configuration_id`: digest of the resolved harness configuration.
- `provider_id` and `model_id`: provider and model selected by the host.
- `started_at`, `finished_at`: run bounds.
- `status`: running, completed, failed, cancelled, or limit reached.
- `usage`: aggregate reported token/model usage where available.

**Rules**: IDs are unique within the host's chosen identifier domain. A completed run has a terminal root task. Run metadata excludes raw prompts and tool payloads by default.

## Agent Task

One bounded unit of model-directed work. Root tasks and subagents share the same lifecycle model.

- `task_id`, `run_id`: stable identities.
- `parent_task_id`: absent for a root task; otherwise the direct parent.
- `depth`: zero for a root and parent depth plus one for a child.
- `input`: explicit task request and the context the host chose to pass.
- `tool_grants`: effective subset of host-approved capabilities.
- `provider_id`, `model_id`: resolved provider and model used for this task; child tasks inherit both from their parent in v1.
- `limits`: turn, elapsed-time, child-count/depth, concurrency, and usage limits; child allocations are reserved from the parent's remaining budget.
- `state`: `queued`, `running`, `awaiting_approval`, `completed`, `failed`, `cancelled`, or `limit_reached`.
- `result`: declared child result or final root result when available.
- `stop_reason`: completion, provider/tool error, denied approval, cancellation, or specific limit.
- `started_at`, `finished_at`: lifecycle bounds.

**Relationships**: One run has one root task and zero or more descendants. A task can have multiple children. Each child has exactly one parent and an independent conversation state. Active parent and child work shares a finite run-level budget; child allocations cannot exceed the parent's remaining budget. In v1, each child inherits the provider and model of its parent and cannot override either.

**State transitions**:

```text
queued -> running
running -> awaiting_approval -> running
running -> completed | failed | cancelled | limit_reached
awaiting_approval -> cancelled | failed | limit_reached
```

Terminal tasks do not transition again. Parent cancellation requests cancellation for all active descendants; a parent records final state only after descendants have stopped or have been marked as failed to stop cleanly.

## Provider Exchange

One request/response boundary with a model provider.

- `exchange_id`, `task_id`, `sequence`: correlation and ordering.
- `request`: normalized messages, offered host tools, generation controls, and opaque provider continuation data.
- `events`: streaming text, tool proposals, usage, provider metadata, and terminal reason.
- `provider_payload`: opaque, serializable round-trip data owned by the provider adapter.
- `error`: provider failure with category and safe diagnostic context.

**Rules**: Unknown provider payload fields are preserved or explicitly rejected by the adapter; core normalization does not silently discard continuation metadata.

## Tool and Tool Proposal

The tool is a host-registered capability. A proposal is untrusted provider output requesting that the host invoke a tool.

- `tool_id`: stable host-defined identity.
- `input_schema`: input shape used for proposal validation.
- `effect`: host-defined risk/effect classification.
- `proposal_id`, `arguments`: model-provided request, validated before execution.
- `decision`: invalid, denied, approval required, approved, or executed.
- `result`: bounded tool output, redacted according to host policy.

**Rules**: The model cannot register tools, authorize itself, or execute a proposal directly. A child task's tool grants are an intersection of host grants and parent grants.

## Harness Package

A versioned declarative contribution to runtime behavior.

- `package_id`, `version`: stable identity and release version.
- `source`: local source or remote repository reference.
- `revision`: immutable source revision when remote.
- `content_digest`: integrity digest of the resolved package content.
- `runtime_compatibility`: supported runtime versions/capabilities.
- `dependencies`: other package IDs and pinned versions.
- `contributions`: prompt sections, routing hints, model defaults, non-authority task defaults, and references to host-registered tools.
- `priority`: explicit composition order when non-conflicting contributions have order-sensitive semantics.

**Rules**: Packages cannot declare executable install hooks, arbitrary host access, approval policy, credentials, or capabilities beyond those registered by the host. Task defaults may tighten but not exceed host ceilings. Conflicts require explicit override or fail validation.

## Harness Configuration

The saved composition of packages and user choices.

- `configuration_id`: digest of normalized package identities, order, and overrides.
- `packages`: pinned package IDs, versions, source revisions, and content digests.
- `overrides`: user-selected values with provenance.
- `resolved_behavior`: effective prompts, routing guidance, defaults, and referenced host tools.
- `validation`: compatibility and conflict results.

**Rules**: The resolved behavior is inspectable and reproducible. Host authorization and tool grants are applied separately and cannot be overridden here.

## Run Event

An ordered observation of a task or run lifecycle change.

- `run_id`, `task_id`, `parent_task_id`: scope and tree correlation.
- `sequence`: monotonically increasing sequence within the run.
- `timestamp`: event time.
- `kind`: task, provider, tool, approval, delegation, limit, cancellation, or completion event.
- `metadata`: redacted event-specific fields and payload references.
- `payload_digest`: optional integrity/replay reference when payload retention is enabled.

**Rules**: Event persistence is host-pluggable. Event emission failure is visible to the host; it must not silently convert a failed trace write into a successful tool authorization or execution decision.

## Evaluation Set and Evaluation Case

A revisioned set of requests and expected behavior used to compare harness configurations.

- `set_id`, `revision`: stable evaluation identity.
- `case_id`, `input`: unique case and user request.
- `provider_fixture`: scripted provider exchange sequence.
- `tool_fixtures`: registered tool inputs, outputs, and failures.
- `expected`: assertions over tool decisions, final outcome, task tree, and safety behavior.
- `scoring`: case-level pass/fail and optional metric definitions.

**Rules**: Fixture evaluation is deterministic. Live-model runs identify model and provider but are reported separately and do not promise exact replay.
