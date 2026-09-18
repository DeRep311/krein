# claude-cli-adapter Specification

## Purpose

The `claude` CLI is the one pinned coding-agent adapter this slice wires into `execd`. This specification defines its pinned invocation contract (argv, tool allowlist/denylist), the requirement that its output be validated against the `review-diff` findings schema before being trusted, and the explicit mapping of every adapter-caused failure mode — tool denial, turn exhaustion, timeout, non-zero exit, schema-invalid output — to an observable outcome rather than a silently smaller result. The completeness/partial/failure *schema* itself is defined by `review-diff-skill`; this specification defines how the adapter's concrete behavior is translated into that schema's fields.

## Requirements

### Requirement: Pinned Invocation and Explicit Tool Scope

`execd` MUST invoke the `claude` CLI with a pinned, tested argv: `claude -p --bare --output-format json --json-schema <findings-schema> --permission-mode auto --permission-prompts none`, accompanied by an explicit read-only tool allowlist (`Read`, `Grep`, `Glob`) and an explicit denylist of mutating/egress-capable tools (`Write`, `Edit`, `Bash`, `WebFetch`, `WebSearch`, `Task`). The exact flag names used to express the allowlist/denylist MUST be verified against the installed `claude` CLI version before implementation and asserted by a test; this requirement fixes the intended behavior (only read-only tools reachable), not a specific flag spelling.

#### Scenario: Adapter is invoked with the pinned, tested argv

- GIVEN `execd` is about to run a `RunReviewAdapter` job
- WHEN it constructs the `claude` CLI invocation
- THEN the invocation matches the pinned argv and tool scope asserted by the adapter's own test
- AND no mutating or network-egress tool (`Write`, `Edit`, `Bash`, `WebFetch`, `WebSearch`, `Task`) is reachable by the invocation

### Requirement: Output Validated Before Being Trusted

`execd` MUST validate the adapter's stdout against the `review-diff` findings JSON schema before treating it as a result. Output that is unparseable or fails schema validation MUST NOT be treated as a valid result.

#### Scenario: Schema-invalid output is never treated as a result

- GIVEN the adapter exits with code 0 but emits output that is plain text or JSON violating the findings schema
- WHEN `execd` validates the output
- THEN `execd` treats validation as failed and does not construct a `review-diff` result from that output
- AND `execd` logs the raw unparseable output for diagnosis

### Requirement: Tool Denial and Turn Exhaustion Map to a Partial Outcome

If the adapter's output or execution metadata indicates one or more denied tool calls, `execd` MUST map that to `completeness.status = "partial"` with `completeness.tools_denied` populated. If the adapter exhausts its configured turn budget without natural completion, `execd` MUST map that to `completeness.status = "partial"` with `completeness.turns_exhausted = true`.

#### Scenario: Detected tool denial is mapped to a partial result

- GIVEN the adapter's execution reports a denied call to a tool outside its allowlist
- WHEN `execd` constructs the `review-diff` result
- THEN `execd` sets `completeness.status = "partial"` and lists the denied tool(s) in `completeness.tools_denied`

#### Scenario: Turn exhaustion is mapped to a partial result

- GIVEN the adapter reaches its configured turn budget without reaching natural completion
- WHEN `execd` constructs the `review-diff` result
- THEN `execd` sets `completeness.status = "partial"` and `completeness.turns_exhausted = true`

### Requirement: Timeout Maps to a Failed Task

If the adapter exceeds its wall-clock timeout, `execd` MUST report an execution-timeout failure (after the sandbox termination and worktree release defined in `sandboxed-execution`), and `orchd` MUST transition the A2A task to `failed`.

#### Scenario: Timeout is surfaced as a failed task, never a silent partial success

- GIVEN an adapter run exceeds its configured wall-clock timeout
- WHEN `execd` reports the outcome to `orchd`
- THEN `execd` returns an execution-timeout error
- AND `orchd` transitions the A2A task to `failed` with reason `execution_timeout`

### Requirement: Non-Zero Exit Maps to a Failed Task

If the adapter process exits with a non-zero code, `execd` MUST capture its `stderr`, clean up the session's worktree and lease, and report an adapter-failure outcome; `orchd` MUST transition the A2A task to `failed`.

#### Scenario: Non-zero exit surfaces stderr and fails the task

- GIVEN the adapter process crashes or exits with a non-zero code
- WHEN `execd` detects the non-zero exit
- THEN `execd` captures the process `stderr`, cleans up the worktree and lease, and returns an adapter-failure response including the captured `stderr`
- AND `orchd` marks the A2A task as `failed`

### Requirement: Schema-Invalid Output Maps to a Failed Task

If the adapter's output fails schema validation (per the Output-Validated-Before-Being-Trusted requirement), `execd` MUST clean up the session's worktree and lease and report a schema-validation failure; `orchd` MUST transition the A2A task to `failed` with reason `invalid_output_schema`.

#### Scenario: Schema-invalid output fails the task with a clear reason

- GIVEN the adapter's output fails validation against the findings schema
- WHEN `execd` reports the outcome
- THEN `execd` cleans up the worktree and lease
- AND `orchd` marks the A2A task as `failed` with reason `invalid_output_schema`
