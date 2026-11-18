# Analyze an exchange

[Programming guide](index.md) · [Read a single stream](analysis.md)

Use an exchange when the question is “where did this send spend time?” or “which subscriber missed it?” The join key is the correlation `EventId`. A receiver count by itself can hide a missing callback behind a duplicate at another subscriber.

## Preserve the source when reading records

Start with the checked Disk files for **one run**. Read each beside its own manifest and keep the original monotonic timestamps. Here `runFiles` is the collection of publisher, RTI and subscriber file paths you selected.

<pre><code class="language-csharp">var observations = new List&lt;SourcedTelemetryRecord&gt;();
foreach (string fortFile in runFiles)
{
    var manifest = TelemetryManifestReader.Read(Path.Combine(Path.GetDirectoryName(fortFile)!, "manifest.json"));
    using var reader = FortFileReader.Open(fortFile);
    observations.AddRange(reader.ReadAll().Select(r =&gt;
        new SourcedTelemetryRecord(Path.GetFullPath(fortFile), reader.FederateId, manifest, r)));
}
</code></pre>

Select the window on the sender. Other event types can use the same extension fields for different purposes. Here `publisherId` identifies the sender. In a larger workload, also select the class, property set or iteration of interest.

<pre><code class="language-csharp">var slowest = observations.Where(e =&gt; e.FederateId == publisherId)
    .Select(e =&gt; e.Record).FilterType(EventTypeCode.ENC_OC).FilterSteadyState().FilterFailed()
    .OrderByDescending(r =&gt; r.DurationNs).Take(1).ToArray();
</code></pre>

An empty result means there was no matching successful send. For a nonempty result, retrieve the **whole** exchange from the original observations, including any failed callbacks.

<pre><code class="language-csharp">var eventId = slowest[0].EventId;
var chain = observations.Where(e =&gt; e.Record.EventId == eventId).ToArray();
</code></pre>

## Verify expected recipients and stages

Supply the intended receiver IDs from the workload or subscription plan. This example expects an object update from the selected publisher to two receivers. Use `ExpectedExchange.Interaction` for an interaction, and set `servicePath` to reflect the configured recording.

<pre><code class="language-csharp">var expected = ExpectedExchange.ObjectUpdate(["display", "recorder"], servicePath: true, sender: publisherId);
var coverage = chain.CheckCoverage(expected);
Console.WriteLine($"Sender and stages complete: {coverage.SenderSideComplete}");
Console.WriteLine($"Receivers complete: {coverage.ReceiversComplete}");
Console.WriteLine($"Missing: {string.Join(", ", coverage.MissingReceivers)}");
</code></pre>

Read `DuplicateReceivers`, `UnexpectedReceivers`, `FailedReceivers`, `StageMismatches` and `SenderDiagnostics` for the cause of incomplete coverage. A missing receiver prevents a claim about all deliveries. Sender and RTI measurements can still be valid. The object factory expects subscription matching and distribution; the interaction factory expects routing. Enabling the service path adds one client call and one RTI call to the expected stages.

A missing **entire** exchange has no `EventId` to query. Use the workload overload to detect it. In the snippet below, `planned` is a dictionary keyed by `(string Sender, int Operation)` with an `ExpectedExchange` for each intended send. The application stored `Operation` using `SetScenarioIteration`.

<pre><code class="language-csharp">static bool MeasuredOperation(SourcedTelemetryRecord sender, out (string Sender, int Operation) key)
{
    int context = sender.Record.ExtensionField3;
    key = (sender.FederateId, TelemetryExecutionContextCodec.UnpackLogicalCycleId(context));
    return TelemetryExecutionContextCodec.UnpackScenarioStep(context) == 2;
}
var report = observations.CheckCoverage(planned, MeasuredOperation);
Console.WriteLine($"Complete: {report.Complete}, missing sends: {report.MissingExchanges.Count}");
</code></pre>

