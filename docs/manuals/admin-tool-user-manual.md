---
description: Full reference for fora-admin CLI — installation, all commands, JSON output, and automation.
---
# Fora.Rti.Admin User Manual

`fora-admin` is the headless remote administration CLI for `Fora.Rti.Server`. It connects to a running server over HTTP and executes one-off admin commands — suitable for automation, CI pipelines, and SSH sessions. It does not start an RTI itself.

## Table of Contents
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Global Options](#global-options)
- [Authentication](#authentication)
- [Commands](#commands)
  - [version](#version)
  - [health](#health)
  - [shutdown](#shutdown)
  - [lsfed](#lsfed)
  - [lsfr](#lsfr)
  - [sessions](#sessions)
  - [save](#save)
  - [restore](#restore)
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

The server's admin API listens on `http://localhost:7112` by default (`Kestrel:Endpoints:Http:Url`); the container image uses port `8080`.

<pre><code class="language-bash">
# Check server health
fora-admin --connect http://localhost:7112 health

# List active federations
fora-admin --connect http://localhost:7112 lsfed

# Same commands with JSON output
fora-admin --connect http://localhost:7112 --format json health
fora-admin --connect http://localhost:7112 --format json lsfed
</code></pre>

---

## Global Options

<pre><code class="language-text">
fora-admin --connect &lt;url&gt; [--format json|text] [--token &lt;key&gt;] &lt;command&gt; [arguments]
</code></pre>

| Option | Description |
| :--- | :--- |
| `--connect <url>` | Base URL of the `Fora.Rti.Server` admin API (required for all commands, and must come first). Must be an absolute HTTP/HTTPS URL. |
| `--format json\|text` | Output format. `text` (default) writes human-readable lines; `json` writes machine-parseable JSON. |
| `--token <key>` | Admin API key, sent as an `Authorization: Bearer <key>` header. Required only when the server has an admin key configured. Falls back to the `FORA_RTI_TOKEN` environment variable; the argument wins when both are set. See [Authentication](#authentication). |
| `--help`, `-h` | Print usage and exit with code `0`. |

Command names are case-insensitive. `lsfed` is also available as `listfederations`, `lsfr` as `listjoinedfederates`.

---

## Authentication

By default the admin HTTP surface is unauthenticated. When the server sets `ForaRtiServer:Admin:ApiKey`, every `/admin/*` command requires a matching key; `version` and `health` stay open (they map to `/version` and `/health`, which are never gated).

Supply the key one of two ways — the argument takes precedence when both are present:

<pre><code class="language-bash">
# Explicit flag
fora-admin --connect https://rti-host:8443 --token "$ADMIN_KEY" shutdown

# Environment variable (keeps the secret out of the process argument list — preferred for scripts and CI)
export FORA_RTI_TOKEN="s3cr3t-admin-key"
fora-admin --connect https://rti-host:8443 shutdown
</code></pre>

A missing or wrong key returns HTTP `401`; the CLI writes `Admin authentication failed (401 Unauthorized). Supply a valid --token or FORA_RTI_TOKEN.` to `stderr` and exits with code `1`.

> The Bearer token is transmitted in clear text over plain HTTP. Use an `https://` endpoint (or a trusted network boundary) whenever an admin API key is configured.

---

## Commands

### version

Displays the server component, package version, HLA standard, and capability level.

<pre><code class="language-bash">
fora-admin --connect http://localhost:7112 version
</code></pre>

**Text output:**
<pre><code class="language-text">
Fora.Rti v20261004.0.0
  Standard : IEEE 1516-2025
  HLA      : HLA 4
  Capability: Phase7
</code></pre>

**JSON output (`--format json`):**
<pre><code class="language-json">
{
  "component": "Fora.Rti",
  "packageVersion": "20261004.0.0",
  "standard": "IEEE 1516-2025",
  "hla": "HLA 4",
  "capabilityLevel": "Phase7"
}
</code></pre>

---

### health

Checks whether the server's Federate Protocol listener is running. Exits with `1` when the server is unhealthy, so a script can gate on it.

<pre><code class="language-bash">
fora-admin --connect http://localhost:7112 health
</code></pre>

**Text output:**
<pre><code class="language-text">
Server: Healthy - FP listener is running.
Active sessions: 2
HLA authorization: Disabled (none)
</code></pre>

An unhealthy server prints `Server: Unhealthy - FP listener is not running.` to `stderr`.

**JSON output:**
<pre><code class="language-json">
{
  "healthy": true,
  "activeSessions": 2,
  "authorizationEnabled": false,
  "authorizationServiceName": "none"
}
</code></pre>

---

### shutdown

Requests a graceful server shutdown via `POST /admin/shutdown`.

<pre><code class="language-bash">
fora-admin --connect http://localhost:7112 shutdown
</code></pre>

**Text output:** `Shutdown initiated...`

**JSON output:**
<pre><code class="language-json">
{
  "requested": true
}
</code></pre>

---

### lsfed

Lists the names of all active federation executions. Alias: `listfederations`.

<pre><code class="language-bash">
fora-admin --connect http://localhost:7112 lsfed
</code></pre>

**Text output:** `Active Federations (2): SpacecraftSimulation, GroundStation` (`none` when there are none).

**JSON output:**
<pre><code class="language-json">
[
  "SpacecraftSimulation",
  "GroundStation"
]
</code></pre>

---

### lsfr

Lists all joined federates with their name, federate type, and federation name. Alias: `listjoinedfederates`.

<pre><code class="language-bash">
fora-admin --connect http://localhost:7112 lsfr
</code></pre>

**Text output:**
<pre><code class="language-text">
Joined Federates (2)
  - Sensor1 (SensorType) @ SpacecraftSimulation
  - Control (ControlType) @ SpacecraftSimulation
</code></pre>

**JSON output:**
<pre><code class="language-json">
[
  { "name": "Sensor1", "type": "SensorType", "federation": "SpacecraftSimulation" },
  { "name": "Control", "type": "ControlType", "federation": "SpacecraftSimulation" }
]
</code></pre>

---

### sessions

Lists all Federate Protocol sessions with their last-seen time and whether they are online.

<pre><code class="language-bash">
fora-admin --connect http://localhost:7112 sessions
</code></pre>

**Text output:**
<pre><code class="language-text">
FP Sessions (1)
  - 3f2c9a1e…  last=10:30:00  online
</code></pre>

**JSON output:**
<pre><code class="language-json">
[
  {
    "sessionId": "3f2c9a1e…",
    "connectionId": "c-abc",
    "lastSeenUtc": "2026-10-04T10:30:00+00:00",
    "disconnectedAtUtc": null,
    "isOnline": true
  }
]
</code></pre>

---

### save

Saves a full RTI state snapshot to the configured `ISnapshotStore` under the given label. This bypasses the HLA save/restore state machine and can be invoked at any time.

<pre><code class="language-bash">
fora-admin --connect http://localhost:7112 save baseline
</code></pre>

**Text output:** `Snapshot 'baseline' saved.`

**JSON output:**
<pre><code class="language-json">
{
  "label": "baseline",
  "saved": true
}
</code></pre>

Without a label the command prints `Usage: save <label>` to `stderr` and exits with `1`.

---

### restore

Restores RTI state from a previously saved snapshot.

<pre><code class="language-bash">
fora-admin --connect http://localhost:7112 restore baseline
</code></pre>

**Text output:** `Snapshot 'baseline' restored.`

**JSON output:**
<pre><code class="language-json">
{
  "label": "baseline",
  "restored": true
}
</code></pre>

**Label not found:** `Snapshot 'baseline' not found.` on `stderr`, exit code `1` (in JSON mode the result object shows `"restored": false`).

---

### logs tail

Fetches the current log backlog and then streams new log entries until interrupted (`Ctrl+C`).

<pre><code class="language-bash">
fora-admin --connect http://localhost:7112 logs tail
</code></pre>

**Text output:**
<pre><code class="language-text">
2026-10-04T10:00:00.0000000+00:00 [Success] Federation 'SpacecraftSimulation' created.
2026-10-04T10:00:01.0000000+00:00 [Success] Federate 'Sensor1' joined.
</code></pre>

**JSON output (NDJSON — one JSON object per line):**
<pre><code class="language-json">
{"TimestampUtc":"2026-10-04T10:00:00+00:00","Level":"Success","Message":"Federation 'SpacecraftSimulation' created."}
</code></pre>

> `logs tail` runs indefinitely. Use `Ctrl+C` or a process timeout to stop it.

---

## Output Formats

| Format | Flag | Use case |
| :--- | :--- | :--- |
| `text` (default) | `--format text` or omit | Human-readable terminal output. |
| `json` | `--format json` | Machine-parseable output for scripts, CI, and log aggregators. List commands produce a JSON array, the others a JSON object (indented, camelCase); `logs tail` produces NDJSON, one object per line. |

Error messages (an unhealthy server, a snapshot not found, a connection failure) are always written to `stderr` as plain text regardless of `--format`, so `stdout` stays parseable.

---

## Exit Codes

| Code | Meaning |
| :--- | :--- |
| `0` | Command succeeded. |
| `1` | Command failed: invalid arguments, unknown command, a missing label, a snapshot not found, an unhealthy server (`health`), a connection error, or an authentication failure. |

---

## Script Examples

### CI health gate

<pre><code class="language-bash">
fora-admin --connect http://localhost:7112 health || { echo "RTI not healthy"; exit 1; }
</code></pre>

### Save snapshot before test run

<pre><code class="language-bash">
fora-admin --connect http://rti-host:7112 save pre-test
</code></pre>

### Authenticated command against a secured server

<pre><code class="language-bash">
export FORA_RTI_TOKEN="s3cr3t-admin-key"
fora-admin --connect https://rti-host:8443 save pre-test
</code></pre>

### Parse federation list in a shell script

<pre><code class="language-bash">
fora-admin --connect http://localhost:7112 --format json lsfed | jq '.[]'
</code></pre>

### Log collection with timeout

<pre><code class="language-bash">
timeout 60 fora-admin --connect http://localhost:7112 --format json logs tail \
  &gt;&gt; /var/log/fora-rti.ndjson
</code></pre>

### PowerShell — check federation count

<pre><code class="language-powershell">
$feds = fora-admin --connect http://localhost:7112 --format json lsfed | ConvertFrom-Json
Write-Host "Active federations: $($feds.Count)"
</code></pre>

---

## Disclaimer

Fora is provided as a research and educational environment. Review the project license and release notes before using it in production-like workflows.
