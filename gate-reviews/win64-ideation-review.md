# Ideation Gate Review — Win64 cross-compilation (GOARCH=amd64)

## Selected approach
`GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build -o spacedock-win64.exe ./cmd/spacedock/`
No MinGW cross-compiler needed. No source code changes expected.

## Risk evidence resolved

| Risk | Status | Evidence |
|---|---|---|
| `creack/pty` CGO dependency | ✅ RESOLVED | `host_launch_pty_test.go:3: //go:build unix` — test-only, excluded from Windows build |
| No Windows stub for signal model | ✅ ALREADY EXISTS | `host_launch_other.go:3: //go:build !unix` — no-op shims committed |
| `os/exec` usage | ✅ Safe | Standard library, cross-platform |
| `go` toolchain absent on dev machine | ⚠️ Prerequisite | Worker must install Go 1.22+ before cross-compiling |

## Offline AC proof (each can fail)
- **AC-1:** `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build -o spacedock-win64.exe ./cmd/spacedock/` → exit 0 + `file spacedock-win64.exe` → "PE32+ executable … x86-64". Fails if any import is missing or CGO surfaces.
- **AC-2:** `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go test -run=^$ ./...` → exit 0. Fails if any package references Unix-only syscalls without a build tag.
- **AC-4:** `git diff --stat` — expect empty diff or only targeted `//go:build` additions. Fails if shared packages are modified.

## Interactive ACs (deferred)
- **AC-3:** `spacedock-win64.exe --version` on physical Windows 64-bit machine → declared deferred to captain's Windows smoke test.

## Out of scope confirmed
Win32, GoReleaser, CI lane, ARM64 Windows, installer.
