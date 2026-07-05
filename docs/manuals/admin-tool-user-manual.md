---
description: Full reference for fora-admin CLI â€” installation, all commands, JSON output, and automation.
---
# Fora.Rti.Admin User Manual

`fora-admin` is the headless remote administration CLI for `Fora.Rti.Server`. It connects to a running server over HTTP and executes one-off admin commands â€” suitable for automation, CI pipelines, and SSH sessions.

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

<pre><code class="language-bash">
dotnet tool install --global Fora.Rti.Admin
</code></pre>

After installation the `fora-admin` command is available on `PATH`.

### From source

<pre><code class="language-powershell">
dotnet run --project src\Fora.Rti.Admin\Fora.Rti.Admin.csproj -- --connect &lt;url&gt; &lt;command&gt;
</code></pre>

---

## Quick Start

<pre><code class="language-bash">
# Check server health
fora-admin --connect http://localhost:8080 health

# List active federations
fora-admin --connect http://localhost:8080 federations list

# Same commands with JSON output
fora-admin --connect http://localhost:8080 --format json health
fora-admin --connect http://localhost:8080 --format json federations list
</code></pre>

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

Supply the key one of two ways â€” the argument takes precedence when both are present:

<pre><code class="language-bash">
# Explicit flag
fora-admin --connect https://rti-host:8443 --token "$ADMIN_KEY" shutdown

# Environment variable (keeps the secret out of the process argument list â€” preferred for scripts and CI)
export FORA_RTI_TOKEN="s3cr3t-admin-key"
fora-admin --connect https://rti-host:8443 shutdown
</code></pre>

A missing or wrong key returns HTTP `401`; the CLI writes `Admin authentication failed (401 Unauthorized). Supply a valid --token or FORA_RTI_TOKEN.` to `stderr` and exits with code `1`.

> The Bearer token is transmitted in clear text over plain HTTP. Use an `https://` endpoint (or a trusted network boundary) whenever an admin API key is configured.

---

## Commands

### version

Displays server component name, package version, HLA standard, and capability level.

<pre><code class="language-bash">
fora-admin --connect http://localhost:8080 version
</code></pre>

**Text output:**
<pre><code>
Fora.Rti v20260603.0.0
Standard: HLA 4
HLA: 1516-2025
Capability: Full
</code></pre>

**JSON output (`--format json`):**
<pre><code class="language-json">
{"component":"Fora.Rti","packageVersion":"20260603.0.0","standard":"HLA 4","hla":"1516-2025","capabilityLevel":"Full"}
</code></pre>

---

### health

Checks whether the server is healthy. Returns exit code `0` when healthy, `2` when unhealthy.

<pre><code class="language-bash">
fora-admin --connect http://localhost:8080 health
</code></pre>

**Text output:** `Healthy` or `Unhealthy`

**JSON output:**
<pre><code class="language-json">
{"healthy":true}
</code></pre>

---

### shutdown

Requests a graceful server shutdown via `POST /admin/shutdown`.

<pre><code class="language-bash">
fora-admin --connect http://localhost:8080 shutdown
</code></pre>

**Text output:** `Shutdown requested.`

**JSON output:**
<pre><code class="language-json">
{"requested":true}
</code></pre>

---

### federations list

Lists the names of all active federation executions.

<pre><code class="language-bash">
fora-admin --connect http://localhost:8080 federations list
</code></pre>

**Text output:** One federation name per line.

**JSON output:**
<pre><code class="language-json">
["SpacecraftSimulation","GroundStation"]
</code></pre>

---

### federates list

Lists all joined federates with their name, federate type, and federation name.

<pre><code class="language-bash">
fora-admin --connect http://localhost:8080 federates list
</code></pre>

**Text output:** Tab-separated `Name`, `Type`, `Federation` â€” one federate per line.

