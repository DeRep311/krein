# session-worktree Specification

## Purpose

`execd` resolves review targets exclusively against a locally configured repository registry and allocates one ephemeral git worktree per session at a validated commit. This specification defines that resolution, the lease-before-mutation invariant that makes crash recovery possible, normal teardown, startup and periodic reconciliation of abandoned leases, and the concurrency cap that turns unbounded leakage into an explicit, observable error. `execd` never fetches from a git remote to satisfy this capability.

## Requirements

### Requirement: Local Repository Registry Resolution

`execd` MUST resolve an incoming `repo_id` exclusively against a pre-configured local repository registry mapping `repo_id` to a local filesystem path. `execd` MUST NOT perform any remote network fetch, clone, or pull operation to acquire a missing repository or commit.

#### Scenario: Registered repository resolves to its local path

- GIVEN `execd`'s local repository registry contains `repo-alpha` mapped to a local path
- WHEN `execd` receives an `AllocateWorktree` job naming `repo_id="repo-alpha"`
- THEN `execd` resolves the request against that local path
- AND `execd` performs no network fetch, clone, or pull operation

### Requirement: Unavailable Repository or Commit Fails Explicitly

If the requested `repo_id` does not exist in the local registry, or the requested `base_commit` does not exist within the resolved local repository, `execd` MUST return an explicit `repo_unavailable` error without allocating any resources (no worktree, no lease file).

#### Scenario: Unregistered repository is rejected

- GIVEN `execd`'s local repository registry contains only `repo-alpha`
- WHEN `orchd` dispatches `AllocateWorktree` naming `repo_id="repo-nonexistent"`
- THEN `execd` returns an error with code `repo_unavailable`
- AND no worktree directory or lease file is created
- AND `orchd` transitions the A2A task to `failed` with diagnostic `repo_unavailable`

#### Scenario: Absent base commit is rejected

- GIVEN `repo-alpha` is registered locally but does not contain commit `deadbeef`
- WHEN `orchd` dispatches `AllocateWorktree` naming `repo_id="repo-alpha"` and `base_commit="deadbeef"`
- THEN `execd` returns an error with code `repo_unavailable`
- AND no worktree directory or lease file is created

### Requirement: Lease-Before-Allocation Worktree Provisioning

For each valid session, `execd` MUST allocate an isolated git worktree at `<state-dir>/worktrees/<session-id>` targeting the validated `base_commit`. `execd` MUST atomically write a lease file at `<state-dir>/leases/<session-id>.json` — containing `session_id`, `task_id`, `worktree_path`, `repo_id`, `owning_pid`, and `created_at` — before invoking `git worktree add`, so a crash between the two steps always leaves lease evidence rather than a silent orphan.

#### Scenario: Lease file precedes worktree creation

- GIVEN a valid `AllocateWorktree` request for a registered repository and existing commit
- WHEN `execd` provisions the session worktree
- THEN the lease file at `<state-dir>/leases/<session-id>.json` is written before `git worktree add` runs
- AND the resulting worktree exists at `<state-dir>/worktrees/<session-id>`

### Requirement: Deterministic Teardown

On `ReleaseWorktree`, `execd` MUST run `git worktree remove --force` on the session's worktree and then delete the corresponding lease file.

#### Scenario: Normal teardown removes worktree then lease

- GIVEN a session holds an active worktree and lease file
- WHEN `orchd` dispatches `ReleaseWorktree` for that session
- THEN `execd` removes the worktree with `git worktree remove --force`
- AND `execd` deletes the session's lease file after successful removal

### Requirement: Worktree Concurrency Cap

`execd` MUST enforce a configured concurrency cap on active worktree leases. When the cap is reached, `execd` MUST reject new `AllocateWorktree` requests with an explicit capacity error rather than allocating past the cap.

#### Scenario: Allocation beyond the cap is rejected explicitly

- GIVEN `execd` is configured with a worktree concurrency cap of `N`, and `N` sessions currently hold active leases
- WHEN `orchd` sends an `AllocateWorktree` request for session `N+1`
- THEN `execd` rejects the request with code `concurrency_limit_exceeded`
- AND `execd` does not create a new worktree directory or lease file

### Requirement: Startup Lease Reconciliation

On startup, `execd` MUST sweep `<state-dir>/leases/`. Any lease whose `owning_pid` is no longer running MUST be reclaimed: `git worktree remove --force` on the worktree, `git worktree prune` on the parent repository, then deletion of the lease file.

#### Scenario: Orphaned lease from a crashed process is reclaimed at startup

- GIVEN a prior `execd` process terminated unexpectedly, leaving `<state-dir>/leases/sess-orphan-01.json` and worktree `<state-dir>/worktrees/sess-orphan-01`, whose recorded `owning_pid` no longer exists
- WHEN a new `execd` process starts
- THEN startup reconciliation identifies the lease as abandoned
- AND `execd` runs `git worktree remove --force` on the orphaned worktree, then `git worktree prune` on the parent repository, then deletes the lease file
- AND `git worktree list` shows no dangling worktree under `<state-dir>/worktrees/`

### Requirement: Periodic Lease Sweep

`execd` MUST run the same reconciliation sweep on a periodic timer, at an interval short enough that a lease exceeding the configured session TTL is reclaimed within one TTL of expiry, independent of process restarts.

#### Scenario: Expired lease from a hung session is reclaimed by the periodic sweep

- GIVEN a session's adapter run hangs or is killed mid-task without `ReleaseWorktree` being called, and the lease's age exceeds the configured session TTL
- WHEN the periodic reconciliation timer fires
- THEN `execd` detects the expired lease and forcefully removes the worktree and lease file
- AND `execd` logs a reconciliation reclamation event

### Requirement: Bounded Reconciliation Blast Radius

Reconciliation MUST operate strictly within `<state-dir>/worktrees/`. `execd` MUST NOT delete, prune, or otherwise mutate any worktree path located outside the managed root; a worktree reported by git outside that root MUST be reported, never removed.

#### Scenario: Worktree outside the managed root is never touched

- GIVEN `git worktree list` on a registered repository reports a worktree path outside `<state-dir>/worktrees/`
- WHEN reconciliation runs (startup or periodic)
- THEN `execd` does not remove, prune, or otherwise mutate that out-of-root worktree
- AND `execd` reports it rather than silently ignoring or deleting it
