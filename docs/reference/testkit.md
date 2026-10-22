# TestKit

API reference for the `Wolfgang.Etl.TestKit` and `Wolfgang.Etl.TestKit.Xunit` packages. These provide test doubles and contract test base classes for verifying that your extractors, transformers, and loaders conform to the expected behavior defined by the Abstractions. Both packages ship from the [ETL-Abstractions](https://github.com/Chris-Wolfgang/ETL-Abstractions) repository, in the same release as `Wolfgang.Etl.Abstractions`.

## Packages

| Package | Version | Purpose |
|---------|---------|---------|
| `Wolfgang.Etl.TestKit` | 0.26.1 | Test doubles (`TestExtractor<T>`, `TestTransformer<T>`, `TestLoader<T>`, fault / delay / retry / snapshot doubles) and deterministic timing helpers (`ManualProgressTimerCore`, `ManualTimeSource`) |
| `Wolfgang.Etl.TestKit.Xunit` | 0.26.1 | Contract test base classes for xUnit, plus `ManualProgressTimer`, `SynchronousProgress<T>`, `ProgressCapture<T>` and `ProgressAssert` |


## Why Use TestKit?

When you build a custom extractor, transformer, or loader, the base classes in `Wolfgang.Etl.Abstractions` handle orchestration (progress reporting, cancellation token wiring, async enumeration). Your derived class only implements `ExtractWorkerAsync`, `TransformWorkerAsync`, or `LoadWorkerAsync`. But you still need to verify that:

1. Your worker methods correctly call `IncrementCurrentItemCount()` for each item
2. `SkipItemCount` and `MaximumItemCount` are respected
3. Cancellation tokens are honored
4. Progress reporting works
5. Configuration values out of range are rejected at construction

The contract test base classes in `TestKit.Xunit` test **all of this automatically**. You implement two factory methods, and the base class provides more than 40 test cases (43 for extractors, 46 for loaders, 48 for transformers) that verify your implementation against the contract.


## Contract Test Base Classes

### ExtractorBaseContractTests\<TSut, TItem, TProgress\>

Use this base class to test any class that inherits from `ExtractorBase<TSource, TProgress>`.

#### Abstract Methods You Must Implement

| Method | Description |
|--------|-------------|
| `CreateSut(int itemCount, int maximumItemCount, int skipItemCount, int reportingInterval)` | Create an instance of your extractor that yields exactly `itemCount` items, configured with the three values (pass them straight through to your options record without validating them — some tests pass out-of-range values on purpose and expect construction to throw `ArgumentOutOfRangeException`) |
| `CreateExpectedItems()` | Return a list of **at least 5** expected items with value equality; the first `itemCount` of them are what `CreateSut` must yield |

#### Optional and Deprecated Members

| Member | Description |
|--------|-------------|
| `CreateSut(int itemCount)` | Non-virtual convenience for your own tests: calls the four-argument `CreateSut` with the defaults (`int.MaxValue`, `0`, `1000`) |
| `CreateSutOverSource(IEnumerable<TItem> source, int maximumItemCount)` | Virtual. Override to return an extractor that reads from `source` and the base class adds "no over-read" tests (it stops pulling once `MaximumItemCount` is reached). The default returns `null`, which skips those tests |
| `CreateSutWithTimer(IProgressTimer timer)` | **Deprecated** — virtual and no longer called by the contract. Progress timing is driven through `ManualProgressTimerCore` (see [Progress Timing in Tests](#progress-timing-in-tests)), so remove existing overrides |

#### What the Base Class Tests

- All four `ExtractAsync` overloads (plain, cancellation, progress, progress+cancellation)
- `CancellationToken` stops extraction mid-sequence
- An already-cancelled token throws `OperationCanceledException` immediately
- `CurrentItemCount` increments for each yielded item and resets per run
- `ReportingInterval` defaults to 1000 and rejects values less than 1
- `MaximumItemCount` stops extraction at the limit and rejects values less than 1
- `SkipItemCount` skips the first N items and rejects values less than 0
- Progress callbacks fire when the timer ticks
- Empty source handling works correctly
- `CurrentErrorItemCount` starts at 0


### LoaderBaseContractTests\<TSut, TItem, TProgress\>

Use this base class to test any class that inherits from `LoaderBase<TDestination, TProgress>`.

#### Abstract Methods You Must Implement

| Method | Description |
|--------|-------------|
| `CreateSut(int itemCount, int maximumItemCount, int skipItemCount, int reportingInterval)` | Create an instance of your loader configured with the three values (`itemCount` says how many items will be loaded) |
| `CreateSourceItems()` | Return a list of **at least 5** source items with value equality |

`CreateSut(int itemCount)` and the deprecated `CreateSutWithTimer` work as for extractors.

#### What the Base Class Tests

- All four `LoadAsync` overloads
- Null guards: throws `ArgumentNullException` for null `items` or `progress`
- `CancellationToken` stops loading mid-sequence
- `CurrentItemCount` increments for each loaded item
- `MaximumItemCount`, `SkipItemCount` validation and behavior
- Progress callbacks fire deterministically
- Empty source handling


### TransformerBaseContractTests\<TSut, TItem, TProgress\>

Same pattern as the extractor tests, for a `TransformerBase<TItem, TItem, TProgress>` (the contract feeds items in and expects the same type out). Implement the four-argument `CreateSut` and `CreateExpectedItems`.


### Other Contract Test Base Classes

`Wolfgang.Etl.TestKit.Xunit` also ships narrower and opt-in contracts. Each is an abstract xUnit class you derive from in the same way.

| Class | Verifies |
|-------|----------|
| `ExtractAsyncContractTests<TSut, TItem>` and the `ExtractWith...`, `LoadAsync...`, `LoadWith...`, `TransformAsync...`, `TransformWith...` variants | A component that implements one of the framework interfaces directly, without deriving from a base class |
| `CancellationContractTests<TSut>` | The stage stops promptly after cancellation instead of draining its source |
| `DisposableStageContractTests<TSut>` | `Dispose` / `DisposeAsync` behaviour, and that a disposed stage throws `ObjectDisposedException` |
| `ErrorHandlingContractTests<TSut>` | The per-item error hook (`OnItemError` / `HandleItemError`, `CurrentErrorItemCount`) |
| `RetryContractTests<TSut>` | A `WrapWorkerExecution` retry override recovers from a transient fault and gives up on a permanent one |
| `IdempotentExtractorContractTests<TSut, TItem, TProgress>`, `IdempotentLoaderContractTests<TSut, TItem>`, `IdempotentTransformerContractTests<TSut, TItem>` | Running the stage twice produces the same result |
| `AllocationBudgetContractTests<TSut>` | The hot path stays within a declared per-item allocation budget |
| `SupportsDryRunContractTests` | A stage in dry-run mode runs its pipeline without the destination side effect |
| `EtlPipelineContractTests<TItem, TProgress>` | Composition through `EtlPipeline.Create().From(...).To(...).RunAsync()` and the `EtlPipelineProgress` it reports |


## Test Doubles

All in `Wolfgang.Etl.TestKit`.

| Class | Purpose |
|-------|---------|
| `TestExtractor<T>` | Yields items from an `IEnumerable<T>`, an `IEnumerator<T>` or a factory delegate |
| `TestTransformer<T>` | Passes items through unchanged |
| `TestLoader<T>` | Consumes the stream; with `collectItems: true` exposes what it received via `GetCollectedItems()` |
| `FaultyExtractor<T>`, `FaultyTransformer<T>`, `FaultyLoader<T>` | Like the doubles above, but inject deterministic faults (throw at a given item, after completion, ...) for error-path tests |
| `DelayingExtractor<T>` | Waits a configurable delay before each item — for timeout, cancellation and progress-cadence tests |
| `RetryingExtractor<T>` | Fails its first N worker runs with a transient fault, then succeeds — for retry tests |
| `SnapshotTestLoader<T>` | Captures items and renders a deterministic, diff-friendly snapshot string for approval tests |
| `RecordingMiddleware<T>` | An `IItemMiddleware<T>` that records every item it sees |


## Test Helpers

### Progress Timing in Tests

The base classes create their progress timer when a run starts. To make progress callbacks deterministic, attach a `ManualProgressTimerCore` (namespace `Wolfgang.Etl.TestKit`) with the `WithManualProgressTimer` extension before the run, then call `Tick()` whenever you want a report:

```csharp
var timer = new ManualProgressTimerCore();
var extractor = new TestExtractor<int>(new[] { 1, 2, 3 }).WithManualProgressTimer(timer);

await using var e = extractor.ExtractAsync(progress).GetAsyncEnumerator();
await e.MoveNextAsync();
timer.Tick();            // fires the progress callback exactly once
```

`WithManualProgressTimer` has overloads for `ExtractorBase`, `TransformerBase` and `LoaderBase` and reaches the base class's internal timer seam, so **your component needs no timer-injection plumbing** — no `IProgressTimer` field, internal constructor or `CreateProgressTimer` override. The contract base classes use this mechanism themselves.

Likewise, `ManualTimeSource` plus `WithTimeSource(...)` freezes the clock behind `Report.Elapsed`, `ItemsPerSecond`, `PercentComplete` and `EstimatedRemaining`; call `Advance(TimeSpan)` to move it.

!!! note "Earlier releases: the timer-injection pattern"
    Before `ManualProgressTimerCore`, each component carried an `IProgressTimer? _progressTimer` field, an internal constructor taking an `IProgressTimer`, and a `CreateProgressTimer` override, so tests could inject `ManualProgressTimer` and the contract could call `CreateSutWithTimer`. That pattern still compiles, but the contract no longer uses it; new components should leave it out.

### ManualProgressTimer

`Wolfgang.Etl.TestKit.Xunit.ManualProgressTimer` is an `IProgressTimer` whose `Elapsed` event fires synchronously when you call `Fire()`. Use it when you test code that takes an `IProgressTimer` directly.

### SynchronousProgress\<T\> and ProgressCapture\<T\>

Both are `IProgress<T>` implementations in `Wolfgang.Etl.TestKit.Xunit` that run on the calling thread, unlike `Progress<T>` which dispatches through `SynchronizationContext`. `SynchronousProgress<T>` invokes a callback; `ProgressCapture<T>` records every report so `ProgressAssert` can make assertions over the sequence.

```csharp
var reports = new List<JsonReport>();
var progress = new SynchronousProgress<JsonReport>(report => reports.Add(report));
```


## Complete Example: Testing a JSON Extractor

This is how `Wolfgang.Etl.Json` tests its `JsonLineExtractor` with the contract tests. `JsonLineExtractorOptions` derives from `ExtractorOptions`, so the three configuration values go straight into it:

```csharp
using Wolfgang.Etl.Json;
using Wolfgang.Etl.TestKit.Xunit;

public class JsonLineExtractorTests
    : ExtractorBaseContractTests
    <
        JsonLineExtractor<PersonRecord>,
        PersonRecord,
        JsonReport
    >
{
    // At least 5 items required by the contract base
    private static readonly IReadOnlyList<PersonRecord> ExpectedItems = new List<PersonRecord>
    {
        new() { FirstName = "Alice", LastName = "Smith", Age = 30 },
        new() { FirstName = "Bob", LastName = "Jones", Age = 25 },
        new() { FirstName = "Carol", LastName = "White", Age = 35 },
        new() { FirstName = "Dave", LastName = "Brown", Age = 40 },
        new() { FirstName = "Eve", LastName = "Davis", Age = 28 },
    };



    // Helper: create a JSONL stream with the given number of items
    private static MemoryStream CreateJsonlStream(int itemCount)
    {
        var lines = ExpectedItems
            .Take(itemCount)
            .Select(item => JsonSerializer.Serialize(item));
        var content = string.Join("\n", lines);
        return new MemoryStream(Encoding.UTF8.GetBytes(content));
    }



    // Factory: an extractor that yields exactly itemCount items, configured as the contract asks
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



    // Return the full list of expected items
    protected override IReadOnlyList<PersonRecord> CreateExpectedItems() => ExpectedItems;



    // Domain-specific tests beyond what the contract covers:

    [Fact]
    public async Task ExtractAsync_when_blank_lines_present_skips_them()
    {
        var line1 = JsonSerializer.Serialize(ExpectedItems[0]);
        var line2 = JsonSerializer.Serialize(ExpectedItems[1]);
        var content = $"{line1}\n\n  \n{line2}\n";
        var stream = new MemoryStream(Encoding.UTF8.GetBytes(content));

        var sut = new JsonLineExtractor<PersonRecord>(stream);

        var results = new List<PersonRecord>();
        await foreach (var item in sut.ExtractAsync())
        {
            results.Add(item);
        }

        Assert.Equal(2, results.Count);
    }
}
```

With just the two factory methods, the base class provides the full `ExtractorBase` contract. Your domain-specific tests only need to cover behavior unique to your implementation (blank lines, null deserialization, custom options, etc.).


## NuGet References

In your test project `.csproj`:

```xml
<PackageReference Include="Wolfgang.Etl.TestKit" Version="0.26.1" />
<PackageReference Include="Wolfgang.Etl.TestKit.Xunit" Version="0.26.1" />
<PackageReference Include="xunit" Version="2.9.3" />
<!-- net8.0 and later: runner 3.x. Test projects that also target net462 to net7.0: 2.8.2 -->
<PackageReference Include="xunit.runner.visualstudio" Version="3.0.1" />
```

!!! warning "Use the xUnit v2 framework, not v3"
    The TestKit.Xunit contract test base classes are built on the xUnit v2 framework (`xunit.core` 2.9.3); the xUnit v3 framework packages are incompatible. The *runner* is different: `xunit.runner.visualstudio` 3.x runs v2 tests and works on net8.0 and later, while test projects that also target net462 to net7.0 need runner 2.8.2 for those targets.


## TItem Must Support Value Equality

The contract tests compare extracted/loaded items using `Assert.Equal`, which relies on value equality. Use `record` types for your test items:

```csharp
[ExcludeFromCodeCoverage]
public record PersonRecord
{
    public string? FirstName { get; set; }
    public string? LastName { get; set; }
    public int Age { get; set; }
}
```

See [Your First Extractor § Define your test model](../getting-started/your-first-extractor.md#define-your-test-model) for why `[ExcludeFromCodeCoverage]` is applied on test-support POCOs.


## See Also

- [Your First Extractor](../getting-started/your-first-extractor.md) — step-by-step extractor guide
- [Your First Loader](../getting-started/your-first-loader.md) — step-by-step loader guide
- [Extractor](extractor.md) — API reference for extractor interfaces and base class
- [Loader](loader.md) — API reference for loader interfaces and base class
- [Transformer](transformer.md) — API reference for transformer interfaces and base class
- [Pipeline Composition](../guides/pipeline-composition.md) — wiring extractor, transformer, and loader together into a pipeline
