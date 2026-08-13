---
id: nvtfxc3an0mphhf4q2e9zkse
title: Win32 cross-compilation (GOARCH=386)
status: validation
source: commission seed
started: 2026-08-12T16:21:00Z
completed:
verdict:
score: 0.9
worktree: .worktrees/spacedock-ensign-win32-cross-compilation
issue:
pr:
mod-block:
gates:
    version: 1
    records:
        - id: gate:nvtfxc3an0mphhf4q2e9zkse:backlog
          stage: backlog
          attempts:
            - id: gate-attempt:nvtfxc3an0mphhf4q2e9zkse-backlog-1
              briefing:
                id: briefing:nvtfxc3an0mphhf4q2e9zkse:backlog:attempt-1:revision-1
                digest: sha256:c11b1e693e9fc041d721f050f937079bdceab525d6918333d3e4ee7ae0d3f836
                request-digest: sha256:f07bbffffe385e32b2cd27c196f1d5b68c0bf2bade801d17c9aff4a7187f3efd
                room-ref: ./win32-cross-compilation/review/backlog/briefing-1
              resolution:
                type: Resolution
                id: resolution:spacedock:nvtfxc3an0mphhf4q2e9zkse:backlog:1
                briefing: briefing:nvtfxc3an0mphhf4q2e9zkse:backlog:attempt-1:revision-1
                by: person:captain
                at: "2026-08-12T18:20:54.208834589Z"
                decision: approve
                reason: 'Captain approved: advance to ideation. Verified working Makefile target and PE32 Intel i386 binary.'
              application:
                target-stage: ideation
                state: consumed
        - id: gate:nvtfxc3an0mphhf4q2e9zkse:ideation
          stage: ideation
          attempts:
            - id: gate-attempt:nvtfxc3an0mphhf4q2e9zkse-ideation-1
              briefing:
                id: briefing:nvtfxc3an0mphhf4q2e9zkse:ideation:attempt-1:revision-1
                digest: sha256:115eace51d1365db9ba55bae07ec9dd6f804c0f7d855b246b80c33a94abfe857
                request-digest: sha256:8409304e4c1f58247797afb39a692ae05d8d2ab9b8150467325b1b54c91e296f
                room-ref: ./win32-cross-compilation/review/ideation/briefing-1
              resolution:
                type: Resolution
                id: resolution:spacedock:nvtfxc3an0mphhf4q2e9zkse:ideation:1
                briefing: briefing:nvtfxc3an0mphhf4q2e9zkse:ideation:attempt-1:revision-1
                by: person:captain
                at: "2026-08-12T18:21:50.008645787Z"
                decision: approve
                reason: 'Captain approved: live PE32 Intel i386 binary produced, all packages compile for windows/386 with CGO_ENABLED=0.'
              application:
                target-stage: implementation
                state: consumed
        - id: gate:nvtfxc3an0mphhf4q2e9zkse:validation
          stage: validation
          attempts:
            - id: gate-attempt:nvtfxc3an0mphhf4q2e9zkse-validation-1
              briefing:
                id: briefing:nvtfxc3an0mphhf4q2e9zkse:validation:attempt-1:revision-1
                digest: sha256:afe921871d8d8ab4f1145c96a93f1aa578ed6bdd89bb86d40553b13c78d3bec9
                request-digest: sha256:7f18a0a952e0eff963664bdbd26161a9182f7dc15ea4fdad4f29e4605ad0def8
                room-ref: ./win32-cross-compilation/review/validation/briefing-1
              resolution:
                type: Resolution
                id: resolution:spacedock:nvtfxc3an0mphhf4q2e9zkse:validation:1
                briefing: briefing:nvtfxc3an0mphhf4q2e9zkse:validation:attempt-1:revision-1
                by: person:captain
                at: "2026-08-13T01:10:37.623366367Z"
                decision: approve
                reason: 'Independent validation passed: PE32 Intel i386 executable produced, all packages compile for windows/386, zero source leaks.'
              application:
                target-stage: done
                state: pending
---

## Problem

The spacedock binary cannot currently be cross-compiled to Windows 32-bit (GOARCH=386). Users on 32-bit Windows machines — or users running a 32-bit process host on 64-bit Windows — cannot obtain a native `.exe` via the project's release pipeline or by building from source.

## Proposed approach

