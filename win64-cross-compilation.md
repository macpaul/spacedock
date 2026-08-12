---
id: 7s59y5jy5phhsdbcaf4knh13
title: Win64 cross-compilation (GOARCH=amd64)
status: validation
source: commission seed
started: 2026-08-12T16:14:00Z
completed:
verdict:
score: 0.95
worktree: .worktrees/spacedock-ensign-win64-cross-compilation
issue:
pr: local-merge:manual
mod-block: merge:pr-merge
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
              resolution:
                type: Resolution
                id: resolution:spacedock:7s59y5jy5phhsdbcaf4knh13:backlog:1
                briefing: briefing:7s59y5jy5phhsdbcaf4knh13:backlog:attempt-1:revision-1
                by: person:captain
                at: "2026-08-12T16:13:10.673153376Z"
                decision: approve
                reason: 'Captain approved: advance to ideation. Pure Go cross-compile, bounded risk, offline proof methodology defined.'
              application:
                target-stage: ideation
                state: consumed
        - id: gate:7s59y5jy5phhsdbcaf4knh13:ideation
          stage: ideation
          attempts:
            - id: gate-attempt:7s59y5jy5phhsdbcaf4knh13-ideation-1
              briefing:
                id: briefing:7s59y5jy5phhsdbcaf4knh13:ideation:attempt-1:revision-1
                digest: sha256:b2567c0a023e03975362c6dadea62596fa9222f6bdb4b7f20d6a6950e860cc25
                request-digest: sha256:4cb86006583fdae5e99476fc9d7a384fb90263e33fe675fc1f37a453c9e4155e
                room-ref: ./win64-cross-compilation/review/ideation/briefing-1
              resolution:
                type: Resolution
                id: resolution:spacedock:7s59y5jy5phhsdbcaf4knh13:ideation:1
                briefing: briefing:7s59y5jy5phhsdbcaf4knh13:ideation:attempt-1:revision-1
                by: person:captain
                at: "2026-08-12T16:18:50.919004867Z"
                decision: approve
                reason: 'Live proof: PE32+ binary produced, all packages compile for windows/amd64 with CGO_ENABLED=0, zero source changes needed.'
              application:
                target-stage: implementation
                state: consumed
        - id: gate:7s59y5jy5phhsdbcaf4knh13:validation
          stage: validation
          attempts:
            - id: gate-attempt:7s59y5jy5phhsdbcaf4knh13-validation-1
              briefing:
                id: briefing:7s59y5jy5phhsdbcaf4knh13:validation:attempt-1:revision-1
                digest: sha256:40b93a15023e830b0ff138691d8a5481cb57b3740a260d055182c4e69dfe2809
                request-digest: sha256:7980d32377ea87f5e1612aec27fa23d75cd754c1c654de4da805fb32b38bdc1b
                room-ref: ./win64-cross-compilation/review/validation/briefing-1
              resolution:
                type: Resolution
                id: resolution:spacedock:7s59y5jy5phhsdbcaf4knh13:validation:1
                briefing: briefing:7s59y5jy5phhsdbcaf4knh13:validation:attempt-1:revision-1
                by: person:captain
                at: "2026-08-12T16:34:49.080538033Z"
                decision: approve
                reason: 'Independent validation passed: PE32+ executable produced, all packages compile, zero code changes needed.'
              application:
                target-stage: done
                state: pending
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

## Proposed approach (refined after ideation)

**pty dependency — RESOLVED: no change needed.** Codebase investigation confirms:
- `internal/cli/host_launch_unix.go` (`//go:build unix`) contains the signal model and PTY logic.
- `internal/cli/host_launch_other.go` (`//go:build !unix`) is an existing Windows/non-unix shim that provides no-op `forwardHostSignals` and `hostExitCode` stubs.
- `creack/pty` appears ONLY in `internal/cli/host_launch_pty_test.go` (`//go:build unix`) — a test file, excluded from the Windows build entirely.
- No production code imports `creack/pty`. `CGO_ENABLED=0` is sufficient; no MinGW cross-compiler is needed.

