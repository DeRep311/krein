# Design: review-diff-slice — first vertical slice (A2A "review this diff" over orchd/execd)

## Technical Approach

The slice is built as **four concentric trust rings**, each with its own Go package boundary, so that a compromise or a bug in an outer ring cannot reach an inner one:

```
ring 0  remote brain            untrusted: supplies bytes only
ring 1  orchd (internal/a2a)    parses untrusted bytes; holds the A2A token; holds NO credentials,
                                NO filesystem authority, NO subprocess authority
ring 2  execd (internal/execd)  holds the adapter secret, the repo registry, worktree authority,
                                sandbox authority; never parses A2A, never sees a remote payload
                                as anything but already-validated parameters
ring 3  bwrap + claude          untrusted again: one API key, a read-only tree, no host identity
```

The two *narrow waists* between rings are the only places the design spends complexity:

1. **`internal/skill/reviewdiff`** — the A2A-facing contract. A remote request is a closed, strictly-decoded struct. Nothing outside that struct crosses ring 1.
2. **`internal/jobspec`** — the orchd↔execd contract. A closed three-member operation enum, decoded through a fixed table with an exhaustive test. Nothing outside that enum crosses ring 2.

Everything else (worktree leases, reconciliation, the bwrap profile, the adapter argv, the completeness taxonomy) hangs off ring 2 and never touches ring 1.

Two principles drive most of the concrete decisions below, and they are worth stating once:

- **Closedness is enforced at decode, not by the type system.** Go interfaces are open and `encoding/gob` picks concrete types from the wire. A "closed enum" that is only a Go type is not closed. Every union in this design is a string discriminator resolved through a fixed `map` with a test asserting the exact key set.
- **The adapter is never trusted to describe its own run.** The `completeness` block is synthesized by `execd` from out-of-band process evidence (exit code, deadline, envelope metadata, byte counters), not read from adapter output. Only the `findings` array comes from the adapter.

Implements the proposal's Approach §1–§6. Capability mapping: `a2a-edge` → `internal/a2a`, `review-diff-skill` → `internal/skill/reviewdiff`, `execd-job-protocol` → `internal/jobspec` + `internal/{orchd,execd}/socket`, `session-worktree` → `internal/execd/worktree`, `sandboxed-execution` → `internal/execd/sandbox` + `internal/execd/secret` + `cmd/krein-shim`, `claude-cli-adapter` → `internal/adapter/claude`.

Module path resolved from the configured `origin` remote: **`github.com/DeRep311/krein`**. Go directive `go 1.25` (floor required by `a2a-go`; installed toolchain is go1.26.8).

---

## Architecture Decisions

### Decision: orchd↔execd wire encoding is length-prefixed JSON with a hand-written discriminated decoder

**Choice**: 4-byte big-endian `uint32` length prefix + `encoding/json` body, max frame 4 MiB. The job union is a `{"op": "<string>", "payload": {…}}` envelope resolved through a fixed `map[Op]decoder` in `internal/jobspec/codec.go`. Every payload decoder uses `json.Decoder` + `DisallowUnknownFields()`, because `execd-job-protocol` "Closed Job-Spec Enum" requires `execd` to reject not only an unknown job type but also any message *carrying unrecognized payload fields*. One job per connection: request frame → result frame → close.

**Wire spelling of the three job types.** `execd-job-protocol` fixes the enum as exactly `AllocateWorktree`, `RunReviewAdapter`, `ReleaseWorktree`. This design uses snake_case tokens on the wire — `allocate_worktree`, `run_review_adapter`, `release_worktree` — matching every other JSON key in the protocol, with the Go constants named after the spec (`OpAllocateWorktree`, …). The mapping is one-to-one and total; the enum's membership and closedness are unchanged. This is a flagged encoding decision, not a change to the three job identities.

**Alternatives considered**:
- **`encoding/gob`** — Go-native and cheap to wire up.
- **Protocol Buffers with `oneof`** — a union genuinely closed by generated code.
- **Multiplexed connection with request ids** — one long-lived socket carrying concurrent jobs.

**Rationale**:
- `gob` is rejected on a security ground, not an ergonomic one. Decoding into an interface-typed field requires `gob.Register`, and the **decoder** selects the concrete type from a name on the wire — an open-set decode sitting exactly on the trust boundary this slice exists to prove. It is also opaque to `tcpdump`/audit and unreadable from any non-Go tool, which makes the "no arbitrary-command verb" invariant harder to demonstrate to a reviewer than to assert.
- Protobuf's `oneof` is the technically strongest closure, but it adds `protoc` + a codegen step + a `buf`-style lint gate to a repository that currently has zero Go files. For a three-member enum the codegen tax buys nothing that a 20-line table and one exhaustive test do not. It is the correct upgrade if the enum ever grows past ~10 operations or gains a second-language consumer; that is recorded as the migration trigger, not deferred silently.
- JSON keeps the socket bytes human-inspectable, which matters because "the enum is closed" is a claim reviewers must be able to check by reading a frame dump.
- One-job-per-connection removes the multiplexer entirely: cancellation is `conn.Close()`, `SO_PEERCRED` is evaluated per job, and a stuck job cannot head-of-line-block another. Connection setup on a Unix socket is microseconds; with a concurrency cap of 4 there is nothing to amortise.

**Closure enforcement (the actual invariant)**:

```go
// internal/jobspec/op.go
type Op string

const (
	OpAllocateWorktree Op = "allocate_worktree"
	OpRunReviewAdapter Op = "run_review_adapter"
	OpReleaseWorktree  Op = "release_worktree"
)

// decoders is the ONLY path from wire bytes to a Job. Unknown ops fail closed.
var decoders = map[Op]func(json.RawMessage) (Job, error){
	OpAllocateWorktree: decodeAllocateWorktree,
	OpRunReviewAdapter: decodeRunReviewAdapter,
	OpReleaseWorktree:  decodeReleaseWorktree,
}

func AllOps() []Op // sorted; used by tests and by the dispatch exhaustiveness check
```

Three tests keep it closed:
- `TestOpSetIsExactlyThree` — `AllOps()` deep-equals the literal expected slice; adding an op without editing the test fails.
- `TestUnknownOpRejected` — a table of hostile `op` values (`"exec"`, `"run"`, `"ALLOCATE_WORKTREE"`, `""`, `"allocate_worktree\x00exec"`) all return `ErrUnknownOp` and never reach dispatch.
- `TestNoJobCarriesFreeformExecution` — reflection over every registered job struct asserts every field type is in an allowlist of typed scalars/slices, and that no field is a bare `[]string`, `map[string]string`, `any`, or a type named `*Cmd`. This is the structural form of "there is no arbitrary-command verb".

### Decision: socket peer authentication is `SO_PEERCRED` **and** filesystem mode, with no shared secret

**Choice**: socket at `<state-dir>/run/execd.sock`; parent directory mode `0700` owned by the execd uid; socket file chmod `0600` after `net.Listen`. On every `Accept`, `execd` reads `SO_PEERCRED` (`unix.GetsockoptUcred`) and requires `ucred.Uid == cfg.AllowedPeerUID` (default: the execd process uid). Mismatch → close immediately, one audit record, no frame read.

**Alternatives considered**: filesystem mode alone; `SO_PEERCRED` alone; a shared handshake secret on the socket.

**Rationale**: filesystem mode alone is portable but silently wrong if the socket is created before the chmod or if a parent directory is loosened later; `SO_PEERCRED` alone is authoritative but leaves the socket connectable by anyone who can reach the path, which turns a mode mistake into an audit-log flood instead of a closed door. Together they fail independently. A shared handshake secret is rejected outright: it adds a second long-lived secret to provision, rotate and leak, to re-prove something the kernel already proves for free on a local socket.

### Decision: the A2A input `patch_content` is a UTF-8 string; the jobspec `patch_content` is opaque bytes

**Choice**: `review-diff` accepts `patch_content` as a JSON string (a UTF-8 unified diff, ≤ 1 MiB of raw bytes). `jobspec.RunReviewAdapterJob.PatchContent` is `[]byte` (base64 in JSON). The field name `patch_content` is fixed by `review-diff-skill` "Accepted Request Inputs" and is carried unchanged through both layers so there is no rename to track across the trust boundary.

**Alternatives considered**: base64 at both layers; string at both layers.

**Rationale**: a binary diff carries no review value and would force a second parsing surface at the edge, so rejecting non-UTF-8 at ring 1 is a real reduction in attack surface, not a limitation. Below ring 1 the patch is never interpreted again — only written to an fd — so `[]byte` is the honest type and base64 avoids re-validating UTF-8 on a path where it no longer matters.

### Decision: the sandbox has unrestricted network egress (`--share-net`) in this slice

**Choice**: the sandbox runs with `--share-net`, giving the adapter process the host's unrestricted network access. **This is a closed decision, taken by the user for this slice**, not an open question.

**Alternatives considered**: a constrained egress path — a private network namespace (slirp or a veth pair) plus a filtering forward proxy that permits only the model API endpoint, enforced by a preflight check that the proxy is reachable and that direct egress is not.

