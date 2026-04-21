# Building an Extractor

This guide walks through building a complete extractor from scratch, using `JsonLineExtractor` from the `Wolfgang.Etl.Json` library as a real-world example. By the end, you will have a fully tested extractor that reads JSONL data from a stream.

## Prerequisites

Your source project needs these NuGet references:

```xml
<PackageReference Include="Wolfgang.Etl.Abstractions" Version="0.10.2" />
```

Plus whatever libraries your extractor needs to read its format (e.g., `System.Text.Json` for JSON, a CSV parser for CSV, etc.).


## Interfaces vs. the Base Class

There are two ways to build an extractor:

1. **Inherit from `ExtractorBase<TSource, TProgress>`** -- the recommended path. The base class handles orchestration (progress reporting, cancellation wiring, skip/max item counts, progress timer setup) and leaves you to implement a single `ExtractWorkerAsync` method. This guide focuses on this path.
2. **Implement one of the extractor interfaces directly** -- for full control. You get to decide exactly how every piece of the extractor behaves, at the cost of implementing everything yourself.

### The Four Extractor Interfaces

The extractor interfaces form a diamond -- start with `IExtractAsync<TSource>` and add only the capabilities you need:

```
           IExtractAsync<TSource>
                  /       \
                 /         \
IExtractWithCancellation   IExtractWithProgress
     Async<T>              Async<T, TProgress>
                 \         /
                  \       /
     IExtractWithProgressAndCancellation
                Async<T, TProgress>
```

| Interface | What it adds |
|-----------|--------------|
| `IExtractAsync<TSource>` | `ExtractAsync()` returning `IAsyncEnumerable<TSource>` -- the bare minimum |
| `IExtractWithCancellationAsync<TSource>` | Adds a `CancellationToken` overload |
| `IExtractWithProgressAsync<TSource, TProgress>` | Adds an `IProgress<TProgress>` overload |
| `IExtractWithProgressAndCancellationAsync<TSource, TProgress>` | Adds an overload with both |

The cancellation and progress interfaces both inherit from `IExtractAsync<TSource>`. The combined interface inherits from both, which is the diamond. Implement only the interface your extractor actually needs -- an extractor that does not meaningfully support cancellation should not implement a cancellation interface.

### When to Use the Base Class

Use `ExtractorBase<TSource, TProgress>` unless you have a specific reason not to. It implements all four interfaces at once, honors `SkipItemCount` and `MaximumItemCount` consistently, and gives you the timer-injection pattern needed to test progress reporting (see [TestKit](TestKit)).

Hand-implementing the interfaces is valid and supported -- for example, if you are wrapping a third-party reader with its own async loop and cannot match the base-class lifecycle -- but you are responsible for matching the contract that other extractors implement. The contract test base classes in [TestKit.Xunit](TestKit) verify behavior consistent with the base class, so deliberate deviations may fail those tests by design.


## Step 1: Define Your Progress Report

Every extractor needs a progress report type. The base class `Report` from Abstractions provides `CurrentItemCount`. You can extend it with format-specific information:

```csharp
using Wolfgang.Etl.Abstractions;

namespace Wolfgang.Etl.Json;

public record JsonReport : Report
{
    public JsonReport
    (
        int currentItemCount,
        int currentSkippedItemCount
    )
        : base(currentItemCount)
    {
        CurrentSkippedItemCount = currentSkippedItemCount;
    }

    public int CurrentSkippedItemCount { get; }
}
```

Using a `record` type gives you value equality, immutability, and a clean `ToString()` for free.


## Step 2: Create the Extractor Class

Inherit from `ExtractorBase<TRecord, TProgress>` where:
- `TRecord` is the type of items your extractor yields (must be `notnull`)
- `TProgress` is your progress report type

```csharp
public class JsonLineExtractor<TRecord> : ExtractorBase<TRecord, JsonReport>
    where TRecord : notnull
{
    private readonly Stream _stream;
    private readonly JsonSerializerOptions? _options;
    private readonly ILogger _logger;
    private readonly IProgressTimer? _progressTimer;
    private bool _progressTimerWired;
```

### Key Fields

| Field | Purpose |
|-------|---------|
| `_stream` | The data source your extractor reads from |
| `_options` | Format-specific configuration (optional in JSON; could be delimiter settings for CSV, etc.) |
| `_logger` | Structured logging via `Microsoft.Extensions.Logging` |
| `_progressTimer` | Nullable timer for test injection (see [TestKit](TestKit)) |
| `_progressTimerWired` | Guard flag to prevent duplicate timer event subscriptions |


