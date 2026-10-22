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

**`Wolfgang.Etl.TestKit`** — three test doubles plus deterministic test helpers:

- `TestExtractor<T>` — yields items from a list
- `TestLoader<T>` — captures loaded items into a list
- `TestTransformer<T>` — passes items through unchanged
- `ManualProgressTimer` — control when progress callbacks fire (deterministic alternative to `SystemProgressTimer`)
- `SynchronousProgress<T>` — `IProgress<T>` implementation whose callback runs on the calling thread, so tests do not have to deal with `SynchronizationContext`

**`Wolfgang.Etl.TestKit.Xunit`** — three contract test base classes (`ExtractorBaseContractTests<TSut, TItem, TProgress>`, `TransformerBaseContractTests<TSut, TItem, TProgress>`, `LoaderBaseContractTests<TSut, TItem, TProgress>`). Inherit one, implement three factory methods, and the base class provides 20+ tests covering the full base-class contract: all four async overloads, cancellation, skip/max enforcement, progress callbacks, empty-source handling, and counting correctness.

## When to use it

- **Unit-testing your own extractor, transformer, or loader** — inherit from the corresponding `*BaseContractTests` class to verify your implementation against the framework's contract for free.
- **Integration tests of pipeline wiring** — drop a `TestExtractor<T>` or `TestLoader<T>` into the pipeline as a stand-in for a real component to verify items flow correctly through the rest of the pipeline.
- **Testing progress reporting** — `ManualProgressTimer` lets you fire progress callbacks deterministically, eliminating timer-related flakiness.

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

    protected override JsonLineExtractor<PersonRecord> CreateSut(int itemCount) =>
        new(CreateJsonlStream(itemCount), NullLogger<JsonLineExtractor<PersonRecord>>.Instance);

    protected override IReadOnlyList<PersonRecord> CreateExpectedItems() => ExpectedItems;

    protected override JsonLineExtractor<PersonRecord> CreateSutWithTimer(IProgressTimer timer) =>
        new(CreateJsonlStream(ExpectedItems.Count), new(), NullLogger<JsonLineExtractor<PersonRecord>>.Instance, timer);
}
```

That is the entire test class. Inheriting `ExtractorBaseContractTests` provides 20+ tests covering the full extractor contract.

## More detail

- **Full reference (this site)** — [TestKit](../reference/testkit.md) covers the complete API: contract test base classes, test helpers, the timer-injection pattern, and an end-to-end worked example.
- **API reference (source)** — see the [ETL-Abstractions README](https://github.com/Chris-Wolfgang/ETL-Abstractions#readme) and XML doc comments on the public types.
- **Examples** — see the `tests/` folder in the source repo for canonical usage of each test double and contract test base class.
- **Release notes** — [Releases](https://github.com/Chris-Wolfgang/ETL-Abstractions/releases) (TestKit ships in the same release as Abstractions).
