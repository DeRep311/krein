# sandboxed-execution Specification

## Purpose

`execd` runs the review adapter inside a bubblewrap (`bwrap`) sandbox that guarantees the adapter can see the code under review and the one scoped secret it needs, and nothing else of `execd`'s or the host's identity. This specification defines the namespace/filesystem isolation profile, environment clearing, the file-descriptor-based secret injection path that keeps the key out of `argv` and inherited environment, secret provenance checks, and the wall-clock/turn resource budget that bounds every invocation.

## Requirements

### Requirement: Namespace and Filesystem Isolation Profile

`execd` MUST run the adapter inside a `bwrap` sandbox with: full namespace unsharing except network egress (`--unshare-all --share-net`), a fresh session (`--new-session`), termination tied to the parent (`--die-with-parent`), non-zero UID/GID mapping, a fresh `/proc`, a minimal `/dev`, and read-only mounts of `/usr` and `/lib*`.

#### Scenario: Sandbox process runs under the full isolation profile

- GIVEN `execd` prepares a `bwrap` invocation for `RunReviewAdapter`
- WHEN the sandbox process starts
- THEN it runs with all namespaces unshared except network (needed for the model API), a fresh session, non-zero UID/GID, a fresh `/proc`, a minimal `/dev`, and read-only `/usr` and `/lib*`
- AND the sandbox process terminates if its parent (`execd`'s subprocess supervisor) terminates

### Requirement: Environment Clearing

`execd` MUST clear the host environment before constructing the sandbox environment (`bwrap --clearenv`) and MUST pass only an explicitly constructed minimal environment: `PATH`, `HOME`, `LANG`, and `TERM=dumb`. `execd` MUST NOT pass its own `os.Environ()` through to the sandboxed process.

#### Scenario: Sandboxed adapter sees only the minimal constructed environment

- GIVEN `execd` starts a sandboxed adapter run
- WHEN the adapter process inspects its own environment
- THEN it observes exactly `PATH`, `HOME`, `LANG`, and `TERM=dumb`, and no variable inherited from `execd`'s own process environment

### Requirement: File-Descriptor-Based Secret Injection

`execd` MUST NOT pass the adapter's model API key via process arguments or as a `bwrap --setenv` value. The secret MUST be delivered via an open file descriptor passed to `bwrap --file <fd> /run/krein/adapter.env`; an in-sandbox entry shim MUST read the secret from that path, export it into the adapter's process environment, unlink the file, and then `exec` the adapter binary.

#### Scenario: Secret never appears in sandbox argv or a persistent file

- GIVEN `execd` launches a sandboxed adapter run requiring a model API key
- WHEN the sandbox process and its `bwrap` invocation are inspected (e.g. via `/proc/$PID/cmdline`)
- THEN no API key or secret value appears in the `bwrap` or adapter process argv
- AND after the entry shim runs, `/run/krein/adapter.env` no longer exists inside the sandbox

### Requirement: Masked Home Directory

`execd` MUST mask the sandbox's home directory with an empty tmpfs (`--tmpfs $HOME`) so that `~/.claude`, `~/.claude.json`, `~/.gitconfig`, `~/.ssh`, and any other host credential file are structurally absent from the sandbox, not merely unreferenced.

#### Scenario: Host credential files are structurally absent in the sandbox

- GIVEN the sandbox host has `~/.claude`, `~/.gitconfig`, and `~/.ssh` present outside the sandbox
- WHEN the adapter process inside the sandbox attempts to read any of those paths under its masked `$HOME`
- THEN none of those paths exist inside the sandbox

### Requirement: Read-Only Worktree and Masked Git Internals

The session worktree MUST be bind-mounted read-only at a fixed in-sandbox path. `<worktree>/.git` MUST be masked with an empty tmpfs so the adapter cannot reach the host repository's object store, hooks, or config through the mounted worktree.

#### Scenario: Adapter cannot write to the worktree or reach .git internals

- GIVEN a session worktree is bind-mounted into the sandbox
- WHEN the adapter attempts to write to any file in the mounted worktree
- THEN the write fails with a read-only filesystem error
- AND the adapter cannot read the host repository's `.git` object store, hooks, or config through the mounted worktree

### Requirement: Read-Only Patch File Mount

The inline diff/patch under review MUST be written by `execd` to a file and mounted into the sandbox as a separate read-only file, distinct from the worktree mount.

#### Scenario: Patch is visible to the adapter as a read-only file

- GIVEN `execd` has written the inline patch content to a temporary file
- WHEN the sandbox starts
- THEN the adapter can read the patch content from its mounted read-only path
- AND the adapter cannot modify that file

### Requirement: Secret File Permission Enforcement at Startup

`execd` MUST verify the file permissions of its configured adapter-secret file at startup and MUST refuse to start if that file is readable by group or others (i.e. permissions more permissive than `0600`).

#### Scenario: execd refuses to start with an over-permissive secret file

- GIVEN `execd`'s configured adapter-secret file has mode `0644`
- WHEN `execd` starts
- THEN `execd` refuses to start and reports the permission violation
- AND no adapter invocation is attempted

### Requirement: Wall-Clock and Turn-Budget Resource Enforcement

`execd` MUST enforce a configured wall-clock execution timeout and turn budget on every adapter invocation. If the wall-clock timeout is exceeded, `execd` MUST terminate the sandboxed process tree (`SIGTERM` followed by `SIGKILL` if unresponsive) and MUST release the session's worktree as part of that termination.

#### Scenario: Timeout terminates the sandbox and releases the worktree

- GIVEN an adapter run is configured with a wall-clock timeout
- WHEN the adapter subprocess exceeds that timeout without terminating on its own
- THEN `execd` sends `SIGTERM` to the sandbox process group, followed by `SIGKILL` if it does not exit
- AND `execd` releases the session's worktree and lease as part of handling the timeout