**Rationale**: `sandboxed-execution` fixes `--unshare-all --share-net` as the isolation profile, and the model API is the sandbox's only *intended* egress. The constrained alternative is roughly one extra package, one extra preflight check, and a TLS-interception or SNI-allowlist decision that this slice does not otherwise need. Slice 1 exists to prove the trust-boundary and sandbox-isolation claims; adding an egress-filtering subsystem would enlarge the thing being proved.

**Accepted risk (documented, not pending)**: the sandbox binds a read-only checkout of a **locally registered repository at `base_commit`**, so the adapter can read the whole local tree at that commit, not only the submitted patch. The tool allowlist denies `Bash`, `WebFetch` and `WebSearch`, so the adapter has no *tool* with which to exfiltrate — but a compromised or prompt-injected adapter **process** is not constrained by its own tool allowlist, and with `--share-net` that process has free egress. The blast radius of this slice is therefore: one model API key, plus read access to any repository an operator explicitly registered, exfiltrable by a process-level compromise of the CLI. Registering a `repo_id` is an explicit local act of trust and must be treated as such. Constraining egress to the model API endpoint is recorded as a hardening step alongside seccomp, not as an unresolved question.

### Decision: `completeness` is synthesized by `execd`, never read from adapter output

**Choice**: the `--json-schema` handed to the `claude` CLI describes **only** the `findings` array. `execd` computes the `completeness` block from process evidence: exit code, whether the deadline fired, the CLI envelope's turn count / stop reason / permission-denial metadata, and `execd`'s own stdout byte counter. `internal/adapter/claude/outcome.go` is a pure function `Outcome(env Envelope, ev ProcEvidence) (Completeness, FailureCode)`.

**Alternatives considered**: extend the adapter-facing schema so the model fills `completeness` itself; trust the CLI envelope's own success flag.

**Rationale**: an adapter that has been prompt-injected by the patch under review is exactly the adapter most motivated to report `status: "complete"` while withholding findings. Any self-reported completeness is a claim by the thing whose reliability is in question. Out-of-band evidence cannot be authored by the model. This is the single most load-bearing correction the design makes to the naive reading of the proposal's §5.

**Consequence (fail-loud on unverifiable evidence)**: if the installed CLI's JSON envelope does **not** expose a permission-denial field, `execd` cannot prove the run was undenied. It MUST then emit `status: "partial"` with `reason_code: "denial_visibility_unavailable"` rather than default to `complete`. Absence of evidence is never evidence of completeness.

### Decision: partial results are delivered as A2A `completed` with a mandatory `completeness` block, not as `failed`

**Choice**:

| Outcome class | A2A task state | Artifact | `completeness.status` |
|---|---|---|---|
| Clean run | `completed` | present | `complete` |
| Degraded run (denials, turn exhaustion, truncation, denial-visibility gap) | `completed` | present | `partial` |
| No usable result | `failed` | none — structured error | — |

The final task status message additionally carries a human-readable line `partial: <reason_code>` so a consumer that reads prose and ignores the data part still sees it.

**Alternatives considered**: map every degraded run to `failed`; add a bespoke `completed_with_warnings` state.

**Rationale**: a review that found eight real issues but was denied one tool is more valuable delivered than discarded, and `failed` carries no artifact in A2A — mapping partial to `failed` would throw the findings away. A bespoke state is not in the A2A vocabulary and would break interop with a second brain running the stock SDK. The proposal's actual requirement — "never a clean `completed` carrying a quietly smaller findings list" — is met by making `completeness` a **required** field of the result schema with a two-value `status` enum: a schema-conforming consumer cannot decode the result without materialising the partial flag.

### Decision: payloads reach the sandbox via `memfd`-backed file descriptors, not pipes

**Choice**: `execd` creates four anonymous in-memory files with `unix.MemfdCreate(name, unix.MFD_CLOEXEC)`, writes the payload, seeks to 0, and passes them as `cmd.ExtraFiles` (Go maps `ExtraFiles[0]`→fd 3, `[1]`→fd 4, `[2]`→fd 5, `[3]`→fd 6).

| fd | content | in-sandbox destination | bwrap operation |
|---|---|---|---|
| 3 | patch under review | `/work/input/patch.diff` | `--ro-bind-data` |
| 4 | focus text (empty file when absent) | `/work/input/focus.txt` | `--ro-bind-data` |
| 5 | **adapter API key**, raw bytes, no trailing newline | `/run/krein/adapter.env` | `--file` |
| 6 | embedded `findings.schema.json` | `/work/input/findings.schema.json` | `--ro-bind-data` |

**Why two different bwrap operations.** `--file <fd> <dest>` copies the fd into a *writable* regular file on the destination tmpfs; `--ro-bind-data <fd> <dest>` copies it into a file that is then bind-mounted **read-only**. The distinction is load-bearing in both directions:

- `sandboxed-execution` "Read-Only Patch File Mount" requires that the adapter *cannot modify* the patch file, so fds 3, 4 and 6 use `--ro-bind-data`. A plain `--file` onto the `/work/input` tmpfs would leave them writable and would not satisfy that requirement.
- `sandboxed-execution` "File-Descriptor-Based Secret Injection" names `--file <fd> /run/krein/adapter.env` explicitly *and* requires the shim to unlink that path. Unlinking a bind mountpoint fails with `EBUSY`, so the secret must be a plain writable-tmpfs file. fd 5 therefore stays `--file`, exactly as the specification fixes it.

fd 6 exists because the pinned argv passes `--json-schema /work/input/findings.schema.json` to the CLI; that path has to be materialised inside the sandbox by something. It is the schema `//go:embed`ed in `internal/skill/reviewdiff`, so the adapter is always validated against the same bytes the design ships.

**Alternatives considered**: `os.Pipe` pairs with writer goroutines; a temporary file on disk; `--setenv`/argv.

**Rationale**:
- argv and `--setenv` are already excluded by the proposal (`/proc/<pid>/cmdline` and `/proc/<pid>/environ` are readable by same-uid processes).
- A temp file on disk means the key touches a filesystem, survives a crash, and is subject to backup/indexing.
- Pipes work but introduce a **real deadlock**: `bwrap` reads `--file`/`--ro-bind-data` fds sequentially in argv order, and a 1 MiB patch exceeds the 64 KiB pipe buffer, so `execd` must run a writer goroutine per pipe and get the close ordering right. A `memfd` is already fully populated before `bwrap` starts, so there is no writer, no ordering, and no deadlock class to test for. It also never appears in any filesystem namespace.

### Decision: the in-sandbox entry point is a static Go shim binary, not a shell script

**Choice**: a third binary, `cmd/krein-shim`, built `CGO_ENABLED=0`, bind-mounted read-only into the sandbox. It:
1. reads `--secret-file` to `[]byte`,
2. `os.Remove`s that path (it lives on a private tmpfs, so it is gone from the namespace),
3. builds an explicit env slice — exactly the four variables it inherited from the sandbox (`PATH`, `HOME`, `LANG`, `TERM`), plus `--secret-env=<NAME>` set to the bytes read — with nothing else inherited and nothing else added,
4. `syscall.Exec`s the argv that follows `--`.

**Alternatives considered**: `/bin/sh -c 'export K=$(cat …); exec claude …'`; having `claude` read the key from a file directly.

**Rationale**: a shell inside the sandbox is an interpreter the design does not otherwise need, and `$(cat …)` introduces word-splitting and trailing-newline semantics on a secret. A static Go binary has no dynamic loader dependency, no interpreter, and a deterministic, unit-testable argv contract. `claude` reading the key from a file directly would leave the key readable for the whole process lifetime; `syscall.Exec` after unlink means the file is gone before the adapter starts, and the key exists only in the adapter's own process environment.

**Flagged**: `cmd/krein-shim` is a **third binary** that the proposal's Affected Areas table did not list. It is a design-level addition, not a scope expansion — the proposal already required "a tiny in-sandbox entry shim"; this decides what it is made of.

### Decision: input validation at ring 1 is strict Go decoding, with JSON Schema as the adapter-facing contract only

**Choice**: the edge validates with `json.Decoder` + `DisallowUnknownFields()` into a closed struct, followed by explicit bound checks in `internal/skill/reviewdiff/contract.go`. The embedded `findings.schema.json` is handed to the CLI via `--json-schema` and is the **contract**, not the validator. A conformance test table drives one shared fixture corpus through both paths and asserts they agree on accept/reject.

**Alternatives considered**: `santhosh-tekuri/jsonschema/v6` as the enforcing validator at the edge.

**Rationale**: the edge parses bytes from an untrusted remote brain. Every dependency on that path is CVE surface and a supply-chain root. A closed struct with `DisallowUnknownFields` is ~40 lines, has no dependency, and gives the strongest property this design needs for free: a request **cannot even contain** a `tool`, `skill`, `adapter` or `command` field without being rejected — the anti-confused-deputy invariant enforced by the decoder rather than by a downstream check someone can forget. The drift risk (two sources of truth) is real and is paid for by the conformance test, which is cheaper than owning a schema-engine dependency at the trust boundary.

