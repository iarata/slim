# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Slim is a Go CLI tool that maps custom local HTTPS domains to development server ports. It manages TLS certificates, a background reverse-proxy daemon, hosts file entries, and port forwarding. It also supports tunneling local services to public URLs via WebSocket.

## Build & Test Commands

```bash
make build          # Build binary with version from git tags
make install        # Build and install to /usr/local/bin/
make test           # Run all tests (go test ./...)
make clean          # Remove built binary
go test ./cmd/...   # Run tests for a single package
go test -run TestName ./cmd/...  # Run a specific test
go vet ./...        # Static analysis
golangci-lint run   # Lint (used in CI)
```

Requires Go 1.25+. CI runs `go build`, `go test`, `go vet`, and `golangci-lint` on PRs.

## Architecture

### Entry Point & CLI Layer (`cmd/`)

- `main.go` → `cmd.Execute()` — Cobra-based CLI
- Each subcommand (`start`, `stop`, `up`, `down`, `share`, `list`, `logs`, `doctor`, etc.) is a separate file in `cmd/`
- Version injected via `-ldflags` at build time into `cmd.Version`

### Core Flow (start a domain)

1. **Setup** (`internal/setup`) — First-run: generates CA cert, trusts it in OS keychain, configures port forwarding
2. **Config** (`internal/config`) — Reads/writes `~/.slim/config.yaml` with file locking (`syscall.Flock`). Domain names auto-get `.test` suffix if no TLD specified
3. **System** (`internal/system`) — Manages `/etc/hosts` entries (marked with `# slim`) and port forwarding via `pfctl` on macOS (80→10080, 443→10443)
4. **Cert** (`internal/cert`) — CA is RSA 2048 (10yr), leaf certs are ECDSA P-256 (825 days). Auto-renewed if <30 days to expiry
5. **Daemon** (`internal/daemon`) — Background process via go-daemon. IPC over Unix socket (`~/.slim/slim.sock`) with JSON messages (status/reload/shutdown). PID stored in `~/.slim/slim.pid`
6. **Proxy** (`internal/proxy`) — HTTP (10080) + HTTPS (10443) reverse proxy. Uses singleflight for cert loading, longest-prefix path matching for routes, optional CORS injection

### Tunnel System (`internal/tunnel` + `protocol/`)

- WebSocket client connects to remote server, registers with token/subdomain
- Binary frame protocol: `[4-byte requestID][serialized HTTP request/response]`
- Reconnects with exponential backoff (1s→30s), 20s ping keepalive
- Auth managed via `internal/auth` (OAuth flow, stored in `~/.slim/auth.json`)

### State Directory

All runtime state lives in `~/.slim/` — config, certs, logs, sockets, PID file, auth tokens.

### Platform Integration

- macOS: `security add-trusted-cert` for CA trust, `pfctl` for port forwarding
- Linux: system cert store, iptables (implied)
- Privileged operations use auto-sudo via `internal/osutil`
