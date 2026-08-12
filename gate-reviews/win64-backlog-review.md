# Backlog Gate Review — Win64 cross-compilation (GOARCH=amd64)

## Seed description
Cross-compile spacedock to `windows/amd64` using Go's native cross-compilation. Verify on Linux using `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build` plus `file`/`objdump` inspection of the produced `.exe`. Final smoke test deferred to a physical Windows machine.

## Scope (included)
- Go build flag changes and any required build-tag guards for `GOOS=windows`
- Audit of `creack/pty` dependency (Unix-only PTY) for Windows compatibility
- Offline cross-compile verification producing a PE32+ executable

## Scope (excluded)
- Win32 (386) — separate task
- GoReleaser pipeline — separate task
- Windows CI lane
- Windows installer

## Risk evidence
- `creack/pty` uses CGO for pseudo-terminal on Unix; needs confirming it either has a no-op Windows stub or that `CGO_ENABLED=0` excludes it entirely
- No other dependencies use CGO (pure Go: cobra, pflag, yaml, json-canonicalization)

## Offline proof for each AC
- **AC-1**: `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go build -o spacedock-win64.exe ./cmd/spacedock/` + `file spacedock-win64.exe` → "PE32+ executable … x86-64"
- **AC-2**: `GOOS=windows GOARCH=amd64 CGO_ENABLED=0 go test -run=^$ ./...` → exit 0 (compile check)
- **AC-3**: deferred to captain Windows smoke test
- **AC-4**: `git diff --stat main HEAD` review of any new `//go:build windows` additions

## Recommendation
**Advance to ideation.** This is the highest-priority target (score 0.95), pure Go cross-compile is low-risk, and the pty dependency is the only unknown — ideation should resolve it in the proposed approach. No blockers to design starting.
