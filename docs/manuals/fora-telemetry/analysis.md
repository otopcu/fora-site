# Read and summarize measurements

[Programming guide](index.md) · [Analyze an exchange](exchanges.md)

The snippets use `using Fora.Telemetry;` and LINQ. They operate on one finalized Disk recording. A publisher stream answers questions about sends; a subscriber stream answers questions about callbacks.

## Open a recording

Read the manifest next to the file and check the file itself. The following checks catch an interrupted write before using durations in a report.

<pre><code class="language-csharp">var manifest = TelemetryManifestReader.Read(Path.Combine(Path.GetDirectoryName(fortPath)!, "manifest.json"));
using var reader = FortFileReader.Open(fortPath);
if (!reader.HasCompleteEof || reader.RecoveredPartialTailBytes != 0 || manifest.Telemetry.FlusherFaulted)
    throw new InvalidDataException($"Incomplete telemetry file: {fortPath}");
Console.WriteLine($"{reader.FederateId}: {reader.RecordCount} records, semantics {manifest.Fora.TelemetryVersion}");
var records = reader.ReadAll().ToArray();
</code></pre>

Check that `manifest.Telemetry.LogFiles` names the file. For a lossless single-file run, reconcile `RecordCount` with accepted and persisted counts, verify zero drops, zero pending records, and inspect `Sampling.Lossless` and `Sampling.SampledOutCount`. The raw manifest also includes finalization, sub-phase integrity and backpressure counters. A clean EOF establishes file closure; the workload plan establishes whether every intended operation is present. For large files, enumerate `ReadAll()` while the reader is open and filter before materializing.

## Get a summary for one class

**Question:** How expensive were successful measured updates for one object class? Select the sender's event type, class, measurement window and API boundary. In this example, `classHandle` is the integer handle from the recorded run. Require known, active sub-phase coverage so that records with different instrumentation settings cannot share the summary.

<pre><code class="language-csharp">var required = TelemetryRecordFlags.SubPhaseCoverageKnown | TelemetryRecordFlags.SubPhaseActive;
var updates = records.FilterType(EventTypeCode.ENC_OC).FilterClass(classHandle)
    .FilterSteadyState().FilterFailed()
    .Where(r =&gt; r.GetSenderMeasurementBoundary() == "encoder_and_awaited_local_service")
    .Where(r =&gt; ((TelemetryRecordFlags)r.Flags &amp; required) == required).ToArray();

TelemetrySummary summary = updates.GetSummary();
Console.WriteLine($"{summary.EventCount} updates, {summary.AverageDurationUs:F2} us/update, " +
    $"{summary.AveragePayloadBytes:F1} bytes/update");
</code></pre>

`FilterFailed()` excludes failed operations. Count failures before applying it if failure rate matters. GC-flagged successes remain in this query; add `.FilterGcContaminated()` when the analysis explicitly excludes them. To summarize interaction sends, use `ENC_IC` and the interaction class handle. For one class sending different member subsets, apply `.Where(r => r.ExtensionField1 == propertySetId)` before aggregation, or group by this field to report each subset separately.

| Summary field | Unit and interpretation |
|---|---|
| `EventCount` | Number of selected records. Check this first; an empty input produces a zero summary. |
| `TotalDurationNs`, `AverageDurationUs` | Sum of enclosing intervals in ns and arithmetic mean in µs. Intervals can overlap across sends, so their sum differs from wall-clock completion time. |
| `TotalPayloadBytes`, `AveragePayloadBytes` | Attribute or parameter value bytes on the selected sender records. Protocol framing has a separate cost. |
| `MeasurementBoundary` | `encoder_and_awaited_local_service` includes encoder invocation. `local_service_excludes_prior_encoding` starts with prepared values. |
| `AverageSubPhase1Us` … `AverageSubPhase4Us` | Mean self-time in µs over the observations that entered each slot. |
| `SubPhaseObservationCounts` | Four denominators, indexed 0–3, when coverage is known and input is nonempty. Legacy coverage can have an empty list. |

For example, one recorded NETN-CBRN qualification update for class `Human` carried **1,576 bytes**. The enclosing service lasted **540.20 µs** and the atomic slot accounted for **8.10 µs**. That single observation illustrates the two measurement boundaries. Estimating a typical class cost requires repeated observations.

## Distinguish missing slots from measured zero

<pre><code class="language-csharp">if (summary.SubPhaseObservationCounts.Count == 4 &amp;&amp; summary.SubPhaseObservationCounts[0] &gt; 0)
    Console.WriteLine($"Atomic {summary.AverageSubPhase1Us:F2} us, " +
        $"observed on {summary.SubPhaseObservationCounts[0]}/{summary.EventCount} updates");
</code></pre>

If two of ten updates entered a slot and took 3 and 5 µs, its reported mean is 4 µs with coverage 2/10. A wholly unobserved slot in a nonempty known-coverage summary yields `NaN`; export the mean as JSON `null` with count zero. On one record, `GetMeasuredSubPhaseNs(1)` returns a nullable duration and preserves a measured zero.

## Compare classes and identify bursts

For a comparison across classes, start from the file again, omit the single-class filter and keep the same boundary and activation conditions. The steady-state class helper applies the phase and success filters. Resolve class handles using the source manifest.

<pre><code class="language-csharp">var byClass = records.FilterType(EventTypeCode.ENC_OC)
    .Where(r =&gt; r.GetSenderMeasurementBoundary() == "encoder_and_awaited_local_service")
    .Where(r =&gt; ((TelemetryRecordFlags)r.Flags &amp; required) == required)
    .GetSteadyStateClassSummaries();
foreach (var (handle, s) in byClass.OrderByDescending(x =&gt; x.Value.AverageDurationUs))
    Console.WriteLine($"{manifest.Fom.ClassHandleMap.GetValueOrDefault(handle.ToString(), handle.ToString())}: " +
        $"{s.EventCount} sends, mean {s.AverageDurationUs:F2} us");
</code></pre>

Use `InteractionHandleMap` for interactions. For partial updates, the manifest's `fom.property_set_map` gives the members selected by each property-set ID under its class and domain. Use `AttributeHandleMap` or `ParameterHandleMap` to resolve member names.

| Question | API applied to the selected population |
|---|---|
| Average and peak send frequency? | `updates.GetAverageRateHz()` uses `(N−1)/elapsed seconds`. `updates.GetPeakRateHz(windowSizeNs: 100_000_000)` reports the busiest 100 ms window in Hz. With fewer than two timestamps the mean rate is zero. |
| Which instance produced the most updates? | `updates.GroupByInstance()`, then count each group. Preserve the source run when interpreting instance handles. |
| Are sequence numbers missing? | `records.GetSequenceGaps()` on **one unfiltered stream** finds interior gaps. Pass expected first and last sequence numbers to check the edges. Interpret gaps alongside sampling and writer health. |
| What was the observed resource load? | Select `MEMORY` records. Under semantics 3.1, `DurationNs` on these records stores resident bytes and `ClassHandle` stores callback queue depth. Inspect maxima over the sampled window. Analyze `RTI_STATE` and class-census samples separately. |

`GetSummary()` can reject mixed known boundaries, event types, activation states and legacy/known coverage. Group or filter on those differences before aggregating. Keep sender service, RTI handling and receiver callback durations as separate quantities.
