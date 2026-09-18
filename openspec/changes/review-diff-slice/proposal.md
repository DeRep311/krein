# Proposal: review-diff-slice — first vertical slice (A2A "review this diff" over orchd/execd)

## Intent

Krein currently exists as an architecture baseline and an empty Go repository: zero commits, no `go.mod`, no source. Every security-relevant decision in the baseline (the `orchd`/`execd` process split, the closed job-spec enum, the anti-confused-deputy invariant, ephemeral sandboxed execution, worktree-per-session, the `endpoint+token` transport contract) is currently unproven prose. Nothing yet demonstrates that the trust boundary survives contact with a real remote request.

This change builds the **minimal end-to-end walking skeleton** that makes those invariants executable: two brains, one shared token, exactly one exposed A2A skill (`review-diff`), a fresh git worktree per session, one pinned CLI adapter (`claude`) executed inside a bubblewrap sandbox, and structured findings returned over the A2A edge.

It exists now because every later capability (self-created skills, shared/DMZ memory, additional adapters, a control plane) inherits this trust boundary. Establishing it first is far cheaper than retrofitting isolation under features that already assume the host.

Success looks like: brain A sends a `review-diff` A2A task to brain B; brain B authenticates it, never lets remote content select a capability, allocates an isolated worktree, runs one sandboxed adapter that can see the code under review and nothing else of B's, and returns schema-valid findings — with every failure mode visible rather than silent.

## Scope

### In Scope

- `go.mod` module bootstrap pinning the Go version floor (installed toolchain verified: go1.26.8, above the `a2a-go` 1.25.0+ requirement).
- `cmd/orchd` and `cmd/execd` entry points as two separate processes over a local Unix socket.
- A2A edge in `orchd` using `a2aproject/a2a-go` (JSON-RPC binding) at a pinned module version: AgentCard publication, one advertised skill (`review-diff`), task lifecycle, shared-token authentication, append-only audit log of inbound remote requests.
- Enforcement of the anti-confused-deputy invariant at the edge: an inbound A2A request can only fill the parameters of the locally pre-registered `review-diff` capability; remote content never selects a capability, tool, or adapter.
- The orchd↔execd socket protocol with a **closed** job-spec enum — for this slice exactly: `AllocateWorktree`, `RunReviewAdapter`, `ReleaseWorktree`. No arbitrary-command job exists in the enum or in the wire type.
- Per-session git worktree allocation and teardown in `execd` via `os/exec` over native `git worktree add/remove/prune`, with on-disk lease records and crash/orphan reconciliation (see Approach §4).
- A bubblewrap sandbox profile in `execd` with explicit credential isolation: cleared environment, masked home, read-only worktree, no git credentials, single scoped adapter secret delivered by file descriptor (see Approach §3).
- The `claude` CLI adapter: pinned argv contract, read-only tool allowlist, JSON-schema-constrained structured findings, and explicit mapping of denials/timeouts/non-zero exits to a *visible* partial or failed result (see Approach §5).
- The `review-diff` request/response contract, including the structured findings schema and its `completeness` block.
- Tests under strict TDD per `openspec/config.yaml` (`go test ./...`, `gofmt`, `go vet`), including an end-to-end test exercising the A2A task against a local fixture repository.

### Out of Scope

- Shared/DMZ memory, and any promotion of results into it.
- The self-skill-creation loop and any dynamically registered capability.
- Any GUI or local control plane; the control-plane surface stays unimplemented and deliberately distinct from the A2A edge.
- Any adapter other than `claude` (Antigravity `agy`, Codex).
- Cost ledger / per-brain cost attribution (the inbound-request audit log **is** in scope; cost accounting is not).
- Formal identity federation (mTLS, SPIFFE/SPIRE, Cedar policy scoping). This slice uses one shared token.
- Streaming partial progress (`--output-format stream-json`); this slice returns one final result.
- Stronger isolation backends (gVisor, Firecracker, E2B, Vercel Sandbox) — deferred hardening, explicitly re-opened before self-created skills execute.
- Fetching code from a git remote. `execd` never holds or uses git remote credentials in this slice (see Approach §2).
- Multi-tenant concurrency tuning; a small fixed concurrency cap is sufficient here.

## Capabilities

### New Capabilities

