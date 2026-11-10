# Feature Specification: Extensible Agent Runtime

**Feature Branch**: `001-agent-runtime`

**Created**: 2026-11-10

**Status**: Draft

**Input**: User description: "Create an independent, extensible agent runtime inspired by Pi and spaex, with first-class subagent orchestration and user-configurable harness packages."

## Clarifications

### Session 2026-11-10

- Q: Sollen Nutzer-Harness-Pakete in der ersten Version nur Anweisungen, Routing und sichere Vorgaben konfigurieren, oder auch eigenen Erweiterungscode ausführen dürfen? → A: Nur deklarative Pakete in v1; ausführbare Erweiterungen sind ausgeschlossen.
- Q: Wie soll v1 lokale Modelle nutzen: über einen laufenden lokalen HTTP-Server, direkt im Runtime-Prozess eingebettet oder beides? → A: Beides in v1.
- Q: Sollen Subagents dasselbe Modell wie ihr Elternagent verwenden, oder darf der Host sie auf andere Modelle routen? → A: Subagents übernehmen in v1 Provider und Modell ihres Elternagents; abweichendes Modell-Routing pro Subagent ist ausgeschlossen.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Run an agent through a reusable runtime (Priority: P1)

A developer embeds the runtime in an application, selects a model provider, supplies tools and a user request, and receives the agent's final answer and a trace of the work. The runtime continues through tool calls until it completes, reaches a configured limit, is cancelled, or encounters an error.

**Why this priority**: This is the smallest useful product: a host application can use one consistent agent loop without owning its own orchestration implementation.

**Independent Test**: Use a deterministic fake model and two fake tools to complete a multi-step request, then verify the final result and ordered event trace.

**Acceptance Scenarios**:

1. **Given** a configured model and available tools, **When** the user submits a request requiring a tool, **Then** the runtime calls the tool, returns its result to the model, and reports the completed answer.
2. **Given** a request that exceeds a configured turn or resource limit, **When** the limit is reached, **Then** the runtime stops safely and reports why it stopped.
3. **Given** an active request, **When** the host cancels it, **Then** the runtime stops further model and tool work and reports cancellation.
4. **Given** a local model, **When** the host selects either local-server or in-process mode, **Then** the same tool and authorization behavior is available in both modes.

### User Story 2 - Customize and share agent behavior (Priority: P1)

A developer or advanced user composes a harness from reusable, versioned packages. They can adjust instructions, routing preferences, available tools, and task policies for their use case, validate the result, and share or restore a known version without changing runtime code.

**Why this priority**: Adaptability is a core reason to create a separate runtime, and it enables faster, reproducible experiments.

**Independent Test**: Compose two packages with non-conflicting contributions, override one configurable setting, validate the result, and load the same configuration in a clean runtime instance.

**Acceptance Scenarios**:

1. **Given** multiple compatible harness packages, **When** a user composes them, **Then** the runtime reports the effective instructions, tools, policies, and their sources.
2. **Given** two packages that conflict, **When** they are composed, **Then** the runtime applies an explicit precedence rule or reports a validation error; it never silently picks an arbitrary value.
3. **Given** an invalid or unavailable package, **When** a configuration is loaded, **Then** the runtime rejects or clearly disables that package and preserves the last valid configuration.
4. **Given** a valid configuration, **When** a user exports it, **Then** another installation can validate and load the same pinned package versions.

### User Story 3 - Delegate bounded work to subagents (Priority: P1)

An agent delegates a well-scoped part of a task to one or more subagents, optionally in parallel, and uses their results to finish the parent task. The user or host can inspect, cancel, or limit each child task.

**Why this priority**: Subagent orchestration is a required capability and should shape the runtime's task and permission model from the beginning.

**Independent Test**: A parent agent delegates two independent tasks to deterministic child agents, collects both results, and verifies their task states, limits, and event ordering.

**Acceptance Scenarios**:

1. **Given** an agent with delegation enabled, **When** it starts a child task, **Then** the child receives only the declared task context, tools, and resource limits.
2. **Given** multiple independent child tasks, **When** the parent requests parallel execution, **Then** they can progress independently and the parent receives a distinct result for each.
3. **Given** a child task that exceeds its limit or is cancelled, **When** it stops, **Then** its resources are released and its status and reason are visible to the parent.
4. **Given** a child task, **When** it requests an action outside the authority granted to its parent, **Then** the action is denied and cannot be enabled by the child's own harness configuration.
5. **Given** a parent task that is cancelled, **When** cancellation propagates, **Then** its active descendants stop and report their final states.
6. **Given** a parent task using a provider and model, **When** it starts a child task, **Then** the child uses the same provider and model.

### User Story 4 - Evaluate and reproduce harness changes (Priority: P2)

A developer compares harness versions on a fixed task set, reviews outcomes and resource use, and can replay a recorded run to understand regressions.

**Why this priority**: Customization is useful only if changes can be compared and their effects understood.

**Independent Test**: Run two harness versions against the same deterministic task set and verify that reports identify the exact configuration and show per-task outcomes.

**Acceptance Scenarios**:

1. **Given** a task set and two harness versions, **When** both are evaluated under the same conditions, **Then** a report compares completion, failures, tool use, latency, and resource limits per task.
2. **Given** a recorded run, **When** a developer inspects it, **Then** the report identifies the model/provider, harness packages and versions, tool definitions, and stop reason.
3. **Given** a recorded run with deterministic fixtures, **When** it is replayed, **Then** the same tool and orchestration decisions can be inspected without requiring live external services.

### Edge Cases

