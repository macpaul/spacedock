---
id: 7s59y5jy5phhsdbcaf4knh13
title: Win64 cross-compilation (GOARCH=amd64)
status: backlog
source: commission seed
started:
completed:
verdict:
score: 0.95
worktree:
issue:
pr:
mod-block:
gates:
    version: 1
    records:
        - id: gate:7s59y5jy5phhsdbcaf4knh13:backlog
          stage: backlog
          attempts:
            - id: gate-attempt:7s59y5jy5phhsdbcaf4knh13-backlog-1
              briefing:
                id: briefing:7s59y5jy5phhsdbcaf4knh13:backlog:attempt-1:revision-1
                digest: sha256:8c482e4c321a961c5e9d273940f8c3c242cb015e43780b73d54cd7b7958c0108
                request-digest: sha256:fd6a620bded35939d1ebade814dfbe6347debc0a90e9b916ce641688b69cf7af
                room-ref: ./win64-cross-compilation/review/backlog/briefing-1
---

## Problem

The spacedock binary cannot currently be cross-compiled to Windows 64-bit (GOARCH=amd64, GOOS=windows). Users on the dominant Windows platform (64-bit) cannot obtain a native `.exe`. This is the higher-priority target since most Windows machines in the field are 64-bit.

## Proposed approach

Use Go's built-in cross-compilation with `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build ./cmd/spacedock/` to produce a `spacedock.exe`. Verify the output with `file spacedock.exe` (should report "PE32+ executable" for 64-bit) and `objdump -f spacedock.exe` (should show `x86-64` architecture). If CGO is involved, use `CC=x86_64-w64-mingw32-gcc` and install `gcc-mingw-w64-x86-64`.

Check the `creack/pty` dependency — it is Unix-only (pseudo-terminal). Confirm it has a Windows no-op stub or that `CGO_ENABLED=0` is sufficient to avoid it. If not, a `//go:build !windows` guard may be needed at the call site.

## Acceptance criteria

**AC-1 — Cross-compilation produces a valid PE32+ executable without build errors.**
Verified by: `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build -o spacedock-win64.exe ./cmd/spacedock/` exits 0. `file spacedock-win64.exe` output contains "PE32+ executable" and "x86-64". Would fail if the build errors on missing Windows stubs, unsatisfied imports, or CGO symbols.

**AC-2 — The go test suite passes for GOOS=windows (compile-time check).**
Verified by: `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go test -run=^$ ./...` exits 0 (compile-only, no execution). This catches Windows-incompatible syscall or platform-specific code paths that compile fine on Linux but fail to compile for Windows. Would fail if any package references Unix-only syscalls without a build tag guard.

**AC-3 — The binary reports its version correctly when invoked (deferred: captain-windows-smoke).**
Verified by: running `spacedock.exe --version` on a physical 64-bit Windows machine outputs the correct version string. Deferred to captain's Windows smoke test after merge.

**AC-4 — No new platform ifdefs leak into shared packages without isolation.**
Verified by: `git diff --stat main HEAD` shows any `//go:build windows` additions in `internal/` are paired with a `//go:build !windows` stub in the same package. Would fail if a Windows-only code path is introduced without a Linux-side no-op.

## Test plan

**Offline (Linux, reproducible by fresh agent):**
1. `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build -o spacedock-win64.exe ./cmd/spacedock/` — exit 0 required.
2. `file spacedock-win64.exe` — must contain "PE32+ executable" and "x86-64".
3. `objdump -f spacedock-win64.exe` — must show `architecture: i386:x86-64`.
4. `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go test -run=^$ ./...` — exit 0 required.
5. `go vet ./...` on the main module — must pass.

**Interactive (deferred — Windows machine required):**
- Run `spacedock-win64.exe --version` on a 64-bit Windows machine, confirm version string.
- Run `spacedock-win64.exe --help`, confirm usage text.
- Run `spacedock-win64.exe status --help`, confirm the status subcommand is accessible.

## Out of scope

- 32-bit Windows (GOARCH=386) — separate task.
- GoReleaser pipeline changes — separate task.
- Windows installer or package manager support — not in scope.
- Windows CI lane — deferred.
- ARM64 Windows — not in scope for this workflow.
