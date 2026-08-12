---
id: 1xp4ea31vhdgp1r5vgxrvtwy
title: GoReleaser Windows targets (win32 + win64)
status: backlog
source: commission seed
started:
completed:
verdict:
score: 0.75
worktree:
issue:
pr:
mod-block:
---

## Problem

The `.goreleaser.yaml` release pipeline does not include Windows build targets. Even after the cross-compilation tasks confirm `GOOS=windows` builds correctly, users cannot obtain pre-built `.exe` binaries from the GitHub Releases page. This task adds `windows/386` and `windows/amd64` to the GoReleaser configuration so that stable and pre-release tags produce Windows artifacts automatically.

This task should only be promoted from backlog to ideation after both `win32-cross-compilation` and `win64-cross-compilation` tasks have reached `done`, since GoReleaser amplifies whatever the cross-compile tasks established.

## Proposed approach

Edit `.goreleaser.yaml` to add `windows` as a target OS with `386` and `amd64` architectures in the `builds` section. Verify the change locally with `goreleaser build --snapshot --clean` (or `goreleaser check`) to confirm the config is valid without actually running a full release. The resulting artifacts in `dist/` should include `spacedock_windows_386.exe` and `spacedock_windows_amd64.exe`.

No CGO changes needed — the goreleaser change inherits `CGO_ENABLED=0` from the environment or from the `env:` block in `.goreleaser.yaml`.

## Acceptance criteria

**AC-1 — GoReleaser config is valid with Windows targets added.**
Verified by: `goreleaser check` exits 0 (or `goreleaser build --snapshot --clean` exits 0). Would fail if the added `goos`/`goarch` entries are malformed or reference an unsupported combination.

**AC-2 — Snapshot build produces Windows 32-bit and 64-bit executables.**
Verified by: after `goreleaser build --snapshot --clean`, `file dist/spacedock_windows_386_v1/spacedock.exe` contains "PE32 executable" and `file dist/spacedock_windows_amd64_v1/spacedock.exe` contains "PE32+ executable". Would fail if the Windows artifacts are absent from `dist/` or have wrong PE type.

**AC-3 — Existing Linux and macOS targets are unaffected.**
Verified by: `goreleaser build --snapshot --clean` produces the same Linux/macOS artifact paths as before the change. Check with `ls dist/ | grep -v windows`. Would fail if an existing target is dropped or renamed.

**AC-4 — Windows artifacts appear in the release archive list (deferred: captain-windows-smoke).**
Verified by: on a future tag-triggered release, `gh release view vX.Y.Z --json assets` shows `spacedock_windows_386.exe` and `spacedock_windows_amd64.exe` in the asset list. Deferred to the next actual release cut after merge.

## Test plan

**Offline (Linux, reproducible by fresh agent):**
1. `goreleaser check` — exit 0 required (validates goreleaser YAML without building).
2. `goreleaser build --snapshot --clean` — exit 0 required.
3. `file dist/spacedock_windows_386_*/spacedock.exe` — must contain "PE32 executable".
4. `file dist/spacedock_windows_amd64_*/spacedock.exe` — must contain "PE32+ executable".
5. `ls dist/ | grep linux` — must match pre-change linux artifact count.

**Interactive (deferred — next release required):**
- After a real `vX.Y.Z` tag is pushed, confirm GitHub Release assets include both Windows `.exe` files.
- Download and run on a Windows machine to confirm they execute.

## Out of scope

- Windows code changes — handled by the win32 and win64 tasks.
- Windows installer packaging (`.msi`, NSIS, WiX) — not in scope.
- Windows CI/test lane in GitHub Actions — deferred.
- ARM64 Windows goreleaser target — not in scope for this workflow.
- Checksums/signatures for Windows artifacts — inherited from existing goreleaser config; no new config needed.
