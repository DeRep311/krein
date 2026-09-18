# review-diff-skill Specification

## Purpose

`review-diff` is the one A2A capability this slice exposes. This specification defines its accepted inputs and their bounds, the structured findings and completeness result it returns, and the invariant that a returned result must always visibly reflect any partial or failed execution rather than silently hiding it. `orchd` enforces input bounds before dispatch; `execd` and its adapter produce the result content; the completeness/partial/failure semantics defined here bind both.

## Requirements

### Requirement: Accepted Request Inputs and Patch Size Bound

A `review-diff` request MUST carry a repository identifier (`repo_id`), a base commit (`base_commit`), and an inline unified diff (`patch_content`). `orchd` MUST reject any request whose inline patch exceeds the configured maximum patch size (default 1 MiB / 1,048,576 bytes) with an explicit input-size error, before dispatching to `execd`.

#### Scenario: Oversized patch is rejected before dispatch

- GIVEN `orchd` is configured with a maximum patch size of 1 MiB
- WHEN a remote client submits a `review-diff` request whose inline patch exceeds 1 MiB
- THEN `orchd` rejects the request with an explicit input-size error
- AND no `AllocateWorktree` or other job is dispatched to `execd`

#### Scenario: Patch within bound is accepted for dispatch

- GIVEN `orchd` is configured with a maximum patch size of 1 MiB
- WHEN a remote client submits a `review-diff` request with `repo_id`, `base_commit`, and an inline patch under 1 MiB
- THEN `orchd` accepts the request for capability binding and dispatch

### Requirement: Structured Findings Schema

A `review-diff` result MUST carry a `findings` array. Each finding MUST include `file_path`, `line_start`, `line_end`, `severity` (one of `info`, `warning`, `error`, `critical`), `title`, and `description`, and MAY include `suggested_fix`. Output failing this schema MUST NOT be treated as a valid result (see `claude-cli-adapter` for the validation mechanics).

#### Scenario: Valid findings are returned on a successful review

- GIVEN a `review-diff` task completes with the adapter producing schema-valid output
- WHEN `orchd` returns the result to the requesting brain
- THEN each entry in `findings` includes `file_path`, `line_start`, `line_end`, a `severity` value from the defined enum, `title`, and `description`

### Requirement: Completeness Block and No-Silent-Success Invariant

Every `review-diff` result MUST carry a `completeness` block with `status` (`complete` or `partial`), `tools_denied` (array of denied tool names), `turns_exhausted` (boolean), and `truncated` (boolean). If any execution failure, tool denial, turn exhaustion, or truncation occurred, the result MUST NOT report `completeness.status = "complete"`, and the A2A task MUST NOT be reported as a clean `completed` carrying a quietly smaller findings list than what was actually gathered.

#### Scenario: Tool denial is surfaced as partial, never silently dropped

- GIVEN the adapter's execution encounters one or more tool denials while producing findings
- WHEN `execd` returns the `review-diff` result
- THEN `completeness.status` is `"partial"`
- AND `completeness.tools_denied` lists the denied tool names
- AND the task is not reported as a clean `completed`

#### Scenario: Turn budget exhaustion is surfaced as partial

- GIVEN the adapter reaches its configured turn budget without reaching natural completion
- WHEN `execd` returns the `review-diff` result
- THEN `completeness.status` is `"partial"` and `completeness.turns_exhausted` is `true`
- AND the findings gathered up to that point are still returned alongside the partial indicator

#### Scenario: Complete review reports complete status

- GIVEN the adapter finishes within its turn and time budget, with no tool denials and no truncation
- WHEN `execd` returns the `review-diff` result
- THEN `completeness.status` is `"complete"`, `tools_denied` is empty, `turns_exhausted` is `false`, and `truncated` is `false`
