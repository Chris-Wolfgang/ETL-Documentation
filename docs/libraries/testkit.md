# Wolfgang.Etl.TestKit

Test doubles and contract test base classes for verifying components built on `Wolfgang.Etl.Abstractions`.

| | |
|---|---|
| **NuGet** | `Wolfgang.Etl.TestKit` and `Wolfgang.Etl.TestKit.Xunit` |
| **Version** | ![NuGet](https://img.shields.io/nuget/v/Wolfgang.Etl.TestKit) ![NuGet](https://img.shields.io/nuget/v/Wolfgang.Etl.TestKit.Xunit) |
| **Dependencies** | Wolfgang.Etl.Abstractions, System.Linq.Async; `Wolfgang.Etl.TestKit.Xunit` additionally references `Wolfgang.Etl.TestKit` and xUnit v2 (`xunit.core` / `xunit.assert` 2.9.3) |
| **Target Frameworks** | net462, net481, netstandard2.0, net5.0, net6.0, net7.0, net8.0, net10.0 |
| **Source** | [ETL-Abstractions](https://github.com/Chris-Wolfgang/ETL-Abstractions) (`src/Wolfgang.Etl.TestKit`, `src/Wolfgang.Etl.TestKit.Xunit`); the older ETL-Test-Kit repository is archived |

## What it does

Two packages that make it easy to test extractors, transformers, and loaders built on `Wolfgang.Etl.Abstractions`:

**`Wolfgang.Etl.TestKit`** — test doubles plus deterministic timing helpers:

- `TestExtractor<T>` — yields items from a list, an enumerator or a factory
- `TestLoader<T>` — consumes items and optionally collects them for assertions
- `TestTransformer<T>` — passes items through unchanged
- `FaultyExtractor<T>` / `FaultyTransformer<T>` / `FaultyLoader<T>` — inject deterministic faults for error-path tests
- `DelayingExtractor<T>`, `RetryingExtractor<T>`, `SnapshotTestLoader<T>`, `RecordingMiddleware<T>` — latency, retry, snapshot and middleware testing
- `ManualProgressTimerCore` (attached with `WithManualProgressTimer`) — fire progress callbacks on demand instead of on the built-in timer's interval
- `ManualTimeSource` (attached with `WithTimeSource`) — freeze and advance the clock behind `Report.Elapsed` and the rate / ETA properties

**`Wolfgang.Etl.TestKit.Xunit`** — contract test base classes (`ExtractorBaseContractTests<TSut, TItem, TProgress>`, `TransformerBaseContractTests<TSut, TItem, TProgress>`, `LoaderBaseContractTests<TSut, TItem, TProgress>`, plus cancellation, disposal, error-handling, retry, idempotency, allocation-budget, dry-run and pipeline contracts) and xUnit helpers: `ManualProgressTimer`, `SynchronousProgress<T>`, `ProgressCapture<T>` and `ProgressAssert`. Inherit a base-class contract, implement two factory methods, and you get more than 40 tests covering the full base-class contract: all four async overloads, cancellation, skip/max enforcement, progress callbacks, empty-source handling, and counting correctness.

## When to use it

- **Unit-testing your own extractor, transformer, or loader** — inherit from the corresponding `*BaseContractTests` class to verify your implementation against the framework's contract for free.
- **Integration tests of pipeline wiring** — drop a `TestExtractor<T>` or `TestLoader<T>` into the pipeline as a stand-in for a real component to verify items flow correctly through the rest of the pipeline.
- **Testing progress reporting** — `ManualProgressTimerCore` lets you fire progress callbacks deterministically, eliminating timer-related flakiness, with no timer plumbing in the component under test.

## When to use something else

- For non-Wolfgang.Etl components, use a general-purpose mocking library (Moq, NSubstitute, etc.). The `*BaseContractTests` classes are specifically tied to the `Abstractions` base classes.

## Quick example

```csharp
public class JsonLineExtractorTests
    : ExtractorBaseContractTests<JsonLineExtractor<PersonRecord>, PersonRecord, JsonReport>
{
    private static readonly IReadOnlyList<PersonRecord> ExpectedItems = new List<PersonRecord>
    {
        new() { FirstName = "Alice", LastName = "Smith", Age = 30 },
        // ...at least 5 items
    };

    protected override JsonLineExtractor<PersonRecord> CreateSut
    (
        int itemCount,
        int maximumItemCount,
        int skipItemCount,
        int reportingInterval
    ) =>
        new
        (
            CreateJsonlStream(itemCount),
            new JsonLineExtractorOptions
            {
                MaximumItemCount = maximumItemCount,
                SkipItemCount = skipItemCount,
                ReportingInterval = reportingInterval,
            }
        );

    protected override IReadOnlyList<PersonRecord> CreateExpectedItems() => ExpectedItems;
}
```

That is the entire test class (plus the `CreateJsonlStream` helper shown in the [full example](../reference/testkit.md#complete-example-testing-a-json-extractor)). Inheriting `ExtractorBaseContractTests` provides 43 tests covering the full extractor contract.

## More detail

- **Full reference (this site)** — [TestKit](../reference/testkit.md) covers the complete API: contract test base classes, test doubles and helpers, progress timing in tests, and an end-to-end worked example.
- **API reference (source)** — see the [ETL-Abstractions README](https://github.com/Chris-Wolfgang/ETL-Abstractions#readme) and XML doc comments on the public types.
- **Examples** — see the `tests/` folder in the source repo for canonical usage of each test double and contract test base class.
- **Release notes** — [Releases](https://github.com/Chris-Wolfgang/ETL-Abstractions/releases) (TestKit ships in the same release as Abstractions).
