# Task, Event, and Evaluation Contract

## Task tree

- Each run has exactly one root task.
- Every child task has one parent, a stable task ID, explicit input, an independent conversation, selected tools, and limits.
- In v1, a child inherits its parent's provider/model and resolved harness configuration; child-level provider/model overrides are not supported.
- Parallelism is bounded per run and by parent/host limits.
- Parent and child tasks share a finite run-level budget; child allocations are reserved before work starts and cannot exceed the parent's remaining budget.
- Parent cancellation stops admission of new work, propagates to descendants, and waits for graceful stop before using abort as a fallback.
- A child result is returned to its parent with the child ID, final status, stop reason, and declared output.

## Events

Events are ordered within a run and carry `run_id`, `task_id`, optional `parent_task_id`, sequence, timestamp, kind, and redacted metadata. Required event kinds cover task lifecycle, provider request/result, tool proposal/decision/result, approval, delegation, resource limit, cancellation, and final completion.

Prompt text and full tool inputs/results are omitted by default. A host may opt into retaining selected payloads under its own storage and privacy controls. Optional telemetry adapters may export events but do not define canonical persistence.

## Evaluation

An evaluation set pins cases, scripted provider events, fake tool results/errors, expected decisions, and scoring rules. Each report records the evaluation set revision, harness configuration digest, provider/model identity, per-case outcomes, task tree, tool events, stop reasons, and available resource usage.

Fixture-based evaluation must produce the same tool decisions and task-tree outcomes on repeated runs. Live model runs are separate measurements; their reports identify the provider/model and configuration but do not claim deterministic response replay.

## CLI validation surface

The initial CLI provides commands to validate a harness configuration, inspect its resolved contributions and provenance, run a fixture evaluation, and compare two evaluation reports. Exact command spelling is set during implementation.
