# cs-stream
Signed-off-by: Guenther Alka gea@napp-it.org<br>
Concept Co-Authored-By: Claude Fable 5 noreply@anthropic.com<br>

Encrypted TCP stream transport for ZFS replication. Replaces ssh, mbuffer, netcat (nc) and pv in ZFS send/receive pipelines.

Part of [napp-it cs web-gui](https://www.napp-it.org) -- used by job-replicate and job-filesync for encrypted ZFS replication and rclone-over-SFTP folder sync between members.

Renamed from "zstream" as of v2.0.0 (avoids clashing with the unrelated
OpenZFS tool of the same name) -- functionally identical to zstream v1.3.3,
same wire protocol and CLI.

## Status

**v2.1.0.** `listen`/`send` (ZFS replication) and `tunnel-listen`/`tunnel-send`
(rclone-over-SFTP folder sync) are both implemented and live-deployed to
the official napp-it CS menu path
(`data/cs_server/tools/cs-stream/<os>.amd64/`) on all reachable cluster
members. `--allow-ip` is now **mandatory** on `listen` and `tunnel-listen`
(see Security Audit below) -- this is a breaking CLI change from v2.0.0.

### Changelog

**v2.1.0** (2026-07-28) -- security audit: mandatory `--allow-ip`, tunnel
session cap, rate/progress warning.

A full source audit (single-file `main.go`, not triggered by a live-test
failure) found and fixed three issues:

1. **No source-IP restriction on `listen`/`tunnel-listen`.** Both accepted
   connections from *any* source unconditionally. `listen` only ever
   accepts a single connection for the whole process lifetime -- an
   unauthenticated host reaching the port could connect first and occupy
   that one slot, blocking the legitimate sender (a pure availability/DoS
   issue; AES-256-GCM already prevented any actual data read or write
   without the key, so this was never a confidentiality or integrity gap).
   `tunnel-listen` is a persistent multi-session accept loop with no cap
   at all, so the exposure window was open for as long as the tunnel ran.
   [cs-sync](https://github.com/guenther-alka/cs-sync)'s sibling `serve`
   command has required `--allow-ip` since its own v2.1.0 for exactly this
   reason; cs-stream now matches that convention.
   **Fix:** `--allow-ip=IP` is now mandatory on `listen` and
   `tunnel-listen`. `doListen` loops on `Accept()`, rejecting and closing
   any non-matching connection and continuing to wait rather than
   consuming the single accept slot -- the original accept-timeout
   deadline keeps counting down across every attempt, so a flood of
   wrong-IP connections cannot extend the window and delay or starve the
   real sender. `doTunnelListen` checks the allow-listed IP per session in
   its existing accept loop.
2. **No cap on concurrent `tunnel-listen` sessions.** Added
   `maxTunnelSessions = 64` with an atomic counter -- defense in depth
   alongside the `--allow-ip` check, bounding resource use even from the
   allow-listed IP.
3. **`--rate` and `--progress` were silently ignored in tunnel mode**
   (unlike `--buf`, which has an explicit architectural justification in
   its own doc comment for why it doesn't apply there). Now logs an
   explicit one-time `WARNING` instead of silently no-op'ing.

**Verified:** `go build`/`go vet`/`staticcheck` all clean; cross-compiled
clean for all 8 release platforms; live-tested on all 5 reachable cluster
members -- see **Security audit & live test results** below.

## Usage

```bash
# Receiver (start first) -- --allow-ip is REQUIRED as of v2.1.0:
cs-stream listen 9000 SESSIONKEY --allow-ip=192.168.1.20 | zfs receive -F tank/backup

# Sender:
zfs send tank@snap | cs-stream send 192.168.1.10 9000 SESSIONKEY

# With options (rate limit, progress, log):
zfs send tank@snap | cs-stream send 192.168.1.10 9000 SESSIONKEY --rate=100m --progress --log=/tmp/cs-stream.log
```

## Options

| Option | Default | Description |
|--------|---------|-------------|
| `--buf=SIZE` | `128m` | Read-ahead buffer (0 = off). Units: k/m/g |
| `--rate=SPEED` | off | Throughput limit per second. Units: k/m/g (not supported in tunnel mode) |
| `--progress` | off | Live progress on stderr (not supported in tunnel mode) |
| `--log=FILE` | off | Append transfer summary to file |
| `--bind=IP` | `0.0.0.0` | Bind listener to specific IP |
| `--allow-ip=IP` | *(none)* | **Required** on `listen`/`tunnel-listen`: only this source IP may connect |

### Tunnel mode
```
# Folder sync via rclone over encrypted tunnel.
# --local=ADDR and --allow-ip=IP are both required on tunnel-listen:
# --local is the local address the wrapped child process binds to
# (tunnel-listen) or connects to (tunnel-send); --allow-ip is the only
# source IP tunnel-listen will accept sessions from.

# Receiver (start first):
cs-stream tunnel-listen 9000 KEY --local=127.0.0.1:2222 --allow-ip=192.168.1.10 -- rclone serve sftp /dst --addr 127.0.0.1:2222

# Sender:
cs-stream tunnel-send HOST 9000 KEY --local=127.0.0.1:2222 -- rclone sync /src :sftp: --sftp-host=127.0.0.1 --sftp-port=2222
```
## Encryption

- **AES-256-GCM** authenticated encryption (Go stdlib, AES-NI accelerated)
- Session key per transfer (generated by job-replicate.pl / job-filesync.pl)
- Each 64KB chunk independently encrypted with random nonce
- Authentication tag detects any tampering or corruption
- `--allow-ip` (required on `listen`/`tunnel-listen`) restricts which
  source IP may even attempt a handshake
- Accept timeout (120s) prevents listener from waiting forever
- Read timeout (5 min) detects dead sender or network failure
- `--bind=IP` restricts listener to specific interface

## Security audit & live test results (v2.1.0)

Full source read of `main.go` (no `_test.go` files exist in this repo --
noted as a gap for a future session, same finding as
[cs-sync](https://github.com/guenther-alka/cs-sync)'s own audit).
`go vet`/`staticcheck` clean before and after. The three fixes above were
each verified functionally, then live-tested across the full napp-it CS
cluster platform matrix after deploying v2.1.0 binaries everywhere.

| Test | What it checks |
|---|---|
| A | `listen`/`tunnel-listen` refuse to start at all without `--allow-ip` |
| B | A connection from a non-matching IP is refused and logged; the listener keeps running (does not exit or consume its one accept slot) |
| C | A connection from the correct allow-listed IP succeeds end-to-end, payload decrypts correctly |

| Host | OS | Test A | Test B | Test C |
|---|---|---|---|---|
| my-w11 | Windows | pass | pass | pass |
| .112 | Linux | pass | pass | pass |
| .191 | FreeBSD | pass | pass | pass |
| .196 | macOS | pass | pass | pass |
| **.189** | **OmniOS/illumos** | pass | pass | pass |

All five reachable cluster members confirmed clean, zero errors beyond
the intentional refusal log lines. `cs-stream version` confirmed v2.1.0
on every host before testing.

Reviewed but not flagged as actionable (documented for a future audit so
the same analysis isn't repeated):

- The nonce/key lifetime: random 96-bit nonces, fresh per 64KB chunk,
  under a static key that never rotates across reconnects. The
  birthday-bound collision risk for random 96-bit nonces becomes
  non-negligible around 2^32 messages under one key -- at 64KB/chunk
  that's on the order of 256 terabytes of lifetime traffic under a single
  static key to become a real concern, not realistically reachable for
  this tool's actual use case (one key per transfer, generated fresh by
  job-replicate.pl/job-filesync.pl for every job run). Noted as a
  defense-in-depth/forward-secrecy observation, not an actionable
  vulnerability.
- `deriveKey` uses a plain SHA-256 hash rather than a proper KDF (HKDF/
  PBKDF2) to expand the session key string to 32 bytes. Acceptable given
  the actual key material is always a freshly-generated
  `sha256_hex(time+ips+snap)` string from the Perl caller (high entropy
  already), not a user-chosen password; a proper KDF would be a
  hardening nice-to-have, not a fix for a live weakness.

## Download

Pre-built binaries for all platforms in [Releases](https://github.com/guenther-alka/cs-stream/releases/latest):

| Platform | File |
|----------|------|
| macOS x86_64 | `cs-stream-darwin-amd64` |
| macOS ARM64 | `cs-stream-darwin-arm64` |
| Linux x86_64 | `cs-stream-linux-amd64` |
| Linux ARM64 | `cs-stream-linux-arm64` |
| Windows x86_64 | `cs-stream-windows-amd64.exe` |
| FreeBSD x86_64 | `cs-stream-freebsd-amd64` |
| Illumos x86_64 | `cs-stream-illumos-amd64` |
| OmniOS/Solaris x86_64 | `cs-stream-solaris-amd64` |

## Build

```bash
git clone https://github.com/guenther-alka/cs-stream.git
cd cs-stream
go build -o cs-stream .

# All platforms:
GOOS=linux   GOARCH=amd64 go build -o cs-stream-linux-amd64 .
GOOS=windows GOARCH=amd64 go build -o cs-stream-windows-amd64.exe .
GOOS=freebsd GOARCH=amd64 go build -o cs-stream-freebsd-amd64 .
GOOS=solaris GOARCH=amd64 go build -o cs-stream-solaris-amd64 .
```

Requires Go 1.21+. No external dependencies.

## Install (csweb-gui)

Place binary in `data/cs_server/tools/cs-stream/<os>.amd64/cs-stream` (Linux/BSD/OmniOS/Solaris)
or `data\cs_server\tools\cs-stream\mswin.amd64\cs-stream.exe` (Windows).

csweb-gui deploys and self-heals this automatically per member; manual
placement is only needed to seed the admin host itself with a new version.

See `.github/workflows/cs-stream.info` for full documentation.

## Standalone setup without napp-it csweb-gui

cs-stream is a self-contained, single-file Go binary with no dependency
on a napp-it CS installation at all -- unlike its sibling
[cs-sync](https://github.com/guenther-alka/cs-sync) (which has one
opt-in `--service-id` exception writing to a fixed `/opt` path),
cs-stream's `main.go` has no `/opt` or napp-it reference whatsoever. It
is a generic encrypted stream transport: `listen`/`send` pipe stdin to
stdout like an encrypted netcat, `tunnel-listen`/`tunnel-send` wrap an
arbitrary local TCP service (e.g. rclone's SFTP server). Everything is
driven by CLI flags and the session key you supply.

Build and run standalone:

```bash
git clone https://github.com/guenther-alka/cs-stream.git
cd cs-stream
go build -o cs-stream .

# receiver:
cs-stream listen 9000 SESSIONKEY --allow-ip=192.168.1.20 > out.bin
# sender:
cs-stream send 192.168.1.10 9000 SESSIONKEY < in.bin
```

No config file, registry entry, or napp-it environment is expected --
only a matching session key on both ends and, as of v2.1.0,
`--allow-ip` on the receiving side.

## Warranty

napp-it cs-sync/stream is open source. You may use, analyze, or modify
it free of charge. You use it "as is" and bear sole responsibility for
its use. This note does not replace the BSD 2-Clause license terms
below -- it summarizes them in plain language.

## License

BSD 2-Clause License -- Copyright (c) 2026 Guenther Alka / napp-it.org

Redistribution in source or binary form requires retaining the copyright notice.
See [LICENSE](LICENSE) for full terms.
