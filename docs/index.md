---
description: Fora documentation home page.
---
# Fora

Fora is a .NET environment for building, running, and inspecting HLA 4 / IEEE 1516-2025 federations. It includes a federate SDK, an RTI server, and command-line tooling for local development, automation, and research workflows.

This site starts with the public manuals. Internal architecture notes are intentionally kept out of the first public documentation pass.

## Start Here

| Manual | Use it for |
| :--- | :--- |
| [Fora.Client Programmer's Manual](manuals/fora-client-programmers-manual.md) | Building .NET federates with the SDK: connect, join, callbacks, publish/subscribe, updates, interactions, DDM, time, and shutdown. |
| [Fora.Rti.Admin.Client Programmer's Manual](manuals/admin-client-programmers-manual.md) | Building custom RTI management tools, dashboards, monitors, and automation with the admin SDK. |
| [Fora.Rti.Admin User Manual](manuals/admin-tool-user-manual.md) | Running one-off admin commands, automation, JSON output, health checks, logs, snapshots, and remote server management. |
| [Fora.Rti.Console User Manual](manuals/console-user-manual.md) | Using the interactive console for local RTI development or attaching to a running RTI server. |

## Products

### Fora.Client

The canonical .NET SDK for federates. It connects to an RTI through the IEEE 1516-2025 Federate Protocol and exposes asynchronous HLA service APIs plus ambassador callback dispatch.

### Fora.Rti.Server

A container-friendly RTI server for local and hosted federation execution. It exposes Federate Protocol listeners for federates and an HTTP admin surface for tools.

### Fora.Rti.Admin

A headless administration CLI for scripts, CI checks, remote diagnostics, and server automation.

### Fora.Rti.Admin.Client

The programmatic admin SDK behind custom dashboards, monitors, and automation clients.

### Fora.Rti.Console

An interactive terminal UI for local development and live RTI inspection.

## Quick SDK Sketch

<pre><code class="language-csharp">
await using var client = new ForaClient();
var ambassador = new MyFederateAmbassador();

await client.ConnectAsync(new RtiConfiguration
{
    RtiAddress = "127.0.0.1:15164",
    Transport = RtiTransportKind.Tcp
}, ambassador);

await client.EnableAsynchronousDeliveryAsync();
await client.JoinFederationExecutionAsync("FederateA", "Example", "DemoFederation");
</code></pre>

## Documentation Scope

Published now:

- User manuals
- SDK programmer guide
- Tool usage guides

Not published in this first pass:

- Internal architecture notes
- Development notes
- Experimental research drafts
