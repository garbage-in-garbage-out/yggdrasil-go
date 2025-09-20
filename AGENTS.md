# Yggdrasil-Go Agent Guide

## Living document expectations
- This file is the coordination point for AI agents working in this repository. Treat it as a living document: whenever you learn something new about the project structure, conventions, or workflows that would help other agents, update this file in the same change.
- Keep instructions actionable and concise. When you change repository behaviour (e.g. add a new package, command, build step, or convention) update the relevant section below.
- Follow the repository contribution flow in the **Agent workflow** section. Extend that list if new recurring tasks appear.

## Project overview
- This repository contains the Go implementation of the [Yggdrasil](https://yggdrasil-network.github.io) end-to-end encrypted IPv6 overlay network. The source lives in a Go module named `github.com/yggdrasil-network/yggdrasil-go` targeting Go 1.23+ with the Go 1.24 toolchain. 【F:go.mod†L1-L38】
- Builds embed version metadata through `./build`, which shells out to `contrib/semver/{name,version}.sh` and injects linker flags into binaries under `cmd/`. 【F:build†L1-L37】【F:contrib/semver/name.sh†L1-L21】【F:contrib/semver/version.sh†L1-L10】
- Primary deliverables are CLI applications: the node daemon (`cmd/yggdrasil`), the admin client (`cmd/yggdrasilctl`), and the key generator (`cmd/genkeys`). Supporting packages under `src/` implement networking, configuration, and platform abstractions. 【F:cmd/yggdrasil/main.go†L1-L120】【F:cmd/yggdrasilctl/main.go†L1-L120】【F:cmd/genkeys/main.go†L1-L80】

## Directory layout highlights
- `cmd/`
  - `yggdrasil`: bootstraps the node—parses flags, reads HJSON/JSON config via `src/config`, initialises `core.Core`, TUN, multicast discovery, and the admin socket, and manages logging. It uses `suah.dev/protect` to drop privileges and prints build metadata with `src/version`. 【F:cmd/yggdrasil/main.go†L1-L160】
  - `yggdrasilctl`: admin CLI that pledges restricted syscalls, connects to the admin socket, and renders JSON or tabular output for handlers registered in `src/admin`. 【F:cmd/yggdrasilctl/main.go†L1-L160】
  - `genkeys`: generates Ed25519 node or signing keys in parallel, ranking candidates by deterministic scoring. 【F:cmd/genkeys/main.go†L1-L80】
- `src/`
  - `core`: actor-based node implementation. `core.Core` owns the encrypted `ironwood` PacketConn, manages TLS, listeners, links, and node info, using `github.com/Arceliar/phony` inboxes for serialised state transitions. Link implementations cover TCP, TLS, QUIC, UNIX sockets, SOCKS, and WebSocket transports with per-OS specialisations. 【F:src/core/core.go†L1-L120】【F:src/core/link.go†L1-L120】
  - `admin`: JSON admin socket server layered on top of `core`, exposing handlers registered with metadata for introspection. Handles UNIX/TCP listeners and enumerates command help. 【F:src/admin/admin.go†L1-L160】
  - `config`: generates default configuration, including platform-specific defaults via build-tagged files (`defaults_*.go`). 【F:src/config/config.go†L1-L40】【F:src/config/defaults_linux.go†L1-L80】
  - `address`: IPv6 address derivation utilities and tests around node/subnet IDs. 【F:src/address/address.go†L1-L80】
  - `ipv6rwc`: wrappers for treating IPv6 packets as `io.ReadWriteCloser` streams plus ICMPv6 helpers. 【F:src/ipv6rwc/ipv6rwc.go†L1-L80】
  - `multicast`: node discovery via multicast advertisements with per-platform implementations (Darwin CGO, Windows, UNIX). 【F:src/multicast/multicast.go†L1-L120】
  - `tun`: cross-platform TUN/TAP handling built atop WireGuard’s `tun` package and `vishvananda/netlink`, with per-OS build tags such as `tun_linux.go`. 【F:src/tun/tun_linux.go†L1-L48】
  - `version`: stores build name/version string variables populated at link time. 【F:src/version/version.go†L1-L20】
- `contrib/`: packaging, init scripts, Docker assets, mobile bindings, etc. Keep additions consistent with platform expectations.
- `misc/`: helper scripts for development/testing (e.g. namespace experiments).
- `build` & `clean`: convenience scripts for building release binaries and resetting the tree. 【F:build†L1-L37】【F:clean†L1-L3】
- `Dockerfile`: points to `contrib/docker/Dockerfile`, which builds static binaries and assembles a runtime image. 【F:Dockerfile†L1-L1】【F:contrib/docker/Dockerfile†L1-L22】

## Key libraries and patterns
- **Actor model**: Packages such as `core` and `core/links` embed `phony.Inbox`. Mutate actor-owned fields only inside `Inbox.Act`/`phony.Block` callbacks to preserve serialised access. Expose helper methods instead of exporting raw fields. 【F:src/core/core.go†L24-L60】【F:src/core/link.go†L1-L64】
- **Encryption & transport**: Uses `github.com/Arceliar/ironwood` for encrypted packet sessions with bloom filters and path notifications, plus TLS (`crypto/tls`) for authenticated links. 【F:src/core/core.go†L1-L80】
- **Logging**: Standard logging goes through `github.com/gologme/log`, frequently with level enabling (info/warn/error/debug). Tests often call `GetLoggerWithPrefix` to configure verbosity. 【F:src/core/core.go†L14-L44】【F:src/core/core_test.go†L1-L40】
- **Configuration**: `config.GenerateConfig` creates structs with TLS certs, keys, listeners, and defaults; OS-specific defaults live in build-tagged files, so maintainers must update all relevant platforms when adding new options. 【F:src/config/config.go†L1-L80】【F:src/config/defaults_linux.go†L1-L80】
- **Privilege controls**: CLI binaries call `suah.dev/protect.Pledge` to restrict syscalls; maintain compatibility when adding privileged operations. 【F:cmd/yggdrasil/main.go†L1-L80】【F:cmd/yggdrasilctl/main.go†L1-L40】
- **Testing helpers**: `core/core_test.go` provides helper constructors (`CreateAndConnectTwo`, `WaitConnected`) that spin up in-process nodes and network connections. Prefer using these when adding integration-style tests. 【F:src/core/core_test.go†L1-L80】

## Agent workflow and best practices
1. **Plan the change**
   - Identify affected packages (e.g. CLI in `cmd/`, actor logic in `src/core`, platform code in `src/tun`). Account for OS-specific build tags when touching platform abstractions.
   - Update this `AGENTS.md` whenever your change introduces or alters conventions, tooling, or directory structure.
2. **Implement**
   - Write Go code inside the existing module; new packages belong under `src/` unless they are standalone tools.
   - Maintain actor safety: interact with `phony` inbox owners via methods that enqueue work rather than touching fields directly.
   - Keep logging consistent (use `log.Logger` from `github.com/gologme/log` and honour log level conventions).
   - When adding configuration fields, update all `defaults_*.go` files, CLI flag parsing in `cmd/yggdrasil`, and ensure admin handlers expose the new settings if applicable.
   - For new transports or platform features, mirror existing file naming/build tags and document OS support in README or contrib assets.
3. **Formatting & linting**
   - Run `go fmt ./...` on any Go changes.
   - Ensure imports stay organised (use `gofmt` or `goimports`).
4. **Testing**
   - Run `go test ./...` for code changes. Some tests establish local TCP listeners and may take a few seconds but do not require elevated privileges. 【F:src/core/core_test.go†L1-L80】
   - When CLI behaviour changes, add or update unit tests where feasible and document user-facing modifications (e.g. README, CHANGELOG) as required by the feature owner.
   - For release or packaging changes, verify `./build` succeeds or update Docker/packaging scripts accordingly.
5. **Review checklist before committing**
   - Code formatted and tests passing.
   - Updated documentation (`README`, configs, this file) for new features/flags.
   - Consider adding admin API entries (`src/admin`) and CLI output changes if functionality is user-facing.
   - Ensure build tags remain correct and cross-platform defaults stay in sync.

## When to update related docs
- **README.md**: changes that affect installation, configuration flags, or usage instructions.
- **CHANGELOG.md**: user-visible behaviour changes or notable bug fixes (follow existing changelog style).
- **contrib/** assets: if new behaviour affects packaging, init scripts, or container images.
- **misc/** scripts: update or document helper scripts when adding new development workflows.

## Open questions / future enhancements
Document new conventions or TODOs here so future agents can resolve them:
- _None yet. Add entries as you discover gaps or recurring pain points._