### Decision: lease liveness uses boot-id + process start-time, not bare PID existence

**Choice**: a lease records `owning_pid` (name fixed by the `session-worktree` spec), plus two additive disambiguators — `owner_boot_id` (`/proc/sys/kernel/random/boot_id`) and `owner_start_ticks` (field 22 of `/proc/<pid>/stat`). A lease is "live" only when all three match the currently running process at that pid.

**Alternatives considered**: "pid exists" (the proposal's §4 wording).

**Rationale**: bare pid existence is unsound in both directions. After a reboot the pid is meaningless, and after pid-space wraparound an unrelated process can occupy it — making a reconciliation sweep either skip a genuine orphan forever, or, worse, treat a stale lease as live. Start-time is the standard disambiguator and costs one `/proc` read. Additionally `execd` takes an exclusive `flock` on `<state-dir>/execd.lock` at startup so two instances can never share a state directory and reconcile each other's live sessions.

### Decision: masking `<worktree>/.git` with an empty read-only bind, keeping `git worktree add`

**Choice**: `--ro-bind /dev/null /work/repo/.git`. In a linked worktree `.git` is a **regular file** containing `gitdir: …`, not a directory, so a tmpfs cannot be mounted over it; binding a file over a file is the correct operation.

**⚠ Flagged deviation from `sandboxed-execution`.** That specification's "Read-Only Worktree and Masked Git Internals" requirement says `<worktree>/.git` MUST be masked "with an empty tmpfs (`--tmpfs`)". For a linked worktree that instruction is **unimplementable**: a tmpfs mount requires a directory mountpoint, and `git worktree add` creates `.git` as a regular file. This design therefore masks it with a read-only bind of `/dev/null` instead. The requirement's *observable* obligation — its scenario, "the adapter cannot read the host repository's `.git` object store, hooks, or config through the mounted worktree" — is met in full, and has its own RED test (`git -C /work/repo rev-parse` must fail from inside the sandbox). This is a proposed spec amendment (mechanism only, not effect), reported to the orchestrator rather than silently absorbed.

**Alternatives considered**: `git archive <oid> | tar -x` into a plain directory (no `.git` to mask at all); copying the tree.

**Rationale**: `git archive` is genuinely attractive — a pristine tree with no gitdir pointer and no worktree registration to orphan. It is rejected because the `session-worktree` capability, its lease model, and its `git worktree list`-based observability are settled in the proposal, and because a checkout is cheaper than an archive+extract for large repos. The masking is recorded as an implementation-verification item with its own RED test (`git -C /work/repo rev-parse` must fail from inside the sandbox), because "bind a file over a file inside a read-only bind mount" is the one bwrap behaviour here that must be proven rather than assumed.

### Decision: prompt injection is contained by the kernel and the allowlist, not by filtering

**Choice**: no attempt is made to sanitise the patch or the focus text for injection strings. Containment is: no `Bash`/`Write`/`Edit`/`WebFetch`/`WebSearch`/`Task` tool, a read-only bind, a masked home, a masked gitdir, and schema-validated output.

**Rationale**: the patch under review *is* adversarial content by definition — filtering it would corrupt the thing being reviewed, and no filter is sound against an LLM. The reachable consequence of a successful injection is bounded to "returns misleading findings", which is a correctness risk the requester already accepts by asking a model for a review. Escalation beyond that requires a tool the adapter does not have or a filesystem path the kernel does not expose.

---

## Data Flow

### End-to-end

```
brain A                orchd                        execd                      bwrap + claude
   │                     │                            │                              │
   │ POST /a2a           │                            │                              │
   │ Bearer <token>      │                            │                              │
   │ message/send ──────▶│                            │                              │
   │                     │ 1 MaxBytesReader 2 MiB     │                              │
   │                     │ 2 constant-time token cmp  │                              │
   │                     │ 3 audit: task_accepted     │                              │
   │                     │ 4 skill id == "review-diff"│                              │
   │                     │ 5 strict decode DataPart   │                              │
   │                     │   (unknown field → reject) │                              │
   │                     │                            │                              │
   │◀── task submitted ──│                            │                              │
   │                     │─ conn 1: AllocateWorktree ▶│                              │
   │                     │                            │ SO_PEERCRED                  │
   │                     │                            │ registry[repo_id] → abs path │
   │                     │                            │ cat-file -e <oid>            │
   │                     │                            │ cap check                    │
   │                     │                            │ lease write + fsync          │
   │                     │                            │ git -C <abs> worktree add    │
   │                     │◀─ session_id, oid ─────────│                              │
   │                     │                            │                              │
   │                     │─ conn 2: RunReviewAdapter ▶│                              │
   │                     │                            │ 4× memfd (patch/focus/key/   │
   │                     │                            │           findings schema)   │
   │                     │                            │ bwrap argv (pure fn)         │
   │                     │                            │──── exec, fds 3,4,5,6 ──────▶│
   │                     │                            │                    shim: read key,
   │                     │                            │                    unlink, exec claude
   │                     │                            │◀─── stdout JSON envelope ────│
   │                     │                            │ parse findings (strict)      │
   │                     │                            │ Outcome(env, procEvidence)   │
   │                     │◀─ findings + completeness ─│                              │
   │                     │                            │                              │
   │                     │─ conn 3: ReleaseWorktree ──▶ git worktree remove --force  │
   │                     │◀─ removed ─────────────────│ lease delete                 │
   │                     │ audit: task_completed      │                              │
   │◀── artifact ────────│                            │                              │
```

`ReleaseWorktree` is issued from a `defer` in the orchd executor and is idempotent, so a failure on the adapter step still releases. If orchd dies between allocate and release, the lease survives and the reconciliation sweep reclaims it.

### Failure fan-out

```
                    ┌─ unauthenticated ─────┐
                    ├─ unknown_skill ───────┤
  ring 1 rejects ───┼─ invalid_params ──────┼──▶ JSON-RPC error, NO task created,
                    ├─ patch_too_large ─────┤     audit: auth_rejected / request_rejected
                    └─ edge_unavailable ────┘

                    ┌─ repo_unavailable ──────────┐
                    ├─ concurrency_limit_exceeded ┤
  ring 2 rejects ───┼─ worktree_unavailable ──────┼──▶ task → failed, audit: task_failed
                    ├─ sandbox_unavailable ───────┤
                    └─ adapter_unavailable ───────┘

                    ┌─ adapter_failed ────────────┐
  ring 3 fails  ────┼─ execution_timeout ─────────┼──▶ task → failed (no artifact)
                    └─ invalid_output_schema ─────┘

                    ┌─ tools_denied ────────────────────┐
  ring 3 degrades ──┼─ turns_exhausted ─────────────────┼──▶ task → completed,
                    ├─ truncated ───────────────────────┤     completeness.status = partial
                    └─ denial_visibility_unavailable ───┘
```

### Sequence: worktree lease lifecycle and crash recovery

```
happy path                          crash path
──────────                          ──────────
write lease.tmp                     write lease.tmp
fsync file                          fsync file
rename → lease.json                 rename → lease.json
fsync dir                           fsync dir
git worktree add                    git worktree add
  … adapter runs …                    … execd is SIGKILLed …
git worktree remove --force
git worktree prune                  (restart) reconcile():
unlink lease.json                     read lease.json
                                      boot_id mismatch OR pid dead OR
                                        start_ticks mismatch OR age > TTL
                                      path under managed root?  ── no ──▶ report, keep
                                                                  yes
                                      git worktree remove --force
                                      RemoveAll(path) if still present
                                      git worktree prune
                                      unlink lease.json
```

---

## File Changes

| File | Action | Description |
|---|---|---|
| `go.mod` | Create | `module github.com/DeRep311/krein`, `go 1.25`, pinned `a2aproject/a2a-go` + `golang.org/x/sys` |
| `go.sum` | Create | Checksums for the two direct dependencies and their transitive set |
| `cmd/orchd/main.go` | Create | Flag/env config, `slog` setup, A2A HTTP server, execd socket client, graceful shutdown |
| `cmd/execd/main.go` | Create | Config load, secret load + mode check, preflight, `flock`, startup reconcile, socket listen, sweep timer |
| `cmd/krein-shim/main.go` | Create | `CGO_ENABLED=0` in-sandbox entry point: read secret file, unlink, build env, `syscall.Exec` |
| `internal/a2a/server.go` | Create | `a2asrv` handler + JSON-RPC binding, `http.MaxBytesReader`, route wiring |
| `internal/a2a/agentcard.go` | Create | AgentCard: one skill (`review-diff`), bearer security scheme, no repo ids |
| `internal/a2a/auth.go` | Create | Bearer extraction, `subtle.ConstantTimeCompare`, token file loading with mode check |
| `internal/a2a/executor.go` | Create | `AgentExecutor`: capability lookup → param bind → allocate/run/release → artifact, `defer` release |
| `internal/a2a/audit/audit.go` | Create | Append-only JSONL writer, HMAC token fingerprint, monotonic `seq`, redaction rules |
| `internal/a2a/capability/registry.go` | Create | Closed `map[SkillID]Capability`, built at init, no dynamic registration path |
| `internal/a2a/capability/reviewdiff.go` | Create | Binds a validated `reviewdiff.Request` to the pre-registered capability |
| `internal/skill/reviewdiff/contract.go` | Create | `Request`, `Result`, `Finding`, `Completeness` Go types + bound checks |
| `internal/skill/reviewdiff/schema.go` | Create | `//go:embed` the three schema files; `FindingsSchema()` for `--json-schema` |
| `internal/skill/reviewdiff/request.schema.json` | Create | A2A input schema, `additionalProperties: false` |
| `internal/skill/reviewdiff/findings.schema.json` | Create | Adapter-facing schema — **findings only**, no completeness |
| `internal/skill/reviewdiff/result.schema.json` | Create | Edge-facing schema — findings + **required** completeness |
| `internal/jobspec/op.go` | Create | Closed `Op` enum, `AllOps()` |
| `internal/jobspec/job.go` | Create | Three job structs, three result structs, `AdapterID` enum |
| `internal/jobspec/codec.go` | Create | Length-prefixed framing, discriminated decode table, frame bounds |
| `internal/jobspec/failure.go` | Create | `FailureCode` enum, `Completeness`, `ReasonCode` |
| `internal/orchd/config/config.go` | Create | Listen addr, socket path, token path, timeouts |
| `internal/orchd/socket/client.go` | Create | Dial, round-trip one job per connection, context cancellation → close |
| `internal/execd/config/config.go` | Create | State dir, repo registry, secret path, shim path, budgets, caps, peer uid |
| `internal/execd/socket/server.go` | Create | Listen, chmod `0600`, `SO_PEERCRED` check, frame read, dispatch |
| `internal/execd/dispatch/dispatch.go` | Create | Exhaustive `switch` over `Op` with a compile-adjacent completeness test |
| `internal/execd/repo/registry.go` | Create | `repo_id` → absolute path (closed map), commit existence via `git cat-file -e` |
| `internal/execd/worktree/manager.go` | Create | Allocate/release, concurrency cap, per-repo mutex, session id generation |
| `internal/execd/worktree/lease.go` | Create | Lease record, atomic durable write, boot-id/start-ticks liveness |
| `internal/execd/worktree/reconcile.go` | Create | Startup + periodic sweep, managed-root confinement, corrupt-lease quarantine |
| `internal/execd/sandbox/profile.go` | Create | Pure `BwrapArgv(Profile) []string` |
| `internal/execd/sandbox/memfd.go` | Create | `memfd`-backed payloads, `ExtraFiles` ordering |
| `internal/execd/sandbox/run.go` | Create | Exec, deadline, stdout/stderr caps, process-group kill, `ProcEvidence` |
| `internal/execd/sandbox/preflight.go` | Create | `bwrap`/`git`/`claude`/shim presence, version capture, unprivileged-userns check |
| `internal/execd/secret/secret.go` | Create | Redacting `Secret` type, mode-checked load via opened fd |
| `internal/adapter/claude/argv.go` | Create | Pure `Argv(Spec) []string` — the pinned invocation |
| `internal/adapter/claude/envelope.go` | Create | CLI JSON envelope struct (strict decode) |
| `internal/adapter/claude/parse.go` | Create | Findings extraction + bound checks |
| `internal/adapter/claude/outcome.go` | Create | Pure `Outcome(Envelope, ProcEvidence) (Completeness, FailureCode)` |
| `internal/ids/ids.go` | Create | 128-bit `crypto/rand` session ids, base32 lowercase, validation regex |
| `internal/logging/logging.go` | Create | `slog` handler with a redacting `ReplaceAttr` |
| `internal/testsupport/gitfixture/fixture.go` | Create | Throwaway git repo builder for tests |
| `internal/testsupport/fakeadapter/main.go` | Create | Deterministic stand-in for `claude`; performs in-sandbox isolation probes |
| `.github/workflows/ci.yml` | Create | `gofmt -l .`, `go vet ./...`, `go build ./...`, `go test ./...` with `KREIN_REQUIRE_SANDBOX=1` |

No files are modified or deleted — the repository contains no Go source.

---

## Interfaces / Contracts

### A2A skill input — `review-diff`

Delivered as a `Message` with exactly one `DataPart`. A `TextPart`-only message is rejected: the contract is structurally typed, never parsed out of prose.

```json
{
  "schema_version": "krein.review-diff.request/v1",
  "repo_id": "krein",
  "base_commit": "5f2a0c1e9b7d4a3f8c6e2b1d0a9f8e7c6b5a4d3c",
  "patch_content": "diff --git a/x.go b/x.go\n@@ …",
  "focus": "concurrency and error handling"
}
```

| Field | Type | Required | Bound / validation |
|---|---|---|---|
| `schema_version` | string | yes | must equal `krein.review-diff.request/v1` |
| `repo_id` | string | yes | `^[a-z0-9][a-z0-9._-]{0,63}$`; resolved through a closed map, never used as a path |
| `base_commit` | string | yes | `^[0-9a-f]{40}$` or `^[0-9a-f]{64}$` — full oid only |
| `patch_content` | string | yes | valid UTF-8, `1 ≤ len(bytes) ≤ 1048576` |
| `focus` | string | no | valid UTF-8, `≤ 4096` bytes |

`repo_id`, `base_commit` and `patch_content` are the three inputs fixed by `review-diff-skill`. `schema_version` and `focus` are design-level additions, flagged in "Decisions This Design Resolved" below.

`additionalProperties: false`. Any unknown key — notably `tool`, `skill`, `adapter`, `command`, `cmd`, `args` — is a hard reject at the decoder.

### A2A skill output

Delivered as an `Artifact` with one `DataPart`.

```json
{
  "schema_version": "krein.review-diff.findings/v1",
  "findings": [
    {
      "id": "f-0001",
      "severity": "error",
      "confidence": "medium",
      "title": "Unsynchronised map write in the sweep goroutine",
      "description": "…",
      "file_path": "internal/execd/worktree/reconcile.go",
      "line_start": 88,
      "line_end": 94,
      "suggested_fix": "…"
    }
  ],
  "completeness": {
    "status": "partial",
    "tools_denied": ["Bash"],
    "turns_exhausted": false,
    "truncated": false,
    "reason_code": "tools_denied"
  }
}
```

| Field | Type | Required | Source | Notes |
|---|---|---|---|---|
| `findings[].file_path` | string | yes | spec | repo-relative, `..` and absolute rejected. **Non-nullable** |
| `findings[].line_start` | int | yes | spec | ≥ 1. **Non-nullable** |
| `findings[].line_end` | int | yes | spec | ≥ `line_start`. **Non-nullable** |
| `findings[].severity` | enum | yes | spec | `info\|warning\|error\|critical` |
| `findings[].title` | string | yes | spec | ≤ 120 bytes |
| `findings[].description` | string | yes | spec | ≤ 4000 bytes |
| `findings[].suggested_fix` | string \| null | no | spec (`MAY`) | ≤ 2000 bytes |
| `findings[].id` | string | yes | **design addition** | `f-%04d`, assigned by `execd`, not by the adapter |
| `findings[].confidence` | enum | yes | **design addition** | `high\|medium\|low` — kept separate from severity because model findings need a calibration axis |
| `completeness` | object | **yes** | spec | required — this is the mechanism that makes partial results unmissable |
| `completeness.status` | enum | yes | spec | `complete\|partial` |
| `completeness.tools_denied` | string[] | yes | spec | may be empty; never null |
| `completeness.turns_exhausted` | bool | yes | spec | |
| `completeness.truncated` | bool | yes | spec | |
| `completeness.reason_code` | string \| null | yes | **design addition** | machine-readable degradation reason |
| `schema_version` | string | yes | **design addition** | `krein.review-diff.findings/v1` |

The field names, the four-value `severity` enum, and the unconditional presence of `file_path`, `line_start` and `line_end` are fixed by `review-diff-skill` "Structured Findings Schema" and are reproduced here exactly. Rows marked **design addition** are additive fields this design introduces beyond the spec's named set; each is flagged in "Decisions This Design Resolved" below.

**Consequence of unconditional anchoring.** Because the spec requires `file_path`, `line_start` and `line_end` on *every* finding, a whole-change observation with no single anchor is not representable in v1. The adapter-facing `findings.schema.json` marks all six spec fields `required`, so an unanchored finding is schema-invalid at the adapter boundary and — per `claude-cli-adapter` "Schema-Invalid Output Maps to a Failed Task" — fails the task with `invalid_output_schema` rather than being silently dropped. The prompt therefore instructs the adapter to anchor every finding to a concrete line range. This is a real constraint of the locked spec, recorded here so it is not rediscovered during apply.

`findings` is capped at 200 entries; beyond that `execd` truncates and sets `truncated: true`, `status: partial`.

A deliberate omission: `category`/`cwe` are not in v1. Adding an optional field later is backward-compatible; guessing a taxonomy now is not.

### orchd↔execd job types

```go
package jobspec

type SessionID string // execd-generated, ^[a-z2-7]{26}$
type RepoID string
type AdapterID string

const AdapterClaude AdapterID = "claude" // closed enum; exactly one member in this slice

type Envelope struct {
	SchemaVersion string          `json:"schema_version"` // "krein.jobspec/v1"
	Op            Op              `json:"op"`
	Payload       json.RawMessage `json:"payload"`
}

type AllocateWorktreeJob struct {
	RepoID     RepoID `json:"repo_id"`
	BaseCommit string `json:"base_commit"`
}
type AllocateWorktreeResult struct {
	SessionID   SessionID `json:"session_id"`
	ResolvedOID string    `json:"resolved_oid"`
	// WorktreePath is deliberately absent: orchd has no filesystem authority
	// and the path must never reach an A2A artifact.
}

type RunReviewAdapterJob struct {
	SessionID    SessionID `json:"session_id"`
	Adapter      AdapterID `json:"adapter"`
	PatchContent []byte    `json:"patch_content"`   // base64; ≤ 1 MiB decoded
	Focus        string    `json:"focus,omitempty"` // ≤ 4 KiB; reaches the sandbox by fd, never argv
	Budget       Budget    `json:"budget"`
}
type Budget struct {
	WallClockMS    int64 `json:"wall_clock_ms"`
	MaxTurns       int   `json:"max_turns"`
	MaxOutputBytes int64 `json:"max_output_bytes"`
}
type RunReviewAdapterResult struct {
	Findings     []Finding    `json:"findings"`
	Completeness Completeness `json:"completeness"`
}

type ReleaseWorktreeJob struct {
	SessionID SessionID `json:"session_id"`
}
type ReleaseWorktreeResult struct {
	Removed bool `json:"removed"` // false when already released — release is idempotent
}

type Result struct {
	SchemaVersion string          `json:"schema_version"`
	OK            bool            `json:"ok"`
	Payload       json.RawMessage `json:"payload,omitempty"`
	Failure       *Failure        `json:"failure,omitempty"`
}
type Failure struct {
	Code   FailureCode `json:"code"`
	Detail string      `json:"detail"` // execd-local diagnostics; NEVER forwarded verbatim to A2A
}
```

`AllocateWorktreeResult` withholding the worktree path is load-bearing: `orchd` cannot leak a host path into an artifact it does not have.

### Failure taxonomy

```go
type FailureCode string

// ring 1 — no task is created; JSON-RPC error
const (
	FailUnauthenticated FailureCode = "unauthenticated"
	FailUnknownSkill    FailureCode = "unknown_skill"
	FailInvalidParams   FailureCode = "invalid_params"
	FailPatchTooLarge   FailureCode = "patch_too_large"
	FailEdgeUnavailable FailureCode = "edge_unavailable"
)

// ring 2 — task → failed
const (
	FailRepoUnavailable          FailureCode = "repo_unavailable"
	FailConcurrencyLimitExceeded FailureCode = "concurrency_limit_exceeded" // spec: session-worktree
	FailWorktreeUnavailable      FailureCode = "worktree_unavailable"
	FailSandboxUnavailable       FailureCode = "sandbox_unavailable"
	FailAdapterUnavailable       FailureCode = "adapter_unavailable"
)

// ring 3 — task → failed
const (
	FailAdapterFailed       FailureCode = "adapter_failed"
	FailExecutionTimeout    FailureCode = "execution_timeout"     // spec: claude-cli-adapter
	FailInvalidOutputSchema FailureCode = "invalid_output_schema" // spec: claude-cli-adapter
	FailInternal            FailureCode = "internal"
)

// degradation reasons — task → completed, completeness.status = "partial"
type ReasonCode string

const (
	ReasonToolsDenied                ReasonCode = "tools_denied"
	ReasonTurnsExhausted             ReasonCode = "turns_exhausted"
	ReasonTruncated                  ReasonCode = "truncated"
	ReasonFindingsCapped             ReasonCode = "findings_capped"
	ReasonDenialVisibilityUnavailable ReasonCode = "denial_visibility_unavailable"
)
```

**Information-leak rule**: `repo_unavailable` is returned identically for "repo id not registered" and "base commit not present locally". Distinguishing them would let a remote brain enumerate the local repo registry and probe commit existence. The precise cause is recorded only in `execd`'s local log and in `Failure.Detail`, which `orchd` logs and drops rather than forwarding.

**Timeout honesty**: with `--output-format json` the CLI emits a single final object, so a wall-clock timeout yields *no* parseable output. `execution_timeout` therefore always maps to `failed`, never to `partial`. A "timed out but salvaged partial findings" outcome is unreachable in this slice and becomes reachable only with `--output-format stream-json`, which is explicitly out of scope. This is stated rather than papered over.

### Worktree lease record

`<state-dir>/leases/<session-id>.json`, mode `0600`, directory `0700`.

```go
type Lease struct {
	SchemaVersion   string    `json:"schema_version"` // "krein.lease/v1"
	SessionID       string    `json:"session_id"`
	TaskID          string    `json:"task_id"`        // audit correlation ONLY — never used in a path
	RepoID          string    `json:"repo_id"`
	RepoPath        string    `json:"repo_path"`
	BaseOID         string    `json:"base_oid"`
	WorktreePath    string    `json:"worktree_path"`
	OwningPID       int       `json:"owning_pid"` // name fixed by session-worktree spec
	OwnerBootID     string    `json:"owner_boot_id"`
	OwnerStartTicks uint64    `json:"owner_start_ticks"`
	CreatedAt       time.Time `json:"created_at"`
	TTLMS           int64     `json:"ttl_ms"`
}
```

Write protocol: marshal → write `<sid>.json.tmp` → `f.Sync()` → `os.Rename` → `fsync` the directory → *then* `git worktree add`. A crash at any point leaves either nothing or a complete lease; never a worktree without evidence.

`TaskID` is A2A-supplied and therefore attacker-influenced. It is recorded for correlation and is **never** interpolated into a filesystem path. Paths use only `SessionID`, which `execd` generates from `crypto/rand` and validates against `^[a-z2-7]{26}$` before every use.

### Audit log record

`<state-dir>/audit/inbound-<YYYYMMDD>.jsonl`, opened `O_APPEND|O_CREATE|O_WRONLY` mode `0600`, `fsync` after each record (volume is one to three records per task).

```json
{
  "schema_version": "krein.audit.inbound/v1",
  "ts": "2026-09-18T15:04:05.123456789Z",
  "seq": 4211,
  "event": "task_accepted",
  "remote_addr": "10.0.0.7:51234",
  "token_fp": "9f3ac71b",
  "token_valid": true,
  "skill_id": "review-diff",
  "task_id": "…",
  "session_id": "…",
  "repo_id": "krein",
  "base_commit": "5f2a0c1e…",
  "patch_bytes": 18422,
  "patch_sha256": "c1d2…",
  "outcome": null,
  "completeness_status": null,
  "failure_code": null,
  "duration_ms": null
}
```

Events: `auth_rejected`, `request_rejected`, `task_accepted`, `task_completed`, `task_failed`.

**Never recorded**: the bearer token, the patch body, the focus text, the adapter API key, finding `description` text, or the worktree path.

`token_fp` is `HMAC-SHA256(audit_key, presented_token)` truncated to 8 hex — **not** a bare hash. A bare truncated hash would let anyone who can read the audit log confirm a guessed token offline; keying it with a per-install random `audit_key` (32 bytes, generated on first start, mode `0600`) makes the log useless for that. It still correlates repeated attempts from the same wrong token, which is the only thing it exists for.

### Preflight contract (`execd` startup)

`execd` refuses to start — with a named diagnostic, not a runtime surprise mid-task — unless all of:

| Check | Failure message |
|---|---|
| `bwrap` on `PATH`, `--version` succeeds | `sandbox_unavailable: bubblewrap not found` |
| unprivileged user namespaces permitted (probe `bwrap --unshare-all --ro-bind /usr /usr -- /bin/true`) | `sandbox_unavailable: unprivileged userns denied` |
| `git` ≥ 2.20 with `worktree` support | `worktree_unavailable: git worktree unsupported` |
| `claude` resolves to a real file | `adapter_unavailable: claude CLI not found` |
| shim exists, is executable, not group/world writable | `sandbox_unavailable: shim unusable` |
| secret file is a regular file with `mode & 0o077 == 0` | `secret file has permissive mode %o` |
| A2A token file, same mode check | `token file has permissive mode %o` |
| `flock` on `<state-dir>/execd.lock` acquired | `another execd instance owns this state directory` |

### Configuration defaults (proposal Open Question 3, resolved)

| Setting | Default | Reasoning |
|---|---|---|
| `patch_max_bytes` | `1048576` (1 MiB) | fixed by the proposal |
| `focus_max_bytes` | `4096` | one screen of guidance; not a prompt channel |
| `http_body_max_bytes` | `2097152` (2 MiB) | 1 MiB patch + JSON-RPC/A2A envelope headroom |
| `frame_max_bytes` | `4194304` (4 MiB) | 1 MiB patch → ~1.34 MiB base64, ×3 headroom |
| `adapter_wall_clock` | `300s` | long enough for a 1 MiB diff review; short enough that a hang is caught within one sweep |
| `adapter_max_turns` | `30` | bounds cost and makes `turns_exhausted` observable rather than theoretical |
| `adapter_max_output_bytes` | `4194304` (4 MiB) | 200 findings × ~6 KiB worst case, ×3 headroom |
| `findings_max` | `200` | beyond this a review is not actionable; capping is honest and flagged |
| `session_ttl` | `30m` | 6× the adapter budget — a lease older than this is definitionally abandoned |
| `reconcile_interval` | `5m` | reclaims within one TTL even if every sweep but the last is missed |
| `concurrency_cap` | `4` | matches the proposal's "small fixed cap"; a leak becomes `concurrency_limit_exceeded`, not disk growth |
| `socket_deadline` | `wall_clock + 30s` | the socket must outlive the job it is carrying |

---

## Subprocess-Adapter Lifecycle (`claude`)

Required by `openspec/config.yaml` `rules.design`.

**Spawn.** `execd` builds four `memfd`s, then a single `*exec.Cmd` for `bwrap` with `ExtraFiles = [patchFD, focusFD, secretFD, schemaFD]`, `Dir` set to the state directory (the sandbox `--chdir` governs the child's cwd), and `SysProcAttr{Setpgid: true}` so the whole tree can be signalled. `cmd.Stdin` is `nil` (an explicit `/dev/null`, so the adapter can never block on input).

**Three distinct environments — explicitly, because this is easy to conflate.** `cmd.Env` and the `--setenv` lines in the argv below are *not* the same thing. There are three separate environments on this path:

| # | Environment | Set by | Contents |
|---|---|---|---|
| A | the host-side `bwrap` process env | Go, `cmd.Env` | **empty** — `[]string{}` |
| B | the sandbox env the shim inherits | `bwrap --clearenv` + `--setenv` | exactly `PATH`, `HOME`, `LANG`, `TERM=dumb` |
| C | the adapter's own process env | `cmd/krein-shim`, before `syscall.Exec` | B, plus exactly one added variable: the secret |

- **A is empty, not four entries.** `bwrap` itself needs no environment: `execd` invokes it by the absolute path resolved at preflight, and `--clearenv` discards whatever it was given anyway. Setting `cmd.Env` to an explicit, **non-nil** empty slice is required and is the whole point — in Go, `cmd.Env == nil` means *inherit the parent's environment*, so the difference between `nil` and `[]string{}` is the difference between leaking every one of `execd`'s variables to the sandbox boundary and leaking none. An earlier draft of this design described A as "an explicit four-entry slice", which conflated it with B; A carries nothing.
- **B is the environment `sandboxed-execution` "Environment Clearing" fixes** as exactly `PATH`, `HOME`, `LANG`, `TERM=dumb` — four entries, no more. It is produced solely by the `--setenv` lines in the argv, and it is what the "adapter observes exactly these four" scenario is about at the moment the shim starts.
- **C is B plus the secret**, added by the shim because `sandboxed-execution` "File-Descriptor-Based Secret Injection" requires the shim to export the key into the adapter's process environment. The key is never a `--setenv` value and never reaches B, which is precisely what keeps it out of `bwrap`'s own `/proc/<pid>/environ`.

**Handshake.** There is none, and that is deliberate. The adapter is a one-shot process with a pinned argv and a schema-constrained output contract; a handshake would be a second protocol surface to version and to attack. The equivalent of a handshake happens at `execd` startup (preflight: `bwrap --version`, `claude --version`, both captured into the run record so a version drift is visible in the audit trail).

**Stream.** There is no streaming in this slice. `stdout` is drained through an `io.LimitedReader` capped at `adapter_max_output_bytes` into a buffer; `stderr` is drained concurrently into a 64 KiB ring buffer (last-N bytes, so a chatty adapter cannot exhaust memory and a fatal message at the end is always retained). Both drains run in goroutines joined before `cmd.Wait()` returns, so no output is lost to a race.

**Teardown.** A `context.WithTimeout(adapter_wall_clock)` governs the run. On expiry `execd` sends `SIGTERM` to the process **group**, waits 5 s, then `SIGKILL` to the group. `--die-with-parent` is the backstop if `execd` itself dies. After `cmd.Wait()`, `execd` closes all four `memfd`s, records `ProcEvidence{ExitCode, Signal, DeadlineFired, StdoutBytes, StdoutCapped, Stderr, Duration}`, then parses. The worktree release is a separate job so that teardown of the adapter and teardown of the session are independently observable — a wedged adapter does not strand a worktree, and a failed release does not hide a successful review.

---

## The `bwrap` Invocation

`internal/execd/sandbox/profile.go` exposes a **pure function** `BwrapArgv(Profile) []string` so the entire invocation is golden-testable without executing anything. Order matters: bwrap applies operations sequentially onto the new root.

```
bwrap
  --die-with-parent
  --unshare-all
  --share-net                              # required: the model API is the only egress
  --new-session                            # no controlling TTY → no TIOCSTI injection
  --uid 65534 --gid 65534
  --clearenv
  --setenv PATH   /usr/local/bin:/usr/bin:/bin
  --setenv HOME   /home/reviewer
  --setenv LANG   C.UTF-8
  --setenv TERM   dumb
  # exactly four --setenv lines: sandboxed-execution fixes the sandbox env as
  # exactly PATH, HOME, LANG, TERM=dumb. Nothing else may be added here.
  --ro-bind /usr /usr
  --ro-bind-try /lib   /lib                # absent on usrmerge systems
  --ro-bind-try /lib64 /lib64
  --ro-bind-try /bin   /bin
  --ro-bind /etc/ssl /etc/ssl
  --ro-bind-try /etc/pki /etc/pki
  --ro-bind-try /etc/ca-certificates /etc/ca-certificates
  --ro-bind /etc/resolv.conf   /etc/resolv.conf
  --ro-bind-try /etc/nsswitch.conf /etc/nsswitch.conf
  --ro-bind <realpath of claude>      /usr/local/bin/claude
  --ro-bind <realpath of krein-shim>  /usr/local/bin/krein-shim
  --proc /proc
  --dev  /dev
  --tmpfs /tmp
  --tmpfs /home/reviewer                   # masked HOME: ~/.claude, ~/.ssh, ~/.gitconfig absent
  --tmpfs /run/krein                       # private tmpfs that will hold the secret
  --tmpfs /work/input
  --ro-bind <state>/worktrees/<session-id> /work/repo
  --ro-bind /dev/null /work/repo/.git      # mask the gitdir pointer file (see flagged deviation)
  --ro-bind-data 3 /work/input/patch.diff          # read-only: adapter cannot modify
  --ro-bind-data 4 /work/input/focus.txt           # read-only
  --ro-bind-data 6 /work/input/findings.schema.json # read-only; --json-schema target
  --file         5 /run/krein/adapter.env          # writable: the shim must unlink it
  --chdir /work/repo
  --
  /usr/local/bin/krein-shim
      --secret-file /run/krein/adapter.env
      --secret-env  ANTHROPIC_API_KEY
      --
      /usr/local/bin/claude
          -p
          --bare
          --output-format json
          --json-schema /work/input/findings.schema.json
          --permission-mode auto
          --permission-prompts none
          --max-turns 30
          <tool allowlist flags>           # exact spelling: verification item, see Open Questions
          <prompt>                          # fixed execd-authored constant; no remote bytes.
                                           # references /work/input/patch.diff by path
```

Properties this argv is required to have, each with a RED test:

- **No secret anywhere in argv.** `assert.NotContains(strings.Join(argv, "\x00"), string(secret))`, plus a runtime assertion reading `/proc/<pid>/cmdline` of the live `bwrap`.
- **No remote-supplied bytes in argv.** The patch and the focus text reach the sandbox as read-only files (fds 3 and 4); `repo_id` and `base_commit` never appear at all. The `<prompt>` argument *is* an argv element, and that is safe precisely because it is a **fixed constant template authored by `execd`** — it contains no remote bytes and only names `/work/input/patch.diff` and `/work/input/focus.txt` by path. `assert` that no argv element contains the patch bytes, the focus bytes, `repo_id` or `base_commit`, and that the prompt element equals the compiled-in constant exactly.
- **No shell.** `exec.Command("bwrap", argv...)` — never `sh -c`, never a joined string.
- **Deterministic.** `BwrapArgv` on a fixed `Profile` equals a golden file byte for byte.

`--ro-bind-try` is used for paths whose presence is distribution-dependent so that the profile is portable without becoming permissive: a missing `/lib64` is fine, a missing `/usr` is a hard failure.

**Seccomp is deliberately not set in this slice.** `--seccomp <fd>` would require compiling and maintaining a BPF program; the namespace + filesystem boundary is what this slice is proving. It is recorded as the first hardening step alongside the gVisor/microVM item the proposal already defers.

---

## Testing Strategy

Strict TDD per `openspec/config.yaml`: every row below is written RED first.

| Layer | What to test | Approach |
|---|---|---|
| Unit | `jobspec` op closure, unknown-op rejection, no-freeform-execution reflection check | Table tests over hostile `op` strings; reflection walk over registered job structs |
| Unit | Frame codec: length prefix, oversize frame, truncated frame, zero-length frame | Table over `bytes.Reader` fixtures |
| Unit | `reviewdiff` request validation | Shared fixture corpus; asserts strict-decode and JSON Schema agree on every case (`additionalProperties`, oid regex, byte bounds, non-UTF-8 `patch_content`) |
| Unit | Spec-conformance of both schemas | Asserts `severity` is exactly `{info,warning,error,critical}`; that `file_path`, `line_start`, `line_end`, `severity`, `title`, `description` are all `required` and non-nullable in `findings.schema.json`; and that the failure-code constants include the spec-fixed strings `execution_timeout`, `invalid_output_schema`, `concurrency_limit_exceeded` |
| Unit | Sandbox env layering | `BwrapArgv` emits exactly four `--setenv` lines (`PATH`,`HOME`,`LANG`,`TERM=dumb`); `cmd.Env` is non-nil and empty; the shim's constructed env is those four plus exactly one secret variable |
| Unit | `BwrapArgv` golden; `claude.Argv` golden | `testdata/*.golden` with a `-update` flag |
| Unit | `claude.Outcome` taxonomy | Table: `(envelope, procEvidence)` → `(Completeness, FailureCode)`, one row per code above, including `denial_visibility_unavailable` |
| Unit | Secret redaction | `fmt.Sprintf("%v"/"%s"/"%+v")`, `json.Marshal`, and a `slog` round trip all yield `[REDACTED]`; a fuzz-lite test asserts the raw bytes never appear in any handler output |
| Unit | Lease liveness | Injected `procReader` interface; cases: live, dead pid, pid reused with different `start_ticks`, boot-id mismatch, TTL exceeded |
| Unit | Audit record shaping | Asserts the token, patch, focus, key and worktree path never appear in a marshalled record |
| Integration | Socket + `SO_PEERCRED` | Real `net.UnixListener` in `t.TempDir()`; asserts socket mode `0600`, dir `0700`, and that a wrong-uid peer is rejected (skipped with a recorded reason when the test cannot obtain a second uid) |
| Integration | Worktree allocate/release against a real repo | `gitfixture` builds a repo with two commits; asserts `git worktree list` is clean afterwards |
| Integration | Reconciliation | Write a lease with a dead pid / stale boot-id / expired TTL; assert removal. Write a lease pointing outside the managed root; assert it is **reported and kept** |
| Integration | Repo/commit resolution | `repo_id` values `../x`, `/etc`, `-C`, `--upload-pack=x`, unregistered, and a valid id with an absent oid — all yield `repo_unavailable` and no `git` process is spawned with a remote-derived path |
| Integration | Sandbox isolation | `bwrap` bound to `fakeadapter` instead of `claude`; the fake probes from inside and reports results as findings: reading `$HOME/.claude`, `~/.ssh/id_*`, `~/.gitconfig`, the host secret path, `/work/repo/.git`, `git rev-parse`, writing `/work/repo/x`, and writing `/work/input/patch.diff` must all fail |
| Integration | Secret delivery | The fake adapter asserts `ANTHROPIC_API_KEY` is set and correct in its env, that `/run/krein/adapter.env` no longer exists, and that its own `/proc/self/cmdline` does not contain the key |
| E2E | Full A2A task | `httptest` orchd + real execd over a socket + fake adapter: AgentCard lists exactly one skill; valid task → `completed` + schema-valid artifact; bad token → rejected + audited; unknown skill id → rejected, never dispatched; forced denial → `completed` + `partial` |
| E2E | Crash recovery | Kill `execd` mid-adapter; restart; assert the sweep removes the worktree and lease and `git worktree list` is clean |
| E2E | Leak check | After the full suite, assert zero directories remain under the managed worktree root and zero leases remain |

**Fake-adapter design** is what makes the sandbox suite deterministic. `internal/testsupport/fakeadapter` is a normal Go test binary, built once with `go build`, bound into the sandbox at `/usr/local/bin/claude`. It runs the isolation probes *inside* the namespace and emits their results as a schema-valid findings document, so every isolation assertion is an ordinary Go table assertion on structured output rather than a brittle log grep.

**Environment-dependent skips must be loud.** Tests needing real `bwrap` call `testsupport.RequireSandbox(t)`, which `t.Skip`s with a named reason locally but calls `t.Fatal` when `KREIN_REQUIRE_SANDBOX=1` is set. CI sets it, so a CI host that loses unprivileged user namespaces fails the build instead of quietly passing a suite that proved nothing.

---

## Threat Matrix

The design changes shell commands, subprocesses, git automation and process integration, so the matrix is **applicable**.

| Boundary | Minimum adversarial cases | Applicability | Design response | Planned RED tests |
|---|---|---|---|---|
| Documentation-like paths | `requirements.txt`, `CMakeLists.txt`, executable Markdown/MDX, `README.sh` | **Applicable** — the reviewed worktree and the patch may contain any of these | No path in the reviewed tree is ever classified as executable or handed to any executor. The tool allowlist excludes `Bash`, `Write`, `Edit`, `Task`; the bind is read-only; `execd` never invokes a build or install step | One test per class: a `gitfixture` repo containing `README.sh`, `Makefile`, `requirements.txt`, `CMakeLists.txt`, `setup.py` and a `.git/hooks/pre-commit` — the run completes, and a filesystem sentinel written by each of those files is asserted absent |
| Git repository selection | `git -C`, relative paths, absolute paths | **Applicable** — `execd` runs `git -C <path> worktree add` | Remote input supplies only `repo_id`, resolved through a closed `map[RepoID]string` of operator-configured absolute paths. A remote-derived string is never a path argument. `base_commit` must match the full-oid regex, blocking `-`-prefixed argument injection. Every git call is a fixed argv slice with `--` before positionals; never a shell | One test per selector: `repo_id` ∈ {`../etc`, `/etc/passwd`, `-C`, `--git-dir=/x`, `a/b`, unregistered}; `base_commit` ∈ {`--upload-pack=x`, `-n`, `HEAD`, `main`, short oid, `deadbeef…\n…`} — all `repo_unavailable`, asserted by a `git` exec recorder showing zero invocations with a remote-derived token |
| Commit state | staged, `commit -a`, empty index | **N/A** — krein creates no commits and mutates no index in this slice. `git worktree add` writes a fresh checkout; nothing is ever staged, committed or amended. The reviewed tree is read-only inside the sandbox | None |
| Push state | tracking branch, first push, explicit refspec | **N/A** — there is no push, no remote operation and no network git. `execd` holds zero git remote credentials by design (proposal §2), and the sandbox has no git object store to push from | None |
| PR commands | explicit `--head`, environment prefix, composed commands | **N/A** — no PR automation, no `gh`/forge client, and no VCS-hosting integration exists anywhere in this slice | None |

### Additional applicable boundaries (this design)

| Boundary | Adversarial cases | Design response | Planned RED tests |
|---|---|---|---|
| Subprocess argv composition | Remote bytes reaching `bwrap`/`claude`/shim argv; shell metacharacters; `--`-prefixed values | `BwrapArgv` and `claude.Argv` are pure functions over typed structs; all remote bytes travel by fd; no `sh -c`; `--` terminates every option list | Golden argv equality; assert no argv element contains the patch, focus, `repo_id`, `base_commit` or the secret; a patch whose body is `--dangerously-skip-permissions` changes nothing about argv |
| Secret handling | argv exposure, env inheritance, on-disk residue, log leakage, crash dump | `memfd` + fd 5 + tmpfs + shim unlink; empty non-nil `cmd.Env`; redacting `Secret` type; startup mode check | `/proc/<bwrap-pid>/cmdline` and `/proc/<bwrap-pid>/environ` contain no key; `cmd.Env` is a non-nil **empty** slice (env A); the sandbox env (B) is exactly `PATH`,`HOME`,`LANG`,`TERM=dumb`; the adapter env (C) is B plus exactly one secret variable; the secret file is absent inside the sandbox after `exec`; every `slog` level emits `[REDACTED]` |
| Capability selection | A remote payload naming a tool, skill, adapter or command | `additionalProperties: false` at the decoder; closed capability map; closed `AdapterID`; closed `Op` map | A request carrying `{"tool":"Bash"}`, `{"skill":"shell"}`, `{"adapter":"../x"}` or a second `DataPart` is rejected before any dispatch, and the audit log shows `request_rejected` with no `task_accepted` |
| Filesystem path construction | Remote-influenced identifiers reaching a path | Only `SessionID` (crypto/rand, regex-validated) is used in paths; `TaskID` and `repo_id` never are; managed-root confinement on every removal | A task id of `../../etc` produces a lease whose `worktree_path` is under the managed root; reconciliation refuses to remove any path outside it |

Every applicable row above carries into `tasks.md` unchanged; each RED test is written before the production code it constrains.

---

## Migration / Rollout

No migration. The repository has two commits and no Go source, no persisted data, no released protocol version and no consumers. Rollout is the `auto-chain` delivery strategy already cached for this session; `sdd-tasks` slices along the capability boundaries so no single PR carries both the trust boundary and the adapter.

Suggested slice order, each independently verifiable:

1. `go.mod` + `internal/jobspec` + both socket halves (closed enum provable with no sandbox, no git, no A2A).
2. `internal/execd/repo` + `internal/execd/worktree` (leases and reconciliation provable against a fixture repo alone).
3. `internal/execd/secret` + `internal/execd/sandbox` + `cmd/krein-shim` (isolation provable with the fake adapter, no A2A).
4. `internal/adapter/claude` (argv and outcome taxonomy provable as pure functions).
5. `internal/skill/reviewdiff` + `internal/a2a` + `cmd/orchd` + `cmd/execd` (the E2E close).

Rollback is unchanged from the proposal: delete the branch, or delete the state directory. Per-layer rollback is preserved by the package boundaries — swapping `a2a-go` for a hand-rolled JSON-RPC server touches only `internal/a2a`; swapping bubblewrap for another backend touches only `internal/execd/sandbox`.

---

## Decisions This Design Resolved That The Proposal Left Open

Recorded explicitly so `sdd-tasks` and `sdd-apply` do not re-decide them:

1. **Module path** — `github.com/DeRep311/krein`, read from the configured `origin` remote rather than guessed.
2. **Socket peer authentication** (proposal OQ 4) — `SO_PEERCRED` **and** filesystem mode, no shared secret.
3. **Finding schema** (proposal OQ 2) — the field names, the four-value `severity` enum (`info|warning|error|critical`), and the unconditional presence of `file_path`, `line_start` and `line_end` are taken **verbatim from `review-diff-skill`**, not chosen here. What this design adds on top is flagged in item 18. 200-finding cap; `category`/`cwe` deliberately deferred.
4. **Concrete defaults** (proposal OQ 3) — the full table above.
5. **Audit-log format and location** (proposal OQ 5) — daily JSONL under `<state-dir>/audit/`, keyed-HMAC token fingerprint rather than a bare hash, explicit never-log list.
6. **Wire encoding** — length-prefixed JSON with a hand-written discriminated decoder; gob rejected on a security ground, protobuf deferred with a named migration trigger.
7. **`completeness` is synthesized by `execd`, not self-reported by the adapter** — the most substantive correction this design makes, with `denial_visibility_unavailable` as the fail-loud path when the installed CLI cannot prove it was undenied.
8. **Partial results map to A2A `completed`, not `failed`** — with a required `completeness` field and a prose marker as the mechanism that makes them unmissable.
9. **`memfd` rather than pipes** for all four fd payloads — removes a genuine deadlock class on a 1 MiB patch.
10. **A third binary, `cmd/krein-shim`**, static and Go, rather than a shell one-liner.
11. **Lease liveness uses boot-id + start-ticks**, not bare pid existence, plus a single-instance `flock` on the state directory.
12. **`TaskID` is never a path component**; only `execd`-generated, regex-validated `SessionID` is.
13. **`repo_unavailable` is deliberately ambiguous** at the A2A edge to prevent registry enumeration.
14. **Edge validation is dependency-free strict Go decoding**; JSON Schema is the adapter-facing contract with a conformance test bridging them.
15. **An optional bounded `focus` field** is included in the skill input, delivered by fd rather than argv. It is an **addition beyond the three inputs `review-diff-skill` names** (`repo_id`, `base_commit`, `patch_content`). It could equally have been omitted from v1; it is included because "review this diff, focusing on X" is the obvious first request and adding it later is a schema change.
16. **Seccomp is out of this slice** and named as the first hardening step.
17. **Unrestricted sandbox egress (`--share-net`)** — resolved by explicit user decision for this slice, superseding the former Open Question 1. The residual blast radius (a process-level compromise of the CLI has free egress and read access to the registered repository at `base_commit`, independent of its tool allowlist) is an **accepted, documented risk**, not a pending item. Egress confinement to the model API endpoint is deferred to hardening alongside seccomp.

---

### Explicitly flagged deviations from and additions to the locked specs

Every difference between this design and the six specification files is listed here. Nothing outside this list departs from them.

| # | Spec | Kind | What, and why |
|---|---|---|---|
| 18 | `review-diff-skill` — Structured Findings Schema | **Addition** | Adds three fields beyond the spec's named set: `findings[].id` (`f-%04d`, assigned by `execd` so findings are addressable in audit and prose), `findings[].confidence` (`high\|medium\|low` — model findings need a calibration axis that `severity` does not provide), and `completeness.reason_code` (a machine-readable degradation reason the spec's four completeness fields cannot express, notably `denial_visibility_unavailable`). All are additive; every spec-named field keeps its exact name, type, nullability and enum. |
| 19 | `review-diff-skill` — Accepted Request Inputs | **Addition** | Adds an optional bounded `focus` input and a required `schema_version` discriminator alongside the three spec-fixed inputs. See item 15. |
| 20 | `sandboxed-execution` — Read-Only Worktree and Masked Git Internals | **Deviation (mechanism)** | The spec says mask `<worktree>/.git` "with an empty tmpfs". That is unimplementable for a linked worktree, where `.git` is a regular **file** and a tmpfs needs a directory mountpoint. This design uses `--ro-bind /dev/null /work/repo/.git`, which achieves the requirement's scenario in full. **Proposed spec amendment** — mechanism only; the observable obligation is unchanged. |
| 21 | `execd-job-protocol` — Closed Job-Spec Enum | **Encoding decision** | The three spec-named job types are spelled snake_case on the wire (`allocate_worktree`, `run_review_adapter`, `release_worktree`) to match every other JSON key. One-to-one and total; membership and closedness unchanged. |
| 22 | `session-worktree` — Lease-Before-Allocation | **Addition** | The lease carries the six spec-named fields (`session_id`, `task_id`, `worktree_path`, `repo_id`, `owning_pid`, `created_at`) plus `schema_version`, `repo_path`, `base_oid`, `owner_boot_id`, `owner_start_ticks` and `ttl_ms`. `owner_boot_id`/`owner_start_ticks` exist to make the spec's "`owning_pid` is no longer running" test sound across reboots and pid reuse (item 11). |
| 23 | `sandboxed-execution` — Read-Only Patch File Mount | **Conformance correction** | fds 3, 4 and 6 are mounted `--ro-bind-data` rather than `--file`, because `--file` lands a *writable* file on the `/work/input` tmpfs and the spec requires the adapter be unable to modify the patch. fd 5 remains `--file`, which the secret-injection requirement names explicitly and which the shim's mandatory `unlink` requires. |

---

## Open Questions

**Resolved since the previous revision:** egress scope is no longer open. The user selected unrestricted `--share-net` for this slice; it is now a closed Architecture Decision with its accepted risk documented there and in item 17. Nothing below requires a user decision — every remaining item is an implementation-verification task for `sdd-apply`.

- [ ] **Exact `claude` CLI flag spelling for the tool allowlist/denylist, and whether the installed version exposes a permission-denial field in its `--output-format json` envelope.** The proposal delegated this to verification against the installed CLI. I could not verify it: this design phase had no command-execution tool. The design is written so that this is a *fail-loud* unknown rather than a silent assumption — `claude.Argv` is a pure function with a golden test, so a wrong spelling fails a test rather than a production run, and a missing denial field forces `status: partial` with `denial_visibility_unavailable` rather than a false `complete`. `sdd-apply` must run `claude --help` and `claude -p --output-format json` against a trivial prompt, and pin both the flag spelling and the envelope field set before the adapter package is written.
- [ ] **Does `bwrap` permit `--ro-bind /dev/null <path>` over a regular file that lives inside an already-read-only bind mount?** This is the one bwrap behaviour the design assumes rather than proves, and the `.git` masking depends on it. It has a dedicated RED test; if it fails, the fallback is to bind the worktree's parent as a tmpfs overlay or to reconsider the `git archive` alternative recorded above. Not a user question — an implementation-verification item for `sdd-apply`.
- [ ] **Does `bwrap --ro-bind-data <fd> <dest>` behave as assumed** — copying from the fd and bind-mounting the result read-only, so a write from inside the sandbox fails with `EROFS`? The read-only patch mount (item 23) depends on it. Same verification class as the `/dev/null` item above: a dedicated RED test asserts a write to `/work/input/patch.diff` fails from inside the sandbox. If the flag is unavailable in the installed bubblewrap, the fallback is `--file` onto a tmpfs that is subsequently remounted read-only, or a `--ro-bind` of a short-lived host file. Not a user question — an implementation-verification item for `sdd-apply`.
- [ ] **Do `orchd` and `execd` run as the same uid in the target deployment?** The design defaults `allowed_peer_uid` to the `execd` process uid, which is correct for the same-user slice-1 deployment. If the intended posture is "`execd` privileged" in the stronger sense of a distinct uid, the default changes and the socket directory ownership changes with it. Configurable either way; the default is stated so it is not an accident.
