# Exploration: review-diff-slice — first vertical slice (A2A "review this diff" skill over orchd/execd)

## Current State

`~/dev/krein` is an empty Go repo (branch `main`, zero commits/files except the SDD/git scaffold). No `go.mod`, no source. Existing scaffold: `openspec/config.yaml` (declares the orchd/execd architecture, `go test ./...`, strict TDD), `openspec/specs/`, `openspec/changes/archive/`, `.atl/skill-registry.md` (no project-level skills yet; `go-testing`, `work-unit-commits`, `chained-pr` are the only relevant user-level skills, none apply to this exploration phase itself).

The full prior design baseline lives in Engram at `krein/architecture-baseline` (obs #11) and is treated as settled: Go, `orchd` (core daemon: A2A edge, Airlock, workflow engine, registry, memory, MCP pool) / `execd` (privileged: credentials, sandbox control, git worktrees) split over a local socket with a closed job-spec enum, A2A for brain-to-brain federation, MCP for local tools, the anti-confused-deputy invariant (remote content never selects a tool/skill directly, only fills params of an already-approved capability), ephemeral sandbox execution, worktree-per-session, `endpoint+token` transport contract.

This slice is explicitly the minimal walking skeleton: two machines/sessions, one shared token, one exposed A2A skill ("review this diff") → allocate a fresh git worktree → run one pinned CLI adapter (`claude` CLI) in a sandbox → return structured findings.

## Affected Areas (all net-new — repo is empty)

- `go.mod` / module root — nothing exists yet; first commit must establish the module and Go version floor (confirmed installed: **go1.26.8**, comfortably above the `a2a-go` 1.25.0+ requirement).
- `cmd/orchd/`, `cmd/execd/` — the two process entry points from the settled split.
- `internal/a2a/` (or equivalent) — A2A server: AgentCard, JSON-RPC/REST handler exposing exactly one skill (`review-diff`), lifecycle state machine.
- `internal/execd/worktree/` — git worktree allocation/teardown per session.
- `internal/execd/sandbox/` — sandbox process wrapper around the adapter invocation.
- `internal/adapter/claude/` — subprocess wrapper for the `claude` CLI, argument construction, stdout parsing, result mapping to the structured findings contract.
- `internal/orchd/socket/` — the orchd↔execd local-socket job-spec protocol (closed enum, per baseline).
- Test scaffolding under each of the above per `go test ./...` / strict TDD (`openspec/config.yaml` already pins this).

## Approaches

### Axis 1 — A2A server implementation in Go

1. **`a2aproject/a2a-go` (official SDK)** — high-level `a2asrv`/`a2aclient` packages, transport-agnostic handler wrapped into JSON-RPC, gRPC, or REST bindings; implements A2A v1.0 spec; Apache-2.0, ~468 stars, active CI.
   - Pros: canonical/spec-compliant, saves reimplementing AgentCard + 8-state lifecycle + JSON-RPC 2.0 envelope; multi-transport out of the box.
   - Cons: requires Go 1.25.0+ (now confirmed satisfied); young v2 surface, API may still shift; pulls in a real dependency for a one-skill MVP.
   - Effort: Low-Medium.
2. **`trpc-group/trpc-a2a-go`** — alternative community Go implementation, framework-agnostic.
   - Pros: fallback if the official SDK's API churn is a blocker.
   - Cons: not canonical; less certain long-term spec alignment.
   - Effort: Medium.
3. **Hand-rolled minimal JSON-RPC 2.0 server** — implement just enough of AgentCard discovery + the 8-state lifecycle for one skill.
   - Pros: zero external protocol dependency, full control.
   - Cons: reimplements a spec Google already published a reference SDK for; higher risk of subtle non-compliance breaking interop with a second brain running the official SDK.
   - Effort: High.

### Axis 2 — sandbox for execd-triggered execution

1. **gVisor (`runsc`)** — user-space kernel intercepting syscalls, ships as an OCI runtime.
   - Pros: strong isolation, mature, Google-maintained.
   - Cons: expects Docker/containerd underneath — real infra weight for a two-machine MVP.
   - Effort: Medium-High.
2. **bubblewrap (`bwrap`)** — unprivileged namespace-based sandboxing, no daemon, used by Flatpak.
   - Pros: no daemon, no OCI runtime prerequisite, `execd` shells out to it directly per adapter invocation; matches the execd "host you trust, subprocess you don't" trust boundary already decided.
   - Cons: weaker isolation than a userspace-kernel/microVM approach; needs careful namespace/seccomp flag tuning per adapter.
   - Effort: Low-Medium — best fit for a first slice given no container infra exists yet.
3. **Firecracker / E2B / Vercel Sandbox** — candidates already named in the baseline.
   - Pros: strongest isolation.
   - Cons: Firecracker needs KVM + jailer setup on every host; E2B/Vercel are cloud services, conflicting with the MVP's local/offline "two machines, one shared token" shape and reintroducing a third-party trust dependency.
   - Effort: High for this slice; revisit once self-created skills execute.

### Axis 3 — git worktree allocation

1. **Shell out to `git worktree add/remove` via `os/exec`** — execd already needs subprocess execution machinery for git credentials.
   - Pros: uses git's actual worktree feature; no extra dependency; composable with the sandbox wrapper.
   - Cons: must handle concurrent worktree names/paths and cleanup-on-crash carefully.
   - Effort: Low.
2. **`go-git`** — pure-Go git implementation.
   - Cons: its `Worktree` type models a single checkout, not git's multi-checkout `git worktree add` feature — wrong abstraction for "worktree per session".
   - Effort: High relative to the actual need.
3. **`Worktrunk`** — CLI wrapper around `git worktree` built for parallel AI-agent workflows.
   - Pros: purpose-built, has direct Claude Code integration.
   - Cons: adds an external CLI/process for something `os/exec` + native `git worktree` already does in ~3 commands; scope-inappropriate for the first slice.
   - Effort: Low to adopt, but premature.

### Axis 4 — subprocess-wrapping the `claude` CLI

1. **`claude -p --output-format json` (or `--json-schema`)** — non-interactive mode, structured JSON envelope on stdout.
   - Pros: documented, stable public contract (unlike the internal `.jsonl` transcript format, explicitly unstable); `--json-schema` lets the adapter request the exact "structured findings" shape the skill needs; `--bare` avoids loading the invoking host's own hooks/MCP/CLAUDE.md inside the sandboxed worktree — the sandboxed adapter run must not inherit ambient trust from whatever `~/.claude` exists in that environment; exit code gives execd a clean success signal.
   - Cons: needs `--permission-prompts none` and an explicit permission mode since `-p` defaults to Manual and would otherwise hang; the API key must be provisioned into the sandbox per the baseline's credential-isolation intent, not inherited from execd's own environment.
   - Effort: Low-Medium.
2. **`claude -p --output-format stream-json`** — NDJSON event stream.
   - Pros: only needed if the skill must stream partial progress back through the A2A edge.
   - Cons: bigger parser surface for no clear MVP benefit; more moving parts to keep in schema-sync with an evolving internal event vocabulary.
   - Effort: Medium; defer past MVP.

## Recommendation

For this slice: **a2a-go (JSON-RPC binding) + bubblewrap + `os/exec` git worktree + `claude -p --bare --output-format json --json-schema <findings-schema> --permission-mode auto --permission-prompts none`**.

This reuses the canonical A2A SDK instead of reimplementing lifecycle/AgentCard semantics, avoids pulling in container/VM infrastructure the architecture doesn't otherwise need yet (bubblewrap fits the execd trust boundary with zero daemon), uses git's real worktree feature directly, and pins the `claude` CLI to its one documented, schema-validated non-streaming output contract — giving "structured findings back to the requesting brain" a concrete, enforceable shape instead of ad hoc prompt-output parsing. Revisit gVisor/Firecracker/Worktrunk/streaming once the MVP is proven and the deferred requirements (self-skill-creation, multiple adapters, many concurrent sessions) materialize.

## Risks

- ~~Unverified Go toolchain floor~~ — **resolved**: installed toolchain is go1.26.8, above the `a2a-go` 1.25.0+ requirement.
- **`a2a-go` API maturity**: young v2 surface (266 commits); pin an exact module version/commit and revisit if churn is high.
- **Sandbox credential isolation is unresolved in detail**: the mechanism for injecting only an API key into a bubblewrap-sandboxed `claude` invocation — without leaking execd's broader git/API credentials — is not yet designed; this is a design-phase decision.
- **`claude` CLI permission/consent bypass in an unattended sandbox**: `--permission-mode auto --permission-prompts none` denies anything a classifier doesn't clear; the skill's prompt/tool allowlist needs to be narrow enough that this doesn't silently drop expected findings — worth an explicit scenario in `sdd-spec`.
- **bubblewrap isolation is weaker than gVisor/microVM**: acceptable for this slice's stated scope, but flagged as a deliberately deferred hardening step once self-created skills go live.
- **Worktree cleanup on crash**: shelling out to `git worktree add/remove` needs explicit crash/orphan handling, or stale worktrees accumulate.

## Ready for Proposal

Yes. The four technical axes have a concrete, justified recommendation and the open risks are decisions for `sdd-propose`/`sdd-design` (credential injection, permission-mode/tool-allowlist, worktree crash recovery), not blockers.

## Key Learnings

1. `a2aproject/a2a-go` is the official Go A2A SDK (v2, JSON-RPC/gRPC/REST via `a2asrv`); Go 1.25.0+ requirement is satisfied by the installed go1.26.8 toolchain.
2. The `claude` CLI's only documented, version-stable programmatic output contract is `-p --output-format json` (optionally with `--json-schema`); the internal `.jsonl` transcript format is explicitly unstable and unsuitable for parsing.
3. `go-git`'s Worktree type models a single checkout, not git's multi-checkout `git worktree add` feature, so the baseline's "worktree per session" requirement is better served by shelling out to the real `git worktree` command.
4. bubblewrap (`bwrap`) needs no daemon or OCI runtime and fits the execd "host you trust, subprocess you don't" trust boundary better than gVisor for a first MVP slice.
5. `claude -p` defaults to Manual permission mode and will hang waiting for approval unless `--permission-mode auto` and `--permission-prompts none` are both set for unattended sandboxed execution.

---
*Mirrored from Engram `sdd/review-diff-slice/explore` (obs #15) by the orchestrator — the `sdd-explore` phase agent's toolset in this session had no Write access, so this OpenSpec file was written directly by the parent orchestrator from the persisted observation content, with the Go-toolchain risk item resolved (`go version` confirmed go1.26.8) before handoff to `sdd-propose`.*