- `a2a-edge`: AgentCard publication, shared-token authentication of inbound brain-to-brain requests, A2A task lifecycle for one skill, the append-only inbound audit log, and enforcement of the anti-confused-deputy invariant.
- `review-diff-skill`: the `review-diff` request/response contract — accepted inputs, input bounds, the structured findings schema, and the completeness/partial/failure semantics of a returned result.
- `execd-job-protocol`: the orchd↔execd local socket — framing, peer authentication, and the closed job-spec enum with its exhaustive operation set.
- `session-worktree`: per-session worktree allocation from a locally registered repository, lease records, deterministic teardown, crash/orphan reconciliation, and the concurrency cap.
- `sandboxed-execution`: the bubblewrap profile — namespace and filesystem isolation, environment clearing, scoped secret injection, and the wall-clock/resource budget.
- `claude-cli-adapter`: the pinned `claude` CLI invocation contract, the read-only tool allowlist, output parsing against the findings schema, and the mapping of denials, timeouts, and exit codes to observable outcomes.

### Modified Capabilities

None. `openspec/specs/` contains no specs yet (only `.gitkeep`); every capability above is net-new.

## Approach

Follows the exploration recommendation: `a2a-go` (JSON-RPC) + bubblewrap + native `git worktree` via `os/exec` + `claude -p` with a JSON schema. The four axes are settled; what follows resolves the three risks the exploration deliberately left to this phase.

### 1. Process and data flow

```
brain A ──A2A/JSON-RPC (endpoint+token)──▶ orchd (edge, audit, capability binding)
                                             │ local Unix socket, closed job enum
                                             ▼
                                           execd (privileged: repo registry, worktrees, secrets, sandbox)
                                             │ bwrap
                                             ▼
                                           claude CLI (read-only, isolated) ──▶ structured findings
```

`orchd` never touches credentials, worktrees, or the sandbox. `execd` never parses A2A, never sees a remote payload as anything but already-validated parameters of an approved capability.

### 2. Decision: the request carries the patch; `execd` never fetches

The `review-diff` request carries the **patch inline** (bounded, default 1 MiB), plus a repository identifier and a base commit. `execd` resolves the identifier against a **locally configured repo registry** and allocates a worktree at that base commit. If the repo or commit is not present locally, the task fails with an explicit `repo_unavailable` error.

Rationale: the alternative — accepting a remote URL and cloning/fetching — would require `execd` to hold git remote credentials and would let a remote brain name the fetch target, which is exactly the confused-deputy shape the baseline forbids. Inline patch + local registry keeps the sandbox at **zero git credentials** and keeps target selection entirely local.

### 3. Resolution — sandbox credential isolation

The sandboxed adapter receives exactly one secret and nothing else of `execd`'s identity.

- **No inheritance.** The adapter subprocess is built with an explicitly constructed environment; `os.Environ()` is never passed through. `bwrap --clearenv` plus an explicit minimal set (`PATH`, `HOME`, `LANG`, `TERM=dumb`).
- **No secret in argv.** The API key is *not* passed via `--setenv` or a CLI flag, because `bwrap`'s argv is readable from `/proc` by other same-uid processes. `execd` writes the key to a pipe and passes the read end to `bwrap --file <fd> /run/krein/adapter.env`; a tiny in-sandbox entry shim reads that file, exports `ANTHROPIC_API_KEY`, unlinks the file, and `exec`s `claude`. The secret therefore exists only in sandbox-private memory and a sandbox-private tmpfs.
- **Masked home.** `--tmpfs $HOME` so `~/.claude`, `~/.claude.json`, `~/.gitconfig`, `~/.ssh`, `~/.config/gh`, and `execd`'s own credential directory are structurally absent, not merely unreferenced. `--bare` on the CLI is defence in depth, not the boundary.
- **Read-only code, masked git.** The session worktree is bind-mounted read-only at a fixed in-sandbox path; `<worktree>/.git` is masked with a tmpfs so the adapter cannot reach the main repository's object store, config, or credential helper. The patch under review is supplied as a separate read-only file.
- **Namespaces.** `--unshare-all --share-net` (network egress is required for the model API, and only for it), `--die-with-parent`, `--new-session`, non-zero `--uid/--gid` mapping, `--proc`, `--dev`, read-only `/usr` and `/lib*`.
- **Secret provenance.** `execd` loads the adapter key at startup from a configured file path and **refuses to start** if it is group- or world-readable. OS keyring / systemd credentials are deferred.

Net effect: compromising the adapter yields one model API key and read access to code the requester already sent, and nothing else.

### 4. Resolution — worktree crash and orphan recovery

