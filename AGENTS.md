# Reseau Agent Notes (Go Rewrite)

This repository is in a full rewrite state.

- Current active code lives in `src/`.
- Archived pre-rewrite code lives in `src_old/`.
- Current active tests live in `test/`.
- Archived pre-rewrite tests live in `test_old/`.

## Rewrite Mandate

Reseau is being rewritten to mirror Go's networking stack architecture and semantics.

Required reference source:

- `~/golang/src/runtime`
- `~/golang/src/internal/poll`
- `~/golang/src/net`
- `~/golang/src/crypto/tls`

All implementation work must directly reference Go's logical flow of:

- data structures and ownership
- function boundaries and call ordering
- wait/unblock/deadline semantics
- event loop behavior and wake mechanisms

The intended public surface should stay small and explicit:

- `TCP`
- `TLS`

Treat Base/stdlib behavior as a contract reference when shaping APIs, especially for `IO`, deadlines, and buffer-handling behavior, but do not depend on `Sockets` stdlib internally.

## Hard Rules

- No backwards compatibility layers.
- No API shims for legacy Reseau interfaces.
- No partial migration tricks that preserve old behavior.
- Prefer semantic parity with Go over preserving old package behavior.
- Do not add or rely on `Sockets` stdlib dependency; use native socket/name-resolution calls directly.
- Delete dead code instead of preserving cruft during rewrites.
- Avoid abstract fields, `Any`, and hot-path dynamic dispatch in new code.
- Prefer direct args/kwargs over thin `XOptions` structs that only shuttle a few fields around.
- Never use `Threads.Atomic` in new code.
- Use `@atomic` fields on `mutable struct` types instead.
- Do not check in `Manifest.toml`.

## API and Design Preferences

- Keep the public API centered on `TCP` and `TLS`; do not grow internal subsystems into user-facing surface area unless there is a strong reason.
- Favor `IO`-like contracts and Base/Sockets signature parity for public connection APIs where it improves usability.
- Prefer buffer APIs that can work with `AbstractVector{UInt8}` and view-like inputs where the implementation can support them cleanly.
- Prefer root-cause simplification over adding extra wrapper types, callback adapters, or compatibility scaffolding.
- Keep internal comments and docstrings focused on the current code and behavior, not on historical merge/refactor context.

## Validation and Workflow Rules

- No shortcuts: do not paper over lifecycle or correctness issues with fake timeouts, fake waits, stub behavior, or production code that exists only to quiet a test/compiler path.
- Keep precompile and `--trim=safe` validation real and focused on main public entrypoints and real transport behavior.
- Prefer fixing the actual root cause over permanent skips, narrow hacks, or temporary debug code left behind.
- If a Linux CI job that normally finishes in a few minutes starts running much longer, assume it is hung and debug it accordingly.
- When Windows work is in scope, aim for real parity and real entrypoint coverage, not Windows-only behavioral compromises, unless blocked by a documented Julia/compiler issue.
- When a change affects companion packages or shared behavior, validate downstream packages as needed instead of relying on compatibility shims.
- Leave user-owned untracked files, local probes, and working markdown plans alone unless explicitly asked to clean them up.
- Action-item markdown files are often local working artifacts; do not commit them unless explicitly asked.

## Phase Order

The rewrite is currently macOS-first.

If `golang-rewrite.md` is present in the checkout, treat it as the more detailed roadmap.

- Phases 1-8: macOS-only implementation (kqueue/POSIX/TLS)
- Phase 9: Linux + Windows expansion (epoll + IOCP)

Do not start Linux/Windows implementation work before macOS phase gates are complete.

## Current Test Entry Point

The current test entrypoint is `test/runtests.jl`.

At the time these notes were updated, it includes:

- IOPoll runtime tests
- internal poll tests
- socket ops tests
- TCP tests
- host resolver tests
- TLS tests
- trim compile tests

Run from repo root:

```sh
cd "$(git rev-parse --show-toplevel)"
JULIA_NUM_THREADS=1 julia --project=. --startup-file=no --history-file=no -e 'using Pkg; Pkg.instantiate(); Pkg.test(; coverage=false)'
```
