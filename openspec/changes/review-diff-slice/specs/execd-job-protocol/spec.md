# execd-job-protocol Specification

## Purpose

The `orchd`↔`execd` socket is the only channel by which `orchd`, the unprivileged edge daemon, can cause `execd`, the privileged daemon holding credentials, worktrees, and sandbox control, to do anything. This specification defines the transport, the message framing, peer authentication, and the closed job-spec enum. The enum is exhaustive by design: no wire value maps to arbitrary command execution, so a compromised or confused `orchd` still cannot ask `execd` to do more than allocate, run-review, or release.

## Requirements

### Requirement: Unix Domain Socket Transport

Communication between `orchd` and `execd` MUST occur exclusively over a local Unix domain socket. Neither process MUST expose this protocol over a network-reachable transport.

#### Scenario: Job dispatch uses the local socket

- GIVEN `execd` is listening on its configured Unix domain socket path
- WHEN `orchd` dispatches a job (e.g. `AllocateWorktree`) after accepting a `review-diff` request
- THEN the job request is sent exclusively over that local Unix domain socket

### Requirement: Unambiguous Message Framing

The socket protocol MUST use a message framing mechanism (e.g. length-prefixed or newline-delimited encoding) that unambiguously delimits one job request or response from the next, so a partial or concatenated read can never be misinterpreted as a different or additional job. The specific framing choice is a design-phase decision; this requirement fixes only that framing be unambiguous and validated by a test.

#### Scenario: Concatenated writes do not merge into a different job

- GIVEN a client writes two valid job requests to the socket in immediate succession
- WHEN `execd` reads and decodes the stream
- THEN `execd` parses exactly two distinct job requests, matching the two that were sent, with no merged or truncated job

### Requirement: Closed Job-Spec Enum

The socket protocol MUST enforce a closed job-spec enum consisting exactly of `AllocateWorktree`, `RunReviewAdapter`, and `ReleaseWorktree`. `execd` MUST reject any message naming an unknown job type or carrying unrecognized payload fields. No generic "exec", "eval", or arbitrary-shell-command job type MUST exist on the wire or in `execd`'s dispatch table.

#### Scenario: Unknown job type is rejected without executing anything

- GIVEN a client sends a raw message on the `execd` socket with an unrecognized job type (e.g. `{"job": "RunArbitraryCommand", "command": "rm -rf /"}`)
- WHEN `execd` receives and attempts to dispatch the message
- THEN `execd` rejects the message as an unsupported job type
- AND `execd` drops the connection or returns an error without executing any subprocess

### Requirement: Local Peer Authentication

`execd` MUST authenticate the identity of the connecting local peer before accepting any job request from it, so an unauthorized local process cannot submit execution jobs. The exact authentication mechanism (e.g. restrictive socket file permissions vs. an `SO_PEERCRED`-style credential check) is a design-phase decision, not fixed by this specification.

#### Scenario: Unauthorized local peer is rejected

- GIVEN `execd` is configured to accept jobs only from an authorized local peer identity
- WHEN a local process other than the authorized `orchd` instance connects to the socket and submits a job request
- THEN `execd` rejects the connection or the request without dispatching any job