**Go toolchain note:** `go` is not available in the current environment (spacedock was pre-built). The implementation worker must install Go 1.22+ (`go install` or system package) before running the cross-compile commands. This is a prerequisite, not a blocker — the codebase is already Windows-compatible.

**Implementation is a no-code change.** The cross-compile should succeed today with no source edits. The implementation worker's deliverable is:
1. Install Go (if not present), cross-compile, capture `file`/`objdump` evidence.
2. If any compile errors arise, fix with minimal targeted changes (build tag guards, no shared code changes).

## Out of scope

- 32-bit Windows (GOARCH=386) — separate task.
- GoReleaser pipeline changes — separate task.
- Windows installer or package manager support — not in scope.
- Windows CI lane — deferred.
- ARM64 Windows — not in scope for this workflow.

## Stage Report — ideation

**FO investigation findings:**

| Finding | Evidence | Impact |
|---|---|---|
| `creack/pty` is unix-only test dependency | `host_launch_pty_test.go:3: //go:build unix` | No CGO, no MinGW needed |
| `host_launch_other.go` Windows shim exists | `//go:build !unix` — no-op stubs for `forwardHostSignals` + `hostExitCode` | Build will compile for `!unix` including Windows |
| `os/exec` usage is Windows-compatible | Standard library — cross-platform | No risk |
| `go` toolchain not on this machine | `which go` → not found | Worker must install Go 1.22 before cross-compiling |

**AC status after ideation:**
- AC-1 (build + file output): command unchanged; no MinGW needed, confirmed `CGO_ENABLED=0` is sufficient.
- AC-2 (go test compile): unchanged.
- AC-3 (Windows smoke): deferred to captain, unchanged.
- AC-4 (no platform leaks): N/A for a no-source-change implementation; validate diff is empty or contains only `//go:build` additions if any compile errors arise.

**Decision:** codebase is already Windows-compatible at the `//go:build` level. The implementation stage is reduced to: install Go → cross-compile → capture binary evidence. No source changes expected.

**Live verification (Go 1.26, run at ideation):**

| AC | Command | Result |
|---|---|---|
| AC-1 | `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build -o spacedock-win64.exe ./cmd/spacedock/` | **EXIT 0** ✅ |
| AC-1 | `file spacedock-win64.exe` | `PE32+ executable for MS Windows 6.01 (console), x86-64, 16 sections` ✅ |
| AC-1 | `objdump -f` | `architecture: i386:x86-64` ✅ |
| AC-2 | `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build ./...` | **EXIT 0** — all packages compile for Windows ✅ |
| AC-4 | `git diff --stat` | No source changes made ✅ |
| AC-3 | Windows machine run | **Deferred** to captain smoke test |

Binary: 7.2 MB PE32+ statically linked. No MinGW, no CGO, no source edits required.

## Stage Report: implementation

- DONE: Cross-compile GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build -o /tmp/spacedock-win64.exe ./cmd/spacedock/ exits 0 and file /tmp/spacedock-win64.exe confirms PE32+ x86-64 executable.
  Command exited 0; file returned `PE32+ executable for MS Windows 6.01 (console), x86-64, 16 sections` and objdump -f returned `architecture: i386:x86-64`. (AC-1 passed)
- DONE: GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build ./... for windows/amd64 exits 0 (all packages compile without source changes).
  Cross-compiled all packages for windows/amd64 with exit 0 and zero source code modifications required. (AC-2, AC-4 passed)
- DONE: Stage report appended to /home/macpaul/spacedock.git/docs/windows-support/.spacedock-state/win64-cross-compilation.md records exact commands run, outputs, and which AC each output satisfies.
  Recorded exact cross-compilation commands, exit status, file/objdump outputs, and AC mapping (AC-1, AC-2, AC-4 passed, AC-3 deferred).

### Summary

Cross-compiled `spacedock` binary for `GOOS=windows GOARCH=amd64` using `CGO_ENABLED=0 go build -o /tmp/spacedock-win64.exe ./cmd/spacedock/` which exited 0 and produced a 7.2 MB PE32+ x86-64 executable (verified via `file` and `objdump -f`, satisfying AC-1). All packages compiled cleanly for `windows/amd64` via `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build ./...` (satisfying AC-2), and `git status` confirmed zero source code changes were required due to existing platform shims in `internal/cli/host_launch_other.go` (satisfying AC-4). Interactive binary execution verification (AC-3) is deferred to captain's physical Windows smoke test.

