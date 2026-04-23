# Your First Extractor

This guide walks through building a complete extractor from scratch, using `JsonLineExtractor` from `Wolfgang.Etl.Json` as a real-world example.

## Prerequisites

Your source project needs:

```xml
<PackageReference Include="Wolfgang.Etl.Abstractions" Version="0.10.2" />
```

Plus whatever libraries your extractor needs to read its format (e.g. `System.Text.Json` for JSON).


## Interfaces vs. the Base Class

There are two ways to build an extractor:

1. **Inherit from `ExtractorBase<TSource, TProgress>`** — the recommended path. The base class handles orchestration (progress reporting, cancellation wiring, skip/max item counts, progress timer setup) and leaves you to implement a single `ExtractWorkerAsync` method. This guide focuses on this path.
2. **Implement one of the extractor interfaces directly** — for full control. You get to decide exactly how every piece of the extractor behaves, at the cost of implementing everything yourself.

### The Four Extractor Interfaces

See [Architecture](../architecture.md#interfaces) for the full diamond structure shared across all three stages. For extractors specifically:

| Interface | What it adds |
|-----------|--------------|
| `IExtractAsync<TSource>` | `ExtractAsync()` returning `IAsyncEnumerable<TSource>` — the bare minimum |
| `IExtractWithCancellationAsync<TSource>` | Adds a `CancellationToken` overload |
| `IExtractWithProgressAsync<TSource, TProgress>` | Adds an `IProgress<TProgress>` overload |
| `IExtractWithProgressAndCancellationAsync<TSource, TProgress>` | Adds an overload with both |

Implement only the interface your extractor actually needs — an extractor that does not meaningfully support cancellation should not implement a cancellation interface.

### When to Use the Base Class

Use `ExtractorBase<TSource, TProgress>` unless you have a specific reason not to. It implements all four interfaces at once, honors `SkipItemCount` and `MaximumItemCount` consistently, and gives you the timer-injection pattern needed to test progress reporting (see Step 8).

Hand-implementing the interfaces is valid and supported — for example, if you are wrapping a third-party reader with its own async loop and cannot match the base-class lifecycle — but you are responsible for matching the contract that other extractors implement. The contract test base classes in `TestKit.Xunit` (see Step 8) verify behavior consistent with the base class, so deliberate deviations may fail those tests by design.


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
    private int _progressTimerWired;
```

| Field | Purpose |
|-------|---------|
| `_stream` | The data source your extractor reads from |
| `_options` | Format-specific configuration (optional) |
| `_logger` | Structured logging via `Microsoft.Extensions.Logging` |
| `_progressTimer` | Nullable timer for test injection (see [TestKit](#step-8-write-tests)) |
| `_progressTimerWired` | Guard flag (`int`, set atomically via `Interlocked.CompareExchange`) preventing duplicate timer event subscriptions |

The guard is an `int` rather than a `bool` so it can be set atomically with `Interlocked.CompareExchange` in Step 6 — the `CreateProgressTimer` override may be entered from more than one thread if a caller starts multiple progress-reporting operations concurrently.


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
            if (Interlocked.CompareExchange(ref _progressTimerWired, 1, 0) == 0)
            {
                _progressTimer.Elapsed += () => progress.Report(CreateProgressReport());
            }

            return _progressTimer;
        }

        return base.CreateProgressTimer(progress);
    }
```

Two things to note about this override:

- **The `Interlocked.CompareExchange` guard prevents duplicate event subscriptions** if `CreateProgressTimer` is called more than once, and does so atomically so it remains correct under concurrent entry. The subscription is wired exactly once regardless of how many threads reach the override.
- **The fall-through `return base.CreateProgressTimer(progress)` is required.** When no test timer is injected (the production path), the base class default constructs a `SystemProgressTimer` wired to `ReportProgress`, starts it at `ReportingInterval`, and returns it. Omitting the fall-through would leave the production code path without a working timer.


## Step 7: Expose Internals to the Test Project

Create `Properties/AssemblyInfo.cs`:

```csharp
using System.Runtime.CompilerServices;

[assembly: InternalsVisibleTo("Wolfgang.Etl.Json.Tests.Unit")]
```


## Step 8: Write Tests

### Define your test model

Test-support records used to drive the contract tests should be marked `[ExcludeFromCodeCoverage]`:

```csharp
[ExcludeFromCodeCoverage]
public record PersonRecord
{
    public string? FirstName { get; set; }
    public string? LastName { get; set; }
    public int Age { get; set; }
}
```

**Why `[ExcludeFromCodeCoverage]`?** Test-support POCOs like `PersonRecord` exist only to drive tests — they have no production logic to verify. `record` types also generate compiler-authored members (`Equals`, `GetHashCode`, `<Clone>$`, the copy constructor, deconstruction) that coverage tools flag as uncovered branches unless the test happens to exercise every generated path. Applying `[ExcludeFromCodeCoverage]` keeps your coverage numbers honest: they reflect how well your production code is tested, not how thoroughly you exercised synthetic equality on a test-only class.

Use this attribute sparingly in production code. It is appropriate on test models and fixtures, generated code that does not round-trip through your edits, and genuinely unreachable defensive branches — not as a way to hide poorly tested production code.

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

**Why this matters beyond "it saves writing tests."** Inheriting the contract test base class verifies that your extractor behaves the same as every other extractor built on `ExtractorBase`. Two extractors with the same `TSource` are interchangeable not just in signature but in *behavior* — `CurrentItemCount` increments at the same moment, `SkipItemCount` and `MaximumItemCount` are enforced at the same boundaries, cancellation stops the pull at the same point, and so on.

This is the [Liskov substitution principle](https://en.wikipedia.org/wiki/Liskov_substitution_principle) in practice. Without the shared contract test, one implementor might increment `CurrentItemCount` before yielding each item while another increments it after — both "work", but swapping them breaks anything downstream that reads `CurrentItemCount` from a progress callback. Consumers of the ETL pipeline should not have to know which concrete extractor they are holding. The contract tests make that promise enforceable.

Deliberately deviating from the contract (for example, if you are hand-implementing the interfaces rather than inheriting from `ExtractorBase`) is supported, but you should expect the contract tests to fail for your deviation by design — treat any such failure as a signal that consumers swapping in your implementation will see different behavior than they would from a sibling extractor.

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