- A model provider becomes unavailable during a parent or child task.
- A tool returns malformed data, a timeout, or a partial result.
- A child agent attempts recursive delegation beyond the configured depth or task count.
- Parallel children complete in a different order, fail independently, or return mutually inconsistent results.
- A package is removed or its pinned source cannot be reached after a configuration has been saved.
- The parent reaches its resource limit while children are still active.
- An event sink or run store becomes unavailable while an agent is running.
- An agent proposes a consequential action for which approval is required, and approval is declined or times out.

## Requirements *(mandatory)*

### Functional Requirements

- **FR-001**: The runtime MUST provide a reusable agent loop that accepts a request, model interaction, available tools, and a bounded execution policy, and returns a final result with a stop reason.
- **FR-002**: The runtime MUST support multiple model providers through a common interaction contract and MUST expose provider errors without presenting them as successful answers.
- **FR-003**: The runtime MUST expose ordered lifecycle events for model requests, tool calls and results, delegation, approvals, cancellation, limits, and completion.
- **FR-004**: A harness configuration MUST be composable from independently versioned packages that contribute user instructions, routing preferences, tool availability, or non-authority task defaults within host-enforced limits.
- **FR-005**: The runtime MUST validate package identity, version, compatibility, dependencies, and conflicts before activation. V1 MUST NOT execute package-provided code or install hooks.
- **FR-006**: The runtime MUST report the effective configuration and the source package for each contributed or overridden setting.
- **FR-007**: Users MUST be able to save, export, import, pin, and restore a valid harness configuration without changing runtime code.
- **FR-008**: The runtime MUST support starting child agent tasks with explicit task descriptions, inherited context, selected tools, and independent resource limits.
- **FR-009**: The runtime MUST support both sequential delegation and bounded parallel child tasks, and MUST associate every child with its parent task.
- **FR-010**: A child task MUST NOT receive more authority than its parent and MUST NOT grant itself additional tools, data access, or approval rights through its own configuration.
- **FR-011**: The runtime MUST support cancellation of a task and propagation of parent cancellation to active descendants, with resource cleanup and an observable final state.
- **FR-012**: The runtime MUST enforce configurable limits for turns, elapsed time, child depth, number of child tasks, and model usage, and report which limit stopped a task. Child task limits MUST be allocated from the parent's remaining run budget so concurrent children cannot collectively exceed the host's configured run ceiling.
- **FR-013**: The runtime MUST preserve separate conversation state for parent and child tasks while allowing the parent to receive each child's declared result.
- **FR-014**: The runtime MUST allow a host application to apply approval rules to consequential tool actions and MUST prevent a declined action from executing.
- **FR-015**: The runtime MUST provide an evaluation workflow that compares pinned harness configurations on the same task set and reports outcomes and resource use per task.
- **FR-016**: Evaluation runs MUST support deterministic model and tool fixtures so core runtime behavior can be replayed without live provider or application services.
- **FR-017**: The runtime MUST allow applications to embed and control it without requiring a particular user interface.
- **FR-018**: The runtime MUST treat external package contents and tool results as untrusted input and MUST keep them from overriding host-level safety and authorization policy.
- **FR-019**: V1 MUST support running local models both through a local inference server and directly within the host process.
- **FR-020**: In v1, every child task MUST inherit the provider and model of its parent; child tasks MUST NOT override them.

### Key Entities

- **Agent Task**: One bounded unit of agent work with a parent relationship, conversation state, lifecycle, limits, and result.
- **Harness Package**: A named, versioned contribution to agent behavior, tools, routing, or policies, with declared compatibility and dependencies.
- **Harness Configuration**: An ordered composition of package versions and user overrides that resolves to one effective runtime behavior.
- **Tool**: A named capability with an input contract, execution result, and host-defined authorization behavior.
- **Provider**: A model service or local model that accepts agent messages and returns text or tool proposals.
- **Run Record**: An inspectable record of a task tree, configuration versions, events, outcomes, and resource use.
- **Evaluation Set**: A versioned collection of requests, fixtures, expected behavior, and scoring rules.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: A host application can complete a multi-step tool task through the runtime without implementing its own agent loop.
- **SC-002**: A user can change, validate, save, and restore harness behavior without changing runtime source code.
- **SC-003**: In the conformance evaluation, 100% of attempted child actions beyond the parent's granted authority are denied before execution.
- **SC-004**: A user can inspect the status and stop reason of every task in a parent-child task tree from its run record.
- **SC-005**: Replaying a fixture-based evaluation produces the same tool decisions and task-tree outcomes across repeated runs.
- **SC-006**: A configuration comparison identifies the exact package versions and per-task outcomes for every evaluated run.
- **SC-007**: The same local-model task can be run in both local connection modes with the same tool authorization and task-limit behavior.
- **SC-008**: 100% of child tasks use the same provider and model as their parent task.

## Assumptions

- The runtime is a reusable library first; an interactive shell or graphical interface may be a separate consumer.
- The initial release targets local and hosted text-based models; local models are available through both a separate local server and in-process inference. Multimodal inputs can be added through provider capabilities later.
- V1 harness packages are declarative only and cannot execute code; any future executable extension system requires a separate trust and isolation design.
- Child tasks inherit a restricted subset of the parent's context and authority by default; hosts may restrict them further.
- In v1, child tasks inherit their parent's provider and model; model routing overrides for child tasks are deferred.
- User-specific configuration changes behavior but cannot weaken host-enforced approval, privacy, or authorization rules.
- Pi and spaex are design references, not required runtime dependencies or sources of executable code.