**JSON output:**
<pre><code class="language-json">
[
  {"name":"Sensor1","type":"HLA_EVOLVED","federation":"SpacecraftSimulation"},
  {"name":"Control","type":"HLA_EVOLVED","federation":"SpacecraftSimulation"}
]
</code></pre>

---

### sessions list

Lists all Federate Protocol sessions with their connection state and last-seen timestamp.

<pre><code class="language-bash">
fora-admin --connect http://localhost:8080 sessions list
</code></pre>

**Text output:** Tab-separated `SessionId`, `ConnectionId`, `LastSeenUtc` (ISO 8601), `online|offline`.

**JSON output:**
<pre><code class="language-json">
[
  {"sessionId":"s-001","connectionId":"c-abc","lastSeenUtc":"2026-04-24T10:30:00Z","disconnectedAtUtc":null,"isOnline":true}
]
</code></pre>

---

### snapshot save

Saves a full RTI state snapshot to the configured `ISnapshotStore` under the given label. This bypasses the HLA save/restore state machine and can be invoked at any time.

<pre><code class="language-bash">
fora-admin --connect http://localhost:8080 snapshot save baseline
</code></pre>

**Text output:** `Saved snapshot 'baseline'.`

**JSON output:**
<pre><code class="language-json">
{"label":"baseline","saved":true}
</code></pre>

---

### snapshot restore

Restores RTI state from a previously saved snapshot. Returns exit code `1` if the label is not found.

<pre><code class="language-bash">
fora-admin --connect http://localhost:8080 snapshot restore baseline
</code></pre>

**Text output:** `Restored snapshot 'baseline'.`

**JSON output:**
<pre><code class="language-json">
{"label":"baseline","restored":true}
</code></pre>

**Label not found:** exits with code `1`; error message written to `stderr`.

---

### logs tail

Fetches the current log backlog and then streams new log entries via Server-Sent Events until interrupted (`Ctrl+C`).

<pre><code class="language-bash">
fora-admin --connect http://localhost:8080 logs tail
</code></pre>

**Text output:**
<pre><code>
2026-04-24T10:00:00.000Z [Information] Federation 'SpacecraftSimulation' created.
2026-04-24T10:00:01.000Z [Information] Federate 'Sensor1' joined.
</code></pre>

**JSON output (NDJSON â€” one JSON object per line):**
<pre><code class="language-json">
{"timestampUtc":"2026-04-24T10:00:00Z","level":"Information","message":"Federation 'SpacecraftSimulation' created."}
{"timestampUtc":"2026-04-24T10:00:01Z","level":"Information","message":"Federate 'Sensor1' joined."}
</code></pre>

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

<pre><code class="language-bash">
fora-admin --connect http://localhost:8080 health
if [ $? -ne 0 ]; then echo "RTI not healthy"; exit 1; fi
</code></pre>

### Save snapshot before test run

<pre><code class="language-bash">
fora-admin --connect http://rti-host:8080 snapshot save pre-test
</code></pre>

### Authenticated command against a secured server

<pre><code class="language-bash">
export FORA_RTI_TOKEN="s3cr3t-admin-key"
fora-admin --connect https://rti-host:8443 snapshot save pre-test
</code></pre>

### Parse federation list in a shell script

<pre><code class="language-bash">
fora-admin --connect http://localhost:8080 --format json federations list \
  | jq '.[]'
</code></pre>

### Log collection with timeout

<pre><code class="language-bash">
timeout 60 fora-admin --connect http://localhost:8080 --format json logs tail \
  &gt;&gt; /var/log/fora-rti.ndjson
</code></pre>

### PowerShell â€” check federation count

<pre><code class="language-powershell">
$feds = fora-admin --connect http://localhost:8080 --format json federations list | ConvertFrom-Json
Write-Host "Active federations: $($feds.Count)"
</code></pre>

---

## Disclaimer

Fora is provided as a research and educational environment. Review the project license and release notes before using it in production-like workflows.
