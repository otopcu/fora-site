# Fora.Telemetry Programming Guide

Use the public C# API to measure an existing HLA 4 federate, inspect its recordings and follow a send through the Fora RTI to its receivers. The snippets assume your application already connects, joins, publishes or subscribes, and implements its callbacks. Use your existing handles, encoder, ambassador and cancellation token in place of the names in the examples.

| Question | API | Guide |
|---|---|---|
| How do I record my federate or time part of a codec? | `ForaClient.TelemetryConfiguration`, `IForaTelemetry`, `EncodingTelemetryScope` | [Record a federate](recording.md) |
| How long do updates of a given class take? | `FortFileReader`, `TelemetryManifestReader`, `GetSummary`, `GetSteadyStateClassSummaries` | [Read and summarize measurements](analysis.md) |
| Where did a slow send spend time? Did every intended subscriber receive it? | `SourcedTelemetryRecord`, `CheckCoverage`, `DecomposeServicePath` | [Analyze an exchange](exchanges.md) |

For sender costs, a single publisher stream is sufficient. For callback costs, record the subscriber. For request/response attribution, record the sender and the RTI with service-path events enabled. For delivery coverage, keep the expected recipient identities from your workload or subscriptions.

These examples use telemetry semantics **3.1**, measurement contract **1.1**, manifest schema **1.1** and `.fort` format **2**. Use the source manifest to interpret archived files. Binary format alone does not identify the meaning of an observation. [Fora.Client Programmer's Manual](../fora-client-programmers-manual.md)

## Disclaimer

See the [Disclaimer](../../disclaimer.md) for usage terms and research-only conditions.

---

Updated November 18, 2026, 21:47:16 (UTC+03:00)