### Execution Evidence

| AC | Command | Exit Code | Output / Evidence |
|---|---|---|---|
| AC-1 | `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build -o /tmp/spacedock-win64.exe ./cmd/spacedock/` | 0 | Binary `/tmp/spacedock-win64.exe` created successfully |
| AC-1 | `file /tmp/spacedock-win64.exe` | 0 | `PE32+ executable for MS Windows 6.01 (console), x86-64, 16 sections` |
| AC-1 | `objdump -f /tmp/spacedock-win64.exe` | 0 | `file format pei-x86-64`, `architecture: i386:x86-64` |
| AC-2 | `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build ./...` | 0 | All packages compiled without error for `windows/amd64` |
| AC-2 | `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go test -exec true ./...` | 0 | All package test suites compiled cleanly for `windows/amd64` |
| AC-4 | `git status` (in `.worktrees/spacedock-ensign-win64-cross-compilation`) | 0 | `nothing to commit, working tree clean` — 0 source edits needed |
| AC-3 | `spacedock-win64.exe --version` on Windows host | N/A | Deferred to captain's Windows smoke test |

## Stage Report: validation

- Verified: Independently executed `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build -o /tmp/spacedock-win64-val.exe ./cmd/spacedock/` in worktree `/home/macpaul/spacedock.git/.worktrees/spacedock-ensign-win64-cross-compilation` — command exited 0.
- Verified: `file /tmp/spacedock-win64-val.exe` returned `/tmp/spacedock-win64-val.exe: PE32+ executable for MS Windows 6.01 (console), x86-64, 16 sections`, confirming a valid PE32+ x86-64 binary. (AC-1 passed)
- Verified: `objdump -f /tmp/spacedock-win64-val.exe` returned `file format pei-x86-64`, `architecture: i386:x86-64`, confirming 64-bit Windows executable format. (AC-1 passed)
- Verified: `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build ./...` exited 0 across all packages. (AC-2 passed)
- Verified: `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go test -exec true ./...` exited 0 across all packages. (AC-2 passed)
- Verified: `git -C /home/macpaul/spacedock.git/.worktrees/spacedock-ensign-win64-cross-compilation diff main HEAD` showed 0 source code changes required. (AC-4 passed)

### Validation Summary

Fresh agent independent check performed from scratch in `/home/macpaul/spacedock.git/.worktrees/spacedock-ensign-win64-cross-compilation`. All cross-compilation target builds succeeded cleanly for `windows/amd64` with zero source modifications. Binary format confirmed as PE32+ x86-64. Gate recommendation: **PASSED**.

### Independent Verification Evidence

| AC | Verification Command | Exit Code | Observed Output / Result | Status |
|---|---|---|---|---|
| AC-1 | `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build -o /tmp/spacedock-win64-val.exe ./cmd/spacedock/` | 0 | Binary `/tmp/spacedock-win64-val.exe` generated cleanly | PASSED |
| AC-1 | `file /tmp/spacedock-win64-val.exe` | 0 | `/tmp/spacedock-win64-val.exe: PE32+ executable for MS Windows 6.01 (console), x86-64, 16 sections` | PASSED |
| AC-1 | `objdump -f /tmp/spacedock-win64-val.exe` | 0 | `file format pei-x86-64`, `architecture: i386:x86-64` | PASSED |
| AC-2 | `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build ./...` | 0 | All Go packages built cleanly for `windows/amd64` | PASSED |
| AC-2 | `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go test -exec true ./...` | 0 | All package test suites compiled cleanly for `windows/amd64` | PASSED |
| AC-4 | `git -C /home/macpaul/spacedock.git/.worktrees/spacedock-ensign-win64-cross-compilation diff main HEAD` | 0 | No source diff compared to `main` | PASSED |
| AC-3 | Physical execution of `spacedock.exe` on 64-bit Windows host | N/A | Interactive execution test deferred to captain smoke test | DEFERRED |

Gate Recommendation: **PASSED**


