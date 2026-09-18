# a2a-edge Specification

## Purpose

The A2A edge is `orchd`'s exposed surface to remote brains. It publishes discovery metadata, authenticates every inbound remote request, enforces the anti-confused-deputy invariant (remote content never selects a capability — it only fills the parameters of a locally pre-registered one), and records every inbound attempt in an append-only audit log. `orchd` owns this surface exclusively; `execd` never parses A2A traffic and never sees an unvalidated remote payload.

## Requirements

### Requirement: AgentCard Publication

`orchd` MUST publish an A2A AgentCard that advertises exactly one skill, `review-diff`.

#### Scenario: AgentCard exposes a single skill

- GIVEN `orchd` is running with `review-diff` registered as its only local capability
- WHEN a remote A2A client fetches `orchd`'s AgentCard
- THEN the AgentCard lists exactly one skill, `review-diff`
- AND no other skill, tool, or capability name appears in the AgentCard

### Requirement: Shared-Token Authentication

`orchd` MUST authenticate every inbound A2A request against a configured shared bearer token before any further processing. A request with a missing or invalid token MUST be rejected with an authentication error and MUST NOT reach capability dispatch.

#### Scenario: Missing or invalid token is rejected

- GIVEN `orchd` is configured with a shared bearer token
- WHEN a remote client submits an A2A request carrying a missing or incorrect token
- THEN `orchd` rejects the request with an authentication error
- AND no job is dispatched to `execd`

#### Scenario: Valid token is accepted

- GIVEN `orchd` is configured with a shared bearer token
- WHEN a remote client submits an A2A request carrying the correct token
- THEN `orchd` accepts the request for further processing (capability binding and dispatch)

### Requirement: Append-Only Inbound Audit Log

`orchd` MUST append a record of every inbound A2A request attempt — accepted or rejected — to an append-only audit log, before or as part of processing that request. Each record MUST include a timestamp, a remote identifier, the requested skill name, and the outcome (accepted / rejected, with reason). Audit records MUST NOT be mutated or removed by any later operation.

#### Scenario: Rejected request is recorded

- GIVEN `orchd` is configured with a shared bearer token
- WHEN a remote client submits a request with an invalid token
- THEN `orchd` appends an audit record with the rejection outcome and reason to the audit log
- AND the audit log entry is never later modified or deleted by `orchd`

#### Scenario: Accepted request is recorded

- GIVEN `orchd` is configured with a shared bearer token
- WHEN a remote client submits a valid `review-diff` request
- THEN `orchd` appends an audit record marking the request as accepted, alongside the requested skill name and remote identifier

### Requirement: Anti-Confused-Deputy Enforcement

`orchd` MUST enforce that an inbound A2A request can only fill the parameter schema of `review-diff`, the one locally pre-registered capability. `orchd` MUST reject, before job creation, any request that names an unadvertised skill, attempts to select a tool or adapter, or otherwise overrides capability selection via any parameter. Remote content MUST NEVER select which capability, tool, or adapter runs.

#### Scenario: Remote attempt to select an arbitrary skill or tool is rejected

- GIVEN `orchd` is running with only `review-diff` advertised
- WHEN a remote client submits an A2A task naming an unadvertised skill (e.g. `run_bash`) or including extra parameters that attempt to select a tool or adapter (e.g. `tool_allowlist: ["Bash", "Write"]`)
- THEN `orchd` rejects the request before any job is created
- AND `orchd` records the rejection in the audit log
- AND no parameter override or unknown skill/tool name is propagated to `execd`
