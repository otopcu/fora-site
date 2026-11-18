# Record a federate

[Programming guide](index.md) · [Read the recordings](analysis.md)

## Enable recording before connecting

Set the configuration on the existing `Fora.Client.ForaClient` before its normal `ConnectAsync` call. The client opens a session and instruments the ambassador you pass. In this source tree the built-in client instrumentation is compiled in the **Telemetry** configuration. The loaded client DLL must include it.

<pre><code class="language-csharp">using Fora.Telemetry;

client.TelemetryConfiguration = new TelemetryConfiguration
{
    ExperimentId = runId, FederateId = federateName,
    LogDirectory = Path.Combine(runDirectory, federateName),
    SinkMode = TelemetrySinkMode.Disk,
    EnableSubPhaseDecomposition = true,
    EnableServicePathEvents = true
};
await client.ConnectAsync(ambassador, CallbackModel.Immediate, connection, ct: ct);
</code></pre>

Use one experiment ID across the run and a fresh directory for each writer. A publisher and subscriber in the same process can make independent recording choices. Leave a client's configuration null before connecting to keep recording off for that client. Preserve the callback model your application uses. In an evoked federate, the frequency at which you service callbacks can affect the observed queue wait.

The Fora RTI reads its own `Telemetry` section. To include RTI handling, enable service-path events there too. For example, the RTI counterpart of a client run named `run-042` could use this configuration.

<pre><code class="language-json">{
  "Telemetry": {
    "ExperimentId": "run-042",
    "FederateId": "rti",
    "LogDirectory": "measurements/run-042/rti",
    "EnableServicePathEvents": true
  }
}
</code></pre>

## Mark the measurement window

After connecting, mark setup and warm-up as step 1. Switch to step 2 once subscriptions, discovery and warm-up deliveries are complete. Set the iteration immediately before each measured operation. The analysis can then find the sends planned for a particular iteration.

<pre><code class="language-csharp">// At the start of setup
client.SetScenarioStep(1);

// After the warm-up barrier
client.SetScenarioStep(2);
client.SetScenarioIteration(operationId);
</code></pre>

If your application holds an `IForaClient`, access the optional capability through `client is IForaTelemetry telemetry` (`Fora.Abstractions`). Set step 3 for teardown after your chosen endpoint. For a delivery measurement, wait for the expected callbacks before leaving step 2. Follow the normal resign/disconnect lifecycle and await client shutdown. Stop the RTI gracefully to finalize its own stream.

## Measure an attribute update or interaction

Pass an encoder delegate to include serialization in the sender observation. Here `instance` and `valueHandle` are handles your publisher already resolved. Both ends of this example use a four-byte big-endian integer. The scope belongs around the serialization work inside the delegate.

<pre><code class="language-csharp">await client.UpdateAttributeValuesEncodedAsync(instance, () =&gt;
{
    var bytes = new byte[sizeof(int)];
    using (EncodingTelemetryScope.EnterSubPhase(SubPhaseId.Atomic))
        BinaryPrimitives.WriteInt32BigEndian(bytes, value);
    return new Dictionary&lt;AttributeHandle, byte[]&gt; { [valueHandle] = bytes };
}, tag, ct: ct);
</code></pre>

Use `Fora.Encoding` for the scope, `Fora.Telemetry` for the phase ID, `Fora.Abstractions.Handles` for handles and `System.Buffers.Binary` for the integer writer. The enclosing `ENC_OC.DurationNs` includes the encoder invocation and the awaited local service. Slot 1 measures the marked scalar write. Allocating the array and constructing the dictionary occur outside that slot.

For an interaction, supply an encoder that returns a dictionary keyed by `ParameterHandle`. For example, an existing message encoder can be passed directly through a lambda.

<pre><code class="language-csharp">await client.SendInteractionEncodedAsync(messageClass,
    () =&gt; EncodeParameters(message), tag, ct: ct);
</code></pre>

The same placement rule applies to directed encoded sends. A dictionary-taking call records the service on already prepared bytes. A delegate that merely returns a buffered dictionary also leaves the original encoding outside its measured interval.

## Measure receiver decode and application work

Inside an existing reflection callback, time decoding and state application separately. The built-in client supplies the ambient callback scope. Here `values` is the received attribute dictionary and `sensor` is the application's local model.

<pre><code class="language-csharp">int reading;
using (AmbientTelemetryScope.EnterAttributeDecode())
    reading = BinaryPrimitives.ReadInt32BigEndian(values[valueHandle].Span);
using (AmbientTelemetryScope.EnterEntityProxyApply())
    sensor.Value = reading;
</code></pre>

| Record | Slot 1 | Slot 2 | Slot 3 | Slot 4 |
|---|---|---|---|---|
| Sender codec convention | Atomic | Fixed record | Array | Variant |
| Receiver convention | Dispatch resolution | Payload decode | State application, where applicable | Callback body |

Slots report self-time for the regions actually entered. Nested child time is excluded from the parent. The enclosing `DEC_OC` or `DEC_IC` includes application work awaited by the callback. Work scheduled to run after the callback returns has a different endpoint.

## Choose settings for the question

| Setting | When to use it |
|---|---|
| `EnableSubPhaseDecomposition` (default true) | Enable for codec-region times. Disable to keep enclosing service/callback observations without those regions. A custom codec can check `EncodingTelemetryScope.IsMeasuringSubPhases` once at entry. |
| `EnableServicePathEvents` (default false) | Enable at sender and RTI for the request/response path, and at receivers for per-callback queue wait. |
| `SinkMode` | Use `Disk` for offline analysis. `Memory` drains and serializes in memory. `Null` exercises the instrumentation path. Neither diagnostic sink produces a `.fort` file. |
| `SamplingPolicy` | Use `None` for an exact recipient or event count. `FixedRate` and `Adaptive` can omit events. Steady-state sampling is opt-in through `SampleSteadyState`. |
| `EnableGcPauseFiltering` | Enables the collection-overlap flag. The reader chooses whether to retain flagged observations. |
| `EnableMemorySampling`, `EnableRtiStateSampling` | Set the corresponding sample interval for the resource timescale of interest. |
| `FomChecksum`, scenario seed, `EndpointDictionaryVersion` | Record the model digest under a documented convention, reproducible workload metadata and the slot dictionary used in your study. |

For a custom host that owns its own observations, create a separate session with `await using var session = TelemetrySession.Start(config, "host")` and use `session.Emitter` for its lifetime. The built-in client owns and finalizes its own session.

## Disclaimer

See the [Disclaimer](../../disclaimer.md) for usage terms and research-only conditions.

---

Updated November 18, 2026, 21:47:16 (UTC+03:00)
