---
description: Programmer's manual for building .NET HLA federates with the Fora.Client SDK.
---
# Fora.Client Programmer's Manual

`Fora.Client` is the .NET SDK for writing HLA 4 federates that connect to an RTI through the IEEE 1516-2025 Federate Protocol. This manual is for application programmers who want to build a federate, exchange object updates or interactions, receive RTI callbacks, and shut down cleanly.

For package installation and the short NuGet overview, start with this manual and the Fora package pages. Protocol internals and implementation design notes are intentionally not part of this first public site pass.

## Table of Contents
- [Install](#install)
- [Minimal Federate](#minimal-federate)
- [Connection Configuration](#connection-configuration)
- [Ambassador Callbacks](#ambassador-callbacks)
- [Callback Delivery Models](#callback-delivery-models)
- [Federation Lifecycle](#federation-lifecycle)
- [Handles and FOM Lookup](#handles-and-fom-lookup)
- [Encoding Attribute and Parameter Values](#encoding-attribute-and-parameter-values)
- [Publish, Subscribe, Update, and Interact](#publish-subscribe-update-and-interact)
- [Object Instances](#object-instances)
- [Time Management](#time-management)
- [Data Distribution Management](#data-distribution-management)
- [Ownership Management](#ownership-management)
- [Support Services](#support-services)
- [Logging and Diagnostics](#logging-and-diagnostics)
- [Disconnect and Session Termination](#disconnect-and-session-termination)
- [Error Handling](#error-handling)
- [Recommended Federate Structure](#recommended-federate-structure)

---

## Install

<pre><code class="language-powershell">
dotnet add package Fora.Client --version 20260705.1.0
</code></pre>

Common namespaces:

<pre><code class="language-csharp">
using Fora.Abstractions;
using Fora.Abstractions.Connection;
using Fora.Abstractions.Exceptions;
using Fora.Abstractions.Handles;
using Fora.Client;
using Fora.Encoding;
</code></pre>

`Fora.Client` targets .NET 10 and uses async APIs throughout. Keep one `ForaClient` instance per federate connection.

---

## Minimal Federate

<pre><code class="language-csharp">
using Fora.Abstractions;
using Fora.Abstractions.Connection;
using Fora.Client;

await using var client = new ForaClient();
var ambassador = new MyFederateAmbassador();

await client.ConnectAsync(new RtiConfiguration
{
    RtiAddress = "127.0.0.1:15164",
    Transport = RtiTransportKind.Tcp
}, ambassador);

await client.EnableAsynchronousDeliveryAsync();

await client.CreateFederationExecutionAsync("DemoFederation", "Demo.xml");
var federate = await client.JoinFederationExecutionAsync(
    federateName: "DemoFederate",
    federateType: "DemoType",
    federationName: "DemoFederation");

// Resolve handles, publish/subscribe, register objects, send updates, or send interactions here.

await client.ResignFederationExecutionAsync(ResignAction.DeleteObjects);
await client.DisconnectAsync();
</code></pre>

Your ambassador receives callbacks:

<pre><code class="language-csharp">
public sealed class MyFederateAmbassador : IFederateAmbassador
{
    public Task ConnectionLostAsync(string faultDescription)
    {
        Console.Error.WriteLine($"RTI connection lost: {faultDescription}");
        return Task.CompletedTask;
    }

    public Task DiscoverObjectInstanceAsync(
        ObjectInstanceHandle instance,
        ObjectClassHandle objectClass,
        string instanceName)
    {
        Console.WriteLine($"Discovered {instanceName} as {instance.Value}");
        return Task.CompletedTask;
    }

    public Task ReceiveInteractionAsync(
        InteractionClassHandle interactionClass,
        IReadOnlyDictionary&lt;ParameterHandle, ReadOnlyMemory&lt;byte&gt;&gt; parameterValues,
        ReadOnlyMemory&lt;byte&gt; tag,
        OrderType sentOrder,
        TransportationType transport)
    {
        Console.WriteLine($"Received interaction {interactionClass.Value}");
        return Task.CompletedTask;
    }

    // Implement the remaining IFederateAmbassador members used by your federate.
}
</code></pre>

---

## Connection Configuration

`RtiConfiguration` controls both the FP transport and the standard HLA connect payload.

| Property | Meaning |
| :--- | :--- |
| `RtiAddress` | Host, optional port, or URI. Examples: `localhost`, `localhost:15164`, `tcp://rti.example.com:15164`. |
| `Transport` | Physical transport: `Tcp`, `Tls`, `WebSocket`, or `SecureWebSocket`. |
| `ConfigurationName` | Optional RTI configuration name sent in the HLA connect request. |
| `AdditionalSettings` | Vendor-specific settings string sent in the HLA connect request. |
| `Credentials` | Optional HLA credential record. |

Default ports:

| Transport | Default Port |
| :--- | ---: |
| `Tcp` | `15164` |
| `Tls` | `15165` |
| `WebSocket` | `80` |
| `SecureWebSocket` | `443` |

Plain TCP:

<pre><code class="language-csharp">
await client.ConnectAsync(new RtiConfiguration
{
    RtiAddress = "localhost:15164",
    Transport = RtiTransportKind.Tcp
}, ambassador);
</code></pre>

TLS:

<pre><code class="language-csharp">
await client.ConnectAsync(new RtiConfiguration
{
    RtiAddress = "localhost:15165",
    Transport = RtiTransportKind.Tls
}, ambassador);
</code></pre>

Password credentials:

<pre><code class="language-csharp">
await client.ConnectAsync(new RtiConfiguration
{
    RtiAddress = "localhost:15164",
    Transport = RtiTransportKind.Tcp,
    Credentials = new HlaPlainTextPasswordCredentials("changeit")
}, ambassador);
</code></pre>

`HlaPlainTextPasswordCredentials` carries only the password. Use TLS or a trusted private network when sending password credentials.

---

## Ambassador Callbacks

Implement `IFederateAmbassador` to receive RTI-to-federate callbacks. The interface covers:

| Callback Area | Examples |
| :--- | :--- |
| Connection and federation events | `ConnectionLostAsync`, `ReportFederationExecutionsAsync`, save/restore callbacks |
| Synchronization | `AnnounceSynchronizationPointAsync`, `FederationSynchronizedAsync` |
| Declaration advisories | `StartRegistrationForObjectClassAsync`, `TurnInteractionsOnAsync` |
| Object management | `DiscoverObjectInstanceAsync`, `ReflectAttributeValuesAsync`, `ReceiveInteractionAsync`, `RemoveObjectInstanceAsync` |
| Ownership | `RequestAttributeOwnershipReleaseAsync`, `AttributeOwnershipAcquisitionNotificationAsync` |
| Time management | `TimeRegulationEnabledAsync`, `TimeConstrainedEnabledAsync`, `TimeAdvanceGrantAsync` |

Keep callback methods short. If a callback needs slow application work, queue that work onto your own application pipeline and return promptly. The SDK acknowledges FP callback frames after dispatch, so long callback methods can slow protocol progress.

---

## Callback Delivery Models

The SDK distinguishes two local delivery controls:

| API | Effect |
| :--- | :--- |
| `EnableAsynchronousDeliveryAsync()` | Drains buffered callbacks and dispatches future callbacks to the ambassador as they arrive. This is the usual mode for active applications. |
| `EvokeCallbackAsync()` | In synchronous mode, dispatches at most one buffered callback. |
| `EvokeMultipleCallbacksAsync(min, max)` | In synchronous mode, dispatches buffered callbacks during the requested time window. |
| `DisableCallbacksAsync()` | Locally gates ambassador delivery while the FP session still acknowledges RTI callback frames. |
| `EnableCallbacksAsync()` | Re-enables local ambassador delivery and drains the gated callback buffer. |

Recommended default:

<pre><code class="language-csharp">
await client.ConnectAsync(configuration, ambassador);
await client.EnableAsynchronousDeliveryAsync();
</code></pre>

Synchronous polling style:

<pre><code class="language-csharp">
await client.ConnectAsync(configuration, ambassador);

while (!ct.IsCancellationRequested)
{
    bool delivered = await client.EvokeCallbackAsync(
        approxMinTimeInSeconds: 0.1,
        ct);

    if (!delivered)
        await Task.Delay(10, ct);
}
</code></pre>

Do not call `EvokeCallbackAsync` after enabling asynchronous delivery.

---

## Federation Lifecycle

Typical lifecycle:

1. Connect to the RTI.
2. Create or discover a federation execution.
3. Join the federation.
4. Resolve FOM handles.
5. Publish and subscribe.
6. Register object instances or send interactions.
7. Resign from the federation.
8. Disconnect from the RTI.

Create and join:

<pre><code class="language-csharp">
await client.CreateFederationExecutionAsync(
    federationName: "DemoFederation",
    fomPath: "Demo.xml",
    ct: ct);

FederateHandle federate = await client.JoinFederationExecutionAsync(
    federateName: "PublisherA",
    federateType: "Publisher",
    federationName: "DemoFederation",
    ct: ct);
</code></pre>

List federations and members:

<pre><code class="language-csharp">
var federations = await client.ListFederationExecutionsAsync(ct);
var members = await client.ListFederationExecutionMembersAsync("DemoFederation", ct);
</code></pre>

Resign and cleanup:

<pre><code class="language-csharp">
await client.ResignFederationExecutionAsync(ResignAction.DeleteObjects, ct);
await client.DisconnectAsync(ct);
</code></pre>

Call `DisconnectAsync` only after resigning. RTIs may reject disconnect while the federate is still joined.

---

## Handles and FOM Lookup

Most HLA services use handles instead of names. Resolve handles after joining, then cache them in your application state.

<pre><code class="language-csharp">
ObjectClassHandle vehicleClass =
    await client.GetObjectClassHandleAsync("HLAobjectRoot.Vehicle", ct);

AttributeHandle positionAttribute =
    await client.GetAttributeHandleAsync(vehicleClass, "Position", ct);

InteractionClassHandle fireEvent =
    await client.GetInteractionClassHandleAsync("HLAinteractionRoot.FireEvent", ct);

ParameterHandle munitionParameter =
    await client.GetParameterHandleAsync(fireEvent, "MunitionId", ct);
</code></pre>

Reverse lookup is also available:

<pre><code class="language-csharp">
string className = await client.GetObjectClassNameAsync(vehicleClass, ct);
string interactionName = await client.GetInteractionClassNameAsync(fireEvent, ct);
</code></pre>

Use the support-service lookup calls rather than hard-coding numeric handle values. Numeric values are RTI/FOM-session artifacts.

---

## Encoding Attribute and Parameter Values

HLA attributes and parameters are byte payloads. `Fora.Client` does not infer your FOM datatype. Your federate must encode values according to the FOM.

Simple primitive encoding:

<pre><code class="language-csharp">
byte[] speed = HlaPrimitives.EncodeFloat64Be(12.5);
byte[] name = HlaPrimitives.EncodeUnicodeString("Alpha");
byte[] count = HlaPrimitives.EncodeInt32Be(42);
</code></pre>

Attribute maps use `AttributeHandle` keys:

<pre><code class="language-csharp">
var attributes = new Dictionary&lt;AttributeHandle, byte[]&gt;
{
    [positionAttribute] = EncodePosition(10.0, 20.0, 30.0),
    [nameAttribute] = HlaPrimitives.EncodeUnicodeString("Vehicle-1")
};
</code></pre>

Interaction maps use `ParameterHandle` keys:

<pre><code class="language-csharp">
var parameters = new Dictionary&lt;ParameterHandle, byte[]&gt;
{
    [munitionParameter] = HlaPrimitives.EncodeUnicodeString("M-001")
};
</code></pre>

Generated federates can use the encoded overloads to defer encoding until the SDK telemetry wrapper measures it:

<pre><code class="language-csharp">
await client.UpdateAttributeValuesEncodedAsync(
    instance,
    encoder: () =&gt; VehicleEncoder.Encode(vehicleDto, som),
    tag: Array.Empty&lt;byte&gt;(),
    ct: ct);
</code></pre>

For composed datatypes, use `Fora.Encoding` records and arrays or generated codec code from your FOM tooling. Make sure padding and octet boundaries match IEEE 1516.2-2025.

---

## Publish, Subscribe, Update, and Interact

Publish and subscribe before exchanging data:

<pre><code class="language-csharp">
await client.PublishObjectClassAttributesAsync(
    vehicleClass,
    new HashSet&lt;AttributeHandle&gt; { positionAttribute, nameAttribute },
    ct);

await client.SubscribeObjectClassAttributesAsync(
    vehicleClass,
    new HashSet&lt;AttributeHandle&gt; { positionAttribute, nameAttribute },
    ct);

await client.PublishInteractionClassAsync(fireEvent, ct);
await client.SubscribeInteractionClassAsync(fireEvent, ct);
</code></pre>

Update object attributes:

<pre><code class="language-csharp">
await client.UpdateAttributeValuesAsync(
    instance,
    attributes,
    tag: Array.Empty&lt;byte&gt;(),
    ct: ct);
</code></pre>

Send a receive-order interaction:

<pre><code class="language-csharp">
await client.SendInteractionAsync(
    fireEvent,
    parameters,
    tag: Array.Empty&lt;byte&gt;(),
    ct: ct);
</code></pre>

Send a timestamp-order interaction:

<pre><code class="language-csharp">
MessageRetractionHandle? retraction = await client.SendInteractionAsync(
    fireEvent,
    parameters,
    tag: Array.Empty&lt;byte&gt;(),
    time: 15.0,
    ct: ct);
</code></pre>

The optional return value is a message retraction handle for timestamp-order sends when the RTI supplies one.

---

## Object Instances

Register an object instance after publishing the object class:

<pre><code class="language-csharp">
ObjectInstanceHandle instance =
    await client.RegisterObjectInstanceAsync(vehicleClass, "Vehicle-1", ct);
</code></pre>

Reserve names when your federation needs deterministic object instance names:

<pre><code class="language-csharp">
await client.ReserveObjectInstanceNameAsync("Vehicle-1", ct);
</code></pre>

The result is delivered through `ObjectInstanceNameReservationSucceededAsync` or `ObjectInstanceNameReservationFailedAsync`.

Delete an object instance:

<pre><code class="language-csharp">
await client.DeleteObjectInstanceAsync(
    instance,
    tag: Array.Empty&lt;byte&gt;(),
    ct: ct);
</code></pre>

Subscribers receive `DiscoverObjectInstanceAsync`, `ReflectAttributeValuesAsync`, and `RemoveObjectInstanceAsync` callbacks according to their subscriptions and RTI routing rules.

---

## Time Management

Enable time regulation and/or time constrained mode before using timestamp-order services:

<pre><code class="language-csharp">
await client.EnableTimeRegulationAsync(lookahead: 1.0, ct: ct);
await client.EnableTimeConstrainedAsync(ct);
</code></pre>

The ambassador receives:

<pre><code class="language-csharp">
Task TimeRegulationEnabledAsync(double logicalTime);
Task TimeConstrainedEnabledAsync(double logicalTime);
</code></pre>

Advance time:

<pre><code class="language-csharp">
await client.TimeAdvanceRequestAsync(10.0, ct);
</code></pre>

The grant arrives through:

<pre><code class="language-csharp">
Task TimeAdvanceGrantAsync(double logicalTime);
</code></pre>

Other supported advance services:

| Service | API |
| :--- | :--- |
| Time Advance Request Available | `TimeAdvanceRequestAvailableAsync(time)` |
| Next Message Request | `NextMessageRequestAsync(time)` |
| Next Message Request Available | `NextMessageRequestAvailableAsync(time)` |
| Flush Queue Request | `FlushQueueRequestAsync(time)` |

Query time state:

<pre><code class="language-csharp">
double time = await client.QueryLogicalTimeAsync(ct);
double lookahead = await client.QueryLookaheadAsync(ct);
double galt = await client.QueryGALTAsync(ct);
double lits = await client.QueryLITSAsync(ct);
</code></pre>

---

## Data Distribution Management

Create a region from dimensions, commit ranges, and use region-aware subscription or update services.

<pre><code class="language-csharp">
DimensionHandle area =
    await client.GetDimensionHandleAsync("Area", ct);

RegionHandle region =
    await client.CreateRegionAsync(new HashSet&lt;DimensionHandle&gt; { area }, ct);

await client.CommitRegionModificationsAsync(
    new Dictionary&lt;RegionHandle, IReadOnlyDictionary&lt;DimensionHandle, RangeBounds&gt;&gt;
    {
        [region] = new Dictionary&lt;DimensionHandle, RangeBounds&gt;
        {
            [area] = new RangeBounds(0, 100)
        }
    },
    ct);
</code></pre>

Subscribe attributes with regions:

<pre><code class="language-csharp">
await client.SubscribeObjectClassAttributesWithRegionsAsync(
    vehicleClass,
    new Dictionary&lt;AttributeHandle, IReadOnlySet&lt;RegionHandle&gt;&gt;
    {
        [positionAttribute] = new HashSet&lt;RegionHandle&gt; { region }
    },
    ct);
</code></pre>

Associate a registered object with update regions:

<pre><code class="language-csharp">
await client.AssociateRegionsForUpdatesAsync(
    instance,
    new Dictionary&lt;AttributeHandle, IReadOnlySet&lt;RegionHandle&gt;&gt;
    {
        [positionAttribute] = new HashSet&lt;RegionHandle&gt; { region }
    },
    ct);
</code></pre>

When region conveyance is enabled, matching sent regions can be delivered to the ambassador through the DDM-aware callback overloads.

---

## Ownership Management

Ownership APIs map directly to HLA attribute ownership workflows.

Acquire if available:

<pre><code class="language-csharp">
await client.AttributeOwnershipAcquisitionIfAvailableAsync(
    instance,
    new HashSet&lt;AttributeHandle&gt; { positionAttribute },
    ct);
</code></pre>

Request negotiated acquisition:

<pre><code class="language-csharp">
await client.AttributeOwnershipAcquisitionAsync(
    instance,
    new HashSet&lt;AttributeHandle&gt; { positionAttribute },
    tag: Array.Empty&lt;byte&gt;(),
    ct);
</code></pre>

Release ownership:

<pre><code class="language-csharp">
await client.UnconditionalAttributeOwnershipDivestitureAsync(
    instance,
    new HashSet&lt;AttributeHandle&gt; { positionAttribute },
    ct);
</code></pre>

Relevant callbacks include `RequestAttributeOwnershipReleaseAsync`, `AttributeOwnershipAcquisitionNotificationAsync`, `AttributeOwnershipUnavailableAsync`, and `InformAttributeOwnershipAsync`.

---

## Support Services

Support services are used heavily by generated federates and diagnostics:

| Need | API Examples |
| :--- | :--- |
| Handle/name lookup | `GetObjectClassHandleAsync`, `GetAttributeHandleAsync`, `GetInteractionClassHandleAsync`, `GetParameterHandleAsync` |
| FOM metadata | `GetFddInfoAsync`, `GetObjectClassInformationAsync`, `GetInteractionClassInformationAsync` |
| DDM metadata | `GetDimensionHandleAsync`, `GetDimensionUpperBoundAsync`, `GetAvailableDimensionsForClassAttributeAsync` |
| Transportation/order | `GetTransportationTypeHandleAsync`, `GetOrderTypeAsync`, `ChangeAttributeOrderTypeAsync` |
| Callback control | `EnableCallbacksAsync`, `DisableCallbacksAsync`, `EvokeCallbackAsync` |
| RTI switches | `SetConveyRegionDesignatorSetsAsync`, `SetServiceReportingAsync`, advisory switch APIs |

---

## Logging and Diagnostics

Subscribe to the `Logged` event for SDK diagnostics:

<pre><code class="language-csharp">
client.MinimumLogLevel = ForaLogLevel.Debug;
client.Logged += (_, e) =&gt;
{
    Console.WriteLine($"[{e.Level}] {e.Message}");
};
</code></pre>

The `ForaClient(ILogger<ForaClient>)` constructor also forwards SDK logs into Microsoft.Extensions.Logging.

Useful diagnostic signals:

| Signal | Meaning |
| :--- | :--- |
| `ConnectionLostAsync` | RTI or transport ended the FP session. Stop issuing HLA calls and reconnect intentionally. |
| `HlaNotConnected` | The client is not connected, has been disposed, or the RTI terminated the FP session. |
| `Logged` warnings/errors | Protocol, RTI service, or lifecycle issues surfaced by the SDK. |

---

## Disconnect and Session Termination

For normal shutdown:

<pre><code class="language-csharp">
await client.ResignFederationExecutionAsync(ResignAction.DeleteObjects, ct);
await client.DisconnectAsync(ct);
</code></pre>

`DisconnectAsync` sends the HLA disconnect service and then disposes the FP connection. During disposal, the SDK performs the standard `CTRL_TERMINATE_SESSION` to `CTRL_SESSION_TERMINATED` handshake when the session is still active.

If the RTI initiates session termination, the SDK:

1. Stops local heartbeat, callback, receive, and transport services.
2. Acknowledges `CTRL_TERMINATE_SESSION` with `CTRL_SESSION_TERMINATED` when required.
3. Raises `IFederateAmbassador.ConnectionLostAsync`.
4. Rejects later HLA service calls with `HlaNotConnected` instead of writing to a dead FP session.

Treat `ConnectionLostAsync` as a terminal state for the current `ForaClient` instance. Create a new client and reconnect when your application policy allows it.

---

## Error Handling

RTI-reported service exceptions are mapped to `HlaException` subclasses when possible.

<pre><code class="language-csharp">
try
{
    await client.CreateFederationExecutionAsync("DemoFederation", "Demo.xml", ct);
}
catch (HlaFederationExecutionAlreadyExists)
{
    // Safe to continue if another federate already created it.
}
catch (HlaException ex)
{
    Console.Error.WriteLine($"{ex.ErrorType}: {ex.Message}");
    throw;
}
</code></pre>

Common lifecycle exceptions:

| Exception | Typical Cause |
| :--- | :--- |
| `HlaAlreadyConnected` | `ConnectAsync` was called on an already connected client. |
| `HlaNotConnected` | A service was called before connect, after disconnect, after dispose, or after RTI-side session termination. |
| `HlaFederateAlreadyExecutionMember` | Join was requested when already joined. |
| `HlaFederateNotExecutionMember` | A joined-only service was called before joining. |
| `HlaFederateIsExecutionMember` | A not-joined service, such as destroy/disconnect, was requested while still joined. |
| `HlaUnauthorized` / `HlaInvalidCredentials` | RTI authorization rejected the connect request. |

Always pass cancellation tokens from your application shutdown path. If your application needs per-call timeouts, wrap calls with a `CancellationTokenSource` until your policy is enforced at a higher layer.

---

## Recommended Federate Structure

A maintainable federate usually separates these responsibilities:

| Class | Responsibility |
| :--- | :--- |
| `FederateApp` | Owns `ForaClient`, startup/shutdown, and top-level state. |
| `FederateAmbassador` | Implements `IFederateAmbassador` and queues domain events. |
| `SomHandles` | Stores resolved object, attribute, interaction, parameter, and dimension handles. |
| `FederateCodecs` | Encodes/decodes FOM datatypes to and from byte arrays. |
| `PublicationService` | Registers objects and sends updates/interactions. |
| `SubscriptionService` | Applies received callbacks to application state. |

Suggested startup order:

1. Build `ForaClient` and ambassador.
2. Connect.
3. Enable asynchronous callback delivery unless your federate intentionally polls callbacks.
4. Create or join the federation.
5. Resolve and cache handles.
6. Publish and subscribe.
7. Register objects and run the simulation loop.

Suggested shutdown order:

1. Stop your simulation loop.
2. Delete or resign-owned objects if your policy requires it.
3. Resign from the federation.
4. Disconnect.
5. Dispose the client.

This keeps HLA protocol concerns out of UI, simulation-domain, and encoding code while still letting the ambassador react quickly to RTI callbacks.