- **Deterministic identity.** One worktree per task at `<state-dir>/worktrees/<session-id>`, never reused. A lease file `<state-dir>/leases/<session-id>.json` (session id, task id, worktree path, owning pid, repo id, created-at) is written **before** `git worktree add` and deleted **after** a successful removal — so a crash always leaves evidence, never a silent orphan.
- **Normal teardown.** `git worktree remove --force` on the deferred path, then lease deletion.
- **Startup reconciliation.** On boot, `execd` sweeps the lease directory: any lease whose owning pid is gone, or whose age exceeds the session TTL, is force-removed and pruned (`git worktree remove --force` then `git worktree prune`), then its lease is deleted.
- **Periodic sweep.** The same reconciliation runs on a timer, so a long-lived `execd` that lost a child still reclaims within one TTL.
- **Bounded blast radius.** Reconciliation only ever touches paths under the krein-managed worktree root; a worktree listed by git outside that root is reported and never removed. `execd` refuses new allocations past a configured concurrency cap, so a leak degrades into an explicit, observable error instead of unbounded disk growth.

### 5. Resolution — adapter permission scope without silent finding loss

The exploration flagged that `--permission-mode auto --permission-prompts none` denies whatever a classifier does not clear, which could silently shrink the findings set. Resolution has two halves:

- **The kernel is the boundary, not the classifier.** Because the sandbox already guarantees read-only code, a masked home, and no git or repo-write access, the adapter does not need the CLI's own permission heuristics to be the security control. The invocation therefore pins an explicit **read-only tool allowlist** (`Read`, `Grep`, `Glob`) and explicitly denies the mutating and egress-capable set (`Write`, `Edit`, `Bash`, `WebFetch`, `WebSearch`, `Task`) so the adapter's capability set is fixed and auditable rather than negotiated per invocation. The exact flag spelling for the allowlist is a spec-phase verification item against the installed CLI version; the *contract* is that the allowlist is explicit and the pinned argv is asserted by a test.
- **Denials must be loud.** The findings schema carries a mandatory `completeness` block: `{ status: complete|partial, tools_denied: [...], turns_exhausted: bool, truncated: bool }`. If the adapter reports any permission denial, exhausts its turn budget, hits the wall-clock timeout, exits non-zero, or emits output that fails schema validation, `execd` returns a result that is explicitly `partial` or the A2A task transitions to `failed` — never a clean `completed` carrying a quietly smaller findings list. A wall-clock timeout and a turn budget are mandatory so a hung adapter surfaces as a timeout instead of a hang.

Pinned invocation shape (from exploration, plus the allowlist): `claude -p --bare --output-format json --json-schema <findings-schema> --permission-mode auto --permission-prompts none <allowlist flags>`.

### 6. Protocol-surface impact

Per `openspec/config.yaml`, protocol surfaces touched:

- **A2A wire format** — new. AgentCard + exactly one skill (`review-diff`) + its input/output schema. Spec-compliance is delegated to the pinned `a2a-go` version.
- **orchd/execd socket** — new. Closed job-spec enum; adding an operation is a deliberate spec change, not an implementation detail.
- **MCP tool contracts** — untouched. No MCP surface exists in this slice, and an A2A capability is never mapped 1:1 to an MCP tool.

## Affected Areas

| Area | Impact | Description |
|------|--------|-------------|
| `go.mod`, `go.sum` | New | Module bootstrap, Go version floor, pinned `a2a-go` version |
| `cmd/orchd/` | New | Core daemon entry point: A2A edge + execd client |
| `cmd/execd/` | New | Privileged daemon entry point: socket server, worktrees, sandbox, secrets |
| `internal/a2a/` | New | AgentCard, skill registration, token auth, task lifecycle, audit log |
| `internal/a2a/capability/` | New | Anti-confused-deputy binding: remote payload → params of a pre-registered capability only |
| `internal/jobspec/` | New | Closed job-spec enum and wire types shared by orchd and execd |
| `internal/orchd/socket/` | New | Client side of the local socket protocol |
| `internal/execd/socket/` | New | Server side: peer auth, framing, dispatch |
| `internal/execd/repo/` | New | Local repo registry (repo id → path) |
| `internal/execd/worktree/` | New | Allocation, leases, teardown, reconciliation sweep |
| `internal/execd/sandbox/` | New | bubblewrap profile construction, fd-based secret injection |
| `internal/execd/secret/` | New | Adapter key loading, permission-mode enforcement at startup |
| `internal/adapter/claude/` | New | argv construction, findings schema, output parsing, outcome mapping |
| `openspec/specs/` | New | Six new capability specs land here at archive |

## Risks

