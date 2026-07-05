---
description: Full reference for the fora-console interactive TUI — in-process and remote mode, commands, layout.
---
# Fora.Rti.Console User Manual

`Fora.Rti.Console` is the interactive Terminal User Interface (TUI) for `Fora.Rti`. It provides a live dashboard for local development and remote server diagnostics — either by starting the RTI in-process or by connecting to an already-running `Fora.Rti.Server` instance.

## Table of Contents
- [When to Use Console](#when-to-use-Console)
- [Starting Console](#starting-Console)
  - [In-Process Mode](#in-process-mode)
  - [Remote Mode](#remote-mode)
- [TUI Layout](#tui-layout)
- [Commands](#commands)
- [Usage Examples](#usage-examples)
- [Runtime Notes](#runtime-notes)
- [Disclaimer](#disclaimer)

---

## When to Use Console

| Scenario | Recommended tool |
| :--- | :--- |
| Local development with a live dashboard | `Fora.Rti.Console` (in-process mode) |
| Attaching a UI to a running Docker/cloud RTI | `Fora.Rti.Console --connect <url>` (remote mode) |
| Headless server (CI, Docker, background service) | `Fora.Rti.Server` |
| One-off remote commands from a script | `fora-admin` — see [Fora.Rti.Admin User Manual](admin-tool-user-manual.md) |

---

## Starting Console

### In-Process Mode

Starts the RTI server and the TUI in a single process. Federates connect to the same Federate Protocol endpoint that the embedded server listens on.

```powershell
dotnet run --project src\Fora.Rti.Console\Fora.Rti.Console.csproj
```

Or, if installed from a published build:

```bash
Fora.Rti.Console
```

The server configuration (FP port, bind address, save path) is read from `appsettings.json` in the Console output directory. Defaults:

| Setting | Default |
| :--- | :--- |
| `FederateProtocolPort` | `15164` |
| `FederateProtocolBindAddress` | `localhost` |
| `SavePath` | `saves` |

### Remote Mode

Attaches Console as a management UI to an already-running `Fora.Rti.Server` instance. No in-process RTI is started.

```powershell
dotnet run --project src\Fora.Rti.Console\Fora.Rti.Console.csproj -- --connect http://localhost:8080
```

Or against a remote host:

```bash
Fora.Rti.Console --connect http://my-rti-host:8080
```

In remote mode:
- All commands execute against the remote server via `/admin/*` and `/version` HTTP endpoints.
- The TUI header (federation count, federate count, version) refreshes every second.
- The log panel seeds from `GET /admin/logs` and then tails `GET /admin/logs/stream` with automatic reconnect.
- `exit` / `quit` closes Console only — the remote server keeps running.

---

## TUI Layout

```
┌ Fora.Rti.Console ────────────────────────────────────────────────┐
│ 10:00:01 [INFO   ] Federation 'Sim' created.                     │
│ 10:00:02 [INFO   ] Federate 'Sensor1' joined federation 'Sim'.   │
│ 10:00:03 [INFO   ] Federate 'Control' joined federation 'Sim'.   │
│                                                                  │
│                                                                  │
│ ──────────────────────────────────────────────────────────────── │
│ Command > _                                                      │
├──────────────────────────────────────────────────────────────────┤
│ F1 Help | F2 Filter: ALL | Ctrl+F Search | F10 Exit              │
└──────────────────────────────────────────────────────────────────┘
```

**System logs** — a full-width, dedicated `TextView` area for RTI logs and command output:
- **Auto-scroll:** New entries automatically scroll the view to the bottom.
- **Selection:** Supports text selection using Mouse or `Shift + Arrow Keys`.
- **Copy:** Supports standard `Ctrl + C` for copying selected logs.
- **Navigation:** Supports PageUp/PageDown and Scrollbar navigation.
- **Filtering:** Press `F2` to cycle through log levels (ALL, INFO, SUCCESS, WARN, ERROR).
- **Searching:** Press `Ctrl+F` to toggle **Inline Search**. Type to filter logs in real-time. Press `Esc` or `Ctrl+F` again to clear the search and return to command mode. Both active filters are displayed in the status bar.

**Command input** — fixed bottom area separated by a line:
- **Prompt:** A distinct `Command >` prompt with a high-contrast input field.
- **Command History:** Use **Up/Down Arrow Keys** to navigate through previously entered commands.
- **Execution:** Press **Enter** to execute.

**Status bar** — bottom bar, refreshed every second:
- Interactive shortcuts (`F1 Help`, `F2 Filter`, `Ctrl+F Search`, `F10 Exit`).
- Active search pattern (if any).
- Server version.
- Active federation execution count.
- Joined federate count.

---

## Commands

| Command | Alias | Description |
| :--- | :--- | :--- |
| `version` | | Displays server-side version information (Component, Standard, HLA, CapabilityLevel). |
| `health` | | Queries the RTI health check and prints `Healthy` or `Unhealthy`. |
| `lsfed` | `listfederations` | Lists all active federation executions. |
| `lsfr` | `listjoinedfederates` | Lists all joined federates with their type and federation name. |
| `sessions` | | Lists all active Federate Protocol sessions. |
| `save <label>` | | Saves a full RTI state snapshot with the given label. |
| `restore <label>` | | Restores RTI state from a previously saved snapshot. |
| `shutdown` | | Initiates a graceful server shutdown. |
| `help` | `?` | Prints available commands into the log area. |
| `exit` | `quit` | Exits Console. |

---

## Usage Examples

### Check server health

```
> health
Healthy
```

### List active federations

```
> lsfed
Sim
GroundStation
```

### List joined federates

```
> lsfr
Sensor1    HLA_EVOLVED    Sim
Control    HLA_EVOLVED    Sim
Relay      HLA_EVOLVED    GroundStation
```

### Save a snapshot before a test run

```
> save pre-test
Saved snapshot 'pre-test'.
```

### Restore from a snapshot

```
> restore pre-test
Restored snapshot 'pre-test'.
```

### Graceful shutdown

```
> shutdown
Shutdown requested.
```

### Monitoring live connections

As federates connect and disconnect, log entries appear automatically:

```
10:05:01 [INFO] Accepted FP TCP connection from 127.0.0.1:49672.
10:05:02 [INFO] Federate 'Sensor1' joined federation 'Sim'.
10:05:45 [INFO] Federate 'Sensor1' resigned from federation 'Sim'.
```

---

## Runtime Notes

- **Terminal.Gui (gui.cs)** — Uses a professional TUI library that provides true windowing, event-driven input, and robust cross-platform terminal handling (avoiding standard ANSI redraw issues).
- **Thread isolation** — UI rendering and input handling run on a dedicated main loop thread, isolated from the RTI message loop.
- **Async logging** — Log entries are queued and rendered via `Application.MainLoop.Invoke` without blocking the FP network or simulation engine.
- **Unified Command System** — Both Console and `fora-admin` share the same `IAdminCommand` implementations from `Fora.Rti.Admin.Abstractions`.
- **`IRtiAdminClient` abstraction** — Both in-process (`InProcessAdminClient`) and remote (`HttpAdminClient`) modes implement the same interface.

---

## Disclaimer

Fora is provided as a research and educational environment. Review the project license and release notes before using it in production-like workflows.
