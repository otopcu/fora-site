---
description: Full reference for fora-admin CLI — installation, all commands, JSON output, and automation.
---
# Fora.Rti.Admin User Manual

`fora-admin` is the headless remote administration CLI for `Fora.Rti.Server`. It connects to a running server over HTTP and executes one-off admin commands — suitable for automation, CI pipelines, and SSH sessions.

## Table of Contents
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Global Options](#global-options)
- [Authentication](#authentication)
- [Commands](#commands)
  - [version](#version)
  - [health](#health)
  - [shutdown](#shutdown)
  - [federations list](#federations-list)
  - [federates list](#federates-list)
  - [sessions list](#sessions-list)
  - [snapshot save](#snapshot-save)
  - [snapshot restore](#snapshot-restore)
  - [logs tail](#logs-tail)
- [Output Formats](#output-formats)
- [Exit Codes](#exit-codes)
- [Script Examples](#script-examples)
- [Disclaimer](#disclaimer)

---

## Installation

### As a .NET global tool

```bash
dotnet tool install --global Fora.Rti.Admin
```

After installation the `fora-admin` command is available on `PATH`.

### From source

```powershell
dotnet run --project src\Fora.Rti.Admin\Fora.Rti.Admin.csproj -- --connect <url> <command>
```

---

## Quick Start

```bash
# Check server health
fora-admin --connect http://localhost:8080 health

# List active federations
fora-admin --connect http://localhost:8080 federations list

# Same commands with JSON output
fora-admin --connect http://localhost:8080 --format json health
fora-admin --connect http://localhost:8080 --format json federations list
```

---

## Global Options

| Option | Description |
| :--- | :--- |
| `--connect <url>` | Base URL of the `Fora.Rti.Server` host (required for all commands). Must be an absolute HTTP/HTTPS URL. |
| `--format json\|text` | Output format. `text` (default) writes human-readable lines; `json` writes machine-parseable JSON. |
| `--token <key>` | Admin API key, sent as an `Authorization: Bearer <key>` header. Required only when the server has an admin key configured. Falls back to the `FORA_RTI_TOKEN` environment variable; the argument wins when both are set. See [Authentication](#authentication). |
| `--help`, `-h` | Print usage and exit with code `0`. |

---

## Authentication

By default the admin HTTP surface is unauthenticated. When the server sets `ForaRtiServer:Admin:ApiKey`, every `/admin/*` command requires a matching key; `version` and `health` stay open (they map to `/version` and `/health`, which are never gated).

Supply the key one of two ways — the argument takes precedence when both are present:

```bash
# Explicit flag
fora-admin --connect https://rti-host:8443 --token "$ADMIN_KEY" shutdown

# Environment variable (keeps the secret out of the process argument list — preferred for scripts and CI)
export FORA_RTI_TOKEN="s3cr3t-admin-key"
fora-admin --connect https://rti-host:8443 shutdown
```

A missing or wrong key returns HTTP `401`; the CLI writes `Admin authentication failed (401 Unauthorized). Supply a valid --token or FORA_RTI_TOKEN.` to `stderr` and exits with code `1`.

> The Bearer token is transmitted in clear text over plain HTTP. Use an `https://` endpoint (or a trusted network boundary) whenever an admin API key is configured.

---

## Commands

### version

Displays server component name, package version, HLA standard, and capability level.

```bash
fora-admin --connect http://localhost:8080 version
```

**Text output:**
```
Fora.Rti v20260603.0.0
Standard: HLA 4
HLA: 1516-2025
Capability: Full
```

**JSON output (`--format json`):**
```json
{"component":"Fora.Rti","packageVersion":"20260603.0.0","standard":"HLA 4","hla":"1516-2025","capabilityLevel":"Full"}
```

---

### health

Checks whether the server is healthy. Returns exit code `0` when healthy, `2` when unhealthy.

```bash
fora-admin --connect http://localhost:8080 health
```

**Text output:** `Healthy` or `Unhealthy`

**JSON output:**
```json
{"healthy":true}
```

---

### shutdown

Requests a graceful server shutdown via `POST /admin/shutdown`.

```bash
fora-admin --connect http://localhost:8080 shutdown
```

**Text output:** `Shutdown requested.`

**JSON output:**
```json
{"requested":true}
```

---

### federations list

Lists the names of all active federation executions.

```bash
fora-admin --connect http://localhost:8080 federations list
```

**Text output:** One federation name per line.

**JSON output:**
```json
["SpacecraftSimulation","GroundStation"]
```

---

### federates list

Lists all joined federates with their name, federate type, and federation name.

```bash
fora-admin --connect http://localhost:8080 federates list
```

**Text output:** Tab-separated `Name`, `Type`, `Federation` — one federate per line.

**JSON output:**
```json
[
  {"name":"Sensor1","type":"HLA_EVOLVED","federation":"SpacecraftSimulation"},
  {"name":"Control","type":"HLA_EVOLVED","federation":"SpacecraftSimulation"}
]
```

---

### sessions list

Lists all Federate Protocol sessions with their connection state and last-seen timestamp.

```bash
fora-admin --connect http://localhost:8080 sessions list
```

**Text output:** Tab-separated `SessionId`, `ConnectionId`, `LastSeenUtc` (ISO 8601), `online|offline`.

**JSON output:**
```json
[
  {"sessionId":"s-001","connectionId":"c-abc","lastSeenUtc":"2026-04-24T10:30:00Z","disconnectedAtUtc":null,"isOnline":true}
]
```

---

### snapshot save

Saves a full RTI state snapshot to the configured `ISnapshotStore` under the given label. This bypasses the HLA save/restore state machine and can be invoked at any time.

```bash
fora-admin --connect http://localhost:8080 snapshot save baseline
```

**Text output:** `Saved snapshot 'baseline'.`

**JSON output:**
```json
{"label":"baseline","saved":true}
```

---

### snapshot restore

Restores RTI state from a previously saved snapshot. Returns exit code `1` if the label is not found.

```bash
fora-admin --connect http://localhost:8080 snapshot restore baseline
```

**Text output:** `Restored snapshot 'baseline'.`

**JSON output:**
```json
{"label":"baseline","restored":true}
```

**Label not found:** exits with code `1`; error message written to `stderr`.

---

### logs tail

Fetches the current log backlog and then streams new log entries via Server-Sent Events until interrupted (`Ctrl+C`).

```bash
fora-admin --connect http://localhost:8080 logs tail
```

**Text output:**
```
2026-04-24T10:00:00.000Z [Information] Federation 'SpacecraftSimulation' created.
2026-04-24T10:00:01.000Z [Information] Federate 'Sensor1' joined.
```

**JSON output (NDJSON — one JSON object per line):**
```json
{"timestampUtc":"2026-04-24T10:00:00Z","level":"Information","message":"Federation 'SpacecraftSimulation' created."}
{"timestampUtc":"2026-04-24T10:00:01Z","level":"Information","message":"Federate 'Sensor1' joined."}
```

> `logs tail` runs indefinitely. Use `Ctrl+C` or a process timeout to stop it.

---

## Output Formats

| Format | Flag | Use case |
| :--- | :--- | :--- |
| `text` (default) | `--format text` or omit | Human-readable terminal output. |
| `json` | `--format json` | Machine-parseable output for scripts, CI, and log aggregators. List commands produce a JSON array; scalar commands produce a JSON object; `logs tail` produces NDJSON (one object per line). |

Error messages (e.g. snapshot not found, connection failure) are always written to `stderr` as plain text regardless of `--format`.

---

## Exit Codes

| Code | Meaning |
| :--- | :--- |
| `0` | Command succeeded. |
| `1` | Command failed (snapshot not found, invalid arguments, connection error, authentication failure, or unknown command). |
| `2` | Server reported unhealthy (`health` command only). |

Authentication failures (`401`/`403`) map to exit code `1` with a descriptive `stderr` message.

---

## Script Examples

### CI health gate

```bash
fora-admin --connect http://localhost:8080 health
if [ $? -ne 0 ]; then echo "RTI not healthy"; exit 1; fi
```

### Save snapshot before test run

```bash
fora-admin --connect http://rti-host:8080 snapshot save pre-test
```

### Authenticated command against a secured server

```bash
export FORA_RTI_TOKEN="s3cr3t-admin-key"
fora-admin --connect https://rti-host:8443 snapshot save pre-test
```

### Parse federation list in a shell script

```bash
fora-admin --connect http://localhost:8080 --format json federations list \
  | jq '.[]'
```

### Log collection with timeout

```bash
timeout 60 fora-admin --connect http://localhost:8080 --format json logs tail \
  >> /var/log/fora-rti.ndjson
```

### PowerShell — check federation count

```powershell
$feds = fora-admin --connect http://localhost:8080 --format json federations list | ConvertFrom-Json
Write-Host "Active federations: $($feds.Count)"
```

---

## Disclaimer

Fora is provided as a research and educational environment. Review the project license and release notes before using it in production-like workflows.