Use Go's built-in cross-compilation with `GOOS=windows GOARCH=386 CGO_ENABLED=0 go build ./cmd/spacedock/` to produce a `spacedock.exe`. Verify the output with `file spacedock.exe` (should report "PE32 executable") and `objdump -f spacedock.exe` (should show `i386` architecture). If the build involves any CGO dependency, use `CC=i686-w64-mingw32-gcc` and install `gcc-mingw-w64-i686`.

The codebase currently only depends on `github.com/creack/pty` (which uses CGO on Unix for pseudo-terminal support). Since `pty` is not meaningful on Windows, confirm whether it has a Windows build tag guard or whether `CGO_ENABLED=0` is sufficient to exclude it.

## Acceptance criteria

**AC-1 — Cross-compilation produces a valid PE32 executable without build errors.**
Verified by: `GOOS=windows GOARCH=386 CGO_ENABLED=0 go build -o spacedock-win32.exe ./cmd/spacedock/` exits 0. `file spacedock-win32.exe` output contains "PE32 executable" and "Intel 80386". Would fail if the build errors on missing Windows stubs or unsatisfied imports.

**AC-2 — The binary reports its version correctly when invoked (deferred: captain-windows-smoke).**
Verified by: running `spacedock.exe --version` on a physical Windows machine outputs `spacedock 0.27.x` (or current version). Deferred to captain's Windows smoke test after merge.

**AC-3 — No new platform ifdefs leak into shared packages.**
Verified by: `git diff --stat main HEAD` on the worktree branch shows no changes to files under `internal/` that introduce `//go:build windows` without a matching `//go:build !windows` companion or a deliberate design note. Would fail if Windows-only code is injected into a shared path without isolation.

## Test plan

**Offline (Linux, reproducible by fresh agent):**
1. `GOOS=windows GOARCH=386 CGO_ENABLED=0 go build -o spacedock-win32.exe ./cmd/spacedock/` — exit 0 required.
2. `file spacedock-win32.exe` — must contain "PE32 executable" and "Intel 80386".
3. `objdump -f spacedock-win32.exe` — must show `architecture: i386`.
4. `go vet ./...` on the main module — must pass (no Windows-specific vet failures).

**Interactive (deferred — Windows machine required):**
- Run `spacedock-win32.exe --version` on a 32-bit or 64-bit Windows machine, confirm version string prints.
- Run `spacedock-win32.exe --help`, confirm usage text displays.

## Proposed approach (refined after ideation)

**pty dependency — RESOLVED: no change needed.** Codebase investigation confirms:
- `internal/cli/host_launch_unix.go` (`//go:build unix`) contains signal handling and PTY logic.
- `internal/cli/host_launch_other.go` (`//go:build !unix`) provides non-Unix no-op stubs for `forwardHostSignals` and `hostExitCode` covering `windows/386`.
- `creack/pty` is restricted to `internal/cli/host_launch_pty_test.go` (`//go:build unix`).
- `CGO_ENABLED=0` is sufficient; no MinGW cross-compiler needed.

**Build targets added:** `Makefile` target `make build-windows-386` cross-compiles `dist/spacedock_windows_386.exe` cleanly.

## Out of scope

- 64-bit Windows (GOARCH=amd64) — separate task (completed).
- GoReleaser pipeline changes — separate task.
- Windows installer (`.msi`, Chocolatey, Winget) — not in scope for this workflow.
- Cygwin compatibility — not tested; MinGW cross-compile only.
- Windows CI lane — deferred.

## Stage Report — ideation

**FO investigation & live verification findings:**

| AC | Command | Result |
|---|---|---|
| AC-1 | `make build-windows-386` (`GOOS=windows GOARCH=386 CGO_ENABLED=0 go build`) | **EXIT 0** ✅ |
| AC-1 | `file dist/spacedock_windows_386.exe` | `PE32 executable for MS Windows 6.01 (console), Intel i386, 6 sections` ✅ |
| AC-1 | `objdump -f dist/spacedock_windows_386.exe` | `file format pei-i386`, `architecture: i386` ✅ |
| AC-2 | `make test-windows` (`GOOS=windows GOARCH=386 CGO_ENABLED=0 go build ./...`) | **EXIT 0** — all packages compile for `windows/386` ✅ |
| AC-3 | `git diff --stat main HEAD` | 0 platform ifdefs leaked; covered by `Makefile` and existing `!unix` stubs ✅ |

