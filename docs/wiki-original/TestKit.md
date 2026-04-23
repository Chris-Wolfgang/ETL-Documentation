# TestKit

The `Wolfgang.Etl.TestKit` and `Wolfgang.Etl.TestKit.Xunit` packages provide test doubles and contract test base classes for verifying that your extractors, transformers, and loaders conform to the expected behavior defined by the Abstractions.

## Packages

| Package | Version | Purpose |
|---------|---------|---------|
| `Wolfgang.Etl.TestKit` | 0.5.0 | Test doubles: `TestExtractor<T>`, `TestLoader<T>`, `TestTransformer<T>` |
| `Wolfgang.Etl.TestKit.Xunit` | 0.5.0 | Contract test base classes for xUnit |

## Why Use TestKit?

When you build a custom extractor, loader, or transformer, the base classes in `Wolfgang.Etl.Abstractions` handle orchestration (progress reporting, cancellation token wiring, async enumeration). Your derived class only implements `ExtractWorkerAsync`, `LoadWorkerAsync`, or `TransformWorkerAsync`. But you still need to verify that:

1. Your worker methods correctly call `IncrementCurrentItemCount()` for each item
2. `SkipItemCount` and `MaximumItemCount` are respected
3. Cancellation tokens are honored
4. Progress reporting works through the timer injection pattern
5. Constructor validation rejects null arguments

The contract test base classes in `TestKit.Xunit` test **all of this automatically**. You implement three factory methods, and the base class provides 20+ test cases that verify your implementation against the contract.


## Contract Test Base Classes

### ExtractorBaseContractTests\<TSut, TItem, TProgress\>

Use this base class to test any class that inherits from `ExtractorBase<TSource, TProgress>`.

#### Abstract Methods You Must Implement

| Method | Description |
|--------|-------------|
| `CreateSut(int itemCount)` | Create an instance of your extractor configured to yield exactly `itemCount` items |
| `CreateExpectedItems()` | Return a list of **at least 5** expected items with value equality |
| `CreateSutWithTimer(IProgressTimer timer)` | Create an instance with an injected `IProgressTimer` for testing progress reporting |

#### What the Base Class Tests

- All four `ExtractAsync` overloads (plain, cancellation, progress, progress+cancellation)
- `CancellationToken` stops extraction mid-sequence
- An already-cancelled token throws `OperationCanceledException` immediately
- `CurrentItemCount` increments for each yielded item
- `ReportingInterval` defaults to 1000 and rejects values less than 1
- `MaximumItemCount` stops extraction at the limit and rejects values less than 1
- `SkipItemCount` skips the first N items and rejects values less than 0
- Progress callbacks fire when the timer fires
- Empty source handling works correctly


### LoaderBaseContractTests\<TSut, TItem, TProgress\>

Use this base class to test any class that inherits from `LoaderBase<TDestination, TProgress>`.

#### Abstract Methods You Must Implement

| Method | Description |
|--------|-------------|
| `CreateSut(int itemCount)` | Create an instance of your loader (the `itemCount` parameter indicates how many items will be loaded) |
| `CreateSourceItems()` | Return a list of **at least 5** source items with value equality |
| `CreateSutWithTimer(IProgressTimer timer)` | Create an instance with an injected `IProgressTimer` |

#### What the Base Class Tests

- All four `LoadAsync` overloads
- Null guards: throws `ArgumentNullException` for null `items` or `progress`
- `CancellationToken` stops loading mid-sequence
- `CurrentItemCount` increments for each loaded item
- `MaximumItemCount`, `SkipItemCount` validation and behavior
- Progress callbacks fire deterministically with `ManualProgressTimer`
- Empty source handling


### TransformerBaseContractTests\<TSut, TItem, TProgress\>

Same pattern as the extractor tests. Implement `CreateSut`, `CreateExpectedItems`, and `CreateSutWithTimer`.


## Test Helpers

### ManualProgressTimer

A test double for `IProgressTimer` that gives you deterministic control over when progress callbacks fire.

```csharp
var timer = new ManualProgressTimer();

// ... start an extraction or load operation ...

timer.Fire();  // Synchronously triggers the Elapsed event
```

In production, `SystemProgressTimer` fires on a background thread at the configured `ReportingInterval`. During tests, `ManualProgressTimer` lets you fire the event exactly when you want, making assertions deterministic.

### SynchronousProgress\<T\>

A test double for `IProgress<T>` that invokes the callback synchronously on the calling thread, unlike `Progress<T>` which dispatches through `SynchronizationContext`.

```csharp
var reports = new List<JsonReport>();
var progress = new SynchronousProgress<JsonReport>(report => reports.Add(report));
```