## Step 3: Constructors

You need at least two public constructors plus one internal constructor for testing:

### Public constructor (minimal)

```csharp
public JsonLineExtractor
(
    Stream stream,
    ILogger<JsonLineExtractor<TRecord>> logger
)
{
    _stream = stream ?? throw new ArgumentNullException(nameof(stream));
    _logger = logger ?? throw new ArgumentNullException(nameof(logger));
    _options = null;
}
```

### Public constructor (with options)

```csharp
public JsonLineExtractor
(
    Stream stream,
    JsonSerializerOptions options,
    ILogger<JsonLineExtractor<TRecord>> logger
)
{
    _stream = stream ?? throw new ArgumentNullException(nameof(stream));
    _options = options ?? throw new ArgumentNullException(nameof(options));
    _logger = logger ?? throw new ArgumentNullException(nameof(logger));
}
```

### Internal constructor (for timer injection in tests)

```csharp
internal JsonLineExtractor
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

Note: The internal constructor accepts `ILogger` (not `ILogger<T>`) so the test project can pass `NullLogger` easily.


## Step 4: Implement ExtractWorkerAsync

This is the core of your extractor. It is an `async` iterator method that yields items one at a time.

```csharp
protected override async IAsyncEnumerable<TRecord> ExtractWorkerAsync
(
    [EnumeratorCancellation] CancellationToken token
)
{
    var skipBudget = SkipItemCount;

    using var reader = new StreamReader(_stream, leaveOpen: true);

    string? line;
    while ((line = await reader.ReadLineAsync(token).ConfigureAwait(false)) is not null)
    {
        token.ThrowIfCancellationRequested();

        // Skip blank lines
        if (string.IsNullOrWhiteSpace(line))
        {
            continue;
        }

        // Deserialize the line
        var item = JsonSerializer.Deserialize<TRecord>(line, _options);
        if (item is null)
        {
            continue;
        }

        // Handle SkipItemCount
        if (skipBudget > 0)
        {
            skipBudget--;
            IncrementCurrentSkippedItemCount();
            continue;
        }

        // Handle MaximumItemCount
        if (CurrentItemCount >= MaximumItemCount)
        {
            break;
        }

        // Count and yield the item
        IncrementCurrentItemCount();
        yield return item;
    }
}
```

### Critical Rules

1. **Call `IncrementCurrentItemCount()` exactly once per yielded item.** Not in helper methods, not conditionally -- once, right before or after the `yield return`.

2. **Implement `SkipItemCount` yourself.** The base class does not handle skipping. Use a local `skipBudget` variable or check `CurrentSkippedItemCount < SkipItemCount`.

3. **Implement `MaximumItemCount` yourself.** Check `CurrentItemCount >= MaximumItemCount` before yielding and `break` when reached.

4. **Respect the `CancellationToken`.** Call `token.ThrowIfCancellationRequested()` in your loop and pass `token` to any async calls that accept it.

5. **Use `[EnumeratorCancellation]` on the token parameter.** This is required for `async IAsyncEnumerable` methods so that `WithCancellation()` works correctly.

6. **Use `.ConfigureAwait(false)`** on all `await` calls in library code.


## Step 5: Implement CreateProgressReport

Return a new instance of your progress report:

```csharp
protected override JsonReport CreateProgressReport() =>
    new
    (
        CurrentItemCount,
        CurrentSkippedItemCount
    );
```

This method is called by the base class on a timer interval and passed to the `IProgress<T>` callback.


## Step 6: Override CreateProgressTimer (for testing)

```csharp
protected override IProgressTimer CreateProgressTimer(IProgress<JsonReport> progress)
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

When no timer is injected, the base class creates a `SystemProgressTimer` that fires on a background thread. During testing, the injected `ManualProgressTimer` lets you control exactly when progress callbacks fire.


## Step 7: Expose Internals to the Test Project

Create `Properties/AssemblyInfo.cs`:

```csharp
using System.Runtime.CompilerServices;

[assembly: InternalsVisibleTo("Wolfgang.Etl.Json.Tests.Unit")]
```


## Step 8: Write Tests

### Test project structure

```
tests/
  Wolfgang.Etl.Json.Tests.Unit/
    JsonLineExtractorTests.cs
    TestModels/
      PersonRecord.cs
```

### Define your test model

```csharp
[ExcludeFromCodeCoverage]
public record PersonRecord
{
    public string? FirstName { get; set; }
    public string? LastName { get; set; }
    public int Age { get; set; }
}
```

### Inherit the contract test base class

