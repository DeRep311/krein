# Krein

A lightweight, Go-based multi-agent orchestrator for coding CLIs — built to be faster and simpler than heavy Python agents, while keeping the parts worth keeping: a self-improving skill loop, deep MCP support, and safe federation between independently-run instances ("brains").

> Status: early design phase. No working code yet — see [`openspec/`](openspec/) for the active spec-driven development trail.

## Why

Existing options force a tradeoff: heavy self-improving agents (like Hermes) are slow to start and hard to extend; deterministic workflow tools (like gentle-ai) don't execute anything themselves; parallel-session managers (like Claude Squad) isolate agents but don't let them collaborate. Krein aims to combine what each does well into one small runtime.

## What it does

- **Orchestrates external coding CLIs** — spawns and talks to Claude Code, Google's Antigravity CLI (`agy`), and OpenAI Codex as subprocess adapters, choosing the right one for the job (e.g. delegating to a plan-mode helper).
- **Federates safely across people** — two independent Krein instances ("brains"), each under their own auth/session/billing, can hand each other bounded, sandboxed tasks over [A2A](https://a2a-protocol.org/latest/) — never by letting remote input pick which local tool runs.
- **Extends via MCP** — local tools and integrations come through the Model Context Protocol.
- **Learns over time** — an agent-curated memory and self-skill-creation loop, in the spirit of Hermes, aimed at cutting future token spend.

## Architecture (short version)

Two cooperating processes:

- **`orchd`** — unprivileged core: A2A edge, local control-plane API, workflow engine, capability registry, memory, MCP client pool.
- **`execd`** — privileged executor: holds git/repo credentials, drives the sandbox, manages git worktrees. Receives a *closed* job-spec enum over a local socket — there is no "run arbitrary command" verb.

Content that arrives from a remote brain can never select which local capability runs; it can only fill the parameters of a capability already approved locally. Any work triggered by a remote brain, or by a self-created skill, executes inside an ephemeral sandbox — never directly on the host.

Full rationale and the architecture options that were compared live in project memory and will land in `openspec/specs/` as the first slice solidifies.

## First milestone: `review-diff-slice`

A minimal walking skeleton proving the shape end-to-end: one exposed A2A skill (`review-diff`) → fresh git worktree → one pinned CLI adapter running inside a sandbox → structured findings back to the requesting brain. No shared memory, no skill-creation loop yet — those come after the skeleton is proven.

## Development

Krein follows Spec-Driven Development (SDD). Active and archived changes live under [`openspec/`](openspec/); see `openspec/config.yaml` for the project's testing and process conventions (Go, `go test ./...`, strict TDD).

## License

[MIT](LICENSE)