## Timer Injection Pattern

The contract tests need to inject a `ManualProgressTimer` into your component. Since this is test-only plumbing, the pattern uses an `internal` constructor exposed to the test project via `InternalsVisibleTo`.

### Step 1: Add the fields to your class

```csharp
public class MyExtractor<TRecord> : ExtractorBase<TRecord, MyReport>
    where TRecord : notnull
{
    private readonly IProgressTimer? _progressTimer;
    private bool _progressTimerWired;

    // ... other fields ...
}
```

### Step 2: Add the internal constructor

```csharp
internal MyExtractor
(
    Stream stream,
    JsonSerializerOptions options,
    ILogger logger,
    IProgressTimer timer
)
{
    _stream = stream ?? throw new ArgumentNullException(nameof(stream));
    _options = options ?? throw new ArgumentNullException(nameof(options));
    _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    _progressTimer = timer ?? throw new ArgumentNullException(nameof(timer));
}
```

### Step 3: Override CreateProgressTimer

```csharp
protected override IProgressTimer CreateProgressTimer(IProgress<MyReport> progress)
{
    if (_progressTimer is not null)
    {
        if (!_progressTimerWired)
        {
            _progressTimerWired = true;
            _progressTimer.Elapsed += () => progress.Report(CreateProgressReport());
        }

        return _progressTimer;
    }

    return base.CreateProgressTimer(progress);
}
```

The `_progressTimerWired` flag prevents duplicate event subscriptions if `CreateProgressTimer` is called more than once.

### Step 4: Expose internals to the test project

Create `Properties/AssemblyInfo.cs` in your source project:

```csharp
using System.Runtime.CompilerServices;

[assembly: InternalsVisibleTo("MyProject.Tests.Unit")]
```


## Complete Example: Testing a JSON Extractor

Here is how `Wolfgang.Etl.Json` tests its `JsonLineExtractor` using the contract tests:

```csharp
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


    // Factory: create an extractor that yields exactly itemCount items
    protected override JsonLineExtractor<PersonRecord> CreateSut(int itemCount) =>
        new
        (
            CreateJsonlStream(itemCount),
            NullLogger<JsonLineExtractor<PersonRecord>>.Instance
        );


    // Return the full list of expected items
    protected override IReadOnlyList<PersonRecord> CreateExpectedItems() => ExpectedItems;


    // Factory: create an extractor with an injected timer (uses internal constructor)
    protected override JsonLineExtractor<PersonRecord> CreateSutWithTimer
    (
        IProgressTimer timer
    ) =>
        new
        (
            CreateJsonlStream(ExpectedItems.Count),
            new JsonSerializerOptions(),
            NullLogger<JsonLineExtractor<PersonRecord>>.Instance,
            timer
        );


    // Domain-specific tests beyond what the contract covers:

    [Fact]
    public async Task ExtractAsync_when_blank_lines_present_skips_them()
    {
        var line1 = JsonSerializer.Serialize(ExpectedItems[0]);
        var line2 = JsonSerializer.Serialize(ExpectedItems[1]);
        var content = $"{line1}\n\n  \n{line2}\n";
        var stream = new MemoryStream(Encoding.UTF8.GetBytes(content));

        var sut = new JsonLineExtractor<PersonRecord>
        (
            stream,
            NullLogger<JsonLineExtractor<PersonRecord>>.Instance
        );

        var results = new List<PersonRecord>();
        await foreach (var item in sut.ExtractAsync())
        {
            results.Add(item);
        }

        Assert.Equal(2, results.Count);
    }
}
```

With just the three factory methods, the base class provides 20+ tests covering the full `ExtractorBase` contract. Your domain-specific tests only need to cover behavior unique to your implementation (blank lines, null deserialization, custom options, etc.).


## NuGet References

In your test project `.csproj`:

```xml
<PackageReference Include="Wolfgang.Etl.TestKit" Version="0.5.0" />
<PackageReference Include="Wolfgang.Etl.TestKit.Xunit" Version="0.5.0" />
<PackageReference Include="xunit" Version="2.9.3" />
<PackageReference Include="xunit.runner.visualstudio" Version="2.8.2" />
```

> **Important:** Use xunit 2.x, not 3.x. xunit v3 is incompatible with the TestKit.Xunit contract test base classes.


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


## See Also

- [Building an Extractor](Building-an-Extractor) -- step-by-step guide with real code
- [Building a Loader](Building-a-Loader) -- step-by-step guide with real code
- [Extractor](Extractor) -- API reference for extractor interfaces and base class
- [Loader](Loader) -- API reference for loader interfaces and base class
- [Transformer](Transformer) -- API reference for transformer interfaces and base class
