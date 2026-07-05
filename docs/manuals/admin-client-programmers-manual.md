---
description: Programmer's manual for building .NET RTI administration tools with the Fora.Rti.Admin.Client SDK.
---
# Fora.Rti.Admin.Client Programmer's Manual

`Fora.Rti.Admin.Client` is the .NET SDK for building custom management tools, dashboards, monitors, and automation around a running `Fora.Rti.Server`. It exposes the same administrative contract used by the built-in `fora-admin` CLI and remote `Fora.Rti.Console` mode.

Use this guide when you want to write code against the RTI admin HTTP surface. For command-line operation, see the `Fora.Rti.Admin` user manual.

## Table of Contents
- [Install](#install)
- [Minimal Client](#minimal-client)
- [HTTP Client Setup](#http-client-setup)
- [Authentication](#authentication)
- [Status, Health, and Version](#status-health-and-version)
- [Federations, Federates, and Sessions](#federations-federates-and-sessions)
- [Logs and Streaming](#logs-and-streaming)
- [Snapshots](#snapshots)
- [Shutdown](#shutdown)
- [Error Handling](#error-handling)
- [Using Shared Admin Commands](#using-shared-admin-commands)
- [Recommended Tool Structure](#recommended-tool-structure)

---

## Install

<pre><code class="language-powershell">
dotnet add package Fora.Rti.Admin.Client
</code></pre>

Common namespaces:

<pre><code class="language-csharp">
using System.Net.Http.Headers;
using Fora.Rti.Admin;
using Fora.Rti.Admin.Commands;
</code></pre>

`Fora.Rti.Admin.Client` is async-first. Keep one `HttpClient` for the lifetime of your tool or dashboard client.

---

## Minimal Client

<pre><code class="language-csharp">
using Fora.Rti.Admin;

using var http = new HttpClient
{
    BaseAddress = new Uri("http://localhost:8080")
};

IRtiAdminClient client = new HttpAdminClient(http);

VersionInfo version = await client.GetVersionAsync();
AdminStatus status = await client.GetStatusAsync();
IReadOnlyList&lt;string&gt; federations = await client.GetFederationsAsync();

Console.WriteLine($"{version.Component} {version.PackageVersion}");
Console.WriteLine($"Healthy: {status.Healthy}, sessions: {status.ActiveSessions}");
Console.WriteLine($"Federations: {string.Join(", ", federations)}");
</code></pre>

---

## HTTP Client Setup

`HttpAdminClient` uses the `BaseAddress` on the supplied `HttpClient` and calls:

| API | HTTP endpoint |
| :--- | :--- |
| `GetVersionAsync` | `GET /version` |
| `IsHealthyAsync` | `GET /health` |
| `GetStatusAsync` | `GET /admin/status` |
| `GetFederationsAsync` | `GET /admin/federations` |
| `GetFederatesAsync` | `GET /admin/federates` |
| `GetSessionsAsync` | `GET /admin/sessions` |
| `GetLogsAsync` | `GET /admin/logs` |
| `StreamLogsAsync` | `GET /admin/logs/stream` |
| `SaveSnapshotAsync` | `POST /admin/snapshot/{label}` |
| `RestoreSnapshotAsync` | `POST /admin/snapshot/{label}/restore` |
| `ShutdownAsync` | `POST /admin/shutdown` |

Basic setup:

<pre><code class="language-csharp">
using var http = new HttpClient
{
    BaseAddress = new Uri("https://rti-host:8443"),
    Timeout = TimeSpan.FromSeconds(15)
};

IRtiAdminClient client = new HttpAdminClient(http);
</code></pre>

For ASP.NET Core dashboards, register a typed client:

<pre><code class="language-csharp">
builder.Services.AddHttpClient&lt;IRtiAdminClient, HttpAdminClient&gt;(http =&gt;
{
    http.BaseAddress = new Uri(builder.Configuration["Fora:RtiAdminUrl"]!);
    http.Timeout = TimeSpan.FromSeconds(15);
});
</code></pre>

---

## Authentication

When `Fora.Rti.Server` has `ForaRtiServer:Admin:ApiKey` configured, `/admin/*` endpoints require a bearer token. Attach the token to the `HttpClient`.

<pre><code class="language-csharp">
using var http = new HttpClient
{
    BaseAddress = new Uri("https://rti-host:8443")
};

http.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", apiKey);

IRtiAdminClient client = new HttpAdminClient(http);
</code></pre>

Prefer `https://` when sending an admin token. A bearer token over plain HTTP is visible to anyone who can observe the network path.

`/version` and `/health` may remain public depending on server configuration, but `/admin/*` is protected when the API key is enabled.

---

## Status, Health, and Version

Use `IsHealthyAsync` for quick checks:

<pre><code class="language-csharp">
bool healthy = await client.IsHealthyAsync(ct);
</code></pre>

`IsHealthyAsync` returns `false` if the HTTP request fails. It is intentionally suitable for probes and monitors.

Use `GetStatusAsync` when you need server state:

<pre><code class="language-csharp">
AdminStatus status = await client.GetStatusAsync(ct);

Console.WriteLine($"Healthy: {status.Healthy}");
Console.WriteLine($"Active sessions: {status.ActiveSessions}");
Console.WriteLine($"Auth enabled: {status.AuthorizationEnabled}");
Console.WriteLine($"Auth service: {status.AuthorizationServiceName}");
</code></pre>

Use `GetVersionAsync` for display and compatibility checks:

<pre><code class="language-csharp">
VersionInfo version = await client.GetVersionAsync(ct);

Console.WriteLine(version.Component);
Console.WriteLine(version.PackageVersion);
Console.WriteLine(version.Standard);
Console.WriteLine(version.Hla);
Console.WriteLine(version.CapabilityLevel);
</code></pre>

---

## Federations, Federates, and Sessions

List active federation executions:

<pre><code class="language-csharp">
IReadOnlyList&lt;string&gt; federations = await client.GetFederationsAsync(ct);
</code></pre>

List joined federates:

<pre><code class="language-csharp">
IReadOnlyList&lt;FederateInfo&gt; federates = await client.GetFederatesAsync(ct);

foreach (var federate in federates)
{
    Console.WriteLine($"{federate.Federation}: {federate.Name} ({federate.Type})");
}
</code></pre>

List Federate Protocol sessions:

<pre><code class="language-csharp">
IReadOnlyList&lt;SessionInfo&gt; sessions = await client.GetSessionsAsync(ct);

foreach (var session in sessions)
{
    Console.WriteLine(
        $"{session.SessionId} online={session.IsOnline} lastSeen={session.LastSeenUtc:O}");
}
</code></pre>

`DisconnectedAtUtc` is populated when the server has observed a session disconnect.

---

## Logs and Streaming

Fetch recent logs:

<pre><code class="language-csharp">
IReadOnlyList&lt;LogEntry&gt; logs = await client.GetLogsAsync(ct: ct);
</code></pre>

Fetch logs after a known timestamp:

<pre><code class="language-csharp">
DateTimeOffset since = DateTimeOffset.UtcNow.AddMinutes(-5);
IReadOnlyList&lt;LogEntry&gt; recent = await client.GetLogsAsync(since, ct);
</code></pre>

Stream live logs with Server-Sent Events:

<pre><code class="language-csharp">
await foreach (LogEntry entry in client.StreamLogsAsync(ct))
{
    Console.WriteLine($"[{entry.TimestampUtc:O}] {entry.Level}: {entry.Message}");
}
</code></pre>

`StreamLogsAsync` completes when the server closes the stream or the cancellation token is canceled. If your dashboard needs continuous tailing, wrap it in a reconnect loop with backoff.

<pre><code class="language-csharp">
while (!ct.IsCancellationRequested)
{
    try
    {
        await foreach (var entry in client.StreamLogsAsync(ct))
            Render(entry);
    }
    catch (OperationCanceledException) when (ct.IsCancellationRequested)
    {
        break;
    }
    catch (Exception ex)
    {
        Log.Warning(ex, "Log stream dropped; reconnecting shortly.");
        await Task.Delay(TimeSpan.FromSeconds(2), ct);
    }
}
</code></pre>

---

## Snapshots

Save a snapshot:

<pre><code class="language-csharp">
await client.SaveSnapshotAsync("baseline", ct);
</code></pre>

Restore a snapshot:

<pre><code class="language-csharp">
bool restored = await client.RestoreSnapshotAsync("baseline", ct);

if (!restored)
{
    Console.WriteLine("Snapshot not found.");
}
</code></pre>

`RestoreSnapshotAsync` returns `false` only for a genuine missing snapshot (`404`). Other server failures throw.

Snapshot labels are URL-escaped by `HttpAdminClient`, so labels may contain characters that are valid for the server's snapshot store.

---

## Shutdown

<pre><code class="language-csharp">
await client.ShutdownAsync(ct);
</code></pre>

This sends a remote shutdown request to the server. Use it only from trusted operator workflows. If the server requires an admin API key, the request must include the bearer token.

---

## Error Handling

`HttpAdminClient` surfaces failures instead of fabricating success values.

| Case | Behavior |
| :--- | :--- |
| `401` / `403` from an admin endpoint | Throws `InvalidOperationException` with an authentication-specific message. |
| Non-success response from `SaveSnapshotAsync` or `ShutdownAsync` | Throws through `EnsureSuccessStatusCode`. |
| `RestoreSnapshotAsync` receives `404` | Returns `false`. |
| `RestoreSnapshotAsync` receives another non-success status | Throws. |
| A successful JSON endpoint returns an empty body | Throws `InvalidOperationException`. |
| `IsHealthyAsync` request fails | Returns `false`. |

Typical pattern:

<pre><code class="language-csharp">
try
{
    await client.SaveSnapshotAsync("before-test", ct);
}
catch (InvalidOperationException ex) when (ex.Message.Contains("authentication", StringComparison.OrdinalIgnoreCase))
{
    Console.Error.WriteLine("Admin token is missing or invalid.");
}
catch (HttpRequestException ex)
{
    Console.Error.WriteLine($"Admin request failed: {ex.Message}");
}
</code></pre>

For dashboards, distinguish probe-style calls (`IsHealthyAsync`) from contract calls (`GetStatusAsync`, `GetSessionsAsync`, `SaveSnapshotAsync`) so operators can see real server errors.

---

## Using Shared Admin Commands

The shared abstractions package also contains a small command framework used by the built-in admin tooling. Advanced tools can reuse it when they want CLI-like commands over an `IRtiAdminClient`.

<pre><code class="language-csharp">
var registry = new CommandRegistry();
IAdminCommand? command = registry.GetCommand("federations");

if (command is not null)
{
    var context = new AdminCommandContext(
        Client: client,
        Arguments: [],
        Output: output,
        PreferJson: true,
        CancellationToken: ct);

    await command.ExecuteAsync(context);
}
</code></pre>

For most dashboards and service integrations, call `IRtiAdminClient` directly. Use the command layer when you are building a command runner, shell, or automation surface that should behave like `fora-admin`.

---

## Recommended Tool Structure

| Class | Responsibility |
| :--- | :--- |
| `AdminClientFactory` | Builds `HttpClient`, base URL, timeout, and bearer token. |
| `RtiStatusService` | Polls `GetStatusAsync`, `GetFederationsAsync`, and `GetSessionsAsync`. |
| `RtiLogTailService` | Owns `StreamLogsAsync` and reconnect/backoff behavior. |
| `SnapshotService` | Wraps save/restore operations with operator confirmation and audit logging. |
| `DashboardViewModel` | Presents DTOs and derived health indicators to UI code. |

Suggested polling cadence:

| Data | Typical Cadence |
| :--- | :--- |
| Health | 1-5 seconds |
| Status/session count | 1-5 seconds |
| Federations/federates/sessions | 2-10 seconds |
| Logs | Use `StreamLogsAsync`; fall back to `GetLogsAsync(since)` after reconnect |

Keep admin tooling separate from federate SDK code. `Fora.Client` is for HLA federates; `Fora.Rti.Admin.Client` is for observing and controlling the RTI server.