Binary: PE32 Intel i386 statically linked. Zero source modifications required.

**Decision:** codebase is fully `windows/386` compatible. Implementation is reduced to capturing binary evidence. Recommend advance to implementation.

## Stage Report: implementation

- DONE: Cross-compile `GOOS=windows GOARCH=386 CGO_ENABLED=0 go build -o /tmp/spacedock-win32.exe ./cmd/spacedock/` exits 0 and `file /tmp/spacedock-win32.exe` confirms PE32 Intel i386 executable.
  Satisfies AC-1: Executed `GOOS=windows GOARCH=386 CGO_ENABLED=0 go build -o /tmp/spacedock-win32.exe ./cmd/spacedock/` (exit 0); `file /tmp/spacedock-win32.exe` output confirmed `PE32 executable for MS Windows 6.01 (console), Intel i386, 14 sections`; `objdump -f /tmp/spacedock-win32.exe` confirmed `file format pei-i386` and `architecture: i386`.
- DONE: `make build-windows-386` and `make test-windows` exit 0 for windows/386.
  Satisfies AC-1 and AC-3: `make build-windows-386` compiled `dist/spacedock_windows_386.exe` (exit 0); `make test-windows` compiled all packages for windows/amd64 and windows/386 (exit 0) with zero leaky ifdefs added.
- SKIPPED: Run `spacedock.exe --version` on physical Windows machine.
  AC-2 deferred to captain's physical Windows smoke test after merge.

### Command Log & Acceptance Criteria Mapping

| Command | Exit Code / Output | AC Satisfied |
| --- | --- | --- |
| `GOOS=windows GOARCH=386 CGO_ENABLED=0 go build -o /tmp/spacedock-win32.exe ./cmd/spacedock/` | 0 | AC-1 |
| `file /tmp/spacedock-win32.exe` | `PE32 executable for MS Windows 6.01 (console), Intel i386, 14 sections` | AC-1 |
| `objdump -f /tmp/spacedock-win32.exe` | `file format pei-i386`, `architecture: i386` | AC-1 |
| `make build-windows-386` | 0 (`dist/spacedock_windows_386.exe` produced) | AC-1 |
| `make test-windows` | 0 (`GOOS=windows GOARCH=386 CGO_ENABLED=0 go build ./...` passed) | AC-3 |

### Summary

Cross-compilation for Win32 (GOARCH=386) was verified in the dedicated worktree. All Go packages compile cleanly with CGO disabled, producing valid PE32 Intel i386 executables without requiring code changes or platform ifdef additions (AC-1 and AC-3 passed; AC-2 deferred).

## Stage Report: validation

- DONE: Independently reproduce `GOOS=windows GOARCH=386 CGO_ENABLED=0 go build -o /tmp/spacedock-win32-val.exe ./cmd/spacedock/` exits 0 and `file /tmp/spacedock-win32-val.exe` confirms PE32 Intel i386 executable.
  Satisfies AC-1: `GOOS=windows GOARCH=386 CGO_ENABLED=0 go build` exited 0; `file` confirmed `PE32 executable for MS Windows 6.01 (console), Intel i386, 14 sections`; `objdump -f` confirmed `file format pei-i386`, `architecture: i386`.
- DONE: Independently reproduce `make build-windows-386` and `make test-windows` exit 0 across all packages.
  Satisfies AC-1 and AC-3: `make build-windows-386` produced `dist/spacedock_windows_386.exe` (exit 0); `make test-windows` verified compilation for `windows/amd64` and `windows/386` (exit 0).
- DONE: Verify `git diff main HEAD` in the code worktree shows zero platform ifdef leaks (or only valid //go:build additions).
  Satisfies AC-3: `git diff main HEAD` in worktree returned 0 diff lines; zero leaky platform ifdefs introduced.
- SKIPPED: Run `spacedock.exe --version` on physical Windows machine.
  AC-2 deferred to captain's physical Windows smoke test after merge.

### Summary

Independently verified Win32 cross-compilation (`GOARCH=386`) from scratch in the dedicated worktree. All compilation commands (`go build`, `make build-windows-386`, `make test-windows`, `go vet ./...`) exited 0, and inspection via `file` and `objdump -f` confirmed a valid PE32 Intel i386 executable. No platform ifdef leaks were found against `main`. Gate recommendation: PASSED (approve to done).