```csharp
public class JsonLineExtractorTests
    : ExtractorBaseContractTests
    <
        JsonLineExtractor<PersonRecord>,
        PersonRecord,
        JsonReport
    >
{
    private static readonly IReadOnlyList<PersonRecord> ExpectedItems = new List<PersonRecord>
    {
        new() { FirstName = "Alice", LastName = "Smith", Age = 30 },
        new() { FirstName = "Bob", LastName = "Jones", Age = 25 },
        new() { FirstName = "Carol", LastName = "White", Age = 35 },
        new() { FirstName = "Dave", LastName = "Brown", Age = 40 },
        new() { FirstName = "Eve", LastName = "Davis", Age = 28 },
    };


    private static MemoryStream CreateJsonlStream(int itemCount)
    {
        var lines = ExpectedItems
            .Take(itemCount)
            .Select(item => JsonSerializer.Serialize(item));
        var content = string.Join("\n", lines);
        return new MemoryStream(Encoding.UTF8.GetBytes(content));
    }


    protected override JsonLineExtractor<PersonRecord> CreateSut(int itemCount) =>
        new
        (
            CreateJsonlStream(itemCount),
            NullLogger<JsonLineExtractor<PersonRecord>>.Instance
        );


    protected override IReadOnlyList<PersonRecord> CreateExpectedItems() => ExpectedItems;


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
}
```

This single class, with just three factory methods, gives you 20+ tests that validate your extractor against the full `ExtractorBase` contract.

### Add domain-specific tests

On top of the contract tests, add tests for behavior unique to your extractor:

```csharp
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
```

### Constructor null guard tests

```csharp
[Fact]
public void Constructor_when_stream_is_null_throws_ArgumentNullException()
{
    Assert.Throws<ArgumentNullException>
    (
        () => new JsonLineExtractor<PersonRecord>
        (
            null!,
            NullLogger<JsonLineExtractor<PersonRecord>>.Instance
        )
    );
}
```


## Using Your Extractor

```csharp
// Simple usage
using var stream = File.OpenRead("data.jsonl");
var extractor = new JsonLineExtractor<Person>(stream, logger);

await foreach (var person in extractor.ExtractAsync(cancellationToken))
{
    Console.WriteLine(person.Name);
}

// With skip and maximum
var extractor = new JsonLineExtractor<Person>(stream, logger);
extractor.SkipItemCount = 100;      // Skip first 100 items
extractor.MaximumItemCount = 50;    // Then take 50 items

// With progress reporting
var progress = new Progress<JsonReport>(report =>
    Console.WriteLine($"Extracted {report.CurrentItemCount} items")
);

await foreach (var person in extractor.ExtractAsync(progress, cancellationToken))
{
    // ...
}
```


## Multi-TFM Considerations

If targeting older frameworks (netstandard2.0, net462, net481), some APIs do not support `CancellationToken` overloads. Use `#if` preprocessor directives:

```csharp
#if NETSTANDARD2_0 || NET462 || NET481
    while ((line = await reader.ReadLineAsync().ConfigureAwait(false)) is not null)
#else
    while ((line = await reader.ReadLineAsync(token).ConfigureAwait(false)) is not null)
#endif
```


## Checklist

- [ ] Inherit `ExtractorBase<TRecord, TProgress>` with `where TRecord : notnull`
- [ ] Implement `ExtractWorkerAsync` with `[EnumeratorCancellation]` on the token
- [ ] Implement `CreateProgressReport()`
- [ ] Call `IncrementCurrentItemCount()` exactly once per yielded item
- [ ] Implement `SkipItemCount` logic in your worker method
- [ ] Implement `MaximumItemCount` logic in your worker method
- [ ] Respect `CancellationToken` (pass to async calls, call `ThrowIfCancellationRequested`)
- [ ] Add `_progressTimer` field, `_progressTimerWired` flag, and internal constructor
- [ ] Override `CreateProgressTimer` with duplicate subscription guard
- [ ] Add `InternalsVisibleTo` in `Properties/AssemblyInfo.cs`
- [ ] Test class inherits `ExtractorBaseContractTests<TSut, TItem, TProgress>`
- [ ] `CreateExpectedItems()` returns at least 5 items
- [ ] Items use `record` types for value equality
- [ ] Add domain-specific tests for format-specific behavior


## See Also

- [Extractor](Extractor) -- API reference for interfaces and base class
- [TestKit](TestKit) -- contract test base classes and test helpers
- [Building a Loader](Building-a-Loader) -- the companion guide for loaders
- [Building a Complete ETL](Building-a-Complete-ETL) -- wiring extractor, transformer, and loader together
