---
id: nvtfxc3an0mphhf4q2e9zkse
title: Win32 cross-compilation (GOARCH=386)
status: ideation
source: commission seed
started:
completed:
verdict:
score: 0.9
worktree:
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

## Out of scope

- 64-bit Windows (GOARCH=amd64) — separate task.
- GoReleaser pipeline changes — separate task.
- Windows installer (`.msi`, Chocolatey, Winget) — not in scope for this workflow.
- Cygwin compatibility — not tested; MinGW cross-compile only.
- Windows CI lane — deferred; this PR does not add a CI workflow for Windows.
