# Backlog Gate Review — Win32 cross-compilation (GOARCH=386)

## Seed description
Cross-compile `spacedock` to `windows/386` (Intel 80386 32-bit PE executable) using Go's built-in cross-compilation (`GOOS=windows GOARCH=386 CGO_ENABLED=0`).

## Scope (included)
- `make build-windows-386` / `GOOS=windows GOARCH=386 CGO_ENABLED=0 go build` target
- PE32 Intel 80386 binary verification via `file` and `objdump -f`
- Compile check across all Go packages for `windows/386`

## Risk evidence
- `creack/pty` is restricted to `//go:build unix` (test file only)
- Existing `internal/cli/host_launch_other.go` (`//go:build !unix`) provides the required non-Unix shims
- `make build-windows-386` and `make test-windows` already verified working cleanly

## Recommendation
**Approve** — advance `win32-cross-compilation` to ideation.