`DuplicateExchanges`, `UnexpectedExchanges` and `ChainsWithoutSender` identify different workload errors. An application-level payload or state check establishes that the receiver applied the intended values.

## Attribute elapsed time

Supply the timer tolerance chosen for your run, in ns. The API adds an 8 ns allowance for tick conversion. Read the diagnostics together with the nullable component values.

<pre><code class="language-csharp">var path = chain.DecomposeServicePath(toleranceNs);
Console.WriteLine($"Diagnostics: {string.Join(", ", path.Diagnostics)}");
if (path.RoundTripNs is long roundTrip &amp;&amp; path.RtiHandlingNs is long rti)
    Console.WriteLine($"Round trip {roundTrip} ns, RTI {rti} ns, transport and queue {path.TransportAndQueueNs} ns");
foreach (var receiver in path.Receivers)
    Console.WriteLine($"{receiver.FederateId}: callback {receiver.CallbackNs} ns, " +
        $"queue wait {receiver.WaitUs?.ToString() ?? "unobserved"} us");
</code></pre>

| Value | Meaning |
|---|---|
| `RoundTripNs` | Client request-write start to response-frame read. |
| `RtiHandlingNs` | RTI-local handling before handing the response to the transport. Matching, distribution or routing occurs inside dispatch. |
| `TransportAndQueueNs` | Round trip minus RTI handling. This combines both transport legs and their queueing/scheduling effects. Local duration subtraction avoids a cross-host clock offset. |
| `Sender` | When available, partitions the sender interval into encoder self-time, before-call work, request framing, RTI handling, combined transport/queue, continuation and after-call work. |
| `RtiStages`, `RtiDispatchOutsideStagesNs` | Admitted nested stages and the remaining RTI dispatch interval. Summing nested stages again with total RTI handling would double count them. |
| `CallbackNs`, `WaitUs` | Receiver callback duration in **ns** and frame-read to callback-entry wait in **µs**. A null wait is unavailable; `WaitAtEncodingLimit` identifies saturation. |

In one recorded NETN-CBRN qualification exchange, a sender service lasted **2,836.2 µs**. Its **1,264.4 µs** round trip comprised **566.7 µs** of RTI handling and **697.7 µs** of combined transport and queueing. The largest sender component was actually **1,489.4 µs after the nested client call**. One receiver waited **1,496 µs** before executing a **5.3 µs** callback. That wait belongs to the receiver's timeline. Measure federation completion separately through the last confirmed delivery.

A negative `RawResponseWaitNs` can occur when the response is read before the write continuation resumes. Individual write and wait durations then become unavailable, while their signed sum can remain meaningful. A negative transport/queue result beyond the declared tolerance signals a timing or placement problem. Preserve the signed result for diagnosis. To compare absolute timestamps across hosts, align each source with its own `EpochStopwatchNs` and `EpochNs` using `AlignToUtc` and check the active clock-alignment evidence.

## Diagnose unexpected results

| Observation | Check |
|---|---|
| No client file | Instrumented client build, configuration before connect, Disk sink, resolved output directory and completed shutdown. |
| No measured sends | Inspect event types and phase labels before filtering. A subscriber may have only callbacks. |
| Codec times are absent | Check actual encoder placement, activation and slot coverage. Returning buffered bytes can leave every encoding slot unobserved. |
| No sender partition | Check the sender and RTI manifests, presence of both call records, the sender boundary, and observed encoder slots. |
| Writer checks pass but delivery coverage fails | Inspect expected receiver IDs, subscription or region membership, callback pumping, and shutdown timing. |
| Recording appears to change performance | Pair ON/OFF runs of the same optimized binaries and workload. Compare elapsed time to the same completion endpoint, validate deliveries in both arms, and report variation across run pairs. |

## Disclaimer

See the [Disclaimer](../../disclaimer.md) for usage terms and research-only conditions.

---

Updated November 18, 2026, 21:47:16 (UTC+03:00)
