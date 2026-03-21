# Your First Extractor

This guide walks through building a complete extractor from scratch, using `JsonLineExtractor` from `Wolfgang.Etl.Json` as a real-world example.

## Prerequisites

Your source project needs:

```xml
<PackageReference Include="Wolfgang.Etl.Abstractions" Version="0.10.2" />
```

Plus whatever libraries your extractor needs to read its format (e.g. `System.Text.Json` for JSON).


## Step 1: Define Your Progress Report

Every extractor needs a progress report type. The base class `Report` from Abstractions provides `CurrentItemCount`. Extend it with format-specific information:

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

| Field | Purpose |
|-------|---------|
| `_stream` | The data source your extractor reads from |
| `_options` | Format-specific configuration (optional) |
| `_logger` | Structured logging via `Microsoft.Extensions.Logging` |
| `_progressTimer` | Nullable timer for test injection (see [TestKit](#step-8-write-tests)) |
| `_progressTimerWired` | Guard flag to prevent duplicate timer event subscriptions |


## Step 3: Constructors

You need at least two public constructors plus one internal constructor for testing:

```csharp
    // Public constructor (minimal)
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



    // Public constructor (with options)
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



    // Internal constructor (for timer injection in tests)
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

!!! note
    The internal constructor accepts `ILogger` (not `ILogger<T>`) so the test project can pass `NullLogger` easily.


## Step 4: Implement ExtractWorkerAsync

This is the core of your extractor. It is an `async` iterator method that yields items one at a time:

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

            if (string.IsNullOrWhiteSpace(line))
            {
                continue;
            }

            var item = JsonSerializer.Deserialize<TRecord>(line, _options);
            if (item is null)
            {
                continue;
            }

            if (skipBudget > 0)
            {
                skipBudget--;
                IncrementCurrentSkippedItemCount();
                continue;
            }

            if (CurrentItemCount >= MaximumItemCount)
            {
                break;
            }

            IncrementCurrentItemCount();
            yield return item;
        }
    }
```

### Critical Rules

1. **Call `IncrementCurrentItemCount()` exactly once per yielded item.** Not in helper methods, not conditionally — once, right before or after the `yield return`.

2. **Implement `SkipItemCount` yourself.** The base class does not handle skipping. Use a local `skipBudget` variable.

3. **Implement `MaximumItemCount` yourself.** Check `CurrentItemCount >= MaximumItemCount` before yielding and `break` when reached.

4. **Respect the `CancellationToken`.** Call `token.ThrowIfCancellationRequested()` in your loop and pass `token` to any async calls that accept it.

5. **Use `[EnumeratorCancellation]` on the token parameter.** Required for `async IAsyncEnumerable` methods so that `WithCancellation()` works correctly.

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


## Step 6: Override CreateProgressTimer

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

The `_progressTimerWired` flag is critical — without it, calling `ExtractAsync` with progress twice would subscribe the `Elapsed` event handler a second time, causing duplicate reports.


## Step 7: Expose Internals to the Test Project

Create `Properties/AssemblyInfo.cs`:

```csharp
using System.Runtime.CompilerServices;

[assembly: InternalsVisibleTo("Wolfgang.Etl.Json.Tests.Unit")]
```


## Step 8: Write Tests

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



    private static MemoryStream CreateJsonlStream(int itemCount)
    {
        var lines = ExpectedItems
            .Take(itemCount)
            .Select(item => JsonSerializer.Serialize(item));
        var content = string.Join("\n", lines);
        return new MemoryStream(Encoding.UTF8.GetBytes(content));
    }
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
extractor.SkipItemCount = 100;
extractor.MaximumItemCount = 50;

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

If targeting older frameworks (netstandard2.0, net462), some APIs do not support `CancellationToken` overloads. Use `#if` preprocessor directives:

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