| Risk | Likelihood | Mitigation |
|------|------------|------------|
| `a2a-go` API churn (young v2 surface) breaks the build | Med | Pin an exact module version; keep the edge behind a small internal interface so a hand-rolled JSON-RPC fallback is a swap, not a rewrite |
| Allowlist flag spelling differs from the installed `claude` CLI version | Med | Spec-phase verification against the installed CLI; a test asserts the exact pinned argv so drift fails loudly |
| bubblewrap profile too tight — adapter fails to start (missing `/lib`, cert bundle, locale) | Med | Build the profile test-first against a fixture repo; treat startup failure as a `failed` task with the bwrap stderr captured |
| bubblewrap isolation weaker than gVisor/microVM | Med | Accepted for this slice's scope; formally re-opened before self-created skills execute. Blast radius already reduced to one API key + requester-supplied code |
| Adapter non-determinism makes end-to-end assertions flaky | High | Assert schema validity, task lifecycle, and isolation properties — not finding content. Adapter output is faked at the boundary for unit tests |
| Secret leaks via an unexpected path (crash dump, adapter telemetry, log) | Low | fd-based injection, no argv, no parent env; `execd` logs redact the key; the entry shim unlinks the secret file |
| bubblewrap / `claude` CLI absent on the execd host | Med | Preflight capability check at `execd` startup with a clear diagnostic instead of a runtime failure mid-task |
| Worktree reconciliation removes something it should not | Low | Sweep is confined to the managed worktree root; anything outside is reported, never removed |
| Slice exceeds the 400-line review budget | High | `auto-chain` delivery strategy is already cached for this session; `sdd-tasks` slices along the capability boundaries above |

## Rollback Plan

- **Repository state**: the repo has zero commits. Rollback is deleting the feature branch or resetting to the pre-slice commit; there are no consumers, no persisted data, no migrations, and no published protocol version to be compatible with.
- **Per-layer**: each capability lands behind its own package boundary. If `a2a-go` proves unworkable, `internal/a2a` is replaced with a hand-rolled JSON-RPC server without touching `execd`. If bubblewrap proves too restrictive, `internal/execd/sandbox` is the only package changed to swap the isolation backend.
- **Operational**: `execd` stop + the reconciliation sweep reclaims every worktree; removing the configured secret file fully revokes the adapter's ability to run. No host state survives a rollback beyond the managed state directory, which is safe to delete wholesale.

## Dependencies

- Go toolchain ≥ 1.25 (verified installed: go1.26.8).
- `a2aproject/a2a-go` at a pinned version (Apache-2.0).
- `bubblewrap` (`bwrap`) present on every `execd` host.
- `git` with worktree support on every `execd` host.
- `claude` CLI installed on the `execd` host, plus an Anthropic API key readable by `execd` at a configured path with non-permissive file mode.
- At least one repository registered in `execd`'s local repo registry for the end-to-end demo.
- Two hosts (or two sessions) sharing one token for the federation demo.

## Success Criteria

- [ ] `go build ./...`, `go test ./...`, `gofmt -l .` (empty), and `go vet ./...` all pass.
- [ ] An A2A client fetches brain B's AgentCard and sees exactly one skill, `review-diff`.
- [ ] A `review-diff` task with a valid token and inline patch returns schema-valid structured findings and reaches `completed`.
- [ ] A request with a missing or wrong token is rejected and recorded in the append-only inbound audit log.
- [ ] A test proves remote payload content cannot select a capability — an unknown or injected skill/tool name is rejected, never dispatched.
- [ ] A test proves the job-spec enum is closed: no wire value maps to arbitrary command execution.
- [ ] An isolation test proves the sandboxed adapter cannot read `$HOME/.claude`, `~/.ssh`, `~/.gitconfig`, `execd`'s secret file, or the main repo's `.git` object store, and cannot write to the worktree.
- [ ] A test proves the adapter secret never appears in the `bwrap` argv or in `execd`'s inherited environment.
- [ ] A task naming an unregistered repo or an absent base commit fails with `repo_unavailable`.
- [ ] Killing the adapter mid-task leaves a lease; a subsequent reconciliation sweep removes the worktree and the lease, and `git worktree list` is clean.
- [ ] A forced tool denial, turn exhaustion, or timeout surfaces as `partial` or `failed` with a populated `completeness` block — never as a clean `completed`.
- [ ] Every worktree allocated during the full test run is released; no orphan remains under the managed root.

## Open Questions (for sdd-spec / sdd-design, non-blocking)

No unresolved product decision blocks this proposal. The following are technical details delegated forward:

1. Exact `claude` CLI flag spelling for the tool allowlist/denylist, verified against the installed version.
2. Exact JSON schema for a finding (severity vocabulary, file/line/range shape, remediation field).
3. Concrete defaults for the session TTL, wall-clock timeout, turn budget, concurrency cap, and the 1 MiB patch bound.
4. Socket peer-authentication mechanism (filesystem mode on the socket path vs. `SO_PEERCRED` check).
5. Audit-log record format and on-disk location.
